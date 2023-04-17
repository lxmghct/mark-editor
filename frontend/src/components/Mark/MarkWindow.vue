<!--
 * @FileDescription: 标注窗口
 -->
<template>
  <div class="mark-window-main">
    <div
      class="header"
      @mousedown="_handleHeaderMouseDown"
      @mousemove="_handleHeaderMouseMove"
      @mouseup="_handleHeaderMouseUp"
      @mouseleave="_handleHeaderMouseLeave"
    >
      {{ windowType }}
      <span class="header-buttons">
        <el-button
          size="mini"
          type="danger"
          icon="el-icon-close"
          circle
          @click="show(false)"
        />
      </span>
    </div>

    <div v-if="windowType == '实体融合'" style="height: 350px">
      <div class="entity-merge-header">
        <span style="white-space:nowrap">
          <i class="el-icon-warning" />
          存在以下已创建的同名实体, 请选择要融合的实体。
          <span style="font-weight: bold">实体名: </span>
        </span>
        <span class="entity-name" :title="entityDetail.name">
          {{ entityDetail.name }}
        </span>
      </div>
      <el-table v-loading="loading" :data="entityMergeList" height="240">
        <el-table-column label="类型" prop="entityNode.type" width="150" />
        <el-table-column label="释义" width="270">
          <template slot-scope="scope">
            <span :title="scope.row.entityNode.annotation" class="text-overflow">
              {{ scope.row.entityNode.annotation ? scope.row.entityNode.annotation : '暂无释义' }}
            </span>
          </template>
        </el-table-column>
        <el-table-column label="三元组" width="150">
          <template slot-scope="scope">
            <el-popover placement="top" width="300" trigger="click">
              <el-table :data="scope.row.relatedEntity" height="300">
                <!-- <el-table-column label="主语">{{ entityDetail.name }}</el-table-column> -->
                <el-table-column label="关系名" prop="relationName" width="100" />
                <el-table-column label="宾语">
                  <template slot-scope="prop">
                    <span :title="prop.row.entity.name" class="text-overflow">
                      {{ prop.row.entity.name }}
                    </span>
                  </template>
                </el-table-column>
              </el-table>
              <span slot="reference" style="color: #0000ff; cursor: pointer">
                共 {{ scope.row.relatedEntity.length }} 条
              </span>
            </el-popover>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="100">
          <template slot-scope="scope">
            <el-button type="primary" size="mini" @click="_handleClickMergeEntity(scope.row.entityNode.nodeId)">融合</el-button>
          </template>
        </el-table-column>
      </el-table>
      <div style="margin-top: 15px">
        <el-button size="medium" @click="show(false)"> 取消 </el-button>
        <el-button type="primary" size="medium" @click="_changeToEntityCreateWindow"> 不融合, 继续创建实体 </el-button>
      </div>
    </div>

    <div v-if="windowType == '实体标注'" style="height: 350px;">
      <div class="side" style="width: 80%">
        <h3>实体创建</h3>
        <entity-edit-form ref="editForm" :entity-class-list="entityClassList" />
        <el-button type="primary" @click="_handleCreateEntity">确定</el-button>
      </div>
      <div class="side" style="width: 15%; border-left: 1px #000 solid;">
        <el-link type="primary" style="margin-top: 20px" @click="_changeToEntityMergeWindow">
          <span>实体融合</span>
        </el-link>
        <p>
          共存在 <span style="font-weight: bold">{{ entityMergeCount > 0 ? entityMergeCount : 0 }}</span> 个同名实体
        </p>
        <el-link type="primary" @click="_changeToEntityMergeWindow">
          <span>查看</span>
        </el-link>
      </div>
    </div>

    <div v-if="windowType == '三元组标注'" v-loading="loading" style="height: 350px;">
      <split-pane split="vertical" :default-percent="36">
        <template slot="paneL">
          <div style="height:100%; overflow-y:auto">
            <div class="title">主语</div>
            <el-button v-if="relationDetail.subject" type="text" @click="_handleChooseEntityDialogShow('subject')">重新选择</el-button>
            <template v-if="!relationDetail.subject">
              <h3>暂无实体信息</h3>
              <el-button
                type="primary"
                size="medium"
                @click="_handleChooseEntityDialogShow('subject')"
              >选择实体</el-button>
            </template>
            <entity-edit-form
              v-if="relationDetail.subject"
              ref="subject"
              :entity-class-list="entityClassList"
              form-type="relation"
              @update-entity="_handleUpdateEntity"
            />
          </div>
        </template>
        <template slot="paneR">
          <split-pane split="vertical" :default-percent="43.75">
            <template slot="paneL">
              <div style="height:100%; overflow-y:auto; padding: 5px">
                <h3>关系选择</h3>
                <el-select
                  v-model="relationDetail.relationName"
                  filterable
                  placeholder="请选择关系"
                  size="medium"
                  @visible-change="(val)=>{val && _getRelationNameList()}"
                >
                  <el-option v-for="item in relationNameList" :key="item.value" :label="item.name" :value="item.value" />
                </el-select>
                <div>
                  <el-button type="text" @click="_getEntityAndRelationClassList">刷新</el-button>
                  <el-button
                    type="text"
                    :disabled="!relationDetail.subject || !relationDetail.object"
                    @click="_handleRelationAddDialogShow"
                  >添加</el-button>
                  <el-button type="text" @click="_handleExchange">交换</el-button>
                </div>
                <el-button type="primary" @click="_handleCreateRelation">标注三元组</el-button>
              </div>
            </template>
            <template slot="paneR">
              <div style="height:100%; overflow-y:auto">
                <div class="title">宾语</div>
                <el-button v-if="relationDetail.object" type="text" @click="_handleChooseEntityDialogShow('object')">重新选择</el-button>
                <template v-if="!relationDetail.object">
                  <h3>暂无实体信息</h3>
                  <el-button
                    type="primary"
                    size="medium"
                    @click="_handleChooseEntityDialogShow('object')"
                  >选择实体</el-button>
                </template>
                <entity-edit-form
                  v-if="relationDetail.object"
                  ref="object"
                  :entity-class-list="entityClassList"
                  form-type="relation"
                  @update-entity="_handleUpdateEntity"
                />
              </div>
            </template>
          </split-pane>
        </template>
      </split-pane>
    </div>

    <el-dialog :visible.sync="entityChooseDialogShow" width="500px" :modal="false">
      <div style="height: 400px; display: inline-block; width: 90%">
        <entity-table
          ref="entityTable"
          @click-operation="_handleChooseEntity"
        />
      </div>
    </el-dialog>

    <el-dialog :visible.sync="relationAddDialogShow" width="400px" :modal="false">
      <el-form :model="relationAddDialodData" size="small" label-width="100px">
        <el-form-item label="主语类型">
          <el-select v-model="relationAddDialodData.subjectType" filterable placeholder="请选择主语类型">
            <el-option v-for="item in entityClassList" :key="item.nodeId" :label="item.name" :value="item.name" />
          </el-select>
        </el-form-item>
        <el-form-item label="关系类型">
          <el-input
            v-model="relationAddDialodData.relationName"
            style="width: 205px"
          />
        </el-form-item>
        <el-form-item label="宾语类型">
          <el-select v-model="relationAddDialodData.objectType" filterable placeholder="请选择宾语类型">
            <el-option v-for="item in entityClassList" :key="item.nodeId" :label="item.name" :value="item.name" />
          </el-select>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="relationAddDialogShow = false">取 消</el-button>
        <el-button type="primary" @click="_handleRelationAdd">确 定</el-button>
      </div>
    </el-dialog>

  </div>
</template>

<script>
import { getPointX, getPointY } from '@/assets/js/index'
import EntityEditForm from './EntityEditForm.vue'
import EntityTable from './EntityTable.vue'

export default {

  name: 'MarkWindow',
  components: { EntityEditForm, EntityTable },

  props: {
    fileData: {
      type: Object,
      default: () => {}
    }

  },

  data() {
    return {
      markWindow: null,
      dialogShow: false,
      isMouseDown: false,
      windowType: '实体融合', // 窗口类型
      oldMousePoint: { x: 0, y: 0 },
      oldWindowPoint: { x: 0, y: 0 },

      entityClassList: [], // 实体类列表
      entityDetail: {
        name: '',
        type: '',
        annotation: ''
      },
      entityMergeList: [],
      entityMergeCount: -1,

      relationClassList: [], // 关系类列表
      relationDetail: {
        subject: null,
        object: null,
        relationName: ''
      },
      relationNameList: [],

      entityChooseDialogShow: false,
      chosenEntityKey: '',

      relationAddDialogShow: false,
      relationAddDialodData: {
        subjectType: '',
        objectType: '',
        relationName: ''
      },

      loading: false

    }
  },

  mounted() {
    this.markWindow = document.getElementsByClassName('mark-window-main')[0]
    this.markWindow.style.left = 300 + 'px'
    this.markWindow.style.top = 100 + 'px'
    this._getEntityAndRelationClassList()
  },

  methods: {

    /**
     * @description: 显示/隐藏当前窗口
     * @param {Boolean} isShow 是否显示
     * @return {void}
     */
    show(isShow = true) {
      this.dialogShow = isShow
      if (isShow) {
        this.markWindow.style.display = 'block'
      } else {
        this.markWindow.style.display = 'none'
        this._initialize()
      }
    },

    /**
     * @description: 实体标注
     * @param {String} name
     * @return {void}
     */
    markEntity(name) {
      this._initialize()
      this.entityDetail.name = name
      this._changeToEntityCreateWindow()
      this.show()
    },

    /**
     * @description: 三元组标注
     * @param {Object} subject
     * @param {Object} object
     * @return {void}
     */
    markRelation(subject, object = null) {
      this._changeToRelationCreateWindow()
      this.relationDetail.subject = subject
      this.relationDetail.object = object
      this.$nextTick(() => {
        if (subject) {
          this.$refs.subject.setEntity(this.relationDetail.subject)
        }
        if (object) {
          this.$refs.object.setEntity(this.relationDetail.object)
        }
      })
      this.show()
    },

    _initialize() {
      this.isMouseDown = false
      this.entityDetail.name = ''
      this.entityDetail.annotation = ''
      this.entityMergeList = []
      this.entityMergeCount = -1
      this.relationDetail.subject = null
      this.relationDetail.object = null
      this.relationDetail.relationName = ''
    },

    _getEntityAndRelationClassList() {
      if (!this.$route.query.projectId) { return }
      let params = new URLSearchParams()
      params.append('nodeId', this.$route.query.projectId)
      this.loading = true
      this.$http.post('/neo4j/project/getEntityClass', params).then(res => {
        if (res.data.status === 200) {
          this.entityClassList.splice(0, this.entityClassList.length)
          this.relationClassList.splice(0, this.relationClassList.length)
          for (let i = 0; i < res.data.data.length; i++) {
            let temp = res.data.data[i]
            this.entityClassList.push(temp.entityClass)
            try {
              for (let j = 0; j < temp['关系'].length; j++) {
                let t = temp['关系'][j]
                this.relationClassList.push({
                  nodeId: t.ID,
                  relationName: t['关系名'],
                  subjectId: temp.entityClass.nodeId,
                  subjectName: temp.entityClass.name,
                  objectId: t['宾语类'].nodeId,
                  objectName: t['宾语类'].name
                })
              }
            } catch (e) {
              console.log(e)
            }
          }
        }
      }).finally(() => {
        this.loading = false
      })
    },

    _changeToEntityMergeWindow() {
      this.windowType = '实体融合'
      if (this.entityMergeCount >= 0) { return }
      this.entityMergeList = []
      this.loading = true
      let params = new URLSearchParams()
      params.append('entityName', this.entityDetail.name)
      this.$http.post('/neo4j/entity/checkEntityByName', params).then(response => {
        if (response.data.status === 200) {
          this.entityMergeList = response.data.data
          this.entityMergeCount = this.entityMergeList.length
          if (this.entityMergeCount === 0) {
            this._changeToEntityCreateWindow()
          }
        }
      }).finally(() => {
        this.loading = false
      })
    },

    _changeToEntityCreateWindow() {
      this.windowType = '实体标注'
      this.$nextTick(() => {
        this.$refs.editForm.setEntity(this.entityDetail)
      })
    },

    _changeToRelationCreateWindow() {
      this.windowType = '三元组标注'
    },

    _getRelationNameList() {
      let list = []
      try {
        for (let i = 0; i < this.relationClassList.length; i++) {
          let temp = this.relationClassList[i]
          if (temp.subjectName === this.relationDetail.subject.type && temp.objectName === this.relationDetail.object.type) {
            list.push({
              name: `${temp.subjectName}--${temp.relationName}--${temp.objectName}`,
              value: temp.relationName
            })
          }
        }
      } catch (e) {
        console.log(e)
      }
      this.relationNameList = list
    },

    _handleRelationAddDialogShow() {
      if (!this.relationDetail.subject || !this.relationDetail.object) {
        this.$message.error('主语或宾语不存在')
        return
      }
      this.relationAddDialodData = {
        subjectType: this.relationDetail.subject.type,
        objectType: this.relationDetail.object.type,
        relationName: ''
      }
      this.relationAddDialogShow = true
    },

    _handleRelationAdd() {
      if (this.relationAddDialodData.relationName === '') {
        this.$message.error('关系名不能为空')
        return
      }
      for (let i = 0; i < this.relationClassList.length; i++) {
        let temp = this.relationClassList[i]
        if (temp.subjectName === this.relationAddDialodData.subjectType && temp.objectName === this.relationAddDialodData.objectType && temp.relationName === this.relationAddDialodData.relationName) {
          this.$message.error('关系已存在')
          return
        }
      }
      let subjectId, objectId
      for (let entity of this.entityClassList) {
        if (entity.name === this.relationAddDialodData.subjectType) {
          subjectId = entity.nodeId
        }
        if (entity.name === this.relationAddDialodData.objectType) {
          objectId = entity.nodeId
        }
      }
      const params = new URLSearchParams()
      params.append('subjectId', subjectId)
      params.append('objectId', objectId)
      params.append('predicateName', this.relationAddDialodData.relationName)
      params.append('time', new Date().toLocaleString())
      params.append('user', localStorage.getItem('userId'))
      this.$http.post('/neo4j/entityClass/addRelationship', params).then(res => {
        if (res.data.status === 200) {
          this.$message.success('添加成功')
          this.relationClassList.push({
            nodeId: res.data.data,
            relationName: this.relationAddDialodData.relationName,
            subjectId: subjectId,
            subjectName: this.relationAddDialodData.subjectType,
            objectId: objectId,
            objectName: this.relationAddDialodData.objectType
          })
          this.relationAddDialogShow = false
        }
      })
    },

    _handleClickMergeEntity(entityId) {
      this.$emit('mark-event', {
        type: 'merge_entity',
        data: { entityId: entityId }
      })
      this.show(false)
    },

    _handleChooseEntityDialogShow(entityKeyName) {
      this.entityChooseDialogShow = true
      this.chosenEntityKey = entityKeyName
      this.$nextTick(() => {
        this.$refs.entityTable.setOperationText('选择')
        this.$refs.entityTable.changeDocument(this.fileData.documentLocation)
      })
    },

    _handleCreateRelation() {
      if (!this.relationDetail.subject || !this.relationDetail.object) {
        this.$message.error('主语或宾语不存在')
        return
      }
      if (!this.relationDetail.relationName) {
        this.$message.error('关系名不能为空')
        return
      }
      this.$emit('mark-event', {
        type: 'create_relation',
        data: this.relationDetail
      })
      this.show(false)
    },

    _handleCreateEntity() {
      if (!this.entityDetail.name || !this.entityDetail.type) {
        this.$message.error('实体名和类型不能为空')
        return
      }
      this.$emit('mark-event', {
        type: 'create_entity',
        data: this.entityDetail
      })
      this.show(false)
    },

    _handleUpdateEntity(entity) {
      this.$emit('mark-event', {
        type: 'update_entity',
        data: entity
      })
      this.show(false)
    },

    _handleChooseEntity(entity) {
      this.relationDetail[this.chosenEntityKey] = entity
      this.$nextTick(() => {
        this.$refs[this.chosenEntityKey].setEntity(this.relationDetail[this.chosenEntityKey])
      })
      this.entityChooseDialogShow = false
    },

    _handleExchange() {
      let obj1 = this.relationDetail.subject
      let obj2 = this.relationDetail.object
      this.markRelation(obj2, obj1)
    },

    _handleHeaderMouseDown(e) {
      this.isMouseDown = true
      this.oldMousePoint = { x: e.clientX, y: e.clientY }
      this.oldWindowPoint = { x: getPointX(this.markWindow), y: getPointY(this.markWindow) }
    },

    _handleHeaderMouseUp() {
      this.isMouseDown = false
    },

    _handleHeaderMouseMove(e) {
      if (this.isMouseDown) {
        // let scrollLeft = document.documentElement.scrollLeft || document.body.scrollLeft
        // let scrollTop = document.documentElement.scrollTop || document.body.scrollTop
        this.markWindow.style.left = this.oldWindowPoint.x + e.clientX - this.oldMousePoint.x + 'px'
        this.markWindow.style.top = this.oldWindowPoint.y + e.clientY - this.oldMousePoint.y + 'px'
      }
    },

    _handleHeaderMouseLeave() {
      this.isMouseDown = false
    }

  }
}
</script>

<style scoped>
.mark-window-main {
    width: 700px;
    background: #fff;
    border: 1px #000 solid;
    position: absolute;
    display: none;
    z-index: 10;
    border-radius: 10px;
}
.header {
    background: black;
    color: white;
    font-size: 20px;
    text-align: left;
    padding: 8px;
    border-radius: 10px 10px 0 0;
    cursor: move;
}
.header-buttons {
    float: right;
    width: 30px;
}
.entity-edit-form {
    margin: 10px;
}
.entity-merge-header {
    background: rgb(240, 249, 235);
    padding: 10px;
    font-size: 16px;
    text-align: left;
    display: flex;
}
.entity-merge-header .entity-name {
    color: rgb(246, 118, 27);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    font-size: 24px;
    cursor: pointer;
}
.entity-merge-header i {
    color: rgb(245, 163, 0);
    font-size: 30px;
}
.text-overflow {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
.side {
    display: inline-block;
    vertical-align: top;
    height: 100%;
}
.title {
    display: inline-block;
    font-size: 20px;
    font-weight: bold;
}
</style>
