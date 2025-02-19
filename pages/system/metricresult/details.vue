<template>
  <view class="detail-container">
    <view class="detail-card">
      <view class="card-header">
        <text class="header-title">评估详情</text>
        <button @click="goBack" class="uni-button" type="default" size="mini">返回</button>
      </view>
      
      <!-- 基础信息 -->
      <view class="basic-info">
        <view class="info-item">
          <text class="label">评估ID：</text>
          <text>{{detail?.resultId || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">记录ID：</text>
          <text>{{detail?.recordId || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">评估时间：</text>
          <text>{{formattedTime || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">平均得分：</text>
          <text>{{averageScore}}</text>
        </view>
        <view class="info-item">
          <text class="label">总Token：</text>
          <text>{{totalTokens}}</text>
        </view>
      </view>
      
      <!-- 指标详情列表 -->
      <view class="metrics-detail">
        <view class="section-title">指标详情</view>
        <view v-for="metric in detail?.metrics" :key="metric.metricId" class="metric-card">
          <view class="metric-header">
            <view class="metric-title">
              <text class="metric-name">{{metric.metricname || '未命名指标'}}</text>
              <view class="metric-stats">
                <view class="stat-item">
                  <text class="stat-label">得分：</text>
                  <text :class="['stat-value', getScoreClass(metric.score)]">{{metric.score?.toFixed(2) || '-'}}</text>
                </view>
                <view class="stat-item">
                  <text class="stat-label">Token：</text>
                  <text class="stat-value">{{metric.token?.toFixed(0) || '-'}}</text>
                </view>
              </view>
            </view>
          </view>

          <view class="metric-content">
            <!-- 评估等级 -->
            <view class="content-section">
              <text class="section-subtitle">评估等级</text>
              <view class="level-content">
                <text :class="['level-badge', getLevelClass(metric.description?.level)]">
                  {{metric.description?.level || '未评级'}}
                </text>
              </view>
            </view>

            <!-- 评估结论 -->
            <view class="content-section" v-if="metric.description?.evaluation">
              <text class="section-subtitle">评估结论</text>
              <view class="evaluation-content">
                <text class="evaluation-text">{{metric.description.evaluation}}</text>
              </view>
            </view>

            <!-- 改进建议 -->
            <view class="content-section" v-if="metric.description?.suggestion">
              <text class="section-subtitle">改进建议</text>
              <view class="suggestion-content">
                <text class="suggestion-text">{{metric.description.suggestion}}</text>
              </view>
            </view>
          </view>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database()

export default {
  data() {
    return {
      detail: null
    }
  },

  computed: {
    formattedTime() {
      return this.formatTime(this.detail?.evaluationTime)
    },
    averageScore() {
      if (!this.detail?.metrics?.length) return '-'
      const avg = this.detail.metrics.reduce((sum, metric) => sum + (metric.score || 0), 0) / this.detail.metrics.length
      return avg.toFixed(2)
    },
    totalTokens() {
      if (!this.detail?.metrics?.length) return '-'
      const total = this.detail.metrics.reduce((sum, metric) => sum + (metric.token || 0), 0)
      return total.toFixed(0)
    }
  },

  onLoad(options) {
    if (options.id) {
      this.loadDetail(options.id)
    }
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

    async loadDetail(id) {
      try {
        const res = await db.collection('MetricResult').doc(id).get()
        if (res.result.data && res.result.data.length > 0) {
          this.detail = res.result.data[0]
        }
      } catch (error) {
        console.error('获取详情失败:', error)
        uni.showToast({
          title: '获取详情失败',
          icon: 'error'
        })
      }
    },

    getScoreClass(score) {
      if (!score && score !== 0) return ''
      if (score >= 90) return 'score-excellent'
      if (score >= 80) return 'score-good'
      if (score >= 60) return 'score-fair'
      return 'score-poor'
    },

    getLevelClass(level) {
      if (!level) return ''
      const levelMap = {
        'A': 'level-excellent',
        'B': 'level-good',
        'C': 'level-fair',
        'D': 'level-poor'
      }
      return levelMap[level] || ''
    },

    goBack() {
      uni.navigateBack()
    }
  }
}
</script>

<style>
.detail-container {
  padding: 16px;
  background: #f5f7fa;
  min-height: 100vh;
}

.detail-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0,0,0,0.1);
}

.card-header {
  padding: 16px;
  border-bottom: 1px solid #EBEEF5;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header-title {
  font-size: 18px;
  font-weight: bold;
}

.basic-info {
  padding: 16px;
  border-bottom: 1px solid #EBEEF5;
  background: #fafafa;
}

.info-item {
  margin-bottom: 12px;
  display: flex;
  align-items: center;
}

.info-item:last-child {
  margin-bottom: 0;
}

.label {
  color: #606266;
  width: 100px;
}

.section-title {
  padding: 16px;
  font-size: 16px;
  font-weight: bold;
  color: #303133;
}

.metric-card {
  margin: 0 16px 16px;
  border: 1px solid #EBEEF5;
  border-radius: 8px;
  overflow: hidden;
}

.metric-header {
  padding: 16px;
  background: #f8f9fb;
  border-bottom: 1px solid #EBEEF5;
}

.metric-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.metric-name {
  font-size: 16px;
  font-weight: bold;
  color: #303133;
}

.metric-stats {
  display: flex;
  gap: 16px;
}

.stat-item {
  display: flex;
  align-items: center;
}

.stat-label {
  color: #606266;
  margin-right: 4px;
}

.stat-value {
  font-weight: bold;
}

.score-excellent {
  color: #67C23A;
}

.score-good {
  color: #409EFF;
}

.score-fair {
  color: #E6A23C;
}

.score-poor {
  color: #F56C6C;
}

.metric-content {
  padding: 16px;
}

.content-section {
  margin-bottom: 16px;
}

.content-section:last-child {
  margin-bottom: 0;
}

.section-subtitle {
  font-size: 14px;
  font-weight: bold;
  color: #303133;
  margin-bottom: 8px;
}

.level-content {
  margin: 8px 0;
}

.level-badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 4px;
  font-weight: bold;
}

.level-excellent {
  background: #f0f9eb;
  color: #67C23A;
}

.level-good {
  background: #ecf5ff;
  color: #409EFF;
}

.level-fair {
  background: #fdf6ec;
  color: #E6A23C;
}

.level-poor {
  background: #fef0f0;
  color: #F56C6C;
}

.evaluation-content,
.suggestion-content {
  padding: 12px;
  background: #fafafa;
  border-radius: 4px;
  margin-top: 8px;
}

.evaluation-text,
.suggestion-text {
  color: #606266;
  line-height: 1.6;
  white-space: pre-wrap;
}

@media screen and (max-width: 768px) {
  .detail-container {
    padding: 8px;
  }
  
  .metric-title {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .metric-stats {
    margin-top: 8px;
  }
}
</style>