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
          placeholder="搜索选择..." 
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
        collection="Select" 
        field="selectId,fileId,userId,packageName,packageId,timestamp"
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
            <uni-th align="center" sortable @sort-change="sortChange($event, 'selectId')">选择ID</uni-th>
            <uni-th align="center">用户昵称</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'fileId')">文件ID</uni-th>
            <uni-th align="center">套餐名称</uni-th>
            <uni-th align="center">套餐ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'timestamp')">提交时间</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in displayData" :key="item._id">
            <uni-td align="center">{{item.selectId}}</uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.nickname">
                {{item.nickname}}
              </view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.fileId">
                {{item.fileId || '-'}}
              </view>
            </uni-td>
            <uni-td align="center">{{item.packageName}}</uni-td>
            <uni-td align="center">{{item.packageId || '-'}}</uni-td>
            <uni-td align="center">{{formatTime(item.timestamp)}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
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
const dbOrderBy = 'timestamp desc' // 默认按时间倒序
const dbSearchFields = ['selectId', 'fileId', 'packageName'] // 支持模糊搜索的字段列表
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
      userMap: new Map()
    }
  },

  computed: {
    displayData() {
      return this.data.map(item => ({
        ...item,
        nickname: this.userMap.get(item.userId) || '未知用户'
      }))
    }
  },

  mounted() {
    this.loadData()
  },

  methods: {
    formatTime(timestamp) {
      if (!timestamp) return '-'
      const date = new Date(timestamp)
      if (isNaN(date.getTime())) return '-'
      
      return date.toLocaleString()
    },

    async fetchUserNicknames(userIds) {
      if (!userIds.length) return
      
      try {
        const { result } = await db.collection('User')
          .where({
            userId: dbCmd.in([...new Set(userIds)])
          })
          .field('userId,nickname')
          .get()

        if (result && result.data) {
          this.userMap = new Map(
            result.data.map(user => [user.userId, user.nickname])
          )
        }
      } catch (error) {
        console.error('获取用户昵称失败:', error)
      }
    },

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
      const selectData = Array.isArray(data) ? data : data.data
      
      if (!selectData || selectData.length === 0) {
        this.$set(this, 'data', [])
        return
      }
      
      // 获取用户ID并查询昵称
      const userIds = selectData.map(item => item.userId).filter(Boolean)
      await this.fetchUserNicknames(userIds)
      
      this.$set(this, 'data', selectData)
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
              await db.collection('Select').where({
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
          content: '是否确认删除该记录？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('Select').doc(id).remove()
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