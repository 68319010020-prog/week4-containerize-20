<script setup>
import { ref, computed, onMounted } from 'vue'

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

let debounceTimer = null            // สำหรับ debounce search input
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

// คำนวณ 4 ตัวเลข dashboard stats
const stats = computed(() => ({
  total:      products.value.length,
  lowStock:   products.value.filter(p => p.stock < 10).length,
  totalItems: products.value.reduce((s, p) => s + p.stock, 0),
  totalValue: products.value.reduce((s, p) => s + parseFloat(p.price) * p.stock, 0)
}))

// ── Methods ───────────────────────────────────────────────────────

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
  const body = {
    name:        form.value.name,
    category:    form.value.category,
    price:       parseFloat(form.value.price),
    stock:       parseInt(form.value.stock),
    description: form.value.description
  }
  // เลือก URL และ method HTTP ตามว่าเป็น update หรือสร้างใหม่
  const url    = editingId.value ? `/api/products/${editingId.value}` : '/api/products'
  const method = editingId.value ? 'PUT' : 'POST'
  await fetch(url, { method, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
  showModal.value = false
  fetchProducts()   // reload สินค้าใหม่
}

// ลบสินค้า
// ส่ง DELETE request ไปยัง API เพื่อลบสินค้าแล้ว refresh รายการ
async function deleteProduct(id) {
  await fetch(`/api/products/${id}`, { method: 'DELETE' })
  confirmDelete.value = null
  fetchProducts()  // รีเฟรชรายการสินค้าหลังการลบ
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
onMounted(fetchProducts)
</script>

<template>
  <div class="app-shell">

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
          </div>
          <div class="form-row">
            <div class="form-group">
              <label class="form-label">หมวดหมู่ *</label>
              <input class="form-input" v-model="form.category" list="cat-list" placeholder="เช่น Electronics" required />
              <datalist id="cat-list">
                <option v-for="c in categories" :value="c" :key="c" />
              </datalist>
            </div>
            <div class="form-group">
              <label class="form-label">ราคา (บาท) *</label>
              <input class="form-input" v-model="form.price" type="number" min="0" step="0.01" placeholder="0.00" required />
            </div>
          </div>
          <div class="form-group">
            <label class="form-label">จำนวนสต็อก (ชิ้น) *</label>
            <input class="form-input" v-model="form.stock" type="number" min="0" placeholder="0" required />
          </div>
          <div class="form-group">
            <label class="form-label">คำอธิบายสินค้า</label>
            <textarea class="form-input" v-model="form.description" rows="3" placeholder="รายละเอียดเพิ่มเติม (ถ้ามี)" style="resize:vertical"></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="showModal = false">ยกเลิก</button>
            <button type="submit" class="btn-save">
              {{ editingId ? 'บันทึกการเปลี่ยนแปลง' : 'เพิ่มสินค้า' }}
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
          <button class="btn-cancel" @click="confirmDelete = null">ยกเลิก</button>
          <button class="btn-danger-confirm" @click="deleteProduct(confirmDelete.id)">ลบเลย</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

.app-shell {
  min-height: 100vh;
  background: #0a0a0a;
  color: #f5f5f5;
}

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: #111; border-bottom: 1px solid #f472b6;
  height: 62px; padding: 0 1.5rem;
  display: flex; align-items: center; gap: .85rem;
  box-shadow: 0 2px 16px rgba(244, 114, 182, .15);
}
.logo { display: flex; align-items: center; gap: .6rem; }
.logo-icon { font-size: 1.6rem; }
.logo-name { font-weight: 800; font-size: 1.15rem; color: #fde047; line-height: 1; }
.logo-sub  { font-size: .72rem; color: #f9a8d4; }
.btn-add {
  margin-left: auto;
  background: linear-gradient(135deg, #facc15, #f472b6);
  color: #0a0a0a;
  border: none; border-radius: 8px;
  padding: .55rem 1.2rem; font-size: .9rem; font-weight: 700;
  cursor: pointer; transition: opacity .2s, transform .2s;
}
.btn-add:hover { opacity: .9; transform: translateY(-1px); }

.main { max-width: 1280px; margin: 0 auto; padding: 1.75rem 1.5rem; }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem; margin-bottom: 1.5rem;
}
.stat-card {
  background: #161616; border: 1px solid #333;
  border-radius: 12px; padding: 1.1rem;
  display: flex; align-items: center; gap: .9rem;
}
.stat-icon {
  width: 46px; height: 46px; border-radius: 10px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.3rem; flex-shrink: 0;
}
.si-green { background: rgba(250, 204, 21, .15); }
.si-red   { background: rgba(244, 114, 182, .15); }
.si-amber { background: rgba(253, 224, 71, .12); }
.si-blue  { background: rgba(251, 113, 133, .15); }
.stat-val { font-size: 1.65rem; font-weight: 800; line-height: 1; }
.stat-yellow { color: #fde047; }
.stat-pink   { color: #f472b6; }
.stat-money  { font-size: 1.3rem; }
.stat-label { font-size: .8rem; color: #a3a3a3; margin-top: .2rem; }

.alert-low {
  background: rgba(244, 114, 182, .1); border: 1px solid #f472b6;
  border-radius: 10px; padding: .85rem 1.2rem;
  font-size: .92rem; margin-bottom: 1.5rem; color: #fbcfe8;
}

.toolbar { display: flex; gap: .75rem; margin-bottom: 1.5rem; flex-wrap: wrap; align-items: center; }
.input-search {
  flex: 1; min-width: 200px;
  padding: .6rem 1rem; border: 1.5px solid #404040; border-radius: 8px;
  font-size: .95rem; outline: none; transition: border-color .2s;
  background: #161616; color: #f5f5f5;
}
.input-search::placeholder { color: #737373; }
.input-search:focus { border-color: #facc15; }
.input-select {
  padding: .6rem .9rem; border: 1.5px solid #404040; border-radius: 8px;
  background: #161616; color: #f5f5f5; font-size: .9rem; outline: none; cursor: pointer;
}
.input-select:focus { border-color: #f472b6; }
.result-count { font-size: .82rem; color: #a3a3a3; white-space: nowrap; }

.state-box { text-align: center; padding: 4rem 1rem; color: #a3a3a3; }
.state-icon { font-size: 3rem; margin-bottom: .75rem; }
.state-title { font-size: 1.1rem; font-weight: 700; color: #fde047; margin-bottom: .3rem; }
.state-desc { font-size: .9rem; }

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(275px, 1fr));
  gap: 1.2rem;
}
.product-card {
  background: #161616; border: 1px solid #333; border-radius: 14px;
  overflow: hidden; transition: transform .2s, box-shadow .2s, border-color .2s;
}
.product-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 28px rgba(250, 204, 21, .12), 0 4px 16px rgba(244, 114, 182, .1);
  border-color: #f472b6;
}
.product-card.card-low  { border-color: #fb7185; }
.product-card.card-out  { border-color: #525252; opacity: .65; }

.card-top { padding: 1.1rem; }
.cat-badge {
  display: inline-block; padding: .2rem .65rem;
  border-radius: 20px; font-size: .72rem; font-weight: 700; margin-bottom: .6rem;
}
.c-elec  { background: rgba(250, 204, 21, .2); color: #fde047; }
.c-cloth { background: rgba(244, 114, 182, .2); color: #f9a8d4; }
.c-foot  { background: rgba(251, 113, 133, .2); color: #fda4af; }
.c-food  { background: rgba(234, 179, 8, .2); color: #facc15; }
.c-sport { background: rgba(236, 72, 153, .2); color: #f472b6; }
.c-book  { background: rgba(253, 224, 71, .15); color: #fef08a; }
.c-furn  { background: rgba(219, 39, 119, .2); color: #fb7185; }
.c-bag   { background: rgba(250, 204, 21, .15); color: #eab308; }
.c-acc   { background: rgba(244, 114, 182, .15); color: #ec4899; }
.c-tool  { background: rgba(163, 163, 163, .15); color: #d4d4d4; }
.c-other { background: rgba(253, 224, 71, .12); color: #fbbf24; }

.product-name  { font-size: 1rem; font-weight: 700; color: #fafafa; margin-bottom: .3rem; line-height: 1.35; }
.product-desc  { font-size: .82rem; color: #a3a3a3; line-height: 1.5; margin-bottom: .75rem; }
.product-price { font-size: 1.25rem; font-weight: 800; color: #fde047; }

.stock-info { margin-top: .85rem; }
.stock-row  { display: flex; justify-content: space-between; font-size: .82rem; margin-bottom: .3rem; }
.stock-label { color: #a3a3a3; }
.stock-num   { font-weight: 700; }
.stock-num.out  { color: #737373; }
.stock-num.low  { color: #fb7185; }
.stock-num.mid  { color: #facc15; }
.stock-num.high { color: #fde047; }
.stock-track { height: 6px; background: #262626; border-radius: 3px; overflow: hidden; }
.stock-fill  { height: 100%; border-radius: 3px; transition: width .4s ease; min-width: 4px; }
.stock-fill.out  { background: #525252; width: 2% !important; }
.stock-fill.low  { background: #f472b6; }
.stock-fill.mid  { background: #facc15; }
.stock-fill.high { background: #fde047; }

.card-footer {
  display: flex; gap: .5rem;
  padding: .75rem 1.1rem;
  border-top: 1px solid #262626; background: #111;
}
.btn-edit, .btn-del {
  flex: 1; padding: .45rem; border-radius: 7px;
  font-size: .82rem; font-weight: 600; cursor: pointer; border: none; transition: all .2s;
}
.btn-edit { background: rgba(250, 204, 21, .15); color: #fde047; }
.btn-edit:hover { background: rgba(250, 204, 21, .25); }
.btn-del  { background: rgba(244, 114, 182, .15); color: #f472b6; }
.btn-del:hover  { background: rgba(244, 114, 182, .28); }

.overlay {
  position: fixed; inset: 0;
  background: rgba(0, 0, 0, .75);
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1rem;
}
.modal {
  background: #161616; border: 1px solid #404040; border-radius: 16px;
  width: 100%; max-width: 480px;
  max-height: 90vh; overflow-y: auto; padding: 2rem;
  color: #f5f5f5;
}
.modal-title { font-size: 1.2rem; font-weight: 700; margin-bottom: 1.5rem; color: #fde047; }

.form-group { margin-bottom: 1rem; }
.form-label { display: block; font-size: .87rem; font-weight: 600; margin-bottom: .35rem; color: #f9a8d4; }
.form-input {
  width: 100%; padding: .6rem .85rem;
  border: 1.5px solid #404040; border-radius: 8px;
  font-size: .95rem; outline: none; transition: border-color .2s;
  background: #0a0a0a; color: #f5f5f5;
}
.form-input:focus { border-color: #f472b6; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: .75rem; }

.modal-footer {
  display: flex; gap: .75rem; justify-content: flex-end;
  padding-top: 1rem; border-top: 1px solid #262626; margin-top: 1rem;
}
.btn-cancel {
  background: #262626; color: #d4d4d4;
  border: none; border-radius: 8px; padding: .6rem 1.25rem;
  font-weight: 600; cursor: pointer;
}
.btn-cancel:hover { background: #333; }
.btn-save {
  background: linear-gradient(135deg, #facc15, #f472b6);
  color: #0a0a0a;
  border: none; border-radius: 8px; padding: .6rem 1.25rem;
  font-weight: 700; cursor: pointer;
}
.btn-save:hover { opacity: .9; }

.confirm { text-align: center; max-width: 380px; }
.confirm-icon  { font-size: 3rem; margin-bottom: 1rem; }
.confirm-title { font-size: 1.1rem; font-weight: 700; margin-bottom: .5rem; color: #fde047; }
.confirm-desc  { font-size: .9rem; color: #a3a3a3; margin-bottom: 1.5rem; line-height: 1.6; }
.confirm-actions { display: flex; gap: .75rem; justify-content: center; }
.btn-danger-confirm {
  background: #ec4899; color: #0a0a0a;
  border: none; border-radius: 8px; padding: .6rem 1.4rem;
  font-weight: 700; cursor: pointer;
}
.btn-danger-confirm:hover { background: #f472b6; }

@media (max-width: 640px) {
  .form-row { grid-template-columns: 1fr; }
  .stats-grid { grid-template-columns: 1fr 1fr; }
}
</style>