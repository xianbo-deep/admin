<template>
 <view class="uni-container">
   <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
     <!-- 显示自定义的 categoryId 字段 -->
     <uni-forms-item v-if="formData.categoryId" label="分类ID">
       <uni-easyinput 
         v-model="formData.categoryId" 
         :clearable="false" 
         disabled
       />
     </uni-forms-item>
     
     <uni-forms-item name="name" label="分类名称" required>
       <uni-easyinput v-model="formData.name" :clearable="false" placeholder="请输入分类名称（如：前端、后端、数据库）" />
     </uni-forms-item>
     
     <uni-forms-item name="description" label="分类描述">
       <uni-easyinput type="textarea" v-model="formData.description" placeholder="请输入分类描述" />
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
const dbCollectionName = 'Category';

export default {
 data() {
   return {
     formDataId: '', // MongoDB的_id
     formData: {
       categoryId: '', // 自定义的categoryId字段
       name: '',
       description: '',
       createdAt: null
     },
     rules: {
       name: {
         rules: [{
           required: true,
           errorMessage: '请输入分类名称'
         }]
       }
     }
   }
 },
 
 onLoad(e) {
   if (e.id) {
     this.formDataId = e.id;
     this.getDetail(e.id);
   }
 },
 
 methods: {
   // 提交表单
   submitForm() {
     this.$refs.form.submit();
   },

   // 表单提交
   submit(event) {
     const { value, errors } = event.detail;

     if (errors) {
       return;
     }

     uni.showLoading({
       title: this.formDataId ? '修改中...' : '新增中...',
       mask: true
     });

     // 检查分类名称是否已存在
     this.checkNameExists(value.name).then(exists => {
       if (exists) {
         uni.hideLoading();
         uni.showModal({
           content: '分类名称已存在',
           showCancel: false
         });
       } else {
         this.submitData(value);
       }
     }).catch(err => {
       uni.hideLoading();
       uni.showModal({
         content: err.message || '请求服务失败',
         showCancel: false
       });
     });
   },

   // 检查分类名称是否已存在
   async checkNameExists(name) {
     try {
       const where = {
         name: name
       };
       
       // 编辑模式下排除当前记录
       if (this.formDataId) {
         where._id = db.command.neq(this.formDataId);
       }
       
       const res = await db.collection(dbCollectionName)
         .where(where)
         .get();
         
       return res.result.data.length > 0;
     } catch (err) {
       console.error('检查分类名称错误:', err);
       throw err;
     }
   },

   // 提交数据
   submitData(value) {
     // 去除所有字段的前后空格
     ['name', 'description'].forEach(key => {
       if (value[key]) {
         value[key] = value[key].trim();
       }
     });
     
     if (this.formDataId) {
       // 编辑模式 - 不修改 categoryId
       const updateData = {
         name: value.name,
         description: value.description
       };
       
       db.collection(dbCollectionName)
         .doc(this.formDataId)
         .update(updateData)
         .then(() => {
           uni.showToast({
             title: '修改成功'
           });
           this.getOpenerEventChannel().emit('refreshData');
           setTimeout(() => uni.navigateBack(), 500);
         })
         .catch(err => {
           uni.showModal({
             content: err.message || '请求服务失败',
             showCancel: false
           });
         })
         .finally(() => {
           uni.hideLoading();
         });
     } else {
       // 新增模式，生成categoryId并添加创建时间
       const addData = {
         name: value.name,
         description: value.description,
         categoryId: db.command.getObjectId(), // 生成新的ObjectId作为categoryId
         createdAt: Date.now() // 使用时间戳
       };
       
       db.collection(dbCollectionName)
         .add(addData)
         .then(() => {
           uni.showToast({
             title: '新增成功'
           });
           this.getOpenerEventChannel().emit('refreshData');
           setTimeout(() => uni.navigateBack(), 500);
         })
         .catch(err => {
           uni.showModal({
             content: err.message || '请求服务失败',
             showCancel: false
           });
         })
         .finally(() => {
           uni.hideLoading();
         });
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
         .field('categoryId,name,description,createdAt')
         .get();
         
       const data = res.result.data[0];
       if (data) {
         this.formData = Object.assign({}, this.formData, data);
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
</style>