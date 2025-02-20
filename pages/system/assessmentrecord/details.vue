<template>
  <view class="detail-container">
    <view class="detail-card">
      <view class="card-header">
        <text class="header-title">测评详情</text>
        <button @click="goBack" class="uni-button" type="default" size="mini">返回</button>
      </view>
      
      <!-- 基础信息 -->
      <view class="basic-info">
        <view class="info-item">
          <text class="label">主播姓名：</text>
          <text>{{details?.name || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">记录ID：</text>
          <text>{{details?.recordId || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">用户ID：</text>
          <text>{{details?.userId || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">得分：</text>
          <text :class="['score-value', getScoreClass(details?.score)]">{{details?.score || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">测评时间：</text>
          <text>{{formatTime(details?.assessmentTime) || '-'}}</text>
        </view>
        <view class="info-item">
          <text class="label">持续时间：</text>
          <text>{{formatDuration(details?.duration) || '-'}}</text>
        </view>
      </view>

      <!-- 资源信息 -->
      <view class="metrics-detail">
        <view class="section-title">资源信息</view>
        <view class="metric-card">
          <view class="metric-content">
            <view class="info-item">
              <text class="label">文件ID：</text>
              <text>{{details?.fileId || '-'}}</text>
            </view>
            <view class="info-item">
              <text class="label">Token数：</text>
              <text>{{details?.totaltoken || 0}}</text>
            </view>
            <view class="info-item">
              <text class="label">文件链接：</text>
              <view v-if="details?.uploadUrl" class="link-group">
                <text class="link-text">{{details.uploadUrl}}</text>
                <button class="uni-button" type="primary" size="mini" @click="copyUrl(details.uploadUrl)">复制</button>
              </view>
              <text v-else>-</text>
            </view>
          </view>
        </view>
      </view>

      <!-- 指标评测结果 -->
      <view class="metrics-detail">
        <view class="section-title">指标评测结果</view>
        <view class="metric-card" v-if="metricResults?.metrics?.length">
          <view class="metric-content">
            <view v-for="(metric, index) in metricResults.metrics" :key="index" class="metric-result-item">
              <view class="metric-header">
                <text class="metric-name">{{metric.metricname}}</text>
                <text :class="['metric-score', getScoreClass(metric.score)]">{{metric.score}}分</text>
              </view>
              
              <view class="metric-details">
                <view class="metric-detail-item">
                  <text class="detail-label">评估等级：</text>
                  <text class="detail-value">{{metric.description.level}}</text>
                </view>
                <view class="metric-detail-item">
                  <text class="detail-label">评估结论：</text>
                  <text class="detail-value">{{metric.description.evaluation}}</text>
                </view>
                <view class="metric-detail-item">
                  <text class="detail-label">改进建议：</text>
                  <text class="detail-value">{{metric.description.suggestion}}</text>
                </view>
                <view class="metric-detail-item" v-if="metric.token">
                  <text class="detail-label">Token消耗：</text>
                  <text class="detail-value">{{metric.token}}</text>
                </view>
              </view>
            </view>
          </view>
        </view>
        <view class="metric-card" v-else>
          <view class="metric-content">
            <text class="empty-text">暂无评测结果</text>
          </view>
        </view>
      </view>

      <!-- 文本内容 -->
      <view class="metrics-detail">
        <view class="section-title">报告</view>
        <view class="metric-card">
          <view class="metric-content">
            <view class="content-section">
              <view class="report-content">
                <text class="report-text" user-select>{{details?.record || '暂无文本内容'}}</text>
              </view>
            </view>
          </view>
        </view>
      </view>

      <!-- 记录内容 -->
      <view class="metrics-detail">
        <view class="section-title">视频内容</view>
        <view class="metric-card">
          <view class="metric-content">
            <view class="content-section">
              <view class="report-content">
                <text class="report-text" user-select>{{details?.text || '暂无记录内容'}}</text>
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
      details: null,
      metricResults: null
    }
  },

  onLoad(option) {
    if (option.id) {
      this.getDetails(option.id)
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

    formatDuration(minutes) {
      if (!minutes && minutes !== 0) return '-'
      const hours = Math.floor(minutes / 60)
      const remainingMinutes = Math.floor(minutes % 60)
      if (hours > 0) {
        return `${hours}小时${remainingMinutes > 0 ? remainingMinutes + '分钟' : ''}`
      }
      return `${remainingMinutes}分钟`
    },

    getScoreClass(score) {
      if (!score && score !== 0) return ''
      if (score >= 90) return 'score-excellent'
      if (score >= 80) return 'score-good'
      if (score >= 60) return 'score-fair'
      return 'score-poor'
    },

    async getDetails(id) {
      try {
        uni.showLoading({ mask: true });
        // 获取评估记录详情
        const recordRes = await db.collection('AssessmentRecord').doc(id).get()
        if (recordRes.result.data && recordRes.result.data.length > 0) {
          this.details = recordRes.result.data[0]
          
          // 获取对应的指标评测结果
          if (this.details.recordId) {
            const metricRes = await db.collection('MetricResult')
              .where({
                recordId: this.details.recordId
              })
              .get()
              
            if (metricRes.result.data && metricRes.result.data.length > 0) {
              this.metricResults = metricRes.result.data[0]
            }
          }
        }
      } catch (error) {
        console.error('获取详情失败:', error)
        uni.showToast({
          title: '获取详情失败',
          icon: 'error'
        })
      } finally {
        uni.hideLoading()
      }
    },

    copyUrl(url) {
      uni.setClipboardData({
        data: url,
        success: () => {
          uni.showToast({
            title: '复制成功',
            icon: 'success'
          })
        }
      })
    },

    goBack() {
      const eventChannel = this.getOpenerEventChannel()
      eventChannel.emit('refreshData')
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
  align-items: flex-start;
}

.info-item:last-child {
  margin-bottom: 0;
}

.label {
  color: #606266;
  width: 100px;
  flex-shrink: 0;
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

.metric-content {
  padding: 16px;
}

.content-section {
  margin-bottom: 16px;
}

.content-section:last-child {
  margin-bottom: 0;
}

.report-content {
  padding: 12px;
  background: #fafafa;
  border-radius: 4px;
}

.report-text {
  color: #606266;
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-all;
}

.link-group {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 8px;
}

.link-text {
  flex: 1;
  color: #409EFF;
  word-break: break-all;
}

.score-value {
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

.metric-result-item {
  padding: 16px;
  border: 1px solid #EBEEF5;
  border-radius: 8px;
  margin-bottom: 16px;
  background: #fff;
}

.metric-result-item:last-child {
  margin-bottom: 0;
}

.metric-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  padding-bottom: 12px;
  border-bottom: 1px solid #EBEEF5;
}

.metric-name {
  font-size: 16px;
  font-weight: bold;
  color: #303133;
}

.metric-score {
  font-size: 16px;
  font-weight: bold;
}

.metric-details {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.metric-detail-item {
  display: flex;
}

.detail-label {
  width: 80px;
  flex-shrink: 0;
  color: #606266;
}

.detail-value {
  flex: 1;
  color: #303133;
  word-break: break-all;
}

.empty-text {
  color: #909399;
  text-align: center;
  padding: 20px 0;
}

@media screen and (max-width: 768px) {
  .detail-container {
    padding: 8px;
  }
  
  .label {
    width: 80px;
  }

  .metric-card {
    margin: 0 8px 8px;
  }

  .card-header,
  .metric-content,
  .basic-info {
    padding: 12px;
  }

  .header-title {
    font-size: 16px;
  }
  
  .metric-result-item {
    padding: 12px;
  }
  
  .detail-label {
    width: 70px;
  }
}
</style>