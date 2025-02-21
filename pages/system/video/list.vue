<template>
  <view class="fix-top-window">
    <view class="uni-header">
      <uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
      <view class="uni-group">
        <input class="uni-search" type="text" v-model="query" @confirm="search"
          placeholder="搜索视频..." />
        <button class="uni-button hide-on-phone" type="default" size="mini"
          @click="search">搜索</button>
        <button class="uni-button" type="primary" size="mini"
          @click="navigateTo('./add')">新增视频</button>
        <button class="uni-button" type="warn" size="mini" :disabled="!selectedIndexs.length"
          @click="delTable">批量删除</button>
      </view>
    </view>
    <view class="uni-container">
      <unicloud-db ref="udb" collection="Video" :where="where"
        page-data="replace" :getcount="true" :page-size="options.pageSize"
        :page-current="options.pageCurrent" :orderby="orderby" loadtime="manual"
        @load="onqueryload" v-slot:default="{ pagination, loading, error }">
        <uni-table ref="table" :loading="loading" :emptyText="error.message || '没有数据'" border stripe
          type="selection" @selection-change="selectionChange">
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'videoId')">视频ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'title')">视频标题</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'uploadTime')">上传时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'categoryId')">分类</uni-th>
            <uni-th align="center">视频描述</uni-th>
            <uni-th align="center">学习进度</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item,index) in displayData" :key="index">
            <uni-td align="center">{{item.videoId}}</uni-td>
            <uni-td align="center">{{item.title}}</uni-td>
            <uni-td align="center">{{formatDate(item.uploadTime)}}</uni-td>
            <uni-td align="center">{{getCategoryName(item.categoryId)}}</uni-td>
            <uni-td align="center">{{item.description || '-'}}</uni-td>
            <uni-td align="center">{{getProgressSummary(item.learningProgress)}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button @click="navigateTo('./preview?id=' + item._id, false)" class="uni-button" size="mini" 
                  type="default">预览</button>
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
const dbOrderBy = 'uploadTime desc' // 默认按上传时间降序
const dbSearchFields = ['videoId', 'title', 'description', 'categoryId'] // 支持模糊搜索的字段列表
// 分页配置
const pageSize = 10
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
      data: [],
      categoryList: [] // 存储分类数据
    }
  },

  computed: {
    displayData() {
      return this.data || []
    }
  },

  async mounted() {
    await this.loadCategoryList()
    this.loadData()
  },

  methods: {
    // 修改 formatDate 函数以确保正确处理时间戳
    formatDate(timestamp) {
      if (!timestamp) return '-'
      // 确保 timestamp 是数字类型
      const time = typeof timestamp === 'number' ? timestamp : Number(timestamp)
      // 创建日期对象
      const d = new Date(time)
      // 检查日期是否有效
      if (isNaN(d.getTime())) return '-'
      
      return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')} ${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}`
    },

    // 加载分类列表
    async loadCategoryList() {
      try {
        const res = await db.collection('Category').get()
        if (res.result && res.result.data) {
          this.categoryList = res.result.data
        }
      } catch (err) {
        console.error('获取分类列表失败:', err)
      }
    },

    // 获取分类名称
    getCategoryName(categoryId) {
      const category = this.categoryList.find(item => item.categoryId === categoryId)
      return category ? category.name : categoryId
    },

    getProgressSummary(progress) {
      if (!progress || !Array.isArray(progress)) return '暂无数据'
      const completed = progress.filter(p => p.completed).length
      return `${completed}/${progress.length}人完成`
    },

    previewVideo(url) {
      if (!url) return
      uni.navigateTo({
        url: `/pages/video/preview?url=${encodeURIComponent(url)}`
      })
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
      const videoData = Array.isArray(data) ? data : data.data
      
      if (!videoData || videoData.length === 0) {
        this.$set(this, 'data', [])
        return
      }
      
      this.$set(this, 'data', videoData)
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
          content: '是否确认删除选中的视频？',
          success: async (res) => {
            if (res.confirm) {
              const transaction = await db.startTransaction()
              try {
                // 先删除视频文件
                const videosToDelete = items.map(id => 
                  this.data.find(item => item._id === id)
                ).filter(item => item && item.url)
                
                if (videosToDelete.length > 0) {
                  await uniCloud.deleteFile({
                    fileList: videosToDelete.map(item => item.url)
                  })
                }

                // 删除数据库记录
                const promises = items.map(id => 
                  transaction.collection('Video').doc(id).remove()
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
          content: '是否确认删除该视频？',
          success: async (res) => {
            if (res.confirm) {
              const video = this.data.find(item => item._id === id)
              if (video && video.url) {
                await uniCloud.deleteFile({
                  fileList: [video.url]
                })
              }
              
              await db.collection('Video').doc(id).remove()
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