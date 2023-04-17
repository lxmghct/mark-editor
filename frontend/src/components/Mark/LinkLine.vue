<!--
 * @FileDescription: 三元组连接线
 -->
<template>
  <div style="position: absolute">

    <!-- 三元组连接线 -->
    <svg
      version="1.1"
      xmlns="http://www.w3.org/2000/svg"
      :width="maxWidth"
      :height="maxHeight"
      style="position:absolute; top:0; left:0; pointer-events:none;"
    >

      <!-- 连线末端箭头 -->
      <defs>
        <marker
          id="end-arrow"
          fill="#000000"
          refX="5"
          refY="2"
          markerWidth="5"
          markerHeight="4"
          orient="auto"
        >
          <polygon points="0 0, 5 2, 0 4" />
        </marker>
      </defs>

      <!-- 连线轨迹 -->
      <path
        :d="svgPath"
        fill="none"
        :stroke-width="lineWidth"
        stroke="black"
        marker-end="url(#end-arrow)"
      />

    </svg>
    <div
      class="line-info"
      @mousemove="_handleMouseMove"
      @mouseleave="_handleMouseLeave"
      @click="_handleClick"
      @contextmenu="_handleContextMenu"
    >

      <!-- 位于连线上的标签, 若连线不跨行, 则只有一个标签 -->
      <div class="line-text" :style="textStyle1">{{ text }}</div>
      <div class="line-text" :style="textStyle2">{{ text }}</div>

      <!-- 连线跨行时位于连线末端的标签, 若连线不跨行, 则隐藏 -->
      <div class="line-tag" :style="tagStyle1">【{{ tagIndex }}】</div>
      <div class="line-tag" :style="tagStyle2">【{{ tagIndex }}】</div>
    </div>

  </div>
</template>

<script>

export default {

  name: 'LinkLine',

  data() {
    return {
      text: '',
      svgPath: '', // 连线轨迹
      maxHeight: '1000px', // 绘图区最大高度
      maxWidth: '1000px', // 绘图区最大宽度
      // 连线上文字和数字标签样式
      textStyle1: { display: 'block' },
      textStyle2: { display: 'none' },
      tagStyle1: { display: 'none' },
      tagStyle2: { display: 'none' },
      lineWidth: 1, // 连线宽度
      tagIndex: 0 // 连线标签索引
    }
  },

  created() {
    try {
      this.tagIndex = document.getElementsByClassName('line-info').length
    } catch (e) {
      this.tagIndex = 0
    }
  },

  methods: {

    /**
     * @description: 生成连线
     * @param {object} lineValue
     * @return {void}
     */
    createLine(lineValue) {
      this.text = lineValue.text
      try {
        this._getSvgPath(lineValue)
      } catch (e) {
        console.log(e)
      }
    },

    _getSvgPath(value) {
      // 计算连线轨迹
      const startObj = value.obj1
      const endObj = value.obj2
      let x1, y1, x2, y2, x3, y3, x4, y4
      const isAcrossLine = Math.abs(startObj.offsetTop - endObj.offsetTop) > 10
      const tempFlag = (isAcrossLine && startObj.offsetTop < endObj.offsetTop) || (!isAcrossLine && startObj.offsetLeft < endObj.offsetLeft)
      if (tempFlag) {
        x1 = startObj.offsetLeft + startObj.clientWidth
        y1 = startObj.offsetTop
        x4 = endObj.offsetLeft
        y4 = endObj.offsetTop
      } else {
        x1 = startObj.offsetLeft
        y1 = startObj.offsetTop
        x4 = endObj.offsetLeft + endObj.clientWidth
        y4 = endObj.offsetTop
      }
      this.maxWidth = Math.max(x1, x4, value.right) - value.left + 100 + 'px'
      this.maxHeight = Math.max(y1, y4) + 100 + 'px'
      if (isAcrossLine || (!isAcrossLine && Math.abs(x1 - x4) > 40)) {
        x2 = x1 + (tempFlag ? 20 : -20)
        x3 = x4 + (tempFlag ? -20 : 20)
      } else {
        x2 = (x1 + x4) / 2
        x3 = x2
      }
      y2 = y1 - value.height1
      y3 = y4 - value.height2
      let tempStr = ''
      let t1, t2
      if (isAcrossLine) {
        t1 = startObj.offsetTop < endObj.offsetTop ? value.right : value.left
        t2 = startObj.offsetTop < endObj.offsetTop ? value.left : value.right
        tempStr = `L${t1} ${y2} M${t2} ${y3}`
      }
      this.svgPath = `M${x1} ${y1} L${x2} ${y2} ` + tempStr + ` L${x3} ${y3} L${x4} ${y4}`
      // 设置标签位置
      if (isAcrossLine) {
        this.textStyle1 = {
          display: 'block',
          left: (x2 + t1) / 2 + 'px',
          top: y2 - 8 + 'px'
        }
        this.textStyle2 = {
          display: 'block',
          left: (x3 + t2) / 2 + 'px',
          top: y3 - 8 + 'px'
        }
        this.tagStyle1 = {
          display: 'block',
          left: t1 + 'px',
          top: y2 - 8 + 'px'
        }
        this.tagStyle2 = {
          display: 'block',
          left: t2 + 'px',
          top: y3 - 8 + 'px'
        }
      } else {
        this.textStyle2.display = 'none'
        this.tagStyle1.display = 'none'
        this.tagStyle2.display = 'none'
        this.textStyle1 = {
          display: 'block',
          left: (startObj.offsetLeft + endObj.offsetLeft) / 2 + 'px',
          top: (y2 + y3) / 2 - 8 + 'px'
        }
      }
    },

    _handleMouseMove(e) {
      this.lineWidth = 2
      this.textStyle1.fontWeight = 'bold'
      this.textStyle2.fontWeight = 'bold'
      this.tagStyle1.fontWeight = 'bold'
      this.tagStyle2.fontWeight = 'bold'
      this.textStyle1.zIndex = 1
      this.textStyle2.zIndex = 1
      this.tagStyle1.zIndex = 1
      this.tagStyle2.zIndex = 1
      this.$emit('line-mousemove', e)
    },

    _handleMouseLeave(e) {
      this.lineWidth = 1
      this.textStyle1.fontWeight = 'normal'
      this.textStyle2.fontWeight = 'normal'
      this.tagStyle1.fontWeight = 'normal'
      this.tagStyle2.fontWeight = 'normal'
      this.textStyle1.zIndex = 0
      this.textStyle2.zIndex = 0
      this.tagStyle1.zIndex = 0
      this.tagStyle2.zIndex = 0
      this.$emit('line-mouseleave', e)
    },

    _handleClick(e) {
      this.$emit('line-click', e)
    },

    _handleContextMenu(e) {
      this.$emit('line-contextmenu', e)
    }
  }
}
</script>

<style scoped>
.line-text {
    position: absolute;
    white-space: nowrap;
    font-size: 5px;
    color: #000;
    background-color: white;
    user-select: none;
    z-index: 0;
    cursor: pointer;
}
.line-tag {
    position: absolute;
    white-space: nowrap;
    font-size: 5px;
    color: #000;
    background-color: white;
    user-select: none;
    z-index: 0;
    cursor: pointer;
}
.line-info {
    position: absolute;
}
</style>
