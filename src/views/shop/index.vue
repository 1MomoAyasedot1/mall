<template>
  <div class="shop-page">
    <div class="navbar">
      <div class="logo">🛒 优选<span>商城</span></div>
      <div class="user-info">
        <span>{{ userDisplay }}</span>
        <button v-if="!currentUser" class="btn-login" @click="showLogin = true">登录</button>
        <button v-else class="btn-logout" @click="logout">退出</button>
      </div>
    </div>

    <div class="main">
      <div class="section-title">🔥 热销商品</div>
      <div class="product-grid">
        <div v-for="p in products" :key="p.id" class="product-card">
          <div class="image">{{ p.pic || '📦' }}</div>
          <div class="info">
            <div class="name">{{ p.name }}</div>
            <div class="desc">库存: {{ p.stock || 99 }}件</div>
            <div class="price-row">
              <span class="price"><small>¥</small>{{ p.price.toFixed(2) }}</span>
              <button class="btn-buy" @click="addToCart(p)" :disabled="!currentUser">
                {{ !currentUser ? '🔒 登录购买' : '➕ 加入购物车' }}
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 购物车按钮 -->
    <button class="cart-toggle" @click="showCart = true">
      🛒
      <span v-if="cartCount > 0" class="badge">{{ cartCount }}</span>
    </button>

    <!-- 购物车弹窗 -->
    <div v-if="showCart" class="modal-overlay" @click.self="showCart = false">
      <div class="modal-panel cart-panel">
        <div class="modal-title">
          🛒 购物车
          <span class="close" @click="showCart = false">✕</span>
        </div>
        <div v-if="cart.length === 0" class="empty">🛍️ 购物车是空的</div>
        <template v-else>
          <div v-for="(item, index) in cart" :key="index" class="cart-item">
            <span class="name">{{ item.pic || '📦' }} {{ item.name }}</span>
            <div class="qty">
              <button @click="updateQty(index, -1)">−</button>
              <span>{{ item.quantity }}</span>
              <button @click="updateQty(index, 1)">+</button>
            </div>
            <span class="subtotal">¥{{ (item.price * item.quantity).toFixed(2) }}</span>
            <button class="remove" @click="cart.splice(index, 1)">✕</button>
          </div>
          <div class="total-row">
            <span>合计</span>
            <span class="amount">¥{{ cartTotal.toFixed(2) }}</span>
          </div>
          <button class="btn-checkout" @click="checkout" :disabled="loading">
            {{ loading ? '下单中...' : '立即下单' }}
          </button>
        </template>
      </div>
    </div>

    <!-- 登录弹窗 -->
    <div v-if="showLogin" class="modal-overlay" @click.self="showLogin = false">
      <div class="modal-panel login-panel">
        <span class="close" @click="showLogin = false">✕</span>
        <h2>🔐 用户登录</h2>
        <div class="form-group">
          <label>用户名</label>
          <input v-model="loginForm.username" placeholder="请输入用户名" />
        </div>
        <div class="form-group">
          <label>密码</label>
          <input v-model="loginForm.password" type="password" placeholder="请输入密码" @keyup.enter="handleLogin" />
        </div>
        <button class="btn-submit" @click="handleLogin" :disabled="loginLoading">
          {{ loginLoading ? '登录中...' : '登 录' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { createOrderAPI } from '@/apis/order'

// ===== 状态 =====
const currentUser = ref<string | null>(null)
const currentToken = ref<string | null>(null)
const products = ref<any[]>([])
const cart = ref<any[]>([])
const showCart = ref(false)
const showLogin = ref(false)
const loginLoading = ref(false)
const loading = ref(false)

const loginForm = ref({
  username: 'admin',
  password: '123456'
})

// ===== 计算属性 =====
const userDisplay = computed(() => currentUser.value ? '👤 ' + currentUser.value : '👤 游客')
const cartCount = computed(() => cart.value.reduce((s, i) => s + i.quantity, 0))
const cartTotal = computed(() => cart.value.reduce((s, i) => s + i.price * i.quantity, 0))

// ===== 加载商品 =====
const loadProducts = async () => {
  try {
    const res = await fetch('http://localhost:8080/product/list?pageNum=1&pageSize=20')
    const data = await res.json()
    if (data.code === 200 && data.data?.list) {
      products.value = data.data.list
      return
    }
  } catch (e) { /* ignore */ }
  // 模拟数据
  products.value = [
    { id: 1, name: '华为 Mate 60 Pro', price: 6999, stock: 100, pic: '📱' },
    { id: 2, name: 'iPhone 15 Pro Max', price: 9999, stock: 50, pic: '📱' },
    { id: 3, name: '小米 14 Ultra', price: 5999, stock: 80, pic: '📱' },
    { id: 4, name: '联想 ThinkPad X1', price: 8999, stock: 30, pic: '💻' },
    { id: 5, name: '戴尔 XPS 16', price: 12999, stock: 20, pic: '💻' },
    { id: 6, name: '索尼 WH-1000XM5', price: 2999, stock: 60, pic: '🎧' },
    { id: 7, name: 'Apple Watch Series 9', price: 3199, stock: 45, pic: '⌚' },
    { id: 8, name: '大疆 DJI Mini 4 Pro', price: 4788, stock: 15, pic: '📷' },
  ]
}

// ===== 登录 =====
const handleLogin = async () => {
  loginLoading.value = true
  try {
    const res = await fetch('http://localhost:8080/admin/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(loginForm.value)
    })
    const data = await res.json()
    if (data.code === 200) {
      currentToken.value = data.data.token
      currentUser.value = loginForm.value.username
      localStorage.setItem('shopUser', currentUser.value)
      localStorage.setItem('shopToken', currentToken.value)
      showLogin.value = false
      ElMessage.success('登录成功！')
    } else {
      ElMessage.error(data.message || '登录失败')
    }
  } catch (e) {
    ElMessage.error('网络异常，请检查后端服务')
  } finally {
    loginLoading.value = false
  }
}

// ===== 登出 =====
const logout = () => {
  currentUser.value = null
  currentToken.value = null
  localStorage.removeItem('shopUser')
  localStorage.removeItem('shopToken')
  cart.value = []
  ElMessage.info('已退出登录')
}

// ===== 购物车 =====
const addToCart = (product: any) => {
  if (!currentUser.value) {
    ElMessage.warning('请先登录')
    showLogin.value = true
    return
  }
  const existing = cart.value.find(i => i.productId === product.id)
  if (existing) {
    existing.quantity++
  } else {
    cart.value.push({ ...product, productId: product.id, quantity: 1 })
  }
  ElMessage.success('已加入购物车')
}

const updateQty = (index: number, delta: number) => {
  const item = cart.value[index]
  item.quantity += delta
  if (item.quantity <= 0) cart.value.splice(index, 1)
}

// ===== 下单 =====
const checkout = async () => {
  if (cart.value.length === 0) {
    ElMessage.warning('购物车是空的')
    return
  }
  const item = cart.value[0]
  loading.value = true
  try {
    const res = await createOrderAPI({
      memberId: 1,
      productId: item.productId,
      quantity: item.quantity,
      receiverProvince: '广东省',
      receiverCity: '深圳市',
      receiverRegion: '南山区',
      receiverDetailAddress: '科技园南区123号',
      receiverName: currentUser.value || '用户',
      receiverPhone: '13900139000'
    })
    if (res.code === 200) {
      ElMessage.success('🎉 下单成功！去后台查看订单')
      cart.value = []
      showCart.value = false
    } else {
      ElMessage.error(res.message || '下单失败')
    }
  } catch (e) {
    ElMessage.error('网络异常，请检查后端服务')
  } finally {
    loading.value = false
  }
}

// ===== 初始化 =====
onMounted(() => {
  const savedUser = localStorage.getItem('shopUser')
  const savedToken = localStorage.getItem('shopToken')
  if (savedUser && savedToken) {
    currentUser.value = savedUser
    currentToken.value = savedToken
  }
  loadProducts()
})
</script>

<style scoped>
.shop-page {
  min-height: 100vh;
  background: #f5f7fa;
}
.navbar {
  background: linear-gradient(135deg, #1a1a2e, #16213e);
  color: #fff;
  padding: 0 40px;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  position: sticky;
  top: 0;
  z-index: 100;
}
.navbar .logo {
  font-size: 22px;
  font-weight: 700;
}
.navbar .logo span {
  color: #ffd700;
}
.navbar .user-info {
  display: flex;
  align-items: center;
  gap: 16px;
  font-size: 14px;
}
.btn-login {
  background: #ffd700;
  color: #1a1a2e;
  border: none;
  padding: 8px 24px;
  border-radius: 20px;
  font-weight: 700;
  cursor: pointer;
}
.btn-logout {
  background: transparent;
  color: #ffd700;
  border: 1px solid #ffd700;
  padding: 6px 18px;
  border-radius: 20px;
  cursor: pointer;
}
.main {
  max-width: 1200px;
  margin: 0 auto;
  padding: 30px 20px;
}
.section-title {
  font-size: 24px;
  font-weight: 700;
  color: #1a1a2e;
  margin-bottom: 24px;
}
.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 24px;
}
.product-card {
  background: #fff;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
  transition: transform 0.25s;
  cursor: pointer;
}
.product-card:hover {
  transform: translateY(-6px);
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
}
.product-card .image {
  height: 200px;
  background: #f0f2f5;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 48px;
}
.product-card .info {
  padding: 16px 18px 20px;
}
.product-card .info .name {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 6px;
}
.product-card .info .desc {
  font-size: 13px;
  color: #999;
  margin-bottom: 10px;
}
.product-card .price-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.product-card .price {
  font-size: 22px;
  font-weight: 700;
  color: #e74c3c;
}
.product-card .price small {
  font-size: 14px;
}
.product-card .btn-buy {
  background: #409eff;
  color: #fff;
  border: none;
  padding: 6px 18px;
  border-radius: 20px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
}
.product-card .btn-buy:disabled {
  background: #a0cfff;
  cursor: not-allowed;
}

/* 购物车按钮 */
.cart-toggle {
  position: fixed;
  right: 30px;
  bottom: 100px;
  background: #1a1a2e;
  color: #fff;
  width: 56px;
  height: 56px;
  border-radius: 50%;
  border: none;
  font-size: 24px;
  cursor: pointer;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.25);
  z-index: 50;
}
.cart-toggle .badge {
  position: absolute;
  top: -6px;
  right: -6px;
  background: #e74c3c;
  color: #fff;
  font-size: 12px;
  font-weight: 700;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 弹窗 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 200;
  display: flex;
  justify-content: center;
  align-items: center;
}
.modal-panel {
  background: #fff;
  border-radius: 16px;
  padding: 30px;
  max-height: 80vh;
  overflow-y: auto;
  animation: slideUp 0.3s ease;
}
@keyframes slideUp {
  from {
    transform: translateY(40px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}
.cart-panel {
  width: 420px;
}
.login-panel {
  width: 380px;
}
.modal-title {
  font-size: 20px;
  font-weight: 700;
  margin-bottom: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.close {
  font-size: 28px;
  cursor: pointer;
  color: #999;
}
.close:hover {
  color: #333;
}
.cart-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 0;
  border-bottom: 1px solid #f0f2f5;
}
.cart-item .name {
  font-size: 14px;
  color: #333;
  flex: 1;
}
.cart-item .qty {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 0 16px;
}
.cart-item .qty button {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 1px solid #ddd;
  background: #fff;
  cursor: pointer;
  font-size: 16px;
}
.cart-item .subtotal {
  font-weight: 700;
  color: #e74c3c;
  font-size: 15px;
  min-width: 70px;
  text-align: right;
}
.cart-item .remove {
  color: #e74c3c;
  cursor: pointer;
  font-size: 18px;
  background: none;
  border: none;
}
.total-row {
  display: flex;
  justify-content: space-between;
  font-size: 18px;
  font-weight: 700;
  padding: 16px 0;
  border-top: 2px solid #f0f2f5;
  margin-top: 8px;
}
.total-row .amount {
  color: #e74c3c;
  font-size: 22px;
}
.btn-checkout {
  width: 100%;
  padding: 14px;
  background: linear-gradient(135deg, #ffd700, #f0c000);
  color: #1a1a2e;
  border: none;
  border-radius: 10px;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
}
.btn-checkout:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.empty {
  text-align: center;
  color: #999;
  padding: 40px 0;
}
.login-panel h2 {
  text-align: center;
  margin-bottom: 24px;
}
.login-panel .form-group {
  margin-bottom: 16px;
}
.login-panel .form-group label {
  display: block;
  font-weight: 600;
  font-size: 14px;
  margin-bottom: 4px;
  color: #555;
}
.login-panel .form-group input {
  width: 100%;
  padding: 10px 14px;
  border: 1.5px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
  outline: none;
}
.login-panel .form-group input:focus {
  border-color: #409eff;
}
.login-panel .btn-submit {
  width: 100%;
  padding: 12px;
  background: #409eff;
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 700;
  cursor: pointer;
}
.login-panel .btn-submit:disabled {
  background: #a0cfff;
  cursor: not-allowed;
}
@media (max-width: 640px) {
  .navbar {
    padding: 0 16px;
  }
  .navbar .logo {
    font-size: 17px;
  }
  .product-grid {
    grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
    gap: 12px;
  }
  .product-card .image {
    height: 140px;
    font-size: 32px;
  }
  .product-card .info .name {
    font-size: 14px;
  }
  .product-card .price {
    font-size: 18px;
  }
  .cart-panel {
    width: 94%;
    padding: 20px;
  }
  .login-panel {
    width: 92%;
    padding: 24px;
  }
  .cart-toggle {
    right: 16px;
    bottom: 80px;
    width: 48px;
    height: 48px;
    font-size: 20px;
  }
}
</style>