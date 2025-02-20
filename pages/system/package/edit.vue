<template>
  <view class="uni-container">
    <uni-forms ref="form" v-model="formData" :rules="rules" validateTrigger="bind" @submit="submit">
      <uni-forms-item name="package_id" label="套餐ID" required>
        <uni-easyinput 
          v-model="formData.package_id" 
          type="number"
          :clearable="false" 
          placeholder="请输入套餐ID" 
          :disabled="!!formDataId"
        />
      </uni-forms-item>
      
      <uni-forms-item name="package_name" label="套餐名称" required>
        <uni-easyinput 
          v-model="formData.package_name" 
          :clearable="false" 
          placeholder="请输入套餐名称" 
        />
      </uni-forms-item>

      <uni-forms-item name="AgentId" label="Agent" required>
        <view class="select-box">
          <picker 
            @change="onAgentChange" 
            :value="agentIndex" 
            :range="agentOptions" 
            range-key="identifier"
          >
            <view class="picker-item">
              {{ getAgentLabel() }}
            </view>
          </picker>
        </view>
      </uni-forms-item>

      <uni-forms-item name="status" label="套餐状态" required>
        <view class="select-box">
          <picker 
            @change="onStatusChange" 
            :value="statusIndex" 
            :range="statusOptions" 
            range-key="label"
          >
            <view class="picker-item">
              {{ getStatusLabel(formData.status) }}
            </view>
          </picker>
        </view>
      </uni-forms-item>

      <!-- 指标列表管理 -->
      <view class="metrics-section">
        <view class="section-header">
          <text class="section-title">指标列表</text>
          <button 
            class="uni-button" 
            type="primary" 
            size="mini"
            @click="showMetricSelector"
          >添加指标</button>
        </view>
        
        <view class="metrics-list" v-if="formData.metrics.length">
          <view 
            v-for="(metric, index) in formData.metrics" 
            :key="metric.metric_id"
            class="metric-item"
          >
            <view class="metric-info">
              <text class="metric-name">{{metric.metric_name}}</text>
              <text class="metric-id">({{metric.metric_id}})</text>
            </view>
            <button 
              class="uni-button" 
              type="warn" 
              size="mini"
              @click="removeMetric(index)"
            >移除</button>
          </view>
        </view>
        <view v-else class="empty-metrics">
          暂无指标，请添加
        </view>
      </view>
      
      <view class="uni-button-group">
        <button 
          style="width: 100px;" 
          type="primary" 
          class="uni-button" 
          @click="submitForm"
        >提交</button>
        <navigator open-type="navigateBack" style="margin-left: 15px;">
          <button style="width: 100px;" class="uni-button">返回</button>
        </navigator>
      </view>
    </uni-forms>

    <!-- 指标选择弹窗 -->
    <uni-popup ref="metricSelector" type="dialog">
      <uni-popup-dialog
        title="选择指标"
        mode="base"
        :before-close="true"
        @close="closeMetricSelector"
        @confirm="confirmSelectMetrics"
      >
        <view class="metric-selector">
          <checkbox-group @change="onMetricSelect">
            <label 
              v-for="metric in availableMetrics" 
              :key="metric.metricId"
              class="metric-option"
            >
              <checkbox 
                :value="metric.metricId"
                :checked="isMetricSelected(metric.metricId)"
              />
              <text class="metric-label">{{metric.metricname}}</text>
            </label>
          </checkbox-group>
        </view>
      </uni-popup-dialog>
    </uni-popup>
  </view>
</template>

<script>
const db = uniCloud.database();
const dbCollectionName = 'Package';

const statusOptions = [
  { value: 'active', label: '启用' },
  { value: 'inactive', label: '禁用' }
];

export default {
  data() {
    return {
      formDataId: '',
      formData: {
        package_id: '',
        package_name: '',
        AgentId: '',
        status: 'active',
        metrics: []
      },
      statusIndex: 0,
      agentIndex: 0,
      statusOptions,
      agentOptions: [], // Agent列表
      availableMetrics: [], // 可选的指标列表
      selectedMetricIds: [], // 临时存储选中的指标ID
      rules: {
        package_id: {
          rules: [{
            required: true,
            errorMessage: '请输入套餐ID'
          }]
        },
        package_name: {
          rules: [{
            required: true,
            errorMessage: '请输入套餐名称'
          }]
        },
        AgentId: {
          rules: [{
            required: true,
            errorMessage: '请选择Agent'
          }]
        },
        status: {
          rules: [{
            required: true,
            errorMessage: '请选择套餐状态'
          }]
        }
      }
    }
  },
  
  async onLoad(e) {
    await this.getAgentList();
    if (e.id) {
      this.formDataId = e.id;
      await this.getDetail(e.id);
    }
    await this.getMetricsList();
  },
  
  methods: {
    // 获取Agent列表
    async getAgentList() {
      try {
        const res = await db.collection('Agent')
          .where('status == "normal"')
          .field('agentId,identifier')
          .get();
          
        if (res.result && res.result.data) {
          this.agentOptions = res.result.data;
        }
      } catch (err) {
        uni.showModal({
          content: '获取Agent列表失败：' + err.message,
          showCancel: false
        });
      }
    },

    // 获取Agent显示文本
    getAgentLabel() {
      if (!this.formData.AgentId || !this.agentOptions.length) {
        return '请选择Agent';
      }
      const agent = this.agentOptions.find(a => a.agentId === this.formData.AgentId);
      return agent ? agent.identifier : '请选择Agent';
    },

    // Agent选择改变
    onAgentChange(e) {
      const index = e.detail.value;
      this.agentIndex = index;
      this.formData.AgentId = this.agentOptions[index].agentId;
    },

    // 获取状态显示文本
    getStatusLabel(value) {
      const option = statusOptions.find(opt => opt.value === value);
      return option ? option.label : '请选择状态';
    },

    // 状态选择改变
    onStatusChange(e) {
      const index = e.detail.value;
      this.statusIndex = index;
      this.formData.status = statusOptions[index].value;
    },

    // 获取所有可用指标
    async getMetricsList() {
      try {
        const res = await db.collection('Metric')
          .field('metricId,metricname')
          .get();
          
        if (res.result && res.result.data) {
          this.availableMetrics = res.result.data;
        }
      } catch (err) {
        uni.showModal({
          content: '获取指标列表失败：' + err.message,
          showCancel: false
        });
      }
    },

    // 指标选择器相关方法...
    showMetricSelector() {
      this.selectedMetricIds = this.formData.metrics.map(m => m.metric_id);
      this.$refs.metricSelector.open();
    },

    closeMetricSelector() {
      this.$refs.metricSelector.close();
    },

    onMetricSelect(e) {
      this.selectedMetricIds = e.detail.value;
    },

    isMetricSelected(metricId) {
      return this.selectedMetricIds.includes(metricId);
    },

    confirmSelectMetrics() {
      const selectedMetrics = this.availableMetrics
        .filter(metric => this.selectedMetricIds.includes(metric.metricId))
        .map(metric => ({
          metric_id: metric.metricId,
          metric_name: metric.metricname
        }));
      
      this.formData.metrics = selectedMetrics;
      this.closeMetricSelector();
    },

    removeMetric(index) {
      uni.showModal({
        title: '提示',
        content: '确定要移除该指标吗？',
        success: (res) => {
          if (res.confirm) {
            this.formData.metrics.splice(index, 1);
          }
        }
      });
    },

    submitForm() {
      this.$refs.form.submit();
    },

    async submit(event) {
      const { value, errors } = event.detail;

      if (errors) {
        return;
      }

      uni.showLoading({
        title: this.formDataId ? '修改中...' : '新增中...',
        mask: true
      });

      value.package_id = parseInt(value.package_id);

      try {
        if (!this.formDataId) {
          const existCheck = await db.collection(dbCollectionName)
            .where('package_id == $1', [value.package_id])
            .get();
            
          if (existCheck.result.data.length > 0) {
            throw new Error('套餐ID已存在');
          }
        }
        
        await this.submitData(value);
        
        uni.showToast({
          title: this.formDataId ? '修改成功' : '新增成功'
        });
        
        this.getOpenerEventChannel().emit('refreshData');
        setTimeout(() => uni.navigateBack(), 500);
      } catch (err) {
        uni.showModal({
          content: err.message || '提交失败',
          showCancel: false
        });
      } finally {
        uni.hideLoading();
      }
    },

    async submitData(value) {
      if (this.formDataId) {
        await db.collection(dbCollectionName)
          .doc(this.formDataId)
          .update(value);
      } else {
        await db.collection(dbCollectionName)
          .add(value);
      }
    },

    async getDetail(id) {
      uni.showLoading({ mask: true });
      try {
        const res = await db.collection(dbCollectionName)
          .doc(id)
          .field('package_id,package_name,AgentId,status,metrics')
          .get();
          
        const data = res.result.data[0];
        if (data) {
          this.formData = Object.assign({}, this.formData, data);
          this.statusIndex = statusOptions.findIndex(opt => opt.value === data.status);
          // 设置Agent索引
          this.agentIndex = this.agentOptions.findIndex(opt => opt.agentId === data.AgentId);
        }
      } catch (err) {
        uni.showModal({
          content: err.message || '获取详情失败',
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
/* 样式部分保持不变 */
.uni-container {
  padding: 15px;
}

.uni-button-group {
  margin-top: 50px;
  display: flex;
  justify-content: center;
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

.metrics-section {
  margin-top: 20px;
  border-top: 1px solid #eee;
  padding-top: 20px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.section-title {
  font-size: 16px;
  font-weight: bold;
}

.metrics-list {
  border: 1px solid #eee;
  border-radius: 4px;
}

.metric-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid #eee;
}

.metric-item:last-child {
  border-bottom: none;
}

.metric-info {
  display: flex;
  align-items: center;
  gap: 8px;
}

.metric-name {
  font-size: 14px;
}

.metric-id {
  font-size: 12px;
  color: #666;
}

.empty-metrics {
  text-align: center;
  padding: 20px;
  color: #999;
}

.metric-selector {
  max-height: 300px;
  overflow-y: auto;
}

.metric-option {
  display: flex;
  align-items: center;
  padding: 8px 0;
}

.metric-label {
  margin-left: 8px;
}
</style>