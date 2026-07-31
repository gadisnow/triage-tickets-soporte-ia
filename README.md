# Sistema de Triage de Tickets de Soporte con IA

Entrega Final — Ecosistema de Automatización IA Autónomo para Negocios

## 1. Caso de uso

Triage automático de tickets de soporte: un ticket entra vía webhook, la IA (GPT-4o-mini) lee el mensaje del cliente y lo clasifica por **categoría** (Bug Crítico / Duda Facturación / Feature Request / Otro) y **urgencia** (Alta / Media / Baja), generando además un resumen corto. El sistema queda listo para rutear el ticket al área correspondiente.

## 2. Stack utilizado

| Categoría | Herramienta |
|---|---|
| Orquestador | n8n |
| Base de datos | Airtable |
| Procesamiento IA | OpenAI GPT-4o-mini |
| Canal de salida | Slack (HITL) |

## 3. Arquitectura del sistema

Ver diagrama completo: [`Diagrama_Arquitectura.pdf`](./Diagrama_Arquitectura.pdf)

Flujo implementado:

Webhook (POST) → Create Record (Airtable: Tickets, Estado=Pendiente)
→ Message a Model (OpenAI GPT-4o-mini, clasificación estructurada JSON)
→ Update Record (Airtable: Categoria_IA, Urgencia_IA, Resumen_IA, Estado=Procesado por IA)
→ Respond to Webhook (200 OK)

Ruta de error diseñada: cualquier nodo que falle (API de IA caída, datos faltantes) registra el fallo en la tabla `Errores`, vinculada al ticket original mediante `multipleRecordLinks`, evitando pérdida de trazabilidad.

Punto de control humano (HITL): los tickets clasificados con `Urgencia_IA = Alta` requieren aprobación en Slack antes de notificar al área correspondiente, evitando el "efecto metralleta" de acciones automáticas sin supervisión.

## 4. Estructura de datos (Airtable)

Base: **Soporte - Sistema Triage IA** — [Ver en modo lectura](PEGAR_LINK_SHARED_VIEW_ACA)

### Tabla `Tickets`
| Campo | Tipo | Descripción |
|---|---|---|
| Ticket_ID | Autonumber | ID único |
| Cliente_Nombre | Texto | Nombre del cliente |
| Cliente_Email | Email | Email del cliente |
| Mensaje_Original | Texto largo | Mensaje entrante sin procesar |
| Categoria_IA | Single select | Clasificación generada por IA |
| Urgencia_IA | Single select | Alta / Media / Baja |
| Resumen_IA | Texto largo | Resumen generado por IA |
| Estado | Single select | Pendiente → Procesado por IA → Aprobado → Enviado / Error |
| Area_Asignada | Link a `Areas` | Relación con tabla de áreas (evita hardcodear) |
| Fecha_Creacion | Created time | Automático |
| Fecha_Procesado | Date | Cuándo terminó de procesar la IA |
| Aprobado_Por | Texto | Quién aprobó en el paso HITL |
| Slack_Thread_ID | Texto | Para mapear el hilo de Slack |
| Errores | Link a `Errores` | Relación con tabla de errores |

### Tabla `Areas` (catálogo, evita datos hardcodeados)
`Nombre_Area`, `Slack_Channel_ID`, `Email_Responsable`, `Tickets` (link inverso)

### Tabla `Errores`
`Error_ID`, `Ticket` (link), `Tipo_Error` (API IA / Datos Faltantes / Timeout / Otro), `Detalle`, `Fecha`

### Esquema JSON de integración (Webhook → n8n)
```json
{
  "cliente_nombre": "string",
  "cliente_email": "string",
  "mensaje": "string"
}
```

### Esquema JSON de salida (IA → Airtable)
```json
{
  "categoria": "Bug Crítico | Duda Facturación | Feature Request | Otro",
  "urgencia": "Alta | Media | Baja",
  "resumen": "string",
  "area_asignada": "Soporte Técnico | Facturación | Producto"
}
```

## 5. Optimización de costos

Ver matriz completa: [`Matriz_Costos.pdf`](./Matriz_Costos.pdf)

Resumen: se usa **GPT-4o-mini** para la clasificación (tarea simple y estructurada), reservando modelos más caros (Claude Sonnet) solo para tickets con contexto extenso, y **OpenAI Batch API** para reprocesamiento masivo no urgente. Ahorro estimado: ~94% vs. usar GPT-4o para todo.

## 6. Seguridad y resiliencia

- **Minimización de datos**: el prompt de IA solo recibe el mensaje del ticket, sin exponer datos de otros clientes ni credenciales.
- **Rutas de error**: fallos de API o datos faltantes (ej. ticket sin email) se registran en la tabla `Errores` en vez de romper el flujo silenciosamente.
- **Typecast controlado**: se usa validación de tipos en las escrituras a Airtable para evitar datos corruptos.
- **Human-in-the-loop**: tickets de urgencia Alta no se notifican automáticamente al cliente final sin aprobación humana vía Slack.
- **Credenciales**: tokens de Airtable y OpenAI gestionados como credenciales cifradas dentro de n8n, nunca hardcodeados en los nodos.

## 7. Dashboard de control

[Ver dashboard público](PEGAR_LINK_DASHBOARD_ACA) — vista compartida de Airtable con KPIs de tickets procesados, distribución por categoría/urgencia y tasa de errores.

## 8. Evidencias

- `evidencia_hoppscotch_200ok.png` — prueba del webhook respondiendo 200 OK
- `evidencia_airtable_ticket_procesado.png` — tickets clasificados por la IA en Airtable
- `Triage Tickets Soporte - TP Final.json` — export del workflow de n8n

## 9. Test de estrés realizado

Se probó el flujo con 5 tickets de prueba cubriendo: bug crítico (urgencia alta), duda de facturación (urgencia media/alta), feature request (urgencia baja), mensaje ambiguo, y un "camino infeliz" con dato faltante (email vacío) para validar el manejo de errores.
