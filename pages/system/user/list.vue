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
          placeholder="搜索用户..." 
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
            <uni-th align="center" sortable @sort-change="sortChange($event, 'userId')">用户ID</uni-th>
            <uni-th align="center">头像</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'nickname')">昵称</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'phone')">手机号码</uni-th>
            <uni-th align="center" filter-type="select" :filter-data="statusOptions"
              @filter-change="filterChange($event, 'status')">状态</uni-th>
            <uni-th align="center" filter-type="select" :filter-data="memberTypeOptions"
              @filter-change="filterChange($event, 'membertype')">会员类型</uni-th>
            <uni-th align="center" filter-type="select" :filter-data="memberStatusOptions"
              @filter-change="filterChange($event, 'memberStatus')">会员状态</uni-th>
            <uni-th align="center">剩余天数/次数</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'memberExpireTime')">会员到期时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'email')">邮箱</uni-th>
            <uni-th align="center">简介</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'registerTime')">注册时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'lastLoginTime')">最后登录</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          
          <uni-tr v-for="(item, index) in data" :key="index">
            <uni-td align="center">{{item.userId}}</uni-td>
            <uni-td align="center">
              <image 
                class="user-avatar" 
                :src="item.avatarUrl || '/static/avatar.png'" 
                mode="aspectFill"
                @error="onAvatarError(item)"
                @click="downloadAvatar(item.avatarUrl, item.nickname)"
              />
            </uni-td>
            <uni-td align="center">{{item.nickname}}</uni-td>
            <uni-td align="center">{{item.phone}}</uni-td>
            <uni-td align="center">
              <uni-tag :type="getStatusType(item.status)" :text="getStatusText(item.status)"/>
            </uni-td>
            <uni-td align="center">
              <uni-tag :type="getMemberTypeTag(item.membertype)" :text="getMemberTypeText(item.membertype)"/>
            </uni-td>
            <uni-td align="center">
              <uni-tag :type="getMemberStatusTag(item.memberStatus)" :text="getMemberStatusText(item.memberStatus)"/>
            </uni-td>
            <uni-td align="center">
              <template v-if="item.membertype === 'times'">
                {{item.remainingTimes || 0}}次
              </template>
              <template v-else-if="item.membertype === 'daily' || item.membertype === 'monthly'">
                {{item.remainingDays || 0}}天
              </template>
              <template v-else>
                -
              </template>
            </uni-td>
            <uni-td align="center">
              {{item.memberExpireTime ? formatDateTime(item.memberExpireTime) : '-'}}
            </uni-td>
            <uni-td align="center">
              <uni-link :href="'mailto:' + item.email" :text="item.email"></uni-link>
            </uni-td>
            <uni-td align="center">
              <view class="bio-cell" :title="item.bio">{{item.bio || '-'}}</view>
            </uni-td>
            <uni-td align="center">
              {{formatDateTime(item.registerTime)}}
            </uni-td>
            <uni-td align="center">
              {{formatDateTime(item.lastLoginTime)}}
            </uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button @click="navigateTo('./edit?id=' + item.userId, false)" 
                  class="uni-button" size="mini" type="primary">编辑</button>
                <button @click="confirmDelete(item.userId)" 
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
// 数据库查询配置
const dbOrderBy = 'lastLoginTime desc' // 默认按最后登录时间降序
const dbSearchFields = ['userId', 'nickname', 'phone', 'email', 'bio'] // 支持搜索的字段
const pageSize = 20 // 默认分页大小
const pageCurrent = 1 // 默认当前页

// 排序方式映射
const orderByMapping = {
  "ascending": "asc",
  "descending": "desc"
}

export default {
  data() {
    return {
      collection: 'User', // 集合名称
      query: '', // 搜索关键词
      where: '', // 查询条件
      orderby: dbOrderBy,
      selectedIndexs: [], // 选中的行索引
      options: {
        pageSize,
        pageCurrent
      },
      // 用户状态选项
      statusOptions: [
        { text: "正常", value: "active" },
        { text: "未激活", value: "inactive" },
        { text: "已禁用", value: "banned" }
      ],
      // 会员类型选项
      memberTypeOptions: [
        { text: "非会员", value: "none" },
        { text: "日卡", value: "daily" },
        { text: "月卡", value: "monthly" },
        { text: "次卡", value: "times" }
      ],
      // 会员状态选项
      memberStatusOptions: [
        { text: "非会员", value: "none" },
        { text: "有效", value: "active" },
        { text: "已过期", value: "expired" }
      ]
    }
  },
  
  onReady() {
    this.$refs.udb.loadData()
  },
  
  methods: {
    // 头像加载失败处理
    onAvatarError(item) {
      // 设置默认头像
      item.avatarUrl = '/static/avatar.png'
    },

    // 下载头像
    downloadAvatar(url, nickname) {
      if (!url || url === '/static/avatar.png') {
        uni.showToast({
          title: '无头像可下载',
          icon: 'none'
        })
        return
      }
      
      // 创建a标签下载
      const a = document.createElement('a')
      a.href = url
      a.download = `${nickname || 'avatar'}.png` // 使用用户昵称作为文件名
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)
      
      uni.showToast({
        title: '开始下载',
        icon: 'success'
      })
    },

    // 格式化毫秒级时间戳为北京时间
    formatDateTime(timestamp) {
      if (!timestamp) {
        return '';
      }
      
      try {
        // 直接使用毫秒级时间戳创建Date对象
        const date = new Date(timestamp);
        
        // 使用toLocaleString转换为北京时间
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
    
    // 获取会员类型标签样式
    getMemberTypeTag(type) {
      const types = {
        none: 'default',
        daily: 'primary',
        monthly: 'success',
        times: 'warning'
      }
      return types[type] || 'default'
    },
    
    // 获取会员类型显示文本
    getMemberTypeText(type) {
      const texts = {
        none: '非会员',
        daily: '日卡',
        monthly: '月卡',
        times: '次卡'
      }
      return texts[type] || type
    },
    
    // 获取会员状态标签样式
    getMemberStatusTag(status) {
      const types = {
        none: 'default',
        active: 'success',
        expired: 'error'
      }
      return types[status] || 'default'
    },
    
    // 获取会员状态显示文本
    getMemberStatusText(status) {
      const texts = {
        none: '非会员',
        active: '有效',
        expired: '已过期'
      }
      return texts[status] || status
    },
    
    onqueryload(data) {
      // 数据加载完成后的处理
      if (data) {
        data.forEach(item => {
          // 处理特殊字段
          item.bio = item.bio || ''
          // 确保字段存在
          item.membertype = item.membertype || 'none'
          item.memberStatus = item.memberStatus || 'none'
          item.remainingDays = item.remainingDays || 0
          item.remainingTimes = item.remainingTimes || 0
          // 确保时间戳存在
          item.registerTime = item.registerTime || 0
          item.lastLoginTime = item.lastLoginTime || 0
          item.memberStartTime = item.memberStartTime || 0
          item.memberExpireTime = item.memberExpireTime || 0
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
      return this.selectedIndexs.map(i => dataList[i].userId)
    },
    
    // 批量删除
    delTable() {
      uni.showModal({
        title: '确认删除',
        content: '是否删除选中的用户？',
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
        content: '是否删除该用户？',
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

/* 头像样式 */
.user-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
  border: 1px solid #eee;
  cursor: pointer;
  transition: all 0.3s;
}

.user-avatar:hover {
  transform: scale(1.1);
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
}

/* 用户简介单元格样式 */
.bio-cell {
  max-width: 200px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding: 0 8px;
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