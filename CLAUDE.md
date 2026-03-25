# HCH Trading System — CLAUDE.md

## Idioma
**Responder SIEMPRE en español.** El usuario (Manu) es hispanohablante. Toda comunicación, comentarios en código y mensajes de commit deben ser en español.

---

## Resumen del Proyecto

Sistema de trading monolítico (single-file HTML/JS, ~12,500 líneas) para operar el patrón **Hombro-Cabeza-Hombro (HCH)** en forex. Sirve como portal personal: journal de trades, dashboard, calculadora de lotaje, checklist, arena de práctica, estadísticas y análisis.

**Archivos principales:**
- `HCH_v450.html` — Archivo principal del sistema (toda la app)
- `HCH_LiveChart_Test.html` — Test de charts en vivo con LightweightCharts (~974 líneas)

**Stack:** HTML + CSS + vanilla JavaScript. Sin frameworks, sin backend, sin build tools.

---

## Arquitectura

### Persistencia
- **localStorage es la ÚNICA capa de datos.** No hay backend ni base de datos.
- Todas las escrituras usan `safeSave(key, value)` que envuelve try/catch.
- Riesgo conocido: pérdida de datos si se limpia el navegador. Para trading en vivo se necesita backup automático (JSON export on trade close — pendiente de implementar).

### Claves de localStorage
| Clave | Contenido |
|---|---|
| `hch_trades` | Array de trades cerrados (historial) |
| `hch_open_trades` | Array de trades abiertos |
| `hch_watchlist` | Pares en vigilancia |
| `hch_capital` | Capital actual |
| `hch_checklist` / `hch_checklist2` | Estado del checklist |
| `hch_claude_key` / `hch_arena_api_key` | API key de Claude (arena) |
| `hch_arena_stats` / `hch_arena_weak` / `hch_arena_extras` | Estadísticas y datos del Arena |
| `hch_histdata_index` / `hch_hd_log` | Historical data index y log |
| `hch_rates` | Tasas de cambio cacheadas |
| `hch_td_key` | TwelveData API key |
| `hch_calc_config` | Config de calculadora |
| `hch_wf_block_tags` / `hch_wf_calc` | Tags de workflow y calculadora |
| `hch_sidebar_collapsed` | Estado del sidebar |
| `hch_punch_card` | Punch card de sesiones |
| `hch_last_scan` | Último escaneo |
| `hch_an_symbols` | Símbolos de análisis |
| `hch_fases_config` / `hch_prog_config` | Configuración de fases y programa |

### Navegación
Sistema de páginas SPA con `showPage(id)`. Helper `navTo(page)` y `navToMesa(tab)`.

**Páginas (id → sección):**
- `dashboard` — Panel principal con métricas
- `sistema` — Descripción del sistema HCH
- `cuenta` — Gestión de cuenta/capital
- `checklist` / `checklist2` — Checklists de entrada
- `lotaje` — Calculadora de lotaje
- `mesa` — Mesa de operaciones (watchlist → live → history)
- `stats` — Estadísticas avanzadas
- `analisis` — Análisis de mercado
- `arena` — Arena de práctica (quiz, escenarios, lotaje, speed, fases)
- `fases` — Fases del patrón HCH
- `noticias` — Noticias y calendario
- `estructura` — Estructura de mercado
- `patrones` — Biblioteca de patrones
- `aoi` — Áreas de Interés
- `reglas` — Reglas del sistema

### Funciones Clave
```
getCETDateStr(d) / getCETTimeStr(d) — Timezone Europe/Madrid
tNet(t) — P&L neto: (pnl) - (commission) + (swap)
getTradeState() — Cache de estado de trades (invalidar con invalidateTradeState())
getCurrentCapital() — Capital actual desde localStorage
safeSave(key, value) — Escritura segura a localStorage
fmtD(n, dec) / fmtEur(n) / fmtEurC(n) — Formateo de números y moneda
showPage(id) / navTo(page) / navToMesa(tab) — Navegación
computeDashboard() — Renderizado del dashboard principal
openTrade() / confirmCloseTrade() — Ciclo de vida de trades
renderEquityCurve() / renderDailyPnlBars() / renderHourlyAnalysis() — Charts de stats
```

### Mesa de Operaciones (Flujo)
```
Watchlist → Live (trade abierto) → History (trade cerrado)
switchMesaTab(tab) donde tab = 'watch' | 'live' | 'hist'
```

---

## Design System

- **Tema:** Oscuro estilo TradingView
- **Fuentes:** `JetBrains Mono` (datos, números, código) + `DM Sans` (texto, UI)
- **Colores CSS vars:**
  - `--bg-primary: #0a0e17` / `--bg-secondary: #111827` / `--bg-card: #1a1f2e`
  - `--accent-cyan: #00d4ff` / `--accent-green: #00ff88` / `--accent-red: #ff4444`
  - `--accent-yellow: #ffcc00` / `--accent-orange: #ff8800` / `--accent-purple: #a78bfa`
- **Border radius:** 10-12px en cards, 4-8px en badges
- **Imágenes:** Embebidas en base64 (no URLs externas)

---

## Sistema de Trading — Reglas

### Patrón HCH
- Timeframe principal: **M30 exclusivamente** para identificar el patrón
- M15, H1, H4 solo para AOI y contexto de tendencia
- Entrada en la CABEZA (máximo RR, ~25-30% sin AOI, ~70-80% con AOI)
- SL: detrás del wick de la cabeza
- BE: cuando precio supera zona del cuello. NO antes. Necesita 5-8 pips espacio en M30
- "Si no ves el HCH en 3 segundos, no es un HCH"

### Estructura de Mercado
- HCH bearish: HH+HL → LH+LL
  - Hombro izq = último HH normal
  - Cabeza = último HH definitivo
  - Cuello der = HL débil
  - Hombro der = LH (CHoCH)
  - Ruptura neckline = LL (BOS)
- Wicks para extremos, cuerpos para neckline

### Gestión de Riesgo (NO NEGOCIABLE)
- Máximo 1-2% por trade
- Lotaje = (Capital × %Riesgo × EUR/USD) ÷ (SL pips × Pip value)
- Pares correlacionados = mismo trade (3 JPY = dividir lotaje entre 3)
- Spread < mitad del SL
- Máximo 3 trades por sesión

### Horarios (CET/España)
- **Sweet hours:** 14:30-17:30 (overlap London-NY) — lotaje normal, máxima agresividad
- **Sesión B:** 19:00-22:00 — mitad de lotaje
- **Asiática:** 00:00-01:00 — JPY/AUD
- **Evitar:** 22:00 (rebalanceo) y 02:00-08:00 (liquidez mínima)

### Tags de violación de reglas
El sistema trackea violaciones con tag group "⚠️ Fuera de Reglas": `Lotaje excesivo`, `Sin SL definido`, `Revenge trade`, `FOMO entrada`.

---

## Contexto del Trader

- **Manu:** Forex trader en formación. Demo account ~500€ en Fusion Markets (MT5).
- **Objetivo:** 50-100 trades registrados antes de ir a cuenta real.
- **Plataformas:** TradingView para análisis, MT5 para ejecución.
- **Trainer de referencia:** Alex (metodología "Perfect Trade").
- **La IA para detección de patrones fue probada y descartada** — no puede identificar HCH visual desde datos OHLC.

---

## Bugs Conocidos / Deuda Técnica

- Navegación usa índices hardcodeados → frágil al añadir/quitar páginas
- Posible HTML malformado en onclick (revisar ~línea 2029 de versiones anteriores)
- `showPage('registrov2')` referencia div inexistente (legacy)
- localStorage como única persistencia es inaceptable para trading real
- El archivo es monolítico (~12.5K líneas) — difícil de navegar, considerar modularización eventual

---

## Cómo Trabajar en Este Proyecto

1. **Siempre leer el archivo antes de editar** — el estado actual es la verdad.
2. **Usar `grep -n` extensivamente** para localizar funciones, IDs, y contexto antes de cambios.
3. **Probar cambios con grep** después de editar para verificar consistencia.
4. **No romper el flujo de navegación** — verificar que `showPage()` targets existen.
5. **Mantener el design system** — usar las CSS vars existentes, nunca colores hardcodeados nuevos.
6. **Cálculos financieros siempre con `tNet(t)`** para P&L neto.
7. **Timezone siempre `getCETDateStr()` / `getCETTimeStr()`**.
8. **Invalidar cache después de cambios en trades:** llamar `invalidateTradeState()`.
9. **Los items del checklist deben ser binarios** (sí/no) y referir solo a price action cerrada.
10. **Un trade rentable fuera de reglas sigue siendo gambling** — la disciplina es prioridad.
