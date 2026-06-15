<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// ── State ────────────────────────────────────────────────────────
const products      = ref([])       // รายการสินค้าทั้งหมดจาก API
const loading       = ref(true)     // แสดง loading spinner
const search        = ref('')       // ข้อความค้นหา
const catFilter     = ref('')       // category ที่กำลัง filter
const showModal     = ref(false)    // แสดง/ซ่อน modal เพิ่ม/แก้ไข
const editingId     = ref(null)     // null = กำลัง add, มีค่า = กำลัง edit
const confirmDelete = ref(null)     // เก็บ product ที่จะลบ (ใช้ confirm dialog)

// ข้อมูลในฟอร์ม modal
const form = ref({ name: '', category: '', price: '', stock: 0, description: '' })
const formErrors = ref({})          // เก็บ validation errors
const isSaving = ref(false)         // แสดง loading state ของ save button
const isDeleting = ref(false)       // แสดง loading state ของ delete button
const notification = ref(null)      // { type: 'success'|'error', message }

let debounceTimer = null            // สำหรับ debounce search input
let notificationTimer = null        // timer สำหรับ notification auto-close
// ── Computed ─────────────────────────────────────────────────────

// กรองสินค้าตาม search + catFilter พร้อมกัน
const filtered = computed(() => {
  const s = search.value.toLowerCase()
  return products.value.filter(p => {
    // ตรงกับ search text? (เช็คทั้ง name และ description)
    const matchS = !s ||
      p.name.toLowerCase().includes(s) ||
      (p.description || '').toLowerCase().includes(s)
    // ตรงกับ category filter?
    const matchC = !catFilter.value || p.category === catFilter.value
    return matchS && matchC
  })
})

// สร้าง list ของ categories จาก products ที่มีอยู่ (unique + sort)
const categories = computed(() =>
  [...new Set(products.value.map(p => p.category))].sort()
)

// sanitize input สำหรับป้องกัน XSS
const sanitize = (str) => {
  const div = document.createElement('div')
  div.textContent = String(str || '')
  return div.innerHTML
}

// คำนวณ 4 ตัวเลข dashboard stats
const stats = computed(() => ({
  total:      products.value.length,
  lowStock:   products.value.filter(p => p.stock < 10).length,
  totalItems: products.value.reduce((s, p) => s + p.stock, 0),
  totalValue: products.value.reduce((s, p) => s + parseFloat(p.price) * p.stock, 0)
}))

// ── Methods ───────────────────────────────────────────────────────

function showNotification(message, type = 'success', duration = 3000) {
  clearTimeout(notificationTimer)
  notification.value = { message, type }
  notificationTimer = setTimeout(() => { notification.value = null }, duration)
}

function validateForm() {
  formErrors.value = {}
  const { name, category, price, stock } = form.value
  
  if (!name?.trim()) formErrors.value.name = 'ป้อนชื่อสินค้า'
  if (!category?.trim()) formErrors.value.category = 'ป้อนหมวดหมู่'
  if (!price || parseFloat(price) < 0) formErrors.value.price = 'ราคาต้องมากกว่า 0'
  if (stock === '' || parseInt(stock) < 0) formErrors.value.stock = 'จำนวนสต็อกต้องมากกว่า 0'
  
  return Object.keys(formErrors.value).length === 0
}

function handleKeydown(e) {
  if (e.key === 'Escape') {
    if (showModal.value) showModal.value = false
    if (confirmDelete.value) confirmDelete.value = null
  }
}

// debounce: รอ 280ms หลัง user หยุดพิมพ์แล้วค่อย fetch
// (ป้องกัน API ถูกเรียกทุก keystroke)
function debounceFetch() {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(fetchProducts, 280)
}

// ดึงสินค้าจาก API (ส่ง search + catFilter เป็น query params)
// ฟังก์ชันนี้ตรวจสอบค่า search และ category filter ก่อนส่งไปยัง API
async function fetchProducts() {
  loading.value = true
  const q = new URLSearchParams()
  // สร้าง query string จากค่า search และ category ที่เลือก
  if (search.value)    q.set('search',   search.value)
  if (catFilter.value) q.set('category', catFilter.value)
  try {
    const res = await fetch(`/api/products?${q}`)
    products.value = await res.json()
  } catch {
    products.value = []
  } finally {
    loading.value = false
  }
}

// เปิด modal แบบ Add (clear form)
function openAdd() {
  editingId.value = null
  form.value = { name: '', category: '', price: '', stock: 0, description: '' }
  showModal.value = true
}

// เปิด modal แบบ Edit (copy ข้อมูลจาก product เข้า form)
function openEdit(p) {
  editingId.value = p.id
  form.value = {
    name: p.name, category: p.category,
    price: p.price, stock: p.stock,
    description: p.description || ''
  }
  showModal.value = true
}

// บันทึก (ถ้า editingId มีค่า = PUT, ถ้าเป็น null = POST)
// ฟังก์ชันนี้จะเลือก method และ URL ตามว่ากำลัง add หรือ edit
async function saveProduct() {
  if (!validateForm()) {
    showNotification('กรุณาปรหมว่าข้อมูลให้ถูกต้อง', 'error')
    return
  }
  
  isSaving.value = true
  try {
    const body = {
      name:        sanitize(form.value.name.trim()),
      category:    sanitize(form.value.category.trim()),
      price:       parseFloat(form.value.price),
      stock:       parseInt(form.value.stock),
      description: sanitize(form.value.description.trim())
    }
    const url    = editingId.value ? `/api/products/${editingId.value}` : '/api/products'
    const method = editingId.value ? 'PUT' : 'POST'
    const res = await fetch(url, { method, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
    
    if (!res.ok) throw new Error('แห่น API ตอบตรอมมูลคำขอ')
    
    showModal.value = false
    showNotification(
      editingId.value ? 'สองหรแก่หมวดยังไง้แล้ว' : 'เพิ่มสินค้าสำเร็จ!',
      'success'
    )
    fetchProducts()
  } catch (error) {
    showNotification('ซื่อ: ' + (error.message || 'ไม่สามารถบันทึกข้อมูล'), 'error')
  } finally {
    isSaving.value = false
  }
}

// ลบสินค้า
// ส่ง DELETE request ไปยัง API เพื่อลบสินค้าแล้ว refresh รายการ
async function deleteProduct(id) {
  isDeleting.value = true
  try {
    const res = await fetch(`/api/products/${id}`, { method: 'DELETE' })
    if (!res.ok) throw new Error('ไม่สามารถลบสินค้า')
    
    confirmDelete.value = null
    showNotification('ลบสินค้าเรียบร้อยแล้ว!', 'success')
    fetchProducts()  // รีเฟรชรายการสินค้าหลังการลบ
  } catch (error) {
    showNotification('ซื่อ: ' + (error.message || 'ไม่สามารถลบสินค้า'), 'error')
  } finally {
    isDeleting.value = false
  }
}

// คืน CSS class ตาม stock level (ใช้กับ stock bar และตัวเลข)
function stockClass(s) {
  if (s <= 0) return 'out'    // หมด
  if (s < 10) return 'low'    // ใกล้หมด
  if (s < 30) return 'mid'    // ปานกลาง
  return 'high'               // เพียงพอ
}

// คืน CSS class ตาม category (ใช้กับ badge สี)
// ใช้ object map เพื่อจับคู่ชื่อ category กับ CSS class ที่เกี่ยวข้อง
function catClass(cat) {
  const m = {
    Electronics: 'c-elec', Clothing: 'c-cloth',
    Footwear:    'c-foot',  Food:     'c-food',
    Sports:      'c-sport', Books:    'c-book',
    Furniture:   'c-furn',  Bags:     'c-bag',
    Accessories: 'c-acc',   Tools:    'c-tool'
  }
  return m[cat] || 'c-other'  // ส่งค่า class อื่น ถ้า category ไม่ตรงกับรายการ
}

// โหลดสินค้าทันทีเมื่อ component mount ครั้งแรก
onMounted(() => {
  fetchProducts()
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  clearTimeout(notificationTimer)
})
</script>

<template>
  <div class="app-shell">

    <!-- NOTIFICATION TOAST -->
    <div v-if="notification" class="notification" :class="`notification-${notification.type}`">
      {{ notification.message }}
    </div>

    <!-- HEADER -->
    <header class="app-header">
      <div class="logo">
        <span class="logo-icon">📦</span>
        <div>
          <div class="logo-name">StockPro</div>
          <div class="logo-sub">ระบบจัดการสินค้าคงคลัง</div>
        </div>
      </div>
      <button class="btn-add" @click="openAdd">+ เพิ่มสินค้า</button>
    </header>

    <main class="main">

      <!-- STATS DASHBOARD -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon si-green">📦</div>
          <div class="stat-body">
            <div class="stat-val stat-yellow">{{ stats.total }}</div>
            <div class="stat-label">สินค้าทั้งหมด</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-red">⚠️</div>
          <div class="stat-body">
            <div class="stat-val stat-pink">{{ stats.lowStock }}</div>
            <div class="stat-label">สต็อกใกล้หมด (&lt;10)</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-amber">📊</div>
          <div class="stat-body">
            <div class="stat-val stat-yellow">{{ stats.totalItems.toLocaleString() }}</div>
            <div class="stat-label">จำนวนสต็อกรวม</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-blue">💰</div>
          <div class="stat-body">
            <div class="stat-val stat-pink stat-money">
              ฿{{ Math.round(stats.totalValue).toLocaleString() }}
            </div>
            <div class="stat-label">มูลค่าสต็อกรวม</div>
          </div>
        </div>
      </div>

      <!-- LOW STOCK ALERT -->
      <div class="alert-low" v-if="stats.lowStock > 0">
        🚨 มีสินค้า <strong>{{ stats.lowStock }} รายการ</strong>
        ที่มีจำนวนสต็อกน้อยกว่า 10 ชิ้น — กรุณาตรวจสอบและเติมสต็อก
      </div>

      <!-- TOOLBAR -->
      <div class="toolbar">
        <input
          v-model="search"
          @input="debounceFetch"
          class="input-search"
          placeholder="🔍 ค้นหาชื่อสินค้า หรือคำอธิบาย..."
        />
        <select v-model="catFilter" @change="fetchProducts" class="input-select">
          <option value="">ทุกหมวดหมู่</option>
          <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
        </select>
        <span class="result-count" v-if="!loading">
          แสดง {{ filtered.length }} / {{ products.length }} รายการ
        </span>
      </div>
            <!-- LOADING -->
      <div class="state-box" v-if="loading">
        <div class="state-icon">⏳</div>
        <div>กำลังโหลดข้อมูลสินค้า...</div>
      </div>

      <!-- EMPTY -->
      <div class="state-box" v-else-if="filtered.length === 0">
        <div class="state-icon">📭</div>
        <div class="state-title">ไม่พบสินค้า</div>
        <div class="state-desc">
          {{ search || catFilter ? 'ลองเปลี่ยน keyword หรือ filter' : 'กดปุ่ม "+ เพิ่มสินค้า" เพื่อเริ่มต้น' }}
        </div>
      </div>

      <!-- PRODUCT GRID -->
      <div class="product-grid" v-else>
        <div
          v-for="p in filtered"
          :key="p.id"
          class="product-card"
          :class="{ 'card-low': p.stock > 0 && p.stock < 10, 'card-out': p.stock <= 0 }"
        >
          <div class="card-top">
            <span class="cat-badge" :class="catClass(p.category)">{{ p.category }}</span>
            <div class="product-name">{{ p.name }}</div>
            <div class="product-desc" v-if="p.description">{{ p.description }}</div>
            <div class="product-price">
              ฿{{ parseFloat(p.price).toLocaleString('th-TH', { minimumFractionDigits: 2 }) }}
            </div>
            <div class="stock-info">
              <div class="stock-row">
                <span class="stock-label">สต็อก</span>
                <span class="stock-num" :class="stockClass(p.stock)">
                  {{ p.stock.toLocaleString() }} ชิ้น
                  <span v-if="p.stock <= 0"> — หมดแล้ว!</span>
                  <span v-else-if="p.stock < 10"> — ใกล้หมด!</span>
                </span>
              </div>
              <div class="stock-track">
                <div
                  class="stock-fill"
                  :class="stockClass(p.stock)"
                  :style="{ width: Math.min((p.stock / 80) * 100, 100) + '%' }"
                ></div>
              </div>
            </div>
          </div>
          <div class="card-footer">
            <button class="btn-edit" @click="openEdit(p)">✏️ แก้ไข</button>
            <button class="btn-del" @click="confirmDelete = p">🗑️ ลบ</button>
          </div>
        </div>
      </div>

    </main>
        <!-- ADD/EDIT MODAL -->
    <div class="overlay" v-if="showModal" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-title">
          {{ editingId ? '✏️ แก้ไขสินค้า' : '📦 เพิ่มสินค้าใหม่' }}
        </div>
        <form @submit.prevent="saveProduct">
          <div class="form-group">
            <label class="form-label">ชื่อสินค้า *</label>
            <input class="form-input" v-model="form.name" placeholder="เช่น MacBook Air M3" required />
            <div v-if="formErrors.name" class="form-error">{{ formErrors.name }}</div>
          </div>
          <div class="form-row">
            <div class="form-group">
              <label class="form-label">หมวดหมู่ *</label>
              <input class="form-input" v-model="form.category" list="cat-list" placeholder="เช่น Electronics" required />
              <datalist id="cat-list">
                <option v-for="c in categories" :value="c" :key="c" />
              </datalist>
              <div v-if="formErrors.category" class="form-error">{{ formErrors.category }}</div>
            </div>
            <div class="form-group">
              <label class="form-label">ราคา (บาท) *</label>
              <input class="form-input" v-model="form.price" type="number" min="0" step="0.01" placeholder="0.00" required />
              <div v-if="formErrors.price" class="form-error">{{ formErrors.price }}</div>
            </div>
          </div>
          <div class="form-group">
            <label class="form-label">จำนวนสต็อก (ชิ้น) *</label>
            <input class="form-input" v-model="form.stock" type="number" min="0" placeholder="0" required />
            <div v-if="formErrors.stock" class="form-error">{{ formErrors.stock }}</div>
          </div>
          <div class="form-group">
            <label class="form-label">คำอธิบายสินค้า</label>
            <textarea class="form-input" v-model="form.description" rows="3" placeholder="รายละเอียดเพิ่มเติม (ถ้ามี)" style="resize:vertical"></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="showModal = false">ยกเลิก</button>
            <button type="submit" class="btn-save" :disabled="isSaving">
              {{ isSaving ? '⏳ กำลังบันทึก...' : (editingId ? 'บันทึกการเปลี่ยนแปลง' : 'เพิ่มสินค้า') }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- DELETE CONFIRM -->
    <div class="overlay" v-if="confirmDelete" @click.self="confirmDelete = null">
      <div class="modal confirm">
        <div class="confirm-icon">🗑️</div>
        <div class="confirm-title">ยืนยันการลบสินค้า</div>
        <div class="confirm-desc">
          คุณต้องการลบ <strong>{{ confirmDelete?.name }}</strong> ออกจากระบบหรือไม่?<br>
          <span style="color:#fb7185">การกระทำนี้ไม่สามารถย้อนกลับได้</span>
        </div>
        <div class="confirm-actions">
          <button class="btn-cancel" @click="confirmDelete = null" :disabled="isDeleting">ยกเลิก</button>
          <button class="btn-danger-confirm" @click="deleteProduct(confirmDelete.id)" :disabled="isDeleting">
            {{ isDeleting ? '⏳ กำลังลบ...' : 'ลบเลย' }}
          </button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;
  --color-text-primary: #1a202c;
  --color-text-secondary: #64748b;
  --color-bg-light: #f8fafc;
  --color-border: #e2e8f0;
  --color-accent: #f472b6;
}

.app-shell {
  min-height: 100vh;
  background: linear-gradient(135deg, #ffffff 0%, #f8fafc 100%);
  color: var(--color-text-primary);
  line-height: 1.6;
}

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: rgba(255, 255, 255, 0.95);
  border-bottom: 1px solid var(--color-border);
  backdrop-filter: blur(10px);
  height: 70px; padding: 0 var(--spacing-lg);
  display: flex; align-items: center; gap: 1rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05), 0 1px 2px rgba(0, 0, 0, 0.03);
}
.logo { display: flex; align-items: center; gap: 0.75rem; }
.logo-icon { font-size: 1.8rem; }
.logo-name { font-weight: 800; font-size: 1.25rem; color: #fbbf24; line-height: 1; letter-spacing: -0.5px; }
.logo-sub  { font-size: 0.75rem; color: var(--color-text-secondary); font-weight: 500; }
.btn-add {
  margin-left: auto;
  background: linear-gradient(135deg, #facc15 0%, #f472b6 100%);
  color: #ffffff;
  border: none; border-radius: 10px;
  padding: 0.7rem 1.4rem; font-size: 0.95rem; font-weight: 600;
  cursor: pointer; transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(244, 114, 182, 0.2);
}
.btn-add:hover { 
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(244, 114, 182, 0.3);
}

.main { max-width: 1200px; margin: 0 auto; padding: var(--spacing-xl) var(--spacing-lg); }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.25rem; margin-bottom: 2rem;
}
.stat-card {
  background: #ffffff;
  border: 1px solid var(--color-border);
  border-radius: 14px; padding: 1.5rem;
  display: flex; align-items: center; gap: 1.2rem;
  transition: all 0.3s ease;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}
.stat-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  border-color: #fbbf24;
}
.stat-icon {
  width: 52px; height: 52px; border-radius: 12px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.5rem; flex-shrink: 0;
}
.si-green { background: rgba(250, 204, 21, 0.1); }
.si-red   { background: rgba(244, 114, 182, 0.1); }
.si-amber { background: rgba(251, 113, 133, 0.1); }
.si-blue  { background: rgba(59, 130, 246, 0.1); }
.stat-val { font-size: 1.75rem; font-weight: 700; line-height: 1; letter-spacing: -0.5px; }
.stat-yellow { color: #fbbf24; }
.stat-pink   { color: #f472b6; }
.stat-money  { font-size: 1.35rem; }
.stat-label { font-size: 0.85rem; color: var(--color-text-secondary); margin-top: 0.4rem; font-weight: 500; }

.alert-low {
  background: rgba(250, 204, 21, 0.05); border: 1px solid rgba(251, 113, 133, 0.3);
  border-radius: 12px; padding: 1rem 1.25rem;
  font-size: 0.95rem; margin-bottom: 1.75rem; color: #c2410c;
  line-height: 1.6; font-weight: 500;
}

.toolbar { display: flex; gap: 1rem; margin-bottom: 2rem; flex-wrap: wrap; align-items: center; }
.input-search {
  flex: 1; min-width: 220px;
  padding: 0.75rem 1.1rem; border: 1.5px solid var(--color-border); border-radius: 10px;
  font-size: 0.95rem; outline: none; transition: all 0.3s ease;
  background: #ffffff; color: var(--color-text-primary);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.03);
}
.input-search::placeholder { color: #cbd5e1; }
.input-search:focus { 
  border-color: #fbbf24;
  box-shadow: 0 0 0 3px rgba(251, 191, 36, 0.1);
  background: #fafaf9;
}
.input-select {
  padding: 0.75rem 1rem; border: 1.5px solid var(--color-border); border-radius: 10px;
  background: #ffffff; color: var(--color-text-primary); font-size: 0.95rem; outline: none; cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.03);
}
.input-select:focus { 
  border-color: #fbbf24;
  box-shadow: 0 0 0 3px rgba(251, 191, 36, 0.1);
}
.result-count { font-size: 0.85rem; color: var(--color-text-secondary); white-space: nowrap; font-weight: 500; }

.state-box { text-align: center; padding: 4rem 1.5rem; color: var(--color-text-secondary); }
.state-icon { font-size: 3.5rem; margin-bottom: 1rem; opacity: 0.8; }
.state-title { font-size: 1.2rem; font-weight: 700; color: #f472b6; margin-bottom: 0.5rem; }
.state-desc { font-size: 0.95rem; color: var(--color-text-secondary); line-height: 1.6; }

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.5rem;
}
.product-card {
  background: #ffffff;
  border: 1px solid var(--color-border);
  border-radius: 16px; overflow: hidden;
  transition: all 0.3s ease;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}
.product-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.1);
  border-color: #fbbf24;
}
.product-card.card-low  { border-color: rgba(251, 113, 133, 0.3); }
.product-card.card-out  { border-color: var(--color-border); opacity: 0.7; }

.card-top { padding: 1.4rem; }
.cat-badge {
  display: inline-block; padding: 0.35rem 0.8rem;
  border-radius: 20px; font-size: 0.75rem; font-weight: 700; margin-bottom: 0.8rem;
  letter-spacing: 0.5px;
}
.c-elec  { background: rgba(250, 204, 21, 0.15); color: #b45309; }
.c-cloth { background: rgba(244, 114, 182, 0.15); color: #be185d; }
.c-foot  { background: rgba(251, 113, 133, 0.15); color: #be123c; }
.c-food  { background: rgba(234, 179, 8, 0.15); color: #b45309; }
.c-sport { background: rgba(236, 72, 153, 0.15); color: #be185d; }
.c-book  { background: rgba(253, 224, 71, 0.15); color: #b45309; }
.c-furn  { background: rgba(219, 39, 119, 0.15); color: #be185d; }
.c-bag   { background: rgba(250, 204, 21, 0.15); color: #b45309; }
.c-acc   { background: rgba(244, 114, 182, 0.15); color: #be185d; }
.c-tool  { background: rgba(100, 116, 139, 0.1); color: #64748b; }
.c-other { background: rgba(251, 191, 36, 0.1); color: #b45309; }

.product-name  { font-size: 1.05rem; font-weight: 700; color: var(--color-text-primary); margin-bottom: 0.5rem; line-height: 1.4; }
.product-desc  { font-size: 0.85rem; color: var(--color-text-secondary); line-height: 1.5; margin-bottom: 0.9rem; }
.product-price { font-size: 1.35rem; font-weight: 700; color: #fbbf24; }

.stock-info { margin-top: 1rem; }
.stock-row  { display: flex; justify-content: space-between; font-size: 0.85rem; margin-bottom: 0.5rem; }
.stock-label { color: var(--color-text-secondary); font-weight: 500; }
.stock-num   { font-weight: 700; }
.stock-num.out  { color: #94a3b8; }
.stock-num.low  { color: #fb7185; }
.stock-num.mid  { color: #fbbf24; }
.stock-num.high { color: #facc15; }
.stock-track { height: 7px; background: #e2e8f0; border-radius: 4px; overflow: hidden; }
.stock-fill  { height: 100%; border-radius: 4px; transition: width 0.4s ease; min-width: 4px; }
.stock-fill.out  { background: #cbd5e1; width: 2% !important; }
.stock-fill.low  { background: #f472b6; }
.stock-fill.mid  { background: #fbbf24; }
.stock-fill.high { background: #fde047; }

.card-footer {
  display: flex; gap: 0.75rem;
  padding: 1rem 1.4rem;
  border-top: 1px solid var(--color-border); background: #f8fafc;
}
.btn-edit, .btn-del {
  flex: 1; padding: 0.6rem; border-radius: 8px;
  font-size: 0.85rem; font-weight: 600; cursor: pointer; border: none; 
  transition: all 0.3s ease;
}
.btn-edit { background: rgba(251, 191, 36, 0.1); color: #b45309; }
.btn-edit:hover { background: rgba(251, 191, 36, 0.2); }
.btn-del  { background: rgba(244, 114, 182, 0.1); color: #be185d; }
.btn-del:hover  { background: rgba(244, 114, 182, 0.2); }

.overlay {
  position: fixed; inset: 0;
  background: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(4px);
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1.5rem;
}
.modal {
  background: #ffffff; border: 1px solid var(--color-border); border-radius: 18px;
  width: 100%; max-width: 500px;
  max-height: 90vh; overflow-y: auto; padding: 2.5rem;
  color: var(--color-text-primary);
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.15);
}
.modal-title { font-size: 1.3rem; font-weight: 700; margin-bottom: 1.75rem; color: var(--color-text-primary); }

.form-group { margin-bottom: 1.25rem; }
.form-label { display: block; font-size: 0.9rem; font-weight: 600; margin-bottom: 0.5rem; color: var(--color-text-primary); }
.form-input {
  width: 100%; padding: 0.85rem 1rem;
  border: 1.5px solid var(--color-border); border-radius: 10px;
  font-size: 0.95rem; outline: none; transition: all 0.3s ease;
  background: #ffffff; color: var(--color-text-primary);
}
.form-input:focus { 
  border-color: #fbbf24;
  box-shadow: 0 0 0 3px rgba(251, 191, 36, 0.1);
  background: #fafaf9;
}
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }

.modal-footer {
  display: flex; gap: 1rem; justify-content: flex-end;
  padding-top: 1.5rem; border-top: 1px solid var(--color-border); margin-top: 1.5rem;
}
.btn-cancel {
  background: #f1f5f9; color: var(--color-text-secondary);
  border: none; border-radius: 10px; padding: 0.75rem 1.5rem;
  font-weight: 600; cursor: pointer; transition: all 0.3s ease;
}
.btn-cancel:hover { background: #e2e8f0; }
.btn-save {
  background: linear-gradient(135deg, #facc15 0%, #fbbf24 100%);
  color: #ffffff;
  border: none; border-radius: 10px; padding: 0.75rem 1.5rem;
  font-weight: 700; cursor: pointer; transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(251, 191, 36, 0.3);
}
.btn-save:hover { 
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(251, 191, 36, 0.4);
}

.confirm { text-align: center; max-width: 420px; }
.confirm-icon  { font-size: 3.5rem; margin-bottom: 1.25rem; opacity: 0.8; }
.confirm-title { font-size: 1.2rem; font-weight: 700; margin-bottom: 0.75rem; color: #1a202c; }
.confirm-desc  { font-size: 0.95rem; color: #64748b; margin-bottom: 1.75rem; line-height: 1.6; }
.confirm-actions { display: flex; gap: 1rem; justify-content: center; }
.btn-danger-confirm {
  background: linear-gradient(135deg, #fb7185 0%, #f43f5e 100%);
  color: #ffffff;
  border: none; border-radius: 10px; padding: 0.75rem 1.6rem;
  font-weight: 700; cursor: pointer; transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(244, 63, 94, 0.3);
}
.btn-danger-confirm:hover { 
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(244, 63, 94, 0.4);
}

/* Notification Toast */
.notification {
  position: fixed; top: 1.75rem; right: 1.75rem; z-index: 1000;
  padding: 1.1rem 1.6rem; border-radius: 12px; font-weight: 600;
  animation: slideIn 0.3s ease, slideOut 0.3s ease 2.7s forwards;
  max-width: 420px; word-wrap: break-word;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
}
.notification-success {
  background: #dcfce7; color: #166534; border: 1px solid #86efac;
}
.notification-error {
  background: #fee2e2; color: #991b1b; border: 1px solid #fca5a5;
}

@keyframes slideIn {
  from { transform: translateX(420px); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}
@keyframes slideOut {
  from { transform: translateX(0); opacity: 1; }
  to { transform: translateX(420px); opacity: 0; }
}

/* Form Validation Error */
.form-error {
  font-size: 0.8rem; color: #dc2626; margin-top: 0.4rem; font-weight: 600;
}

/* Button Disabled State */
.btn-save:disabled, .btn-danger-confirm:disabled {
  opacity: 0.6; cursor: not-allowed;
}
.btn-cancel:disabled {
  opacity: 0.5; cursor: not-allowed;
}

@media (max-width: 640px) {
  .form-row { grid-template-columns: 1fr; }
  .stats-grid { grid-template-columns: 1fr 1fr; }
  .notification { max-width: calc(100vw - 2rem); right: 1rem; left: 1rem; }
  .modal { padding: 1.75rem; }
  .app-header { padding: 0 1rem; }
  .main { padding: 1.5rem 1rem; }
}
</style>