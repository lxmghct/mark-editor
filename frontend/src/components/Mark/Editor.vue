<!--
 * @FileDescription: 具有实体和三元组标注功能的编辑器组件
 * @Author: longxiaoming
 * @Date: 2022/10/16
 * @LastEditors: longxiaoming
 * @LastEditTime: 2022/10/19
 -->

<template>
  <div class="editor">
    <div class="menu">
      <el-checkbox
        v-if="menu.showEntityTag"
        v-model="tagShowCheck"
        @change="_handleTagShowChange"
      >
        显示标签
      </el-checkbox>
      <el-checkbox
        v-if="menu.showTupleLine"
        v-model="lineShowCheck"
        :disabled="!tagShowCheck"
        @change="_handleLineShowChange"
      >
        显示连线
      </el-checkbox>
    </div>
    <div ref="main" class="editor-main" @mouseup="_handleEditorMouseUp">
      <span
        v-for="(word, index) in text"
        :id="`w_${index}`"
        :key="index"
        class="word"
      >
        {{ word }}
      </span>
      <div v-if="entityTagShow" class="tag-container">
        <div
          v-for="(tag, index) in entityTags"
          :id="`t_${tag.id}_${tag.begin}`"
          :key="index"
          class="tag"
          :style="tagStyle[index]"
          @click="_handleTagClick(tag)"
          @mousemove="_handleTagMouseMove(tag)"
          @mouseleave="_handleTagMouseLeave(tag)"
          @contextmenu="_handleTagContextMenu(tag)"
          @mousedown="_handleTagMouseDown(tag)"
          @mouseup="_handleTagMouseUp(tag)"
        >
          {{ tag.name }}
        </div>
      </div>
      <div v-if="tupleLineShow" class="link-line-container">
        <link-line
          v-for="(line, index) in linkLines"
          :key="index"
          :ref="`linkline_${index}`"
          @line-mousemove="_handleLineMouseMove(line, $event)"
          @line-mouseleave="_handleLineMouseLeave(line, $event)"
          @line-click="_handleLineClick(line, $event)"
          @line-contextmenu="_handleLineContextMenu(line, $event)"
        />
      </div>
    </div>
  </div>
</template>

<script>
import LinkLine from './LinkLine.vue'

const tagHeight = 20

export default {

  components: {
    LinkLine
  },

  data() {
    return {
      text: '', // 编辑器文本
      tagColorMap: {}, // 标签颜色映射表
      entityColorMap: {}, // 实体颜色映射表
      entityColorMapField: 'type', // 颜色映射表的字段
      entityMap: {}, // 实体id与实体对象的映射
      tupleMap: {}, // 三元组id与三元组对象的映射
      entityTagShow: true, // 是否显示实体标签
      tupleLineShow: true, // 是否显示三元组连线
      tagShowCheck: true, // 显示标签复选框
      lineShowCheck: true, // 显示连线复选框
      entityTags: [], // 实体标签
      tagStyle: [], // 标签样式
      entitySortedList: [], // 实体列表
      linkLines: [], // 三元组连线
      maxTagRow: 0, // 标签行数
      menu: {
        'showEntityTag': true,
        'showTupleLine': true
      },
      resizeObserver: null
    }
  },

  mounted() {
    this.setColorMap({})
    const obj = this.$refs.main
    // 监听宽度变化
    this.resizeObserver = new ResizeObserver(entries => {
      for (const entry of entries) {
        const cr = entry.contentRect
        this.width = cr.width
        this.height = cr.height
        this._updateTagPositions()
        for (const line of this.linkLines) {
          line.layerObj1 = { left: 0, right: 0, layer: 1 }
          line.layerObj2 = { left: 0, right: 0, layer: 1 }
        }
        this._updateLinkLinePositions()
      }
    })
    this.resizeObserver.observe(obj)
  },

  methods: {
    /**
     * @description 设置文本
     * @param {string} text
     * @return {void}
     */
    setText(text) {
      this.text = text
      this.text = this.text.replace(/\r/g, '')
      for (let i = 0; i < text.length; i++) {
        const obj = document.getElementById(`w_${i}`)
        if (obj) {
          const style = obj.style
          if (text[i] === '\n') {
            style.display = 'block'
          } else {
            style.display = 'inline-block'
          }
          style.textDecoration = 'none'
          style.fontWeight = 'normal'
          style.marginTop = 0
          style.backgroundColor = 'transparent'
          style.cursor = 'text'
        }
      }
    },

    /**
     * @description 设置菜单
     * @param {object} menu
     * @return {void}
     */
    setMenu(menu) {
      this.menu = menu
    },

    /**
     * @description 设置颜色映射表
     * @param {object} colorMap
     * @param {string} field
     * @return {void}
     */
    setColorMap(colorMap, field = 'type') {
      this.tagColorMap = colorMap
      this.entityColorMapField = field
      for (const key in colorMap) {
        let colorStr = colorMap[key]
        // 将#ffffff格式的颜色转换为rgba格式
        try {
          colorStr = colorStr.replace('#', '0x')
          const r = (colorStr & 0xff0000) >> 16
          const g = (colorStr & 0x00ff00) >> 8
          const b = colorStr & 0x0000ff
          colorStr = `rgba(${r},${g},${b},0.15)`
        } catch (e) {
          console.log('wrong color format: ' + colorStr)
          colorStr = 'rgba(0,0,0,0.15)'
        }
        this.entityColorMap[key] = colorStr
      }
      this.tagColorMap['default'] = '#808080'
      this.entityColorMap['default'] = 'rgba(0,0,0,0.15)'
    },

    /**
     * @description 标注实体
     * @param {int} entityId
     * @param {string} type
     * @param {int} begin
     * @param {int} end
     * @return {void}
     */
    markEntity(entityId, type, begin, end) {
      this._markEntity(entityId, type, begin, end)
      let j
      for (j = 0; j < this.entitySortedList.length; j++) {
        if (this.entitySortedList[j].begin < begin) {
          break
        }
      }
      this.entitySortedList.splice(j, 0, {
        begin: begin,
        id: entityId,
        end: end
      })
      if (!this.entityMap[entityId]) {
        this.entityMap[entityId] = {
          type: type,
          location: [{ begin: begin, end: end }]
        }
      } else {
        this.entityMap[entityId].location.push({ begin: begin, end: end })
      }
      this.$nextTick(() => {
        this._updateTagPositions()
      })
    },

    /**
     * @description 删除实体
     * @param {int} entityId
     * @param {int} begin
     * @return {void}
     */
    deleteEntity(entityId, begin) {
      const entity = this.entityMap[entityId]
      if (!entity) { return }
      const location = entity.location
      let i
      for (i = 0; i < location.length; i++) {
        if (location[i].begin === begin) {
          break
        }
      }
      if (i === location.length) { return }
      location.splice(i, 1)
      if (location.length === 0) {
        delete this.entityMap[entityId]
      }
      for (i = 0; i < this.entitySortedList.length; i++) {
        if (this.entitySortedList[i].id === entityId && this.entitySortedList[i].begin === begin) {
          break
        }
      }
      this.entitySortedList.splice(i, 1)
      this.$nextTick(() => {
        this._updateTagPositions()
      })
    },

    /**
     * @description 批量标注实体
     * @param {object} entityMap
     * @return {void}
     */
    markEntities(entityMap) {
      this.entityMap = entityMap
      this.entityTags = []
      this.tagStyle = []
      this.entitySortedList = []
      this.setText(this.text)
      for (let key in entityMap) {
        key = parseInt(key)
        const entity = entityMap[key]
        for (let i = 0; i < entity.location.length; i++) {
          this._markEntity(key, entity.type, entity.location[i].begin, entity.location[i].end)
          this.entitySortedList.push({
            begin: entity.location[i].begin,
            id: key,
            end: entity.location[i].end
          })
        }
      }
      this.entitySortedList.sort((a, b) => {
        return a.begin - b.begin
      })
      this.$nextTick(() => {
        this._updateTagPositions()
        // this.$nextTick(() => {
        //   this._updateLinkLinePositions()
        // })
      })
      this.$forceUpdate()
    },

    /**
     * @description 标注三元组
     * @param {int} tupleId
     * @param {int} subjectId
     * @param {int} objectId
     * @param {string} tupleName
     */
    markTuple(tupleId, subjectId, objectId, tupleName) {
      if (!this.entityMap[subjectId] || !this.entityMap[objectId]) {
        return
      }
      this._markTuple(tupleId, subjectId, objectId, tupleName)
      if (!this.tupleMap[tupleId]) {
        this.tupleMap[tupleId] = {
          subjectId: subjectId,
          objectId: objectId,
          name: tupleName,
          id: tupleId
        }
      }
      this.$nextTick(() => {
        this._updateLinkLinePositions()
      })
    },

    /**
     * @description 删除三元组
     * @param {int} tupleId
     * @return {void}
     */
    deleteTuple(tupleId) {
      const tuple = this.tupleMap[tupleId]
      if (!tuple) { return }
      delete this.tupleMap[tupleId]
      for (let i = 0; i < this.linkLines.length; i++) {
        if (this.linkLines[i].id === tupleId) {
          this.linkLines.splice(i, 1)
          break
        }
      }
      this.$nextTick(() => {
        this._updateLinkLinePositions()
      })
    },

    /**
     * @description 批量标注三元组
     * @param {object} tupleMap
     * @return {void}
     */
    markTuples(tupleMap) {
      this.tupleMap = tupleMap
      this.linkLines = []
      this.maxTagRow = 0
      this.$nextTick(() => {
        for (let key in tupleMap) {
          key = parseInt(key)
          const tuple = tupleMap[key]
          if (!this.entityMap[tuple.subjectId] || !this.entityMap[tuple.objectId]) {
            continue
          }
          this._markTuple(key, tuple.subjectId, tuple.objectId, tuple.name)
        }
        this.$nextTick(() => {
          this._updateLinkLinePositions()
        })
      })
    },

    /**
     * @description 高亮实体
     * @param {int} entityId
     * @param {int} begin
     * return {void}
     */
    highlightEntity(entityId, begin = -1, isHighlight = true) {
      const entity = this.entityMap[entityId]
      if (!entity) { return }
      const location = entity.location
      if (location.length === 0) { return }
      for (let i = 0; i < location.length; i++) {
        // 未指定begin则高亮所有实体
        if (begin === -1 || location[i].begin === begin) {
          const tag = {
            id: entityId,
            name: entity.type,
            begin: location[i].begin,
            end: location[i].end
          }
          if (isHighlight) {
            this._handleTagMouseMove(tag)
          } else {
            this._handleTagMouseLeave(tag)
          }
        }
      }
      if (isHighlight) {
        // 定位到第一个实体
        const tempBegin = begin === -1 ? location[0].begin : begin
        const tag = document.getElementById('t_' + entityId + '_' + tempBegin)
        if (tag) {
          tag.scrollIntoView()
        } else {
          const word = document.getElementById('w_' + tempBegin)
          if (word) {
            word.scrollIntoView()
          }
        }
        // 取消其他实体的高亮
        setTimeout(() => {
          for (let key in this.entityMap) {
            key = parseInt(key)
            if (key !== entityId) {
              this.highlightEntity(key, -1, false)
            }
          }
        })
      }
    },

    _getColor(type, isTag = true) {
      let colorMap = null
      if (isTag) {
        colorMap = this.tagColorMap
      } else {
        colorMap = this.entityColorMap
      }
      if (!type) {
        return colorMap['default']
      }
      let color = colorMap[type]
      if (!color) {
        color = colorMap['default']
      }
      return color
    },

    _createEntityTag(entityId, type, begin, end) {
      this.entityTags.push({
        id: entityId,
        name: type,
        begin: begin,
        end: end
      })
      this.tagStyle.push({
        backgroundColor: this._getColor(type),
        left: '-100px',
        top: '-100px'
      })
    },

    _markEntity(entityId, type, begin, end) {
      const color = this._getColor(type, false)
      try {
        const that = this
        const tag = { id: entityId, name: type, begin: begin, end: end }
        for (let j = begin; j <= end; j++) {
          const obj = document.getElementById('w_' + j)
          obj.style.backgroundColor = color
          obj.style.cursor = 'pointer'
          setTimeout(() => {
            const obj = document.getElementById('w_' + j)
            if (obj) {
              obj.onmouseover = function(e) { that._handleTagMouseMove(tag, e) }
              obj.onmouseleave = function(e) { that._handleTagMouseLeave(tag, e) }
              obj.onclick = function(e) { that._handleTagClick(tag, e) }
              obj.oncontextmenu = function(e) { that._handleTagContextMenu(tag, e) }
            }
          })
        }
      } catch (e) {
        console.log(e)
      }
      this._createEntityTag(entityId, type, begin, end)
    },

    _markTuple(tupleId, subjectId, objectId, tupleName) {
      this.linkLines.push({
        id: tupleId,
        subjectId: subjectId,
        objectId: objectId,
        name: tupleName,
        layerObj1: { left: 0, right: 0, layer: 1 },
        layerObj2: { left: 0, right: 0, layer: 1 }
      })
    },

    _updateTagPositions() {
      if (!this.entityTagShow) { return }
      for (let i = 0; i < this.entitySortedList.length; i++) {
        const e = this.entitySortedList[i]
        const begin = e.begin
        const end = e.end
        try {
          const tag = document.getElementById('t_' + e.id + '_' + begin)
          const tagWidth = tag.offsetWidth
          const word = document.getElementById('w_' + begin)
          const endWord = document.getElementById('w_' + end)
          // 当出现标签标完之后换行的情况, 需要重新计算实体长度和标签位置, 所以用循环
          for (let temp = 0; temp < 2; temp++) {
            const top = word.offsetTop
            let width = word.offsetWidth
            for (let j = begin + 1; j <= end; j++) {
              const word = document.getElementById('w_' + j)
              if (word.offsetTop > top) { break }
              width += word.offsetWidth
            }
            const halfDiff = Math.abs((width - tagWidth) / 2)
            if (width < tagWidth) {
              if (endWord.offsetTop > top && begin > 0) {
                document.getElementById('w_' + (begin - 1)).style.marginRight = width + 'px'
                continue
              }
              word.style.marginLeft = halfDiff + 'px'
              endWord.style.marginRight = halfDiff + 'px'
              tag.style.left = word.offsetLeft - halfDiff + 'px'
            } else {
              tag.style.left = (word.offsetLeft + halfDiff) + 'px'
            }
            if (word.style.marginTop && this.tupleLineShow) {
              if (parseInt(word.style.marginTop) < tagHeight) {
                word.style.marginTop = tagHeight + 'px'
              }
            } else {
              word.style.marginTop = tagHeight + 'px'
            }
            tag.style.top = word.offsetTop - tagHeight + 'px'
            break
          }
        } catch (e) {
          console.log(e)
        }
      }
    },

    _updateLinkLinePositions() {
      if (!this.tupleLineShow) { return }
      this._calculateRowLayers()
      const left = 0
      const right = this.$refs.main.offsetWidth - 30
      const tempLines = []
      const tempHeight = {}
      for (let i = 0; i < this.linkLines.length; i++) {
        const tuple = this.linkLines[i]
        let obj1, obj2
        try {
          const begin1 = this.entityMap[tuple.subjectId].location[0].begin
          const begin2 = this.entityMap[tuple.objectId].location[0].begin
          obj1 = document.getElementById('t_' + tuple.subjectId + '_' + begin1)
          obj2 = document.getElementById('t_' + tuple.objectId + '_' + begin2)
          const h1 = tuple.layerObj1.layer * 12 + tagHeight + 10
          const h2 = tuple.layerObj2.layer * 12 + tagHeight + 10
          if (tempHeight[begin1]) {
            tempHeight[begin1] = Math.max(tempHeight[begin1], h1)
          } else {
            tempHeight[begin1] = h1
          }
          if (tempHeight[begin2]) {
            tempHeight[begin2] = Math.max(tempHeight[begin2], h2)
          } else {
            tempHeight[begin2] = h2
          }
        } catch (e) {
          console.log(e)
          continue
        }
        tempLines.push({
          obj1: obj1,
          obj2: obj2,
          height1: 12 * tuple.layerObj1.layer,
          height2: 12 * tuple.layerObj2.layer,
          text: tuple.name,
          left: left,
          right: right
        })
      }
      for (const key in tempHeight) {
        document.getElementById('w_' + key).style.marginTop = tempHeight[key] + 'px'
      }
      for (let i = 0; i < this.entitySortedList.length; i++) {
        const id = this.entitySortedList[i].id
        const begin = this.entitySortedList[i].begin
        document.getElementById('t_' + id + '_' + begin).style.top = document.getElementById('w_' + begin).offsetTop - tagHeight + 'px'
      }
      for (let i = 0; i < tempLines.length; i++) {
        try {
          this.$refs[`linkline_${i}`][0].createLine(tempLines[i])
        } catch (e) {
          console.log(e)
        }
      }
    },

    _getEntityTagRowMap() {
      const entityTagTopList = []
      const idSet = new Set()
      for (let i = 0; i < this.entitySortedList.length; i++) {
        if (idSet.has(this.entitySortedList[i].id)) { continue }
        idSet.add(this.entitySortedList[i].id)
        const obj = document.getElementById('t_' + this.entitySortedList[i].id + '_' + this.entitySortedList[i].begin)
        if (obj) {
          entityTagTopList.push({
            id: this.entitySortedList[i].id,
            top: obj.offsetTop,
            begin: this.entitySortedList[i].begin
          })
        }
      }
      entityTagTopList.sort((a, b) => {
        return a.top - b.top
      })
      let row = 0
      let lastTop = -1000
      const entityTagRowMap = {}
      for (let i = 0; i < entityTagTopList.length; i++) {
        const e = entityTagTopList[i]
        if (e.top > lastTop) { row++ }
        entityTagRowMap[e.id] = {
          row: row,
          begin: e.begin
        }
        lastTop = e.top
      }
      this.maxTagRow = row
      return entityTagRowMap
    },

    _calculateRowLayers() {
      const maxLen = this.text.length + 1
      const tagRowMap = this._getEntityTagRowMap()
      const rowLayers = []
      for (let i = 0; i < this.maxTagRow; i++) {
        rowLayers.push([])
      }
      for (let i = 0; i < this.linkLines.length; i++) {
        const tuple = this.linkLines[i]
        const obj1 = tagRowMap[tuple.subjectId]
        const obj2 = tagRowMap[tuple.objectId]
        if (!obj1 || !obj2) { continue }
        if (obj1.row === obj2.row) {
          tuple.layerObj1.left = Math.min(obj1.begin, obj2.begin)
          tuple.layerObj1.right = Math.max(obj1.begin, obj2.begin)
          tuple.layerObj1.len = tuple.layerObj1.right - tuple.layerObj1.left
          tuple.layerObj2 = tuple.layerObj1
          rowLayers[obj1.row - 1].push(tuple.layerObj1)
        } else {
          if (obj1.row < obj2.row) {
            tuple.layerObj1.left = obj1.begin
            tuple.layerObj1.right = maxLen
            tuple.layerObj1.len = maxLen - obj1.begin
            tuple.layerObj2.left = -1
            tuple.layerObj2.right = obj2.begin
            tuple.layerObj2.len = obj2.begin + 1
          } else {
            tuple.layerObj1.left = -1
            tuple.layerObj1.right = obj1.begin
            tuple.layerObj1.len = obj1.begin + 1
            tuple.layerObj2.left = obj2.begin
            tuple.layerObj2.right = maxLen
            tuple.layerObj2.len = maxLen - obj2.begin
          }
          rowLayers[obj1.row - 1].push(tuple.layerObj1)
          rowLayers[obj2.row - 1].push(tuple.layerObj2)
        }
      }
      for (let i = 0; i < rowLayers.length; i++) {
        rowLayers[i].sort((a, b) => {
          return a.len - b.len
        })
        for (let j = 0; j < rowLayers[i].length; j++) {
          const layer1 = rowLayers[i][j]
          for (let k = 0; k < j; k++) {
            const layer2 = rowLayers[i][k]
            if (layer1.left < layer2.right && layer1.right > layer2.left) {
              layer1.layer = Math.max(layer1.layer, layer2.layer + 1)
            }
          }
        }
      }
    },

    _handleTagShowChange() {
      this.entityTagShow = this.tagShowCheck
      this.$nextTick(() => {
        if (this.entityTagShow) {
          this._updateTagPositions()
          if (this.lineShowCheck) {
            this.tupleLineShow = true
            this.$nextTick(() => {
              this._updateLinkLinePositions()
            })
          }
        } else {
          this.tupleLineShow = false
          for (let i = 0; i < this.text.length; i++) {
            document.getElementById('w_' + i).style.marginTop = '0px'
          }
        }
      })
    },

    _handleLineShowChange() {
      this.tupleLineShow = this.lineShowCheck
      this.$nextTick(() => {
        if (this.tupleLineShow) {
          this._updateLinkLinePositions()
        } else {
          this._updateTagPositions()
        }
      })
    },

    _handleEditorMouseUp(e) {
      e.preventDefault()
      const selection = window.getSelection()
      if (selection.type === 'Range') {
        const p1 = parseInt(selection.anchorNode.parentNode.id.substring(2))
        const p2 = parseInt(selection.focusNode.parentNode.id.substring(2))
        const begin = Math.min(p1, p2)
        const end = Math.max(p1, p2)
        this.$emit('select-content', {
          begin: begin,
          end: end,
          content: this.text.substring(begin, end + 1)
        })
      }
    },

    _handleTagClick(tag, e) {
      this.$emit('tag-click', {
        tag: tag,
        event: e
      })
    },

    _handleTagContextMenu(tag, e) {
      this.$emit('tag-contextmenu', {
        tag: tag,
        event: e
      })
    },

    _handleTagMouseMove(tag, e) {
      const tagObj = document.getElementById('t_' + tag.id + '_' + tag.begin)
      if (!tagObj) { return }
      tagObj.style.border = '1px solid #000'
      tagObj.style.fontWeight = 'bold'
      for (let i = tag.begin; i <= tag.end; i++) {
        document.getElementById('w_' + i).style.fontWeight = 'bold'
        document.getElementById('w_' + i).style.textDecoration = 'underline'
      }
      this.$emit('tag-mousemove', { tag: tag, event: e })
    },

    _handleTagMouseLeave(tag, e) {
      const tagObj = document.getElementById('t_' + tag.id + '_' + tag.begin)
      if (!tagObj) { return }
      tagObj.style.border = '1px solid #fff'
      tagObj.style.fontWeight = 'normal'
      for (let i = tag.begin; i <= tag.end; i++) {
        document.getElementById('w_' + i).style.fontWeight = 'normal'
        document.getElementById('w_' + i).style.textDecoration = 'none'
      }
      this.$emit('tag-mouseleave', { tag: tag, event: e })
    },

    _handleTagMouseDown(tag, e) {
      const tagObj = document.getElementById('t_' + tag.id + '_' + tag.begin)
      if (!tagObj) { return }
      this.$emit('tag-mousedown', { tag: tag, event: e })
    },

    _handleTagMouseUp(tag, e) {
      const tagObj = document.getElementById('t_' + tag.id + '_' + tag.begin)
      if (!tagObj) { return }
      this.$emit('tag-mouseup', { tag: tag, event: e })
    },

    _handleLineMouseMove(line, e) {
      const ev = new MouseEvent('mousemove', e)
      const obj1 = document.getElementById('t_' + line.subjectId + '_' + this.entityMap[line.subjectId].location[0].begin)
      const obj2 = document.getElementById('t_' + line.objectId + '_' + this.entityMap[line.objectId].location[0].begin)
      if (!obj1 || !obj2) { return }
      obj1.dispatchEvent(ev)
      obj2.dispatchEvent(ev)
    },

    _handleLineMouseLeave(line, e) {
      const ev = new MouseEvent('mouseleave', e)
      const obj1 = document.getElementById('t_' + line.subjectId + '_' + this.entityMap[line.subjectId].location[0].begin)
      const obj2 = document.getElementById('t_' + line.objectId + '_' + this.entityMap[line.objectId].location[0].begin)
      if (!obj1 || !obj2) { return }
      obj1.dispatchEvent(ev)
      obj2.dispatchEvent(ev)
    },

    _handleLineClick(line, e) {
      this.$emit('line-click', { tupleId: line.id, event: e })
    },

    _handleLineContextMenu(line, e) {
      this.$emit('line-contextmenu', { tupleId: line.id, event: e })
    }

  }

}
</script>

<style scoped>
.editor {
    width: 100%;
    height: 100%;
    /* border: 2px #b2b2b2 solid; */
    display: flex;
    flex-direction: column;
    overflow-y: auto;
    overflow-x: hidden;
}
.menu {
    border-bottom: #b2b2b2 1px solid;
    margin-bottom: 1px;
    text-align: left;
    padding: 2px 10px;
}
.editor-main {
    padding: 10px 35px;
    text-align: left;
    justify-content: space-between;
    position: relative;
    max-width: 800px;
    margin: 0 auto;
    font-size: 18px;
    line-height: 1.5;
}
.word {
    display: inline-block;
    vertical-align: bottom;
}
.tag-container, .link-line-container {
    position: absolute;
    z-index: 1;
    left: 0;
    top: 0;
    line-height: 1.3;
}
.tag {
    position: absolute;
    display: inline-block;
    padding: 1px;
    font-size: 12px;
    text-align: center;
    border: 1px #fff solid;
    white-space: nowrap;
    color: #fff;
    user-select: none;
}
</style>
