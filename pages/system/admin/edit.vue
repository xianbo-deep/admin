# pages/adminManagement/edit.vue
<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <uni-forms-item name="adminId" label="管理员ID" required>
        <uni-easyinput v-model="formData.adminId" :clearable="false" disabled />
      </uni-forms-item>

      <uni-forms-item name="account" label="管理员账号" required>
        <uni-easyinput v-model="formData.account" :clearable="false" placeholder="请输入管理员账号" />
      </uni-forms-item>

      <uni-forms-item name="nickname" label="管理员昵称" required>
        <uni-easyinput v-model="formData.nickname" :clearable="false" placeholder="请输入管理员昵称" />
      </uni-forms-item>

      <uni-forms-item v-if="showPassword" name="password" label="重置密码" key="password">
        <uni-easyinput v-model="formData.password" type="password" :clearable="false" placeholder="请输入新密码">
          <view slot="right" class="cancel-reset-password-btn" @click="trigger">取消</view>
        </uni-easyinput>
      </uni-forms-item>
      <uni-forms-item v-else label="重置密码">
        <span class="reset-password-btn" @click="trigger">点击重置密码</span>
      </uni-forms-item>

      <uni-forms-item name="status" label="账户状态" required>
        <uni-data-select
          v-model="formData.status"
          :localdata="statusOptions"
          :clear="false"
        />
      </uni-forms-item>

      <uni-forms-item name="registerTime" label="注册时间">
        <uni-easyinput v-model="formattedRegisterTime" disabled />
      </uni-forms-item>

      <uni-forms-item name="lastLoginTime" label="最后登录">
        <uni-easyinput v-model="formattedLastLoginTime" disabled />
      </uni-forms-item>

      <view class="uni-button-group">
        <button type="primary" class="uni-button" @click="submitForm">提交</button>
        <navigator open-type="navigateBack" style="margin-left: 15px;">
          <button class="uni-button">返回</button>
        </navigator>
      </view>
    </uni-forms>
  </view>
</template>

<script>
const db = uniCloud.database()
const dbCollectionName = 'Admin'
import { store } from '../../../uni_modules/uni-id-pages/common/store.js'
export default {
  data() {
    return {
      formDataId: '',
      showPassword: false,
      formData: {
        adminId: '',
        nickname: '',
        account: '',
        password: undefined,
        status: 'active',
        registerTime: 0,
        lastLoginTime: 0
      },
      rules: {
        account: {
          rules: [{
            required: true,
            errorMessage: '管理员账号不能为空'
          }, {
            minLength: 3,
            errorMessage: '账号长度不能小于3位'
          }]
        },
        nickname: {
          rules: [{
            required: true,
            errorMessage: '管理员昵称不能为空'
          }]
        },
        password: {
          rules: [{
            minLength: 6,
            errorMessage: '密码长度不能小于6位'
          }]
        }
      },
      statusOptions: [{
        value: 'active',
        text: '正常'
      }, {
        value: 'inactive',
        text: '未激活'
      }, {
        value: 'banned',
        text: '已禁用'
      }]
    }
  },

  computed: {
    formattedRegisterTime() {
      return this.formatDateTime(this.formData.registerTime)
    },
    formattedLastLoginTime() {
      return this.formatDateTime(this.formData.lastLoginTime)
    }
  },

  onLoad(option) {
    if (option.id) {
      this.getDetail(option.id)
    }
  },

  methods: {
    trigger() {
      this.showPassword = !this.showPassword
      if (!this.showPassword) {
        this.formData.password = undefined
      }
    },

    formatDateTime(timestamp) {
      if (!timestamp) {
        return '';
      }
      
      try {
        const date = new Date(timestamp);
        return date.toLocaleString('zh-CN', { 
          timeZone: 'Asia/Shanghai',
          year: 'numeric',
          month: '2-digit',
          day: '2-digit',
          hour: '2-digit',
          minute: '2-digit',
          second: '2-digit',
          hour12: false
        }).replace(/\//g, '-');
      } catch (error) {
        console.error('时间格式化错误:', error);
        return '';
      }
    },

    getDetail(id) {
      uni.showLoading({
        mask: true
      })
      
      db.collection(dbCollectionName)
        .where(`adminId == "${id}"`)
        .get()
        .then(res => {
          const data = res.result.data[0]
          if (data) {
            this.formData = {
              ...this.formData,
              ...data,
              password: undefined
            }
            this.formDataId = data.adminId
          } else {
            uni.showModal({
              content: '未找到管理员信息',
              showCancel: false,
              success: () => {
                uni.navigateBack()
              }
            })
          }
        })
        .catch(err => {
          uni.showModal({
            content: err.message || '请求服务失败',
            showCancel: false
          })
        })
        .finally(() => {
          uni.hideLoading()
        })
    },

    async submitForm() {
      try {
        await this.$refs.form.validate()
        uni.showLoading({
          title: '保存中...',
          mask: true
        })
    
        const updateData = {
          nickname: this.formData.nickname,  
          account: this.formData.account,
          status: this.formData.status
        }
        
        if (this.showPassword && this.formData.password) {
          updateData.password = this.formData.password
        }
    
        await db.collection('uni-id-users')
          .doc(store.userInfo._id)
          .update({
            username: this.formData.account,
            nickname: this.formData.nickname  
        })
    
          // 更新管理员信息
          await db.collection(dbCollectionName)
            .where(`adminId == "${this.formDataId}"`)
            .update(updateData)
    
          uni.showToast({ 
            title: '更新成功',
            duration: 2000
          })
    
          const eventChannel = this.getOpenerEventChannel()
          eventChannel.emit('refreshData')
    
          setTimeout(() => {
            uni.navigateBack({
              delta: 1
            })
          }, 500)
        
    
      } catch(err) {
        console.error('更新失败:', err)
        uni.showModal({
          content: err.message || '更新失败',
          showCancel: false
        })
      } finally {
        uni.hideLoading()
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
  margin-top: 30px;
  display: flex;
  justify-content: center;
}

.uni-button {
  width: 120px;
}

.reset-password-btn {
  line-height: 36px;
  color: #007AFF;
  text-decoration: underline;
  cursor: pointer;
}

.cancel-reset-password-btn {
  color: #007AFF;
  padding-right: 10px;
  cursor: pointer;
}

::v-deep .uni-forms-item__label {
  width: 90px !important;
}
</style>