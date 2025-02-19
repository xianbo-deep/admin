<template>
  <view class="fix-top-window">
    <view class="uni-header">
      <uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
      <view class="uni-group">
        <input class="uni-search" type="text" v-model="query" @confirm="search"
          placeholder="搜索指标..." />
        <button class="uni-button hide-on-phone" type="default" size="mini"
          @click="search">搜索</button>
        <button class="uni-button" type="primary" size="mini"
          @click="navigateTo('./add')">新增</button>
        <button class="uni-button" type="warn" size="mini" :disabled="!selectedIndexs.length"
          @click="delTable">批量删除</button>
      </view>
    </view>
    <view class="uni-container">
      <unicloud-db ref="udb" collection="Metric" :where="where"
        page-data="replace" :getcount="true" :page-size="options.pageSize"
        :page-current="options.pageCurrent" :orderby="orderby" loadtime="manual"
        @load="onqueryload" v-slot:default="{ pagination, loading, error }">
        <uni-table ref="table" :loading="loading" :emptyText="error.message || '没有数据'" border stripe
          type="selection" @selection-change="selectionChange">
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'metricId')">指标ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'packageId')">套餐ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'agentId')">代理ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'metricname')">指标名称</uni-th>
            <uni-th align="center">描述</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item,index) in displayData" :key="index">
            <uni-td align="center">{{item.metricId}}</uni-td>
            <uni-td align="center">{{item.packageId}}</uni-td>
            <uni-td align="center">{{item.agentId}}</uni-td>
            <uni-td align="center">{{item.metricname}}</uni-td>
            <uni-td align="center">{{item.description}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button @click="navigateTo('./edit?id=' + item._id, false)" class="uni-button" size="mini"
                  type="primary">编辑</button>
                <button @click="confirmDelete(item._id)" class="uni-button" size="mini"
                  type="warn">删除</button>
              </view>
            </uni-td>
          </uni-tr>
        </uni-table>
        <view class="uni-pagination-box">
          <uni-pagination show-icon show-page-size :page-size="pagination.size" v-model="pagination.current"
            :total="pagination.count" @change="onPageChanged" @pageSizeChange="changeSize" />
        </view>
      </unicloud-db>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database()
const dbCmd = db.command

// 表查询配置
const dbOrderBy = 'metricId asc' // 默认按指标ID升序
const dbSearchFields = ['metricId', 'packageId', 'agentId', 'metricname', 'description'] // 支持模糊搜索的字段列表
// 分页配置
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
      selectedIndexs: [],
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
      return this.data || []
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
      this.selectedIndexs = []
      this.$refs.table.clearSelection()
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
      this.$refs.table.clearSelection()
      this.$nextTick(() => {
        this.$refs.udb.loadData()
      })
    },

    async onqueryload(data) {
      const metricData = Array.isArray(data) ? data : data.data
      
      if (!metricData || metricData.length === 0) {
        this.$set(this, 'data', [])
        return
      }
      
      this.$set(this, 'data', metricData)
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
    },

    selectedItems() {
      if (!this.data || !this.selectedIndexs) {
        return []
      }
      return this.selectedIndexs.map(i => this.data[i]._id).filter(Boolean)
    },

    async delTable() {
      const items = this.selectedItems()
      if (!items.length) return
      
      try {
        uni.showModal({
          title: '提示',
          content: '是否确认删除选中的记录？',
          success: async (res) => {
            if (res.confirm) {
              const transaction = await db.startTransaction()
              try {
                const promises = items.map(id => 
                  transaction.collection('Metric').doc(id).remove()
                )
                await Promise.all(promises)
                await transaction.commit()
                
                this.selectedIndexs = []
                this.$refs.table?.clearSelection()
                this.loadData(true)
                
                uni.showToast({
                  title: '删除成功',
                  icon: 'success'
                })
              } catch (error) {
                await transaction.rollback()
                throw error
              }
            }
          }
        })
      } catch (error) {
        console.error('删除失败:', error)
        uni.showToast({
          title: '删除失败',
          icon: 'error' 
        })
      }
    },
    
    selectionChange(e) {
      if (!e || !e.detail || !Array.isArray(e.detail.index)) {
        return
      }
      this.selectedIndexs = e.detail.index
    },

    async confirmDelete(id) {
      if (!id) return
      
      try {
        uni.showModal({
          title: '提示',
          content: '是否确认删除该记录？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('Metric').doc(id).remove()
              this.loadData(true)
              
              uni.showToast({
                title: '删除成功',
                icon: 'success'
              })
            }
          }
        })
      } catch (error) {
        console.error('删除失败:', error)
        uni.showToast({
          title: '删除失败', 
          icon: 'error'
        })
      }
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
  flex-direction: row;
  align-items: center;
  gap: 10rpx;
}

.uni-search {
  width: 300rpx;
  padding: 10rpx;
  border: 1px solid #e5e5e5;
  border-radius: 4rpx;
}

.uni-button {
  margin-left: 10rpx;
}

.uni-container {
  padding: 20rpx;
}

.uni-pagination-box {
  margin-top: 20rpx;
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