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

export default {
	data() {
		return {
			formDataId: '',
			formData: {
				agentId: '',
				identifier: '',
				apiPath: '',
				description: '',
				status: 'normal'
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
		/**
		 * 提交表单
		 */
		submitForm() {
			this.$refs.form.submit();
		},

		/**
		 * 表单提交
		 * @param {Object} event 回调参数 Function(callback:{value,errors})
		 */
		submit(event) {
			const {
				value,
				errors
			} = event.detail;

			// 表单校验失败
			if (errors) {
				return;
			}

			uni.showLoading({
				title: this.formDataId ? '修改中...' : '新增中...',
				mask: true
			});

			// 如果是新增，先检查 agentId 是否已存在
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

		/**
		 * 提交数据到数据库
		 * @param {Object} value
		 */
		submitData(value) {
			// 去除所有字段的前后空格
			const submitValue = {
				agentId: value.agentId.trim(),
				identifier: value.identifier.trim(),
				apiPath: value.apiPath.trim(),
				description: value.description?.trim() || '',
				status: value.status || 'normal'
			};
			
			if (this.formDataId) {
				// 更新数据
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
				// 新增数据
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

		/**
		 * 获取表单数据
		 * @param {Object} id 数据库记录的 _id
		 */
		getDetail(id) {
			uni.showLoading({
				mask: true
			});
			db.collection(dbCollectionName)
				.doc(id)
				.field('agentId,identifier,apiPath,description,status')
				.get()
				.then((res) => {
					const data = res.result.data[0];
					if (data) {
						this.formData = Object.assign({}, this.formData, data);
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
</style>