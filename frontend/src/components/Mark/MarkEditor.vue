<template>
  <div
    ref="main"
    class="mark-editor-main"
    @mousemove="_handleMouseMove"
    @mouseup="_handleMouseUp"
  >
    <editor
      ref="editor"
      @select-content="_handleSelectContent"
      @tag-click="_handleTagClick"
      @tag-contextmenu="_handleTagContextMenu"
      @tag-mousedown="_handleTagMouseDown"
      @tag-mouseup="_handleTagMouseUp"
      @tag-mousemove="_handleTagMouseMove"
      @tag-mouseleave="_handleTagMouseLeave"
      @line-click="_handleLineClick"
      @line-contextmenu="_handleLineContextMenu"
    />
    <svg
      v-show="isSvgShow"
      ref="svg"
      version="1.1"
      xmlns="http://www.w3.org/2000/svg"
      width="100%"
      height="100%"
      style="position:fixed; top:0; left:0; z-index:2; pointer-events:none;"
    >
      <defs>
        <marker
          id="end-arrow"
          fill="#0000FF"
          refX="5"
          refY="2"
          markerWidth="5"
          markerHeight="4"
          orient="auto"
        >
          <polygon points="0 0, 5 2, 0 4" />
        </marker>
      </defs>
      <path
        :d="svgPath"
        fill="none"
        stroke-width="2"
        stroke-dasharray="5 5"
        stroke="blue"
        marker-end="url(#end-arrow)"
      />
    </svg>
    <!-- 实体右键菜单 -->
    <div v-show="entityMenuVisible" id="entity-menu" @mouseleave="entityMenuVisible=false">
      <el-card style="width: 160px">
        <el-link
          icon="el-icon-circle-plus-outline"
          :underline="false"
          @click="_handleEntityMenuClick('add_tuple')"
        >
          添加三元组
        </el-link>
        <el-link
          icon="el-icon-remove-outline"
          :underline="false"
          @click="_handleEntityMenuClick('delete_entity')"
        >
          删除该实体
        </el-link>
        <el-link
          icon="el-icon-remove"
          :underline="false"
          @click="_handleEntityMenuClick('delete_location')"
        >
          删除该位置
        </el-link>
        <!-- <el-link icon="el-icon-star-off" :underline="false"
                @click="_handleEntityMenuClick('choose_mark')">
                全文检索&nbsp;&nbsp;&nbsp;
            </el-link> -->
      </el-card>
    </div>
    <!--三元组连线右键菜单-->
    <div v-show="tupleMenuVisible" id="tuple-menu" @mouseleave="tupleMenuVisible=false">
      <el-card style="width: 160px">
        <!-- <el-link icon="el-icon-edit" :underline="false"
                @click="_handleTupleMenuClick('edit_tuple')">
                编辑三元组
            </el-link> -->
        <el-link
          icon="el-icon-remove-outline"
          :underline="false"
          @click="_handleTupleMenuClick('delete_tuple')"
        >
          删除三元组
        </el-link>
      </el-card>
    </div>
  </div>
</template>

<script>
import Editor from './Editor.vue'
export default {
  components: { Editor },

  data() {
    return {
      filePath: '',
      entityMap: {},
      tupleMap: {},
      isSvgShow: false,
      svgPath: '',
      svgParams: {
        isTouchEntity: false, // 鼠标是否碰到了实体
        selectedObj1: null, // 选中的第一个实体
        selectedObj2: null, // 选中的第二个实体
        isMouseDown: false, // 鼠标是否按下
        svgStartPoint: { x: 0, y: 0 } // svg连线的起始点
      },
      entityMenuVisible: false, // 实体右键菜单
      selectedEntity: null, // 选中的实体
      tupleMenuVisible: false, // 三元组右键菜单
      selectedTuple: null, // 选中的三元组
      editorPoint: { x: 0, y: 0 } // 编辑器的坐标
    }
  },

  methods: {

    /**
     * @description: 设置文本
     * @param {string} text
     * @param {string} filePath
     * @return {void}
     */
    setText(text, filePath) {
      this.filePath = filePath
      this.$refs.editor.setText(text)
    },

    /**
     * @description: 设置colorMap
     * @param {object} colorMap
     * @param {string} field
     * @return {void}
     */
    setColorMap(colorMap, field = 'type') {
      this.$refs.editor.setColorMap(colorMap, field)
    },

    /**
     * @description: 批量标注实体
     * @param {object} entityMap
     * @return {void}
     */
    markEntities(entityMap) {
      this.entityMap = entityMap
      this.$refs.editor.markEntities(entityMap)
    },

    /**
     * @description: 批量标注三元组
     * @param {object} tupleMap
     * @return {void}
     */
    markTuples(tupleMap) {
      this.tupleMap = tupleMap
      this.$refs.editor.markTuples(tupleMap)
    },

    /**
     * @description: 刷新editor
     * @return {void}
     */
    updateAll() {
      this.$refs.editor.markEntities(this.entityMap)
      this.$nextTick(() => {
        this.$refs.editor.markTuples(this.tupleMap)
      })
    },

    /**
     * @description: 高亮实体
     * @param {int} entityId
     * @param {begin} begin
     * @return {void}
     */
    highlightEntity(entityId, begin) {
      this.$refs.editor.highlightEntity(entityId, begin)
    },

    _adaptSvg(startX, startY, endX, endY) {
      let x1, y1, x2, y2, x3, y3, x4, y4, radius
      let direction = startX < endX ? 1 : 0
      x1 = startX
      x4 = endX
      let higher = startY < endY ? startY : endY
      y1 = higher - 10
      y4 = higher - 10
      if (Math.abs(endX - startX) >= 28) {
        radius = 14
      } else {
        radius = Math.abs(x4 - x1) / 2
      }
      x2 = x1 + radius * (-1 + direction * 2)
      y2 = y1 - radius
      x3 = x4 - radius * (-1 + direction * 2)
      y3 = y1 - radius
      this.svgPath = `M${startX} ${startY} L${x1} ${y1} A${radius} ${radius} 0 0,${direction} ${x2} ${y2} 
      L${x3} ${y3} A${radius} ${radius} 0 0,${direction} ${x4} ${y4} L${endX} ${endY}`
    },

    _handleMouseMove(e) {
      if (this.svgParams.isMouseDown && this.svgParams.svgStartPoint.x !== -1) {
        this.isSvgShow = true
        this._adaptSvg(this.svgParams.svgStartPoint.x, this.svgParams.svgStartPoint.y, e.clientX, e.clientY)
      }
    },

    _handleMouseUp(event) {
      this.svgParams.isMouseDown = false
      this.svgParams.svgStartPoint.x = -1
      if (this.svgParams.selectedObj1 && this.svgParams.selectedObj2 && this.svgParams.selectedObj1 !== this.svgParams.selectedObj2) {
        let id1 = this.svgParams.selectedObj1.id
        let id2 = this.svgParams.selectedObj2.id
        this.$emit('mark-event', {
          type: 'mark_relation',
          data: {
            subject: parseInt(id1.substring(id1.indexOf('_') + 1, id1.lastIndexOf('_'))),
            object: parseInt(id2.substring(id2.indexOf('_') + 1, id2.lastIndexOf('_')))
          }
        })
      }
      this.svgParams.selectedObj1 = null
      this.svgParams.selectedObj2 = null
      this.isSvgShow = false
    },
    _handleSelectContent(value) {
      this.$emit('select-content', value)
    },
    _handleEntityMenuClick(type) {
      this.entityMenuVisible = false
      this.$emit('mark-event', {
        type: type,
        entity: this.selectedEntity
      })
    },
    _handleTagClick(value) {
      this.selectedEntity = {
        nodeId: value.tag.id,
        type: value.tag.name,
        beginLocation: value.tag.begin,
        endLocation: value.tag.end
      }
      this.$emit('entity-click', this.selectedEntity)
    },
    _handleTagContextMenu(value) {
      this.selectedEntity = {
        nodeId: value.tag.id,
        type: value.tag.name,
        beginLocation: value.tag.begin,
        endLocation: value.tag.end
      }
      this._openContextMenu('#entity-menu')
      this.entityMenuVisible = true
    },
    _handleTagMouseDown(value) {
      this.svgParams.isMouseDown = true
      if (this.svgParams.isTouchEntity) {
        let obj = document.getElementById('t_' + value.tag.id + '_' + value.tag.begin)
        this.svgParams.selectedObj1 = obj
        let rect = obj.getBoundingClientRect()
        this.svgParams.svgStartPoint.x = rect.left + rect.width / 2
        this.svgParams.svgStartPoint.y = rect.top
      }
    },
    _handleTagMouseUp() {
    },
    _handleTagMouseMove(value) {
      this.svgParams.isTouchEntity = true
      if (this.svgParams.isMouseDown && this.svgParams.selectedObj1) {
        let id = 't_' + value.tag.id + '_' + value.tag.begin
        this.svgParams.selectedObj2 = document.getElementById(id)
      }
    },
    _handleTagMouseLeave() {
      this.svgParams.isTouchEntity = false
      this.selectedObj2 = null
      if (!this.svgParams.isMouseDown) {
        this.svgParams.selectedObj1 = null
      }
    },
    _handleLineClick(value) {
      this.$emit('line-click', { relationId: value.tupleId })
    },
    _handleLineContextMenu(value) {
      this.selectedTuple = { relationId: value.tupleId }
      this._openContextMenu('#tuple-menu')
      this.tupleMenuVisible = true
    },
    _handleTupleMenuClick(type) {
      this.tupleMenuVisible = false
      this.$emit('mark-event', {
        type: type,
        data: {
          relationId: this.selectedTuple.relationId
        }
      })
    },
    _openContextMenu(menuId) {
      let menu = document.querySelector(menuId)
      let e = window.event
      e.preventDefault()
      menu.style.cssText = 'position: fixed; left: ' + (e.clientX - 10) + 'px' + '; top: ' + (e.clientY - 25) + 'px; z-index: 999; cursor:pointer;'
    }
  }

}
</script>

<style scoped>
.mark-editor-main {
    position: relative;
    width: 100%;
    height: 100%;
}
</style>
