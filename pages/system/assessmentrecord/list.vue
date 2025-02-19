<!-- AssessmentRecords.vue -->
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
          placeholder="搜索记录..." 
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
        collection="AssessmentRecord" 
        field="recordId,userId,name,score,assessmentTime,duration,totaltoken,uploadUrl"
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
            <uni-th align="center" sortable @sort-change="sortChange($event, 'recordId')">记录ID</uni-th>
            <uni-th align="center">用户昵称</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'name')">主播姓名</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'score')">得分</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'assessmentTime')">评估时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'duration')">持续时间</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'totaltoken')">Token数</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in displayData" :key="index">
            <uni-td align="center">{{item.recordId || '-'}}</uni-td>
            <uni-td align="center">{{item.nickname || '-'}}</uni-td>
            <uni-td align="center">{{item.name || '-'}}</uni-td>
            <uni-td align="center">{{item.score ? item.score.toFixed(2) : '-'}}</uni-td>
            <uni-td align="center">{{formatTime(item.assessmentTime)}}</uni-td>
            <uni-td align="center">{{formatDuration(item.duration)}}</uni-td>
            <uni-td align="center">{{item.totaltoken ? item.totaltoken.toFixed(0) : '0'}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button 
                  @click="navigateTo('./details?id=' + item._id, true)" 
                  class="uni-button" 
                  size="mini" 
                  type="primary"
                >详情</button>
                <button 
                  @click="downloadFile(item)" 
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
            v-model:current="pagination.current"
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

// 表格配置
const dbOrderBy = 'assessmentTime desc'
const dbSearchFields = ['recordId', 'name']
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
      
      const year = date.getFullYear()
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const day = String(date.getDate()).padStart(2, '0')
      const hours = String(date.getHours()).padStart(2, '0')
      const minutes = String(date.getMinutes()).padStart(2, '0')
      const seconds = String(date.getSeconds()).padStart(2, '0')
      
      return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
    },

    formatDuration(minutes) {
      if (!minutes && minutes !== 0) return '-'
      const hours = Math.floor(minutes / 60)
      const remainingMinutes = Math.floor(minutes % 60)
      if (hours > 0) {
        return `${hours}小时${remainingMinutes > 0 ? remainingMinutes + '分钟' : ''}`
      }
      return `${remainingMinutes}分钟`
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
      this.$refs.udb?.loadData({
        clear
      })
    },

    onPageChanged(e) {
      this.selectedIndexs = []
      this.$refs.table?.clearSelection()
      this.$refs.udb?.loadData({
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
      this.$refs.table?.clearSelection()
      this.$nextTick(() => {
        this.$refs.udb?.loadData()
      })
    },

    async onqueryload(data) {
      const assessmentData = Array.isArray(data) ? data : data.data
      
      if (!assessmentData || assessmentData.length === 0) {
        this.data = []
        return
      }

      const userIds = assessmentData.map(item => item.userId).filter(Boolean)
      await this.fetchUserNicknames(userIds)
      
      this.data = assessmentData
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

    async downloadFile(item) {
      if (!item.uploadUrl) {
        uni.showToast({
          title: '文件链接不存在',
          icon: 'none'
        })
        return
      }

      try {
        uni.showLoading({
          title: '下载中...'
        })

        const result = await uniCloud.getTempFileURL({
          fileList: [item.uploadUrl]
        })

        const tempUrl = result.fileList[0].tempFileURL
        
        uni.downloadFile({
          url: tempUrl,
          success: (res) => {
            if (res.statusCode === 200) {
              uni.saveFile({
                tempFilePath: res.tempFilePath,
                success: function(res) {
                  uni.showToast({
                    title: '下载成功',
                    icon: 'success'
                  })
                }
              })
            }
          },
          complete: () => {
            uni.hideLoading()
          }
        })
      } catch (error) {
        console.error('下载失败:', error)
        uni.showToast({
          title: '下载失败',
          icon: 'error'
        })
        uni.hideLoading()
      }
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
              await db.collection('AssessmentRecord').where({
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
      if (!e?.detail?.index) {
        this.selectedIndexs = []
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
              await db.collection('AssessmentRecord').doc(id).remove()
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