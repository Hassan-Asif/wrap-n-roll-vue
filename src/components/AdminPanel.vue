
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import {
  signInWithEmailAndPassword,
  onAuthStateChanged,
  signOut,
} from 'firebase/auth'
import {
  collection,
  query,
  orderBy,
  onSnapshot,
  doc,
  updateDoc,
} from 'firebase/firestore'
import { auth, db } from '../firebase'

const user = ref(null)
const email = ref('')
const password = ref('')
const loginError = ref('')
const loading = ref(false)
const orders = ref([])
const ordersError = ref('')
const updatingOrder = ref('')
const statusOptions = [
  'pending',
  'confirmed',
  'bought_from_vendor',
  'packed',
  'dispatched',
  'delivered',
  'cancelled',
]

let unsubscribeAuth
let unsubscribeOrders

function watchOrders() {
  if (unsubscribeOrders) unsubscribeOrders()

  loading.value = true
  ordersError.value = ''

  const ordersQuery = query(
    collection(db, 'orders'),
    orderBy('createdAt', 'desc')
  )

  unsubscribeOrders = onSnapshot(
    ordersQuery,
    (snapshot) => {
      orders.value = snapshot.docs.map((item) => ({
        id: item.id,
        ...item.data(),
      }))
      loading.value = false
    },
    (error) => {
      console.error('Could not load orders:', error)
      ordersError.value = 'Unable to load orders. Check your admin access and Firestore rules.'
      loading.value = false
    }
  )
}

onMounted(() => {
  unsubscribeAuth = onAuthStateChanged(auth, (currentUser) => {
    user.value = currentUser

    if (currentUser) {
      watchOrders()
    } else {
      if (unsubscribeOrders) unsubscribeOrders()
      orders.value = []
    }
  })
})

onUnmounted(() => {
  if (unsubscribeAuth) unsubscribeAuth()
  if (unsubscribeOrders) unsubscribeOrders()
})

async function login() {
  loginError.value = ''

  if (!email.value.trim() || !password.value) {
    loginError.value = 'Enter your email and password.'
    return
  }

  loading.value = true

  try {
    await signInWithEmailAndPassword(
      auth,
      email.value.trim(),
      password.value
    )
  } catch (error) {
    console.error('Admin login failed:', error)
    loginError.value = 'Login failed. Check your credentials and Firebase Authentication settings.'
  } finally {
    loading.value = false
  }
}

async function logout() {
  await signOut(auth)
}

async function changeStatus(order, newStatus) {
  if (!statusOptions.includes(newStatus)) return

  updatingOrder.value = order.id
  ordersError.value = ''

  try {
    await updateDoc(doc(db, 'orders', order.id), {
      status: newStatus,
    })
  } catch (error) {
    console.error('Status update failed:', error)
    ordersError.value = 'Could not update this order. Check your admin permissions.'
  } finally {
    updatingOrder.value = ''
  }
}

function formatDate(timestamp) {
  if (!timestamp?.toDate) return 'Date pending'
  return timestamp.toDate().toLocaleString()
}

function money(value) {
  return `PKR ${Number(value || 0).toLocaleString()}`
}

function statusLabel(status) {
  return String(status || 'pending')
    .replaceAll('_', ' ')
    .replace(/\b\w/g, (letter) => letter.toUpperCase())
}
</script>

<template>
  <main class="admin-page">
    <header class="admin-header">
      <div>
        <p class="eyebrow">WRAP N ROLL</p>
        <h1>Order Dashboard</h1>
        <p class="muted">Manage your packaging orders</p>
      </div>

      <button v-if="user" class="logout-btn" @click="logout">
        Log Out
      </button>
    </header>

    <section v-if="!user" class="login-card">
      <div class="login-icon">📦</div>
      <h2>Admin Login</h2>
      <p class="muted">Sign in to manage customer orders.</p>

      <form @submit.prevent="login">
        <label for="admin-email">Email address</label>
        <input
          id="admin-email"
          v-model="email"
          type="email"
          autocomplete="username"
          placeholder="admin@example.com"
          required
        />

        <label for="admin-password">Password</label>
        <input
          id="admin-password"
          v-model="password"
          type="password"
          autocomplete="current-password"
          placeholder="Enter password"
          required
        />

        <p v-if="loginError" class="error-message" role="alert">
          {{ loginError }}
        </p>

        <button class="primary-btn" type="submit" :disabled="loading">
          {{ loading ? 'Signing in...' : 'Sign In' }}
        </button>
      </form>
    </section>

    <section v-else class="dashboard-content">
      <div class="stats-row">
        <article class="stat-card">
          <span>Total Orders</span>
          <strong>{{ orders.length }}</strong>
        </article>
        <article class="stat-card">
          <span>Pending Orders</span>
          <strong>
            {{ orders.filter(order => order.status === 'pending').length }}
          </strong>
        </article>
        <article class="stat-card">
          <span>Delivered</span>
          <strong>
            {{ orders.filter(order => order.status === 'delivered').length }}
          </strong>
        </article>
      </div>

      <p v-if="ordersError" class="error-message" role="alert">
        {{ ordersError }}
      </p>

      <div v-if="loading" class="empty-message">Loading orders...</div>

      <div v-else-if="!orders.length" class="empty-message">
        <span>📦</span>
        <h2>No orders yet</h2>
        <p>New customer orders will appear here.</p>
      </div>

      <div v-else class="orders-list">
        <article v-for="order in orders" :key="order.id" class="order-card">
          <div class="order-top">
            <div>
              <p class="order-number">#{{ order.orderNumber || order.id }}</p>
              <h2>{{ order.customer?.fullName || 'Customer' }}</h2>
              <p class="muted">{{ formatDate(order.createdAt) }}</p>
            </div>

            <strong class="order-total">{{ money(order.total) }}</strong>
          </div>

          <div class="customer-details">
            <p><strong>Phone:</strong> {{ order.customer?.phoneNumber || '—' }}</p>
            <p><strong>WhatsApp:</strong> {{ order.customer?.whatsappNumber || '—' }}</p>
            <p>
              <strong>Address:</strong>
              {{ order.customer?.address || '—' }},
              {{ order.customer?.area || '' }},
              {{ order.customer?.city || '' }}
            </p>
            <p v-if="order.customer?.landmark">
              <strong>Landmark:</strong> {{ order.customer.landmark }}
            </p>
            <p><strong>Payment:</strong> {{ order.paymentMethod || 'COD' }}</p>
          </div>

          <div class="ordered-items">
            <h3>Items</h3>
            <div
              v-for="(item, index) in order.items || []"
              :key="`${item.productId || item.name}-${index}`"
              class="item-row"
            >
              <div>
                <strong>{{ item.name }}</strong>
                <p class="muted">
                  <span v-if="item.selectedVariants?.size">
                    {{ item.selectedVariants.size }} ·
                  </span>
                  <span v-if="item.selectedVariants?.thickness">
                    {{ item.selectedVariants.thickness }} ·
                  </span>
                  Qty: {{ item.quantity }}
                </p>
              </div>
              <span>{{ money(item.unitPrice * item.quantity) }}</span>
            </div>

            <div class="totals">
              <p>Subtotal <span>{{ money(order.subtotal) }}</span></p>
              <p>Delivery <span>{{ money(order.deliveryCharge) }}</span></p>
              <p class="grand-total">Total <span>{{ money(order.total) }}</span></p>
            </div>
          </div>

          <div class="status-row">
            <label :for="`status-${order.id}`">Order status</label>
            <select
              :id="`status-${order.id}`"
              :value="order.status || 'pending'"
              :disabled="updatingOrder === order.id"
              @change="changeStatus(order, $event.target.value)"
            >
              <option
                v-for="status in statusOptions"
                :key="status"
                :value="status"
              >
                {{ statusLabel(status) }}
              </option>
            </select>
            <span v-if="updatingOrder === order.id" class="muted">
              Saving...
            </span>
          </div>
        </article>
      </div>
    </section>
  </main>
</template>

<style scoped>
.admin-page {
  min-height: 100vh;
  padding: 32px;
  background: #f4f6fa;
  color: #172033;
  font-family: inherit;
}
.admin-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  max-width: 1100px;
  margin: 0 auto 28px;
}
.eyebrow {
  color: #0052cc;
  font-size: 12px;
  font-weight: 800;
  letter-spacing: 2px;
}
h1 { margin: 4px 0; font-size: clamp(26px, 4vw, 36px); }
.muted { color: #6b7280; font-size: 13px; }
.login-card {
  max-width: 420px;
  margin: 50px auto;
  padding: 32px;
  border-radius: 20px;
  background: white;
  box-shadow: 0 12px 40px #1720330d;
}
.login-card h2 { margin: 8px 0; }
.login-icon { font-size: 36px; }
label { display: block; margin: 16px 0 7px; font-size: 13px; font-weight: 700; }
input, select {
  width: 100%;
  min-width: 0;
  padding: 12px;
  border: 1px solid #dce1ea;
  border-radius: 9px;
  background: white;
  color: #172033;
  font: inherit;
}
.primary-btn, .logout-btn {
  padding: 12px 18px;
  border: 0;
  border-radius: 9px;
  cursor: pointer;
  font-weight: 700;
}
.primary-btn {
  width: 100%;
  margin-top: 20px;
  background: #0052cc;
  color: white;
}
.logout-btn { background: #e8edf5; color: #172033; }
button:disabled, select:disabled { opacity: .6; cursor: wait; }
.dashboard-content { max-width: 1100px; margin: 0 auto; }
.stats-row {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 16px;
  margin-bottom: 24px;
}
.stat-card, .order-card, .empty-message {
  border: 1px solid #e8ebf1;
  border-radius: 16px;
  background: white;
}
.stat-card { padding: 20px; }
.stat-card span { display: block; color: #6b7280; font-size: 13px; }
.stat-card strong { display: block; margin-top: 8px; font-size: 28px; }
.orders-list { display: grid; gap: 18px; }
.order-card { padding: 22px; }
.order-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  padding-bottom: 16px;
  border-bottom: 1px solid #edf0f5;
}
.order-number { color: #0052cc; font-size: 12px; font-weight: 800; }
.order-top h2 { margin: 5px 0; font-size: 20px; }
.order-total { font-size: 20px; white-space: nowrap; }
.customer-details { padding: 12px 0; overflow-wrap: anywhere; }
.customer-details p { margin: 7px 0; font-size: 14px; }
.ordered-items h3 { margin: 8px 0 12px; font-size: 15px; }
.item-row {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  padding: 10px 0;
  border-bottom: 1px solid #edf0f5;
  font-size: 14px;
}
.item-row > span { white-space: nowrap; }
.item-row p { margin: 5px 0 0; }
.totals { max-width: 320px; margin: 14px 0 14px auto; }
.totals p { display: flex; justify-content: space-between; gap: 20px; font-size: 14px; }
.grand-total { padding-top: 10px; border-top: 1px solid #e5e7eb; font-weight: 800; }
.status-row {
  display: grid;
  grid-template-columns: auto minmax(150px, 230px) auto;
  align-items: center;
  gap: 12px;
  padding-top: 16px;
  border-top: 1px solid #edf0f5;
}
.status-row label { margin: 0; }
.error-message { color: #b91c1c; font-size: 13px; overflow-wrap: anywhere; }
.empty-message { padding: 40px 20px; text-align: center; }
.empty-message > span { font-size: 36px; }
@media (max-width: 600px) {
  .admin-page { padding: 20px 12px; }
  .admin-header { align-items: flex-start; }
  .stats-row { gap: 8px; }
  .stat-card { padding: 13px; }
  .stat-card strong { font-size: 23px; }
  .stat-card span { font-size: 11px; }
  .order-card { padding: 16px; }
  .order-top { flex-direction: column; }
  .status-row { grid-template-columns: 1fr; }
  .status-row select { width: 100%; }
}
</style>
