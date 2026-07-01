<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Search, Tickets, Plus } from '@element-plus/icons-vue'
import { getOrderListAPI, orderUpdateCloseAPI, orderDeleteByIdsAPI } from '@/apis/order'
import LogisticsDialog from '@/views/oms/order/components/logisticsDialog.vue'
import { formatDateTime } from '@/utils/datetime'
import type { OmsOrder, OrderQueryParam } from '@/types/order'
import { useOrderStore } from '@/stores/order'
import { createOrderAPI } from '@/apis/order'

const router = useRouter()
const orderStore = useOrderStore()

const listQuery = ref<OrderQueryParam>({
  pageNum: 1,
  pageSize: 10
})
const list = ref<OmsOrder[]>([])
const listLoading = ref(true)
const total = ref(0)

const dialogVisible = ref(false)
const submitLoading = ref(false)
const orderForm = ref({
  memberId: 1,
  productId: 1,
  quantity: 1,
  receiverProvince: '北京市',
  receiverCity: '北京市',
  receiverRegion: '朝阳区',
  receiverDetailAddress: '测试地址123号',
  receiverName: '测试用户',
  receiverPhone: '13800138000'
})

const getList = async () => {
  listLoading.value = true
  try {
    const response = await getOrderListAPI(listQuery.value)
    listLoading.value = false
    list.value = response.data.list
    total.value = response.data.total
  } catch (error) {
    listLoading.value = false
    console.error('获取订单列表失败:', error)
  }
}

onMounted(() => {
  getList()
})

const handleAddOrder = () => {
  dialogVisible.value = true
}

const submitOrder = async () => {
  submitLoading.value = true
  try {
    const response = await createOrderAPI(orderForm.value)
    submitLoading.value = false
    if (response.code === 200) {
      ElMessage.success('订单创建成功！')
      dialogVisible.value = false
      getList()
    } else {
      ElMessage.error('创建失败：' + response.message)
    }
  } catch (error) {
    submitLoading.value = false
    ElMessage.error('创建失败，请检查网络')
    console.error('创建订单失败:', error)
  }
}

const multipleSelection = ref<OmsOrder[]>([])
const operateType = ref<number>()

const closeOrderData = ref({
  dialogVisible: false,
  content: '',
  orderIds: [] as number[]
})

const logisticsDialogVisible = ref(false)

const statusOptions = [
  { label: '待付款', value: 0 },
  { label: '待发货', value: 1 },
  { label: '已发货', value: 2 },
  { label: '已完成', value: 3 },
  { label: '已关闭', value: 4 }
]

const orderTypeOptions = [
  { label: '正常订单', value: 0 },
  { label: '秒杀订单', value: 1 }
]

const sourceTypeOptions = [
  { label: 'PC订单', value: 0 },
  { label: 'APP订单', value: 1 }
]

const operateOptions = [
  { label: "批量发货", value: 1 },
  { label: "关闭订单", value: 2 },
  { label: "删除订单", value: 3 }
]

const formatPayType = (value: number) => {
  if (value === 1) return '支付宝'
  else if (value === 2) return '微信'
  else return '未支付'
}

const formatSourceType = (value: number) => {
  if (value === 1) return 'APP订单'
  else return 'PC订单'
}

const formatStatus = (value: number) => {
  if (value === 1) return '待发货'
  else if (value === 2) return '已发货'
  else if (value === 3) return '已完成'
  else if (value === 4) return '已关闭'
  else if (value === 5) return '无效订单'
  else return '待付款'
}

const handleResetSearch = () => {
  listQuery.value = { pageNum: 1, pageSize: 10 }
}

const handleSearchList = () => {
  listQuery.value.pageNum = 1
  getList()
}

const handleSelectionChange = (val: OmsOrder[]) => {
  multipleSelection.value = val
}

const handleViewOrder = (index: number, row: OmsOrder) => {
  router.push({ path: '/oms/orderDetail', query: { id: row.id } })
}

const handleCloseOrder = (index: number, row: OmsOrder) => {
  closeOrderData.value.dialogVisible = true
  closeOrderData.value.orderIds = [row.id!]
}

const handleDeliveryOrder = (index: number, row: OmsOrder) => {
  orderStore.setDeliverOrderList([row])
  router.push({ path: '/oms/deliverOrderList' })
}

const handleViewLogistics = (index: number, row: OmsOrder) => {
  logisticsDialogVisible.value = true
  console.log(index, row)
}

const handleDeleteOrder = async (index: number, row: OmsOrder) => {
  const ids = [row.id!]
  await deleteOrderFn(ids)
}

const handleBatchOperate = async () => {
  if (!multipleSelection.value || multipleSelection.value.length < 1) {
    ElMessage.warning('请选择要操作的订单')
    return
  }
  if (operateType.value === 1) {
    const listItems = multipleSelection.value.filter(item => item.status === 1)
    if (!listItems || listItems.length < 1) {
      ElMessage.warning('选中订单中没有可以发货的订单')
      return
    }
    orderStore.setDeliverOrderList(listItems)
    router.push({ path: '/oms/deliverOrderList' })
  } else if (operateType.value === 2) {
    closeOrderData.value.orderIds = multipleSelection.value.filter(item => item.status === 0)
      .map(item => item.id)
    closeOrderData.value.dialogVisible = true
  } else if (operateType.value === 3) {
    const ids = multipleSelection.value.filter(item => item.status === 4)
      .map(item => item.id)
    await deleteOrderFn(ids)
  }
}

const handleSizeChange = (val: number) => {
  listQuery.value.pageNum = 1
  listQuery.value.pageSize = val
  getList()
}

const handleCurrentChange = (val: number) => {
  listQuery.value.pageNum = val
  getList()
}

const handleCloseOrderConfirm = async () => {
  if (!closeOrderData.value.content) {
    ElMessage.warning('操作备注不能为空')
    return
  }
  const orderIds = closeOrderData.value.orderIds.join(',')
  await orderUpdateCloseAPI({ ids: orderIds, note: closeOrderData.value.content })
  closeOrderData.value.orderIds = []
  closeOrderData.value.dialogVisible = false
  getList()
  ElMessage.success('修改成功')
}

const deleteOrderFn = async (ids: number[]) => {
  await ElMessageBox.confirm('是否要进行该删除操作?', '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning'
  })
  await orderDeleteByIdsAPI({ ids: ids.join(',') })
  ElMessage.success('删除成功！')
  getList()
}
</script>

<template>
  <div class="app-container">
    <el-card class="filter-container" shadow="never">
      <div>
        <el-icon class="el-icon-middle">
          <Search />
        </el-icon>
        <span>筛选搜索</span>
        <el-button style="float:right" type="primary" @click="handleSearchList()">
          查询搜索
        </el-button>
        <el-button style="float:right;margin-right: 15px" @click="handleResetSearch()">
          重置
        </el-button>
      </div>
      <div style="margin-top: 20px">
        <el-form :inline="true" :model="listQuery" label-width="140px">
          <el-form-item label="输入搜索：">
            <el-input v-model="listQuery.orderSn" class="input-width" placeholder="订单编号"></el-input>
          </el-form-item>
          <el-form-item label="收货人：">
            <el-input v-model="listQuery.receiverKeyword" class="input-width" placeholder="收货人姓名/手机号码"></el-input>
          </el-form-item>
          <el-form-item label="提交时间：">
            <el-date-picker class="input-width" v-model="listQuery.createTime" value-format="yyyy-MM-dd" type="date"
              placeholder="请选择时间">
            </el-date-picker>
          </el-form-item>
          <el-form-item label="订单状态：">
            <el-select v-model="listQuery.status" class="input-width" placeholder="全部" clearable>
              <el-option v-for="item in statusOptions" :key="item.value" :label="item.label" :value="item.value">
              </el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="订单分类：">
            <el-select v-model="listQuery.orderType" class="input-width" placeholder="全部" clearable>
              <el-option v-for="item in orderTypeOptions" :key="item.value" :label="item.label" :value="item.value">
              </el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="订单来源：">
            <el-select v-model="listQuery.sourceType" class="input-width" placeholder="全部" clearable>
              <el-option v-for="item in sourceTypeOptions" :key="item.value" :label="item.label" :value="item.value">
              </el-option>
            </el-select>
          </el-form-item>
        </el-form>
      </div>
    </el-card>
    <el-card class="operate-container" shadow="never">
      <el-icon class="el-icon-middle">
        <Tickets />
      </el-icon>
      <span>数据列表</span>
      <el-button style="float:right;margin-right: 15px" type="primary" size="small" @click="handleAddOrder">
        <el-icon><Plus /></el-icon>创建订单
      </el-button>
    </el-card>
    <div class="table-container">
      <el-table ref="orderTable" :data="list" style="width: 100%;" @selection-change="handleSelectionChange"
        v-loading="listLoading" border>
        <el-table-column type="selection" width="60" align="center"></el-table-column>
        <el-table-column label="编号" width="80" align="center">
          <template #default="scope">{{ scope.row.id }}</template>
        </el-table-column>
        <el-table-column label="订单编号" width="180" align="center">
          <template #default="scope">{{ scope.row.orderSn }}</template>
        </el-table-column>
        <el-table-column label="提交时间" width="180" align="center">
          <template #default="scope">{{ formatDateTime(scope.row.createTime) }}</template>
        </el-table-column>
        <el-table-column label="用户账号" align="center">
          <template #default="scope">{{ scope.row.memberUsername }}</template>
        </el-table-column>
        <el-table-column label="订单金额" width="120" align="center">
          <template #default="scope">￥{{ scope.row.totalAmount }}</template>
        </el-table-column>
        <el-table-column label="支付方式" width="120" align="center">
          <template #default="scope">{{ formatPayType(scope.row.payType) }}</template>
        </el-table-column>
        <el-table-column label="订单来源" width="120" align="center">
          <template #default="scope">{{ formatSourceType(scope.row.sourceType) }}</template>
        </el-table-column>
        <el-table-column label="订单状态" width="120" align="center">
          <template #default="scope">{{ formatStatus(scope.row.status) }}</template>
        </el-table-column>
        <el-table-column label="操作" width="200" align="center">
          <template #default="scope">
            <el-button size="small" @click="handleViewOrder(scope.$index, scope.row)">查看订单</el-button>
            <el-button size="small" @click="handleCloseOrder(scope.$index, scope.row)"
              v-show="scope.row.status === 0">关闭订单</el-button>
            <el-button size="small" @click="handleDeliveryOrder(scope.$index, scope.row)"
              v-show="scope.row.status === 1">订单发货</el-button>
            <el-button size="small" @click="handleViewLogistics(scope.$index, scope.row)"
              v-show="scope.row.status === 2 || scope.row.status === 3">订单跟踪</el-button>
            <el-button size="small" type="danger" @click="handleDeleteOrder(scope.$index, scope.row)"
              v-show="scope.row.status === 4">删除订单</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="batch-operate-container">
      <el-select v-model="operateType" placeholder="批量操作">
        <el-option v-for="item in operateOptions" :key="item.value" :label="item.label" :value="item.value">
        </el-option>
      </el-select>
      <el-button style="margin-left: 20px" class="search-button" @click="handleBatchOperate()" type="primary">
        确定
      </el-button>
    </div>
    <div class="pagination-container">
      <el-pagination background @size-change="handleSizeChange" @current-change="handleCurrentChange"
        layout="total, sizes,prev, pager, next,jumper" v-model:current-page="listQuery.pageNum"
        :page-size="listQuery.pageSize" :page-sizes="[5, 10, 15]" :total="total">
      </el-pagination>
    </div>
    <el-dialog title="关闭订单" v-model="closeOrderData.dialogVisible" width="30%">
      <span style="vertical-align: top">操作备注：</span>
      <el-input style="width: 80%" type="textarea" :rows="5" placeholder="请输入内容" v-model="closeOrderData.content">
      </el-input>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="closeOrderData.dialogVisible = false">取 消</el-button>
          <el-button type="primary" @click="handleCloseOrderConfirm">确 定</el-button>
        </span>
      </template>
    </el-dialog>

    <el-dialog title="创建订单" v-model="dialogVisible" width="500px" @close="dialogVisible = false">
      <el-form :model="orderForm" label-width="100px" size="small">
        <el-form-item label="用户ID">
          <el-input v-model="orderForm.memberId" placeholder="请输入用户ID"></el-input>
        </el-form-item>
        <el-form-item label="商品ID">
          <el-input v-model="orderForm.productId" placeholder="请输入商品ID"></el-input>
        </el-form-item>
        <el-form-item label="购买数量">
          <el-input v-model="orderForm.quantity" placeholder="请输入数量"></el-input>
        </el-form-item>
        <el-form-item label="收货省份">
          <el-input v-model="orderForm.receiverProvince" placeholder="如：北京市"></el-input>
        </el-form-item>
        <el-form-item label="收货城市">
          <el-input v-model="orderForm.receiverCity" placeholder="如：北京市"></el-input>
        </el-form-item>
        <el-form-item label="收货区/县">
          <el-input v-model="orderForm.receiverRegion" placeholder="如：朝阳区"></el-input>
        </el-form-item>
        <el-form-item label="详细地址">
          <el-input v-model="orderForm.receiverDetailAddress" placeholder="请输入详细地址"></el-input>
        </el-form-item>
        <el-form-item label="收货人姓名">
          <el-input v-model="orderForm.receiverName" placeholder="请输入姓名"></el-input>
        </el-form-item>
        <el-form-item label="收货人电话">
          <el-input v-model="orderForm.receiverPhone" placeholder="请输入手机号"></el-input>
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">取 消</el-button>
          <el-button type="primary" @click="submitOrder" :loading="submitLoading">确 定</el-button>
        </span>
      </template>
    </el-dialog>

    <logistics-dialog v-model="logisticsDialogVisible"></logistics-dialog>
  </div>
</template>

<style scoped>
.input-width {
  width: 203px
}
</style>