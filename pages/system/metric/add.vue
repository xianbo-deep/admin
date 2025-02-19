<template>
  <view class="container">
    <uni-forms ref="form" :model="formData" :rules="rules">
      <uni-forms-item label="指标ID" name="metricId" required>
        <uni-easyinput 
          v-model="formData.metricId" 
          placeholder="请输入指标ID"
          :disabled="!!id"
        />
      </uni-forms-item>
      
      <!-- 套餐选择 -->
      <uni-forms-item label="套餐" name="packageId" required>
        <view class="select-box">
          <picker @change="onPackageChange" :value="packageIndex" :range="packageList" range-key="package_name">
            <view class="picker-box">
              <text class="picker-text">{{ formData.packageId ? getPackageName(formData.packageId) : '请选择套餐' }}</text>
              <text class="picker-arrow">▼</text>
            </view>
          </picker>
        </view>
      </uni-forms-item>
      
      <!-- 代理选择 -->
      <uni-forms-item label="代理" name="agentId" required>
        <view class="select-box">
          <picker @change="onAgentChange" :value="agentIndex" :range="agentList" range-key="identifier">
            <view class="picker-box">
              <text class="picker-text">{{ formData.agentId ? getAgentIdentifier(formData.agentId) : '请选择代理' }}</text>
              <text class="picker-arrow">▼</text>
            </view>
          </picker>
        </view>
      </uni-forms-item>
      
      <uni-forms-item label="指标名称" name="metricname" required>
        <uni-easyinput 
          v-model="formData.metricname" 
          placeholder="请输入指标名称"
        />
      </uni-forms-item>

      <uni-forms-item label="描述" name="description">
        <uni-easyinput 
          type="textarea" 
          v-model="formData.description"
          placeholder="请输入描述信息"
        />
      </uni-forms-item>

      <view class="btn-group">
        <button @click="submitForm" type="primary" class="btn">提交</button>
        <button @click="goBack" class="btn">返回</button>
      </view>
    </uni-forms>
  </view>
</template>

<script>
const db = uniCloud.database();
const dbName = 'Metric';

export default {
  data() {
    return {
      id: '',
      packageList: [], // 套餐列表
      packageIndex: -1, // 选中的套餐索引
      agentList: [], // 代理列表
      agentIndex: -1, // 选中的代理索引
      formData: {
        metricId: '',
        packageId: '',
        metricname: '',
        agentId: '',
        description: ''
      },
      rules: {
        metricId: {
          rules: [
            { required: true, errorMessage: '请输入指标ID' },
            { pattern: '^[a-zA-Z0-9_-]+$', errorMessage: '指标ID只能包含字母、数字、下划线和连字符' }
          ]
        },
        packageId: {
          rules: [{ required: true, errorMessage: '请选择套餐' }]
        },
        agentId: {
          rules: [{ required: true, errorMessage: '请选择代理' }]
        },
        metricname: {
          rules: [{ required: true, errorMessage: '请输入指标名称' }]
        }
      }
    }
  },

  async onLoad(options) {
    // 初始化数据
    await Promise.all([
      this.getPackageList(),
      this.getAgentList()
    ]);
    
    if (options.id) {
      this.id = options.id;
      this.getDetail();
    }
  },

  methods: {
    // 获取套餐列表
    async getPackageList() {
      try {
        const res = await db.collection('Package').get();
        if (res.result && res.result.data) {
          this.packageList = res.result.data;
          console.log('获取到的套餐列表:', this.packageList);
        }
      } catch (err) {
        console.error('获取套餐列表错误:', err);
        uni.showModal({
          content: '获取套餐列表失败',
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
        }
      } catch (err) {
        console.error('获取代理列表错误:', err);
        uni.showModal({
          content: '获取代理列表失败',
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
      }
    },
    
    // 代理选择改变
    onAgentChange(e) {
      const index = e.detail.value;
      this.agentIndex = index;
      const selectedAgent = this.agentList[index];
      if (selectedAgent) {
        this.formData.agentId = selectedAgent.agentId;
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

    async submitForm() {
      try {
        const valid = await this.$refs.form.validate();
        if (!valid) return;
        
        uni.showLoading({ title: this.id ? '修改中...' : '新增中...' });

        const submitData = {
          metricId: this.formData.metricId.trim(),
          packageId: this.formData.packageId,
          metricname: this.formData.metricname.trim(),
          agentId: this.formData.agentId,
          description: this.formData.description || ''
        };

        if (!this.id) {
          try {
            const res = await db.collection(dbName)
              .where('metricId == $1', [submitData.metricId])
              .get();
              
            if (res.result.data.length > 0) {
              uni.showModal({ content: '指标ID已存在', showCancel: false });
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
          .field('metricId,packageId,metricname,agentId,description')
          .get();
          
        const data = res.result.data[0];
        if (data) {
          this.formData = data;
          // 设置选中的套餐和代理索引
          this.packageIndex = this.packageList.findIndex(item => 
            item.package_id.toString() === data.packageId
          );
          this.agentIndex = this.agentList.findIndex(item => 
            item.agentId === data.agentId
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
</style>