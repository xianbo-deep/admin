<template>
  <view class="fix-top-window">
    <view class="uni-header">
      <uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
      <view class="uni-group">
        <input 
          class="uni-search" 
          type="text" 
          v-model="query" 
          @confirm="search"
          placeholder="搜索套餐名称..." 
        />
        <button 
          class="uni-button hide-on-phone" 
          type="default" 
          size="mini"
          @click="search"
        >搜索</button>
        <button 
          class="uni-button" 
          type="primary" 
          size="mini"
          @click="navigateTo('./add')"
        >新增套餐</button>
      </view>
    </view>

    <view class="uni-container">
      <unicloud-db 
        ref="udb" 
        collection="Package" 
        field="package_id,package_name,AgentId,status,metrics"
        :where="where"
        page-data="replace" 
        :getcount="true" 
        :page-size="options.pageSize"
        :page-current="options.pageCurrent" 
        :orderby="orderby"
        loadtime="manual"
        @load="onqueryload"
        v-slot:default="{ pagination, loading, error }"
      >
        <uni-table 
          ref="table" 
          :loading="loading" 
          :emptyText="error.message || '没有数据'" 
          border 
          stripe
        >
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'package_id')">套餐ID</uni-th>
            <uni-th align="center">套餐名称</uni-th>
            <uni-th align="center">Agent ID</uni-th>
            <uni-th align="center">包含指标</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'status')">状态</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="item in displayData" :key="item._id">
            <uni-td align="center">{{item.package_id}}</uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.package_name">{{item.package_name}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.AgentId">{{item.AgentId}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="metrics-list">
                <uni-tag 
                  v-for="metric in item.metrics" 
                  :key="metric.metric_id"
                  :text="metric.metric_name"
                  type="primary"
                  size="small"
                />
              </view>
            </uni-td>
            <uni-td align="center">
              <uni-tag 
                :text="item.status === 'active' ? '启用' : '禁用'" 
                :type="item.status === 'active' ? 'success' : 'warning'" 
              />
            </uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button 
                  @click="handleStatusChange(item)" 
                  class="uni-button" 
                  size="mini"
                  :type="item.status === 'active' ? 'warning' : 'primary'"
                >{{item.status === 'active' ? '禁用' : '启用'}}</button>
                <button 
                  @click="navigateTo(`./edit?id=${item._id}`)" 
                  class="uni-button" 
                  size="mini"
                  type="primary"
                >编辑</button>
              </view>
            </uni-td>
          </uni-tr>
        </uni-table>

        <view class="uni-pagination-box">
          <uni-pagination 
            show-icon 
            show-page-size 
            :page-size="pagination.size"
            v-model="pagination.current"
            :total="pagination.count" 
            @change="onPageChanged" 
            @pageSizeChange="changeSize" 
          />
        </view>
      </unicloud-db>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database()
const dbCmd = db.command

// 表查询配置
const dbOrderBy = 'package_id asc'
const dbSearchFields = ['package_name']
const pageSize = 20
const pageCurrent = 1

const orderByMapping = {
  "ascending": "asc",
  "descending": "desc"
}

export default {
  data() {
    return {
      query: '',
      where: '',
      orderby: dbOrderBy,
      orderByFieldName: "",
      options: {
        pageSize,
        pageCurrent,
        getone: false,
        manual: false
      },
      data: []
    }
  },

  computed: {
    displayData() {
      return this.data
    }
  },

  mounted() {
    this.loadData()
  },

  methods: {
    getWhere() {
      const query = this.query.trim()
      if (!query) {
        return ''
      }
      const queryRe = new RegExp(query, 'i')
      return dbCmd.or(
        dbSearchFields.map(name => ({
          [name]: queryRe
        }))
      )
    },

    search() {
      const newWhere = this.getWhere()
      this.where = newWhere
      this.$nextTick(() => {
        this.loadData()
      })
    },

    changeSize(pageSize) {
      this.options.pageSize = pageSize
      this.options.pageCurrent = 1
      this.$nextTick(() => {
        this.loadData()
      })
    },

    loadData(clear = true) {
      if (this.$refs.udb) {
        this.$refs.udb.loadData({
          clear
        })
      }
    },

    onPageChanged(e) {
      this.$refs.udb.loadData({
        current: e.current
      })
    },

    sortChange(e, name) {
      this.orderByFieldName = name
      if (e.order) {
        this.orderby = name + ' ' + orderByMapping[e.order]
      } else {
        this.orderby = dbOrderBy
      }
      this.$nextTick(() => {
        this.$refs.udb.loadData()
      })
    },

    async onqueryload(data) {
      const packageData = Array.isArray(data) ? data : data.data
      if (!packageData || packageData.length === 0) {
        this.$set(this, 'data', [])
        return
      }
      this.$set(this, 'data', packageData)
    },

    async handleStatusChange(item) {
      const newStatus = item.status === 'active' ? 'inactive' : 'active'
      const actionText = newStatus === 'active' ? '启用' : '禁用'
      
      uni.showModal({
        title: '提示',
        content: `确认${actionText}该套餐？`,
        success: async (res) => {
          if (res.confirm) {
            try {
              await db.collection('Package').doc(item._id).update({
                status: newStatus
              })
              
              this.loadData(true)
              
              uni.showToast({
                title: `${actionText}成功`,
                icon: 'success'
              })
            } catch (error) {
              console.error(`${actionText}失败:`, error)
              uni.showToast({
                title: `${actionText}失败`,
                icon: 'error'
              })
            }
          }
        }
      })
    },

    navigateTo(url, clear = true) {
      if (!url) return
      uni.navigateTo({
        url,
        events: {
          refreshData: () => {
            this.loadData(clear)
          }
        }
      })
    }
  }
}
</script>

<style>
.fix-top-window {
  padding-top: 0;
}

.uni-header {
  padding: 20rpx;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #e5e5e5;
}

.uni-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.uni-search {
  flex: 1;
  padding: 0 10px;
  height: 29px;
  line-height: 29px;
  font-size: 12px;
  border: 1px solid #DCDFE6;
  border-radius: 4px;
}

.uni-container {
  padding: 20rpx;
}

.uni-pagination-box {
  margin-top: 20rpx;
}

.metrics-list {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  justify-content: center;
}

.ellipsis {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  max-width: 200px;
}

@media screen and (max-width: 768px) {
  .hide-on-phone {
    display: none;
  }
  
  .uni-stat-breadcrumb-on-phone {
    display: none;
  }
}
</style>