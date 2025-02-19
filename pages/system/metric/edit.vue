<template>
 <view class="uni-container">
   <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
     <uni-forms-item name="metricId" label="指标ID" required>
       <uni-easyinput v-model="formData.metricId" :clearable="false" placeholder="请输入指标ID" :disabled="!!formDataId"/>
     </uni-forms-item>
     
     <!-- 套餐选择 -->
     <uni-forms-item name="packageId" label="套餐" required>
       <view class="select-box">
         <picker @change="onPackageChange" :value="packageIndex" :range="packageList" range-key="package_name">
           <view class="picker-item">
             {{ formData.packageId ? getPackageName(formData.packageId) : '请选择套餐' }}
           </view>
         </picker>
       </view>
     </uni-forms-item>

     <!-- 代理选择 -->
     <uni-forms-item name="agentId" label="代理" required>
       <view class="select-box">
         <picker @change="onAgentChange" :value="agentIndex" :range="agentList" range-key="identifier">
           <view class="picker-item">
             {{ formData.agentId ? getAgentIdentifier(formData.agentId) : '请选择代理' }}
           </view>
         </picker>
       </view>
     </uni-forms-item>
     
     <uni-forms-item name="metricname" label="指标名称" required>
       <uni-easyinput v-model="formData.metricname" :clearable="false" placeholder="请输入指标名称" />
     </uni-forms-item>
     
     <uni-forms-item name="description" label="描述">
       <uni-easyinput type="textarea" v-model="formData.description" placeholder="请输入描述" />
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
const dbCollectionName = 'Metric';

export default {
 data() {
   return {
     formDataId: '',
     packageList: [], // 套餐列表
     packageIndex: -1, // 选中的套餐索引
     agentList: [], // 代理列表
     agentIndex: -1, // 选中的代理索引
     formData: {
       metricId: '',
       packageId: '',
       agentId: '',
       metricname: '',
       description: ''
     },
     rules: {
       metricId: {
         rules: [{
           required: true,
           errorMessage: '请输入指标ID'
         },
         {
           pattern: '^[a-zA-Z0-9]+$',
           errorMessage: '指标ID只能包含字母和数字'
         }]
       },
       packageId: {
         rules: [{
           required: true,
           errorMessage: '请选择套餐'
         }]
       },
       agentId: {
         rules: [{
           required: true,
           errorMessage: '请选择代理'
         }]
       },
       metricname: {
         rules: [{
           required: true,
           errorMessage: '请输入指标名称'
         }]
       }
     }
   }
 },
 
 async onLoad(e) {
   // 初始化数据
   await Promise.all([
     this.getPackageList(),
     this.getAgentList()
   ]);
   
   if (e.id) {
     this.formDataId = e.id;
     this.getDetail(e.id);
   }
 },
 
 methods: {
   // 获取套餐列表
   async getPackageList() {
     try {
       const res = await db.collection('Package')
         .get();
         
       if (res.result && res.result.data) {
         this.packageList = res.result.data;
         console.log('获取到的套餐列表:', this.packageList);
       } else {
         throw new Error('未获取到套餐数据');
       }
     } catch (err) {
       console.error('获取套餐列表错误:', err);
       uni.showModal({
         content: '获取套餐列表失败: ' + (err.message || '未知错误'),
         showCancel: false
       });
     }
   },
   
   // 获取代理列表
   async getAgentList() {
     try {
       const res = await db.collection('Agent')
         .where('status == "normal"')
         .get();
       
       if (res.result && res.result.data) {
         this.agentList = res.result.data;
         console.log('获取到的代理列表:', this.agentList);
       } else {
         throw new Error('未获取到代理数据');
       }
     } catch (err) {
       console.error('获取代理列表错误:', err);
       uni.showModal({
         content: '获取代理列表失败: ' + (err.message || '未知错误'),
         showCancel: false
       });
     }
   },
   
   // 套餐选择改变
   onPackageChange(e) {
     const index = e.detail.value;
     this.packageIndex = index;
     const selectedPackage = this.packageList[index];
     if (selectedPackage) {
       this.formData.packageId = selectedPackage.package_id.toString();
       console.log('选中的套餐:', selectedPackage);
     }
   },
   
   // 代理选择改变
   onAgentChange(e) {
     const index = e.detail.value;
     this.agentIndex = index;
     const selectedAgent = this.agentList[index];
     if (selectedAgent) {
       this.formData.agentId = selectedAgent.agentId;
       console.log('选中的代理:', selectedAgent);
     }
   },
   
   // 获取套餐名称
   getPackageName(packageId) {
     const pkg = this.packageList.find(item => item.package_id.toString() === packageId);
     return pkg ? `${pkg.package_name} (${pkg.package_id})` : '请选择套餐';
   },
   
   // 获取代理标识符
   getAgentIdentifier(agentId) {
     const agent = this.agentList.find(item => item.agentId === agentId);
     return agent ? agent.identifier : '请选择代理';
   },

   // 提交表单
   submitForm() {
     this.$refs.form.submit();
   },

   // 表单提交
   submit(event) {
     const {
       value,
       errors
     } = event.detail;

     if (errors) {
       return;
     }

     uni.showLoading({
       title: this.formDataId ? '修改中...' : '新增中...',
       mask: true
     });

     if (!this.formDataId) {
       db.collection(dbCollectionName)
         .where('metricId == $1', [value.metricId])
         .get()
         .then(res => {
           if (res.result.data.length > 0) {
             uni.showModal({
               content: '指标ID已存在',
               showCancel: false
             });
             uni.hideLoading();
           } else {
             this.submitData(value);
           }
         })
         .catch(err => {
           uni.hideLoading();
           uni.showModal({
             content: err.message || '请求服务失败',
             showCancel: false
           });
         });
     } else {
       this.submitData(value);
     }
   },

   // 提交数据
   submitData(value) {
     // 去除所有字段的前后空格
     ['metricId', 'metricname'].forEach(key => {
       if (value[key]) {
         value[key] = value[key].trim();
       }
     });
     
     if (this.formDataId) {
       db.collection(dbCollectionName)
         .doc(this.formDataId)
         .update(value)
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
       db.collection(dbCollectionName)
         .add(value)
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
         .field('metricId,packageId,agentId,metricname,description')
         .get();
         
       const data = res.result.data[0];
       if (data) {
         this.formData = Object.assign({}, this.formData, data);
         // 设置选中的套餐和代理索引
         this.packageIndex = this.packageList.findIndex(item => item.package_id.toString() === data.packageId);
         this.agentIndex = this.agentList.findIndex(item => item.agentId === data.agentId);
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