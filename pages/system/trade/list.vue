<template>
  <view class="fix-top-window">
    <view class="uni-header">
      <uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
      <view class="uni-group">
        <input 
          class="uni-search" 
          type="text" 
          v-model="query" 
          @confirm="search"
          placeholder="搜索卡号/用户ID..." 
        />
        <button 
          class="uni-button hide-on-phone" 
          type="default" 
          size="mini"
          @click="search"
        >搜索</button>
      </view>
    </view>

    <view class="uni-container">
      <unicloud-db 
        ref="udb" 
        collection="ExchangeRecord" 
        field="userId,cardId,cardCode,redeemTime,cardType,cardValue,cardCategory"
        :where="where"
        page-data="replace" 
        :getcount="true" 
        :page-size="options.pageSize"
        :page-current="options.pageCurrent" 
        :orderby="orderby"
        loadtime="manual"
        @load="onqueryload"
        v-slot:default="{ pagination, loading, error }"
      >
        <uni-table 
          ref="table" 
          :loading="loading" 
          :emptyText="error.message || '没有数据'" 
          border 
          stripe
        >
          <uni-tr>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'cardId')">卡号</uni-th>
            <uni-th align="center">用户昵称</uni-th>
            <uni-th align="center">卡密</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'cardType')">卡密类型</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'cardCategory')">卡类别</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'cardValue')">面值</uni-th>
            <uni-th align="center" sortable @sort-change="sortChange($event, 'redeemTime')">兑换时间</uni-th>
          </uni-tr>
          <uni-tr v-for="(item, index) in displayData" :key="item._id">
            <uni-td align="center">
              <view class="ellipsis" :title="item.cardId">{{item.cardId}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.nickname">{{item.nickname}}</view>
            </uni-td>
            <uni-td align="center">
              <view class="ellipsis" :title="item.cardCode">{{item.cardCode}}</view>
            </uni-td>
            <uni-td align="center">
              <uni-tag :text="formatCardType(item.cardType)" :type="getCardTypeTag(item.cardType)" />
            </uni-td>
            <uni-td align="center">
              <uni-tag :text="formatCardCategory(item.cardCategory)" :type="getCardCategoryTag(item.cardCategory)" />
            </uni-td>
            <uni-td align="center">{{formatCardValue(item.cardType, item.cardValue)}}</uni-td>
            <uni-td align="center">{{formatTime(item.redeemTime)}}</uni-td>
          </uni-tr>
        </uni-table>

        <view class="uni-pagination-box">
          <uni-pagination 
            show-icon 
            show-page-size 
            :page-size="pagination.size"
            v-model="pagination.current"
            :total="pagination.count" 
            @change="onPageChanged" 
            @pageSizeChange="changeSize" 
          />
        </view>
      </unicloud-db>
    </view>
  </view>
</template>

<script>
const db = uniCloud.database()
const dbCmd = db.command

// 表查询配置
const dbOrderBy = 'redeemTime desc' // 默认按兑换时间倒序
const dbSearchFields = ['cardId', 'userId', 'cardCode'] // 支持模糊搜索的字段
const pageSize = 20
const pageCurrent = 1

const orderByMapping = {
  "ascending": "asc",
  "descending": "desc"
}

const CARD_TYPE_MAP = {
  'daily': '日卡',
  'monthly': '月卡'
}

const CARD_TYPE_TAG = {
  'daily': 'primary',
  'monthly': 'success'
}

const CARD_CATEGORY_MAP = {
  'streamer': '主播卡',
  'review': '测评卡',
  'tutorial': '教程卡',
  'enterprise': '企业卡'
}

const CARD_CATEGORY_TAG = {
  'streamer': 'primary',
  'review': 'warning',
  'tutorial': 'success',
  'enterprise': 'info'
}

export default {
  data() {
    return {
      query: '',
      where: '',
      orderby: dbOrderBy,
      orderByFieldName: "",
      options: {
        pageSize,
        pageCurrent,
        getone: false,
        manual: false
      },
      data: [],
      userMap: new Map() // 用于存储用户ID和昵称的映射
    }
  },

  computed: {
    displayData() {
      return this.data.map(item => ({
        ...item,
        nickname: this.userMap.get(item.userId) || '未知用户'
      }))
    }
  },

  mounted() {
    this.loadData()
  },

  methods: {
    formatTime(timestamp) {
      if (!timestamp) return '-'
      const date = new Date(Number(timestamp))
      if (isNaN(date.getTime())) return '-'
      
      return date.toLocaleString('zh-CN', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit',
        hour12: false
      })
    },

    formatCardType(type) {
      return CARD_TYPE_MAP[type] || type
    },

    getCardTypeTag(type) {
      return CARD_TYPE_TAG[type] || 'info'
    },

    formatCardCategory(category) {
      return CARD_CATEGORY_MAP[category] || '未知'
    },

    getCardCategoryTag(category) {
      return CARD_CATEGORY_TAG[category] || 'info'
    },

    formatCardValue(type, value) {
      return `${value}天`
    },

    // 修改 methods 中的搜索相关方法
    
    async getWhere() {
      const query = this.query.trim()
      if (!query) {
        return ''
      }
    
      try {
        // 先查询符合昵称条件的用户
        const userResult = await db.collection('User')
          .where({
            nickname: new RegExp(query, 'i')
          })
          .field('userId')
          .get()
    
        const matchedUserIds = userResult.result.data.map(user => user.userId)
    
        // 构建查询条件
        return dbCmd.or([
          {
            cardId: new RegExp(query, 'i')
          },
          {
            userId: dbCmd.in(matchedUserIds)
          }
        ])
      } catch (error) {
        console.error('搜索用户失败:', error)
        // 如果用户查询失败，退回到只按卡号搜索
        return {
          cardId: new RegExp(query, 'i')
        }
      }
    },
    
    async search() {
      this.options.pageCurrent = 1 // 重置到第一页
      const newWhere = await this.getWhere()
      this.where = newWhere
      this.$nextTick(() => {
        this.loadData(true)
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
      this.$nextTick(() => {
        this.$refs.udb.loadData()
      })
    },

    async onqueryload(data) {
      const exchangeData = Array.isArray(data) ? data : data.data
      
      if (!exchangeData || exchangeData.length === 0) {
        this.$set(this, 'data', [])
        return
      }

      // 获取所有用户ID
      const userIds = [...new Set(exchangeData.map(item => item.userId))]
      
      try {
        // 获取用户信息
        if (userIds.length > 0) {
          const userResult = await db.collection('User')
            .where({
              userId: dbCmd.in(userIds)
            })
            .field('userId,nickname')
            .get()
          
          this.userMap = new Map(
            userResult.result.data.map(user => [user.userId, user.nickname])
          )
        }
        
        this.$set(this, 'data', exchangeData)
        
      } catch (error) {
        console.error('获取用户信息失败:', error)
        this.$set(this, 'data', exchangeData)
      }
    }
  }
}
</script>

<style>
.uni-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.uni-search {
  flex: 1;
  padding: 0 10px;
  height: 29px;
  line-height: 29px;
  font-size: 12px;
  border: 1px solid #DCDFE6;
  border-radius: 4px;
}

.ellipsis {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  max-width: 200px;
}

@media screen and (max-width: 768px) {
  .hide-on-phone {
    display: none;
  }
}
</style>