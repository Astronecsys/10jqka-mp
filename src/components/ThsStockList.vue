<!-- src\components\ThsStockList.vue -->
<template>
  <!-- 快捷排序 -->
  <view v-if="quickSort">
    <view class="title">股票排行</view>
    <view class="buttons">
      <view class="buttons">
        <view
          class="button"
          v-for="(sort, index) in sortOptions"
          :key="index"
          @click="sortBy(sort.key, sort.order)"
        >
          {{ sort.text }}
        </view>
      </view>
    </view>
  </view>
  <!-- 股票列表 -->
  <view class="stock_table">
    <!-- 表头 -->
    <view class="stock_head" role="row">
      <view class="stock_cell" role="columnheader">股票名称</view>
      <view class="stock_cell right" role="columnheader" @click="toggleSortOrder('currentPrice')">
        <span class="header-content">
          最新价
          <span class="sort-icon">
            <span v-if="sortKey === 'currentPrice' && sortOrder === 'desc'">🔽</span>
            <span v-if="sortKey === 'currentPrice' && sortOrder === 'asc'">🔼</span>
            <span v-if="sortKey !== 'currentPrice'">⬍</span>
          </span>
        </span>
      </view>
      <view class="stock_cell right" role="columnheader" @click="toggleSortOrder('changePercent')">
        <span class="header-content">
          涨跌幅
          <span class="sort-icon">
            <span v-if="sortKey === 'changePercent' && sortOrder === 'desc'">🔽</span>
            <span v-if="sortKey === 'changePercent' && sortOrder === 'asc'">🔼</span>
            <span v-if="sortKey !== 'changePercent'">⬍</span>
          </span>
        </span>
      </view>
      <view class="stock_cell right" role="columnheader" @click="toggleSortOrder('speedPercent')">
        <span class="header-content">
          涨速
          <span class="sort-icon">
            <span v-if="sortKey === 'speedPercent' && sortOrder === 'desc'">🔽</span>
            <span v-if="sortKey === 'speedPercent' && sortOrder === 'asc'">🔼</span>
            <span v-if="sortKey !== 'speedPercent'">⬍</span>
          </span>
        </span>
      </view>
    </view>
    <!-- 内容 -->
    <navigator
      class="stock_item"
      v-for="(item, index) in sortedStocks"
      :key="index"
      :url="`/pages/stock-detail/stock-detail?code=${item.code}`"
      open-type="navigate"
      hover-class="navigator-hover"
      role="row"
    >
      <view class="stock_cell" role="cell">
        <view class="stock_name">{{ item.name }}</view>
        <view class="stock_code">{{ item.code }}</view>
      </view>
      <view
        class="stock_cell right"
        :class="{
          positive: item.change > 0,
          negative: item.change < 0,
        }"
        role="cell"
      >
        <text class="stock_number">{{ item.currentPrice }}</text>
      </view>
      <view
        class="stock_cell right"
        :class="{
          positive: item.changePercent > 0,
          negative: item.changePercent < 0,
        }"
        role="cell"
      >
        <text class="stock_number">{{ item.changePercent }}%</text>
      </view>
      <view
        class="stock_cell right"
        :class="{
          positive: item.speedPercent > 0,
          negative: item.speedPercent < 0,
        }"
        role="cell"
      >
        <text class="stock_number">{{ item.speedPercent }}%</text>
      </view>
    </navigator>
  </view>
  <view v-if="!maxRows" class="loading-text">{{ finish ? '没有更多数据...' : '正在加载...' }}</view>
</template>

<script lang="ts" setup>
import { fetchStockData } from '@/service/stockService'
import { StockData } from '@/types/stockService'
import { defineProps, computed, ref } from 'vue'

const props = defineProps<{
  stockCodes: string[]
  maxRows?: number
  quickSort?: boolean
}>()
const loaded = ref(0)
const stocks = ref<StockData[]>([])
const sortOrder = ref<'none' | 'asc' | 'desc'>('none')
const sortKey = ref<string>('')
const sortOptions = ref([
  { key: 'currentPrice', order: 'asc' as const, text: '低价榜' },
  { key: 'currentPrice', order: 'desc' as const, text: '高价榜' },
  { key: 'changePercent', order: 'asc' as const, text: '跌幅榜' },
  { key: 'changePercent', order: 'desc' as const, text: '涨幅榜' },
  { key: 'speedPercent', order: 'asc' as const, text: '跌速榜' },
  { key: 'speedPercent', order: 'desc' as const, text: '涨速榜' },
])

// 计算属性，限制显示的行数
const limitedStocks = computed(() => {
  if (props.maxRows !== undefined) {
    return stocks.value.slice(0, props.maxRows)
  }
  return stocks.value
})

// 排序逻辑
const sortedStocks = computed(() => {
  if (sortOrder.value === 'asc') {
    return [...limitedStocks.value].sort((a, b) => a[sortKey.value] - b[sortKey.value])
  } else if (sortOrder.value === 'desc') {
    return [...limitedStocks.value].sort((a, b) => b[sortKey.value] - a[sortKey.value])
  }
  return limitedStocks.value
})

// 切换排序状态
function toggleSortOrder(key: string) {
  if (sortKey.value === key) {
    if (sortOrder.value === 'asc') {
      sortOrder.value = 'desc'
    } else if (sortOrder.value === 'desc') {
      sortOrder.value = 'none'
      sortKey.value = ''
    } else {
      sortOrder.value = 'asc'
    }
  } else {
    sortKey.value = key
    sortOrder.value = 'asc'
  }
}

// 通用排序方法
function sortBy(key: string, order: 'asc' | 'desc') {
  sortKey.value = key
  sortOrder.value = order
}

// 加载完毕标记
const finish = ref(false)
// 获取股票数据
const getStockData = async () => {
  // 退出判断
  if (finish.value === true) {
    return uni.showToast({ icon: 'none', title: '没有更多数据...' })
  }
  const right = Math.min(props.stockCodes.length, loaded.value + 10)
  console.log('获取数据' + loaded.value + '--' + right)
  const res = await fetchStockData(props.stockCodes.slice(loaded.value, right))
  loaded.value = right
  stocks.value.push(...res)
  if (props.stockCodes.length === loaded.value) {
    finish.value = true
  }
}
// 重置数据
const resetData = () => {
  stocks.value = []
  finish.value = false
  loaded.value = 0
}
// 组价挂载完毕
onMounted(() => {
  getStockData()
})
// 监听 stockCodes 变化
watch(
  () => props.stockCodes,
  () => {
    resetData()
    getStockData()
  },
  { immediate: true },
)
// 暴露方法
defineExpose({
  resetData,
  getStockData,
})
</script>

<style lang="scss" scoped>
.buttons {
  margin-bottom: 10px;
}

button {
  padding: 5px 10px;
  margin-right: 10px;
  font-size: 14px;
  cursor: pointer;
}

.stock_table {
  width: 100%;
  margin: 20px 0;
}

.stock_head {
  position: sticky;
  top: 0;
  display: flex;
  justify-content: space-between;
  padding: 10px;
  font-weight: bold;
  cursor: pointer;
  background-color: #f5f5f5;
  border-bottom: 1px solid #e0e0e0;
}

.stock_item {
  display: flex;
  justify-content: space-between;
  padding: 10px;
  text-decoration: none;
  background-color: #fff;
  border-bottom: 1px solid #e0e0e0;
}

.stock_item:nth-child(even) {
  background-color: #f9f9f9;
}

.stock_item:hover {
  background-color: #e0e0e0;
}

.navigator-hover {
  background-color: #d0d0d0;
}

.stock_cell {
  display: flex;
  flex: 1;
  flex-direction: column;
  justify-content: center;
}

.stock_name {
  font-weight: bold;
  text-align: left;
}

.stock_code {
  color: #999;
  text-align: left;
}

.right {
  text-align: right;
}

.stock_number {
  margin-right: 15px;
}
.positive .stock_number {
  color: red;
}

.negative .stock_number {
  color: green;
}
.header-content {
  display: flex;
  align-items: center;
  justify-content: center;
}

.buttons {
  display: flex;
}

.button {
  padding: 5px 10px;
  margin-right: 10px;
  cursor: pointer;
}
.title {
  font-size: large;
  font-weight: bold;
}
// 加载提示文字
.loading-text {
  padding: 20rpx 0;
  font-size: 28rpx;
  color: #666;
  text-align: center;
}
</style>
