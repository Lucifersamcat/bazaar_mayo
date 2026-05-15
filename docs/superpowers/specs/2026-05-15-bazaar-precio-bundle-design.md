# Bazaar Mayo — Calculadora de Precio de Venta

**Fecha:** 2026-05-15  
**Tipo:** HTML standalone (single file)  
**Uso:** Personal, pre-bazaar, preparación de bundles

---

## Objetivo

Herramienta personal para calcular el **precio de venta sugerido** de productos y bundles en un bazaar. El usuario arma bundles con items de su inventario, agrega extras opcionales, ajusta su margen de ganancia, y el app devuelve un precio sugerido en tiempo real.

---

## Fórmula de Precio

```
costo_total = Σ(items del bundle × cantidad) + Σ(extras seleccionados) + envío (si aplica)
precio_sugerido = costo_total / (1 - margen%)
```

El margen se aplica al total del bundle, no por item individual (resultado matemáticamente equivalente, más simple).

---

## Datos

### Producto (inventario)
| Campo | Tipo | Notas |
|---|---|---|
| nombre | string | requerido |
| costo_unitario | número | costo por pieza, sin flete |
| flete_remesa | número | opcional, flete total de la remesa |
| unidades_remesa | número | requerido si hay flete |

`costo_real = costo_unitario + (flete_remesa / unidades_remesa)`

### Extra (consumibles opcionales)
| Campo | Tipo | Notas |
|---|---|---|
| nombre | string | sticker, bolsa, tarjeta, regalo, etc. |
| costo_unitario | número | costo por pieza, sin flete |
| flete_remesa | número | opcional |
| unidades_remesa | número | requerido si hay flete |

`costo_real = costo_unitario + (flete_remesa / unidades_remesa)`

### Configuración global
| Campo | Tipo | Notas |
|---|---|---|
| margen | número (%) | 0–99%, editable en cualquier momento |

> El envío al cliente NO es un campo separado — el flete de adquisición ya está distribuido en el costo unitario de cada item.

### Persistencia
- Todo guardado en `localStorage` automáticamente al editar.
- El bundle en progreso es temporal (no se guarda entre sesiones).

---

## Layout — Una sola pantalla

```
┌─────────────────────────────────────────┐
│  CONFIGURACIÓN GLOBAL                   │
│  Margen de ganancia: [___]%             │
├─────────────────────────────────────────┤
│  INVENTARIO                             │
│  Productos: [tabla] [+ Agregar]         │
│  Extras:    [tabla] [+ Agregar]         │
├─────────────────────────────────────────┤
│  BUNDLE BUILDER                         │
│  [Selector producto] [cantidad] [+ Add] │
│  Extras: □ Sticker  □ Bolsa  □ Tarjeta  │
│                                         │
│  Resumen:                               │
│  • Producto A × 2   $XX.XX             │
│  • Sticker          $X.XX              │
│  ─────────────────────────────         │
│  Costo total:       $XX.XX             │
│  PRECIO SUGERIDO:   $XX.XX  ← grande   │
│                                         │
│  [Limpiar bundle]                       │
└─────────────────────────────────────────┘
```

---

## Comportamiento

- **Tiempo real:** el precio sugerido se recalcula automáticamente con cada cambio (item, cantidad, extra, margen).
- **Bundle vacío:** muestra $0.00.
- **Margen 0%:** precio sugerido = costo total (sin ganancia).
- **Flete opcional:** si no se ingresa flete en un item, `costo_real = costo_unitario`.
- **Items con costo $0:** permitidos (regalos ya en stock).
- **Sin historial:** no se guardan bundles anteriores.
- **Sin multi-usuario:** uso personal únicamente.

---

## Fuera de alcance

- Historial de bundles guardados
- Control de stock / alertas de inventario
- Cálculo de envío al cliente final
- Múltiples usuarios o sesiones
- Exportar a PDF/Excel
