<template>
  <view class="container">
    <!-- 用户信息区域 -->
    <view class="user-info">
      <text class="user-name">{{ nickname }}</text>
      <text class="user-subtitle">的兑换记录</text>
    </view>
    
    <!-- 统计卡片 -->
    <view class="stats-card">
      <view class="card-header">
        <text class="title">兑换统计</text>
      </view>
      
      <view class="stats-grid">
        <view class="stats-item">
          <text class="stats-value">{{ stats.daily || 0 }}</text>
          <text class="stats-label">日卡 (次)</text>
        </view>
        
        <view class="stats-item">
          <text class="stats-value">{{ stats.monthly || 0 }}</text>
          <text class="stats-label">月卡 (次)</text>
        </view>
      </view>
    </view>

    <!-- 卡类别筛选 -->
    <view class="category-filter">
      <view class="filter-header">
        <text class="title">卡类别筛选</text>
      </view>
      
      <view class="filter-buttons">
        <view 
          class="filter-btn" 
          :class="{ active: activeCategory === 'all' }"
          @tap="filterByCategory('all')"
        >
          <text>全部</text>
        </view>
        
        <view 
          class="filter-btn" 
          :class="{ active: activeCategory === 'streamer' }"
          @tap="filterByCategory('streamer')"
        >
          <text>主播卡</text>
        </view>
        
        <view 
          class="filter-btn" 
          :class="{ active: activeCategory === 'tutorial' }"
          @tap="filterByCategory('tutorial')"
        >
          <text>教程卡</text>
        </view>
        
        <view 
          class="filter-btn" 
          :class="{ active: activeCategory === 'review' }"
          @tap="filterByCategory('review')"
        >
          <text>测评卡</text>
        </view>
        
        <view 
          class="filter-btn" 
          :class="{ active: activeCategory === 'enterprise' }"
          @tap="filterByCategory('enterprise')"
        >
          <text>企业卡</text>
        </view>
      </view>
      
      <!-- 下方显示当前筛选的类别统计 -->
      <view class="category-summary">
        <template v-if="activeCategory === 'all'">
          <text class="summary-text">各类别卡兑换情况：</text>
          <view class="category-stats">
            <view class="category-item" v-for="(count, category) in categoryStats" :key="category">
              <text class="category-name">{{ formatCardCategory(category) }}</text>
              <text class="category-count">{{ count }}张</text>
            </view>
          </view>
        </template>
        <template v-else>
          <text class="summary-text">
            共兑换了 <text class="highlight-count">{{ categoryStats[activeCategory] || 0 }}</text> 张{{ formatCardCategory(activeCategory) }}
          </text>
        </template>
      </view>
    </view>

    <!-- 兑换记录列表 -->
    <view class="exchange-list">
      <view class="list-header">
        <text class="title">兑换记录</text>
        <text class="count">(共{{ filteredRecords.length }}条)</text>
      </view>
      
      <view v-if="filteredRecords.length > 0">
        <view class="record-item" v-for="(record, index) in filteredRecords" :key="record._id">
          <view class="record-main">
            <view class="record-type">
              <text class="type-tag" :class="record.cardType">{{ formatCardType(record.cardType) }}</text>
              <text class="category-tag" :class="'category-'+record.cardCategory">
                {{ formatCardCategory(record.cardCategory) }}
              </text>
              <text class="type-value">{{ formatCardValue(record.cardType, record.cardValue) }}</text>
            </view>
            <view class="record-time">{{ formatTime(record.redeemTime) }}</view>
          </view>
          
          <view class="record-detail">
            <view class="detail-item">
              <text class="label">卡号：</text>
              <text class="value" @tap="copyContent(record.cardId)">{{ record.cardId }}</text>
            </view>
            <view class="detail-item">
              <text class="label">卡密：</text>
              <text class="value" @tap="copyContent(record.cardCode)">{{ record.cardCode }}</text>
            </view>
          </view>
        </view>
      </view>
      
      <view v-else class="empty-state">
        <text>暂无{{ activeCategory === 'all' ? '' : formatCardCategory(activeCategory) }}兑换记录</text>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      userId: '',
      nickname: '',
      records: [],
      activeCategory: 'all', // 当前选中的卡类别筛选
      error: '',
      stats: {
        daily: 0,
        monthly: 0
      },
      categoryStats: {
        streamer: 0,
        review: 0,
        tutorial: 0,
        enterprise: 0
      }
    }
  },
  
  computed: {
    filteredRecords() {
      if (this.activeCategory === 'all') {
        return this.records
      } else {
        return this.records.filter(record => record.cardCategory === this.activeCategory)
      }
    }
  },
  
  onLoad(options) {
    if (options.id) {
      this.userId = options.id
      this.nickname = options.nickname || '用户'
      this.loadExchangeRecords(this.userId)
    } else {
      this.error = '参数错误'
      uni.showToast({
        title: '参数错误',
        icon: 'none'
      })
    }
  },
  
  methods: {
    async loadExchangeRecords(userId) {
      try {
        uni.showLoading({
          title: '加载中...',
          mask: true
        })
        
        const db = uniCloud.database()
        const { result } = await db.collection('ExchangeRecord')
          .where({
            userId: userId
          })
          .orderBy('redeemTime', 'desc')
          .get()
          
        if (result.data && result.data.length > 0) {
          this.records = result.data
          
          // 统计各类型卡的兑换次数
          const typeStats = this.records.reduce((acc, record) => {
            acc[record.cardType] = (acc[record.cardType] || 0) + 1
            return acc
          }, {
            daily: 0,
            monthly: 0
          })
          
          this.stats = typeStats
          
          // 统计各卡类别的数量
          const categoryStats = this.records.reduce((acc, record) => {
            const category = record.cardCategory || 'unknown'
            acc[category] = (acc[category] || 0) + 1
            return acc
          }, {
            streamer: 0,
            review: 0,
            tutorial: 0,
            enterprise: 0
          })
          
          this.categoryStats = categoryStats
        }
        
        uni.hideLoading()
      } catch (e) {
        console.error('加载记录失败:', e)
        uni.hideLoading()
        uni.showToast({
          title: '加载失败',
          icon: 'none'
        })
      }
    },
    
    filterByCategory(category) {
      this.activeCategory = category
    },
    
    formatCardType(type) {
      const typeMap = {
        'daily': '日卡',
        'monthly': '月卡'
      }
      return typeMap[type] || type
    },
    
    formatCardCategory(category) {
      const categoryMap = {
        'all': '全部',
        'streamer': '主播卡',
        'review': '测评卡',
        'tutorial': '教程卡',
        'enterprise': '企业卡',
        'unknown': '未知类别'
      }
      return categoryMap[category] || category
    },
    
    formatCardValue(type, value) {
      return `${value}天`
    },
    
    formatTime(timestamp) {
      if (!timestamp) return '--'
      const date = new Date(timestamp)
      return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')} ${String(date.getHours()).padStart(2, '0')}:${String(date.getMinutes()).padStart(2, '0')}`
    },
    
    copyContent(content) {
      uni.setClipboardData({
        data: content,
        success: () => {
          uni.showToast({
            title: '已复制',
            icon: 'none'
          })
        }
      })
    }
  }
}
</script>

<style>
.container {
  padding: 30rpx;
  background: #f5f7fa;
  min-height: 100vh;
}

.user-info {
  margin-bottom: 20rpx;
  padding: 30rpx;
  background: #fff;
  border-radius: 16rpx;
  box-shadow: 0 2rpx 12rpx rgba(0, 0, 0, 0.05);
}

.user-name {
  font-size: 36rpx;
  font-weight: 600;
  color: #333;
}

.user-subtitle {
  font-size: 28rpx;
  color: #666;
  margin-left: 10rpx;
}

.stats-card, .category-filter, .exchange-list {
  background: #fff;
  border-radius: 16rpx;
  padding: 30rpx;
  margin-bottom: 20rpx;
  box-shadow: 0 2rpx 12rpx rgba(0, 0, 0, 0.05);
}

.card-header, .list-header, .filter-header {
  margin-bottom: 30rpx;
  display: flex;
  align-items: center;
}

.title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
}

.count {
  font-size: 24rpx;
  color: #999;
  margin-left: 10rpx;
}

.stats-grid {
  display: flex;
  justify-content: space-around;
  margin-top: 20rpx;
}

.stats-item {
  text-align: center;
  flex: 1;
  padding: 20rpx;
}

.stats-value {
  display: block;
  font-size: 40rpx;
  font-weight: 600;
  color: #0066cc;
  margin-bottom: 10rpx;
}

.stats-label {
  font-size: 24rpx;
  color: #666;
}

/* 类别筛选样式 */
.filter-buttons {
  display: flex;
  flex-wrap: wrap;
  margin: 0 -10rpx;
}

.filter-btn {
  margin: 10rpx;
  padding: 12rpx 24rpx;
  border-radius: 8rpx;
  background: #f0f2f5;
  font-size: 26rpx;
  color: #666;
  transition: all 0.3s;
}

.filter-btn.active {
  background: #0066cc;
  color: #fff;
}

.category-summary {
  margin-top: 30rpx;
  padding-top: 20rpx;
  border-top: 1px dashed #eee;
}

.summary-text {
  display: block;
  color: #666;
  font-size: 26rpx;
  margin-bottom: 15rpx;
}

.highlight-count {
  color: #0066cc;
  font-weight: bold;
  font-size: 32rpx;
}

.category-stats {
  display: flex;
  flex-wrap: wrap;
}

.category-item {
  width: 50%;
  display: flex;
  justify-content: space-between;
  padding: 10rpx 20rpx;
  font-size: 26rpx;
  box-sizing: border-box;
}

.category-name {
  color: #666;
}

.category-count {
  font-weight: 500;
  color: #333;
}

/* 记录列表样式 */
.record-item {
  border-bottom: 2rpx solid #f0f0f0;
  padding: 20rpx 0;
}

.record-item:last-child {
  border-bottom: none;
}

.record-main {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16rpx;
}

.record-type {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
}

.type-tag, .category-tag {
  padding: 4rpx 16rpx;
  border-radius: 6rpx;
  font-size: 24rpx;
  margin-right: 16rpx;
}

.type-tag.daily {
  background: rgba(64, 169, 255, 0.1);
  color: #40a9ff;
}

.type-tag.monthly {
  background: rgba(82, 196, 26, 0.1);
  color: #52c41a;
}

.category-tag {
  background: rgba(0, 0, 0, 0.05);
  color: #666;
}

.category-tag.category-streamer {
  background: rgba(24, 144, 255, 0.1);
  color: #1890ff;
}

.category-tag.category-tutorial {
  background: rgba(82, 196, 26, 0.1);
  color: #52c41a;
}

.category-tag.category-review {
  background: rgba(250, 173, 20, 0.1);
  color: #faad14;
}

.category-tag.category-enterprise {
  background: rgba(114, 46, 209, 0.1);
  color: #722ed1;
}

.type-value {
  font-size: 28rpx;
  color: #333;
  font-weight: 500;
}

.record-time {
  font-size: 24rpx;
  color: #999;
}

.record-detail {
  background: #f9f9f9;
  padding: 16rpx;
  border-radius: 8rpx;
}

.detail-item {
  display: flex;
  font-size: 26rpx;
  margin-bottom: 8rpx;
}

.detail-item:last-child {
  margin-bottom: 0;
}

.detail-item .label {
  color: #666;
  width: 100rpx;
}

.detail-item .value {
  color: #333;
  flex: 1;
}

.empty-state {
  text-align: center;
  padding: 60rpx 0;
  color: #999;
  font-size: 28rpx;
}

.error-tip {
  text-align: center;
  padding: 40rpx;
  color: #ff4d4f;
  font-size: 28rpx;
}
</style>