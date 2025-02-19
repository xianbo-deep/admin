# pages/user/add.vue
<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <uni-forms-item name="userId" label="用户ID" required>
        <uni-easyinput v-model="formData.userId" placeholder="请输入用户ID" />
      </uni-forms-item>
      
      <uni-forms-item name="nickname" label="用户昵称" required>
        <uni-easyinput v-model="formData.nickname" placeholder="请输入用户昵称" />
      </uni-forms-item>
      
      <uni-forms-item name="password" label="登录密码" required>
        <uni-easyinput v-model="formData.password" type="password" placeholder="请输入登录密码" />
      </uni-forms-item>
      
      <uni-forms-item name="avatarUrl" label="头像URL" required>
        <view class="avatar-container">
          <image 
            class="avatar-preview" 
            :src="formData.avatarUrl || '/static/avatar.png'" 
            mode="aspectFill"
            @error="onAvatarError"
          />
          <uni-easyinput v-model="formData.avatarUrl" placeholder="请输入头像URL" />
        </view>
      </uni-forms-item>
      
      <uni-forms-item name="phone" label="手机号码" required>
        <uni-easyinput v-model="formData.phone" placeholder="请输入手机号码" />
      </uni-forms-item>
      
      <uni-forms-item name="email" label="电子邮件" required>
        <uni-easyinput v-model="formData.email" placeholder="请输入电子邮件" />
      </uni-forms-item>
      
      <uni-forms-item name="status" label="账户状态" required>
        <uni-data-select
          v-model="formData.status"
          :localdata="statusOptions"
          :clear="false"
        />
      </uni-forms-item>
      
      <uni-forms-item name="bio" label="用户简介">
        <uni-easyinput 
          v-model="formData.bio" 
          type="textarea"
          placeholder="请输入用户简介" 
        />
      </uni-forms-item>

      <view class="uni-button-group">
        <button type="primary" class="uni-button" @click="submitForm">提交</button>
        <button class="uni-button" @click="goBack">返回</button>
      </view>
    </uni-forms>
  </view>
</template>

<script>
const db = uniCloud.database();
const dbCollectionName = 'User';

export default {
  data() {
    return {
      formData: {
        userId: '',
        nickname: '',
        password: '',
        avatarUrl: '',
        email: '',
        phone: '',
        status: 'active',
        bio: '',
        registerTime: 0,
        lastLoginTime: 0
      },
      rules: {
        userId: {
          rules: [{
            required: true,
            errorMessage: '用户ID不能为空'
          }, {
            validateFunction: this.validateUserId
          }]
        },
        nickname: {
          rules: [{
            required: true,
            errorMessage: '用户昵称不能为空'
          }]
        },
        password: {
          rules: [{
            required: true,
            errorMessage: '登录密码不能为空'
          }, {
            minLength: 6,
            errorMessage: '密码长度不能小于6位'
          }]
        },
        avatarUrl: {
          rules: [{
            required: true,
            errorMessage: '头像URL不能为空'
          }]
        },
        email: {
          rules: [{
            required: true,
            errorMessage: '电子邮件不能为空'
          }, {
            format: 'email',
            errorMessage: '电子邮件格式不正确'
          }]
        },
        phone: {
          rules: [{
            required: true,
            errorMessage: '手机号码不能为空'
          }, {
            pattern: /^1\d{10}$/,
            errorMessage: '手机号码格式不正确'
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

  methods: {
    // 头像加载失败处理
    onAvatarError() {
      this.formData.avatarUrl = '/static/avatar.png'
    },

    // 验证用户ID是否已存在
    validateUserId(rule, value, data, callback) {
      // 检查用户ID格式
      if (!/^[a-zA-Z0-9_-]{4,20}$/.test(value)) {
        callback('用户ID只能包含字母、数字、下划线和横杠，长度4-20位')
        return
      }

      // 检查用户ID是否已存在
      db.collection(dbCollectionName)
        .where(`userId == "${value}"`)
        .count()
        .then(res => {
          if (res.result.total > 0) {
            callback('该用户ID已存在')
          } else {
            callback()
          }
        })
        .catch(err => {
          callback('验证用户ID时出错')
        })
    },

    // 生成唯一token
    generateToken() {
      return Math.random().toString(36).substring(2) + Date.now().toString(36)
    },

    // 提交表单
    submitForm() {
      this.$refs.form.validate().then(res => {
        uni.showLoading({
          title: '保存中...',
          mask: true
        })

        // 构建新用户数据
        const userData = {
          ...this.formData,
          registerTime: Date.now(),    // 注册时间
          lastLoginTime: Date.now(),   // 初始登录时间
          token: this.generateToken()  // 生成token
        }

        // 创建新用户
        db.collection(dbCollectionName)
          .add(userData)
          .then(() => {
            uni.showToast({
              title: '创建成功'
            })
            const eventChannel = this.getOpenerEventChannel();
            eventChannel.emit('refreshData')
            setTimeout(() => uni.navigateBack(), 500)
          })
          .catch(err => {
            uni.showModal({
              content: err.message || '创建失败',
              showCancel: false
            })
          })
          .finally(() => {
            uni.hideLoading()
          })
      }).catch(errors => {
        console.error('表单验证错误:', errors)
      })
    },

    // 返回上一页
    goBack() {
      uni.navigateBack()
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
  gap: 15px;
}

.uni-button {
  width: 120px;
}

.avatar-container {
  display: flex;
  align-items: center;
  gap: 15px;
}

.avatar-preview {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  border: 1px solid #eee;
  object-fit: cover;
}

::v-deep .uni-forms-item__label {
  width: 90px !important;
}
</style>