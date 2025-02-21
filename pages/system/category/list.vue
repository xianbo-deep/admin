<template>
  <view class="fix-top-window">
    <view class="uni-header">
      <uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
      <view class="uni-group">
        <input class="uni-search" type="text" v-model="query" @confirm="search"
          placeholder="搜索分类..." />
        <button class="uni-button hide-on-phone" type="default" size="mini"
          @click="search">搜索</button>
        <button class="uni-button" type="primary" size="mini"
          @click="navigateTo('./add')">新增</button>
        <button class="uni-button" type="warn" size="mini" :disabled="!selectedIndexs.length"
          @click="delTable">批量删除</button>
      </view>
    </view>
    <view class="uni-container">
      <unicloud-db ref="udb" collection="Category" :where="where"
        page-data="replace" :getcount="true" :page-size="options.pageSize"
        :page-current="options.pageCurrent" :orderby="orderby" loadtime="manual"
        @load="onqueryload" v-slot:default="{ pagination, loading, error }">
        <uni-table ref="table" :loading="loading" :emptyText="error.message || '没有数据'" border stripe
          type="selection" @selection-change="selectionChange">
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'categoryId')">类别ID</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'name')">分类名称</uni-th>
            <uni-th align="center">描述</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'createdAt')">创建时间</uni-th>
            <uni-th align="center">操作</uni-th>
          </uni-tr>
          <uni-tr v-for="(item,index) in displayData" :key="index">
            <uni-td align="center">{{item.categoryId}}</uni-td>
            <uni-td align="center">{{item.name}}</uni-td>
            <uni-td align="center">{{item.description || '-'}}</uni-td>
            <uni-td align="center">{{formatDate(item.createdAt)}}</uni-td>
            <uni-td align="center">
              <view class="uni-group">
                <button @click="navigateTo('./edit?id=' + item._id, false)" class="uni-button" size="mini"
                  type="primary">编辑</button>
                <button @click="confirmDelete(item._id)" class="uni-button" size="mini"
                  type="warn">删除</button>
              </view>
            </uni-td>
          </uni-tr>
        </uni-table>
        <view class="uni-pagination-box">
          <uni-pagination show-icon show-page-size :page-size="pagination.size" v-model="pagination.current"
            :total="pagination.count" @change="onPageChanged" @pageSizeChange="changeSize" />
        </view>
      </unicloud-db>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database()
const dbCmd = db.command

// 表查询配置
const dbOrderBy = 'createdAt desc' // 默认按创建时间降序
const dbSearchFields = ['categoryId', 'name', 'description'] // 支持模糊搜索的字段列表
// 分页配置
const pageSize = 20
const pageCurrent = 1

const orderByMapping = {
  "ascending": "asc",
  "descending": "desc"
}

export default {
  data() {
    return {
      query: '',
      where: '',
      orderby: dbOrderBy,
      orderByFieldName: "",
      selectedIndexs: [],
      options: {
        pageSize,
        pageCurrent,
        getone: false,
        manual: false
      },
      data: []
    }
  },

  computed: {
    displayData() {
      return this.data || []
    }
  },

  mounted() {
    this.loadData()
  },

  methods: {
    formatDate(date) {
      if (!date) return '-'
      const d = new Date(date)
      return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')} ${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}`
    },

    getWhere() {
      const query = this.query.trim()
      if (!query) {
        return ''
      }
      const queryRe = new RegExp(query, 'i')
      return dbCmd.or(
        dbSearchFields.map(name => ({
          [name]: queryRe
        }))
      )
    },

    search() {
      const newWhere = this.getWhere()
      this.where = newWhere
      this.$nextTick(() => {
        this.loadData()
      })
    },

    changeSize(pageSize) {
      this.options.pageSize = pageSize
      this.options.pageCurrent = 1
      this.$nextTick(() => {
        this.loadData()
      })
    },

    loadData(clear = true) {
      if (this.$refs.udb) {
        this.$refs.udb.loadData({
          clear
        })
      }
    },

    onPageChanged(e) {
      this.selectedIndexs = []
      this.$refs.table.clearSelection()
      this.$refs.udb.loadData({
        current: e.current
      })
    },

    sortChange(e, name) {
      this.orderByFieldName = name
      if (e.order) {
        this.orderby = name + ' ' + orderByMapping[e.order]
      } else {
        this.orderby = dbOrderBy
      }
      this.$refs.table.clearSelection()
      this.$nextTick(() => {
        this.$refs.udb.loadData()
      })
    },

    async onqueryload(data) {
      const categoryData = Array.isArray(data) ? data : data.data
      
      if (!categoryData || categoryData.length === 0) {
        this.$set(this, 'data', [])
        return
      }
      
      this.$set(this, 'data', categoryData)
    },

    navigateTo(url, clear = true) {
      if (!url) return
      uni.navigateTo({
        url,
        events: {
          refreshData: () => {
            this.loadData(clear)
          }
        }
      })
    },

    selectedItems() {
      if (!this.data || !this.selectedIndexs) {
        return []
      }
      return this.selectedIndexs.map(i => this.data[i]._id).filter(Boolean)
    },

    async delTable() {
      const items = this.selectedItems()
      if (!items.length) return
      
      try {
        uni.showModal({
          title: '提示',
          content: '是否确认删除选中的分类？删除后关联的视频将无法显示该分类',
          success: async (res) => {
            if (res.confirm) {
              const transaction = await db.startTransaction()
              try {
                // 首先检查是否有视频使用这些分类
                const videos = await db.collection('Video')
                  .where({
                    categoryId: dbCmd.in(items)
                  })
                  .limit(1)
                  .get();

                if (videos.result.data.length > 0) {
                  uni.showModal({
                    content: '选中的分类中存在已被视频使用的分类，无法删除',
                    showCancel: false
                  });
                  return;
                }

                const promises = items.map(id => 
                  transaction.collection('Category').doc(id).remove()
                )
                await Promise.all(promises)
                await transaction.commit()
                
                this.selectedIndexs = []
                this.$refs.table?.clearSelection()
                this.loadData(true)
                
                uni.showToast({
                  title: '删除成功',
                  icon: 'success'
                })
              } catch (error) {
                await transaction.rollback()
                throw error
              }
            }
          }
        })
      } catch (error) {
        console.error('删除失败:', error)
        uni.showToast({
          title: '删除失败',
          icon: 'error' 
        })
      }
    },
    
    selectionChange(e) {
      if (!e || !e.detail || !Array.isArray(e.detail.index)) {
        return
      }
      this.selectedIndexs = e.detail.index
    },

    async confirmDelete(id) {
      if (!id) return
      
      try {
        uni.showModal({
          title: '提示',
          content: '是否确认删除该分类？删除后关联的视频将无法显示该分类',
          success: async (res) => {
            if (res.confirm) {
              // 检查是否有视频使用此分类
              const videos = await db.collection('Video')
                .where({
                  categoryId: id
                })
                .limit(1)
                .get();

              if (videos.result.data.length > 0) {
                uni.showModal({
                  content: '该分类已被视频使用，无法删除',
                  showCancel: false
                });
                return;
              }

              await db.collection('Category').doc(id).remove()
              this.loadData(true)
              
              uni.showToast({
                title: '删除成功',
                icon: 'success'
              })
            }
          }
        })
      } catch (error) {
        console.error('删除失败:', error)
        uni.showToast({
          title: '删除失败', 
          icon: 'error'
        })
      }
    }
  }
}
</script>

<style>
.fix-top-window {
  padding-top: 0;
}

.uni-header {
  padding: 20rpx;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #e5e5e5;
}

.uni-group {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 10rpx;
}

.uni-search {
  width: 300rpx;
  padding: 10rpx;
  border: 1px solid #e5e5e5;
  border-radius: 4rpx;
}

.uni-button {
  margin-left: 10rpx;
}

.uni-container {
  padding: 20rpx;
}

.uni-pagination-box {
  margin-top: 20rpx;
}

@media screen and (max-width: 768px) {
  .hide-on-phone {
    display: none;
  }
  
  .uni-stat-breadcrumb-on-phone {
    display: none;
  }
}
</style>