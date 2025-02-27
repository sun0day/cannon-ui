<template>
  <page-header-wrapper>
    <ConfigEditor v-if="visible" :visible="visible" :onClose="handleCloseConfigEditor" :onSave="handleSaveConfig" />
    <a-card :bordered="false">
      <div class="table-page-search-wrapper">
        <a-form layout="inline">
          <a-row :gutter="48">
            <a-col :md="8" :sm="24">
              <a-form-item label="url">
                <a-input v-model="queryParam.url" placeholder="target url" />
              </a-form-item>
            </a-col>

            <a-col :md="8" :sm="24">
              <a-form-item label="method">
                <a-select v-model="queryParam.method" :allowClear="true" placeholder="http method">
                  <a-select-option value="GET">GET</a-select-option>
                  <a-select-option value="POST">POST</a-select-option>
                  <a-select-option value="PUT">PUT</a-select-option>
                  <a-select-option value="DELETE">DELETE</a-select-option>
                </a-select>
              </a-form-item>
            </a-col>

            <template v-if="advanced">
              <a-col :md="8" :sm="24">
                <a-form-item label="connections">
                  <a-input-number
                    v-model="queryParam.connections"
                    style="width: 100%"
                    placeholder="concurrent connections"
                  />
                </a-form-item>
              </a-col>
              <a-col :md="8" :sm="24">
                <a-form-item label="duration">
                  <a-input-number v-model="queryParam.duration" style="width: 100%" placeholder="connection duration" />
                </a-form-item>
              </a-col>
              <a-col :md="8" :sm="24">
                <a-form-item label="pipelining">
                  <a-input-number
                    v-model="queryParam.pipelining"
                    style="width: 100%"
                    placeholder="request pipeline number"
                  />
                </a-form-item>
              </a-col>
              <a-col :md="8" :sm="24">
                <a-form-item label="create time">
                  <a-date-picker v-model="queryParam.createTime" style="width: 100%" placeholder="create time" />
                </a-form-item>
              </a-col>
            </template>
            <a-col :md="(!advanced && 8) || 24" :sm="24">
              <span
                class="table-page-search-submitButtons"
                :style="(advanced && { float: 'right', overflow: 'hidden' }) || {}"
              >
                <a-button type="primary" @click="$refs.table.refresh(true)">search</a-button>
                <a-button style="margin-left: 8px" @click="() => (this.queryParam = {})">reset</a-button>
                <a @click="toggleAdvanced" style="margin-left: 8px">
                  {{ advanced ? 'collapse' : 'expand' }}
                  <a-icon :type="advanced ? 'up' : 'down'" />
                </a>
              </span>
            </a-col>
          </a-row>
        </a-form>
      </div>

      <div class="table-operator">
        <a-button type="primary" icon="plus" @click="handleAdd">create</a-button>
        <a-dropdown v-action:edit v-if="selectedRowKeys.length > 0">
          <a-menu slot="overlay">
            <a-menu-item key="1"><a-icon type="delete" />删除</a-menu-item>
            <!-- lock | unlock -->
            <a-menu-item key="2"><a-icon type="lock" />锁定</a-menu-item>
          </a-menu>
          <a-button style="margin-left: 8px"> 批量操作 <a-icon type="down" /> </a-button>
        </a-dropdown>
      </div>

      <s-table ref="table" size="default" rowKey="cid" :columns="columns" :data="loadData" showPagination="auto">
        <span slot="serial" slot-scope="text, record, index">
          {{ index + 1 }}
        </span>
        <span slot="status" slot-scope="text">
          <a-badge :status="text | statusTypeFilter" :text="text | statusFilter" />
        </span>
        <span slot="description" slot-scope="text">
          <ellipsis :length="4" tooltip>{{ text }}</ellipsis>
        </span>

        <span slot="action" slot-scope="text, record">
          <template>
            <a @click="handleStartTest(record)">test</a>
          </template>
        </span>
      </s-table>

      <!-- <create-form
        ref="createModal"
        :visible="visible"
        :loading="confirmLoading"
        :model="mdl"
        @cancel="handleCancel"
        @ok="handleOk"
      />
      <step-by-step-modal ref="modal" @ok="handleOk" /> -->
    </a-card>
  </page-header-wrapper>
</template>

<script>
import moment from 'moment'
import { STable, Ellipsis, ConfigEditor } from '@/components'
import { getRoleList, getServiceList } from '@/api/manage'
import { startTest } from '@/api/test'
import { getConfigs, saveTest } from '@/utils/storage'
import { genId } from '@/utils/id'

// import StepByStepModal from './modules/StepByStepModal'
// import CreateForm from './modules/CreateForm'

const columns = [
  {
    title: 'cid',
    dataIndex: 'cid',
  },
  {
    title: 'url',
    dataIndex: 'url',
  },
  {
    title: 'status',
    dataIndex: 'status',
  },
  {
    title: 'connections',
    dataIndex: 'connections',
    width: '120px',
  },
  {
    title: 'duration',
    dataIndex: 'duration',
    width: '100px',
  },
  {
    title: 'pipelining',
    dataIndex: 'pipelining',
    width: '100px',
  },
  {
    title: 'create time',
    dataIndex: 'createTime',
    sorter: true,
    customRender: (text) => moment(text).format('YYYY-MM-DD HH:mm:ss'),
  },
  {
    title: 'action',
    dataIndex: 'action',
    width: '100px',
    scopedSlots: { customRender: 'action' },
    fixed: 'right',
  },
]

export default {
  name: 'Config',
  components: {
    STable,
    Ellipsis,
    ConfigEditor,
    // CreateForm,
    // StepByStepModal,
  },
  data() {
    this.columns = columns
    return {
      // create model
      visible: false,
      confirmLoading: false,
      mdl: null,
      // 高级搜索 展开/关闭
      advanced: false,
      // 查询参数
      queryParam: {},
      // 加载数据方法 必须为 Promise 对象
      loadData: (params) => {
        const data = getConfigs({ sortField: 'createTime', sortOrder: 'descend', ...params, filter: this.queryParam })
        return Promise.resolve({ data, totalCount: data.length, pageNo: params.pageNo })
      },
      selectedRowKeys: [],
      selectedRows: [],
    }
  },
  filters: {
    statusFilter(type) {
      return statusMap[type].text
    },
    statusTypeFilter(type) {
      return statusMap[type].status
    },
  },
  created() {
    getRoleList({ t: new Date() })
  },
  computed: {
    rowSelection() {
      return {
        selectedRowKeys: this.selectedRowKeys,
        onChange: this.onSelectChange,
      }
    },
  },
  methods: {
    handleCloseConfigEditor() {
      this.visible = false
    },
    handleSaveConfig() {
      this.visible = false
      this.$refs.table.refresh()
    },
    handleAdd() {
      this.visible = true
    },
    async handleStartTest(record) {
      const tid = genId()
      const createTime = new Date()
      const test = { ...record, tid, createTime }
      await startTest(test)
      saveTest({ tid, createTime, cid: record.cid })
      this.$notification.success({
        message: `${tid} has already started`,
        description: (
          <p>
            go to <a onclick={() => this.$router.push('/history')}>History</a> menu to check test status
          </p>
        ),
      })
    },
    handleOk() {
      const form = this.$refs.createModal.form
      this.confirmLoading = true
      form.validateFields((errors, values) => {
        if (!errors) {
          console.log('values', values)
          if (values.id > 0) {
            // 修改 e.g.
            new Promise((resolve, reject) => {
              setTimeout(() => {
                resolve()
              }, 1000)
            }).then((res) => {
              this.visible = false
              this.confirmLoading = false
              // 重置表单数据
              form.resetFields()
              // 刷新表格
              this.$refs.table.refresh()

              this.$message.info('修改成功')
            })
          } else {
            // 新增
            new Promise((resolve, reject) => {
              setTimeout(() => {
                resolve()
              }, 1000)
            }).then((res) => {
              this.visible = false
              this.confirmLoading = false
              // 重置表单数据
              form.resetFields()
              // 刷新表格
              this.$refs.table.refresh()

              this.$message.info('新增成功')
            })
          }
        } else {
          this.confirmLoading = false
        }
      })
    },
    handleCancel() {
      this.visible = false

      const form = this.$refs.createModal.form
      form.resetFields() // 清理表单数据（可不做）
    },
    handleSub(record) {
      if (record.status !== 0) {
        this.$message.info(`${record.no} 订阅成功`)
      } else {
        this.$message.error(`${record.no} 订阅失败，规则已关闭`)
      }
    },
    onSelectChange(selectedRowKeys, selectedRows) {
      this.selectedRowKeys = selectedRowKeys
      this.selectedRows = selectedRows
    },
    toggleAdvanced() {
      this.advanced = !this.advanced
    },
    resetSearchForm() {
      this.queryParam = {
        date: moment(new Date()),
      }
    },
  },
}
</script>
