<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <uni-forms-item name="agentId" label="Agent ID" required>
        <uni-easyinput v-model="formData.agentId" :clearable="false" placeholder="请输入Agent ID" :disabled="!!formDataId"/>
      </uni-forms-item>
      
      <uni-forms-item name="identifier" label="标识符" required>
        <uni-easyinput v-model="formData.identifier" :clearable="false" placeholder="请输入标识符" />
      </uni-forms-item>
      
      <uni-forms-item name="apiPath" label="API路径" required>
        <uni-easyinput v-model="formData.apiPath" :clearable="false" placeholder="请输入API路径" />
      </uni-forms-item>

      <uni-forms-item name="method" label="请求方式">
        <view class="select-box">
          <picker 
            @change="onMethodChange" 
            :value="methodIndex" 
            :range="methodOptions"
          >
            <view class="picker-item">
              {{ formData.method || 'POST' }}
            </view>
          </picker>
        </view>
      </uni-forms-item>

      <uni-forms-item name="authorization" label="API授权令牌" required>
        <uni-easyinput 
          type="password"
          v-model="formData.authorization" 
          :clearable="false" 
          placeholder="请输入API授权令牌" 
        />
      </uni-forms-item>

      <uni-forms-item name="workflowId" label="工作流ID" required>
        <uni-easyinput 
          v-model="formData.workflowId" 
          :clearable="false" 
          placeholder="请输入工作流ID" 
        />
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
const dbCollectionName = 'Agent';

const methodOptions = ['GET', 'POST', 'PUT', 'DELETE'];

export default {
  data() {
    return {
      formDataId: '',
      methodIndex: 1, // 默认选择POST
      methodOptions,
      formData: {
        agentId: '',
        identifier: '',
        apiPath: '',
        method: 'POST',
        authorization: '',
        workflowId: '',
        description: '',
        status: 'active'
      },
      rules: {
        agentId: {
          rules: [{
            required: true,
            errorMessage: '请输入Agent ID'
          },
          {
            pattern: '^[a-zA-Z0-9_-]+$',
            errorMessage: 'Agent ID只能包含字母、数字、下划线和连字符'
          }]
        },
        identifier: {
          rules: [{
            required: true,
            errorMessage: '请输入标识符'
          }]
        },
        apiPath: {
          rules: [{
            required: true,
            errorMessage: '请输入API路径'
          }]
        },
        authorization: {
          rules: [{
            required: true,
            errorMessage: '请输入API授权令牌'
          }]
        },
        workflowId: {
          rules: [{
            required: true,
            errorMessage: '请输入工作流ID'
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
    onMethodChange(e) {
      const index = e.detail.value;
      this.methodIndex = index;
      this.formData.method = this.methodOptions[index];
    },

    submitForm() {
      this.$refs.form.submit();
    },

    submit(event) {
      const { value, errors } = event.detail;

      if (errors) {
        return;
      }

      uni.showLoading({
        title: this.formDataId ? '修改中...' : '新增中...',
        mask: true
      });

      if (!this.formDataId) {
        db.collection(dbCollectionName)
          .where({
            agentId: value.agentId
          })
          .get()
          .then(res => {
            if (res.result.data.length > 0) {
              uni.showModal({
                content: 'Agent ID已存在',
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

    submitData(value) {
      const submitValue = {
        agentId: value.agentId.trim(),
        identifier: value.identifier.trim(),
        apiPath: value.apiPath.trim(),
        method: value.method || 'POST',
        authorization: value.authorization.trim(),
        workflowId: value.workflowId.trim(),
        description: value.description?.trim() || '',
        status: value.status || 'active'
      };
      
      if (this.formDataId) {
        db.collection(dbCollectionName)
          .doc(this.formDataId)
          .update(submitValue)
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
          .add(submitValue)
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

    getDetail(id) {
      uni.showLoading({
        mask: true
      });
      db.collection(dbCollectionName)
        .doc(id)
        .field('agentId,identifier,apiPath,method,authorization,workflowId,description,status')
        .get()
        .then((res) => {
          const data = res.result.data[0];
          if (data) {
            this.formData = Object.assign({}, this.formData, data);
            // 设置请求方式的索引
            this.methodIndex = this.methodOptions.indexOf(data.method || 'POST');
          }
        })
        .catch((err) => {
          uni.showModal({
            content: err.message || '请求服务失败',
            showCancel: false
          });
        })
        .finally(() => {
          uni.hideLoading();
        });
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