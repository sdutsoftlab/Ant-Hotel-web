<template>
  <a-card :bordered="false">
    <div class="table-page-search-wrapper">
      <a-form layout="inline">
        <a-row :gutter="48">
          <a-col :md="8" :sm="24">
            <a-form-item label="用户编号">
              <a-input v-model="queryParam.roomId" />
            </a-form-item>
          </a-col>
          <a-col :md="8" :sm="24">
            <a-form-item label="用户名">
              <a-input v-model="queryParam.name" />
            </a-form-item>
          </a-col>
          <!--          <template v-if="advanced">-->
          <!--            <a-col :md="10" :sm="24">-->
          <!--              <a-form-item label="住宿时间">-->
          <!--                <a-date-picker v-model="queryParam.signTime" style="width: 100%" placeholder="请输入更新日期" />-->
          <!--              </a-form-item>-->
          <!--            </a-col>-->
          <!--            <a-col :md="10" :sm="24">-->
          <!--              <a-form-item label="退宿时间">-->
          <!--                <a-date-picker v-model="queryParam.leaveTime" style="width: 100%" placeholder="请输入更新日期" />-->
          <!--              </a-form-item>-->
          <!--            </a-col>-->
          <!--          </template>-->
          <a-col :md="!advanced && 8 || 24" :sm="24">
            <span
              class="table-page-search-submitButtons"
              :style="advanced && { float: 'right', overflow: 'hidden' } || {} "
            >
              <a-button type="primary" @click="handleSumbit(queryParam)">查询</a-button>
              <a-button style="margin-left: 8px" @click="() => queryParam = {}">重置</a-button>
              <a @click="toggleAdvanced" style="margin-left: 8px">
                {{ advanced ? '收起' : '展开' }}
                <a-icon :type="advanced ? 'up' : 'down'" />
              </a>
            </span>
          </a-col>
        </a-row>
      </a-form>
    </div>

    <div class="table-operator">
      <a-dropdown v-action:edit v-if="selectedRowKeys.length > 0">
        <a-menu slot="overlay">
          <a-menu-item key="1">
            <a-icon type="delete" />删除
          </a-menu-item>
          <!-- lock | unlock -->
          <a-menu-item key="2">
            <a-icon type="lock" />锁定
          </a-menu-item>
        </a-menu>
        <a-button style="margin-left: 8px">
          批量操作
          <a-icon type="down" />
        </a-button>
      </a-dropdown>
    </div>
    <a-table
      :loading="loading"
      :columns="columns"
      :pagination="pagination"
      rowKey="updateTime"
      :dataSource="loadData"
    >
      <span slot="type" slot-scope="type">
        <a-tag
          :color="type === 0 ? 'volcano' : (type === 1 ? 'green' : 'geekblue')"
        >{{type === 0 ? '进入' : (type === 1 ? '离开' : '已过期')}}</a-tag>
      </span>
    </a-table>
    <step-by-step-modal ref="modal" @ok="handleOk" />
  </a-card>
</template>

<script>
import moment from 'moment'
import { STable, Ellipsis } from '@/components'
import StepByStepModal from '@/views/list/modules/StepByStepModal'
import CreateForm from '@/views/list/modules/CreateForm'
import { getRoleList, getOrderList, getRecord } from '@/api/manage'
import { inoutSearch } from '../../api/manage'
const formatterTime = val => {
  return val ? moment(val).format('YYYY-MM-DD HH:mm:ss') : ''
}
const formatter = (val, dict) => {
  return val ? dict[val] : ''
}

const statusMap = {
  0: {
    status: 'default',
    text: '进入'
  },
  1: {
    status: 'processing',
    text: '离开'
  }
}

export default {
  name: 'TableList',
  components: {
    STable,
    Ellipsis,
    CreateForm,
    StepByStepModal
  },
  data() {
    return {
      mdl: {},
      // 高级搜索 展开/关闭
      advanced: false,
      // 查询参数
      queryParam: {},
      // 表头
      columns: [
        {
          title: '用户Id',
          dataIndex: 'userId'
        },
        {
          title: '用户姓名',
          dataIndex: 'name'
        },
        {
          title: '房间号',
          dataIndex: 'roomId'
          // scopedSlots: { customRender: 'roomType' }
        },
        {
          title: '时间',
          dataIndex: 'updateTime',
          sorter: true,
          customRender: val => {
            return moment(val).format('YYYY-MM-DD HH:mm:ss')
          }
          // needTotal: true,
          // customRender: text => text + ' RMB'
        },
        {
          title: '进出记录',
          dataIndex: 'type',
          scopedSlots: { customRender: 'type' }
        }
        // {
        //   title: '操作',
        //   dataIndex: 'action',
        //   width: '150px',
        //   align: 'center',
        //   scopedSlots: { customRender: 'action' }
        // }
      ],
      pagination: {
        pageSize: 10,
        defaultPageSize: 10,
        defaultCurrent: 1,
        showSizeChanger: true,
        showQuickJumper: true,
        pageSizeOptions: ['10', '20', '30', '40']
      },
      loading: true,
      // 加载数据方法 必须为 Promise 对象
      loadData: [],
      selectedRowKeys: [],
      selectedRows: [],

      // custom table alert & rowSelection
      options: {
        alert: {
          show: true,
          clear: () => {
            this.selectedRowKeys = []
          }
        },
        rowSelection: {
          selectedRowKeys: this.selectedRowKeys,
          onChange: this.onSelectChange
        }
      },
      optionAlertShow: false
    }
  },
  filters: {
    statusFilter(type) {
      return statusMap[type].text
    },
    statusTypeFilter(type) {
      return statusMap[type].status
    }
  },
  created() {
    this.vueTable()
    // this.tableOption()
    // getRoleList({ t: new Date() })
  },
  methods: {
    handleSumbit(queryParam) {
      this.queryParam = queryParam
      inoutSearch({ roomId: queryParam.roomId, name: queryParam.name })
        .then(res => {
          console.log(res)
          this.loadData = res.data
          this.loading = false
        })
        .catch(e => {
          this.loading = false
        })
    },
    tableOption() {
      if (!this.optionAlertShow) {
        this.options = {
          alert: {
            show: true,
            clear: () => {
              this.selectedRowKeys = []
            }
          },
          rowSelection: {
            selectedRowKeys: this.selectedRowKeys,
            onChange: this.onSelectChange,
            getCheckboxProps: record => ({
              props: {
                disabled: record.no === 'No 2', // Column configuration not to be checked
                name: record.no
              }
            })
          }
        }
        this.optionAlertShow = true
      } else {
        this.options = {
          alert: false,
          rowSelection: null
        }
        this.optionAlertShow = false
      }
    },
    vueTable() {
      getRecord()
        .then(res => {
          console.log(res)
          this.loadData = res.data
          this.loading = false
        })
        .catch(e => {
          this.loading = false
        })
    },
    handleEdit(record) {
      console.log(record)
      this.$refs.modal.edit(record)
    },
    handleSub(record) {
      if (record.status !== 0) {
        this.$message.info(`${record.no} 订阅成功`)
      } else {
        this.$message.error(`${record.no} 订阅失败，规则已关闭`)
      }
    },
    handleOk() {
      this.$refs.table.refresh()
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
        date: moment(new Date())
      }
    }
  }
}
</script>
