<!--
 * @FileDescription: 文件树
-->
<template>
  <div>
    <!-- 搜索框 -->
    <el-input v-model="searchContent" placeholder="请输入内容" size="mini">
      <el-button slot="append" icon="el-icon-search" />
    </el-input>

    <!-- 文件树 -->
    <el-tree
      ref="fileTree"
      :data="fileTreeData"
      node-key="label"
      default-expand-all
      class="filter-tree"
      style="margin-top: 20px; height: 100%"
      :filter-node-method="filterNode"
      :highlight-current="true"

      @node-expand="(data) => {data.iconType = 'el-icon-folder-opened'}"
      @node-collapse="(data) => {data.iconType = 'el-icon-folder'}"
      @node-click="handleFileTreeNodeClick"
    >
      <span slot-scope="{ data }">
        <i :class="data.iconType" />
        <span class="file-tree-text">
          <span
            v-if="data.iconType == 'el-icon-document' || data.label == '工作区'"
            :class="data.iconType == 'el-icon-document' ? 'file-tree-score' : 'file-tree-score-total'"
          >
            <span class="file-tree-effectivetuple-count" :title="`有效三元组数:${data.effectiveTupleCount}`">
              {{ data.effectiveTupleCount }}
            </span>
            /
            <span class="file-tree-tuple-count" :title="`三元组总数:${data.tupleCount}`">
              {{ data.tupleCount }}
            </span>
          </span>
          {{ data.label }}
        </span>
      </span>
    </el-tree>
  </div>
</template>

<script>
export default {

  name: 'FileTree',

  data() {
    return {
      fileTreeData: [{
        label: '工作区',
        children: [],
        iconType: 'el-icon-folder-opened',
        effectiveTupleCount: 0,
        tupleCount: 0
      }], // 文件树的数据
      searchContent: '', // 搜索内容
      issueId: '', // 任务id
      entityTypes: ['', '人物', '地点', '知识点', '事件', '机构', '职官', '文献', '时间', '其他']
    }
  },

  watch: {
    searchContent(val) {
      this.$refs.fileTree.filter(val)
    }
  },

  mounted() {
    if (this.$route.query.issueId) {
      this.initData(this.$route.query.issueId)
    }
  },

  methods: {

    /**
     * @description: 文件树过滤器
     * @param {String} value 搜索内容
     * @param {Object} data 节点数据
     * @return {Boolean} 是否对当前节点进行显示
     */
    filterNode(value, data) {
      if (!value) return true
      return data.label.indexOf(value) !== -1 || data.label === '工作区'
    },

    /**
     * @description: 文件树点击事件，切换文件
     * @param {Object} data 被点击的节点
     * @return void
     */
    handleFileTreeNodeClick(data) {
      if (data.iconType !== 'el-icon-document') { return }
      this.getFileData(data).then(() => {
        if (data.name.substr(0, 3) === 'ybz') {
          this.getPreMarkData(data)
        } else {
          this.$emit('changeCurrentFile', {
            fileData: data,
            type: 'change'
          })
        }
      }).catch(() => {
        this.$message.error('获取文件数据失败')
      }).finally(() => {
        this.$emit('loading-event', { loading: false })
      })
    },

    /**
     * @description: 根据节点名获取存在的节点节点, 若无则返回null
     * @param {Object} node 节点
     * @param {String} name 节点名
     * @return {Object} 节点
     */
    getExistedNode(node, name) {
      if (node.label === name) {
        return node
      }
      if (node.children) {
        for (let i = 0; i < node.children.length; i++) {
          const res = this.getExistedNode(node.children[i], name)
          if (res) {
            return res
          }
        }
      }
      return null
    },

    /**
     * @description: 根据节点名获取对应节点, 若无则创建并返回
     * @param {Object} node 节点
     * @param {String} name 节点名
     * @return void
     */
    getNode(node, nodeName) {
      const res = this.getExistedNode(node, nodeName)
      if (res) {
        return res
      } else {
        const res1 = {
          label: nodeName,
          iconType: 'el-icon-folder-opened'
        }
        if (!node.children) { node.children = [] }
        node.children.push(res1)
        return res1
      }
    },

    getNodeByFileLocation(parentNode, fileLocation) {
      // 用一个或多个斜杠分隔fileLocation
      const paths = fileLocation.split(/\/+/)
      let node = parentNode
      for (let i = 0; i < paths.length; i++) {
        node = this.getNode(node, paths[i])
      }
      return node
    },

    /**
     * @description: 获取文档内容
     * @param {Object} node 节点
     * @return void
     */
    async getFileData(node) {
      if (node.content) {
        return node.content
      }
      this.$emit('loading-event', {
        loading: true,
        loadingText: '正在加载文档内容'
      })
      const params = new URLSearchParams()
      params.append('fileLocations', node.documentLocation)
      const res = await this.$http.post('/neo4j/project/loadFile', params)
      if (res.data.status === 200) {
        node.content = res.data.data[0].data
        return node.content
      }
      this.$emit('loading-event', { loading: false })
    },

    getPreMarkData(node) {
      this.$emit('loading-event', {
        loading: true,
        loadingText: '正在加载预标注数据'
      })
      const filePath = node.documentLocation
      const params = new URLSearchParams()
      params.append('fileLocations', filePath.replace('.txt', '.json'))
      this.$http.post('/neo4j/project/loadFile', params).then(response => {
        if (response.data.status === 200) {
          const entityJSON = {}
          const nameMap = {}
          let entityNum = 0
          try {
            JSON.parse(response.data.data[0].data).forEach(en => {
              const id = nameMap[en.name]
              if (id) {
                entityJSON[id].beginLocation.push(en.beginLocation + '')
                entityJSON[id].endLocation.push(en.endLocation + '')
                entityJSON[id].documentLocation.push(filePath)
              } else {
                const tempId = ++entityNum
                entityJSON[tempId] = {
                  name: en.name,
                  type: en.type ? this.entityTypes[en.type] : en.typename,
                  beginLocation: [en.beginLocation + ''],
                  endLocation: [en.endLocation + ''],
                  documentLocation: [filePath],
                  nodeId: tempId,
                  lineNo: en.lineNo,
                  wordNo: en.wordNo,
                  annotation: ''
                }
              }
            })
          } catch (error) {
            console.log(error)
          }
          for (const key in entityJSON) {
            const e = entityJSON[key]
            this.$http.get(`http://58.250.74.99:10235/Api/QueryByCondition?knowType=0&knowName=${e.name}`).then(res => {
              if (res.data.queryResult === 1) {
                res.data.data.forEach(item => {
                  e.annotation += item.description + '\n出处 : ' + item.addFrom + '\n'
                })
              }
            })
          }
          setTimeout(() => {
            this.$emit('changeCurrentFile', {
              type: 'preMark',
              fileData: node,
              entityJSON: entityJSON
            })
          })
        }
      }).finally(() => {
        this.$emit('loading-event', {
          loading: false
        })
      })
    },

    /**
     * @description: 递归获取文件路径
     * @param {Object} node 当前节点
     * @param {String} fileName 文件名
     * @return void
     */
    getFileLocationByNode(node, fileName) {
      if (node.label === fileName) {
        return fileName
      }
      if (node.children) {
        for (let i = 0; i < node.children.length; i++) {
          const res = this.getFileLocationByNode(node.children[i], fileName)
          if (res) {
            return node.label + '/' + res
          }
        }
      }
      return null
    },

    /**
     * @description: 获取文件路径
     * @param {String} fileName 文件名
     * @return void
     */
    getFileLocation(fileName) {
      const str = this.getFileLocationByNode(this.fileTreeData[0], fileName).substr(4)
      return str
    },

    /**
     * @description: 初始化数据
     * @param {String} issueId 任务id
     * @return void
     */
    initData(issueId) {
      this.issueId = issueId
      const params = new URLSearchParams()
      params.append('issueId', issueId)
      this.$http.post('/neo4j/project/getDocumentsByIssue', params).then(res => {
        if (res.data.status === 200) {
          res.data.data.forEach(item => {
            const node = this.getNodeByFileLocation(this.fileTreeData[0], item.fileLocation)
            node.iconType = 'el-icon-document'
            node.name = item.name
            node.nodeId = item.nodeId
            node.content = ''
            node.effectiveTupleCount = 0
            node.tupleCount = 0
            node.documentLocation = item.fileLocation
          })
          this.updateTupleCounts()
        }
      })
    },

    changeFile(fileName) {
      const node = this.getNode(this.fileTreeData[0], fileName)
      this.$emit('changeCurrentFile', {
        fileData: node,
        type: 'change'
      })
    },

    setTupleCount(documentList, total) {
      for (let i = 0; i < documentList.length; i++) {
        const node = this.getNodeByFileLocation(this.fileTreeData[0], documentList[i].fileLocation)
        node.effectiveTupleCount = documentList[i].effectiveTupleCount
        node.tupleCount = documentList[i].tupleCount
      }
      this.fileTreeData[0].effectiveTupleCount = total.effectiveTupleCount
      this.fileTreeData[0].tupleCount = total.tupleCount
    },

    getDocumentList() {
      const projectRoot = this.fileTreeData[0].children[0]
      const rootName = projectRoot.label
      const documentList = []
      for (let i = 0; i < projectRoot.children.length; i++) {
        documentList.push({
          fileLocation: rootName + '/' + projectRoot.children[i].label,
          effectiveTupleCount: projectRoot.children[i].effectiveTupleCount,
          tupleCount: projectRoot.children[i].tupleCount
        })
      }
      return documentList
    },

    updateTupleCounts() {
      const fileList = this.fileTreeData[0].children[0].children
      if (fileList.length === 0) { return }
      // 防止一次性发过多请求
      for (let i = 0; i < fileList.length; i++) {
        const file = fileList[i]
        if (!file.documentLocation) { continue }
        setTimeout(() => {
          this.updateTupleCountOfDocument(file, file.documentLocation)
        }, i / 5 * 1000)
      }
    },

    updateTupleCountOfDocument(node, documentLocation) {
      const params = new URLSearchParams()
      params.append('documentLocation', documentLocation)
      this.$http.post('/neo4j/relationship/getRelationCountByDocumentLocation', params).then(res => {
        if (res.data.status === 200) {
          try {
            node.tupleCount = res.data.data.relationCount
            node.effectiveTupleCount = res.data.data.effectiveRelationCount
          } catch (error) {
            console.log(error)
          }
        }
        // 计算总数
        this.fileTreeData[0].tupleCount = 0
        this.fileTreeData[0].effectiveTupleCount = 0
        for (const file of this.fileTreeData[0].children[0].children) {
          this.fileTreeData[0].tupleCount += file.tupleCount ? file.tupleCount : 0
          this.fileTreeData[0].effectiveTupleCount += file.effectiveTupleCount ? file.effectiveTupleCount : 0
        }
      })
    }

  }

}

</script>

<style>
.el-tree {
    width: 100%;
    overflow: auto;
}
.el-tree>.el-tree-node {
    display: inline-block;
    min-width: 100%;
}
.el-tree--highlight-current .el-tree-node.is-current > .el-tree-node__content {
    background-color: #dce8fa !important;
}
</style>

<style scoped>
.file-tree-text {
    display: inline-block;
    position: relative;
    margin-left: 5px;
}
.file-tree-score {
    position: absolute;
    display: inline-block;
    left: -80px;
    font-size: 14px;
    font-weight: bold;
}
.file-tree-score-total {
    position: absolute;
    display: inline-block;
    left: 60px;
    top: 2px;
    font-size: 14px;
    font-weight: bold;

}
.file-tree-effectivetuple-count {
    cursor: pointer;
    color: #1f8ffe;
}
.file-tree-tuple-count {
    cursor: pointer;
    color: #ea8817;
}
</style>
