<template>
  <div style="width: 100%">
    <el-form label-width="80px" class="entity-edit-form" size="small">
      <el-form-item label="实体名称">
        <el-input
          v-model="entityEditForm.name"
          type="textarea"
          :rows="1"
          :readonly="!canEdit"
        />
      </el-form-item>
      <el-form-item label="实体类型" style="text-align: left">
        <el-select
          v-if="canEdit"
          v-model="entityEditForm.type"
          placeholder="请选择实体类型"
          filterable
        >
          <el-option
            v-for="item in entityClassList"
            :key="item.nodeId"
            :label="item.name"
            :value="item.name"
          />
        </el-select>
        <el-input v-else v-model="entityEditForm.type" :rows="1" :readonly="true" />
      </el-form-item>
      <el-form-item label="释义">
        <el-input
          v-model="entityEditForm.annotation"
          type="textarea"
          :rows="2"
          :readonly="!canEdit"
        />
      </el-form-item>
      <el-form-item v-if="formType != 'default'" label="三元组" style="text-align: left">
        <el-button type="text" @click="handleShowRelation">查看</el-button>
      </el-form-item>
      <div v-if="formType != 'default'">
        <el-button type="primary" size="small" @click="changeEditStatus(!canEdit)">
          {{ canEdit ? '取消' : '编辑' }}
        </el-button>
        <el-button v-if="canEdit" type="primary" size="small" @click="handleSave">
          保存
        </el-button>
      </div>
    </el-form>
    <el-dialog :visible.sync="relationDialogShow" width="50%" :modal="false">
      <el-table v-loading="tableLoading" :data="relationList">
        <el-table-column label="序号" width="55">
          <template slot-scope="scope">
            {{ scope.$index+1 }}
          </template>
        </el-table-column>
        <el-table-column label="关系名" prop="relationName" width="120" />
        <el-table-column label="宾语">
          <template slot-scope="scope">
            <span class="text-overflow" :title="scope.row.relatedEntity.name">
              {{ scope.row.relatedEntity.name }}
            </span>
          </template>
        </el-table-column>
      </el-table>
    </el-dialog>
  </div>
</template>

<script>
export default {

  props: {
    formType: {
      type: String,
      default: 'default'
    },
    entityClassList: {
      type: Array,
      default: () => []
    }
  },

  data() {
    return {
      entity: { nodeId: null, name: '', type: '', annotation: '' },
      entityEditForm: { name: '', type: '', annotation: '' },
      canEdit: true,
      relationDialogShow: false,
      relationList: [],
      tableLoading: false
    }
  },

  mounted() {
    if (this.formType !== 'default') {
      this.canEdit = false
    }
  },
  methods: {
    setEntity(entity) {
      this.entity = entity
      this.entityEditForm = {
        name: entity.name,
        type: entity.type,
        annotation: entity.annotation
      }
      if (this.formType !== 'default' && this.entity) {
        this.changeEditStatus(false)
      }
      if (this.formType === 'default' && this.entity) {
        this.changeEditStatus(true)
        this.entityEditForm = this.entity
      }
    },
    changeEditStatus(status) {
      this.canEdit = status
      if (!status) {
        this.entityEditForm = {
          name: this.entity.name,
          type: this.entity.type,
          annotation: this.entity.annotation
        }
      }
    },
    handleSave() {
      const params = new URLSearchParams()
      params.append('nodeId', this.entity.nodeId)
      params.append('name', this.entityEditForm.name)
      params.append('type', this.entityEditForm.type)
      params.append('annotation', this.entityEditForm.annotation)
      this.$http.post('/neo4j/entity/updateEntity', params).then(res => {
        if (res.data.status === 200) {
          this.$message.success('修改成功')
          this.canEdit = false
          this.entity.name = this.entityEditForm.name
          this.entity.type = this.entityEditForm.type
          this.entity.annotation = this.entityEditForm.annotation
          this.$emit('updateEntity', this.entity)
        } else {
          this.$message.error('修改失败')
        }
      })
    },

    handleShowRelation() {
      this.relationDialogShow = true
      const params = new URLSearchParams()
      params.append('nodeId', this.entity.nodeId)
      this.tableLoading = true
      this.$http.post('/neo4j/entity/getRelationByEntityId', params).then(res => {
        if (res.data.status === 200) {
          this.relationList = res.data.data
        } else {
          this.$message.error('获取关系列表失败')
        }
      }).finally(() => {
        this.tableLoading = false
      })
    }
  }
}
</script>
<style scoped>
.entity-edit-form {
    margin: 10px;
}
</style>
