<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <!-- 基本信息 -->
      <view class="section-title">基本信息</view>
      
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

      <!-- 会员信息 -->
      <view class="section-title">会员信息</view>
      
      <uni-forms-item name="membertype" label="会员卡类型">
        <uni-data-select
          v-model="formData.membertype"
          :localdata="memberTypeOptions"
          :clear="false"
          @change="onMemberTypeChange"
        />
      </uni-forms-item>
      
      <uni-forms-item name="memberStatus" label="会员状态">
        <uni-data-select
          v-model="formData.memberStatus"
          :localdata="memberStatusOptions"
          :clear="false"
        />
      </uni-forms-item>
      
      <template v-if="formData.membertype === 'daily' || formData.membertype === 'monthly'">
        <uni-forms-item name="remainingDays" label="剩余天数">
          <uni-easyinput 
            v-model="formData.remainingDays" 
            type="number"
            :clearable="false"
            placeholder="请输入剩余天数" 
          />
        </uni-forms-item>
      </template>
      
      <template v-if="formData.membertype === 'times'">
        <uni-forms-item name="remainingTimes" label="剩余次数">
          <uni-easyinput 
            v-model="formData.remainingTimes" 
            type="number"
            :clearable="false"
            placeholder="请输入剩余次数" 
          />
        </uni-forms-item>
      </template>
      
      <template v-if="formData.membertype !== 'none'">
        <uni-forms-item name="memberStartTime" label="会员开始时间">
          <uni-datetime-picker 
            v-model="formData.memberStartTime" 
            type="datetime"
            return-type="timestamp"
          />
        </uni-forms-item>
        
        <uni-forms-item name="memberExpireTime" label="会员到期时间">
          <uni-datetime-picker 
            v-model="formData.memberExpireTime" 
            type="datetime"
            return-type="timestamp"
          />
        </uni-forms-item>
        
        <uni-forms-item name="usedTrial" label="是否体验卡">
          <switch 
            :checked="formData.usedTrial" 
            @change="e => formData.usedTrial = e.detail.value"
          />
        </uni-forms-item>
      </template>

      <!-- 时间信息 -->
      <view class="section-title">时间信息</view>
      
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
        token: '',
        // 会员相关字段
        memberStatus: 'none',
        membertype: 'none',
        remainingDays: 0,
        remainingTimes: 0,
        usedTrial: false,
        memberStartTime: 0,
        memberExpireTime: 0
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
        },
        remainingDays: {
          rules: [{
            validateFunction: (rule, value, data, callback) => {
              if ((data.membertype === 'daily' || data.membertype === 'monthly') && (value === undefined || Number(value) < 0)) {
                callback('剩余天数必须大于或等于0');
              } else {
                callback();
              }
            }
          }]
        },
        remainingTimes: {
          rules: [{
            validateFunction: (rule, value, data, callback) => {
              if (data.membertype === 'times' && (value === undefined || Number(value) < 0)) {
                callback('剩余次数必须大于或等于0');
              } else {
                callback();
              }
            }
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
      }],
      memberTypeOptions: [{
        value: 'none',
        text: '非会员'
      }, {
        value: 'daily',
        text: '日卡'
      }, {
        value: 'monthly',
        text: '月卡'
      }, {
        value: 'times',
        text: '次卡'
      }],
      memberStatusOptions: [{
        value: 'none',
        text: '非会员'
      }, {
        value: 'active',
        text: '有效'
      }, {
        value: 'expired',
        text: '已过期'
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
    // 会员类型变更处理
    onMemberTypeChange(e) {
      const memberType = e;
      
      // 处理会员类型变更
      if (memberType === 'none') {
        // 如果变为非会员，重置相关字段
        this.formData.memberStatus = 'none';
        this.formData.remainingDays = 0;
        this.formData.remainingTimes = 0;
        this.formData.memberExpireTime = 0;
        this.formData.memberStartTime = 0;
        this.formData.usedTrial = false;
      } else if (this.formData.memberStatus === 'none') {
        // 如果原本是非会员，现在选择了会员类型，则设置为有效状态
        this.formData.memberStatus = 'active';
        
        // 如果没有开始时间，设置为当前时间
        if (!this.formData.memberStartTime) {
          this.formData.memberStartTime = Date.now();
        }
        
        // 如果没有到期时间，根据会员类型设置默认到期时间
        if (!this.formData.memberExpireTime) {
          const expireDate = new Date();
          if (memberType === 'daily') {
            expireDate.setDate(expireDate.getDate() + 30);
          } else if (memberType === 'monthly') {
            expireDate.setMonth(expireDate.getMonth() + 1);
          } else if (memberType === 'times') {
            expireDate.setMonth(expireDate.getMonth() + 1);
          }
          this.formData.memberExpireTime = expireDate.getTime();
        }
      }
    },
    
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
            
            // 确保会员相关字段有默认值
            this.formData.memberStatus = this.formData.memberStatus || 'none'
            this.formData.membertype = this.formData.membertype || 'none'
            this.formData.remainingDays = this.formData.remainingDays || 0
            this.formData.remainingTimes = this.formData.remainingTimes || 0
            this.formData.usedTrial = !!this.formData.usedTrial
            this.formData.memberStartTime = this.formData.memberStartTime || 0
            this.formData.memberExpireTime = this.formData.memberExpireTime || 0
            
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
          bio: this.formData.bio || '',
          // 会员相关字段
          memberStatus: this.formData.memberStatus,
          membertype: this.formData.membertype,
          remainingDays: Number(this.formData.remainingDays || 0),
          remainingTimes: Number(this.formData.remainingTimes || 0),
          usedTrial: !!this.formData.usedTrial,
          memberStartTime: this.formData.memberStartTime || 0,
          memberExpireTime: this.formData.memberExpireTime || 0
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

.section-title {
  font-size: 16px;
  font-weight: bold;
  color: #333;
  margin: 20px 0 10px 0;
  padding-bottom: 8px;
  border-bottom: 1px solid #eee;
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
  width: 100px !important;
}

.uni-forms-item-flex-center-x {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
}
</style>