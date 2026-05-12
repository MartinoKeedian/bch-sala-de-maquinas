# BCH — Dashboard Sala de Máquinas

> Proyecto: Dashboard de monitoreo de planta de frío (cooling plant) para el edificio El Bosque del Banco de Chile.
> Stack target: ThingsBoard (TB) con widgets HTML/JS custom.
> Estado actual: mockup HTML funcional con datos simulados, listo para iterar y conectar a TB.

---

## 1. Contexto del cliente

- **Cliente:** Banco de Chile (BCH).
- **Sitio:** Edificio El Bosque (oficinas, edificio antiguo retrofiteado con BMS).
- **Alcance fase 1:** Sala de máquinas (planta de frío central).
- **Fase 2 (futuro):** Dashboards por piso.
- **Plataforma:** ThingsBoard (asume configuración nativa: assets, devices, telemetry, rule chains, alarm system).

---

## 2. Inventario de telemetría disponible

Todo viene del gateway Modbus del BMS. Esquema:

### Estados ON/OFF (binarios, dirección dispositivo 22)

| Punto | Address Modbus |
|---|---|
| Bba 1/2/3 CP (primario) | 4, 5, 6 |
| Bba 1/2/3 CC (condensador) | 7, 8, 9 |
| Bba 1/2/3 CS (secundario) | 10, 11, 12 |
| Torre 1, Torre 2 | 13, 14 |
| Chiller 1, Chiller 2 | (pendiente de mapear) |
| Vex 16, Vex 17 | 19, 20 |
| Humedad sector 1, sector 2 | 47, 48 |

### Señales variables (Holding Register, Integer)

| Punto | Address Modbus |
|---|---|
| Impulsión Chillers 1 y 2 | 2 |
| Retorno Chillers 1 y 2 | 3 |
| Entrada TE 1 | 15 |
| Salida TE 1 | 16 |
| Entrada TE 2 | 17 |
| Salida TE 2 | 18 |
| Variadores Torre 1, 2 | (pendiente) |
| Variadores CS 1, 2 | (pendiente) |

### Fallas térmicas (FT — binarias)

| Punto | Address |
|---|---|
| FT BBA 1/2/3 CP | 23, 24, 25 |
| FT BBA 1/2/3 CC | 26, 27, 28 |
| FT BBA 1/2/3 CS | 29, 30, 31 |
| FT Vex 16, 17 | 21, 22 |
| FT TE 1, 2 | 32, 33 |

### Selectores Manual/Automático

| Punto | Address |
|---|---|
| SELECTOR BBA 1/2/3 CP | 34, 35, 36 |
| SELECTOR BBA 1/2/3 CC | 37, 38, 39 |
| SELECTOR BBA 1/2/3 CS | 40, 41, 42 |
| SELECTOR TE 1, 2 | 43, 44 |
| SELECTOR Vex 16, 17 | 45, 46 |

**Convención:** `1 = ON / AUTO`, `0 = OFF / MANUAL`. Confirmar con cliente la escala de temperaturas (factor decimal).

---

## 3. Arquitectura del dashboard

### Pestañas (top nav)

1. **Overview** — vista alto nivel ejecutiva.
2. **Producción de Frío** — chillers + bombas CP/CC + torres + telemetría por asset.
3. **Distribución** — bombas CS (al edificio).
4. **Operación · Alarmas** — alarmas activas, historial, análisis, secuencias, selectores M/A.

### Por qué esta estructura (clave para no equivocarse)

No se sabe con certeza si la planta tiene **manifold compartido** (bombas como pool intercambiable) o **sistemas independientes** (Sistema 1 / Sistema 2). El edificio es antiguo y la cantidad de bombas (3 de cada tipo para 2 chillers) sugiere manifold con N+1, pero hay que confirmar con el cliente.

La estructura **"Producción de Frío" + "Distribución"** funciona en ambos casos: agrupa por **función térmica** (no por topología física), evitando inventar arquitectura. Si se confirma sistemas independientes, basta con reordenar las filas de la tabla en grupos visuales (Sistema 1 / Sistema 2 / Común).

### Header global

- **Period picker global:** Hoy · 7 días · 30 días · 1 año · Personalizado (date range con presets).
- Cambia TODOS los KPIs/widgets de TODAS las pestañas al mismo tiempo.
- **Badge de alarmas** (rojo pulsante) que aparece si hay alarmas activas. Click → va a pestaña Operación.
- **Live indicator** + timestamp.

---

## 4. Métricas reales (NO inventar otras)

### ΔT Chillers
```
ΔT Chiller = Retorno − Impulsión
Válido solo si: al menos una bomba CP en ON
Banda óptima: 5–7 °C
Warning: 3–5 °C
Crítico: < 3 °C
Compliance: % tiempo dentro de banda óptima sobre tiempo válido
```

### ΔT Torres
```
ΔT Torre = Entrada − Salida
Válido solo si: torre ON Y al menos una bomba CC en ON
Banda óptima: 4–6 °C
Warning: 2–4 °C
Crítico: < 2 °C
```

### Operational Compliance (score ponderado de secuencias)

```
Pesos:
- Chiller → CP: 25% (crítica)
- Chiller → CC: 20% (crítica)
- Chiller → Torre: 20% (crítica)
- CP → CS: 15% (alta)
- Torre → CC: 10% (alta)
- System OFF Consistency: 10% (alta)

Score = Σ(weight × seq_compliance) / Σ(weights aplicables)
```

### Matriz Diagnóstica (4 cuadrantes)

Estado simultáneo de ΔT Chiller × ΔT Torre:
- **System Optimal**: ambos en banda óptima.
- **Hydraulic Issue**: Chiller bajo + Torre OK (problema CHW).
- **Rejection Issue**: Chiller OK + Torre baja (problema torre).
- **Chiller Issue**: ambos bajos (carga baja o problema combinado).

Output: % del tiempo del período en cada cuadrante. Cuadrantes son clickeables → filtran el heatmap.

### Heatmap "Planta activa"

```
Eje X: hora del día (0-23)
Eje Y: día de la semana (Lun-Dom)
Color: % del tiempo en que al menos un chiller estuvo en ON
```

Permite detectar: operación fuera de horario, fines de semana, anomalías nocturnas.

### Métricas que NO usamos (decisión explícita)

- **HRR (Heat Rejection Ratio)** — no es estándar de industria, se descartó.
- **"Score de Eficiencia" agregado** — invento sin base, descartado.
- **Approach Térmico** — requiere sensor de bulbo húmedo ambiental que NO tenemos.
- **Cooling Plant Efficiency real** — requiere medidores de flujo y energía que NO tenemos.

---

## 5. Sistema de alarmas

Usamos el sistema **nativo de ThingsBoard** (objeto Alarm con type/severity/status/originator/start_ts/end_ts/ack_ts/clear_ts/details).

### Decisión clave: modo "mixto sin requerir ack manual"

- El operador **puede** hacer ack (TB lo permite), pero los KPIs NO dependen de eso.
- Las alarmas se levantan y resuelven automáticamente según telemetría.
- Mostramos badge "ACK" sutil si fue reconocida, sin protagonismo.

### Tipos de alarmas configurar en TB (vía Rule Chains)

| Type | Origen | Severity | Condición |
|---|---|---|---|
| `FT BBA N CP` | Bomba CP N | CRITICAL | Señal FT = 1 |
| `FT BBA N CC` | Bomba CC N | CRITICAL | Señal FT = 1 |
| `FT BBA N CS` | Bomba CS N | CRITICAL | Señal FT = 1 |
| `FT VEX N` | Vex N | MAJOR | Señal FT = 1 |
| `FT TE N` | Torre N | CRITICAL | Señal FT = 1 |
| `ΔT-LOW Chiller N` | Chiller N | MAJOR | ΔT < 5°C por >15min Y chiller ON |
| `ΔT-LOW Torre N` | Torre N | MAJOR | ΔT < 4°C por >15min Y torre ON |
| `MANUAL — <Equipo>` | Asset | WARNING | Selector en MAN > 1h |
| `SEQ-FAIL Chiller→CP` | Sistema | CRITICAL | Chiller ON Y todas CP OFF |
| `SEQ-FAIL Chiller→CC` | Sistema | CRITICAL | Chiller ON Y todas CC OFF |
| `SEQ-FAIL Chiller→Torre` | Sistema | CRITICAL | Chiller ON Y todas Torres OFF |

---

## 6. Telemetría derivada (Rule Chain JS)

Cada device tiene estas keys derivadas calculadas en TB:

```javascript
// System ON: sistema produciendo frío
system_on = (chiller_1_on == 1) || (chiller_2_on == 1) ? 1 : 0

// CP any ON
cp_any_on = (bba_1_cp || bba_2_cp || bba_3_cp) ? 1 : 0

// ΔT Chiller (solo válido si hay flujo CHW)
delta_t_chw = cp_any_on ? (chw_return - chw_supply) : null
delta_t_chw_valid = cp_any_on ? 1 : 0
in_compliance_chw = (delta_t_chw_valid && delta_t_chw >= 5 && delta_t_chw <= 7) ? 1 : 0

// ΔT Torre N
cc_any_on = (bba_1_cc || bba_2_cc || bba_3_cc) ? 1 : 0
delta_t_torre_N = (torre_N_on && cc_any_on) ? (te_N_in - te_N_out) : null

// Secuencias (1 = cumplida cuando aplica, null si no aplica)
seq_chiller_cp = chiller_on ? (cp_any_on ? 1 : 0) : null
seq_chiller_cc = chiller_on ? (cc_any_on ? 1 : 0) : null
seq_chiller_tower = chiller_on ? (any_tower_on ? 1 : 0) : null
seq_cp_cs = cp_any_on ? (cs_any_on ? 1 : 0) : null
seq_tower_cc = any_tower_on ? (cc_any_on ? 1 : 0) : null
seq_system_off = (!chiller_on) ? (cp_any_on || cc_any_on || cs_any_on || any_tower_on ? 0 : 1) : null
```

### Atributos server-side configurables

```
target_dt_chw_min = 5
target_dt_chw_max = 7
target_dt_torre_min = 4
target_dt_torre_max = 6
low_delta_t_duration = 15  // min
manual_alarm_duration = 60  // min
```

---

## 7. Estado actual del proyecto

### ✅ Hecho

- Mockup HTML funcional con datos simulados (`index.html` / `bch_sala_maquinas.html`).
- 4 pestañas funcionando con navegación.
- Period picker global con 5 opciones (incluye date range modal).
- KPIs duales (promedio + compliance) con tooltips informativos.
- Matriz Diagnóstica clickeable que filtra heatmap.
- Heatmaps día×hora (planta activa y alarmas).
- Tabla unificada de assets en Producción con flow diagram y filtros.
- Selección múltiple de assets (Ctrl+click) y comparación lado a lado en telemetría.
- Telemetría time series por asset con overlay ON/OFF.
- Lista de alarmas activas + historial + ranking + donut severidad.
- Selectores M/A en vivo.
- Tooltips JS-based con auto-posicionamiento.
- Responsive (4 breakpoints).

### 🔜 Pendiente (Fase implementación TB)

- Confirmar mapeo de chillers en Modbus (no tienen address asignada).
- Confirmar factor decimal de temperaturas (Integer Holding → °C real).
- Confirmar bandas óptimas con cliente (5-7 / 4-6 son referenciales).
- Confirmar topología hidráulica (manifold vs sistemas independientes).
- Generar JSON exportable de Rule Chains.
- Generar JSON exportable de widgets TB.
- Configurar device profiles en TB.
- Configurar reglas de alarmas en TB.
- Conectar telemetría real al dashboard.

---

## 8. Decisiones de diseño tomadas

| # | Decisión | Por qué |
|---|---|---|
| 1 | Period picker global único (no por pestaña) | Evita confusión, todas las vistas respiran al mismo período |
| 2 | Métricas reales solamente (no scores inventados) | El ejecutivo no necesita una métrica diferente; necesita las mismas con vista agregada |
| 3 | Pestañas por función térmica (no por sistema físico) | Funciona haya manifold o sistemas independientes |
| 4 | Tooltips con sistema JS (no CSS pseudo-elementos) | Más confiables, no se cortan ni salen de pantalla |
| 5 | Comparación múltiple en telemetría | Comparar Chiller 1 vs 2, T1 vs T2, etc — diagnóstico real |
| 6 | Heatmap día×hora con métrica "% tiempo ON" | La pregunta del ejecutivo es "cuándo opera", no "qué tan cargada" |
| 7 | Sistema de alarmas: ciclo de vida automático sin ack obligatorio | No depender de operadores; mostrar lo que la telemetría dice |
| 8 | Specs en nombres de assets (modelo + capacidad) | Contexto inmediato sin abrir detalle |
| 9 | Strip de alarmas en Overview + badge en header | Visibilidad cruzada de problemas activos |
| 10 | Tabla con `data-row-values` por período | Métricas de tabla consistentes con period picker |

---

## 9. Sobre integración con Claude

- **Hoy:** Claude se usa en chat (este chat) y vía Claude Code para generar artefactos.
- **No existe** MCP server oficial para ThingsBoard.
- **Futuro posible:** construir MCP server custom envolviendo la REST API de TB para que Claude pueda leer estado de la planta en tiempo real y ayudar a operaciones.

---

## 10. Convenciones de código del HTML actual

- CSS variables en `:root` para todos los colores (`--bg-page`, `--ok`, `--warn`, `--crit`, etc).
- Fuentes: Manrope (display/body) + JetBrains Mono (data).
- Estructura: header → tabs → tab-content (4 secciones).
- JS organizado por bloques: DATA MODEL → GENERATORS → RENDERING → INTERACTIONS → TOOLTIPS → INIT.
- Sin frameworks (vanilla HTML/CSS/JS) para máxima compatibilidad con TB widgets.
- Tooltips usan `<span class="tip" data-tip="...">` con auto-positioning.
- Cada asset tiene un `data-asset-id` único.
- Cada fila de tabla con métricas por período tiene `data-row-values` (JSON serializado).
