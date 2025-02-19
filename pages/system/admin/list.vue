# pages/adminManagement/index.vue
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
          placeholder="搜索管理员..." 
        />
        <button class="uni-button" type="primary" size="mini" @click="search">搜索</button>
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
        :collection="collection" 
        :where="where"
        page-data="replace" 
        :orderby="orderby" 
        :getcount="true" 
        :page-size="options.pageSize"
        :page-current="options.pageCurrent" 
        v-slot:default="{ data, pagination, loading, error }"
        :options="options" 
        loadtime="manual" 
        @load="onqueryload"
      >
        <uni-table 
          ref="table" 
          :loading="loading" 
          :emptyText="error.message || '暂无数据'" 
          border 
          stripe
          type="selection" 
          @selection-change="selectionChange"
        >
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'adminId')">管理员ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'account')">账号</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'nickname')">管理员名称</uni-th>
            <uni-th align="center" filter-type="select" :filter-data="statusOptions"
              @filter-change="filterChange($event, 'status')">状态</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'registerTime')">注册时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'lastLoginTime')">最后登录</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          
          <uni-tr v-for="(item, index) in data" :key="index">
            <uni-td align="center">{{item.adminId}}</uni-td>
            <uni-td align="center">{{item.account}}</uni-td>
            <uni-td align="center">{{item.nickname}}</uni-td>
            <uni-td align="center">
              <uni-tag :type="getStatusType(item.status)" :text="getStatusText(item.status)"/>
            </uni-td>
            <uni-td align="center">
              {{formatDateTime(item.registerTime)}}
            </uni-td>
            <uni-td align="center">
              {{formatDateTime(item.lastLoginTime)}}
            </uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button @click="navigateTo('./edit?id=' + item.adminId, false)" 
                  class="uni-button" size="mini" type="primary">编辑</button>
                <button @click="confirmDelete(item.adminId)" 
                  class="uni-button" size="mini" type="warn">删除</button>
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
const dbOrderBy = 'lastLoginTime desc'
const dbSearchFields = ['adminId', 'account', 'nickname']
const pageSize = 20
const pageCurrent = 1

const orderByMapping = {
  "ascending": "asc",
  "descending": "desc"
}

export default {
  data() {
    return {
      collection: 'Admin',
      query: '',
      where: '',
      orderby: dbOrderBy,
      selectedIndexs: [],
      options: {
        pageSize,
        pageCurrent
      },
      statusOptions: [
        { text: "正常", value: "active" },
        { text: "未激活", value: "inactive" },
        { text: "已禁用", value: "banned" }
      ]
    }
  },
  
  onReady() {
    this.$refs.udb.loadData()
  },
  
  methods: {
    // 格式化时间戳为北京时间
    formatDateTime(timestamp) {
      if (!timestamp) {
        return '';
      }
      
      try {
        const date = new Date(timestamp);
        return date.toLocaleString('zh-CN', { 
          timeZone: 'Asia/Shanghai',
          year: 'numeric',
          month: '2-digit',
          day: '2-digit',
          hour: '2-digit',
          minute: '2-digit',
          second: '2-digit',
          hour12: false
        }).replace(/\//g, '-');
      } catch (error) {
        console.error('时间格式化错误:', error);
        return '';
      }
    },

    // 获取状态标签类型
    getStatusType(status) {
      const types = {
        active: 'success',
        inactive: 'warning',
        banned: 'error'
      }
      return types[status] || 'default'
    },

    // 获取状态显示文本
    getStatusText(status) {
      const texts = {
        active: '正常',
        inactive: '未激活',
        banned: '已禁用'
      }
      return texts[status] || status
    },
    
    onqueryload(data) {
      if (data) {
        data.forEach(item => {
          item.registerTime = item.registerTime || 0
          item.lastLoginTime = item.lastLoginTime || 0
        })
      }
    },
    
    // 构建搜索条件
    getWhere() {
      const query = this.query.trim()
      if (!query) {
        return ''
      }
      const queryRe = new RegExp(query, 'i')
      
      return db.command.or(
        dbSearchFields.map(name => ({
          [name]: queryRe
        }))
      )
    },
    
    // 执行搜索
    search() {
      const newWhere = this.getWhere()
      this.where = newWhere
      this.$nextTick(() => {
        this.loadData()
      })
    },
    
    // 加载数据
    loadData(clear = true) {
      this.$refs.udb.loadData({
        clear
      })
    },
    
    // 修改每页显示数量
    changeSize(pageSize) {
      this.options.pageSize = pageSize
      this.options.pageCurrent = 1
      this.$nextTick(() => {
        this.loadData()
      })
    },
    
    // 页码改变
    onPageChanged(e) {
      this.selectedIndexs.length = 0
      this.$refs.table.clearSelection()
      this.$refs.udb.loadData({
        current: e.current
      })
    },
    
    // 页面跳转
    navigateTo(url, clear = true) {
      uni.navigateTo({
        url,
        events: {
          refreshData: () => {
            this.loadData(clear)
          }
        }
      })
    },
    
    // 获取选中项的ID
    selectedItems() {
      const dataList = this.$refs.udb.dataList
      return this.selectedIndexs.map(i => dataList[i].adminId)
    },
    
    // 批量删除
    delTable() {
      uni.showModal({
        title: '确认删除',
        content: '是否删除选中的管理员？',
        success: (res) => {
          if (res.confirm) {
            this.$refs.udb.remove(this.selectedItems(), {
              success: () => {
                this.$refs.table.clearSelection()
                uni.showToast({
                  title: '删除成功',
                  icon: 'success'
                })
              },
              fail: (err) => {
                uni.showToast({
                  title: '删除失败',
                  icon: 'error'
                })
                console.error(err)
              }
            })
          }
        }
      })
    },
    
    // 选择改变
    selectionChange(e) {
      this.selectedIndexs = e.detail.index
    },
    
    // 确认删除
    confirmDelete(id) {
      uni.showModal({
        title: '确认删除',
        content: '是否删除该管理员？',
        success: (res) => {
          if (res.confirm) {
            this.$refs.udb.remove(id, {
              success: () => {
                this.$refs.table.clearSelection()
                uni.showToast({
                  title: '删除成功',
                  icon: 'success'
                })
              },
              fail: (err) => {
                uni.showToast({
                  title: '删除失败',
                  icon: 'error'
                })
                console.error(err)
              }
            })
          }
        }
      })
    },
    
    // 排序改变
    sortChange(e, name) {
      if (e.order) {
        this.orderby = name + ' ' + orderByMapping[e.order]
      } else {
        this.orderby = ''
      }
      this.$refs.table.clearSelection()
      this.$nextTick(() => {
        this.$refs.udb.loadData()
      })
    },
    
    // 筛选改变
    filterChange(e, name) {
      if (e.filter && e.filter.length) {
        this.where = {
          [name]: e.filter[0]
        }
      } else {
        this.where = ''
      }
      
      this.$nextTick(() => {
        this.$refs.udb.loadData()
      })
    }
  }
}
</script>

<style>
.uni-header {
  padding: 0 15px;
  display: flex;
  height: 55px;
  align-items: center;
  border-bottom: 1px solid #eee;
}

.uni-group {
  display: flex;
  align-items: center;
}

.uni-search {
  height: 30px;
  line-height: 30px;
  font-size: 14px;
  border: 1px solid #eee;
  border-radius: 4px;
  padding: 0 10px;
  margin-right: 10px;
  width: 200px;
}

.uni-button {
  margin: 0 5px;
}

.uni-container {
  padding: 15px;
}

.uni-pagination-box {
  margin-top: 20px;
}

/* 表格通用样式 */
.uni-table-td {
  padding: 8px;
}

.uni-table-th {
  padding: 12px 8px;
  font-weight: bold;
  background-color: #f8f8f8;
}

/* 按钮组样式 */
.operation-group {
  display: flex;
  justify-content: center;
  gap: 8px;
}

.uni-stat-breadcrumb-on-phone {
  padding: 0 20px !important;
  border-bottom: 1px #f5f5f5 solid;
}
</style>