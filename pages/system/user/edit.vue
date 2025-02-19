# pages/user/edit.vue
<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <uni-forms-item name="userId" label="用户ID" required>
        <uni-easyinput v-model="formData.userId" :clearable="false" disabled />
      </uni-forms-item>

      <uni-forms-item name="nickname" label="用户昵称" required>
        <uni-easyinput v-model="formData.nickname" :clearable="false" placeholder="请输入用户昵称" />
      </uni-forms-item>

      <uni-forms-item name="avatarUrl" label="头像URL" required>
        <view class="avatar-container">
          <image 
            class="avatar-preview" 
            :src="formData.avatarUrl || '/static/avatar.png'" 
            mode="aspectFill"
            @error="onAvatarError"
          />
          <uni-easyinput v-model="formData.avatarUrl" :clearable="false" placeholder="请输入头像URL" />
        </view>
      </uni-forms-item>

      <uni-forms-item v-if="showPassword" name="password" label="重置密码" key="password">
        <uni-easyinput v-model="formData.password" type="password" :clearable="false" placeholder="请输入新密码">
          <view slot="right" class="cancel-reset-password-btn" @click="trigger">取消</view>
        </uni-easyinput>
      </uni-forms-item>
      <uni-forms-item v-else label="重置密码">
        <span class="reset-password-btn" @click="trigger">点击重置密码</span>
      </uni-forms-item>

      <uni-forms-item name="phone" label="手机号码" required>
        <uni-easyinput v-model="formData.phone" :clearable="false" placeholder="请输入手机号码" />
      </uni-forms-item>

      <uni-forms-item name="email" label="电子邮件" required>
        <uni-easyinput v-model="formData.email" :clearable="false" placeholder="请输入电子邮件" />
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
          :clearable="false"
          placeholder="请输入用户简介" 
        />
      </uni-forms-item>

      <uni-forms-item name="registerTime" label="注册时间">
        <uni-easyinput v-model="formattedRegisterTime" disabled />
      </uni-forms-item>

      <uni-forms-item name="lastLoginTime" label="最后登录">
        <uni-easyinput v-model="formattedLastLoginTime" disabled />
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
const dbCollectionName = 'User';

export default {
  data() {
    return {
      formDataId: '',
      showPassword: false,
      formData: {
        userId: '',
        nickname: '',
        avatarUrl: '',
        password: undefined,
        phone: '',
        email: '',
        status: 'active',
        bio: '',
        registerTime: 0,
        lastLoginTime: 0,
        token: ''
      },
      rules: {
        userId: {
          rules: [{
            required: true,
            errorMessage: '用户ID不能为空'
          }]
        },
        nickname: {
          rules: [{
            required: true,
            errorMessage: '用户昵称不能为空'
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
      this.formDataId = option.id
      // 获取用户详情
      this.getDetail(option.id)
    } else {
      // 获取路由参数中的数据
      const eventChannel = this.getOpenerEventChannel()
      eventChannel.on('editData', (data) => {
        if (data) {
          this.formData = {
            ...this.formData,
            ...data,
            password: undefined // 重置密码字段
          }
          this.formDataId = data.userId // 保存ID用于更新
        }
      })
    }
  },

  methods: {
    // 头像加载失败处理
    onAvatarError() {
      this.formData.avatarUrl = '/static/avatar.png'
    },

    // 切换密码重置显示
    trigger() {
      this.showPassword = !this.showPassword
      if (!this.showPassword) {
        this.formData.password = undefined
      }
    },

    // 格式化时间戳
    formatDateTime(timestamp) {
      if (!timestamp) {
        return ''
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

    // 获取用户详情
    getDetail(id) {
      uni.showLoading({
        mask: true
      })
      
      console.log('查询ID:', id) // 调试输出

      db.collection(dbCollectionName)
        .where(`userId == "${id}"`)
        .get()
        .then(res => {
          console.log('查询结果:', res) // 调试输出
          const data = res.result.data[0]
          if (data) {
            this.formData = Object.assign({}, this.formData, data)
          } else {
            uni.showModal({
              content: '未找到用户数据',
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

    // 提交表单
    submitForm() {
      this.$refs.form.validate().then(res => {
        uni.showLoading({
          title: '保存中...',
          mask: true
        })

        // 只提取需要更新的字段
        const updateData = {
          userId: this.formData.userId,   // 主键字段
          nickname: this.formData.nickname,
          avatarUrl: this.formData.avatarUrl,
          email: this.formData.email,
          phone: this.formData.phone,
          status: this.formData.status,
          bio: this.formData.bio || ''
        }

        // 如果有设置新密码，添加密码字段
        if (this.showPassword && this.formData.password) {
          updateData.password = this.formData.password
        }

        // 删除undefined字段
        Object.keys(updateData).forEach(key => {
          if (updateData[key] === undefined) {
            delete updateData[key]
          }
        })

        // 确保不传递_id字段
        if (updateData._id) {
          delete updateData._id
        }

        console.log('更新数据:', updateData) // 调试输出

        // 更新数据
        db.collection(dbCollectionName)
          .where(`userId == "${this.formData.userId}"`)
          .update(updateData)
          .then(() => {
            uni.showToast({
              title: '保存成功'
            })
            const eventChannel = this.getOpenerEventChannel();
            eventChannel.emit('refreshData')
            setTimeout(() => uni.navigateBack(), 500)
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
      }).catch(errors => {
        console.error('表单验证错误:', errors)
      })
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

.uni-forms-item-flex-center-x {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
}
</style>