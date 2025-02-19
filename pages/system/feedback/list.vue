<!-- feedback.vue -->
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
          placeholder="搜索反馈..." 
        />
        <button 
          class="uni-button hide-on-phone" 
          type="default" 
          size="mini"
          @click="search"
        >搜索</button>
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
        collection="Feedback" 
        field="feedbackId,userId,submitTime,content,status,processTime,adminId,contact,feedbacktype"
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
            <uni-th align="center" sortable @sort-change="sortChange($event, 'feedbackId')">反馈ID</uni-th>
            <uni-th align="center">用户昵称</uni-th>
            <uni-th align="center">联系方式</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'feedbacktype')">反馈类型</uni-th>
            <uni-th align="center">反馈内容</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'submitTime')">提交时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'processTime')">处理时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'status')">状态</uni-th>
            <uni-th align="center">处理人</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in displayData" :key="item._id">
            <uni-td align="center">
              <view class="ellipsis" :title="item.feedbackId">{{item.feedbackId}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.nickname">
                {{item.nickname}}
              </view>
            </uni-td>
            <uni-td align="center">{{item.contact}}</uni-td>
            <uni-td align="center">{{item.feedbacktype}}</uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.content">{{item.content}}</view>
            </uni-td>
            <uni-td align="center">{{item.formattedSubmitTime}}</uni-td>
            <uni-td align="center">{{item.formattedProcessTime}}</uni-td>
            <uni-td align="center">
              <uni-tag :text="item.status === 'pending' ? '待处理' : '已处理'" 
                      :type="item.status === 'pending' ? 'warning' : 'success'" />
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.adminNickname">
                {{item.adminNickname}}
              </view>
            </uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button 
                  v-if="item.status === 'pending'"
                  @click="handleProcess(item)" 
                  class="uni-button" 
                  size="mini"
                  type="primary"
                >处理</button>
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
import { store } from '../../../uni_modules/uni-id-pages/common/store.js'

const db = uniCloud.database()
const dbCmd = db.command
const $ = db.command.aggregate

// 表查询配置
const dbOrderBy = 'submitTime desc' // 默认按提交时间倒序
const dbSearchFields = ['feedbackId', 'content', 'contact'] // 支持模糊搜索的字段列表
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
      loading: false,
      error: { message: '' },
      data: [],
      userMap: new Map(),
      adminMap: new Map()
    }
  },

  computed: {
    displayData() {
      return this.data.map(item => ({
        ...item,
        nickname: this.userMap.get(item.userId) || '未知用户',
        adminNickname: this.adminMap.get(item.adminId) || '-',
        formattedSubmitTime: this.formatTime(item.submitTime),
        formattedProcessTime: item.processTime ? this.formatTime(item.processTime) : '未处理'
      }))
    }
  },

  mounted() {
    this.loadData()
  },

  methods: {
    formatTime(timestamp) {
      if (!timestamp) return '-'
      const timeNum = Number(timestamp)
      if (isNaN(timeNum)) return '-'
      
      const date = new Date(timeNum)
      if (isNaN(date.getTime())) return '-'
      
      const year = date.getFullYear()
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const day = String(date.getDate()).padStart(2, '0')
      const hours = String(date.getHours()).padStart(2, '0')
      const minutes = String(date.getMinutes()).padStart(2, '0')
      const seconds = String(date.getSeconds()).padStart(2, '0')
      
      return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
    },

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
      console.log('当前管理员:', store.userInfo)
      console.log('加载的数据:', data)
      const feedbackData = Array.isArray(data) ? data : data.data
      
      if (!feedbackData || feedbackData.length === 0) {
        this.$set(this, 'data', [])
        return
      }

      const processedData = feedbackData.map(item => ({
        ...item,
        submitTime: item.submitTime ? Number(item.submitTime) : Date.now(),
        processTime: item.processTime ? Number(item.processTime) : null
      }))

      const userIds = [...new Set(processedData.map(item => item.userId))]
      const adminIds = [...new Set(processedData.map(item => item.adminId).filter(Boolean))]
      
      try {
        if (userIds.length > 0) {
          const userResult = await db.collection('User')
            .where({
              userId: dbCmd.in(userIds)
            })
            .field('userId,nickname')
            .get()
          
          this.userMap = new Map(
            userResult.result.data.map(user => [user.userId, user.nickname])
          )
        }
        
        if (adminIds.length > 0) {
          const adminResult = await db.collection('Admin')
            .where({
              adminId: dbCmd.in(adminIds)
            })
            .field('adminId,nickname')
            .get()
          
          this.adminMap = new Map(
            adminResult.result.data.map(admin => [admin.adminId, admin.nickname])
          )
        }
        
        this.$set(this, 'data', processedData)
        
      } catch (error) {
        console.error('获取用户或管理员信息失败:', error)
        this.$set(this, 'data', processedData)
      }
    },

    async handleProcess(item) {
      if (!item || !item._id) return
      
      const adminId = store.userInfo._id
      const adminName = store.userInfo.username
      
      if (!adminId || !adminName) {
        uni.showToast({
          title: '请先登录',
          icon: 'none'
        })
        return
      }
      
      uni.showModal({
        title: '提示',
        content: '确认处理该反馈？',
        success: async (res) => {
          if (res.confirm) {
            try {
              const currentTime = Date.now()
              
              await db.collection('Feedback').doc(item._id).update({
                status: 'processed',
                processTime: currentTime,
                adminId: adminId
              })
              
              this.loadData(true)
              
              uni.showToast({
                title: '处理成功',
                icon: 'success'
              })
            } catch (error) {
              console.error('处理失败:', error)
              uni.showToast({
                title: '处理失败',
                icon: 'error'
              })
            }
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
          content: '是否确认删除选中的反馈？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('Feedback').where({
                _id: dbCmd.in(items)
              }).remove()
              
              this.selectedIndexs = []
              this.$refs.table?.clearSelection()
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
          content: '是否确认删除该反馈？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('Feedback').doc(id).remove()
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
}
</style>