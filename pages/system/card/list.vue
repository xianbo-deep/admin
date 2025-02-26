<template>
  <view class="statistics-container">
    <view class="statistics-area">
      <uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
      
      <!-- 卡类别筛选按钮 -->
      <view class="filter-buttons">
        <button 
          class="filter-btn" 
          :class="{ active: activeCardCategory === 'all' }" 
          @click="changeCardCategory('all')"
        >全部</button>
        <button 
          class="filter-btn" 
          :class="{ active: activeCardCategory === 'streamer' }" 
          @click="changeCardCategory('streamer')"
        >主播卡</button>
        <button 
          class="filter-btn" 
          :class="{ active: activeCardCategory === 'review' }" 
          @click="changeCardCategory('review')"
        >测评卡</button>
        <button 
          class="filter-btn" 
          :class="{ active: activeCardCategory === 'tutorial' }" 
          @click="changeCardCategory('tutorial')"
        >教程卡</button>
        <button 
          class="filter-btn" 
          :class="{ active: activeCardCategory === 'enterprise' }" 
          @click="changeCardCategory('enterprise')"
        >企业卡</button>
      </view>

      <!-- 统计卡片区域 -->
      <view class="stat-cards">
        <view class="stat-card" v-for="(stat, type) in cardStats" :key="type">
          <view class="stat-title">{{getCardTypeName(type)}}</view>
          <view class="stat-content">
            <view class="stat-item">
              <text class="label">总数量</text>
              <text class="number">{{stat.total || 0}}</text>
            </view>
            <view class="stat-progress-bar">
              <view class="progress" :style="{width: getProgressWidth(stat)}"></view>
            </view>
            <view class="stat-details">
              <view class="detail-item">
                <text class="sub-label">已使用</text>
                <text class="sub-number used">{{stat.used || 0}}</text>
              </view>
              <view class="detail-item">
                <text class="sub-label">未使用</text>
                <text class="sub-number unused">{{stat.unused || 0}}</text>
              </view>
            </view>
          </view>
        </view>
      </view>
      
      <!-- 定时任务管理模块 -->
      <view class="time-config-section">
        <view class="section-title">定时任务管理</view>
        <view class="time-config-cards">
          <view class="time-config-card" v-for="(config, index) in timeConfigs" :key="index">
            <view class="config-header">
              <view class="config-name">{{config.name}}</view>
              <switch :checked="config.enabled" @change="toggleTaskStatus(index)" color="#409EFF" />
            </view>
            <view class="config-details">
              <view class="config-item">
                <text class="config-label">云函数:</text>
                <text class="config-value">{{config.functionName}}</text>
              </view>
              <view class="config-item">
                <text class="config-label">状态:</text>
                <text class="config-value" :class="config.enabled ? 'status-enabled' : 'status-disabled'">
                  {{config.enabled ? '已启用' : '已禁用'}}
                </text>
              </view>
              <view class="config-item">
                <text class="config-label">更新时间:</text>
                <text class="config-value">{{formatDate(config.updateTime)}}</text>
              </view>
            </view>
            <view class="config-actions">
              <button class="action-btn edit-btn" @click="editTimeConfig(index)">编辑</button>
              <button class="action-btn delete-btn" @click="deleteTimeConfig(index)">删除</button>
            </view>
          </view>
        </view>
        
        <view class="add-config-container">
          <button class="add-config-btn" @click="showAddConfigModal">添加定时任务</button>
        </view>
        
        <!-- 添加/编辑定时任务弹窗 -->
        <uni-popup ref="configPopup" type="dialog">
          <view class="custom-popup">
            <view class="popup-header">
              <text class="popup-title">{{isEditing ? '编辑定时任务' : '添加定时任务'}}</text>
            </view>
            <view class="popup-form">
              <view class="form-item">
                <text class="form-label">任务名称</text>
                <input class="form-input" v-model="currentConfig.name" placeholder="请输入定时任务名称" />
              </view>
              <view class="form-item">
                <text class="form-label">云函数名称</text>
                <input class="form-input" v-model="currentConfig.functionName" placeholder="请输入云函数名称" />
              </view>
            </view>
            <view class="popup-footer">
              <button class="popup-btn cancel-btn" @click="closeConfigPopup">取消</button>
              <button class="popup-btn confirm-btn" @click="confirmSaveConfig">确定</button>
            </view>
          </view>
        </uni-popup>
      </view>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database()
const $ = db.command.aggregate

const cardTypeOptions = [
  { value: 'daily', label: '日卡' },
  { value: 'monthly', label: '月卡' }
]

export default {
  data() {
    return {
      cardStats: {
        daily: { total: 0, unused: 0, used: 0 },
        monthly: { total: 0, unused: 0, used: 0 }
      },
      activeCardCategory: 'all', // 默认显示全部卡类别
      timeConfigs: [],
      currentConfig: {
        name: '',
        functionName: '',
        enabled: false,
        updateTime: new Date()
      },
      isEditing: false,
      editIndex: -1
    }
  },

  mounted() {
    this.getCardStats()
    this.getTimeConfigs()
  },

  methods: {
    changeCardCategory(category) {
      this.activeCardCategory = category
      this.getCardStats()
    },
    
    async getCardStats() {
      try {
        // 构建查询条件
        let query = db.collection('Card').aggregate()
        
        // 如果选择了特定类别，添加筛选条件
        if (this.activeCardCategory !== 'all') {
          query = query.match({
            cardCategory: this.activeCardCategory
          })
        }
        
        // 添加条件只获取日卡和月卡，排除次卡
        query = query.match({
          cardType: db.command.in(['daily', 'monthly'])
        })
        
        // 执行聚合查询
        const { result } = await query
          .group({
            _id: {
              cardType: '$cardType',
              status: '$status'
            },
            count: $.sum(1)
          })
          .end()
        
        // 重置统计数据
        Object.keys(this.cardStats).forEach(type => {
          this.cardStats[type] = { total: 0, unused: 0, used: 0 }
        })
        
        // 处理聚合结果
        result.data.forEach(item => {
          const type = item._id.cardType
          const status = item._id.status
          const count = item.count
    
          if (this.cardStats[type]) {
            if (status === 'unused') {
              this.cardStats[type].unused = count
            } else if (status === 'used') {
              this.cardStats[type].used = count
            }
            this.cardStats[type].total += count
          }
        })
    
      } catch (error) {
        console.error('获取统计数据失败:', error)
      }
    },

    getCardTypeName(type) {
      const option = cardTypeOptions.find(opt => opt.value === type)
      return option ? option.label : type
    },

    getProgressWidth(stat) {
      if (!stat.total) return '0%'
      const percentage = (stat.used / stat.total) * 100
      return `${percentage}%`
    },
    
    // 定时任务管理相关方法
    async getTimeConfigs() {
      try {
        const { result } = await db.collection('TimeConfig').get()
        this.timeConfigs = result.data || []
      } catch (error) {
        console.error('获取定时任务配置失败:', error)
        uni.showToast({
          title: '获取定时任务配置失败',
          icon: 'none'
        })
      }
    },
    
    formatDate(timestamp) {
      if (!timestamp) return '未更新'
      const d = new Date(timestamp)
      return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')} ${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}:${String(d.getSeconds()).padStart(2, '0')}`
    },
    
    showAddConfigModal() {
      this.isEditing = false
      this.currentConfig = {
        name: '',
        functionName: '',
        enabled: false,
        updateTime: new Date()
      }
      this.$refs.configPopup.open()
    },
    
    editTimeConfig(index) {
      this.isEditing = true
      this.editIndex = index
      // 深拷贝确保不直接修改原对象
      const config = JSON.parse(JSON.stringify(this.timeConfigs[index]))
      // 确保enabled是布尔值
      config.enabled = Boolean(config.enabled)
      this.currentConfig = config
      this.$refs.configPopup.open()
    },
    
    closeConfigPopup() {
      this.$refs.configPopup.close()
    },
    
    async confirmSaveConfig() {
      try {
        // 验证表单
        if (!this.currentConfig.name.trim()) {
          uni.showToast({
            title: '请输入任务名称',
            icon: 'none'
          })
          return
        }
        
        if (!this.currentConfig.functionName.trim()) {
          uni.showToast({
            title: '请输入云函数名称',
            icon: 'none'
          })
          return
        }
        
        const updatedConfig = {
          name: this.currentConfig.name,
          functionName: this.currentConfig.functionName,
          enabled: this.isEditing ? this.timeConfigs[this.editIndex].enabled : false,
          updateTime: Date.now()
        }
        
        if (this.isEditing) {
          // 更新现有配置
          const id = this.timeConfigs[this.editIndex]._id
          await db.collection('TimeConfig').doc(id).update(updatedConfig)
          this.timeConfigs[this.editIndex] = { 
            ...updatedConfig, 
            _id: id 
          }
          uni.showToast({
            title: '更新成功',
            icon: 'success'
          })
        } else {
          // 添加新配置
          const { result } = await db.collection('TimeConfig').add(updatedConfig)
          this.timeConfigs.push({
            ...updatedConfig,
            _id: result.id
          })
          uni.showToast({
            title: '添加成功',
            icon: 'success'
          })
        }
        
        this.closeConfigPopup()
      } catch (error) {
        console.error('保存定时任务配置失败:', error)
        console.error('错误详情:', JSON.stringify(error))
        uni.showToast({
          title: '保存失败: ' + (error.message || '未知错误'),
          icon: 'none',
          duration: 3000
        })
      }
    },
    
    async deleteTimeConfig(index) {
      uni.showModal({
        title: '确认删除',
        content: `确定要删除"${this.timeConfigs[index].name}"吗？`,
        success: async (res) => {
          if (res.confirm) {
            try {
              const id = this.timeConfigs[index]._id
              await db.collection('TimeConfig').doc(id).remove()
              this.timeConfigs.splice(index, 1)
              uni.showToast({
                title: '删除成功',
                icon: 'success'
              })
            } catch (error) {
              console.error('删除定时任务配置失败:', error)
              uni.showToast({
                title: '删除失败',
                icon: 'none'
              })
            }
          }
        }
      })
    },
    
    async toggleTaskStatus(index) {
      try {
        const config = this.timeConfigs[index]
        const updatedStatus = !config.enabled
        
        await db.collection('TimeConfig').doc(config._id).update({
          enabled: updatedStatus,
          updateTime: Date.now()
        })
        
        this.timeConfigs[index].enabled = updatedStatus
        this.timeConfigs[index].updateTime = Date.now()
        
        uni.showToast({
          title: updatedStatus ? '已启用' : '已禁用',
          icon: 'success'
        })
      } catch (error) {
        console.error('更新任务状态失败:', error)
        uni.showToast({
          title: '操作失败',
          icon: 'none'
        })
      }
    }
  }
}
</script>

<style>
.statistics-container {
  padding: 20rpx;
  min-height: 100vh;
  background: #f5f7fa;
}

.statistics-area {
  padding: 20rpx;
}

/* 卡类别筛选按钮样式 */
.filter-buttons {
  display: flex;
  justify-content: center;
  gap: 20rpx;
  margin-bottom: 30rpx;
  flex-wrap: wrap;
  padding: 10rpx;
}

.filter-btn {
  min-width: 160rpx;
  height: 80rpx;
  border-radius: 40rpx;
  background: #fff;
  color: #606266;
  font-size: 28rpx;
  border: 2rpx solid #dcdfe6;
  box-shadow: 0 2rpx 12rpx rgba(0, 0, 0, 0.05);
  line-height: 80rpx;
  padding: 0 30rpx;
  transition: all 0.3s;
}

.filter-btn.active {
  background: #409EFF;
  color: #fff;
  border-color: #409EFF;
  box-shadow: 0 4rpx 16rpx rgba(64, 158, 255, 0.4);
}

.stat-cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300rpx, 1fr));
  gap: 30rpx;
}

.stat-card {
  background: #ffffff;
  border-radius: 16rpx;
  padding: 30rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
}

.stat-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 30rpx;
  text-align: center;
}

.stat-content {
  display: flex;
  flex-direction: column;
  gap: 20rpx;
}

.stat-item {
  display: flex;
  justify-content: center;
  align-items: baseline;
  gap: 16rpx;
}

.uni-stat-breadcrumb-on-phone {
  padding: 0 20px !important;
  border-bottom: 1px #f5f5f5 solid;
}

.label {
  font-size: 28rpx;
  color: #666;
}

.number {
  font-size: 48rpx;
  font-weight: bold;
  color: #409EFF;
}

.stat-progress-bar {
  height: 8rpx;
  background: #e5e9f2;
  border-radius: 4rpx;
  overflow: hidden;
  margin: 20rpx 0;
}

.progress {
  height: 100%;
  background: #409EFF;
  transition: width 0.3s ease;
}

.stat-details {
  display: flex;
  justify-content: space-around;
  padding-top: 20rpx;
}

.detail-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8rpx;
}

.sub-label {
  font-size: 24rpx;
  color: #909399;
}

.sub-number {
  font-size: 32rpx;
  font-weight: bold;
}

.sub-number.used {
  color: #67C23A;
}

.sub-number.unused {
  color: #409EFF;
}

/* 定时任务管理样式 */
.time-config-section {
  margin-top: 40rpx;
}

.section-title {
  font-size: 36rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 30rpx;
  padding-left: 10rpx;
}

.time-config-cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(500rpx, 1fr));
  gap: 40rpx;
  margin-bottom: 50rpx;
}

.time-config-card {
  background: #ffffff;
  border-radius: 20rpx;
  padding: 50rpx;
  box-shadow: 0 8rpx 24rpx rgba(0, 0, 0, 0.1);
  min-height: 380rpx;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.config-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30rpx;
  padding-bottom: 30rpx;
  border-bottom: 2px solid #eef2f6;
}

.config-name {
  font-size: 42rpx;
  font-weight: bold;
  color: #2c3e50;
  max-width: 70%;
  word-break: break-word;
  line-height: 1.3;
}

.config-details {
  display: flex;
  flex-direction: column;
  gap: 24rpx;
  margin-bottom: 40rpx;
  flex: 1;
}

.config-item {
  display: flex;
  align-items: center;
}

.config-label {
  font-size: 32rpx;
  color: #666;
  width: 140rpx;
}

.config-value {
  font-size: 32rpx;
  color: #333;
  font-weight: 500;
}

.status-enabled {
  color: #67C23A;
  font-weight: bold;
}

.status-disabled {
  color: #909399;
}

.config-actions {
  display: flex;
  justify-content: flex-end;
  gap: 30rpx;
  margin-top: 30rpx;
}

.action-btn {
  font-size: 28rpx;
  padding: 12rpx 32rpx;
  border-radius: 10rpx;
  transition: all 0.3s;
}

.edit-btn {
  color: #fff;
  background: #409EFF;
}

.edit-btn:hover {
  background: #3a8ee6;
}

.delete-btn {
  color: #fff;
  background: #F56C6C;
}

.delete-btn:hover {
  background: #e74c3c;
}

.add-config-container {
  display: flex;
  justify-content: center;
  margin: 30rpx 0;
}

.add-config-btn {
  background: #409EFF;
  color: #fff;
  font-size: 28rpx;
  padding: 16rpx 40rpx;
  border-radius: 12rpx;
}

/* 自定义弹窗样式 */
.custom-popup {
  background: #fff;
  border-radius: 16rpx;
  width: 90%;
  max-width: 600rpx;
  overflow: hidden;
}

.popup-header {
  padding: 30rpx;
  text-align: center;
  border-bottom: 1px solid #eee;
}

.popup-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
}

.popup-form {
  padding: 30rpx;
}

.form-item {
  margin-bottom: 30rpx;
  display: flex;
  flex-direction: column;
  gap: 15rpx;
}

.form-label {
  font-size: 28rpx;
  color: #606266;
  font-weight: 500;
}

.form-input {
  height: 80rpx;
  border: 1px solid #dcdfe6;
  border-radius: 8rpx;
  padding: 0 20rpx;
  font-size: 28rpx;
  background: #f8f9fc;
}

.popup-footer {
  display: flex;
  border-top: 1px solid #eee;
}

.popup-btn {
  flex: 1;
  height: 90rpx;
  line-height: 90rpx;
  text-align: center;
  font-size: 30rpx;
}

.cancel-btn {
  background: #f5f7fa;
  color: #606266;
}

.confirm-btn {
  background: #409EFF;
  color: #fff;
}

@media screen and (max-width: 768px) {
  .stat-cards, .time-config-cards {
    grid-template-columns: 1fr;
  }
  
  .stat-card, .time-config-card {
    padding: 20rpx;
  }
  
  .filter-buttons {
    flex-wrap: wrap;
  }
  
  .filter-btn {
    min-width: 140rpx;
    font-size: 26rpx;
  }
}
</style>