<template>
  <div ref="main" style="height: 99%">
    <el-tabs v-model="tabs">
      <el-tab-pane label="当前文档" name="0" />
      <el-tab-pane label="全部" name="1" />
    </el-tabs>
    <el-input v-model="searchContent" placeholder="请输入实体名或关系名" size="mini" @keyup.enter.native="handleSearch">
      <el-button slot="append" icon="el-icon-search" @click="handleSearch" />
    </el-input>
    <el-table
      v-loading="currentData.loading"
      :data="currentData.relationList"
      size="mini"
      :max-height="tableHeight"
      @row-contextmenu="handleTableRowRightClick"
      @row-click="handleTableRowClick"
    >

      <el-table-column label="序号" width="55">
        <template slot-scope="scope">
          {{ scope.$index+1 }}
        </template>
      </el-table-column>

      <el-table-column label="主语">
        <template slot-scope="scope">
          <span class="text-overflow" :title="scope.row.subject.name">
            {{ scope.row.subject.name }}
          </span>
        </template>
      </el-table-column>

      <el-table-column prop="name" label="关系名" width="80" />

      <el-table-column label="宾语">
        <template slot-scope="scope">
          <span class="text-overflow" :title="scope.row.object.name">
            {{ scope.row.object.name }}
          </span>
        </template>
      </el-table-column>

      <el-table-column label="操作" width="55">
        <template slot-scope="scope">
          <el-button
            type="text"
            size="small"
            @click.native.prevent="handleClickOperation(scope.row)"
          >
            {{ operationText }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      :current-page="currentData.pageNum"
      :page-sizes="[5, 10, 20, 40]"
      :page-size="currentData.pageSize"
      :pager-count="5"
      small
      layout="total, sizes, pager, jumper"
      :total="currentData.totalNum"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
    />
  </div>
</template>

<script>
export default {
  data() {
    return {
      tabs: '0',
      searchContent: '',
      currentDocument: '',
      localRelationData: {
        allRelations: [],
        relationList: [],
        totalNum: 0,
        pageNum: 1,
        pageSize: 10,
        loading: false
      },
      allRelationData: {
        relationList: [],
        totalNum: 0,
        pageNum: 1,
        pageSize: 10,
        loading: false
      },
      currentData: {
        relationList: [],
        totalNum: 0,
        pageNum: 1,
        pageSize: 10,
        loading: false
      },
      operationText: '删除',

      tableHeight: 300
    }
  },

  watch: {

    tabs(val) {
      if (val === '0') {
        this.currentData = this.localRelationData
      } else {
        this.currentData = this.allRelationData
      }
      this.$forceUpdate()
    }

  },

  mounted() {
    this.updateAllRelations()
    this.currentData = this.localRelationData
    setTimeout(() => { this.updateTableHeight() })
  },

  methods: {

    changeDocument(document) {
      if (this.currentDocument === document) { return }
      this.currentDocument = document
      const params = new URLSearchParams()
      params.append('documentLocation', document)
      this.localRelationData.loading = true
      this.$http.post('/neo4j/entity/getRelationByDocumentLocation', params).then(res => {
        if (res.data.status === 200) {
          this.setRelationList(res.data.data)
        }
      }).finally(() => {
        this.localRelationData.loading = false
      })
    },

    setRelationList(data) {
      this.localRelationData.allRelations = data
      this.localRelationData.totalNum = data.length
      this.localRelationData.pageNum = 1
      this.localRelationData.relationList = this.localRelationData.allRelations.slice(0, this.localRelationData.pageSize)
    },

    updateData(data) {
      this.setRelationList(data)
      this.allRelationData.pageNum = 1
      this.updateAllRelations()
    },

    updateAllRelations() {
      this.getAllRelations()
      this.getAllRelationNum()
    },

    setOperationText(text) {
      this.operationText = text
    },

    setTableHeight(height) {
      this.tableHeight = height
    },

    getAllRelations() {
      let porjectId = this.$route.query.projectId
      if (!porjectId) {
        this.allRelationData.relationList = []
      }
      this.allRelationData.loading = true
      const params = new URLSearchParams()
      params.append('projectId', porjectId)
      params.append('searchContent', this.searchContent)
      params.append('start', (this.allRelationData.pageNum - 1) * this.allRelationData.pageSize)
      params.append('length', this.allRelationData.pageSize)
      this.$http.post('/neo4j/relationship/getRelationSearchResultOfProject', params).then(res => {
        if (res.data.status === 200) {
          this.allRelationData.relationList = res.data.data
        }
      }).finally(() => {
        this.allRelationData.loading = false
      })
    },

    getAllRelationNum() {
      let porjectId = this.$route.query.projectId
      if (!porjectId) {
        this.allRelationData.totalNum = 0
      }
      const params = new URLSearchParams()
      params.append('projectId', porjectId)
      params.append('searchContent', this.searchContent)
      this.$http.post('/neo4j/relationship/getRelationSearchResultNumOfProject', params).then(res => {
        if (res.data.status === 200) {
          this.allRelationData.totalNum = res.data.data
        }
      })
    },

    handleSizeChange(val) {
      this.currentData.pageSize = val
      if (this.tabs === '0') {
        this.localRelationData.relationList = this.localRelationData.allRelations.slice((this.localRelationData.pageNum - 1) * this.localRelationData.pageSize, this.localRelationData.pageNum * this.localRelationData.pageSize)
      } else {
        this.updateAllRelations()
      }
    },

    handleCurrentChange(val) {
      this.currentData.pageNum = val
      if (this.tabs === '0') {
        this.localRelationData.relationList = this.localRelationData.allRelations.slice((this.localRelationData.pageNum - 1) * this.localRelationData.pageSize, this.localRelationData.pageNum * this.localRelationData.pageSize)
      } else {
        this.updateAllRelations()
      }
    },

    handleSearch() {
      if (this.tabs === '0') {
        this.localRelationData.relationList = this.localRelationData.allRelations.filter(item => {
          return item.name.indexOf(this.searchContent) !== -1 || item.subject.name.indexOf(this.searchContent) !== -1 || item.object.name.indexOf(this.searchContent) !== -1
        })
        this.localRelationData.totalNum = this.localRelationData.relationList.length
        this.localRelationData.pageNum = 1
        this.localRelationData.relationList = this.localRelationData.relationList.slice(0, this.localRelationData.pageSize)
      } else {
        this.updateAllRelations()
      }
    },

    updateTableHeight() {
      try {
        this.tableHeight = this.$refs.main.parentNode.offsetHeight - 150
      } catch (e) {
        this.tableHeight = 300
      }
    },

    handleTableRowClick(row) {
      this.$emit('table-row-click', row)
    },

    handleTableRowRightClick(row) {
      this.$emit('table-row-right-click', row)
    },

    handleClickOperation(row) {
      this.$emit('click-operation', row)
    }

  }

}
</script>

<style scoped>
.text-overflow {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
</style>
