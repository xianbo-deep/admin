<!-- agent.vue -->
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
          placeholder="搜索Agent..." 
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
        >新增</button>
        <button 
          class="uni-button" 
          type="warn" 
          size="mini" 
          :disabled="!selectedIndexs.length"
          @click="delTable"
        >批量删除</button>
      </view>
    </view>

    <view class="uni-container">
      <unicloud-db 
        ref="udb" 
        collection="Agent" 
        field="agentId,identifier,description,status,method"
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
          type="selection" 
          @selection-change="selectionChange"
        >
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'agentId')">Agent ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'identifier')">标识符</uni-th>
            <uni-th align="center">描述信息</uni-th>
            <uni-th align="center">请求方式</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'status')">状态</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in data" :key="item._id">
            <uni-td align="center">{{item.agentId}}</uni-td>
            <uni-td align="center">{{item.identifier}}</uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.description">{{item.description || '-'}}</view>
            </uni-td>
            <uni-td align="center">{{item.method || 'POST'}}</uni-td>
            <uni-td align="center">
              <uni-tag :text="item.status === 'normal' ? '正常' : '禁用'" 
                      :type="item.status === 'normal' ? 'success' : 'error'" />
            </uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button 
                  @click="toggleStatus(item)" 
                  class="uni-button" 
                  size="mini"
                  :type="item.status === 'normal' ? 'warning' : 'primary'"
                >{{item.status === 'normal' ? '禁用' : '启用'}}</button>
                <button 
                  @click="navigateTo('./edit?id=' + item._id, false)" 
                  class="uni-button" 
                  size="mini"
                  type="primary"
                >编辑</button>
                <button 
                  @click="confirmDelete(item._id)" 
                  class="uni-button" 
                  size="mini"
                  type="warn"
                >删除</button>
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
const dbOrderBy = 'agentId asc'
const dbSearchFields = ['agentId', 'identifier', 'description']
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
        pageCurrent
      },
      data: []
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

    onqueryload(data) {
      this.data = Array.isArray(data) ? data : data.data || []
    },

    async toggleStatus(item) {
      if (!item || !item._id) return
      
      const newStatus = item.status === 'normal' ? 'disabled' : 'normal'
      const actionText = newStatus === 'normal' ? '启用' : '禁用'
      
      uni.showModal({
        title: '提示',
        content: `确认${actionText}该Agent？`,
        success: async (res) => {
          if (res.confirm) {
            try {
              await db.collection('Agent').doc(item._id).update({
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

    selectionChange(e) {
      if (!e || !e.detail || !Array.isArray(e.detail.index)) {
        return
      }
      this.selectedIndexs = e.detail.index
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
                  transaction.collection('Agent').doc(id).remove()
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

    async confirmDelete(id) {
      if (!id) return
      
      uni.showModal({
        title: '提示',
        content: '是否确认删除该Agent？',
        success: async (res) => {
          if (res.confirm) {
            try {
              await db.collection('Agent').doc(id).remove()
              this.loadData(true)
              
              uni.showToast({
                title: '删除成功',
                icon: 'success'
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
      })
    }
  }
}
</script>

<style>
.fix-top-window {
  padding: 0;
  margin: 0;
}

.uni-container {
  padding: 15px;
}

.uni-header {
  padding: 0 15px;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #e5e5e5;
  background-color: #fff;
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
  min-width: 200px;
}

.ellipsis {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  max-width: 200px;
  display: inline-block;
}

.uni-pagination-box {
  margin-top: 20px;
}

@media screen and (max-width: 768px) {
  .hide-on-phone {
    display: none;
  }
  
  .uni-header {
    flex-direction: column;
    padding: 10px;
    gap: 10px;
  }
  
  .uni-group {
    width: 100%;
    flex-wrap: wrap;
  }
  
  .uni-search {
    min-width: 120px;
  }
  
  .ellipsis {
    max-width: 150px;
  }
}
</style>