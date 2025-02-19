<!-- details.vue -->
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
      details: null
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
        const res = await db.collection('AssessmentRecord').doc(id).get()
        if (res.result.data && res.result.data.length > 0) {
          this.details = res.result.data[0]
        }
      } catch (error) {
        uni.showToast({
          title: '获取详情失败',
          icon: 'error'
        })
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
      // 在返回前通知列表页刷新数据
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
}
</style>