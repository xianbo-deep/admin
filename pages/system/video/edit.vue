<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <uni-forms-item name="videoId" label="视频ID" required>
        <uni-easyinput 
          v-model="formData.videoId" 
          :clearable="false" 
          placeholder="请输入视频ID"
          :disabled="!!formDataId"
        />
      </uni-forms-item>

      <uni-forms-item name="title" label="视频标题" required>
        <uni-easyinput v-model="formData.title" :clearable="false" placeholder="请输入视频标题" />
      </uni-forms-item>
      
      <!-- 分类选择 -->
      <uni-forms-item name="categoryId" label="所属分类" required>
        <view class="select-box">
          <picker @change="onCategoryChange" :value="categoryIndex" :range="categoryList" range-key="name">
            <view class="picker-item">
              {{ formData.categoryId ? getCategoryName(formData.categoryId) : '请选择分类' }}
            </view>
          </picker>
        </view>
      </uni-forms-item>

      <!-- 视频URL输入 -->
      <uni-forms-item name="url" label="视频URL" required>
        <uni-easyinput 
          v-model="formData.url" 
          :clearable="false" 
          placeholder="请输入视频URL"
          @blur="detectVideoDuration"
        />
      </uni-forms-item>
      
      <!-- 视频封面URL输入 -->
      <uni-forms-item name="cover" label="封面URL">
        <uni-easyinput 
          v-model="formData.cover" 
          :clearable="false" 
          placeholder="请输入视频封面URL"
        />
        <view class="cover-preview" v-if="formData.cover">
          <image 
            class="cover-image" 
            :src="formData.cover" 
            mode="aspectFill"
            @error="handleCoverError"
          ></image>
        </view>
      </uni-forms-item>
      
      <!-- 视频时长 -->
      <uni-forms-item name="duration" label="视频时长(秒)" required>
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
      
      <uni-forms-item name="description" label="视频描述">
        <uni-easyinput type="textarea" v-model="formData.description" placeholder="请输入视频描述" />
      </uni-forms-item>
      
      <view class="uni-button-group">
        <button style="width: 100px;" type="primary" class="uni-button" @click="submitForm">提交</button>
        <navigator open-type="navigateBack" style="margin-left: 15px;">
          <button style="width: 100px;" class="uni-button">返回</button>
        </navigator>
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
const dbCollectionName = 'Video';

export default {
  data() {
    return {
      formDataId: '',
      categoryList: [], // 分类列表
      categoryIndex: -1, // 选中的分类索引
      tempVideoUrl: '', // 用于加载视频并获取时长
      durationDetecting: false, // 视频时长检测中状态
      formData: {
        videoId: '',
        title: '',
        categoryId: '',
        url: '',
        cover: '', // 添加封面URL字段
        description: '',
        duration: '', // 添加时长字段
        uploadTime: null
      },
      rules: {
        videoId: {
          rules: [{
            required: true,
            errorMessage: '请输入视频ID'
          }]
        },
        title: {
          rules: [{
            required: true,
            errorMessage: '请输入视频标题'
          }]
        },
        categoryId: {
          rules: [{
            required: true,
            errorMessage: '请选择分类'
          }]
        },
        url: {
          rules: [{
            required: true,
            errorMessage: '请输入视频URL'
          }, {
            pattern: /^https?:\/\/.+/,
            errorMessage: '请输入有效的URL地址'
          }]
        },
        cover: {
          rules: [{
            pattern: /^https?:\/\/.+/,
            errorMessage: '请输入有效的URL地址'
          }]
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
        }
      }
    }
  },
  
  async onLoad(e) {
    await this.getCategoryList();
    
    if (e.id) {
      this.formDataId = e.id;
      this.getDetail(e.id);
    }
  },
  
  methods: {
    // 处理封面图片加载错误
    handleCoverError() {
      uni.showToast({
        icon: 'none',
        title: '封面图片加载失败，请检查URL'
      });
    },
    
    // 获取分类列表
    async getCategoryList() {
      try {
        const res = await db.collection('Category').get();
        if (res.result && res.result.data) {
          this.categoryList = res.result.data;
        } else {
          throw new Error('未获取到分类数据');
        }
      } catch (err) {
        console.error('获取分类列表错误:', err);
        uni.showModal({
          content: '获取分类列表失败: ' + (err.message || '未知错误'),
          showCancel: false
        });
      }
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
      
      // 设置超时处理，避免一直加载
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
        }, 500); // 稍微延迟以确保视频元数据已完全加载
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

    // 提交表单
    submitForm() {
      this.$refs.form.submit();
    },

    // 表单提交
    async submit(event) {
      const { value, errors } = event.detail;

      if (errors) {
        return;
      }

      uni.showLoading({
        title: this.formDataId ? '修改中...' : '添加中...',
        mask: true
      });

      try {
        // 检查videoId是否重复
        const videoIdCheck = await db.collection(dbCollectionName)
          .where({
            videoId: value.videoId,
            _id: this.formDataId ? dbCmd.neq(this.formDataId) : dbCmd.exists(true)
          })
          .get();

        if (videoIdCheck.result.data.length > 0) {
          uni.showModal({
            content: '视频ID已存在',
            showCancel: false
          });
          uni.hideLoading();
          return;
        }

        // 确保转换时长为整数
        const submitData = {
          ...value,
          duration: parseInt(value.duration),
          uploadTime: this.formData.uploadTime || new Date()
        };

        let result;
        if (this.formDataId) {
          // 更新操作
          result = await db.collection(dbCollectionName).doc(this.formDataId).update(submitData);
          uni.showToast({ title: '修改成功' });
        } else {
          // 新增操作
          result = await db.collection(dbCollectionName).add(submitData);
          uni.showToast({ title: '添加成功' });
        }

        // 通知父页面刷新数据
        const eventChannel = this.getOpenerEventChannel();
        if (eventChannel && eventChannel.emit) {
          eventChannel.emit('refreshData');
        }
        
        setTimeout(() => uni.navigateBack(), 500);
      } catch (err) {
        uni.showModal({
          content: err.message || '请求服务失败',
          showCancel: false
        });
      } finally {
        uni.hideLoading();
      }
    },

    // 获取详情
    async getDetail(id) {
      uni.showLoading({
        mask: true
      });
      try {
        const res = await db.collection(dbCollectionName)
          .doc(id)
          .field('videoId,title,categoryId,url,cover,description,duration,uploadTime')
          .get();
          
        const data = res.result.data[0];
        if (data) {
          // 确保 uploadTime 是时间戳格式
          if (data.uploadTime instanceof Date) {
            data.uploadTime = data.uploadTime.getTime();
          } else if (typeof data.uploadTime === 'string') {
            const timestamp = new Date(data.uploadTime).getTime();
            if (!isNaN(timestamp)) {
              data.uploadTime = timestamp;
            }
          }
          
          this.formData = Object.assign({}, this.formData, data);
          // 设置选中的分类索引
          this.categoryIndex = this.categoryList.findIndex(item => 
            item.categoryId === data.categoryId
          );
        }
      } catch (err) {
        uni.showModal({
          content: err.message || '请求服务失败',
          showCancel: false
        });
      } finally {
        uni.hideLoading();
      }
    }
  }
}
</script>

<style>
.uni-container {
  padding: 15px;
}

.uni-button-group {
  margin-top: 50px;
  display: flex;
  justify-content: center;
}

.uni-button {
  padding: 12px 20px;
  font-size: 14px;
  border-radius: 4px;
  line-height: 1;
  margin: 0;
}

.select-box {
  width: 100%;
}

.picker-item {
  height: 35px;
  line-height: 35px;
  padding: 0 10px;
  border: 1px solid #eee;
  border-radius: 4px;
  font-size: 14px;
  background-color: #fff;
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

/* 封面预览样式 */
.cover-preview {
  margin-top: 8px;
  width: 100%;
}

.cover-image {
  width: 120px;
  height: 68px;
  border-radius: 4px;
  border: 1px solid #eee;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
</style>