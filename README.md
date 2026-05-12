# BCH — Dashboard Sala de Máquinas (El Bosque)

Dashboard de monitoreo de la planta de frío para el edificio El Bosque del Banco de Chile.

## Stack

- **Frontend:** HTML/CSS/JS vanilla (sin frameworks, para compatibilidad con ThingsBoard widgets).
- **Plataforma target:** ThingsBoard (TB) — assets, devices, telemetry, rule chains, alarm system.
- **Hosting actual:** Vercel/Netlify para preview. En producción se incrusta en widgets de TB.

## Estructura

```
.
├── index.html              # Dashboard completo (mockup con datos simulados)
├── PROJECT_CONTEXT.md      # Contexto del proyecto, decisiones, fórmulas
├── README.md               # Este archivo
└── docs/                   # (futuro) documentación técnica adicional
    ├── modbus-map.md       # Mapeo Modbus de cada punto
    ├── rule-chains.md      # Reglas TB a configurar
    └── alarms-config.md    # Configuración de alarmas
```

## Pestañas

1. **Overview** — vista alto nivel con KPIs, matriz diagnóstica, evolución mensual, heatmap de operación.
2. **Producción de Frío** — chillers + bombas CP/CC + torres con telemetría por asset.
3. **Distribución** — bombas CS al edificio.
4. **Operación · Alarmas** — alarmas activas, historial, análisis, secuencias, selectores M/A.

## Métricas clave

- **ΔT Chillers** (5–7°C óptimo) — eficiencia de producción.
- **ΔT Torres** (4–6°C óptimo) — eficiencia de rechazo térmico.
- **Operational Compliance** — score ponderado de 6 secuencias críticas.
- **Matriz Diagnóstica** — cuadrante de operación (Optimal / Hydraulic / Rejection / Chiller Issue).

## Cómo ver el dashboard

Abrir `index.html` en un navegador moderno. No requiere build ni dependencias.

## Estado

🚧 Mockup funcional con datos simulados. Listo para iterar y conectar a telemetría real de ThingsBoard.

Ver `PROJECT_CONTEXT.md` para detalles completos.
