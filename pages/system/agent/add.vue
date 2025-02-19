<!-- add.vue -->
<template>
  <view class="container">
    <uni-forms ref="form" :model="formData" :rules="rules">
      <uni-forms-item label="Agent ID" name="agentId" required>
        <uni-easyinput 
          v-model="formData.agentId" 
          placeholder="请输入Agent ID"
          :disabled="!!id"
        />
      </uni-forms-item>
      
      <uni-forms-item label="标识符" name="identifier" required>
        <uni-easyinput 
          v-model="formData.identifier" 
          placeholder="请输入标识符"
        />
      </uni-forms-item>

      <uni-forms-item label="API路径" name="apiPath" required>
        <uni-easyinput 
          v-model="formData.apiPath" 
          placeholder="请输入API路径"
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
const dbName = 'Agent';

export default {
  data() {
    return {
      id: '',
      formData: {
        agentId: '',
        identifier: '',
        apiPath: '',
        description: '',
        status: 'normal'
      },
      rules: {
        agentId: {
          rules: [
            { required: true, errorMessage: '请输入Agent ID' },
            { pattern: '^[a-zA-Z0-9_-]+$', errorMessage: 'Agent ID只能包含字母、数字、下划线和连字符' }
          ]
        },
        identifier: {
          rules: [{ required: true, errorMessage: '请输入标识符' }]
        },
        apiPath: {
          rules: [{ required: true, errorMessage: '请输入API路径' }]
        }
      }
    }
  },

  onLoad(options) {
    if (options.id) {
      this.id = options.id;
      this.getDetail();
    }
  },

  methods: {
    async submitForm() {
      try {
        const valid = await this.$refs.form.validate();
        if (!valid) return;
        
        uni.showLoading({ title: this.id ? '修改中...' : '新增中...' });

        const submitData = {
          agentId: this.formData.agentId.trim(),
          identifier: this.formData.identifier.trim(),
          apiPath: this.formData.apiPath.trim(),
          description: this.formData.description || '',
          status: this.formData.status
        };

        if (!this.id) {
          // 检查Agent ID是否重复
          try {
            const res = await db.collection(dbName)
              .where({ agentId: submitData.agentId })
              .get();
              
            const duplicates = res?.data || [];
            if (duplicates.length > 0) {
              uni.showModal({ content: 'Agent ID已存在', showCancel: false });
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

        // 通知更新列表
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
        const { data } = await db.collection(dbName).doc(this.id)
          .field('agentId,identifier,apiPath,description,status')
          .get();
          
        if (data[0]) {
          this.formData = data[0];
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
</style>