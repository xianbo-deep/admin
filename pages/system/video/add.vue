<template>
  <view class="container">
    <uni-forms ref="form" :model="formData" :rules="rules">
      <uni-forms-item label="视频ID" name="videoId" required>
        <uni-easyinput 
          v-model="formData.videoId" 
          placeholder="请输入视频ID"
          :disabled="!!id"
        />
      </uni-forms-item>

      <uni-forms-item label="视频标题" name="title" required>
        <uni-easyinput 
          v-model="formData.title" 
          placeholder="请输入视频标题"
        />
      </uni-forms-item>
      
      <!-- 分类选择 -->
      <uni-forms-item label="所属分类" name="categoryId" required>
        <view class="select-box">
          <picker @change="onCategoryChange" :value="categoryIndex" :range="categoryList" range-key="name">
            <view class="picker-box">
              <text class="picker-text">{{ formData.categoryId ? getCategoryName(formData.categoryId) : '请选择分类' }}</text>
              <text class="picker-arrow">▼</text>
            </view>
          </picker>
        </view>
      </uni-forms-item>
      
      <!-- 视频URL -->
      <uni-forms-item label="视频URL" name="url" required>
        <uni-easyinput 
          v-model="formData.url" 
          placeholder="请输入视频URL地址"
          @blur="detectVideoDuration"
        />
      </uni-forms-item>

      <!-- 视频封面URL -->
      <uni-forms-item label="封面URL" name="cover">
        <uni-easyinput 
          v-model="formData.cover" 
          placeholder="请输入视频封面URL地址"
        />
      </uni-forms-item>

      <!-- 视频时长 -->
      <uni-forms-item label="视频时长(秒)" name="duration" required>
        <uni-easyinput 
          v-model="formData.duration" 
          type="number"
          placeholder="输入URL后自动检测或手动输入"
          :disabled="durationDetecting"
        />
        <view class="duration-status">
          <view v-if="durationDetecting" class="detecting">
            <view class="loading-circle"></view>
            <text>检测中...</text>
          </view>
          <text v-else-if="formData.duration" class="duration-text">
            约 {{formatDuration(formData.duration)}}
          </text>
        </view>
      </uni-forms-item>

      <uni-forms-item label="视频描述" name="description">
        <uni-easyinput 
          type="textarea" 
          v-model="formData.description"
          placeholder="请输入视频描述信息"
        />
      </uni-forms-item>

      <view class="btn-group">
        <button @click="submitForm" type="primary" class="btn">提交</button>
        <button @click="goBack" class="btn">返回</button>
      </view>
    </uni-forms>

    <!-- 用于检测视频时长的隐藏视频元素 -->
    <video 
      id="durationDetector" 
      :src="tempVideoUrl" 
      style="width:0; height:0; position:absolute; opacity:0;"
      @error="handleVideoError"
      @loadedmetadata="handleVideoLoaded"
    ></video>
  </view>
</template>

<script>
const db = uniCloud.database();
const dbCmd = db.command;
const dbName = 'Video';

export default {
  data() {
    return {
      id: '',
      categoryList: [], // 分类列表
      categoryIndex: -1, // 选中的分类索引
      tempVideoUrl: '', // 用于加载视频并获取时长
      durationDetecting: false, // 视频时长检测中状态
      formData: {
        videoId: '',
        title: '',
        categoryId: '',
        url: '',
        cover: '', // 新增封面URL字段
        description: '',
        duration: '',
        uploadTime: null
      },
      rules: {
        videoId: {
          rules: [
            { required: true, errorMessage: '请输入视频ID' }
          ]
        },
        title: {
          rules: [
            { required: true, errorMessage: '请输入视频标题' }
          ]
        },
        categoryId: {
          rules: [{ required: true, errorMessage: '请选择所属分类' }]
        },
        url: {
          rules: [
            { required: true, errorMessage: '请输入视频URL地址' },
            { pattern: /^https?:\/\/.+/, errorMessage: '请输入有效的URL地址' }
          ]
        },
        cover: {
          rules: [
            { pattern: /^https?:\/\/.+/, errorMessage: '请输入有效的URL地址' }
          ]
        },
        duration: {
          rules: [{
            required: true,
            errorMessage: '请输入视频时长'
          }, {
            type: 'number',
            min: 1,
            transform(value) {
              return Number(value);
            },
            errorMessage: '请输入有效的视频时长'
          }]
        },
        description: {
          rules: [{ maxLength: 500, errorMessage: '描述内容不能超过500字' }]
        }
      }
    }
  },

  async onLoad(options) {
    await this.getCategoryList();
    
    if (options.id) {
      this.id = options.id;
      this.getDetail();
    }
  },

  methods: {
    // 获取分类列表
    async getCategoryList() {
      try {
        const res = await db.collection('Category').get();
        if (res.result && res.result.data) {
          this.categoryList = res.result.data;
          console.log('获取到的分类列表:', this.categoryList);
        }
      } catch (err) {
        console.error('获取分类列表错误:', err);
        uni.showModal({
          content: '获取分类列表失败',
          showCancel: false
        });
      }
    },
    
    // 分类选择改变
    onCategoryChange(e) {
      const index = e.detail.value;
      this.categoryIndex = index;
      const selectedCategory = this.categoryList[index];
      if (selectedCategory) {
        this.formData.categoryId = selectedCategory.categoryId;
      }
    },
    
    // 获取分类名称
    getCategoryName(categoryId) {
      const category = this.categoryList.find(item => item.categoryId === categoryId);
      return category ? category.name : '请选择分类';
    },

    // 检测视频时长
    detectVideoDuration() {
      if (!this.formData.url || this.formData.url.trim() === '') {
        return;
      }
      
      // 设置检测状态
      this.durationDetecting = true;
      
      // 设置临时URL以加载视频
      this.tempVideoUrl = '';
      setTimeout(() => {
        this.tempVideoUrl = this.formData.url;
      }, 100);
      
      // 设置超时处理
      setTimeout(() => {
        if (this.durationDetecting) {
          this.durationDetecting = false;
          uni.showToast({
            icon: 'none',
            title: '时长检测超时，请手动输入'
          });
        }
      }, 15000); // 15秒超时
    },
    
    // 视频加载完成，获取时长
    handleVideoLoaded(e) {
      try {
        setTimeout(() => {
          const videoContext = uni.createVideoContext('durationDetector');
          
          videoContext.getDuration({
            success: (res) => {
              this.durationDetecting = false;
              if (res.duration && res.duration > 0) {
                // 将时长转换为整数秒
                this.formData.duration = Math.round(res.duration);
                uni.showToast({
                  icon: 'success',
                  title: '已获取视频时长'
                });
              } else {
                uni.showToast({
                  icon: 'none',
                  title: '无法获取时长，请手动输入'
                });
              }
            },
            fail: (err) => {
              console.error('获取时长失败:', err);
              this.durationDetecting = false;
              uni.showToast({
                icon: 'none',
                title: '获取时长失败，请手动输入'
              });
            }
          });
          
          // 停止视频播放，释放资源
          videoContext.stop();
        }, 500);
      } catch (error) {
        this.durationDetecting = false;
        console.error('视频时长检测错误:', error);
        uni.showToast({
          icon: 'none',
          title: '检测出错，请手动输入'
        });
      }
    },
    
    // 视频加载错误处理
    handleVideoError(err) {
      console.error('视频加载错误:', err);
      if (this.durationDetecting) {
        this.durationDetecting = false;
        uni.showToast({
          icon: 'none',
          title: '视频加载失败，请检查URL'
        });
      }
    },
    
    // 格式化时长显示
    formatDuration(seconds) {
      if (!seconds) return '00:00';
      
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = Math.floor(seconds % 60);
      
      if (minutes >= 60) {
        const hours = Math.floor(minutes / 60);
        const remainingMinutes = minutes % 60;
        return `${hours}:${String(remainingMinutes).padStart(2, '0')}:${String(remainingSeconds).padStart(2, '0')}`;
      }
      
      return `${String(minutes).padStart(2, '0')}:${String(remainingSeconds).padStart(2, '0')}`;
    },

    async submitForm() {
      try {
        const valid = await this.$refs.form.validate();
        if (!valid) return;
        
        uni.showLoading({ title: this.id ? '修改中...' : '新增中...' });

        const submitData = {
          videoId: this.formData.videoId.trim(),
          title: this.formData.title.trim(),
          categoryId: this.formData.categoryId,
          url: this.formData.url.trim(),
          cover: (this.formData.cover || '').trim(), // 添加封面字段处理
          description: (this.formData.description || '').trim(),
          duration: parseInt(this.formData.duration),
          uploadTime: this.id && this.formData.uploadTime ? this.formData.uploadTime : Date.now()
        };

        if (!this.id) {
          try {
            // 检查视频ID是否已存在
            const idRes = await db.collection(dbName)
              .where({
                videoId: submitData.videoId
              })
              .get();
              
            if (idRes.result.data.length > 0) {
              uni.showModal({ content: '视频ID已存在', showCancel: false });
              return;
            }
            
            await db.collection(dbName).add(submitData);
            uni.showToast({ title: '新增成功' });
          } catch (error) {
            uni.showModal({
              content: error.message || '新增失败',
              showCancel: false
            });
            return;
          }
        } else {
          // 编辑时检查除了自己之外是否有重复的videoId
          const idRes = await db.collection(dbName)
            .where({
              videoId: submitData.videoId,
              _id: dbCmd.neq(this.id)
            })
            .get();
            
          if (idRes.result.data.length > 0) {
            uni.showModal({ content: '视频ID已存在', showCancel: false });
            return;
          }
          
          await db.collection(dbName).doc(this.id).update(submitData);
          uni.showToast({ title: '修改成功' });
        }

        this.getOpenerEventChannel().emit('refreshData');
        setTimeout(() => uni.navigateBack(), 500);
        
      } catch (err) {
        uni.showModal({
          content: err.message || '操作失败',
          showCancel: false
        });
      } finally {
        uni.hideLoading();
      }
    },

    async getDetail() {
      try {
        uni.showLoading({ mask: true });
        const res = await db.collection(dbName).doc(this.id)
          .field('videoId,title,categoryId,url,cover,description,duration,uploadTime')
          .get();
          
        const data = res.result.data[0];
        if (data) {
          this.formData = data;
          // 设置选中的分类索引
          this.categoryIndex = this.categoryList.findIndex(item => 
            item.categoryId === data.categoryId
          );
        }
      } catch (err) {
        uni.showModal({
          content: err.message || '获取详情失败',
          showCancel: false
        });
      } finally {
        uni.hideLoading();
      }
    },

    goBack() {
      uni.navigateBack();
    }
  }
}
</script>

<style>
.container {
  padding: 30rpx;
}

.btn-group {
  margin-top: 100rpx;
  display: flex;
  justify-content: center;
  gap: 30rpx;
}

.btn {
  width: 200rpx;
}

.select-box {
  width: 100%;
}

.picker-box {
  height: 35px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 10px;
  border: 1px solid #eee;
  border-radius: 4px;
  background-color: #fff;
}

.picker-text {
  font-size: 14px;
  color: #333;
}

.picker-arrow {
  font-size: 12px;
  color: #999;
}

.duration-status {
  margin-top: 8px;
  display: flex;
  align-items: center;
}

.detecting {
  display: flex;
  align-items: center;
  color: #999;
  font-size: 12px;
}

.loading-circle {
  width: 14px;
  height: 14px;
  border: 2px solid rgba(0, 122, 255, 0.1);
  border-left-color: #007AFF;
  border-radius: 50%;
  margin-right: 6px;
  animation: spin 1s linear infinite;
}

.duration-text {
  color: #666;
  font-size: 12px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
</style>