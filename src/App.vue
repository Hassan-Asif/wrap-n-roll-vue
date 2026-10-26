<script setup>
import { computed, onMounted, onUnmounted, reactive, ref, watch } from 'vue'
import { addDoc, collection, serverTimestamp } from 'firebase/firestore'
import AdminPanel from './components/AdminPanel.vue'
import { db } from './firebase'


const CONFIG = { whatsappNumber: '+923361271119', storeName: 'Wrap N Roll', deliveryCity: 'Karachi', deliveryCharge: 300 }
const PRODUCTS = [
  { id:'bw-100', name:'Bubble Wrap 100 FT', category:'Bubble Wrap', price:2800, image:'🎁', description:'High-quality protective bubble wrap roll', details:'Length: 100 FT', variants:{type:'simple'} },
  { id:'bw-50', name:'Bubble Wrap 50 FT', category:'Bubble Wrap', price:1600, image:'🎁', description:'Perfect for small to medium packaging', details:'Length: 50 FT', variants:{type:'simple'} },
  { id:'ts-base', name:'Thermocol Sheet', category:'Thermocol Sheets', basePrice:1800, image:'📦', description:'Lightweight and durable thermocol sheets', variants:{type:'multi',sizes:['2 × 4 ft','4 × 8 ft'],thicknesses:['1"','2"','3"','4"'],priceModifier:{'1"':0,'2"':300,'3"':600,'4"':900}} },
  { id:'wtb-5kg', name:'White Thermocol Box 5 KG', category:'Thermocol Boxes', price:280, image:'📫', description:'Perfect for small shipments', details:'Capacity: 5 KG', variants:{type:'simple'} },
  { id:'wtb-10kg', name:'White Thermocol Box 10 KG', category:'Thermocol Boxes', price:420, image:'📫', description:'Ideal for medium packages', details:'Capacity: 10 KG', variants:{type:'simple'} },
  { id:'wtb-15kg', name:'White Thermocol Box 15 KG', category:'Thermocol Boxes', price:600, image:'📫', description:'Great for larger shipments', details:'Capacity: 15 KG', variants:{type:'simple'} },
  { id:'wtb-20kg', name:'White Thermocol Box 20 KG', category:'Thermocol Boxes', price:780, image:'📫', description:'Maximum capacity white box', details:'Capacity: 20 KG', variants:{type:'simple'} },
  { id:'btb-20kg', name:'Black Thermocol Box 20 KG', category:'Thermocol Boxes', price:1350, image:'⬛', description:'Premium black thermocol box', details:'Capacity: 20 KG', variants:{type:'simple'} },
  { id:'xps-base', name:'XPS Sheet', category:'XPS Sheets', basePrice:2200, image:'💗', description:'High-insulation pink XPS board', variants:{type:'multi',sizes:['2 × 4 ft','4 × 8 ft'],thicknesses:['1"','2"','3"','4"'],priceModifier:{'1"':0,'2"':400,'3"':800,'4"':1200}} }
]
const categories = ['Bubble Wrap','Thermocol Sheets','Thermocol Boxes','XPS Sheets']
const categoryData = {
  'Bubble Wrap':{emoji:'🎁',desc:'Keep your items safe with high-quality bubble wrap.'},
  'Thermocol Sheets':{emoji:'📦',desc:'Lightweight, durable and perfect for all your packaging needs.'},
  'Thermocol Boxes':{emoji:'📫',desc:'Available in multiple sizes for safe and secure packaging.'},
  'XPS Sheets':{emoji:'💗',desc:'High insulation, strong and built for protection.'}
}
const whatsapp = `https://wa.me/${CONFIG.whatsappNumber.replace('+','')}`
const cartItems = ref([])
const currentPage = ref('home')
const selectedCategory = ref('')
const selectedProduct = ref(null)
const orderData = ref(null)
const toast = ref('')
const menuOpen = ref(false)
const searchQuery = ref('')
const productSelections = reactive({})
const checkout = reactive({fullName:'',phoneNumber:'',whatsappNumber:'',address:'',area:'',landmark:'',city:'Karachi',paymentMethod:'COD'})
const errors = ref({})
const isSubmitting = ref(false)
const orderError = ref('')
let toastTimer

const subtotal = computed(() => cartItems.value.reduce((sum,item)=>sum + item.finalPrice*item.quantity,0))
const total = computed(() => subtotal.value + CONFIG.deliveryCharge)
const categoryProducts = computed(() => PRODUCTS.filter(p=>p.category===selectedCategory.value))
const filteredProducts = computed(() => PRODUCTS.filter(p => `${p.name} ${p.category} ${p.description}`.toLowerCase().includes(searchQuery.value.toLowerCase())))
function selection(product) {
  if (!productSelections[product.id]) productSelections[product.id] = { size:product.variants?.sizes?.[0] || '', thickness:product.variants?.thicknesses?.[0] || '', quantity:1 }
  return productSelections[product.id]
}
function priceFor(product, selected=selection(product)) {
  return product.variants?.type === 'multi' ? product.basePrice + (product.variants.priceModifier[selected.thickness] || 0) : product.price
}
function navigate(hash) { window.location.hash = hash }

function routeFromHash() {
  const path = window.location.pathname
  const hash = decodeURIComponent(window.location.hash.slice(1))

  // Support direct URLs such as /admin
  if (path === '/admin' || path === '/admin/') {
    currentPage.value = 'admin'
  } else if (!hash || hash === '/') {
    currentPage.value = 'home'
  } else if (hash === 'admin') {
    currentPage.value = 'admin'
  } else if (hash === 'shop') {
    currentPage.value = 'shop'
  } else if (hash === 'cart') {
    currentPage.value = 'cart'
  } else if (hash === 'checkout') {
    currentPage.value = 'checkout'
  } else if (hash === 'contact') {
    currentPage.value = 'home'
    document.getElementById('contact')?.scrollIntoView({
      behavior: 'smooth'
    })
  } else if (hash.startsWith('category/')) {
    selectedCategory.value = hash.slice(9)
    currentPage.value = 'category'
  } else if (hash === 'product') {
    currentPage.value = selectedProduct.value
      ? 'product-detail'
      : 'shop'
  }

  menuOpen.value = false
}

function addToCart(product, chosen=null, qty=null) {
  const sel = chosen || selection(product)
  const item = {...product, selectedVariants:{size:sel.size || '', thickness:sel.thickness || ''}, quantity:qty ?? sel.quantity ?? 1, finalPrice:priceFor(product,sel)}
  const key = itemKey(item)
  const existing = cartItems.value.find(i=>itemKey(i)===key)
  if(existing) existing.quantity += item.quantity
  else cartItems.value.push(item)
  toast.value='✅ Added to cart!'
  clearTimeout(toastTimer); toastTimer=setTimeout(()=>toast.value='',3000)
}
function itemKey(item) { return `${item.id}-${JSON.stringify(item.selectedVariants || {})}` }
function removeFromCart(id, variants) { cartItems.value=cartItems.value.filter(i=>itemKey(i)!==`${id}-${JSON.stringify(variants || {})}`) }
function updateQuantity(id, variants, qty) {
  const item=cartItems.value.find(i=>itemKey(i)===`${id}-${JSON.stringify(variants || {})}`)
  if(item) item.quantity=Math.max(1,Number(qty)||1)
}
function openCategory(category) { selectedCategory.value=category; navigate(`#category/${encodeURIComponent(category)}`) }
function openProduct(product) { selectedProduct.value=product; currentPage.value='product-detail'; navigate('#product') }
function buyNow(product) { addToCart(product); navigate('#checkout') }

async function placeOrder() {
  if (isSubmitting.value) return

  const e = {}

  for (const field of [
    'fullName',
    'phoneNumber',
    'whatsappNumber',
    'address',
    'area',
  ]) {
    if (!checkout[field].trim()) {
      e[field] = true
    }
  }

  if (checkout.city !== 'Karachi') {
    e.city = true
  }

  errors.value = e
  orderError.value = ''

  if (Object.keys(e).length > 0) return

  if (!cartItems.value.length) {
    orderError.value = 'Your cart is empty. Please add a product first.'
    return
  }

  isSubmitting.value = true

  const orderNumber = Math.random()
    .toString(36)
    .slice(2, 8)
    .toUpperCase()

  const items = cartItems.value.map((item) => ({
    productId: item.id,
    name: item.name,
    category: item.category,
    unitPrice: item.finalPrice,
    quantity: item.quantity,
    selectedVariants: item.selectedVariants || {},
  }))

  const order = {
    orderNumber,
    customer: {
      fullName: checkout.fullName.trim(),
      phoneNumber: checkout.phoneNumber.trim(),
      whatsappNumber: checkout.whatsappNumber.trim(),
      address: checkout.address.trim(),
      area: checkout.area.trim(),
      landmark: checkout.landmark.trim(),
      city: checkout.city,
    },
    items,
    subtotal: subtotal.value,
    deliveryCharge: CONFIG.deliveryCharge,
    total: total.value,
    paymentMethod: checkout.paymentMethod,
    status: 'pending',
    createdAt: serverTimestamp(),
  }

  try {
    const docRef = await addDoc(collection(db, 'orders'), order)

    // Show confirmation only after Firestore saves the order.
    orderData.value = {
      ...checkout,
      total: order.total,
      items: cartItems.value.map((item) => ({ ...item })),
      number: orderNumber,
      firestoreId: docRef.id,
    }

    cartItems.value = []
    currentPage.value = 'confirmation'
    window.location.hash = 'confirmation'
  } catch (error) {
    console.error('Failed to save order:', error)

    orderError.value =
      'We could not save your order. Please check your connection and try again.'
  } finally {
    isSubmitting.value = false
  }
}

watch(cartItems, v=>{ try{localStorage.setItem('cart',JSON.stringify(v))}catch{} },{deep:true})
onMounted(()=>{
  try { cartItems.value=JSON.parse(localStorage.getItem('cart')) || [] } catch { cartItems.value=[] }
  window.addEventListener('hashchange',routeFromHash); routeFromHash()
})
onUnmounted(()=>{window.removeEventListener('hashchange',routeFromHash);clearTimeout(toastTimer)})
</script>

<template>
  <div class="app-shell">
    <AdminPanel v-if="currentPage === 'admin'" />
    <nav v-if="currentPage !== 'admin'" class="navbar">
      <div class="nav-content">
        <a href="#" class="logo">📦 WRAP N ROLL</a>
        <button class="menu-toggle" aria-label="Toggle menu" @click="menuOpen=!menuOpen"><span></span><span></span><span></span></button>
        <div :class="['nav-links',{active:menuOpen}]">
          <a href="#" @click="menuOpen=false">Home</a><a href="#shop" @click="menuOpen=false">Shop</a>
          <a href="#category/Bubble%20Wrap" @click="menuOpen=false">Bubble Wrap</a><a href="#category/Thermocol%20Sheets" @click="menuOpen=false">Thermocol</a>
          <a href="#category/XPS%20Sheets" @click="menuOpen=false">XPS Sheets</a>
        </div>
        <div class="nav-icons"><button class="nav-icon-btn" style="position:relative" @click="navigate('#cart')">🛒<span v-if="cartItems.length" class="cart-count">{{cartItems.length}}</span></button></div>
      </div>
    </nav>

    <template v-if="currentPage==='home'">
      <section class="hero"><div class="hero-content">
        <div class="delivery-badge">✓ Delivery Available Across Karachi</div>
        <h1>Premium Packaging Materials, Delivered <span class="hero-highlight">Across Karachi</span></h1>
        <p>Bubble wrap, thermocol sheets, boxes and XPS insulation — everything you need to pack and protect.</p>
        <div class="hero-buttons"><button class="btn btn-primary" @click="navigate('#shop')">🛍️ Shop Now</button><a class="btn btn-secondary" :href="`${whatsapp}?text=Hello%20Wrap%20N%20Roll%2C%20I%20want%20to%20know%20more%20about%20your%20packaging%20products.`" target="_blank">💬 WhatsApp Us</a></div>
      </div></section>
      <section class="section"><div class="container"><h2 class="section-title">Our Products</h2><div class="grid grid-4"><div v-for="cat in categories" :key="cat" :class="['category-card',{blue:cat==='Thermocol Boxes'||cat==='Bubble Wrap'}]" @click="openCategory(cat)"><div class="category-image">{{categoryData[cat].emoji}}</div><div class="category-content"><h3 class="category-name">{{cat}}</h3><p class="category-desc">{{categoryData[cat].desc}}</p><button class="btn btn-primary" style="margin-top:1rem;width:100%">Shop Now →</button></div></div></div></div></section>
      <section class="section" style="background:var(--light-gray)"><div class="container"><h2 class="section-title">Why Choose Wrap N Roll?</h2><div class="grid grid-3"><div class="benefit"><div>🚚</div><h3>Fast Delivery</h3><p>Deliver your orders across Karachi quickly</p></div><div class="benefit"><div>✅</div><h3>Premium Quality</h3><p>High-quality packaging materials</p></div><div class="benefit"><div>💬</div><h3>Expert Support</h3><p>Chat with us on WhatsApp anytime</p></div></div></div></section>
    </template>

    <template v-else-if="currentPage==='shop'">
      <div class="shop-header"><h1>Shop All Products</h1><p style="color:#666">Browse our complete range of packaging materials</p></div>
      <section class="products-container"><div class="search-bar"><span>🔎</span><input v-model="searchQuery" placeholder="Search packaging products..." /></div><div class="grid grid-3"><div v-for="p in filteredProducts" :key="p.id" class="product-card"><div class="product-image" @click="openProduct(p)">{{p.image}}</div><div class="product-info"><h3 class="product-name">{{p.name}}</h3><p class="product-desc">{{p.description}}</p><div v-if="p.variants.type==='multi'" class="variant-section"><label class="variant-label">Size</label><div class="variant-options"><button v-for="size in p.variants.sizes" :key="size" :class="['variant-option',{active:selection(p).size===size}]" @click="selection(p).size=size">{{size}}</button></div><label class="variant-label">Thickness</label><div class="variant-options"><button v-for="th in p.variants.thicknesses" :key="th" :class="['variant-option',{active:selection(p).thickness===th}]" @click="selection(p).thickness=th">{{th}}</button></div></div><div class="product-price">PKR {{priceFor(p).toLocaleString()}}</div><div class="quantity-selector"><button class="qty-btn" @click="selection(p).quantity=Math.max(1,selection(p).quantity-1)">−</button><input class="qty-input" type="number" min="1" v-model.number="selection(p).quantity"><button class="qty-btn" @click="selection(p).quantity++">+</button></div><div class="product-actions"><button class="btn-add-cart" @click="addToCart(p)">🛒 Add to Cart</button><button class="btn-buy-now" @click="buyNow(p)">Buy Now</button></div></div></div></div></section>
    </template>

    <section v-else-if="currentPage==='category'" class="section"><div class="container"><h1 class="section-title">{{selectedCategory}}</h1><div v-if="!categoryProducts.length" class="empty-state"><p>No products found</p></div><div v-else class="grid grid-3"><div v-for="p in categoryProducts" :key="p.id" class="product-card"><div class="product-image" @click="openProduct(p)">{{p.image}}</div><div class="product-info"><h3 class="product-name">{{p.name}}</h3><p class="product-desc">{{p.description}}</p><p v-if="p.details" class="product-desc">{{p.details}}</p><div v-if="p.variants.type==='multi'" class="variant-section"><label class="variant-label">Size</label><div class="variant-options"><button v-for="size in p.variants.sizes" :key="size" :class="['variant-option',{active:selection(p).size===size}]" @click="selection(p).size=size">{{size}}</button></div><label class="variant-label">Thickness</label><div class="variant-options"><button v-for="th in p.variants.thicknesses" :key="th" :class="['variant-option',{active:selection(p).thickness===th}]" @click="selection(p).thickness=th">{{th}}</button></div></div><div class="product-price">PKR {{priceFor(p).toLocaleString()}}</div><div class="quantity-selector"><button class="qty-btn" @click="selection(p).quantity=Math.max(1,selection(p).quantity-1)">−</button><input class="qty-input" type="number" min="1" v-model.number="selection(p).quantity"><button class="qty-btn" @click="selection(p).quantity++">+</button></div><div class="product-actions"><button class="btn-add-cart" @click="addToCart(p)">🛒 Add to Cart</button><button class="btn-buy-now" @click="buyNow(p)">Buy Now</button></div></div></div></div></div></section>

    <section v-else-if="currentPage==='product-detail' && selectedProduct" class="product-detail"><div class="detail-content"><div class="detail-image">{{selectedProduct.image}}</div><div class="detail-info"><h1>{{selectedProduct.name}}</h1><p class="detail-desc">{{selectedProduct.description}} <span v-if="selectedProduct.details">{{selectedProduct.details}}</span></p><div v-if="selectedProduct.variants.type==='multi'" class="variant-section"><label class="variant-label">Select Size</label><div class="variant-options"><button v-for="size in selectedProduct.variants.sizes" :key="size" :class="['variant-option',{active:selection(selectedProduct).size===size}]" @click="selection(selectedProduct).size=size">{{size}}</button></div><label class="variant-label">Select Thickness</label><div class="variant-options"><button v-for="th in selectedProduct.variants.thicknesses" :key="th" :class="['variant-option',{active:selection(selectedProduct).thickness===th}]" @click="selection(selectedProduct).thickness=th">{{th}}</button></div></div><div class="product-price">PKR {{priceFor(selectedProduct).toLocaleString()}}</div><label class="variant-label">Quantity</label><div class="quantity-selector"><button class="qty-btn" @click="selection(selectedProduct).quantity=Math.max(1,selection(selectedProduct).quantity-1)">−</button><input class="qty-input" type="number" min="1" v-model.number="selection(selectedProduct).quantity"><button class="qty-btn" @click="selection(selectedProduct).quantity++">+</button></div><div class="product-actions"><button class="btn-add-cart" @click="addToCart(selectedProduct)">🛒 Add to Cart</button><button class="btn-buy-now" @click="buyNow(selectedProduct)">Buy Now</button></div><div class="whatsapp-help"><h4>Need help choosing the right size?</h4><a :href="`${whatsapp}?text=${encodeURIComponent('Hi Wrap N Roll, I need help choosing the right size for '+selectedProduct.name)}`" target="_blank">Chat with us on WhatsApp →</a></div></div></div></section>

    <section v-else-if="currentPage==='cart'" class="cart-container"><template v-if="cartItems.length"><h1 class="section-title">Shopping Cart</h1><div class="cart-items"><div v-for="item in cartItems" :key="itemKey(item)" class="cart-item"><div class="cart-item-image">{{item.image}}</div><div class="cart-item-details"><h3>{{item.name}}</h3><p v-if="item.selectedVariants?.size">Size: {{item.selectedVariants.size}}</p><p v-if="item.selectedVariants?.thickness">Thickness: {{item.selectedVariants.thickness}}</p><div class="quantity-selector"><button class="qty-btn" @click="updateQuantity(item.id,item.selectedVariants,item.quantity-1)">−</button><input class="qty-input" type="number" min="1" :value="item.quantity" @change="updateQuantity(item.id,item.selectedVariants,$event.target.value)"><button class="qty-btn" @click="updateQuantity(item.id,item.selectedVariants,item.quantity+1)">+</button></div></div><div class="cart-item-actions"><div class="cart-item-price">PKR {{(item.finalPrice*item.quantity).toLocaleString()}}</div><button class="btn-remove" @click="removeFromCart(item.id,item.selectedVariants)">Remove</button></div></div></div><div class="cart-summary"><div class="summary-row"><span>Subtotal:</span><span>PKR {{subtotal.toLocaleString()}}</span></div><div class="summary-row"><span>Delivery Charge (Karachi):</span><span>PKR {{CONFIG.deliveryCharge.toLocaleString()}}</span></div><div class="summary-row total"><span>Total:</span><span>PKR {{total.toLocaleString()}}</span></div><button class="checkout-btn" @click="navigate('#checkout')">Proceed to Checkout</button></div></template><div v-else class="empty-state"><div class="empty-state-icon">🛒</div><h2>Your cart is empty</h2><p>Start shopping to add items to your cart</p><a href="#shop" class="btn btn-primary">Continue Shopping</a></div></section>

    <section v-else-if="currentPage==='checkout'" class="checkout-container"><h1 class="section-title">Checkout</h1><div v-if="!cartItems.length" class="empty-state">Please add items to your cart first</div><template v-else><div class="delivery-warning">✓ We currently deliver only within <strong>Karachi</strong>. Delivery charge: <strong>PKR {{CONFIG.deliveryCharge}}</strong></div><form @submit.prevent="placeOrder"><div class="form-section"><h2 class="form-heading">Delivery Information</h2><div class="form-row"><div class="form-group"><label>Full Name *</label><input v-model="checkout.fullName" :class="{invalid:errors.fullName}"><small v-if="errors.fullName">This field is required</small></div><div class="form-group"><label>Phone Number *</label><input v-model="checkout.phoneNumber" type="tel" :class="{invalid:errors.phoneNumber}"><small v-if="errors.phoneNumber">This field is required</small></div></div><div class="form-group"><label>WhatsApp Number *</label><input v-model="checkout.whatsappNumber" type="tel" :class="{invalid:errors.whatsappNumber}"><small v-if="errors.whatsappNumber">This field is required</small></div><div class="form-group"><label>Complete Address *</label><textarea v-model="checkout.address" :class="{invalid:errors.address}"></textarea><small v-if="errors.address">This field is required</small></div><div class="form-row"><div class="form-group"><label>Area / Locality *</label><input v-model="checkout.area" :class="{invalid:errors.area}"><small v-if="errors.area">This field is required</small></div><div class="form-group"><label>Landmark (Optional)</label><input v-model="checkout.landmark" placeholder="e.g., Near XYZ Store"></div></div><div class="form-group"><label>City *</label><select v-model="checkout.city"><option>Karachi</option><option value="Other">Other Cities (Not Available)</option></select><small v-if="errors.city">Delivery is currently available only in Karachi.</small></div></div><div class="form-section"><h2 class="form-heading">Payment Method</h2>
<div class="payment-methods">
  <button
    type="button"
    :class="['payment-option', { active: checkout.paymentMethod === 'Advance' }]"
    @click="checkout.paymentMethod = 'Advance'"
  >
    <div class="payment-option-icon">📱</div>
    <div>
      <strong>JazzCash / Easypaisa</strong>
      <p class="product-desc" style="margin: 4px 0 0">
        Advance payment — 0336-1271119
      </p>
    </div>
  </button>
</div>
<p class="product-desc"></p></div><div class="form-section"><h2 class="form-heading">Order Summary</h2><div v-for="item in cartItems" :key="itemKey(item)" class="summary-row"><span>{{item.name}} × {{item.quantity}}</span><span>PKR {{(item.finalPrice*item.quantity).toLocaleString()}}</span></div><div class="summary-row"><span>Subtotal:</span><span>PKR {{subtotal.toLocaleString()}}</span></div><div class="summary-row"><span>Delivery Charge:</span><span>PKR {{CONFIG.deliveryCharge.toLocaleString()}}</span></div><div class="summary-row total"><span>Total:</span><span>PKR {{total.toLocaleString()}}</span></div></div>
<p v-if="orderError" role="alert" style="color: #dc2626; margin: 12px 0;">
  {{ orderError }}
</p>

<button
  class="checkout-btn"
  type="button"
  :disabled="isSubmitting"
  @click="placeOrder"
>
  {{ isSubmitting ? 'Saving Order...' : 'Place Order' }}
</button>
</form></template></section>

    <section v-else-if="currentPage==='confirmation' && orderData" class="order-confirmation"><div class="success-icon">✅</div><h2 class="confirmation-title">Order Confirmed!</h2><p>Thank you for shopping with Wrap N Roll</p><div class="confirmation-details"><div class="detail-row"><span class="detail-label">Order Number:</span><span class="detail-value">#{{orderData.number}}</span></div><div class="detail-row"><span class="detail-label">Customer Name:</span><span class="detail-value">{{orderData.fullName}}</span></div><div class="detail-row"><span class="detail-label">Phone:</span><span class="detail-value">{{orderData.phoneNumber}}</span></div><div class="detail-row"><span class="detail-label">Delivery Area:</span><span class="detail-value">{{orderData.area}}, Karachi</span></div><div class="detail-row"><span class="detail-label">Payment Method:</span><span class="detail-value">{{orderData.paymentMethod==='COD'?'Cash on Delivery':'Card Payment (not integrated)'}}</span></div><div class="detail-row"><span class="detail-label">Total Amount:</span><span class="detail-value">PKR {{orderData.total?.toLocaleString()}}</span></div></div><div class="confirmation-details"><p>📦 Your order will be delivered to:</p><p>{{orderData.address}}<br>{{orderData.area}}{{orderData.landmark?', '+orderData.landmark:''}}<br>Karachi</p></div><p style="margin:1rem">Have questions? Chat with us on WhatsApp</p><a :href="whatsapp" target="_blank" class="btn btn-secondary">💬 Contact Us on WhatsApp</a><a href="#" class="btn btn-primary" style="margin-top:1rem">Continue Shopping</a></section>

    <footer v-if="currentPage !== 'admin'" id="contact" class="footer"><div class="footer-content"><div class="footer-section"><h3>📦 WRAP N ROLL</h3><p style="font-size:.9rem;color:rgba(255,255,255,.8)">Protect. Pack. Deliver.</p></div><div class="footer-section"><h3>Quick Links</h3><ul><li><a href="#shop">Shop</a></li><li><a href="#category/Bubble%20Wrap">Bubble Wrap</a></li><li><a href="#category/Thermocol%20Sheets">Thermocol</a></li><li><a href="#category/XPS%20Sheets">XPS Sheets</a></li></ul></div><div class="footer-section"><h3>Contact</h3><ul><li><a :href="whatsapp" target="_blank">💬 WhatsApp</a></li><li><p>📍 Karachi, Pakistan</p></li><li></li></ul></div></div><div class="footer-bottom"><p>
          ©. {{ new Date().getFullYear() }}
          <span class="font-semibold text-zinc-300">
           
          </span>
          All Rights Reserved.

          Powered By
          <a
            href="https://aqil-lab.vercel.app"
            target="_blank"
            rel="noopener noreferrer"
            class="font-semibold text-zinc-300 no-underline transition hover:text-white hover:underline"
          >
            AqilLabs
          </a>
        </p>


        </div></footer>
    <a :href="whatsapp" target="_blank" rel="noopener noreferrer" class="whatsapp-float">💬</a>
    <div v-if="toast" class="toast">{{toast}}</div>
  </div>
</template>
