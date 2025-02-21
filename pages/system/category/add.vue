<template>
  <view class="container">
    <uni-forms ref="form" :model="formData" :rules="rules">
      <uni-forms-item label="类别ID" name="categoryId" required>
        <uni-easyinput 
          v-model="formData.categoryId" 
          placeholder="请输入类别ID"
          :disabled="!!id"
        />
      </uni-forms-item>

      <uni-forms-item label="分类名称" name="name" required>
        <uni-easyinput 
          v-model="formData.name" 
          placeholder="请输入分类名称"
        />
      </uni-forms-item>
      
      <uni-forms-item label="分类描述" name="description">
        <uni-easyinput 
          type="textarea" 
          v-model="formData.description"
          placeholder="请输入分类描述信息"
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
const dbCmd = db.command;
const dbName = 'Category';

export default {
  data() {
    return {
      id: '',
      formData: {
        categoryId: '',
        name: '',
        description: '',
        createdAt: null
      },
      rules: {
        categoryId: {
          rules: [
            { required: true, errorMessage: '请输入类别ID' }
          ]
        },
        name: {
          rules: [
            { required: true, errorMessage: '请输入分类名称' }
          ]
        }
      }
    }
  },

  async onLoad(options) {
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
          categoryId: this.formData.categoryId,
          name: this.formData.name.trim(),
          description: (this.formData.description || '').trim()
        };

        if (!this.id) {
          try {
            // 同时检查类别ID和名称是否已存在
            const [idRes, nameRes] = await Promise.all([
              db.collection(dbName).where('categoryId == $1', [submitData.categoryId]).get(),
              db.collection(dbName).where('name == $1', [submitData.name]).get()
            ]);
              
            if (idRes.result.data.length > 0) {
              uni.showModal({ content: '类别ID已存在', showCancel: false });
              return;
            }
            
            if (nameRes.result.data.length > 0) {
              uni.showModal({ content: '分类名称已存在', showCancel: false });
              return;
            }
            
            // 设置创建时间
            submitData.createdAt = new Date();
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
          // 编辑时同时检查名称和ID是否与其他记录重复
          const [idRes, nameRes] = await Promise.all([
            db.collection(dbName).where({
              categoryId: submitData.categoryId,
              _id: dbCmd.neq(this.id)
            }).get(),
            db.collection(dbName).where({
              name: submitData.name,
              _id: dbCmd.neq(this.id)
            }).get()
          ]);
            
          if (idRes.result.data.length > 0) {
            uni.showModal({ content: '类别ID已存在', showCancel: false });
            return;
          }
          
          if (nameRes.result.data.length > 0) {
            uni.showModal({ content: '分类名称已存在', showCancel: false });
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
          .field('categoryId,name,description,createdAt')
          .get();
          
        const data = res.result.data[0];
        if (data) {
          this.formData = data;
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