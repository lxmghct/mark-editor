<template>
  <div ref="main" style="height: 99%">
    <el-tabs v-model="tabs">
      <el-tab-pane label="当前文档" name="0" />
      <el-tab-pane label="全部" name="1" />
    </el-tabs>
    <el-input v-model="searchContent" placeholder="请输入实体名" size="mini" @keyup.enter.native="handleSearch">
      <el-button slot="append" icon="el-icon-search" @click="handleSearch" />
    </el-input>
    <el-table
      v-loading="currentData.loading"
      :data="currentData.entityList"
      size="mini"
      :max-height="tableHeight"
      @row-contextmenu="handleTableRowRightClick"
      @row-click="handleTableRowClick"
    >
      <el-table-column type="expand">
        <template slot-scope="scope">
          <p>创建时间: {{ scope.row.createTime }} </p>
          <p>释义: {{ scope.row.annotation ? scope.row.annotation : '暂无释义' }}</p>
          <p>文档位置: {{ Array.from(new Set(scope.row.documentLocation)) }} </p>
        </template>
      </el-table-column>

      <el-table-column label="序号" width="55">
        <template slot-scope="scope">
          {{ scope.$index+1 }}
        </template>
      </el-table-column>

      <el-table-column label="类型" width="100" prop="type" />

      <el-table-column prop="name" label="实体名">
        <template slot-scope="scope">
          <span class="text-overflow" :title="scope.row.name">
            {{ scope.row.name }}
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
      localEntityData: {
        allEntities: [],
        entityList: [],
        totalNum: 0,
        pageNum: 1,
        pageSize: 10,
        loading: false
      },
      allEntityData: {
        entityList: [],
        totalNum: 0,
        pageNum: 1,
        pageSize: 10,
        loading: false
      },
      currentData: {
        entityList: [],
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
        this.currentData = this.localEntityData
      } else {
        this.currentData = this.allEntityData
      }
      this.$forceUpdate()
    }
  },

  mounted() {
    this.updateAllEntities()
    this.currentData = this.localEntityData
    setTimeout(() => { this.updateTableHeight() })
  },

  methods: {

    changeDocument(document) {
    //   if (this.currentDocument === document) { return }
      this.currentDocument = document
      const params = new URLSearchParams()
      params.append('documentLocation', document)
      this.localEntityData.loading = true
      this.$http.post('/neo4j/entity/getEntityByDocumentLocation', params).then(res => {
        this.localEntityData.loading = false
        if (res.data.status === 200) {
          this.setEntityList(res.data.data)
        }
      })
    },

    setEntityList(data) {
      this.localEntityData.allEntities = data
      this.localEntityData.totalNum = data.length
      this.localEntityData.pageNum = 1
      this.localEntityData.entityList = this.localEntityData.allEntities.slice(0, this.localEntityData.pageSize)
    },

    updateData(data) {
      this.setEntityList(data)
      this.allEntityData.pageNum = 1
      this.updateAllEntities()
    },

    locateEntity(entityId) {
      this.tabs = '0'
      for (let i = 0; i < this.localEntityData.allEntities.length; i++) {
        if (this.localEntityData.allEntities[i].id === entityId) {
          this.localEntityData.pageNum = Math.floor(i / this.localEntityData.pageSize) + 1
          this.localEntityData.entityList = this.localEntityData.allEntities.slice((this.localEntityData.pageNum - 1) * this.localEntityData.pageSize, this.localEntityData.pageNum * this.localEntityData.pageSize)
          break
        }
      }
    },

    updateAllEntities() {
      this.getAllEntities()
      this.getAllEntityNum()
    },

    setOperationText(text) {
      this.operationText = text
    },

    setTableHeight(height) {
      this.tableHeight = height
    },

    getAllEntities() {
      const porjectId = this.$route.query.projectId
      if (!porjectId) {
        this.allEntityData.entityList = []
      }
      this.allEntityData.loading = true
      const params = new URLSearchParams()
      params.append('projectId', porjectId)
      params.append('searchContent', this.searchContent)
      params.append('start', (this.allEntityData.pageNum - 1) * this.allEntityData.pageSize)
      params.append('length', this.allEntityData.pageSize)
      this.$http.post('/neo4j/entity/getEntitySearchResultOfProject', params).then(res => {
        if (res.data.status === 200) {
          this.allEntityData.entityList = res.data.data
        }
      }).finally(() => {
        this.allEntityData.loading = false
      })
    },

    getAllEntityNum() {
      const porjectId = this.$route.query.projectId
      if (!porjectId) {
        this.allEntityData.totalNum = 0
      }
      const params = new URLSearchParams()
      params.append('projectId', porjectId)
      params.append('searchContent', this.searchContent)
      this.$http.post('/neo4j/entity/getEntitySearchResultNumOfProject', params).then(res => {
        if (res.data.status === 200) {
          this.allEntityData.totalNum = res.data.data
        }
      })
    },

    handleSearch() {
      if (this.tabs === '0') {
        this.localEntityData.entityList = this.localEntityData.allEntities.filter(item => {
          return item.name.indexOf(this.searchContent) !== -1
        })
        this.localEntityData.totalNum = this.localEntityData.entityList.length
        this.localEntityData.pageNum = 1
        this.localEntityData.entityList = this.localEntityData.entityList.slice(0, this.localEntityData.pageSize)
      } else {
        this.getAllEntities()
        this.getAllEntityNum()
      }
    },

    handleSizeChange(val) {
      this.currentData.pageSize = val
      if (this.tabs === '0') {
        this.localEntityData.entityList = this.localEntityData.allEntities.slice((this.localEntityData.pageNum - 1) * this.localEntityData.pageSize, this.localEntityData.pageNum * this.localEntityData.pageSize)
      } else {
        this.updateAllEntities()
      }
    },

    handleCurrentChange(val) {
      this.currentData.pageNum = val
      if (this.tabs === '0') {
        this.localEntityData.entityList = this.localEntityData.allEntities.slice((this.localEntityData.pageNum - 1) * this.localEntityData.pageSize, this.localEntityData.pageNum * this.localEntityData.pageSize)
      } else {
        this.updateAllEntities()
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
