<template>
  <view class="preview-container">
    <!-- 视频标题 -->
    <view class="video-title">
      <text>{{ videoData.title || '加载中...' }}</text>
    </view>
    
    <!-- 视频播放器 -->
    <view class="video-player-container">
      <video 
        v-if="videoData.url"
        id="myVideo" 
        class="video-player" 
        :src="videoData.url"
        @error="onVideoError"
        controls
      ></video>
      <view v-else class="video-placeholder">
        <text>加载中...</text>
      </view>
    </view>
    
    <!-- 视频信息 -->
    <view class="video-info-section">
      <view class="section-title">
        <text>视频信息</text>
      </view>
      
      <view class="info-item">
        <text class="info-label">视频ID:</text>
        <text class="info-value">{{ videoData.videoId || '-' }}</text>
      </view>
      
      <view class="info-item">
        <text class="info-label">分类:</text>
        <text class="info-value">{{ categoryName }}</text>
      </view>
      
      <view class="info-item">
        <text class="info-label">上传时间:</text>
        <text class="info-value">{{ formatDate(videoData.uploadTime) }}</text>
      </view>
      
      <view class="description-item" v-if="videoData.description">
        <text class="info-label">描述:</text>
        <text class="description-content">{{ videoData.description }}</text>
      </view>
    </view>
    
    <!-- 学习进度列表 -->
    <view class="progress-section">
      <view class="section-title">
        <text>学习进度列表</text>
        <text class="progress-summary" v-if="progressSummary">{{ progressSummary }}</text>
      </view>
      
      <view class="no-progress" v-if="!progressList || progressList.length === 0">
        <text>暂无学习记录</text>
      </view>
      
      <view class="progress-list" v-else>
        <view class="progress-header">
          <text class="progress-col user-col">用户</text>
          <text class="progress-col duration-col">观看时长</text>
          <text class="progress-col status-col">状态</text>
        </view>
        
        <view 
          class="progress-row" 
          v-for="(progress, index) in progressList" 
          :key="index"
        >
          <text class="progress-col user-col">{{ getUserDisplayName(progress) }}</text>
          <text class="progress-col duration-col">{{ formatDuration(progress.watchedDuration) }}</text>
          <text class="progress-col status-col" :class="{ 'completed': progress.completed }">
            {{ progress.completed ? '已完成' : '未完成' }}
          </text>
        </view>
      </view>
    </view>
    
    <!-- 底部按钮区 -->
    <view class="button-group">
      <button class="uni-button" type="default" @click="goBack">返回</button>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database();
const dbVideo = 'Video';
const dbCategory = 'Category';
const dbProgress = 'Progress';
const dbUser = 'User';

export default {
  data() {
    return {
      videoId: '',
      videoData: {},
      categoryName: '',
      progressList: [],
      progressSummary: '',
      userMap: new Map() // 用于存储用户ID和用户信息的映射
    };
  },
  
  onLoad(options) {
    if (options.id) {
      this.videoId = options.id;
      this.loadData();
    } else {
      uni.showToast({
        title: '视频参数错误',
        icon: 'none'
      });
      setTimeout(() => {
        uni.navigateBack();
      }, 1500);
    }
  },
  
  methods: {
    // 加载所有数据
    async loadData() {
      try {
        uni.showLoading({ title: '加载中...' });
        await Promise.all([
          this.loadVideoData(),
          this.loadProgressData()
        ]);
        uni.hideLoading();
      } catch (error) {
        uni.hideLoading();
        uni.showModal({
          title: '加载失败',
          content: error.message || '无法加载数据',
          showCancel: false,
          success: () => {
            uni.navigateBack();
          }
        });
      }
    },
    
    // 加载视频数据
    async loadVideoData() {
      try {
        const videoRes = await db.collection(dbVideo)
          .doc(this.videoId)
          .get();
          
        if (!videoRes.result.data.length) {
          throw new Error('视频不存在');
        }
        
        // 检查视频URL
        const videoData = videoRes.result.data[0];
        if (!videoData.url) {
          throw new Error('视频地址无效');
        }
        
        this.videoData = videoData;
        
        // 获取分类信息
        if (this.videoData.categoryId) {
          const categoryRes = await db.collection(dbCategory)
            .where({ categoryId: this.videoData.categoryId })
            .get();
            
          if (categoryRes.result.data.length > 0) {
            this.categoryName = categoryRes.result.data[0].name;
          } else {
            this.categoryName = this.videoData.categoryId;
          }
        }
      } catch (error) {
        console.error('加载视频数据失败:', error);
        uni.showToast({
          title: error.message || '视频加载失败',
          icon: 'none'
        });
        throw error;
      }
    },
    
    // 加载进度数据及相关用户信息
    async loadProgressData() {
      try {
        const progressRes = await db.collection(dbProgress)
          .where({
            videoId: this.videoId
          })
          .orderBy('updateTime', 'desc')
          .get();
        
        this.progressList = progressRes.result.data;
        
        // 获取所有用户ID
        const userIds = [...new Set(this.progressList.map(p => p.userId))];
        
        // 批量获取用户信息
        if (userIds.length > 0) {
          const userRes = await db.collection(dbUser)
            .where({
              _id: db.command.in(userIds)
            })
            .field({
              _id: true,
              nickname: true,
              avatar: true
            })
            .get();
            
          // 建立用户信息映射
          userRes.result.data.forEach(user => {
            this.userMap.set(user._id, user);
          });
        }
        
        // 计算进度摘要
        if (this.progressList.length > 0) {
          const totalUsers = this.progressList.length;
          const completedUsers = this.progressList.filter(p => p.completed).length;
          this.progressSummary = `完成率: ${completedUsers}/${totalUsers} (${Math.round(completedUsers/totalUsers*100) || 0}%)`;
        }
      } catch (error) {
        console.error('加载进度数据失败:', error);
        throw error;
      }
    },
    
    // 获取用户显示名称
    getUserDisplayName(progress) {
      const user = this.userMap.get(progress.userId);
      return user ? user.nickname || progress.userId : progress.userId;
    },
    
    // 视频播放错误
    onVideoError(e) {
      console.error('视频播放错误:', e);
      uni.showToast({
        title: '视频播放失败，请稍后再试',
        icon: 'none'
      });
    },
    
    // 格式化时间
    formatDuration(seconds) {
      if (!seconds && seconds !== 0) return '-';
      
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = seconds % 60;
      return `${minutes}:${remainingSeconds.toString().padStart(2, '0')}`;
    },
    
    // 格式化日期
    formatDate(timestamp) {
      if (!timestamp) return '-';
      
      const date = new Date(timestamp);
      return `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}-${date.getDate().toString().padStart(2, '0')} ${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}`;
    },
    
    // 返回上一页
    goBack() {
      uni.navigateBack();
    }
  }
};
</script>

<style>
.preview-container {
  padding: 20px;
}

.video-title {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 15px;
}

.video-player-container {
  position: relative;
  width: 100%;
  margin-bottom: 20px;
}

.video-player {
  width: 100%;
  height: 225px;
  background-color: #000;
}

.video-placeholder {
  width: 100%;
  height: 225px;
  background-color: #f0f0f0;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #999;
}

.section-title {
  font-size: 18px;
  font-weight: bold;
  margin: 20px 0 10px 0;
  border-left: 4px solid #007AFF;
  padding-left: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.progress-summary {
  font-size: 14px;
  color: #666;
  font-weight: normal;
}

.progress-section, .video-info-section {
  background-color: #f8f8f8;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 20px;
}

.info-item {
  display: flex;
  margin-bottom: 10px;
}

.info-label {
  width: 100px;
  color: #666;
}

.info-value {
  flex: 1;
}

.description-item {
  margin-top: 10px;
}

.description-content {
  display: block;
  margin-top: 5px;
  color: #333;
  line-height: 1.5;
}

.progress-list {
  margin-top: 10px;
}

.progress-header, .progress-row {
  display: flex;
  padding: 10px 0;
  border-bottom: 1px solid #e0e0e0;
}

.progress-header {
  font-weight: bold;
  color: #333;
}

.progress-col {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.user-col {
  flex: 2;
}

.duration-col {
  flex: 1;
  text-align: center;
}

.status-col {
  flex: 1;
  text-align: center;
}

.completed {
  color: #4cd964;
}

.no-progress {
  color: #999;
  font-style: italic;
  padding: 10px 0;
  text-align: center;
}

.button-group {
  display: flex;
  justify-content: center;
  margin-top: 30px;
}

.uni-button {
  padding: 10px 0;
  width: 120px;
  font-size: 16px;
  border-radius: 4px;
}
</style>