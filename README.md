# Sistema de Triage de Tickets de Soporte con IA

Entrega Final — Ecosistema de Automatización IA Autónomo para Negocios

## 1. Caso de uso

Triage automático de tickets de soporte: un ticket entra vía webhook, la IA (**Claude Haiku 4.5**) lee el mensaje del cliente y lo clasifica por **categoría** (Bug Crítico / Duda Facturación / Feature Request / Otro), **urgencia** (Alta / Media / Baja) y **área responsable** (Soporte Técnico / Facturación / Producto), generando además un resumen corto. El sistema aprueba automáticamente los tickets de urgencia Media/Baja y exige aprobación humana para los de urgencia Alta, y en ambos casos **notifica el resultado final al canal de Slack del área correspondiente** — cerrando el ciclo de principio a fin, sin intervención manual salvo el punto de control HITL.

## 2. Stack utilizado

| Categoría | Herramienta |
|---|---|
| Orquestador | n8n |
| Base de datos | Airtable |
| Procesamiento IA | Anthropic Claude Haiku 4.5 |
| Canal de salida | Slack (HITL + notificación final) + Gmail (alertas de error) |

## 3. Arquitectura del sistema

Ver diagrama completo: [`Diagrama_Arquitectura.pdf`](./Diagrama_Arquitectura.pdf)

Flujo implementado (15 nodos):

```
Webhook (POST)
→ Create Record (Airtable: Tickets, Estado=Pendiente)
→ Message a Model (Claude Haiku 4.5, clasificación JSON estructurada)
→ Update Record (Categoria_IA, Urgencia_IA, Resumen_IA, Area_Asignada, Estado=Procesado por IA)
→ IF Urgencia Alta
   → SÍ: Slack Aprobación (sendAndWait, botones Aprobar/Rechazar) → Update Estado Post-Aprobación
   → NO: Update Estado Auto (Aprobado automático)
→ Buscar Canal del Área (Airtable: Areas, lookup dinámico por Area_Asignada)
→ Notificar al Área (Slack, mensaje al canal correcto según el área)
→ Guardar Thread ID (Airtable: Slack_Thread_ID, trazabilidad del hilo)
→ Respond to Webhook (200 OK)
```

**Ruta de error (resiliencia):** los 3 nodos críticos (`Create a record`, `Message a model`, `Update record`) tienen activado `onError: continueErrorOutput`. Cualquier fallo se redirige a:
```
Log Error (Airtable: Errores, con Tipo_Error clasificado dinámicamente y Detalle real del fallo)
→ Notificar Error por Email (Gmail, alerta HTML al equipo técnico)
→ Respond Error (200/500 controlado, nunca deja el request colgado)
```

**Punto de control humano (HITL):** los tickets con `Urgencia_IA = Alta` se detienen antes de notificar al área — el sistema espera hasta 24hs una aprobación manual en Slack (botones nativos Aprobar/Rechazar) antes de continuar. Los de urgencia Media/Baja avanzan automáticamente. Esto evita el "efecto metralleta" de notificaciones automáticas sin supervisión en casos críticos.

**Salida multicanal real:** a diferencia de una notificación única de aprobación, el sistema **completa el ciclo** avisando al área responsable el resultado final (no solo pidiendo permiso), mapeando el canal de Slack dinámicamente desde la tabla `Areas` (sin hardcodear IDs de canal en el flujo) y guardando el `ts` del mensaje real en `Slack_Thread_ID` para trazabilidad.

## 4. Estructura de datos (Airtable)

Base: **Soporte - Sistema Triage IA** — [Ver en modo lectura](https://airtable.com/applOlsjUbjmCJWc5/shrLQtEXPIbOI5r7u)

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
| Estado | Single select | Pendiente → Procesado por IA → Aprobado / Error |
| Area_Asignada | Link a `Areas` | Relación real con tabla de áreas (evita hardcodear) |
| Fecha_Creacion | Created time | Automático |
| Fecha_Procesado | Date | Cuándo terminó de procesar la IA |
| Aprobado_Por | Texto | Quién aprobó en el paso HITL (solo tickets Alta) |
| Slack_Thread_ID | Texto | `ts` real del mensaje de notificación en Slack |

### Tabla `Areas` (catálogo, evita datos hardcodeados)
`Nombre_Area`, `Slack_Channel_ID` (canal real usado por el nodo "Notificar al Área"), `Email_Responsable`, `Tickets` (link inverso)

### Tabla `Errores`
`Error_ID`, `Tipo_Error` (API IA / Datos Faltantes / Timeout / Otro — clasificado dinámicamente según el mensaje real de fallo), `Detalle` (mensaje de error real capturado), `Fecha`

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

Resumen: se usa **Claude Haiku 4.5** para la clasificación (tarea simple y estructurada, `max_tokens=300`, `temperature=0`), reservando modelos más grandes (Claude Sonnet 5) solo para tickets con contexto extenso o adjuntos, y **Anthropic Batch API** para reprocesamiento masivo no urgente. Ahorro estimado: ~98% vs. usar el modelo top de la familia para todo.

## 6. Seguridad y resiliencia

- **Minimización de datos**: el prompt de IA solo recibe el mensaje del ticket, sin exponer datos de otros clientes ni credenciales.
- **Rutas de error reales**: fallos de la API de IA o datos faltantes se capturan con `continueErrorOutput`, se registran en la tabla `Errores` con tipo y detalle real (no genérico), y se notifican por email — probado en vivo múltiples veces.
- **Typecast controlado**: se usa validación de tipos en las escrituras a Airtable para evitar datos corruptos.
- **Human-in-the-loop**: tickets de urgencia Alta no se notifican al área sin aprobación humana vía Slack, probado con aprobaciones reales.
- **Credenciales**: tokens de Airtable, Anthropic, Slack y Gmail gestionados como credenciales cifradas dentro de n8n, nunca hardcodeados en los nodos ni en este repositorio.
- **Sin datos hardcodeados**: el canal de Slack y el área asignada se resuelven dinámicamente desde la tabla `Areas` en cada ejecución, no están fijos en el código del flujo.

## 7. Dashboard de control

Los KPIs de tickets por estado, categoría, urgencia y área, y la tasa de errores, se consultan en tiempo real desde la misma vista de solo lectura de Airtable enlazada en la sección 4: [Ver base/dashboard](https://airtable.com/applOlsjUbjmCJWc5/shrLQtEXPIbOI5r7u). Al estar vinculada directamente a la base, cualquier persona puede ver el estado del sistema (tickets procesados, errores, urgencias) sin necesidad de acceso a n8n ni credenciales.

## 8. Evidencias

Ver capturas en la carpeta del repositorio: ejecución completa del flujo en n8n (11-15 nodos en verde), prueba de webhook con 200 OK, tickets clasificados en Airtable, mensaje de aprobación HITL en Slack, notificación final al área, y alerta de error por email.

`Triage_Tickets_Soporte_TP_Final.json` — export del workflow de n8n (credenciales excluidas por seguridad; se reconfiguran manualmente al importar).

## 9. Test de estrés realizado

Se probó el flujo con **más de 45 ejecuciones** cubriendo: bug crítico (urgencia alta, con aprobación HITL real), duda de facturación (urgencia media/alta), feature request (urgencia baja), comentarios generales (categoría "Otro"), y el "camino infeliz" con datos faltantes (sin campo `mensaje`) para validar la ruta de error completa (registro en Airtable + alerta por email + respuesta controlada). La base de datos final contiene 38 tickets con clasificación completa y 3 registros de error correctamente tipificados.
