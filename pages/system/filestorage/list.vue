<!-- list.vue -->
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
          placeholder="搜索视频..." 
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
        collection="FileStorage" 
        field="fileId,userId,fileUrl,fileFormat,uploadTime"
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
            <uni-th align="center" sortable @sort-change="sortChange($event, 'fileId')">文件ID</uni-th>
            <uni-th align="center">用户昵称</uni-th>
            <uni-th align="center">视频URL</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'fileFormat')">格式</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'uploadTime')">上传时间</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in displayData" :key="item._id">
            <uni-td align="center">
              <view class="ellipsis" :title="item.fileId">{{item.fileId}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.nickname">
                {{item.nickname}}
              </view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.fileUrl">{{item.fileUrl}}</view>
            </uni-td>
            <uni-td align="center">{{item.fileFormat}}</uni-td>
            <uni-td align="center">{{formatTime(item.uploadTime)}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button 
                  @click="downloadVideo(item)" 
                  class="uni-button" 
                  size="mini"
                  type="default"
                >下载</button>
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
const $ = db.command.aggregate

// 表查询配置
const dbOrderBy = 'uploadTime desc' // 默认按上传时间倒序
const dbSearchFields = ['fileId', 'fileUrl'] // 支持模糊搜索的字段列表
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
      const timeNum = Number(timestamp)
      if (isNaN(timeNum)) return '-'
      
      const date = new Date(timeNum)
      if (isNaN(date.getTime())) return '-'
      
      const year = date.getFullYear()
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const day = String(date.getDate()).padStart(2, '0')
      const hours = String(date.getHours()).padStart(2, '0')
      const minutes = String(date.getMinutes()).padStart(2, '0')
      
      return `${year}-${month}-${day} ${hours}:${minutes}`
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
      const fileData = Array.isArray(data) ? data : data.data
      
      if (!fileData || fileData.length === 0) {
        this.$set(this, 'data', [])
        return
      }

      // 处理数据中的时间戳
      const processedData = fileData.map(item => ({
        ...item,
        uploadTime: item.uploadTime ? Number(item.uploadTime) : Date.now()
      }))

      // 获取用户ID并查询昵称
      const userIds = processedData.map(item => item.userId).filter(Boolean)
      await this.fetchUserNicknames(userIds)
      
      this.$set(this, 'data', processedData)
    },

    downloadVideo(item) {
      if (!item.fileUrl) {
        uni.showToast({
          title: '下载链接不可用',
          icon: 'none'
        })
        return
      }

      uni.showLoading({
        title: '准备下载...'
      })

      uni.downloadFile({
        url: item.fileUrl,
        success: (res) => {
          if (res.statusCode === 200) {
            uni.hideLoading()
            uni.openDocument({
              filePath: res.tempFilePath,
              success: function(res) {
                console.log('文件打开成功')
              },
              fail: function(error) {
                console.error('打开文件失败:', error)
                uni.showToast({
                  title: '打开文件失败',
                  icon: 'none'
                })
              }
            })
          }
        },
        fail: (error) => {
          console.error('下载失败:', error)
          uni.hideLoading()
          uni.showToast({
            title: '下载失败',
            icon: 'none'
          })
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
          content: '是否确认删除选中的视频？',
          success: async (res) => {
            if (res.confirm) {
              const promises = items.map(id => 
                db.collection('FileStorage').doc(id).remove()
              )
              await Promise.all(promises)
              
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
          content: '是否确认删除该视频？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('FileStorage').doc(id).remove()
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