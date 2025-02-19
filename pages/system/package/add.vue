<template>
  <view class="container">
    <uni-forms ref="form" :model="formData" :rules="rules">
      <uni-forms-item label="套餐ID" name="package_id" required>
        <uni-easyinput 
          v-model="formData.package_id" 
          type="number"
          placeholder="请输入套餐ID"
          :disabled="!!id"
        />
      </uni-forms-item>
      
      <uni-forms-item label="套餐名称" name="package_name" required>
        <uni-easyinput 
          v-model="formData.package_name" 
          placeholder="请输入套餐名称"
        />
      </uni-forms-item>

      <uni-forms-item label="描述" name="description">
        <uni-easyinput 
          type="textarea" 
          v-model="formData.description"
          placeholder="请输入描述信息"
        />
      </uni-forms-item>

      <uni-forms-item label="状态" name="status" required>
        <view class="select-box">
          <picker 
            @change="onStatusChange" 
            :value="statusIndex" 
            :range="statusOptions" 
            range-key="label"
          >
            <view class="picker-box">
              <text class="picker-text">{{ getStatusLabel(formData.status) }}</text>
              <text class="picker-arrow">▼</text>
            </view>
          </picker>
        </view>
      </uni-forms-item>

      <!-- 指标选择区域 -->
      <view class="metrics-section">
        <view class="section-header">
          <text class="section-title">指标列表</text>
          <button @click="showMetricSelector" type="primary" size="mini">添加指标</button>
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
              @click="removeMetric(index)" 
              type="warn" 
              size="mini"
            >移除</button>
          </view>
        </view>
        <view v-else class="empty-metrics">
          暂无指标，请添加
        </view>
      </view>

      <view class="btn-group">
        <button @click="submitForm" type="primary" class="btn">提交</button>
        <button @click="goBack" class="btn">返回</button>
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
const dbName = 'Package';

const statusOptions = [
  { value: 'active', label: '启用' },
  { value: 'inactive', label: '禁用' }
];

export default {
  data() {
    return {
      id: '',
      statusIndex: 0,
      statusOptions,
      availableMetrics: [], // 可选的指标列表
      selectedMetricIds: [], // 临时存储选中的指标ID
      formData: {
        package_id: '',
        package_name: '',
        description: '',
        status: 'active',
        metrics: []
      },
      rules: {
        package_id: {
          rules: [
            { required: true, errorMessage: '请输入套餐ID' }
          ]
        },
        package_name: {
          rules: [
            { required: true, errorMessage: '请输入套餐名称' }
          ]
        },
        status: {
          rules: [
            { required: true, errorMessage: '请选择状态' }
          ]
        }
      }
    }
  },

  async onLoad(options) {
    await this.getMetricsList();
    
    if (options.id) {
      this.id = options.id;
      this.getDetail();
    }
  },

  methods: {
    async getMetricsList() {
      try {
        const res = await db.collection('Metric')
          .field('metricId,metricname,description')
          .get();
          
        if (res.result && res.result.data) {
          this.availableMetrics = res.result.data;
        }
      } catch (err) {
        uni.showModal({
          content: '获取指标列表失败',
          showCancel: false
        });
      }
    },

    getStatusLabel(value) {
      const option = statusOptions.find(opt => opt.value === value);
      return option ? option.label : '请选择状态';
    },

    onStatusChange(e) {
      const index = e.detail.value;
      this.statusIndex = index;
      this.formData.status = statusOptions[index].value;
    },

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

    async submitForm() {
      try {
        const valid = await this.$refs.form.validate();
        if (!valid) return;
        
        if (this.formData.metrics.length === 0) {
          uni.showModal({
            content: '请至少选择一个指标',
            showCancel: false
          });
          return;
        }

        uni.showLoading({ title: this.id ? '修改中...' : '新增中...' });

        const submitData = {
          package_id: parseInt(this.formData.package_id),
          package_name: this.formData.package_name.trim(),
          description: this.formData.description || '',
          status: this.formData.status,
          metrics: this.formData.metrics
        };

        if (!this.id) {
          // 检查套餐ID是否存在
          const existCheck = await db.collection(dbName)
            .where('package_id == $1', [submitData.package_id])
            .get();
            
          if (existCheck.result.data.length > 0) {
            throw new Error('套餐ID已存在');
          }
          
          await db.collection(dbName).add(submitData);
          uni.showToast({ title: '新增成功' });
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
          .field('package_id,package_name,description,status,metrics')
          .get();
          
        const data = res.result.data[0];
        if (data) {
          this.formData = data;
          this.statusIndex = statusOptions.findIndex(opt => opt.value === data.status);
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