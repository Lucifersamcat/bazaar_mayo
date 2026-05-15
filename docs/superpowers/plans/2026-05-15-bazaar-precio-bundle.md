# Bazaar Mayo — Calculadora de Precio de Venta — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file HTML app that calculates suggested sale prices for bazaar bundles with real-time updates, persistent inventory, and dynamic margin control.

**Architecture:** Single `index.html` with embedded CSS and vanilla JavaScript. All data (products, extras, margin) persisted in `localStorage`. Bundle state is in-memory only (not persisted). No build step, no dependencies, no server required.

**Tech Stack:** HTML5, CSS3, Vanilla JavaScript, localStorage

---

## File Structure

| File | Responsibility |
|------|----------------|
| `index.html` | Entire app — HTML structure, embedded CSS, embedded JS |

---

## Data Model

```js
// Persisted in localStorage under key "bazaar_state"
state = {
  products: [{ id, name, unitCost, freight, units }],
  extras:   [{ id, name, unitCost, freight, units }],
  margin:   30   // percentage 0–99
}

// In-memory only, reset on page load
bundle = {
  items:  [{ productId, qty }],
  extras: [extraId, ...]
}
```

**Computed helpers:**
```js
realCost(item)      = item.unitCost + (item.freight && item.units ? item.freight / item.units : 0)
bundleTotal()       = Σ(item.realCost × qty) + Σ(extra.realCost)
suggestedPrice()    = bundleTotal() / (1 - margin/100)   // margin=0 → price = cost
```

---

## Task 1: HTML Skeleton + CSS

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with 3-section layout and all styles**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Bazaar Mayo</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, sans-serif; background: #f5f5f5; color: #222; padding: 16px; }
    h1 { font-size: 1.3rem; margin-bottom: 16px; color: #444; }
    h2 { font-size: 1rem; font-weight: 600; margin-bottom: 10px; color: #555; text-transform: uppercase; letter-spacing: .05em; }
    h3 { font-size: .9rem; font-weight: 600; margin-bottom: 8px; color: #666; }

    section { background: #fff; border-radius: 8px; padding: 16px; margin-bottom: 14px; box-shadow: 0 1px 3px rgba(0,0,0,.08); }

    /* Config */
    #config { display: flex; align-items: center; gap: 12px; }
    #config label { font-size: .9rem; display: flex; align-items: center; gap: 6px; }
    #config input[type=number] { width: 64px; padding: 4px 6px; border: 1px solid #ccc; border-radius: 4px; font-size: .9rem; }

    /* Inventory */
    .inventory-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
    @media (max-width: 600px) { .inventory-grid { grid-template-columns: 1fr; } }

    table { width: 100%; border-collapse: collapse; font-size: .82rem; }
    th { text-align: left; padding: 5px 6px; border-bottom: 2px solid #eee; color: #888; font-weight: 600; }
    td { padding: 5px 6px; border-bottom: 1px solid #f0f0f0; vertical-align: middle; }
    tr:last-child td { border-bottom: none; }
    .empty-row td { color: #bbb; font-style: italic; text-align: center; }

    /* Buttons */
    button { cursor: pointer; border: none; border-radius: 4px; padding: 5px 10px; font-size: .82rem; }
    .btn-primary { background: #3b82f6; color: #fff; }
    .btn-primary:hover { background: #2563eb; }
    .btn-danger { background: none; color: #ef4444; padding: 2px 6px; }
    .btn-danger:hover { background: #fef2f2; }
    .btn-edit { background: none; color: #6b7280; padding: 2px 6px; }
    .btn-edit:hover { background: #f3f4f6; }
    .btn-secondary { background: #e5e7eb; color: #374151; }
    .btn-secondary:hover { background: #d1d5db; }
    .add-btn { margin-top: 8px; }

    /* Bundle */
    .bundle-adder { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; margin-bottom: 12px; }
    .bundle-adder select { padding: 5px 8px; border: 1px solid #ccc; border-radius: 4px; font-size: .85rem; flex: 1; min-width: 120px; }
    .bundle-adder input[type=number] { width: 60px; padding: 5px 6px; border: 1px solid #ccc; border-radius: 4px; font-size: .85rem; }

    .extras-checks { display: flex; gap: 10px; flex-wrap: wrap; margin-bottom: 14px; }
    .extras-checks label { display: flex; align-items: center; gap: 4px; font-size: .85rem; cursor: pointer; background: #f9fafb; border: 1px solid #e5e7eb; border-radius: 4px; padding: 4px 8px; }
    .extras-checks label:has(input:checked) { background: #eff6ff; border-color: #93c5fd; }

    .bundle-summary { background: #f9fafb; border-radius: 6px; padding: 10px 12px; margin-bottom: 14px; font-size: .85rem; }
    .bundle-summary .line { display: flex; justify-content: space-between; padding: 2px 0; }
    .bundle-summary .line.total { border-top: 1px solid #e5e7eb; margin-top: 6px; padding-top: 6px; font-weight: 600; }
    .bundle-summary .remove-item { background: none; border: none; color: #ef4444; cursor: pointer; font-size: .75rem; padding: 0 4px; }

    .price-display { text-align: center; padding: 16px 0 8px; }
    .price-label { font-size: .8rem; color: #888; text-transform: uppercase; letter-spacing: .08em; margin-bottom: 4px; }
    .price-value { font-size: 2.4rem; font-weight: 700; color: #16a34a; letter-spacing: -.02em; }

    .bundle-actions { display: flex; gap: 8px; justify-content: flex-end; margin-top: 8px; }

    /* Modal */
    .modal-backdrop { display: none; position: fixed; inset: 0; background: rgba(0,0,0,.4); z-index: 100; align-items: center; justify-content: center; }
    .modal-backdrop.open { display: flex; }
    .modal { background: #fff; border-radius: 10px; padding: 20px; width: 340px; max-width: 95vw; }
    .modal h3 { margin-bottom: 14px; }
    .modal-field { margin-bottom: 12px; }
    .modal-field label { display: block; font-size: .82rem; color: #555; margin-bottom: 3px; }
    .modal-field input { width: 100%; padding: 6px 8px; border: 1px solid #ccc; border-radius: 4px; font-size: .88rem; }
    .modal-field .hint { font-size: .75rem; color: #9ca3af; margin-top: 2px; }
    .modal-actions { display: flex; gap: 8px; justify-content: flex-end; margin-top: 16px; }
    .modal-error { color: #ef4444; font-size: .8rem; margin-top: -8px; margin-bottom: 8px; display: none; }
  </style>
</head>
<body>
  <h1>Bazaar Mayo — Calculadora de Precios</h1>

  <!-- SECTION 1: Config -->
  <section id="config">
    <h2>Margen de ganancia</h2>
    <label>
      <input type="number" id="margin-input" min="0" max="99" step="1" value="30">
      %
    </label>
    <span style="font-size:.8rem;color:#9ca3af;margin-left:8px;">Precio = Costo ÷ (1 - margen)</span>
  </section>

  <!-- SECTION 2: Inventory -->
  <section id="inventory">
    <h2>Inventario</h2>
    <div class="inventory-grid">
      <div>
        <h3>Productos</h3>
        <table id="products-table">
          <thead><tr><th>Nombre</th><th>Costo real</th><th></th></tr></thead>
          <tbody></tbody>
        </table>
        <button class="btn-primary add-btn" onclick="openModal('product', null)">+ Agregar producto</button>
      </div>
      <div>
        <h3>Extras</h3>
        <table id="extras-table">
          <thead><tr><th>Nombre</th><th>Costo real</th><th></th></tr></thead>
          <tbody></tbody>
        </table>
        <button class="btn-primary add-btn" onclick="openModal('extra', null)">+ Agregar extra</button>
      </div>
    </div>
  </section>

  <!-- SECTION 3: Bundle Builder -->
  <section id="bundle-section">
    <h2>Bundle Builder</h2>
    <div class="bundle-adder">
      <select id="product-select"><option value="">— Seleccionar producto —</option></select>
      <input type="number" id="qty-input" min="1" value="1">
      <button class="btn-primary" onclick="addBundleItem()">+ Agregar</button>
    </div>
    <div class="extras-checks" id="extras-checks"></div>
    <div class="bundle-summary" id="bundle-summary">
      <div class="line" style="color:#bbb;font-style:italic">Bundle vacío</div>
    </div>
    <div class="price-display">
      <div class="price-label">Precio sugerido</div>
      <div class="price-value" id="price-value">$0.00</div>
    </div>
    <div class="bundle-actions">
      <button class="btn-secondary" onclick="clearBundle()">Limpiar bundle</button>
    </div>
  </section>

  <!-- Modal -->
  <div class="modal-backdrop" id="modal-backdrop" onclick="closeModalOnBackdrop(event)">
    <div class="modal">
      <h3 id="modal-title">Agregar</h3>
      <div class="modal-field">
        <label>Nombre *</label>
        <input type="text" id="modal-name" placeholder="Ej: Tote bag">
      </div>
      <div class="modal-field">
        <label>Costo unitario * ($)</label>
        <input type="number" id="modal-unit-cost" min="0" step="0.01" placeholder="0.00">
        <div class="hint">Lo que pagaste por cada pieza (sin flete)</div>
      </div>
      <div class="modal-field">
        <label>Flete total de la remesa ($) <span style="color:#9ca3af">(opcional)</span></label>
        <input type="number" id="modal-freight" min="0" step="0.01" placeholder="0.00">
      </div>
      <div class="modal-field">
        <label>Unidades en la remesa</label>
        <input type="number" id="modal-units" min="1" step="1" placeholder="1">
        <div class="hint">Requerido si hay flete</div>
      </div>
      <div class="modal-error" id="modal-error"></div>
      <div class="modal-actions">
        <button class="btn-secondary" onclick="closeModal()">Cancelar</button>
        <button class="btn-primary" onclick="saveModalItem()">Guardar</button>
      </div>
    </div>
  </div>

  <script>
    // ── STATE ──────────────────────────────────────────────────────────
    const STORAGE_KEY = 'bazaar_state'

    let state = { products: [], extras: [], margin: 30 }
    let bundle = { items: [], extras: new Set() }
    let modalCtx = { type: null, id: null } // 'product' | 'extra', id or null for new

    function loadState() {
      try {
        const raw = localStorage.getItem(STORAGE_KEY)
        if (raw) state = JSON.parse(raw)
      } catch (e) { /* ignore corrupt data */ }
    }

    function saveState() {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(state))
    }

    function genId() {
      return Date.now().toString(36) + Math.random().toString(36).slice(2, 6)
    }

    // ── COMPUTED ──────────────────────────────────────────────────────
    function realCost(item) {
      const freight = parseFloat(item.freight) || 0
      const units   = parseFloat(item.units)   || 1
      return (parseFloat(item.unitCost) || 0) + (freight > 0 ? freight / units : 0)
    }

    function bundleTotal() {
      const itemsTotal = bundle.items.reduce((sum, bi) => {
        const p = state.products.find(p => p.id === bi.productId)
        return sum + (p ? realCost(p) * bi.qty : 0)
      }, 0)
      const extrasTotal = [...bundle.extras].reduce((sum, eid) => {
        const e = state.extras.find(e => e.id === eid)
        return sum + (e ? realCost(e) : 0)
      }, 0)
      return itemsTotal + extrasTotal
    }

    function suggestedPrice() {
      const total  = bundleTotal()
      const margin = (parseFloat(state.margin) || 0) / 100
      if (margin <= 0) return total
      if (margin >= 1) return Infinity
      return total / (1 - margin)
    }

    // ── RENDER ────────────────────────────────────────────────────────
    function fmt(n) { return '$' + n.toFixed(2) }

    function renderAll() {
      renderMargin()
      renderProductsTable()
      renderExtrasTable()
      renderProductSelect()
      renderExtrasChecks()
      renderBundleSummary()
    }

    function renderMargin() {
      document.getElementById('margin-input').value = state.margin
    }

    function renderProductsTable() {
      const tbody = document.querySelector('#products-table tbody')
      if (!state.products.length) {
        tbody.innerHTML = '<tr class="empty-row"><td colspan="3">Sin productos</td></tr>'
        return
      }
      tbody.innerHTML = state.products.map(p => `
        <tr>
          <td>${escHtml(p.name)}</td>
          <td>${fmt(realCost(p))}</td>
          <td>
            <button class="btn-edit" onclick="openModal('product','${p.id}')">✎</button>
            <button class="btn-danger" onclick="deleteItem('product','${p.id}')">✕</button>
          </td>
        </tr>`).join('')
    }

    function renderExtrasTable() {
      const tbody = document.querySelector('#extras-table tbody')
      if (!state.extras.length) {
        tbody.innerHTML = '<tr class="empty-row"><td colspan="3">Sin extras</td></tr>'
        return
      }
      tbody.innerHTML = state.extras.map(e => `
        <tr>
          <td>${escHtml(e.name)}</td>
          <td>${fmt(realCost(e))}</td>
          <td>
            <button class="btn-edit" onclick="openModal('extra','${e.id}')">✎</button>
            <button class="btn-danger" onclick="deleteItem('extra','${e.id}')">✕</button>
          </td>
        </tr>`).join('')
    }

    function renderProductSelect() {
      const sel = document.getElementById('product-select')
      const prev = sel.value
      sel.innerHTML = '<option value="">— Seleccionar producto —</option>' +
        state.products.map(p => `<option value="${p.id}">${escHtml(p.name)} (${fmt(realCost(p))})</option>`).join('')
      if (prev) sel.value = prev
    }

    function renderExtrasChecks() {
      const div = document.getElementById('extras-checks')
      if (!state.extras.length) { div.innerHTML = '<span style="color:#bbb;font-size:.82rem">Sin extras disponibles</span>'; return }
      div.innerHTML = state.extras.map(e => `
        <label>
          <input type="checkbox" value="${e.id}" ${bundle.extras.has(e.id) ? 'checked' : ''}
            onchange="toggleExtra('${e.id}', this.checked)">
          ${escHtml(e.name)} (${fmt(realCost(e))})
        </label>`).join('')
    }

    function renderBundleSummary() {
      const div = document.getElementById('bundle-summary')
      const lines = []

      bundle.items.forEach(bi => {
        const p = state.products.find(p => p.id === bi.productId)
        if (!p) return
        const cost = realCost(p) * bi.qty
        lines.push(`<div class="line">
          <span>${escHtml(p.name)} × ${bi.qty}
            <button class="remove-item" onclick="removeBundleItem('${bi.productId}')">✕</button>
          </span>
          <span>${fmt(cost)}</span>
        </div>`)
      });

      [...bundle.extras].forEach(eid => {
        const e = state.extras.find(e => e.id === eid)
        if (!e) return
        lines.push(`<div class="line">
          <span>${escHtml(e.name)} (extra)</span>
          <span>${fmt(realCost(e))}</span>
        </div>`)
      })

      if (!lines.length) {
        div.innerHTML = '<div class="line" style="color:#bbb;font-style:italic">Bundle vacío</div>'
      } else {
        const total = bundleTotal()
        div.innerHTML = lines.join('') +
          `<div class="line total"><span>Costo total</span><span>${fmt(total)}</span></div>`
      }

      const price = suggestedPrice()
      document.getElementById('price-value').textContent = isFinite(price) ? fmt(price) : '—'
    }

    // ── INVENTORY CRUD ────────────────────────────────────────────────
    function openModal(type, id) {
      modalCtx = { type, id }
      const isEdit = id !== null
      document.getElementById('modal-title').textContent =
        (isEdit ? 'Editar ' : 'Agregar ') + (type === 'product' ? 'producto' : 'extra')
      document.getElementById('modal-error').style.display = 'none'

      const collection = type === 'product' ? state.products : state.extras
      const item = isEdit ? collection.find(x => x.id === id) : null

      document.getElementById('modal-name').value      = item?.name      ?? ''
      document.getElementById('modal-unit-cost').value = item?.unitCost  ?? ''
      document.getElementById('modal-freight').value   = item?.freight   ?? ''
      document.getElementById('modal-units').value     = item?.units     ?? ''

      document.getElementById('modal-backdrop').classList.add('open')
      document.getElementById('modal-name').focus()
    }

    function closeModal() {
      document.getElementById('modal-backdrop').classList.remove('open')
    }

    function closeModalOnBackdrop(e) {
      if (e.target === document.getElementById('modal-backdrop')) closeModal()
    }

    function saveModalItem() {
      const name     = document.getElementById('modal-name').value.trim()
      const unitCost = parseFloat(document.getElementById('modal-unit-cost').value)
      const freight  = parseFloat(document.getElementById('modal-freight').value) || 0
      const units    = parseInt(document.getElementById('modal-units').value)     || 1
      const errEl    = document.getElementById('modal-error')

      if (!name) { showModalError('El nombre es requerido.'); return }
      if (isNaN(unitCost) || unitCost < 0) { showModalError('Ingresa un costo unitario válido.'); return }
      if (freight > 0 && units < 1) { showModalError('Ingresa las unidades de la remesa.'); return }

      const collection = modalCtx.type === 'product' ? state.products : state.extras
      if (modalCtx.id) {
        const item = collection.find(x => x.id === modalCtx.id)
        Object.assign(item, { name, unitCost, freight, units })
      } else {
        collection.push({ id: genId(), name, unitCost, freight, units })
      }

      saveState()
      renderAll()
      closeModal()
    }

    function showModalError(msg) {
      const el = document.getElementById('modal-error')
      el.textContent = msg
      el.style.display = 'block'
    }

    function deleteItem(type, id) {
      if (type === 'product') {
        state.products = state.products.filter(p => p.id !== id)
        bundle.items   = bundle.items.filter(bi => bi.productId !== id)
      } else {
        state.extras = state.extras.filter(e => e.id !== id)
        bundle.extras.delete(id)
      }
      saveState()
      renderAll()
    }

    // ── BUNDLE ACTIONS ────────────────────────────────────────────────
    function addBundleItem() {
      const productId = document.getElementById('product-select').value
      const qty       = parseInt(document.getElementById('qty-input').value) || 1
      if (!productId) return

      const existing = bundle.items.find(bi => bi.productId === productId)
      if (existing) { existing.qty += qty }
      else           { bundle.items.push({ productId, qty }) }

      renderBundleSummary()
    }

    function removeBundleItem(productId) {
      bundle.items = bundle.items.filter(bi => bi.productId !== productId)
      renderBundleSummary()
    }

    function toggleExtra(extraId, checked) {
      if (checked) bundle.extras.add(extraId)
      else         bundle.extras.delete(extraId)
      renderBundleSummary()
    }

    function clearBundle() {
      bundle.items  = []
      bundle.extras = new Set()
      renderExtrasChecks()
      renderBundleSummary()
    }

    // ── CONFIG EVENTS ─────────────────────────────────────────────────
    document.getElementById('margin-input').addEventListener('input', function() {
      state.margin = parseFloat(this.value) || 0
      saveState()
      renderBundleSummary()
    })

    // ── UTILS ─────────────────────────────────────────────────────────
    function escHtml(str) {
      return String(str).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;')
    }

    // ── INIT ──────────────────────────────────────────────────────────
    loadState()
    renderAll()
  </script>
</body>
</html>
```

- [ ] **Step 2: Open `index.html` in the browser and verify:**
  - 3 sections visible: Configuración, Inventario, Bundle Builder
  - Margin input shows 30%
  - Inventory tables show "Sin productos" / "Sin extras"
  - Price shows $0.00

- [ ] **Step 3: Test products CRUD**
  - Click "+ Agregar producto" → modal opens
  - Add product: "Tote bag", unitCost $25, freight $50, units 100 → Guardar
  - Table shows "Tote bag" with cost $25.50 (25 + 50/100)
  - Edit → change name → Guardar → table updates
  - Reload page → product persists (localStorage working)
  - Delete → product disappears

- [ ] **Step 4: Test extras CRUD**
  - Add extra: "Sticker", unitCost $0.50, no freight → Guardar
  - Extras table shows "Sticker" with cost $0.50
  - Extra appears as checkbox in Bundle Builder section

- [ ] **Step 5: Test bundle builder**
  - Select "Tote bag", qty 2 → click "+ Agregar"
  - Summary shows "Tote bag × 2  $51.00"
  - Check "Sticker" → summary adds "Sticker $0.50", total $51.50
  - Price updates in real-time with margin 30%: $51.50 / 0.7 = $73.57
  - Change margin to 50% → price updates to $103.00
  - Click "Limpiar bundle" → bundle resets to vacío, price $0.00

- [ ] **Step 6: Commit**

```bash
git init
git add index.html docs/
git commit -m "feat: bazaar price calculator — single-file HTML app"
```

---

## Self-Review Checklist (pre-execution)

- [x] Spec coverage: margin ✓, products CRUD ✓, extras CRUD ✓, flete distribution ✓, bundle items ✓, extras optional ✓, real-time price ✓, clear bundle ✓, localStorage persistence ✓
- [x] No placeholders or TBDs
- [x] Type consistency: `realCost()` used consistently, `bundle.extras` is a Set throughout, `genId()` used in all new-item paths
- [x] Edge cases covered: empty bundle → $0, margin 0% → price = cost, item $0 allowed, freight optional
