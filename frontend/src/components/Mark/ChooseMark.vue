<template>
  <div class="choose-mark-main" :style="windowStyle">
    <div class="header" @mousedown="handleMouseDown" @mousemove="handleMouseMove" @mouseup="handleMouseUp" @mouseleave="handleMouseLeave">
      实体选择标注
      <!-- 关闭按钮 -->
      <el-button
        style="float:right"
        size="mini"
        type="danger"
        icon="el-icon-close"
        circle
        @click="show(false)"
      />
    </div>
    <div style="padding: 10px; border-bottom: #000 1px dashed;">
      <span>实体名: {{ currentEntity.name }} </span>
      <span style="margin:0 20px">实体类型: {{ currentEntity.type }} </span>
      <span style="margin-left: 20px; color:cornflowerblue">标注:
        <span :style="'display: inline-block; width:20px; height:20px; background:'+markColors[1]">&nbsp;</span>
      </span>
      <span style="margin-left: 20px; color:cornflowerblue">忽略:
        <span :style="'display: inline-block; width:20px; height:20px; background:'+markColors[0]">&nbsp;</span>
      </span>
      <el-button style="float:right;margin:0 3px" type="success" size="mini" @click="handleSaveMark">保存</el-button>
      <el-button style="float:right;margin:0 3px" type="danger" size="mini" @click="handleRemoveAll">全部删除</el-button>
      <el-button style="float:right;margin:0 3px" type="primary" size="mini" @click="handleReserveAll">全部保留</el-button>
    </div>
    <div ref="chooseMarkEditor" class="choose-mark-editor" />
  </div>
</template>

<script>
import { getPointX, getPointY } from '@/assets/js/index'
export default {
  name: 'ChooseMark',
  data() {
    return {
      windowStyle: { left: '500px', top: '100px' }, // 窗口位置
      isMouseDown: false,
      oldMousePoint: { x: 0, y: 0 },
      oldWindowPoint: { x: 0, y: 0 },
      currentEntity: {},
      currentFile: {
        name: '',
        content: ''
      },
      documentLocation: '',
      beginLocations: [],
      enables: [],
      markColors: ['rgba(250,183,40,0.4)', 'rgba(249,91,24,0.5)']
    }
  },

  mounted() {
    this.show(false)
  },
  methods: {

    handleMouseDown(e) {
      if (e.offsetX > 700) { return }
      this.isMouseDown = true
      this.oldMousePoint = { x: e.clientX, y: e.clientY }
      const obj = document.getElementsByClassName('choose-mark-main')[0]
      this.oldWindowPoint = { x: getPointX(obj), y: getPointY(obj) }
    },

    handleMouseMove(e) {
      if (e.offsetX > 700) { return }
      if (this.isMouseDown) {
        this.windowStyle.left = this.oldWindowPoint.x + e.clientX - this.oldMousePoint.x + 'px'
        this.windowStyle.top = this.oldWindowPoint.y + e.clientY - this.oldMousePoint.y + 'px'
      }
    },

    handleMouseUp() {
      this.isMouseDown = false
    },

    handleMouseLeave() {
      this.isMouseDown = false
    },

    setValue(file, documentLocation, entity) {
      this.currentFile = file
      this.documentLocation = documentLocation
      this.currentEntity = entity
    },

    show(isShow = true) {
      this.isShow = isShow
      const obj = document.getElementsByClassName('choose-mark-main')[0]
      if (isShow) {
        obj.style.display = 'block'
        this.init()
      } else {
        obj.style.display = 'none'
      }
    },

    init() {
      this.enables = []
      this.beginLocations = []
      let str = this.currentFile.content
      const name = this.currentEntity.name
      if (!str || !name) { return }
      let index = str.indexOf(name)
      const len = str.length
      while (index !== -1 && index < len) {
        this.beginLocations.push(index)
        index = str.indexOf(name, index + name.length)
      }
      for (let i = this.beginLocations.length - 1; i >= 0; i--) {
        const index = this.beginLocations[i]
        const markStr = `<span id=choose${index} style="cursor:pointer">${name}</span>`
        str = str.substring(0, index) + markStr + str.substring(index + name.length)
      }
      this.$refs.chooseMarkEditor.innerHTML = str
      for (let i = 0; i < this.beginLocations.length; i++) {
        const obj = document.getElementById(`choose${this.beginLocations[i]}`)
        const t = this.currentEntity.beginLocation.indexOf(this.beginLocations[i] + '')
        if (t === -1) {
          this.enables.push(false)
          obj.style.backgroundColor = this.markColors[0]
        } else {
          this.enables.push(true)
          obj.style.backgroundColor = this.markColors[1]
        }
        const that = this
        obj.onclick = function() {
          that.enables[i] = !that.enables[i]
          this.style.backgroundColor = that.enables[i] ? that.markColors[1] : that.markColors[0]
        }
        obj.onmouseover = function() {
          this.style.fontWeight = 'bold'
          this.style.textDecoration = 'underline'
        }
        obj.onmouseleave = function() {
          this.style.fontWeight = 'normal'
          this.style.textDecoration = 'none'
        }
      }
    },

    handleRemoveAll() {
      for (let i = 0; i < this.enables.length; i++) {
        if (this.enables[i]) {
          this.enables[i] = false
          const obj = document.getElementById(`choose${this.beginLocations[i]}`)
          obj.style.backgroundColor = this.markColors[0]
        }
      }
    },

    handleReserveAll() {
      for (let i = 0; i < this.enables.length; i++) {
        if (!this.enables[i]) {
          this.enables[i] = true
          const obj = document.getElementById(`choose${this.beginLocations[i]}`)
          obj.style.backgroundColor = this.markColors[1]
        }
      }
    },

    handleSaveMark() {
      const locations = []
      for (let i = 0; i < this.enables.length; i++) {
        if (this.enables[i]) {
          locations.push(this.beginLocations[i])
        }
      }
      this.modifyEntityLocation(this.currentEntity.nodeId, this.documentLocation, locations)
      this.show(false)
    },

    modifyEntityLocation(nodeId, documentLocation, beginLocations) {
      const params = new URLSearchParams()
      params.append('nodeId', nodeId)
      params.append('documentLocation', documentLocation)
      params.append('beginLocations', beginLocations)
      this.$http.post('/neo4j/entity/modifyEntityLocation', params).then(res => {
        if (res.data.status === 200) {
          this.$message.success('保存成功')
          this.$emit('markEvent', {
            nodeId: nodeId,
            documentLocation: documentLocation,
            entity: res.data.data
          })
        } else {
          this.$message.error('保存失败')
        }
      })
    }
  }
}
</script>

<style scoped>
.choose-mark-main {
    position: absolute;
    width: 800px;
    z-index: 10;
    background: #fff;
    border: 2px solid #000;
}
.header {
    font-size: 20px;
    text-align: left;
    padding: 8px;
    border-bottom: #000 solid 1px;
}
.choose-mark-editor {
    padding: 15px;
    font-size: 18px;
    text-align: left;
    height: 450px;
    overflow: auto;
}
</style>
