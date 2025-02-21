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
            placeholder="请输入剩余天数" 
          />
        </uni-forms-item>
      </template>
      
      <template v-if="formData.membertype === 'times'">
        <uni-forms-item name="remainingTimes" label="剩余次数">
          <uni-easyinput 
            v-model="formData.remainingTimes" 
            type="number"
            placeholder="请输入剩余次数" 
          />
        </uni-forms-item>
      </template>
      
      <template v-if="formData.membertype !== 'none'">
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
        lastLoginTime: 0,
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
        },
        remainingDays: {
          rules: [{
            validateFunction: (rule, value, data, callback) => {
              if ((data.membertype === 'daily' || data.membertype === 'monthly') && (!value || value < 0)) {
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
              if (data.membertype === 'times' && (!value || value < 0)) {
                callback('剩余次数必须大于或等于0');
              } else {
                callback();
              }
            }
          }]
        },
        memberExpireTime: {
          rules: [{
            validateFunction: (rule, value, data, callback) => {
              if (data.membertype !== 'none' && data.memberStatus === 'active' && !value) {
                callback('会员到期时间不能为空');
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

  methods: {
    // 会员类型变更处理
    onMemberTypeChange(e) {
      const memberType = e;
      
      // 重置相关字段
      if (memberType === 'none') {
        this.formData.memberStatus = 'none';
        this.formData.remainingDays = 0;
        this.formData.remainingTimes = 0;
        this.formData.memberExpireTime = 0;
        this.formData.memberStartTime = 0;
        this.formData.usedTrial = false;
      } else {
        // 默认设置为有效状态
        this.formData.memberStatus = 'active';
        this.formData.memberStartTime = Date.now();
        
        // 根据会员类型设置默认值
        if (memberType === 'times') {
          this.formData.remainingDays = 0;
          this.formData.remainingTimes = 10; // 默认10次
          
          // 默认设置一个月的到期时间
          const expireDate = new Date();
          expireDate.setMonth(expireDate.getMonth() + 1);
          this.formData.memberExpireTime = expireDate.getTime();
        } else {
          this.formData.remainingTimes = 0;
          
          if (memberType === 'daily') {
            this.formData.remainingDays = 30; // 默认30天
            
            // 默认设置30天后的到期时间
            const expireDate = new Date();
            expireDate.setDate(expireDate.getDate() + 30);
            this.formData.memberExpireTime = expireDate.getTime();
          } else if (memberType === 'monthly') {
            this.formData.remainingDays = 30; // 默认30天
            
            // 默认设置一个月后的到期时间
            const expireDate = new Date();
            expireDate.setMonth(expireDate.getMonth() + 1);
            this.formData.memberExpireTime = expireDate.getTime();
          }
        }
      }
    },
    
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
          token: this.generateToken(),  // 生成token
          // 确保数值型字段为数字类型
          remainingDays: Number(this.formData.remainingDays || 0),
          remainingTimes: Number(this.formData.remainingTimes || 0)
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
  width: 100px !important;
}
</style>