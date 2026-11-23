# Runbook mínimo: observabilidad y alertas de errores en producción (KAN-83)

Este documento cubre las dos señales de salud del sistema en producción — **uptime del proceso**
(`GET /health` + monitor externo) y **errores en tiempo real** (Sentry) — y qué hacer cuando
disparan. Pensado para que cualquiera del equipo, no solo quien lo escribió, pueda actuar sin tener
que releer el código primero.

## 1. Healthcheck (`GET /health`)

- **Qué es:** endpoint público, sin auth, sin dependencia de Supabase (KAN-83) — refleja únicamente
  si el proceso Node está arriba y respondiendo, no la salud de servicios downstream (a propósito:
  un healthcheck que dependa de Supabase generaría falsos negativos de infraestructura cada vez que
  Supabase tenga un blip transitorio, incluso si el proceso en sí está sano).
- **Implementación real:** `src/app.ts#createApp()` monta `app.get('/health', ...)`, que delega en
  `buildHealthPayload()` (`src/utils/health.ts`, KAN-141). Devuelve `200` siempre que el proceso
  responda, con `{ status: 'ok', liveness: true, uptimeSeconds, memory: {...}, cpu: {...} }`.
- **Ya está usado por:** el healthcheck nativo de Railway (deploy no se considera exitoso hasta que
  este endpoint responde 200).
- **Pendiente, fuera de este código — acción manual en un servicio externo:** dar de alta un monitor
  de uptime de terceros (ej. UptimeRobot, BetterStack/BetterUptime, Pingdom, o el propio "Health
  Checks" de Railway con notificación) que pegue a `https://<host-de-producción>/health` cada 1-5
  minutos y alerte por email/Slack si deja de responder 200 (timeout o cualquier status ≠ 200
  durante N chequeos consecutivos, para no alertar por un blip de red de un solo request). No es
  configurable desde este repo — requiere credenciales de un servicio externo que no están
  disponibles en este entorno. Pasos sugeridos:
  1. Crear una cuenta/monitor en el servicio elegido, apuntando a `/health` con método `GET`.
  2. Intervalo de chequeo: 1-5 min (health checks livianos, sin costo de red/DB real).
  3. Umbral de alerta: ≥2 fallos consecutivos antes de notificar (evita ruido por un timeout único).
  4. Canal de notificación: email del equipo + (opcional) webhook de Slack al mismo canal que Sentry
     (ver sección 2) para que todas las alertas de producción converjan en un solo lugar.

## 2. Errores de aplicación (Sentry)

### 2.1 Cómo llegan los errores a Sentry

`instrument.js` inicializa el SDK (`Sentry.init(...)`, DSN real, `dataCollection` filtrado de PII —
ver `docs/sentry-security-audit.md`). Hasta KAN-83, esa era toda la integración: **ningún punto del
código llamaba `Sentry.captureException`/`captureMessage`**, así que salvo una excepción
verdaderamente no controlada que tirara el proceso, nada llegaba a Sentry — el pipeline de alertas
no existía en la práctica.

KAN-83 conecta esto en un único punto: `src/services/logger.ts` (el logger de Pino usado en todo el
repo) tiene un hook (`hooks.logMethod`) que reenvía a Sentry **cualquier `logger.error`/`.fatal`/
`.warn` ya existente**, sin tener que tocar los ~100+ call sites de `logger.*` que ya hay en el
código. La lógica de clasificación (qué nivel de Pino se reenvía y con qué severidad) es pura y
está testeada en `tests/sentryForwarding.test.ts` (`src/utils/sentryForwarding.ts`).

### 2.2 Criterios de severidad (AC de KAN-83)

| Nivel de Pino en el código | Tag `severity` en Sentry | `level` en Sentry | Significado |
|---|---|---|---|
| `logger.fatal(...)` | `critical` | `fatal` | Falla que compromete el proceso completo. |
| `logger.error(...)` | `critical` | `error` | Error real de una operación (ej. falla de Supabase, excepción no controlada en una ruta, fallo de un servicio externo). **Alerta inmediata.** |
| `logger.warn(...)` | `low` | `warning` | Degradación esperada/manejada (ej. rate limit excedido, fallback de IA activado, sesión expirada). **Resumen periódico**, no alerta inmediata. |
| `logger.info(...)` | — (no se reenvía) | — | Operación normal, nunca llega a Sentry. |

Este criterio reutiliza la convención de niveles que el código YA sigue (`logger.error` para fallas
reales, `logger.warn` para degradaciones esperadas) — no hace falta re-etiquetar nada retroactivo.

### 2.3 Configuración pendiente en el dashboard de Sentry (acción manual, no vía código)

El código ya envía `level`/`tags.severity` correctos en cada evento — falta crear las **Alert
Rules** en el dashboard de Sentry (`https://sentry.io/organizations/<org>/alerts/rules/`, proyecto
del DSN en `instrument.js`) que traducen eso en notificaciones reales. No se pudo hacer desde este
entorno (requiere credenciales/API token de Sentry no disponibles acá). Configuración sugerida:

1. **Regla "Errores críticos — inmediato":** condición `event.level:error OR event.level:fatal` (o
   `tags.severity:critical`) → acción "Send a notification" inmediata a email + Slack, sin agrupar
   por ventana de tiempo. Frecuencia de re-notificación: cada vez que el issue vuelve a ocurrir tras
   estar resuelto/ignorado (no silenciar reincidencias).
2. **Regla "Errores leves — resumen":** condición `tags.severity:low` (o `event.level:warning`) →
   usar el "Weekly Reports"/digest nativo de Sentry, o una regla con "digest interval" (agrupa
   varios eventos del mismo tipo en una sola notificación periódica en vez de una por evento) — diario
   o semanal según volumen real una vez que haya datos.
3. Canal de Slack: mismo que el sugerido para el monitor de uptime (sección 1), para no fragmentar
   las alertas de producción en múltiples canales.

### 2.4 Qué NO llega a Sentry (a propósito)

`dataCollection` (`src/config/sentryDataCollection.ts`, KAN-126) sigue aplicando igual — bodies HTTP
deshabilitados, cookies/headers de sesión excluidos. Además, el hook de reenvío de KAN-83 filtra del
contexto adicional que manda a Sentry los mismos campos de PII que ya redacta Pino (`PII_LOG_FIELDS`,
`src/config/logRedaction.ts` — email, ip, domicilio, texto de búsqueda, etc.), así que un
`logger.error({ email, tenantId }, '...')` llega a Sentry con `tenantId` pero sin `email`.

## 3. Qué hacer ante una alerta

**Alerta de uptime (`/health` no responde):**
1. Confirmar en Railway si el deploy está arriba (`railway logs`/dashboard) — puede ser un deploy en
   curso legítimo, no necesariamente un incidente.
2. Si el proceso está caído/reiniciando en loop: revisar los logs de arranque (errores de
   `validateConfig()` en `src/config/env.ts` — variables de entorno faltantes son la causa más común
   de un arranque fallido, tiran `Error` explícito y documentado).
3. Si el proceso está arriba pero `/health` no responde: puede ser saturación de CPU/memoria — el
   propio payload de `/health` (`cpu.loadAvg*`, `memory.*`) es el primer dato a mirar una vez que
   vuelva a responder.

**Alerta de Sentry (crítica):**
1. Abrir el issue en Sentry — el `tags.severity`/contexto `log` (tenantId, searchId, etc., sin PII)
   da el alcance sin tener que ir a los logs de Railway primero.
2. Si son varios eventos del mismo issue en poco tiempo: puede ser un servicio externo caído
   (Supabase, Gemini/OpenAI, Resend) — los `logger.error` de este repo ya distinguen ese caso en el
   mensaje (`[AUTH] No se pudo conectar con Supabase...`, `[AI STRATEGY] Todas las estrategias...
   fallaron`, etc.), buscar ese patrón antes de asumir un bug de código nuevo.
3. Correlacionar con Railway logs por timestamp si hace falta más contexto que el que trae Sentry
   (Sentry no tiene el log completo, solo el evento puntual).

**Resumen periódico de Sentry (leve):** revisión de rutina, no requiere acción inmediata — sirve para
detectar patrones (ej. un rate limit que se excede sistemáticamente, señal de que el límite está mal
calibrado, no de un ataque puntual).
