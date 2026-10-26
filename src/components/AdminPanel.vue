
<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
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
const ordersError = ref('')
const loginLoading = ref(false)
const ordersLoading = ref(true)
const updatingOrder = ref('')
const sidebarOpen = ref(false)

const orders = ref([])
const searchQuery = ref('')
const statusFilter = ref('all')
const paymentFilter = ref('all')
const activePage = ref('overview')

let unsubscribeOrders = null
let unsubscribeAuth = null

const statuses = [
  { value: 'pending', label: 'Pending', color: 'amber' },
  { value: 'confirmed', label: 'Confirmed', color: 'blue' },
  {
    value: 'bought_from_vendor',
    label: 'Bought from vendor',
    color: 'purple',
  },
  { value: 'packed', label: 'Packed', color: 'cyan' },
  { value: 'dispatched', label: 'Dispatched', color: 'indigo' },
  { value: 'delivered', label: 'Delivered', color: 'green' },
  { value: 'cancelled', label: 'Cancelled', color: 'red' },
]

const statusMap = Object.fromEntries(
  statuses.map(status => [status.value, status])
)

const navItems = [
  { id: 'overview', label: 'Overview', icon: 'grid' },
  { id: 'orders', label: 'Orders', icon: 'package' },
]

const formatDate = timestamp => {
  if (!timestamp) return '—'

  try {
    const date = typeof timestamp.toDate === 'function'
      ? timestamp.toDate()
      : new Date(timestamp)

    return date.toLocaleDateString('en-PK', {
      day: '2-digit',
      month: 'short',
      year: 'numeric',
    })
  } catch {
    return '—'
  }
}

const formatDateTime = timestamp => {
  if (!timestamp) return '—'

  try {
    const date = typeof timestamp.toDate === 'function'
      ? timestamp.toDate()
      : new Date(timestamp)

    return date.toLocaleString('en-PK', {
      day: 'numeric',
      month: 'short',
      year: 'numeric',
      hour: '2-digit',
      minute: '2-digit',
    })
  } catch {
    return '—'
  }
}

const money = value =>
  `Rs. ${Number(value || 0).toLocaleString('en-PK')}`

const statusLabel = status =>
  statusMap[status]?.label || status || 'Pending'

const statusColor = status =>
  statusMap[status]?.color || 'gray'

const customerName = order =>
  order.customer?.fullName || 'Unknown customer'

const orderNumber = order =>
  order.orderNumber || order.id

const itemCount = order =>
  (order.items || []).reduce(
    (total, item) => total + Number(item.quantity || 0),
    0
  )

const totalOrders = computed(() => orders.value.length)

const pendingOrders = computed(() =>
  orders.value.filter(order =>
    ['pending', 'confirmed', 'bought_from_vendor', 'packed', 'dispatched']
      .includes(order.status || 'pending')
  ).length
)

const deliveredOrders = computed(() =>
  orders.value.filter(order => order.status === 'delivered').length
)

const cancelledOrders = computed(() =>
  orders.value.filter(order => order.status === 'cancelled').length
)

const totalSales = computed(() =>
  orders.value
    .filter(order => order.status !== 'cancelled')
    .reduce((sum, order) => sum + Number(order.total || 0), 0)
)

const filteredOrders = computed(() => {
  const search = searchQuery.value.trim().toLowerCase()

  return [...orders.value]
    .filter(order => {
      const matchesSearch = !search || [
        orderNumber(order),
        customerName(order),
        order.customer?.phoneNumber,
        order.customer?.whatsappNumber,
        order.customer?.city,
      ].some(value => String(value || '').toLowerCase().includes(search))

      const matchesStatus =
        statusFilter.value === 'all' ||
        (order.status || 'pending') === statusFilter.value

      const paymentMethod = order.paymentMethod || 'COD'
      const matchesPayment =
        paymentFilter.value === 'all' ||
        paymentMethod.toLowerCase() === paymentFilter.value.toLowerCase()

      return matchesSearch && matchesStatus && matchesPayment
    })
    .sort((a, b) => {
      const aDate = a.createdAt?.toMillis?.() || 0
      const bDate = b.createdAt?.toMillis?.() || 0
      return bDate - aDate
    })
})

const recentOrders = computed(() => filteredOrders.value.slice(0, 6))

const pageTitle = computed(() =>
  activePage.value === 'orders' ? 'Orders management' : 'Dashboard overview'
)

function watchOrders() {
  if (unsubscribeOrders) unsubscribeOrders()

  ordersLoading.value = true
  ordersError.value = ''

  const ordersQuery = query(
    collection(db, 'orders'),
    orderBy('createdAt', 'desc')
  )

  unsubscribeOrders = onSnapshot(
    ordersQuery,
    snapshot => {
      orders.value = snapshot.docs.map(document => ({
        id: document.id,
        ...document.data(),
      }))

      ordersLoading.value = false
    },
    error => {
      console.error('Error loading orders:', error)
      ordersError.value = 'Unable to load orders. Please check your Firestore permissions.'
      ordersLoading.value = false
    }
  )
}

async function login() {
  loginError.value = ''
  loginLoading.value = true

  try {
    await signInWithEmailAndPassword(
      auth,
      email.value.trim(),
      password.value
    )
  } catch (error) {
    console.error('Login failed:', error)

    const messages = {
      'auth/invalid-credential': 'Incorrect email or password.',
      'auth/invalid-email': 'Please enter a valid email address.',
      'auth/user-not-found': 'No account was found with this email.',
      'auth/wrong-password': 'Incorrect email or password.',
      'auth/too-many-requests': 'Too many attempts. Please try again later.',
    }

    loginError.value =
      messages[error.code] || 'Unable to sign in. Please try again.'
  } finally {
    loginLoading.value = false
  }
}

async function logout() {
  try {
    await signOut(auth)
    activePage.value = 'overview'
    searchQuery.value = ''
    statusFilter.value = 'all'
    password.value = ''
  } catch (error) {
    console.error('Logout failed:', error)
    ordersError.value = 'Unable to sign out. Please try again.'
  }
}

async function changeStatus(order, newStatus) {
  if (!order?.id || !newStatus || updatingOrder.value) return
  if (order.status === newStatus) return

  updatingOrder.value = order.id
  ordersError.value = ''

  try {
    await updateDoc(doc(db, 'orders', order.id), {
      status: newStatus,
    })
  } catch (error) {
    console.error('Status update failed:', error)
    ordersError.value = 'Could not update the order status. Check your permissions and try again.'
  } finally {
    updatingOrder.value = ''
  }
}

function openOrders(filter = 'all') {
  activePage.value = 'orders'
  statusFilter.value = filter
  sidebarOpen.value = false
}

function navigate(page) {
  activePage.value = page
  sidebarOpen.value = false

  if (page === 'overview') {
    statusFilter.value = 'all'
  }
}

onMounted(() => {
  unsubscribeAuth = onAuthStateChanged(auth, currentUser => {
    user.value = currentUser

    if (currentUser) {
      if (!unsubscribeOrders) watchOrders()
    } else {
      if (unsubscribeOrders) {
        unsubscribeOrders()
        unsubscribeOrders = null
      }

      orders.value = []
      ordersLoading.value = false
    }
  })
})

onUnmounted(() => {
  if (unsubscribeOrders) unsubscribeOrders()
  if (unsubscribeAuth) unsubscribeAuth()
})
</script>

<template>
  <!-- LOGIN -->
  <main v-if="!user" class="login-screen">
    <div class="login-decoration login-decoration-one"></div>
    <div class="login-decoration login-decoration-two"></div>

    <section class="login-card">
      <div class="brand-lockup">
        <div class="brand-symbol">
          <svg viewBox="0 0 40 40" fill="none" aria-hidden="true">
            <rect x="5" y="5" width="30" height="30" rx="10" stroke="currentColor" stroke-width="2.5"/>
            <path d="M12 15H28M12 20H25M12 25H22" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/>
          </svg>
        </div>

        <div>
          <h1>Wrap <span>N</span> Roll</h1>
          <p>BUSINESS MANAGEMENT</p>
        </div>
      </div>

      <div class="login-heading">
        <span class="eyebrow">ADMINISTRATOR ACCESS</span>
        <h2>Welcome back.</h2>
        <p>Sign in to manage your orders and operations.</p>
      </div>

      <form class="login-form" @submit.prevent="login">
        <label for="admin-email">Email address</label>
        <input
          id="admin-email"
          v-model="email"
          type="email"
          placeholder="admin@example.com"
          autocomplete="username"
          required
        />

        <label for="admin-password">Password</label>
        <input
          id="admin-password"
          v-model="password"
          type="password"
          placeholder="Enter your password"
          autocomplete="current-password"
          required
        />

        <div v-if="loginError" class="alert alert-error">
          {{ loginError }}
        </div>

        <button class="primary-button login-button" type="submit" :disabled="loginLoading">
          <span>{{ loginLoading ? 'Signing in...' : 'Sign in to dashboard' }}</span>
          <svg v-if="!loginLoading" viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <path d="M5 12H19M12 5L19 12L12 19" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </button>
      </form>

      <div class="login-footer">
        <span class="secure-dot"></span>
        Secure administrator portal
      </div>
    </section>

    <p class="login-copyright">© {{ new Date().getFullYear() }} Wrap N Roll. All rights reserved.</p>
  </main>

  <!-- DASHBOARD -->
  <div v-else class="app-shell">
    <div
      v-if="sidebarOpen"
      class="mobile-overlay"
      @click="sidebarOpen = false"
    ></div>

    <aside class="sidebar" :class="{ 'sidebar-open': sidebarOpen }">
      <div class="sidebar-brand">
        <div class="brand-symbol sidebar-symbol">
          <svg viewBox="0 0 40 40" fill="none" aria-hidden="true">
            <rect x="5" y="5" width="30" height="30" rx="10" stroke="currentColor" stroke-width="2.5"/>
            <path d="M12 15H28M12 20H25M12 25H22" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/>
          </svg>
        </div>

        <div class="sidebar-brand-text">
          <h1>Wrap <span>N</span> Roll</h1>
          <p>OPERATIONS</p>
        </div>

        <button class="sidebar-close" aria-label="Close menu" @click="sidebarOpen = false">×</button>
      </div>

      <div class="nav-caption">WORKSPACE</div>

      <nav class="sidebar-nav">
        <button
          v-for="item in navItems"
          :key="item.id"
          class="nav-item"
          :class="{ 'nav-item-active': activePage === item.id }"
          @click="navigate(item.id)"
        >
          <svg v-if="item.icon === 'grid'" viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <rect x="3.5" y="3.5" width="7" height="7" rx="1.5" stroke="currentColor" stroke-width="1.7"/>
            <rect x="13.5" y="3.5" width="7" height="7" rx="1.5" stroke="currentColor" stroke-width="1.7"/>
            <rect x="3.5" y="13.5" width="7" height="7" rx="1.5" stroke="currentColor" stroke-width="1.7"/>
            <rect x="13.5" y="13.5" width="7" height="7" rx="1.5" stroke="currentColor" stroke-width="1.7"/>
          </svg>

          <svg v-else viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <path d="M4 7.5L12 3L20 7.5V17L12 21L4 17V7.5Z" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round"/>
            <path d="M4.5 7.5L12 12L19.5 7.5M12 12V20.5" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round"/>
          </svg>

          <span>{{ item.label }}</span>
          <span v-if="item.id === 'orders' && pendingOrders" class="nav-count">{{ pendingOrders }}</span>
        </button>
      </nav>

      <div class="nav-caption nav-caption-second">ORDER STATUS</div>

      <nav class="sidebar-nav status-navigation">
        <button class="nav-item" @click="openOrders('pending')">
          <span class="nav-status-dot dot-amber"></span>
          <span>Pending</span>
          <span class="nav-count">{{ orders.filter(o => (o.status || 'pending') === 'pending').length }}</span>
        </button>

        <button class="nav-item" @click="openOrders('dispatched')">
          <span class="nav-status-dot dot-indigo"></span>
          <span>Dispatched</span>
        </button>

        <button class="nav-item" @click="openOrders('delivered')">
          <span class="nav-status-dot dot-green"></span>
          <span>Delivered</span>
        </button>

        <button class="nav-item" @click="openOrders('cancelled')">
          <span class="nav-status-dot dot-red"></span>
          <span>Cancelled</span>
        </button>
      </nav>

      <div class="sidebar-bottom">
        <div class="sidebar-help">
          <div class="help-icon">?</div>
          <div>
            <strong>Need assistance?</strong>
            <p>Your operations dashboard</p>
          </div>
        </div>

        <div class="sidebar-user">
          <div class="avatar">{{ (user.email || 'A').charAt(0).toUpperCase() }}</div>
          <div class="sidebar-user-info">
            <strong>Administrator</strong>
            <span>{{ user.email }}</span>
          </div>
          <button class="logout-button" title="Sign out" aria-label="Sign out" @click="logout">
            <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
              <path d="M10 17L15 12L10 7M15 12H3M13 3H19C20.1 3 21 3.9 21 5V19C21 20.1 20.1 21 19 21H13" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </button>
        </div>
      </div>
    </aside>

    <main class="main-content">
      <header class="topbar">
        <div class="topbar-left">
          <button class="mobile-menu-button" aria-label="Open menu" @click="sidebarOpen = true">
            <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
              <path d="M4 7H20M4 12H20M4 17H20" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
            </svg>
          </button>

          <div>
            <div class="breadcrumb">Workspace <span>/</span> {{ activePage === 'orders' ? 'Orders' : 'Overview' }}</div>
            <h2 class="topbar-title">{{ pageTitle }}</h2>
          </div>
        </div>

        <div class="topbar-right">
          <div class="live-indicator">
            <span></span>
            Live data
          </div>

          <div class="topbar-divider"></div>

          <div class="topbar-date">
            <span class="date-label">TODAY</span>
            <strong>{{ new Date().toLocaleDateString('en-PK', { day: '2-digit', month: 'short', year: 'numeric' }) }}</strong>
          </div>

          <div class="topbar-avatar">{{ (user.email || 'A').charAt(0).toUpperCase() }}</div>
        </div>
      </header>

      <div class="page-content">
        <div v-if="ordersError" class="alert alert-error dashboard-alert">
          <span>{{ ordersError }}</span>
          <button @click="ordersError = ''" aria-label="Dismiss error">×</button>
        </div>

        <!-- OVERVIEW -->
        <template v-if="activePage === 'overview'">
          <section class="welcome-row">
            <div>
              <div class="eyebrow">YOUR BUSINESS AT A GLANCE</div>
              <h1>Good to see you again <span class="wave">✳</span></h1>
              <p>Here’s what’s happening with your store today.</p>
            </div>

            <button class="primary-button" @click="openOrders()">
              <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                <path d="M4 5H20V19H4V5ZM8 9H16M8 13H16M8 17H12" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
              Manage orders
            </button>
          </section>

          <section class="stats-grid">
            <article class="stat-card">
              <div class="stat-top">
                <span class="stat-label">Total orders</span>
                <div class="stat-icon icon-blue">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <path d="M4 7.5L12 3L20 7.5V17L12 21L4 17V7.5Z" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round"/>
                    <path d="M4.5 7.5L12 12L19.5 7.5M12 12V20.5" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round"/>
                  </svg>
                </div>
              </div>
              <div class="stat-value">{{ ordersLoading ? '—' : totalOrders.toLocaleString('en-PK') }}</div>
              <div class="stat-bottom">
                <span class="stat-mini-dot blue-dot"></span>
                <span>All recorded orders</span>
              </div>
            </article>

            <article class="stat-card">
              <div class="stat-top">
                <span class="stat-label">Awaiting completion</span>
                <div class="stat-icon icon-amber">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <circle cx="12" cy="12" r="8.5" stroke="currentColor" stroke-width="1.7"/>
                    <path d="M12 7V12L15.5 14" stroke="currentColor" stroke-width="1.7" stroke-linecap="round"/>
                  </svg>
                </div>
              </div>
              <div class="stat-value">{{ ordersLoading ? '—' : pendingOrders.toLocaleString('en-PK') }}</div>
              <div class="stat-bottom">
                <span class="stat-mini-dot amber-dot"></span>
                <span>Active orders in progress</span>
              </div>
            </article>

            <article class="stat-card">
              <div class="stat-top">
                <span class="stat-label">Delivered orders</span>
                <div class="stat-icon icon-green">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <path d="M5 12.5L10 17L19 7" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                    <circle cx="12" cy="12" r="9" stroke="currentColor" stroke-width="1.7"/>
                  </svg>
                </div>
              </div>
              <div class="stat-value">{{ ordersLoading ? '—' : deliveredOrders.toLocaleString('en-PK') }}</div>
              <div class="stat-bottom">
                <span class="stat-mini-dot green-dot"></span>
                <span>Successfully completed</span>
              </div>
            </article>

            <article class="stat-card revenue-card">
              <div class="stat-top">
                <span class="stat-label">Order value</span>
                <div class="stat-icon icon-purple">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <rect x="3.5" y="5.5" width="17" height="13" rx="2.5" stroke="currentColor" stroke-width="1.7"/>
                    <path d="M3.5 10H20.5M8 14H11" stroke="currentColor" stroke-width="1.7" stroke-linecap="round"/>
                  </svg>
                </div>
              </div>
              <div class="stat-value stat-money">{{ ordersLoading ? '—' : money(totalSales) }}</div>
              <div class="stat-bottom">
                <span class="stat-mini-dot purple-dot"></span>
                <span>Excludes cancelled orders</span>
              </div>
            </article>
          </section>

          <section class="overview-grid">
            <article class="panel orders-panel">
              <div class="panel-heading">
                <div>
                  <h3>Recent orders</h3>
                  <p>Your latest customer purchases</p>
                </div>
                <button class="text-button" @click="openOrders()">View all orders <span>→</span></button>
              </div>

              <div v-if="ordersLoading" class="loading-state">
                <span class="spinner"></span>
                Loading orders...
              </div>

              <div v-else-if="recentOrders.length === 0" class="empty-state">
                <div class="empty-icon">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <path d="M4 7.5L12 3L20 7.5V17L12 21L4 17V7.5Z" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
                    <path d="M4.5 7.5L12 12L19.5 7.5" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
                  </svg>
                </div>
                <strong>No orders yet</strong>
                <p>New customer orders will appear here.</p>
              </div>

              <div v-else class="table-scroll">
                <table class="orders-table">
                  <thead>
                    <tr>
                      <th>ORDER</th>
                      <th>CUSTOMER</th>
                      <th>DATE</th>
                      <th>AMOUNT</th>
                      <th>STATUS</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="order in recentOrders" :key="order.id">
                      <td>
                        <button class="order-id-button" @click="openOrders()">
                          #{{ orderNumber(order) }}
                        </button>
                        <span class="cell-subtext">{{ itemCount(order) }} items</span>
                      </td>
                      <td>
                        <div class="customer-cell">
                          <div class="customer-avatar">{{ customerName(order).charAt(0).toUpperCase() }}</div>
                          <div class="customer-cell-info">
                            <strong>{{ customerName(order) }}</strong>
                            <span>{{ order.customer?.city || 'Location unavailable' }}</span>
                          </div>
                        </div>
                      </td>
                      <td class="date-cell">{{ formatDate(order.createdAt) }}</td>
                      <td class="amount-cell">{{ money(order.total) }}</td>
                      <td>
                        <span class="status-badge" :class="`status-${statusColor(order.status || 'pending')}`">
                          <span class="status-dot"></span>
                          {{ statusLabel(order.status || 'pending') }}
                        </span>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </article>

            <article class="panel status-panel">
              <div class="panel-heading">
                <div>
                  <h3>Order snapshot</h3>
                  <p>Current order distribution</p>
                </div>
                <div class="panel-icon">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <path d="M4 19V5M4 19H21M8 15V11M13 15V7M18 15V9" stroke="currentColor" stroke-width="1.7" stroke-linecap="round"/>
                  </svg>
                </div>
              </div>

              <div class="snapshot-total">
                <strong>{{ totalOrders }}</strong>
                <span>total orders</span>
              </div>

              <div class="snapshot-bar" aria-label="Order status distribution">
                <span class="bar-delivered" :style="{ width: `${totalOrders ? deliveredOrders / totalOrders * 100 : 0}%` }"></span>
                <span class="bar-active" :style="{ width: `${totalOrders ? pendingOrders / totalOrders * 100 : 0}%` }"></span>
                <span class="bar-cancelled" :style="{ width: `${totalOrders ? cancelledOrders / totalOrders * 100 : 0}%` }"></span>
              </div>

              <div class="snapshot-legend">
                <button @click="openOrders('delivered')">
                  <span class="legend-dot green-dot"></span>
                  <span>Delivered</span>
                  <strong>{{ deliveredOrders }}</strong>
                </button>
                <button @click="openOrders('pending')">
                  <span class="legend-dot amber-dot"></span>
                  <span>Active pipeline</span>
                  <strong>{{ pendingOrders }}</strong>
                </button>
                <button @click="openOrders('cancelled')">
                  <span class="legend-dot red-dot"></span>
                  <span>Cancelled</span>
                  <strong>{{ cancelledOrders }}</strong>
                </button>
              </div>

              <div class="snapshot-footer">
                <div class="snapshot-footer-icon">↗</div>
                <div>
                  <strong>Keep things moving</strong>
                  <p>Update order statuses as you fulfil each purchase.</p>
                </div>
              </div>
            </article>
          </section>

          <section class="bottom-banner">
            <div class="banner-mark">
              <svg viewBox="0 0 40 40" fill="none" aria-hidden="true">
                <rect x="5" y="5" width="30" height="30" rx="10" stroke="currentColor" stroke-width="2"/>
                <path d="M12 15H28M12 20H25M12 25H22" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/>
              </svg>
            </div>
            <div class="banner-copy">
              <span>WRAP N ROLL · OPERATIONS</span>
              <h3>Every order, organised.</h3>
              <p>Manage your packaging business from one central workspace.</p>
            </div>
            <button class="banner-button" @click="openOrders()">Open order management <span>→</span></button>
          </section>
        </template>

        <!-- ORDER MANAGEMENT -->
        <template v-else>
          <section class="welcome-row orders-welcome">
            <div>
              <div class="eyebrow">FULFILMENT WORKSPACE</div>
              <h1>Manage your orders</h1>
              <p>Review customer purchases and keep every order moving.</p>
            </div>
            <div class="orders-count-chip">
              <strong>{{ filteredOrders.length }}</strong>
              <span>orders shown</span>
            </div>
          </section>

          <section class="order-summary-strip">
            <button :class="{ 'summary-filter-active': statusFilter === 'all' }" @click="statusFilter = 'all'">
              <span>All orders</span>
              <strong>{{ totalOrders }}</strong>
            </button>
            <button :class="{ 'summary-filter-active': statusFilter === 'pending' }" @click="statusFilter = 'pending'">
              <span>Pending</span>
              <strong>{{ orders.filter(o => (o.status || 'pending') === 'pending').length }}</strong>
            </button>
            <button :class="{ 'summary-filter-active': statusFilter === 'dispatched' }" @click="statusFilter = 'dispatched'">
              <span>Dispatched</span>
              <strong>{{ orders.filter(o => o.status === 'dispatched').length }}</strong>
            </button>
            <button :class="{ 'summary-filter-active': statusFilter === 'delivered' }" @click="statusFilter = 'delivered'">
              <span>Delivered</span>
              <strong>{{ deliveredOrders }}</strong>
            </button>
          </section>

          <section class="panel management-panel">
            <div class="management-toolbar">
              <div>
                <h3>All orders</h3>
                <p>Search, filter and update your customer orders.</p>
              </div>
              <div class="toolbar-controls">
                <label class="search-box">
                  <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                    <circle cx="10.8" cy="10.8" r="6.8" stroke="currentColor" stroke-width="1.8"/>
                    <path d="M16 16L21 21" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
                  </svg>
                  <input v-model="searchQuery" type="search" placeholder="Search orders..." aria-label="Search orders" />
                </label>

                <select v-model="statusFilter" aria-label="Filter by status" class="filter-select">
                  <option value="all">All statuses</option>
                  <option v-for="status in statuses" :key="status.value" :value="status.value">
                    {{ status.label }}
                  </option>
                </select>

                <select v-model="paymentFilter" aria-label="Filter by payment" class="filter-select">
                  <option value="all">All payments</option>
                  <option value="COD">Cash on delivery</option>
                  <option value="Bank Transfer">Bank transfer</option>
                  <option value="Easypaisa">Easypaisa</option>
                  <option value="JazzCash">JazzCash</option>
                </select>
              </div>
            </div>

            <div v-if="ordersLoading" class="loading-state">
              <span class="spinner"></span>
              Loading orders...
            </div>

            <div v-else-if="filteredOrders.length === 0" class="empty-state management-empty">
              <div class="empty-icon">
                <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                  <circle cx="10.8" cy="10.8" r="6.8" stroke="currentColor" stroke-width="1.6"/>
                  <path d="M16 16L21 21" stroke="currentColor" stroke-width="1.6" stroke-linecap="round"/>
                </svg>
              </div>
              <strong>No matching orders</strong>
              <p>Try changing your search or filters.</p>
              <button class="text-button" @click="searchQuery = ''; statusFilter = 'all'; paymentFilter = 'all'">
                Clear filters
              </button>
            </div>

            <div v-else class="table-scroll">
              <table class="orders-table management-table">
                <thead>
                  <tr>
                    <th>ORDER DETAILS</th>
                    <th>CUSTOMER</th>
                    <th>DATE</th>
                    <th>PAYMENT</th>
                    <th>ORDER TOTAL</th>
                    <th>STATUS</th>
                    <th>UPDATE STATUS</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="order in filteredOrders" :key="order.id">
                    <td>
                      <strong class="table-order-number">#{{ orderNumber(order) }}</strong>
                      <span class="cell-subtext">{{ itemCount(order) }} items ordered</span>
                    </td>
                    <td>
                      <div class="customer-cell">
                        <div class="customer-avatar">{{ customerName(order).charAt(0).toUpperCase() }}</div>
                        <div class="customer-cell-info">
                          <strong>{{ customerName(order) }}</strong>
                          <span>{{ order.customer?.phoneNumber || 'No phone number' }}</span>
                        </div>
                      </div>
                    </td>
                    <td class="date-cell">{{ formatDate(order.createdAt) }}</td>
                    <td>
                      <span class="payment-label">{{ order.paymentMethod || 'COD' }}</span>
                    </td>
                    <td class="amount-cell">{{ money(order.total) }}</td>
                    <td>
                      <span class="status-badge" :class="`status-${statusColor(order.status || 'pending')}`">
                        <span class="status-dot"></span>
                        {{ statusLabel(order.status || 'pending') }}
                      </span>
                    </td>
                    <td>
                      <select
                        class="status-select"
                        :class="`select-${statusColor(order.status || 'pending')}`"
                        :value="order.status || 'pending'"
                        :disabled="updatingOrder === order.id"
                        :aria-label="`Update status for order ${orderNumber(order)}`"
                        @change="changeStatus(order, $event.target.value)"
                      >
                        <option v-for="status in statuses" :key="status.value" :value="status.value">
                          {{ status.label }}
                        </option>
                      </select>
                      <span v-if="updatingOrder === order.id" class="updating-label">Saving...</span>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div class="table-footer">
              <span>Showing <strong>{{ filteredOrders.length }}</strong> of <strong>{{ totalOrders }}</strong> orders</span>
              <span class="table-live"><span></span> Synced with Firebase</span>
            </div>
          </section>

          <!-- Additional order details -->
          <section class="panel order-details-panel">
            <div class="panel-heading">
              <div>
                <h3>Customer and fulfilment details</h3>
                <p>Select an order below to review its full details.</p>
              </div>
            </div>

            <div v-if="filteredOrders.length" class="detail-order-list">
              <details v-for="order in filteredOrders" :key="order.id" class="order-detail">
                <summary>
                  <div>
                    <strong>#{{ orderNumber(order) }} · {{ customerName(order) }}</strong>
                    <span>{{ formatDateTime(order.createdAt) }}</span>
                  </div>
                  <div class="detail-summary-right">
                    <strong>{{ money(order.total) }}</strong>
                    <span class="status-badge" :class="`status-${statusColor(order.status || 'pending')}`">
                      <span class="status-dot"></span>
                      {{ statusLabel(order.status || 'pending') }}
                    </span>
                  </div>
                </summary>

                <div class="detail-content">
                  <div class="detail-section">
                    <h4>Customer information</h4>
                    <p><span>Full name</span><strong>{{ customerName(order) }}</strong></p>
                    <p><span>Phone</span><strong>{{ order.customer?.phoneNumber || '—' }}</strong></p>
                    <p><span>WhatsApp</span><strong>{{ order.customer?.whatsappNumber || '—' }}</strong></p>
                    <p><span>City / area</span><strong>{{ [order.customer?.area, order.customer?.city].filter(Boolean).join(', ') || '—' }}</strong></p>
                    <p><span>Address</span><strong>{{ order.customer?.address || '—' }}</strong></p>
                    <p><span>Landmark</span><strong>{{ order.customer?.landmark || '—' }}</strong></p>
                  </div>

                  <div class="detail-section">
                    <h4>Order items</h4>
                    <div v-for="(item, index) in order.items || []" :key="item.productId || item.name || index" class="detail-item">
                      <div class="detail-item-icon">
                        <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
                          <path d="M4 7.5L12 3L20 7.5V17L12 21L4 17V7.5Z" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
                          <path d="M4.5 7.5L12 12L19.5 7.5" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
                        </svg>
                      </div>
                      <div class="detail-item-copy">
                        <strong>{{ item.name || item.productId || 'Product' }}</strong>
                        <span>
                          <template v-if="item.selectedVariants?.size">Size: {{ item.selectedVariants.size }} · </template>
                          <template v-if="item.selectedVariants?.thickness">Thickness: {{ item.selectedVariants.thickness }} · </template>
                          Qty: {{ item.quantity || 1 }}
                        </span>
                      </div>
                      <strong class="detail-item-price">{{ money(Number(item.unitPrice || 0) * Number(item.quantity || 1)) }}</strong>
                    </div>
                  </div>

                  <div class="detail-section">
                    <h4>Payment summary</h4>
                    <p><span>Payment method</span><strong>{{ order.paymentMethod || 'COD' }}</strong></p>
                    <p><span>Subtotal</span><strong>{{ money(order.subtotal) }}</strong></p>
                    <p><span>Delivery charge</span><strong>{{ money(order.deliveryCharge) }}</strong></p>
                    <p class="detail-total"><span>Order total</span><strong>{{ money(order.total) }}</strong></p>
                    <label class="detail-status-label">Update order status</label>
                    <select
                      class="status-select detail-status-select"
                      :class="`select-${statusColor(order.status || 'pending')}`"
                      :value="order.status || 'pending'"
                      :disabled="updatingOrder === order.id"
                      @change="changeStatus(order, $event.target.value)"
                    >
                      <option v-for="status in statuses" :key="status.value" :value="status.value">
                        {{ status.label }}
                      </option>
                    </select>
                  </div>
                </div>
              </details>
            </div>

            <div v-else class="detail-no-orders">Order details will appear here when orders are available.</div>
          </section>
        </template>

        <footer class="page-footer">
          <span>© {{ new Date().getFullYear() }} <strong>Wrap N Roll</strong></span>
          <span>Built for smoother business operations <span class="footer-heart">✦</span></span>
        </footer>
      </div>
    </main>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700;800&family=Manrope:wght@400;500;600;700;800&display=swap');

:root {
  font-family: 'DM Sans', sans-serif;
  color: #17243a;
  background: #f5f7fb;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;

  --navy: #142238;
  --navy-2: #1d304a;
  --blue: #3563eb;
  --blue-dark: #254ed0;
  --blue-soft: #edf3ff;
  --canvas: #f5f7fb;
  --white: #fff;
  --text: #17243a;
  --muted: #8792a5;
  --border: #e8ecf3;
  --green: #15845d;
  --amber: #b7791f;
  --red: #c13f4b;
  --radius: 15px;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  min-width: 320px;
  min-height: 100vh;
}

button,
input,
select {
  font: inherit;
}

button {
  cursor: pointer;
}

button:focus-visible,
input:focus-visible,
select:focus-visible,
summary:focus-visible {
  outline: 3px solid #9bb5ff;
  outline-offset: 3px;
}

button:disabled,
select:disabled {
  cursor: not-allowed;
  opacity: .65;
}

.login-screen {
  min-height: 100vh;
  display: grid;
  place-items: center;
  padding: 36px 20px 66px;
  position: relative;
  overflow: hidden;
  background:
    radial-gradient(ellipse at 12% 10%, rgba(74, 110, 220, .11), transparent 36%),
    radial-gradient(ellipse at 90% 85%, rgba(52, 99, 235, .09), transparent 34%),
    #f5f7fb;
}

.login-decoration {
  position: absolute;
  width: 310px;
  height: 310px;
  border: 1px solid rgba(53, 99, 235, .1);
  border-radius: 48px;
  transform: rotate(35deg);
  pointer-events: none;
}

.login-decoration-one {
  top: -200px;
  right: -60px;
}

.login-decoration-two {
  bottom: -235px;
  left: -75px;
  width: 390px;
  height: 390px;
}

.login-card {
  width: min(100%, 450px);
  padding: 38px;
  border: 1px solid rgba(225, 231, 241, .9);
  border-radius: 23px;
  background: rgba(255, 255, 255, .96);
  box-shadow: 0 22px 70px rgba(23, 36, 58, .09), 0 3px 9px rgba(23, 36, 58, .025);
  position: relative;
  z-index: 1;
}

.brand-lockup,
.sidebar-brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.brand-symbol {
  width: 46px;
  height: 46px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  color: white;
  border-radius: 14px;
  background: var(--blue);
  box-shadow: 0 7px 16px rgba(53, 99, 235, .24);
}

.brand-symbol svg {
  width: 35px;
  height: 35px;
}

.brand-lockup h1,
.sidebar-brand h1 {
  margin: 0;
  font: 800 21px/1.2 'Manrope', sans-serif;
  letter-spacing: -1px;
  color: var(--text);
}

.brand-lockup h1 span,
.sidebar-brand h1 span {
  color: var(--blue);
}

.brand-lockup p,
.sidebar-brand p {
  margin: 5px 0 0;
  color: #8b96a8;
  font-size: 9px;
  font-weight: 800;
  letter-spacing: 2px;
}

.login-heading {
  margin: 42px 0 27px;
}

.eyebrow {
  color: var(--blue);
  font-size: 10px;
  font-weight: 800;
  letter-spacing: 1.5px;
}

.login-heading h2 {
  margin: 11px 0 9px;
  color: var(--text);
  font: 800 32px/1.2 'Manrope', sans-serif;
  letter-spacing: -1.3px;
}

.login-heading p {
  margin: 0;
  color: var(--muted);
  font-size: 13px;
  line-height: 1.7;
}

.login-form {
  display: flex;
  flex-direction: column;
}

.login-form label {
  margin: 0 0 8px;
  color: #38465c;
  font-size: 12px;
  font-weight: 700;
}

.login-form input {
  width: 100%;
  height: 48px;
  padding: 0 14px;
  margin-bottom: 20px;
  border: 1px solid #e0e6f0;
  border-radius: 10px;
  background: #fbfcfe;
  color: var(--text);
  font-size: 13px;
  transition: border-color .2s, box-shadow .2s;
}

.login-form input::placeholder {
  color: #a6afbe;
}

.login-form input:focus {
  outline: none;
  border-color: #8aa5ff;
  box-shadow: 0 0 0 4px rgba(53, 99, 235, .08);
}

.primary-button {
  min-height: 43px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 0 17px;
  border: 0;
  border-radius: 10px;
  background: var(--blue);
  color: white;
  font-size: 12px;
  font-weight: 700;
  box-shadow: 0 5px 12px rgba(53, 99, 235, .14);
  transition: transform .2s, background .2s, box-shadow .2s;
}

.primary-button:hover:not(:disabled) {
  background: var(--blue-dark);
  box-shadow: 0 8px 19px rgba(53, 99, 235, .23);
  transform: translateY(-1px);
}

.primary-button svg {
  width: 17px;
  height: 17px;
}

.login-button {
  width: 100%;
  min-height: 49px;
  margin-top: 3px;
  justify-content: space-between;
  padding: 0 17px;
}

.login-footer {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-top: 26px;
  color: #8b96a8;
  font-size: 11px;
}

.secure-dot,
.live-indicator span,
.table-live span {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #19a875;
  box-shadow: 0 0 0 3px rgba(25, 168, 117, .1);
}

.login-copyright {
  position: absolute;
  bottom: 21px;
  left: 0;
  right: 0;
  margin: 0;
  text-align: center;
  color: #929bad;
  font-size: 11px;
}

.app-shell {
  min-height: 100vh;
  background: var(--canvas);
}

.sidebar {
  position: fixed;
  inset: 0 auto 0 0;
  z-index: 20;
  display: flex;
  flex-direction: column;
  width: 254px;
  padding: 25px 16px 16px;
  color: #e4eaf5;
  background: var(--navy);
  box-shadow: 6px 0 25px rgba(19, 34, 56, .045);
}

.sidebar-brand {
  padding: 0 9px 29px;
}

.sidebar-symbol {
  width: 42px;
  height: 42px;
  border-radius: 12px;
  background: #3563eb;
  box-shadow: none;
}

.sidebar-symbol svg {
  width: 32px;
  height: 32px;
}

.sidebar-brand h1 {
  color: #fff;
  font-size: 18px;
}

.sidebar-brand h1 span {
  color: #8da9ff;
}

.sidebar-brand p {
  color: #8494ac;
  font-size: 8px;
  letter-spacing: 1.8px;
}

.sidebar-close {
  display: none;
  margin-left: auto;
  border: 0;
  color: white;
  background: transparent;
  font-size: 27px;
}

.nav-caption {
  margin: 7px 10px 10px;
  color: #7e8ca3;
  font-size: 9px;
  font-weight: 800;
  letter-spacing: 1.5px;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.nav-item {
  width: 100%;
  min-height: 44px;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 0 12px;
  border: 0;
  border-radius: 9px;
  color: #aab6c8;
  background: transparent;
  text-align: left;
  font-size: 12px;
  font-weight: 600;
  transition: color .18s, background .18s;
}

.nav-item > svg {
  width: 18px;
  height: 18px;
  flex: 0 0 auto;
}

.nav-item:hover {
  color: #fff;
  background: rgba(255, 255, 255, .055);
}

.nav-item-active {
  color: #fff;
  background: #2e4e8e;
  box-shadow: inset 3px 0 #83a1ff;
}

.nav-item-active:hover {
  background: #2e4e8e;
}

.nav-count {
  min-width: 23px;
  height: 22px;
  display: grid;
  place-items: center;
  margin-left: auto;
  padding: 0 6px;
  border-radius: 7px;
  color: #d9e4ff;
  background: rgba(255, 255, 255, .1);
  font-size: 10px;
  font-weight: 800;
}

.nav-caption-second {
  margin-top: 32px;
}

.status-navigation .nav-item {
  min-height: 39px;
  font-size: 11px;
}

.nav-status-dot {
  width: 7px;
  height: 7px;
  flex: 0 0 auto;
  border-radius: 50%;
}

.dot-amber { background: #f4b740; }
.dot-indigo { background: #8c9cff; }
.dot-green { background: #46d5a0; }
.dot-red { background: #f07a86; }

.sidebar-bottom {
  margin-top: auto;
}

.sidebar-help {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 13px 10px;
  margin-bottom: 16px;
  border: 1px solid rgba(255, 255, 255, .075);
  border-radius: 11px;
  background: rgba(255, 255, 255, .035);
}

.help-icon {
  width: 28px;
  height: 28px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border-radius: 8px;
  color: #b7c8ff;
  background: rgba(99, 132, 221, .19);
  font-weight: 800;
}

.sidebar-help strong {
  display: block;
  color: #eaf0fb;
  font-size: 10px;
}

.sidebar-help p {
  margin: 4px 0 0;
  color: #8999b1;
  font-size: 9px;
}

.sidebar-user {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 15px 4px 3px;
  border-top: 1px solid rgba(255, 255, 255, .1);
}

.avatar,
.topbar-avatar {
  width: 34px;
  height: 34px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border-radius: 11px;
  color: #e4ebff;
  background: #2e4672;
  font-size: 12px;
  font-weight: 800;
}

.sidebar-user-info {
  min-width: 0;
  display: flex;
  flex: 1;
  flex-direction: column;
  gap: 4px;
}

.sidebar-user-info strong {
  color: #eef3fc;
  font-size: 10px;
}

.sidebar-user-info span {
  overflow: hidden;
  color: #8798b0;
  font-size: 9px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.logout-button {
  width: 31px;
  height: 31px;
  display: grid;
  place-items: center;
  border: 0;
  border-radius: 8px;
  color: #a9b6c9;
  background: transparent;
  transition: background .2s, color .2s;
}

.logout-button:hover {
  color: white;
  background: rgba(255, 255, 255, .1);
}

.logout-button svg {
  width: 18px;
  height: 18px;
}

.main-content {
  min-height: 100vh;
  margin-left: 254px;
}

.topbar {
  height: 84px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  padding: 0 34px;
  border-bottom: 1px solid var(--border);
  background: rgba(255, 255, 255, .94);
}

.topbar-left,
.topbar-right {
  display: flex;
  align-items: center;
}

.topbar-left {
  gap: 12px;
}

.breadcrumb {
  margin-bottom: 5px;
  color: #98a2b3;
  font-size: 10px;
  font-weight: 600;
}

.breadcrumb span {
  margin: 0 5px;
  color: #c5cbd6;
}

.topbar-title {
  margin: 0;
  color: var(--text);
  font: 800 18px/1.2 'Manrope', sans-serif;
  letter-spacing: -.5px;
}

.topbar-right {
  gap: 19px;
}

.live-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #65758b;
  font-size: 11px;
  font-weight: 600;
}

.live-indicator span {
  width: 6px;
  height: 6px;
}

.topbar-divider {
  width: 1px;
  height: 31px;
  background: var(--border);
}

.topbar-date {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.date-label {
  color: #99a3b3;
  font-size: 8px;
  font-weight: 800;
  letter-spacing: 1.3px;
}

.topbar-date strong {
  color: #35445b;
  font-size: 11px;
}

.topbar-avatar {
  width: 35px;
  height: 35px;
  color: #3359be;
  background: #eaf0ff;
}

.mobile-menu-button {
  display: none;
  width: 37px;
  height: 37px;
  place-items: center;
  border: 1px solid var(--border);
  border-radius: 9px;
  background: white;
  color: var(--text);
}

.mobile-menu-button svg {
  width: 19px;
  height: 19px;
}

.page-content {
  width: 100%;
  max-width: 1800px;
  padding: 31px 34px 20px;
  margin: 0 auto;
}

.welcome-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 25px;
}

.welcome-row h1 {
  margin: 8px 0 6px;
  color: #18263d;
  font: 800 clamp(22px, 2vw, 29px)/1.25 'Manrope', sans-serif;
  letter-spacing: -1.2px;
}

.welcome-row p {
  margin: 0;
  color: #8a96a8;
  font-size: 12px;
  line-height: 1.6;
}

.wave {
  color: #edb44b;
  font-size: 20px;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 17px;
  margin-bottom: 23px;
}

.stat-card {
  min-width: 0;
  padding: 19px 19px 17px;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  background: #fff;
  box-shadow: 0 3px 12px rgba(20, 34, 56, .018);
  transition: transform .2s, box-shadow .2s, border-color .2s;
}

.stat-card:hover {
  transform: translateY(-3px);
  border-color: #dce4f3;
  box-shadow: 0 10px 25px rgba(20, 34, 56, .055);
}

.stat-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 9px;
}

.stat-label {
  color: #778398;
  font-size: 11px;
  font-weight: 600;
}

.stat-icon {
  width: 36px;
  height: 36px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border-radius: 11px;
}

.stat-icon svg {
  width: 20px;
  height: 20px;
}

.icon-blue { color: #3d68e7; background: #edf2ff; }
.icon-amber { color: #c28a2e; background: #fff6e4; }
.icon-green { color: #19956b; background: #e7f8f0; }
.icon-purple { color: #8b5fdb; background: #f2edff; }

.stat-value {
  margin: 17px 0 13px;
  color: #1a2940;
  font: 800 29px/1 'Manrope', sans-serif;
  letter-spacing: -1.2px;
  overflow-wrap: anywhere;
}

.stat-money {
  font-size: clamp(18px, 1.6vw, 25px);
  letter-spacing: -.8px;
}

.stat-bottom {
  display: flex;
  align-items: center;
  gap: 7px;
  color: #97a1b0;
  font-size: 10px;
}

.stat-mini-dot,
.legend-dot {
  width: 7px;
  height: 7px;
  flex: 0 0 auto;
  border-radius: 50%;
}

.blue-dot { background: #5278f0; }
.amber-dot { background: #e8a73c; }
.green-dot { background: #20a879; }
.purple-dot { background: #9570e6; }
.red-dot { background: #e36c77; }

.overview-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.85fr) minmax(255px, .9fr);
  gap: 19px;
  align-items: stretch;
}

.panel {
  min-width: 0;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  background: white;
  box-shadow: 0 3px 12px rgba(20, 34, 56, .018);
}

.orders-panel,
.status-panel {
  overflow: hidden;
}

.panel-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 15px;
  padding: 22px 22px 18px;
}

.panel-heading h3,
.management-toolbar h3 {
  margin: 0;
  color: #1c2a40;
  font: 800 14px/1.4 'Manrope', sans-serif;
  letter-spacing: -.3px;
}

.panel-heading p,
.management-toolbar p {
  margin: 5px 0 0;
  color: #929daf;
  font-size: 10px;
  line-height: 1.6;
}

.text-button {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 7px 0;
  border: 0;
  color: var(--blue);
  background: transparent;
  font-size: 10px;
  font-weight: 800;
  white-space: nowrap;
}

.text-button:hover {
  color: var(--blue-dark);
}

.text-button span {
  font-size: 15px;
}

.table-scroll {
  width: 100%;
  overflow-x: auto;
}

.orders-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
}

.orders-table thead {
  background: #f9fafd;
}

.orders-table th {
  padding: 12px 17px;
  border-top: 1px solid #f0f2f7;
  border-bottom: 1px solid #edf0f5;
  color: #929daf;
  font-size: 8px;
  font-weight: 800;
  letter-spacing: .8px;
  white-space: nowrap;
}

.orders-table td {
  padding: 14px 17px;
  border-bottom: 1px solid #f0f2f7;
  color: #536177;
  font-size: 10px;
  vertical-align: middle;
}

.orders-table tbody tr {
  transition: background .15s;
}

.orders-table tbody tr:hover {
  background: #fafbff;
}

.orders-table tbody tr:last-child td {
  border-bottom: 0;
}

.order-id-button {
  display: block;
  padding: 0;
  border: 0;
  color: #3563db;
  background: transparent;
  font-size: 10px;
  font-weight: 800;
  white-space: nowrap;
}

.order-id-button:hover {
  text-decoration: underline;
}

.cell-subtext {
  display: block;
  margin-top: 5px;
  color: #9ba5b4;
  font-size: 9px;
  white-space: nowrap;
}

.customer-cell {
  display: flex;
  align-items: center;
  gap: 9px;
  min-width: 135px;
}

.customer-avatar {
  width: 31px;
  height: 31px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border: 1px solid #e2e9fb;
  border-radius: 10px;
  color: #4666c4;
  background: #edf2ff;
  font-size: 10px;
  font-weight: 800;
}

.customer-cell-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.customer-cell-info strong {
  max-width: 150px;
  overflow: hidden;
  color: #334158;
  font-size: 10px;
  font-weight: 700;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.customer-cell-info span {
  overflow: hidden;
  color: #9ba5b4;
  font-size: 9px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.date-cell {
  color: #7e899b !important;
  white-space: nowrap;
}

.amount-cell {
  color: #283950 !important;
  font-weight: 800;
  white-space: nowrap;
}

.status-badge {
  width: fit-content;
  max-width: 190px;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 8px;
  border: 1px solid transparent;
  border-radius: 7px;
  font-size: 9px;
  font-weight: 700;
  line-height: 1.25;
  white-space: nowrap;
}

.status-dot {
  width: 6px;
  height: 6px;
  flex: 0 0 auto;
  border-radius: 50%;
  background: currentColor;
}

.status-amber { color: #a66a0c; background: #fff6e5; border-color: #ffebc4; }
.status-blue { color: #315fce; background: #edf3ff; border-color: #dce7ff; }
.status-purple { color: #7950bf; background: #f4efff; border-color: #e9ddff; }
.status-cyan { color: #087e99; background: #e8f9fd; border-color: #d2f1f8; }
.status-indigo { color: #4f53bf; background: #efefff; border-color: #dfdfff; }
.status-green { color: #148052; background: #e8f8f0; border-color: #d0f0df; }
.status-red { color: #bb3e4d; background: #fff0f1; border-color: #ffdcdf; }
.status-gray { color: #69768a; background: #f0f2f6; border-color: #e4e8ee; }

.panel-icon {
  width: 34px;
  height: 34px;
  display: grid;
  place-items: center;
  border-radius: 10px;
  color: #5073db;
  background: #eef2ff;
}

.panel-icon svg {
  width: 19px;
  height: 19px;
}

.snapshot-total {
  display: flex;
  align-items: baseline;
  gap: 9px;
  padding: 4px 22px 19px;
}

.snapshot-total strong {
  color: #1b2940;
  font: 800 31px/1 'Manrope', sans-serif;
  letter-spacing: -1px;
}

.snapshot-total span {
  color: #919bad;
  font-size: 10px;
}

.snapshot-bar {
  display: flex;
  gap: 3px;
  height: 8px;
  margin: 0 22px 22px;
  overflow: hidden;
  border-radius: 9px;
  background: #eef1f6;
}

.snapshot-bar span {
  min-width: 0;
  border-radius: 9px;
  transition: width .3s;
}

.bar-delivered { background: #20a879; }
.bar-active { background: #e9b04d; }
.bar-cancelled { background: #e36c77; }

.snapshot-legend {
  display: flex;
  flex-direction: column;
  padding: 0 22px;
}

.snapshot-legend button {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 13px 0;
  border: 0;
  border-bottom: 1px solid #f0f2f7;
  color: #68768a;
  background: transparent;
  text-align: left;
  font-size: 10px;
}

.snapshot-legend button:hover span:nth-child(2) {
  color: var(--blue);
}

.snapshot-legend strong {
  margin-left: auto;
  color: #283950;
  font-size: 11px;
}

.snapshot-footer {
  display: flex;
  gap: 10px;
  margin: 18px 16px 16px;
  padding: 13px;
  border: 1px solid #e7edff;
  border-radius: 10px;
  background: #f5f7ff;
}

.snapshot-footer-icon {
  width: 27px;
  height: 27px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border-radius: 8px;
  color: #3563eb;
  background: #e7edff;
  font-weight: 800;
}

.snapshot-footer strong {
  color: #354b76;
  font-size: 10px;
}

.snapshot-footer p {
  margin: 5px 0 0;
  color: #8794ad;
  font-size: 9px;
  line-height: 1.6;
}

.bottom-banner {
  position: relative;
  display: flex;
  align-items: center;
  gap: 17px;
  margin-top: 22px;
  padding: 23px 25px;
  overflow: hidden;
  border: 1px solid #e0e7fa;
  border-radius: var(--radius);
  background:
    radial-gradient(ellipse at 85% 5%, rgba(103, 139, 239, .13), transparent 38%),
    linear-gradient(110deg, #fff, #f5f7ff);
}

.banner-mark {
  width: 49px;
  height: 49px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border-radius: 15px;
  color: #3563eb;
  background: #e9efff;
}

.banner-mark svg {
  width: 38px;
  height: 38px;
}

.banner-copy {
  min-width: 0;
}

.banner-copy > span {
  color: #6980c3;
  font-size: 8px;
  font-weight: 800;
  letter-spacing: 1.5px;
}

.banner-copy h3 {
  margin: 5px 0 4px;
  color: #1d2c47;
  font: 800 17px/1.35 'Manrope', sans-serif;
  letter-spacing: -.5px;
}

.banner-copy p {
  margin: 0;
  color: #8b96a8;
  font-size: 10px;
  line-height: 1.6;
}

.banner-button {
  display: inline-flex;
  align-items: center;
  gap: 14px;
  margin-left: auto;
  padding: 12px 14px;
  border: 1px solid #dfe6f8;
  border-radius: 9px;
  color: #3858b1;
  background: #fff;
  font-size: 10px;
  font-weight: 800;
  white-space: nowrap;
  transition: border-color .2s, background .2s;
}

.banner-button:hover {
  border-color: #b9c9f7;
  background: #f7f9ff;
}

.banner-button span {
  font-size: 15px;
}

.loading-state {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 11px;
  min-height: 190px;
  color: #8792a5;
  font-size: 12px;
}

.spinner {
  width: 18px;
  height: 18px;
  border: 2px solid #dce5ff;
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: spin .8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 235px;
  padding: 25px;
  text-align: center;
}

.empty-icon {
  width: 47px;
  height: 47px;
  display: grid;
  place-items: center;
  margin-bottom: 13px;
  border-radius: 14px;
  color: #6c86d8;
  background: #eef2ff;
}

.empty-icon svg {
  width: 25px;
  height: 25px;
}

.empty-state > strong {
  color: #334158;
  font-size: 13px;
}

.empty-state p {
  margin: 7px 0 0;
  color: #909bad;
  font-size: 11px;
}

.page-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 23px 2px 3px;
  color: #9aa4b4;
  font-size: 10px;
}

.page-footer strong {
  color: #6d7a8e;
}

.footer-heart {
  margin-left: 4px;
  color: #6481df;
}

.dashboard-alert {
  margin-bottom: 20px;
}

.alert {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 11px 13px;
  border: 1px solid transparent;
  border-radius: 9px;
  font-size: 11px;
  line-height: 1.6;
}

.alert-error {
  border-color: #ffd9dc;
  color: #a82e3b;
  background: #fff1f2;
}

.alert button {
  margin-left: auto;
  padding: 0 4px;
  border: 0;
  color: inherit;
  background: transparent;
  font-size: 19px;
}

.orders-count-chip {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 11px 15px;
  border: 1px solid #e0e7fa;
  border-radius: 11px;
  color: #7182b7;
  background: #fff;
  font-size: 10px;
}

.orders-count-chip strong {
  color: #3159cb;
  font: 800 21px/1 'Manrope', sans-serif;
}

.order-summary-strip {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
  margin-bottom: 20px;
}

.order-summary-strip button {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 15px 17px;
  border: 1px solid var(--border);
  border-radius: 11px;
  color: #778398;
  background: white;
  text-align: left;
  font-size: 11px;
  transition: border-color .2s, background .2s;
}

.order-summary-strip button:hover,
.order-summary-strip .summary-filter-active {
  border-color: #cbd8ff;
  background: #f6f8ff;
}

.order-summary-strip .summary-filter-active {
  box-shadow: inset 3px 0 var(--blue);
}

.order-summary-strip strong {
  color: #283950;
  font: 800 18px/1 'Manrope', sans-serif;
}

.management-panel {
  overflow: hidden;
}

.management-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 18px;
  padding: 22px;
}

.toolbar-controls {
  display: flex;
  align-items: center;
  gap: 9px;
}

.search-box {
  min-width: 200px;
  height: 37px;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 10px;
  border: 1px solid #e3e8f1;
  border-radius: 8px;
  background: #fff;
}

.search-box:focus-within {
  border-color: #9bb2ff;
  box-shadow: 0 0 0 3px rgba(53, 99, 235, .07);
}

.search-box svg {
  width: 16px;
  height: 16px;
  flex: 0 0 auto;
  color: #97a2b4;
}

.search-box input {
  width: 100%;
  min-width: 0;
  border: 0;
  outline: none;
  color: #334158;
  background: transparent;
  font-size: 10px;
}

.search-box input::placeholder {
  color: #a1aaba;
}

.filter-select {
  height: 37px;
  max-width: 155px;
  padding: 0 26px 0 10px;
  border: 1px solid #e3e8f1;
  border-radius: 8px;
  outline-offset: 2px;
  color: #637087;
  background: white;
  font-size: 10px;
}

.management-table {
  min-width: 1000px;
}

.management-table th,
.management-table td {
  padding: 14px 15px;
}

.table-order-number {
  display: block;
  color: #3563db;
  font-size: 10px;
  white-space: nowrap;
}

.payment-label {
  display: inline-block;
  padding: 6px 8px;
  border: 1px solid #e7ebf2;
  border-radius: 6px;
  color: #66758b;
  background: #fafbfc;
  font-size: 9px;
  font-weight: 700;
  white-space: nowrap;
}

.status-select {
  max-width: 155px;
  min-width: 125px;
  height: 32px;
  padding: 0 7px;
  border: 1px solid #e1e6ef;
  border-radius: 7px;
  outline-offset: 2px;
  background: white;
  font-size: 10px;
  font-weight: 600;
}

.select-amber { color: #9d680f; border-color: #f4dfb8; background: #fffaf0; }
.select-blue { color: #315fce; border-color: #d8e4ff; background: #f7f9ff; }
.select-purple { color: #7950bf; border-color: #e4d8ff; background: #fbf9ff; }
.select-cyan { color: #087e99; border-color: #d2edf5; background: #f4fcff; }
.select-indigo { color: #4f53bf; border-color: #dfdfff; background: #f8f8ff; }
.select-green { color: #148052; border-color: #d0f0df; background: #f5fcf8; }
.select-red { color: #bb3e4d; border-color: #ffdcdf; background: #fff8f8; }
.select-gray { color: #69768a; border-color: #e4e8ee; background: #f9fafc; }

.updating-label {
  display: block;
  margin-top: 5px;
  color: #8a96a8;
  font-size: 9px;
}

.table-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 15px;
  padding: 15px 21px;
  border-top: 1px solid #edf0f5;
  color: #929daf;
  background: #fcfcfe;
  font-size: 10px;
}

.table-footer strong {
  color: #526078;
}

.table-live {
  display: flex;
  align-items: center;
  gap: 8px;
}

.table-live span {
  width: 6px;
  height: 6px;
}

.management-empty {
  min-height: 270px;
}

.order-details-panel {
  margin-top: 20px;
  overflow: hidden;
}

.detail-order-list {
  border-top: 1px solid #edf0f5;
}

.order-detail {
  border-bottom: 1px solid #edf0f5;
}

.order-detail:last-child {
  border-bottom: 0;
}

.order-detail summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 18px;
  padding: 16px 22px;
  cursor: pointer;
  list-style: none;
  transition: background .15s;
}

.order-detail summary::-webkit-details-marker {
  display: none;
}

.order-detail summary:hover {
  background: #fafbff;
}

.order-detail summary > div:first-child {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.order-detail summary > div:first-child strong {
  color: #334158;
  font-size: 11px;
}

.order-detail summary > div:first-child span {
  color: #929daf;
  font-size: 10px;
}

.detail-summary-right {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 14px;
}

.detail-summary-right > strong {
  color: #273750;
  font-size: 11px;
  white-space: nowrap;
}

.detail-content {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr) minmax(0, .9fr);
  gap: 20px;
  padding: 22px;
  border-top: 1px solid #edf0f5;
  background: #fbfcfe;
}

.detail-section {
  min-width: 0;
}

.detail-section h4 {
  margin: 0 0 15px;
  color: #334158;
  font: 800 11px/1.4 'Manrope', sans-serif;
}

.detail-section > p {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  margin: 0;
  padding: 9px 0;
  border-bottom: 1px solid #edf0f5;
  font-size: 10px;
}

.detail-section > p span {
  flex: 0 0 auto;
  color: #8995a7;
}

.detail-section > p strong {
  color: #47566c;
  text-align: right;
  overflow-wrap: anywhere;
}

.detail-section > .detail-total {
  margin-top: 4px;
  border-bottom: 0;
}

.detail-section > .detail-total span,
.detail-section > .detail-total strong {
  color: #20334e;
  font-weight: 800;
}

.detail-item {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 11px 0;
  border-bottom: 1px solid #edf0f5;
}

.detail-item:first-of-type {
  padding-top: 0;
}

.detail-item-icon {
  width: 31px;
  height: 31px;
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  border: 1px solid #e1e8fb;
  border-radius: 9px;
  color: #5576d9;
  background: #eff3ff;
}

.detail-item-icon svg {
  width: 19px;
  height: 19px;
}

.detail-item-copy {
  display: flex;
  flex: 1;
  flex-direction: column;
  gap: 5px;
  min-width: 0;
}

.detail-item-copy strong {
  color: #47566c;
  font-size: 10px;
  line-height: 1.5;
  overflow-wrap: anywhere;
}

.detail-item-copy span {
  color: #929daf;
  font-size: 9px;
  line-height: 1.5;
}

.detail-item-price {
  color: #34435a;
  font-size: 10px;
  white-space: nowrap;
}

.detail-status-label {
  display: block;
  margin: 15px 0 7px;
  color: #7c899d;
  font-size: 10px;
  font-weight: 700;
}

.detail-status-select {
  width: 100%;
  max-width: none;
}

.detail-no-orders {
  padding: 22px;
  color: #929daf;
  font-size: 11px;
}

.mobile-overlay {
  display: none;
}

@media (max-width: 1350px) {
  .page-content {
    padding-right: 24px;
    padding-left: 24px;
  }

  .topbar {
    padding-right: 24px;
    padding-left: 24px;
  }

  .stats-grid {
    gap: 12px;
  }

  .stat-card {
    padding: 16px;
  }

  .stat-value {
    font-size: 25px;
  }

  .stat-money {
    font-size: 19px;
  }

  .overview-grid {
    grid-template-columns: minmax(0, 1.65fr) minmax(240px, .85fr);
    gap: 14px;
  }

  .management-toolbar {
    align-items: flex-start;
    flex-direction: column;
  }

  .toolbar-controls {
    width: 100%;
    flex-wrap: wrap;
  }

  .search-box {
    flex: 1;
  }
}

@media (max-width: 1050px) {
  .sidebar {
    width: 225px;
  }

  .main-content {
    margin-left: 225px;
  }

  .stats-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .overview-grid {
    grid-template-columns: 1fr;
  }

  .status-panel {
    min-width: 0;
  }

  .snapshot-legend {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 12px;
  }

  .snapshot-legend button {
    gap: 6px;
  }

  .snapshot-footer {
    margin-top: 13px;
  }

  .detail-content {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .detail-section:last-child {
    grid-column: 1 / -1;
  }

  .detail-section:last-child > p {
    max-width: 420px;
  }

  .detail-status-select {
    max-width: 260px;
  }
}

@media (max-width: 760px) {
  .sidebar {
    width: 270px;
    max-width: 86vw;
    transform: translateX(-105%);
    transition: transform .25s ease;
    box-shadow: 10px 0 40px rgba(0, 0, 0, .16);
  }

  .sidebar.sidebar-open {
    transform: translateX(0);
  }

  .sidebar-close {
    display: block;
  }

  .mobile-overlay {
    position: fixed;
    inset: 0;
    z-index: 19;
    display: block;
    background: rgba(12, 23, 40, .48);
    backdrop-filter: blur(2px);
  }

  .main-content {
    margin-left: 0;
  }

  .topbar {
    height: 72px;
    padding: 0 18px;
  }

  .mobile-menu-button {
    display: grid;
    flex: 0 0 auto;
  }

  .topbar-title {
    font-size: 16px;
  }

  .topbar-right {
    gap: 11px;
  }

  .topbar-divider,
  .topbar-date {
    display: none;
  }

  .page-content {
    padding: 25px 18px 18px;
  }

  .welcome-row {
    align-items: flex-start;
    flex-direction: column;
    gap: 15px;
    margin-bottom: 20px;
  }

  .welcome-row .primary-button {
    align-self: flex-start;
  }

  .welcome-row h1 {
    font-size: 25px;
  }

  .stats-grid {
    gap: 11px;
    margin-bottom: 16px;
  }

  .stat-card {
    padding: 14px;
    border-radius: 12px;
  }

  .stat-label {
    font-size: 10px;
  }

  .stat-icon {
    width: 31px;
    height: 31px;
    border-radius: 9px;
  }

  .stat-icon svg {
    width: 17px;
    height: 17px;
  }

  .stat-value {
    margin: 15px 0 11px;
    font-size: 25px;
  }

  .stat-money {
    font-size: clamp(16px, 4.1vw, 21px);
  }

  .stat-bottom {
    align-items: flex-start;
    font-size: 9px;
    line-height: 1.5;
  }

  .panel-heading {
    padding: 18px 16px 15px;
  }

  .orders-panel .table-scroll {
    overflow-x: auto;
  }

  .orders-panel .orders-table {
    min-width: 650px;
  }

  .bottom-banner {
    align-items: flex-start;
    flex-wrap: wrap;
    padding: 19px;
    gap: 12px;
  }

  .banner-mark {
    width: 40px;
    height: 40px;
    border-radius: 12px;
  }

  .banner-mark svg {
    width: 32px;
    height: 32px;
  }

  .banner-copy {
    flex: 1;
    min-width: 170px;
  }

  .banner-copy h3 {
    font-size: 15px;
  }

  .banner-button {
    width: 100%;
    justify-content: space-between;
    margin: 3px 0 0;
  }

  .page-footer {
    align-items: flex-start;
    flex-direction: column;
    gap: 7px;
    padding-top: 19px;
  }

  .order-summary-strip {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 9px;
  }

  .order-summary-strip button {
    padding: 13px;
  }

  .management-toolbar {
    padding: 17px;
  }

  .toolbar-controls {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }

  .search-box {
    grid-column: 1 / -1;
    min-width: 0;
  }

  .filter-select {
    width: 100%;
    max-width: none;
  }

  .table-footer {
    align-items: flex-start;
    flex-direction: column;
    gap: 8px;
    padding: 14px 17px;
  }

  .detail-content {
    grid-template-columns: 1fr;
    gap: 25px;
    padding: 17px;
  }

  .detail-section:last-child {
    grid-column: auto;
  }

  .detail-section:last-child > p {
    max-width: none;
  }

  .detail-status-select {
    max-width: none;
  }

  .order-detail summary {
    align-items: flex-start;
    padding: 15px 16px;
  }

  .detail-summary-right {
    align-items: flex-end;
    flex-direction: column;
    gap: 6px;
  }

  .detail-summary-right .status-badge {
    white-space: normal;
  }

  .snapshot-legend {
    grid-template-columns: 1fr;
    gap: 0;
  }
}

@media (max-width: 420px) {
  .login-card {
    padding: 27px 22px;
  }

  .login-heading {
    margin-top: 32px;
  }

  .topbar {
    padding: 0 12px;
  }

  .topbar-title {
    font-size: 14px;
  }

  .live-indicator {
    font-size: 0;
    gap: 0;
  }

  .live-indicator span {
    width: 8px;
    height: 8px;
  }

  .topbar-avatar {
    width: 31px;
    height: 31px;
  }

  .page-content {
    padding-right: 12px;
    padding-left: 12px;
  }

  .welcome-row h1 {
    font-size: 22px;
  }

  .stats-grid {
    gap: 8px;
  }

  .stat-card {
    padding: 12px 10px;
  }

  .stat-label {
    font-size: 9px;
  }

  .stat-value {
    font-size: 23px;
  }

  .stat-money {
    font-size: 16px;
  }

  .stat-bottom {
    font-size: 8px;
  }

  .panel-heading {
    padding-right: 13px;
    padding-left: 13px;
  }

  .panel-heading h3 {
    font-size: 13px;
  }

  .text-button {
    font-size: 9px;
  }

  .orders-count-chip {
    padding: 9px 12px;
  }

  .orders-count-chip strong {
    font-size: 18px;
  }

  .order-summary-strip button {
    padding: 11px;
    font-size: 10px;
  }

  .order-summary-strip strong {
    font-size: 16px;
  }

  .management-toolbar {
    padding: 15px 12px;
  }

  .toolbar-controls {
    grid-template-columns: 1fr;
  }

  .search-box {
    grid-column: auto;
  }

  .filter-select {
    height: 39px;
  }

  .detail-summary-right > strong {
    font-size: 10px;
  }
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    scroll-behavior: auto !important;
    animation-duration: .01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: .01ms !important;
  }
}
</style>
