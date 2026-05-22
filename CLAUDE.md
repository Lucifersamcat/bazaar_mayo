# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Bazaar Mayo is a zero-dependency, single-file HTML price calculator for market/bazaar sellers. It builds bundles of products and extras, distributes shipping costs across inventory, and computes a suggested sale price using a configurable profit margin.

No build step, no server, no package manager. Open `index.html` directly in a browser.

## Running the App

```bash
# Open directly in the default browser (Linux)
xdg-open index.html

# Or serve it locally to avoid any browser file:// restrictions
python3 -m http.server 8080
# then visit http://localhost:8080
```

There are no tests, linters, or CI pipelines configured.

## Architecture

The entire app lives in **`index.html`** — HTML structure, `<style>` block, and `<script>` block all in one file. The JavaScript is vanilla, no frameworks.

### State model

```js
// Persisted in localStorage under key "bazaar_state"
state = {
  products:     [{ id, name, qty, price, envioId }],
  extras:       [{ id, name, qty, price, envioId }],
  envios:       [{ id, name, cost }],
  savedBundles: [{ id, savedAt, products, extras, totalCost, suggestedPrice }],
  margin:       30   // integer 0–99
}

// In-memory only — reset on page load, never persisted
bundle = {
  products: [{ productId, qty }],
  extras:   [{ extraId, qty }]
}
```

### Key computed values

- `unitCost(item)` = `item.price / item.qty` + proportional share of its envio cost  
  The envio cost is split evenly across **all units** (products + extras) assigned to that envio via `totalUnitsInEnvio(envioId)`.
- `bundleTotal()` = sum of `unitCost(item) × qty` for every line in the bundle.
- `suggestedPrice()` = `bundleTotal() / (1 − margin/100)`.

### Data flow

1. `loadState()` reads from localStorage on init (includes migration from old `fletes`/`fleteId` field names to `envios`/`envioId`).
2. Every mutation calls `saveState()` then `renderAll()`.
3. `renderAll()` calls all sub-renders; each one reads directly from `state` and `bundle`.
4. The modal (`openModal(type, id)`) handles add/edit for all three entity types (`'product'`, `'extra'`, `'envio'`). Passing `id = null` means "new item".
5. `saveBundle()` validates stock, snapshots unit costs into `state.savedBundles`, and **deducts stock** from `state.products`/`state.extras`.

### Saved bundle snapshots

Saved bundles store a frozen copy of `{ name, unitCost }` per line so historical prices remain correct even if inventory is later edited.

### Export / Import

`exportData()` downloads `state` as a JSON file. `onImportFile()` replaces `state` entirely with the parsed JSON (requires `products`, `extras`, `envios` keys to be present).

## Docs

- `docs/superpowers/specs/` — original product design spec (Spanish)
- `docs/superpowers/plans/` — implementation plan used to build V1; V2 extended it with saved bundles, stock deduction, and envio distribution
