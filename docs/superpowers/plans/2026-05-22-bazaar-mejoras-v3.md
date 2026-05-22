# Bazaar Mayo — Plan de Mejoras V3

> **Para agentes:** Usa el skill `superpowers:executing-plans` para implementar tarea por tarea con checkboxes.

**Objetivo:** Corregir debilidades críticas de integridad de datos, mejorar la UX de flujos destructivos, y agregar capacidades de negocio que el ciclo de uso real demanda.

**Arquitectura:** Sigue siendo un único `index.html` sin dependencias. Todos los cambios son HTML/CSS/JS vanilla.

---

## Prioridades

| Fase | Categoría | Impacto | Esfuerzo |
|------|-----------|---------|----------|
| 1 | Integridad de datos | Crítico | Bajo |
| 2 | UX / flujos destructivos | Alto | Bajo |
| 3 | Persistencia de bundle | Alto | Bajo |
| 4 | Funcionalidad de negocio | Medio | Medio |
| 5 | Calidad técnica | Bajo | Bajo |

---

## Fase 1 — Integridad de datos

### Tarea 1.1 — Confirmación antes de guardar bundle

**Problema:** `saveBundle()` deduce stock inmediatamente sin confirmación. Un clic accidental es irreversible.

**Cambio:** Antes de ejecutar `saveBundle()`, mostrar un diálogo nativo `confirm()` con un resumen: nombre y cantidad de cada item, y total de costo/precio sugerido.

```js
function saveBundle() {
  // ...validaciones existentes...
  const lines = [
    ...bundle.products.map(bi => `• ${p.name} × ${bi.qty}`),
    ...bundle.extras.map(bi => `• ${e.name} × ${bi.qty}`),
    ``,
    `Precio sugerido: ${fmt(suggestedPrice())}`,
    ``,
    `¿Guardar y descontar stock?`
  ].join('\n')
  if (!confirm(lines)) return
  // ...resto de la lógica...
}
```

- [ ] Agregar `confirm()` con resumen en `saveBundle()` antes de modificar state
- [ ] Verificar que cancelar no modifica nada

---

### Tarea 1.2 — Confirmación antes de eliminar items e inventario

**Problema:** `deleteItem()` y `deleteSavedBundle()` eliminan sin confirmación.

**Cambio:** Agregar `confirm()` con nombre del item en ambas funciones.

```js
function deleteItem(type, id) {
  const col = type === 'product' ? state.products : type === 'extra' ? state.extras : state.envios
  const item = col.find(x => x.id === id)
  if (!confirm(`¿Eliminar "${item.name}"?`)) return
  // ...resto existente...
}

function deleteSavedBundle(id) {
  if (!confirm('¿Eliminar este bundle guardado?')) return
  // ...resto existente...
}
```

- [ ] `deleteItem()` pide confirmación con nombre
- [ ] `deleteSavedBundle()` pide confirmación

---

### Tarea 1.3 — Advertencia al eliminar un envío con items asignados

**Problema:** Borrar un envío que tiene productos/extras asignados cambia silenciosamente el costo unitario de esos items.

**Cambio:** Al intentar eliminar un envío, detectar si hay items asignados y advertir cuántos y cuáles se verán afectados.

```js
function deleteItem(type, id) {
  if (type === 'envio') {
    const affected = [...state.products, ...state.extras].filter(x => x.envioId === id)
    if (affected.length > 0) {
      const names = affected.map(x => x.name).join(', ')
      if (!confirm(`Este envío está asignado a: ${names}.\nAl eliminarlo, esos items perderán el costo de envío. ¿Continuar?`)) return
    }
  }
  // ...resto existente...
}
```

- [ ] Detectar items afectados antes de eliminar envío
- [ ] Mostrar confirmación con lista de items afectados

---

## Fase 2 — Alertas en tiempo real al construir bundle

### Tarea 2.1 — Advertir stock insuficiente al agregar al bundle

**Problema:** Se puede agregar más cantidad al bundle que el stock disponible. El error solo aparece al intentar guardar.

**Cambio:** En `addToBundle()`, después de actualizar `bundle`, verificar si algún item supera su stock y mostrar `bundle-error` en tiempo real.

```js
function addToBundle(type) {
  // ...lógica existente de agregar...
  checkBundleStockWarnings()
  renderBundleSummary()
}

function checkBundleStockWarnings() {
  for (const bi of bundle.products) {
    const p = state.products.find(p => p.id === bi.productId)
    if (p && bi.qty > parseInt(p.qty)) {
      showBundleError(`Stock insuficiente: "${p.name}" (tienes ${p.qty}, necesitas ${bi.qty})`)
      return
    }
  }
  for (const bi of bundle.extras) {
    const e = state.extras.find(e => e.id === bi.extraId)
    if (e && bi.qty > parseInt(e.qty)) {
      showBundleError(`Stock insuficiente: "${e.name}" (tienes ${e.qty}, necesitas ${bi.qty})`)
      return
    }
  }
  clearBundleError()
}
```

Llamar `checkBundleStockWarnings()` también desde `removeFromBundle()`.

- [ ] Implementar `checkBundleStockWarnings()`
- [ ] Llamarla en `addToBundle()` y `removeFromBundle()`
- [ ] El botón "Guardar bundle" se deshabilita visualmente si hay warning de stock

---

## Fase 3 — Persistencia del bundle en progreso

### Tarea 3.1 — Guardar bundle activo en localStorage

**Problema:** El bundle en construcción se pierde si el usuario cierra o refresca la pestaña.

**Cambio:** Persistir `bundle` en localStorage bajo una clave separada `bazaar_bundle`. Cargar al inicio. Limpiar al guardar o al hacer "Limpiar bundle".

```js
const BUNDLE_KEY = 'bazaar_bundle'

function saveBundleState() {
  localStorage.setItem(BUNDLE_KEY, JSON.stringify(bundle))
}

function loadBundleState() {
  try {
    const raw = localStorage.getItem(BUNDLE_KEY)
    if (raw) {
      const b = JSON.parse(raw)
      bundle.products = b.products || []
      bundle.extras   = b.extras   || []
      // Limpiar referencias a items que ya no existen en inventario
      bundle.products = bundle.products.filter(bi => state.products.find(p => p.id === bi.productId))
      bundle.extras   = bundle.extras.filter(bi => state.extras.find(e => e.id === bi.extraId))
    }
  } catch (e) {}
}
```

Llamar `saveBundleState()` en `addToBundle()`, `removeFromBundle()`, `clearBundle()`.
Llamar `loadBundleState()` en init, después de `loadState()`.

- [ ] Implementar `saveBundleState()` y `loadBundleState()`
- [ ] Llamar `saveBundleState()` en cada mutación del bundle
- [ ] Limpiar `BUNDLE_KEY` de localStorage al guardar (`saveBundle()`) y al limpiar (`clearBundle()`)
- [ ] Filtrar en `loadBundleState()` referencias a items eliminados del inventario

---

## Fase 4 — Funcionalidad de negocio

### Tarea 4.1 — Cargar bundle guardado en el builder

**Problema:** Para repetir un bundle similar hay que armarlo desde cero.

**Cambio:** Agregar botón "↩ Cargar" en cada `saved-bundle-card`. Al hacer clic, pregunta si quiere reemplazar el bundle actual, y si acepta carga los items (solo los que aún existen en inventario con suficiente stock).

```js
function loadSavedBundle(id) {
  const sb = state.savedBundles.find(sb => sb.id === id)
  if (!sb) return
  if ((bundle.products.length || bundle.extras.length) &&
      !confirm('Reemplazar el bundle actual con este?')) return

  bundle.products = []
  bundle.extras   = []

  sb.products.forEach(pi => {
    const p = state.products.find(p => p.id === pi.productId)
    if (p && parseInt(p.qty) >= pi.qty) bundle.products.push({ productId: pi.productId, qty: pi.qty })
  })
  sb.extras.forEach(xi => {
    const e = state.extras.find(e => e.id === xi.extraId)
    if (e && parseInt(e.qty) >= xi.qty) bundle.extras.push({ extraId: xi.extraId, qty: xi.qty })
  })

  saveBundleState()
  checkBundleStockWarnings()
  renderBundleSummary()
  // Scroll hacia arriba al bundle builder
  document.querySelector('#bundle-section, section:has(#bundle-summary)').scrollIntoView({ behavior: 'smooth' })
}
```

Agregar botón en la card:
```html
<button class="btn-secondary" onclick="loadSavedBundle('${sb.id}')">↩ Cargar</button>
```

- [ ] Implementar `loadSavedBundle(id)`
- [ ] Agregar botón "↩ Cargar" en `renderSavedBundles()`
- [ ] Confirmar reemplazo si hay bundle activo
- [ ] Omitir silenciosamente items sin stock suficiente e indicar cuáles se omitieron

---

### Tarea 4.2 — Operación de restock

**Problema:** Al comprar una nueva remesa del mismo producto, el usuario tiene que editar el item y sumar manualmente al qty, perdiendo el contexto del lote anterior.

**Cambio:** Agregar botón "＋ Restock" en la tabla de inventario (junto a editar/eliminar). Abre un modal simplificado que pide: cantidad adicional, precio pagado por esa cantidad, envío (opcional). Los valores se suman al item existente (qty acumulado, price acumulado, promediando el costo unitario correctamente).

Fórmula de merge de lotes:
```
nuevo_qty   = qty_actual + qty_nuevo
nuevo_price = price_actual + price_nuevo  // suma directa de lo pagado
// unitCost = nuevo_price / nuevo_qty  (se recalcula automáticamente)
```

El `envioId` del restock puede ser diferente al del lote original, pero como hay un solo `envioId` por item, el restock fuerza elegir cuál envío aplica. Solución simple: el restock crea un **nuevo item** con el mismo nombre pero `(lote 2)` sufijo, que el usuario puede renombrar. Más honesto que mezclar lotes.

- [ ] Agregar botón "＋ Restock" en tablas de productos y extras
- [ ] El botón abre el modal de agregar pero con el nombre pre-llenado y sufijo " (lote 2)" autodetectado
- [ ] Documentar en la UI por qué se crea un item nuevo en lugar de editar el existente

---

## Fase 5 — Calidad técnica

### Tarea 5.1 — Cachear `totalUnitsInEnvio` por render

**Problema:** `unitCost(item)` llama a `totalUnitsInEnvio()` que itera todo el inventario. Se invoca N veces por render (una por item visible en tablas + selects + bundle summary).

**Cambio:** Construir un Map `envioUnitsCache` al inicio de cada `renderAll()` y usarlo en `unitCost()`.

```js
let envioUnitsCache = new Map()

function buildEnvioCache() {
  envioUnitsCache = new Map()
  for (const e of state.envios) {
    const total = [...state.products, ...state.extras]
      .filter(x => x.envioId === e.id)
      .reduce((sum, x) => sum + (parseFloat(x.qty) || 0), 0)
    envioUnitsCache.set(e.id, total)
  }
}

function unitCost(item) {
  const base = (parseFloat(item.price) || 0) / (parseFloat(item.qty) || 1)
  if (!item.envioId) return base
  const envio = state.envios.find(e => e.id === item.envioId)
  if (!envio) return base
  const total = envioUnitsCache.get(item.envioId) || 0
  return base + (total > 0 ? (parseFloat(envio.cost) || 0) / total : 0)
}

function renderAll() {
  buildEnvioCache()  // ← primera línea
  // ...resto existente...
}
```

- [ ] Implementar `buildEnvioCache()` y `envioUnitsCache`
- [ ] Llamar `buildEnvioCache()` al inicio de `renderAll()`
- [ ] Reemplazar `totalUnitsInEnvio()` en `unitCost()` por lectura del cache
- [ ] Mantener `totalUnitsInEnvio()` solo donde se necesita el valor en tiempo real (modal de computed-cost)

---

### Tarea 5.2 — Validación profunda en importación

**Problema:** `onImportFile()` solo verifica que existan las llaves raíz. Un JSON con items malformados causa errores silenciosos en render.

**Cambio:** Validar que cada item tenga `id`, `name` y los campos numéricos esperados. Sanitizar valores inesperados en lugar de fallar.

```js
function validateImport(parsed) {
  if (!Array.isArray(parsed.products) || !Array.isArray(parsed.extras) || !Array.isArray(parsed.envios)) return false
  const validItem = x => x && typeof x.id === 'string' && typeof x.name === 'string'
  return parsed.products.every(validItem) && parsed.extras.every(validItem) && parsed.envios.every(validItem)
}
```

- [ ] Implementar `validateImport(parsed)`
- [ ] Reemplazar el `throw new Error()` actual por `if (!validateImport(parsed)) { alert(...); return }`
- [ ] Mostrar mensaje de error más descriptivo en el `alert` de importación fallida

---

### Tarea 5.3 — Layout responsive para móvil

**Problema:** El grid principal de 2 columnas (`app-layout`) no colapsa en pantallas pequeñas.

**Cambio:** Agregar media query para `app-layout` y ajustar `overflow` del body.

```css
@media (max-width: 768px) {
  body { height: auto; overflow: auto; }
  .app-layout { grid-template-columns: 1fr; }
  .panel { overflow-y: visible; }
}
```

- [ ] Agregar media query al `<style>` del `index.html`
- [ ] Verificar en viewport 375px (iPhone SE) que ambas columnas se apilan
- [ ] Verificar que el scroll funciona correctamente en móvil

---

### Tarea 5.4 — Reemplazar `execCommand` deprecado

**Problema:** El fallback de `copyBundle()` usa `document.execCommand('copy')` que está deprecado.

**Cambio:** Si `navigator.clipboard` no está disponible, mostrar el texto en un `<textarea>` seleccionado dentro de un modal para que el usuario lo copie manualmente, en lugar de usar `execCommand`.

```js
function copyBundle(id) {
  // ...generación de `text` existente...
  if (navigator.clipboard?.writeText) {
    navigator.clipboard.writeText(text).then(() => setBtn('✓ Copiado')).catch(() => showCopyFallback(text))
  } else {
    showCopyFallback(text)
  }
}

function showCopyFallback(text) {
  // Reusar el modal existente: mostrar textarea con el texto pre-seleccionado
  // El usuario hace Ctrl+C manualmente
}
```

- [ ] Eliminar el bloque `execCommand` de `copyBundle()`
- [ ] Implementar `showCopyFallback(text)` con textarea en el modal existente
- [ ] Agregar instrucción visual "Selecciona todo y copia (Ctrl+A, Ctrl+C)"

---

## Orden de implementación sugerido

1. Fase 1 completa (confirmaciones) — crítico, mínimo esfuerzo
2. Tarea 2.1 (warning stock en tiempo real) — alto valor, una sola función
3. Tarea 3.1 (persistir bundle) — previene frustración
4. Tarea 5.3 (responsive móvil) — una sola media query
5. Tarea 5.1 (cache envio) — limpieza interna
6. Tarea 4.1 (cargar bundle guardado) — feature nueva
7. Tarea 5.2 (validación import) — robustez
8. Tarea 5.4 (clipboard) — edge case menor
9. Tarea 4.2 (restock) — feature nueva más compleja

---

## Self-Review Checklist

- [x] Todas las tareas de Fase 1 son no-breaking (solo agregan `confirm()` antes de lógica existente)
- [x] `loadBundleState()` filtra referencias huérfanas — no puede romper inventario limpio
- [x] `buildEnvioCache()` es puro — no modifica state, solo lee
- [x] Restock como nuevo item evita complejidad de merge de lotes con envíos distintos
- [x] Responsive CSS es aditivo — no afecta desktop
