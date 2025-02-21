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
       />
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
     formData: {
       videoId: '',
       title: '',
       categoryId: '',
       url: '',
       description: '',
       uploadTime: null, // 初始化为null
       learningProgress: [] // 初始化为空数组
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
       title: '修改中...',
       mask: true
     });

     try {
       // 检查videoId是否重复
       const videoIdCheck = await db.collection(dbCollectionName)
         .where({
           videoId: value.videoId,
           _id: dbCmd.neq(this.formDataId)
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

       // 准备提交数据 - 使用原有的时间戳和学习进度
       const submitData = {
         ...value,
         uploadTime: this.formData.uploadTime,
         learningProgress: this.formData.learningProgress
       };

       await db.collection(dbCollectionName).doc(this.formDataId).update(submitData);
       uni.showToast({ title: '修改成功' });

       this.getOpenerEventChannel().emit('refreshData');
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
         .field('videoId,title,categoryId,url,description,uploadTime,learningProgress')
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
</style>