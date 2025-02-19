<!-- metric-results.vue -->
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
          placeholder="搜索评估结果..." 
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
        collection="MetricResult" 
        field="resultId,recordId,evaluationTime,metrics"
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
            <uni-th align="center" sortable @sort-change="sortChange($event, 'resultId')">评估ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'recordId')">记录ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'evaluationTime')">评估时间</uni-th>
            <uni-th align="center">指标数量</uni-th>
            <uni-th align="center">平均得分</uni-th>
            <uni-th align="center">总Token</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in displayData" :key="item._id">
            <uni-td align="center">
              <view class="ellipsis" :title="item.resultId">{{item.resultId}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.recordId">{{item.recordId}}</view>
            </uni-td>
            <uni-td align="center">{{item.formattedEvaluationTime}}</uni-td>
            <uni-td align="center">{{item.metrics.length}}</uni-td>
            <uni-td align="center">{{item.averageScore.toFixed(2)}}</uni-td>
            <uni-td align="center">{{item.totalTokens.toFixed(0)}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button 
                  @click="navigateTo('./details?id=' + item._id)" 
                  class="uni-button" 
                  size="mini"
                  type="primary"
                >详情</button>
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
const dbOrderBy = 'evaluationTime desc' // 排序字段
const dbSearchFields = ['resultId', 'recordId'] // 支持模糊搜索的字段列表
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
      data: []
    }
  },

  computed: {
    displayData() {
      return this.data.map(item => ({
        ...item,
        formattedEvaluationTime: this.formatTime(item.evaluationTime),
        averageScore: this.calculateAverageScore(item.metrics),
        totalTokens: this.calculateTotalTokens(item.metrics)
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

    calculateAverageScore(metrics) {
      if (!metrics || !metrics.length) return 0
      const totalScore = metrics.reduce((sum, metric) => sum + (metric.score || 0), 0)
      return totalScore / metrics.length
    },

    calculateTotalTokens(metrics) {
      if (!metrics || !metrics.length) return 0
      return metrics.reduce((sum, metric) => sum + (metric.token || 0), 0)
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
          content: '是否确认删除选中的评估结果？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('MetricResult').where({
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
          content: '是否确认删除该评估结果？',
          success: async (res) => {
            if (res.confirm) {
              await db.collection('MetricResult').doc(id).remove()
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