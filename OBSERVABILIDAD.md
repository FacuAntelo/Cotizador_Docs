# Observabilidad — logs, métricas, costos, límites, alertas y perfiles

Cómo se mide el cotizador: qué se registra, dónde queda, cómo se consulta y qué hacer
con eso. Cubre el backend (`Cotizador_BE`) y el front (`Cotizador_FE`).

La idea en una línea: **cada request tiene un código** (`request_id`) que viaja por los
logs, la base, los headers y las trazas. Con ese código se reconstruye todo lo que pasó:
qué tardó, qué falló, cuánto costó la IA y qué vio el vendedor.

```
navegador ──X-Request-ID──▶ middleware ──▶ endpoint ──▶ step("epc.familia") / Claude / SAP
   │                           │                          │
   │ Web Vitals, errores JS,   ├─▶ log (consola + JSON)   ├─▶ llm_calls (tokens, US$)
   │ tiempos de cotización     ├─▶ request_log            ├─▶ dependency_events (logins)
   └──▶ POST /telemetria       └─▶ X-Request-ID +         └─▶ quote_runs (pedido → resultado)
        (frontend_events)          Server-Timing
                                         ▼
                    Monitoreo (/monitoreo) · alertas · /salud · trazas (opcional)
```

Todo funciona sin infraestructura nueva: los datos quedan en Postgres, en un schema
aparte del negocio, y en `logs/`. OpenTelemetry queda listo para mandar trazas a
Grafana el día que se quiera (apagado por defecto).

## Índice

1. [Rastrear un error que reporta un vendedor](#1-rastrear-un-error-que-reporta-un-vendedor)
2. [Logs](#2-logs)
3. [Medición de cada request](#3-medición-de-cada-request)
4. [Integraciones: EPC, SAP y localizador](#4-integraciones-epc-sap-y-localizador)
5. [Costos de la IA y tope diario](#5-costos-de-la-ia-y-tope-diario)
6. [Registro de cotizaciones](#6-registro-de-cotizaciones)
7. [Límites de uso](#7-límites-de-uso)
8. [Salud del sistema](#8-salud-del-sistema)
9. [Recursos del proceso y event loop](#9-recursos-del-proceso-y-event-loop)
10. [Perfiles a pedido](#10-perfiles-a-pedido)
11. [Pantalla Monitoreo](#11-pantalla-monitoreo)
12. [Alertas](#12-alertas)
13. [Telemetría del navegador](#13-telemetría-del-navegador)
14. [Trazas con OpenTelemetry (opcional)](#14-trazas-con-opentelemetry-opcional)
15. [Tablas, retención y consultas útiles](#15-tablas-retención-y-consultas-útiles)
16. [Privacidad](#16-privacidad)
17. [Cuánto cuesta medir](#17-cuánto-cuesta-medir)
18. [Configuración completa](#18-configuración-completa)
19. [Problemas frecuentes](#19-problemas-frecuentes)

Para sumar usuarios cuando exista el login, ver
[LOGIN_Y_OBSERVABILIDAD.md](LOGIN_Y_OBSERVABILIDAD.md). Para las reglas al escribir
código nuevo (cómo loguear, medir y responder errores), ver [STANDARDS.md](STANDARDS.md).

---

## 1. Rastrear un error que reporta un vendedor

Cuando algo falla del lado del servidor, el front muestra un mensaje claro y el código:

> No se pudo completar la cotización. Probá de nuevo en unos segundos.
> Código para soporte: a3f9c2b1d4e5.

Con ese código:

1. **Monitoreo → «Rastrear un código de error»** (desde la PC del servidor). Muestra el
   request con su cascada de etapas, el error técnico, las llamadas a Claude con sus
   tokens y costo, los logins de EPC/SAP de ese request, la cotización (texto del
   pedido, chasis, ítems) y lo que midió el navegador. También en
   `GET /monitoreo/requests/{codigo}`.
2. **Los logs**, para el traceback completo:
   ```powershell
   Select-String -Path logs\cotizador.log* -Pattern "a3f9c2b1d4e5"
   ```
3. **La traza**, si OpenTelemetry está activo: el log en JSON trae el `trace_id`.

Los errores del usuario (4xx: chasis inválido, límite de uso) no llevan código porque no
hay nada que investigar; el mensaje ya dice qué hacer.

**Cómo se arma el mensaje.** El usuario nunca ve tecnicismos: los endpoints responden con
`server_error()` o `report_error()` (`app/core/errors.py`, `app/core/http_errors.py`), que
loguean el error real con su traceback y lo guardan en `request_log.error`, y le
devuelven al vendedor un texto que entiende. Las excepciones cuyo mensaje ya está
escrito para el vendedor (las del EPC, SAP, localizador, el asesor y el tope de la IA)
heredan de `UserFacingError` y se muestran tal cual. Un error no controlado se responde
con un 500 en JSON con el código, en vez del texto plano de Starlette.

---

## 2. Logs

### Dónde quedan

| Destino | Formato | Para qué |
|---|---|---|
| Consola | Texto legible (o JSON con `LOG_FORMAT=json`) | Mirar en vivo mientras corre |
| `logs/cotizador.log` (carpeta del backend) | Un objeto JSON por línea | Buscar después, filtrar, importar a otra herramienta |

El archivo rota a medianoche (`cotizador.log.2026-10-30`, …) y se conservan
`LOG_RETENTION_DAYS` días (30). Con `LOG_FILE=` vacío solo se usa la consola: es lo que
conviene en la nube, donde el proveedor junta la salida de la consola.

Los logs se encolan y los escribe un hilo aparte (`QueueHandler`/`QueueListener`):
escribir en la consola de Windows o en el disco no frena a los requests.

### Formato

Consola:

```
10:15:02 INFO    [a3f9c2b1d4e5] requests: POST /cotizar/stream → 200 en 3.42s (primer evento 0.21s)
10:15:02 WARNING [a3f9c2b1d4e5] epc_scraper: EPC: la licencia está en uso en otro dispositivo; se toma para esta sesión.
```

Archivo (JSON):

```json
{"ts": "2026-10-30T13:15:02.123+00:00", "level": "INFO", "logger": "app.requests",
 "msg": "POST /cotizar/stream → 200 en 3.42s (primer evento 0.21s)",
 "request_id": "a3f9c2b1d4e5", "user_id": "…", "trace_id": "…",
 "http_method": "POST", "route": "/cotizar/stream", "status": 200,
 "duration_ms": 3420.5, "first_byte_ms": 210.2}
```

`user_id` aparece cuando exista el login; `trace_id`, con OpenTelemetry activo; los
errores traen `exception` con el traceback.

### Una línea por request

El middleware escribe una línea por request con ruta, estado y tiempos (reemplaza al log
de acceso de uvicorn, que queda apagado). Nivel:

| Caso | Nivel |
|---|---|
| Error del servidor (5xx) | ERROR |
| Lento (más de `OBS_SLOW_REQUEST_SECONDS`), límite de uso (429) o stream que terminó con error | WARNING |
| Estados que el front consulta seguido (`OBS_QUIET_PATHS`) y salieron bien | DEBUG |
| Lo demás | INFO |

Un request lento suma una línea con el desglose:

```
WARNING Request lento: POST /cotizar tardó 18.2s — epc.familia x12 9.8s (máx 2.1s), claude.match 4.1s
```

### Secretos

Antes de escribir, los logs pasan por `SecretRedactor`: tapa `Bearer …`, `sk-ant-…`,
`password=…`/`token=…`/`cookie=…` y los valores exactos de las contraseñas y tokens del
`.env` (EPC, SAP, Simetrical, Anthropic, agente del DMS). Igual, la regla es no
loguearlos nunca (ver STANDARDS.md).

### Buscar en los logs (PowerShell)

```powershell
# Todo lo de un request
Select-String -Path logs\cotizador.log* -Pattern "a3f9c2b1d4e5"

# Los errores de hoy, como tabla
Get-Content logs\cotizador.log | ConvertFrom-Json |
  Where-Object level -eq "ERROR" | Select-Object ts, request_id, msg

# Requests de más de 10 s
Get-Content logs\cotizador.log | ConvertFrom-Json |
  Where-Object { $_.duration_ms -gt 10000 } | Select-Object ts, route, duration_ms

# Alertas
Select-String -Path logs\cotizador.log* -Pattern "ALERTA"
```

---

## 3. Medición de cada request

`ObservabilityMiddleware` (`app/core/observability/middleware.py`) es ASGI puro: no corta
el streaming de `/cotizar/stream` y el contexto del request llega a los endpoints, a los
hilos de `asyncio.to_thread` y a los loops de Playwright de EPC y SAP.

Para cada request:

- **Código**: el `X-Request-ID` que manda el front (12 caracteres) o uno nuevo. Vuelve en
  el header `X-Request-ID` de la respuesta.
- **Tres tiempos**: hasta los headers, hasta el primer byte y hasta el final. En el stream
  de la cotización, el primer byte es el primer evento y el final, el último.
- **Etapas** medidas con `step()`: cada llamada a Claude, cada búsqueda en el EPC, el stock
  de SAP, los logins. Quedan con su inicio y duración (la cascada de Monitoreo).
- **Atributos de negocio** con `annotate()`: ítems de la cotización, cuántos sin
  resultado, costo de la IA, si chocó con un límite.
- **Header `Server-Timing`** con las etapas terminadas antes de responder. Se ve en las
  herramientas del navegador: F12 → Red → el pedido → Tiempos.

### Etapas que se miden hoy

| Etapa | Dónde |
|---|---|
| `claude.match`, `claude.asesor.propuesta`, `claude.asesor.eleccion` | Llamadas a Claude (`llm_call()`) |
| `epc.vehiculo`, `epc.familia` (con la familia) | Búsquedas en la API del EPC |
| `epc.login`, `epc.licencia` | Apertura de sesión y toma de licencia del EPC |
| `sap.login`, `sap.stock` | Login SAML y consulta de stock |
| `localizador.login`, `localizador.buscar` | Simetrical |

### Qué se guarda

Una fila por request en `request_log` (ruta como plantilla, por ejemplo
`/localizador/{numero_parte}`, estado, tiempos, IP, etapas, atributos, error). No se
guardan:

- las rutas de `OBS_EXCLUDED_PATHS` (documentación);
- las de `OBS_QUIET_PATHS` (`/epc/session`, `/sap/session`, `/dms/stock/estado`, `/salud`,
  `/monitoreo`, `/telemetria`) cuando salen bien y rápido: el front las consulta cada pocos segundos.

La escritura es en lote desde un hilo aparte (`TelemetryWriter`): medir no agrega tiempo
al request, y si la base falla se pierden métricas pero la app sigue andando. Los
pendientes y descartados se ven en `/salud/detalle` → `registros`.

---

## 4. Integraciones: EPC, SAP y localizador

Los logins y las tomas de licencia quedan en `dependency_events` con su duración, si
salieron bien y el error (`error_code` estable, como `account_locked`). Se registran
también fuera de un request, por ejemplo al precalentar las sesiones al arrancar.

En Monitoreo → Integraciones: cuántos logins hubo, cuántos fallaron, cuánto tardan
(p50/p95), el último error, los tiempos de cada consulta externa y la frescura del stock
del taller.

---

## 5. Costos de la IA y tope diario

### Cada llamada a Claude

Toda llamada pasa por `llm_call()` (`app/services/llm_usage.py`), que:

1. revisa el tope diario antes de llamar;
2. mide la duración como etapa (`claude.<operación>`);
3. calcula el costo con el **modelo que respondió** (`response.model`: con el fallback del
   servidor, el asesor puede caer a otro modelo) y los cuatro contadores de tokens
   (entrada, lectura de caché, escritura de caché, salida);
4. lo suma al gasto del día y al request (costo por cotización);
5. deja la fila en `llm_calls`, **también si la llamada falló** (sin tokens, con el error).

Operaciones: `match` (matcher, Haiku), `asesor.propuesta` y `asesor.eleccion` (asesor,
Opus con esfuerzo `ANTHROPIC_ADVISOR_EFFORT`).

### Precios

En `app/core/llm_pricing.py`, en USD por millón de tokens, verificados el 2026-10-30:

| Modelo | Entrada | Salida | Lectura de caché | Escritura 5 min | Escritura 1 h |
|---|---|---|---|---|---|
| claude-haiku-5-5 (prompt ≤ 100K tokens) | 0,10 | 0,50 | 0,01 | 0,125 | 0,20 |
| claude-haiku-5-5 (prompt > 100K tokens) | 0,50 | 2,50 | 0,05 | 0,625 | 1,00 |
| claude-sonnet-5-5 | 2 | 10 | 0,20 | 2,50 | 4 |
| claude-opus-5-5 | 4 | 20 | 0,20 | 5 | 8 |
| claude-opus-5 / claude-opus-4-8 (fallback) | 5 | 25 | 0,50 | 6,25 | 10 |
| claude-fable-5-1 | 10 | 50 | 0,25 | 12,50 | 20 |

El umbral de Haiku cuenta toda la entrada (sin caché + leída + escrita). Un modelo sin
precio deja el costo vacío y avisa en el log: nunca se inventa un número.

**Actualizar precios sin tocar el código**: un JSON en `LLM_PRICES_FILE` que reemplaza o
agrega modelos.

```json
{
  "claude-opus-5-5": {"input": 4, "output": 20, "cache_read": 0.2,
                      "cache_write_5m": 5, "cache_write_1h": 8},
  "claude-haiku-5-5": {"input": 0.1, "output": 0.5, "cache_read": 0.01,
                       "cache_write_5m": 0.125, "cache_write_1h": 0.2,
                       "long_context_threshold": 100000,
                       "long": {"input": 0.5, "output": 2.5, "cache_read": 0.05,
                                "cache_write_5m": 0.625, "cache_write_1h": 1.0}}
}
```

### Caché del prompt del matcher

El prompt de sistema del matcher (catálogo y reglas) no cambia entre pedidos, así que se
cachea en Anthropic. `ANTHROPIC_CACHE_TTL` define cuánto dura: `5m` (por defecto), `1h` o
vacío para no usarla. Cada lectura renueva la duración.

Escribirlo cuesta más que la entrada normal (1,25× con `5m`, 2× con `1h`) y leerlo, 0,1×.
Por eso `5m` conviene si más o menos 1 de cada 5 pedidos encuentra la caché viva, y `1h`,
si lo logra más de la mitad. Para decidir con datos, mirá **Entrada leída de la caché** en
Monitoreo → Costos IA: si queda muy baja, los pedidos llegan demasiado separados y
conviene apagarla.

### Tope de gasto diario

| Configuración | Por defecto | Qué hace |
|---|---|---|
| `LLM_DAILY_BUDGET_USD` | 1 | Tope del día en USD; 0 = sin tope |
| `LLM_BUDGET_WARN_RATIO` | 0.8 | Al 80 % avisa en el log y en las alertas |
| `LLM_BUDGET_ACTION` | `asesor` | Al 100 %: `asesor` apaga solo el asesor (el modelo caro) y el matcher sigue; `todo` apaga toda la IA; `avisar` solo avisa |

- **"Hoy"** es el día en la hora del negocio (`OBS_TIMEZONE`, Buenos Aires): el tope se
  reinicia a medianoche local.
- El gasto se cuenta en memoria y **se recupera de la base al arrancar**, así reiniciar
  el backend no lo pone en cero.
- **Con el asesor apagado**, los repuestos fuera de catálogo salen sin sugerencia y con
  el mensaje «La IA avanzada llegó al límite de gasto de hoy…»; corregir al asesor
  responde 503 con ese mensaje. Con `todo`, la cotización responde 503 con «La IA llegó
  al límite de gasto de hoy…».
- Referencia para dimensionarlo: una llamada del matcher cuesta del orden de milésimas
  de dólar; una consulta al asesor (dos llamadas a Opus), del orden de varios centavos.
  Con US$ 1 por día entran decenas de consultas al asesor: mirá Monitoreo → Costos IA
  las primeras semanas y ajustá el tope con datos.

En Monitoreo → Costos IA: gasto de hoy frente al tope, gasto por día, por operación, por
modelo y por usuario, costo promedio y p95 por cotización y proporción de la entrada
leída de la caché.

---

## 6. Registro de cotizaciones

Cada `/cotizar` y `/cotizar/stream` deja una fila en `quote_runs`:

- el **texto del pedido tal como lo escribió el vendedor** y el chasis;
- por ítem: lo pedido, el nombre del catálogo, la familia, hasta 10 números y, si pasó
  por el asesor, su modo, confianza y cuántas opciones dio;
- cuántos ítems salieron del catálogo, cuántos del asesor y cuántos sin resultado;
- duración, tiempo hasta el primer repuesto (stream) y costo de la IA;
- el error, o «Cotización incompleta» si el vendedor cerró la pestaña antes de terminar.

Para qué: medir la calidad de la IA (Monitoreo → Calidad IA) y juntar casos reales para
evaluar cambios de modelo, esfuerzo, prompt o caché con números antes de pasarlos a
producción. Se conserva `RETENTION_QUOTES_DAYS` días (90).

Las correcciones al asesor se cuentan como requests a `/cotizar/asesor`: que el vendedor
corrija poco es la mejor señal de que el asesor acierta.

---

## 7. Límites de uso

Una dependencia global (`enforce_rate_limit` en `app/core/rate_limit.py`) cuenta los
intentos en una ventana deslizante en memoria, **por usuario cuando haya login y por IP
mientras tanto**. Cada ruta pertenece a un grupo:

| Grupo | Rutas | Límite por defecto | Para qué |
|---|---|---|---|
| `ia` | `POST /match`, `/cotizar`, `/cotizar/stream`, `/cotizar/asesor` | `20/min,400/dia` | Costo |
| `integraciones` | `POST /epc*`, `POST /sap/stock`, `GET /localizador/{…}` | `300/min` | Que un error en loop no castigue al EPC ni a SAP |
| `escrituras` | El resto de los POST/PUT/PATCH/DELETE | `120/min` | Abuso o errores en loop |
| `telemetria` | `POST /telemetria` | `60/min` | Basura |
| `login` | `POST /auth/login` (cuando exista) | `5/15min` | Fuerza bruta |

Sin límite: los GET (salvo el localizador), las lecturas que el front pide por cada fila
de la cotización (`POST /dms/stock`, `/precios/consultar`, `/compatibilidades/consultar`) y
la subida del agente del taller (`PUT /dms/stock`, que ya se autentica con su token).

`integraciones` tiene 300 por minuto y no 60 porque **el front pide el stock de SAP por
cada fila**: una cotización de 40 ítems ya son 40 pedidos en segundos.

**Formato** de `RATE_LIMIT_<GRUPO>`: reglas `cantidad/ventana` separadas por coma, con
unidades `s`, `min`, `h`, `dia` y múltiplos (`5/15min`). Vacío = sin límite para ese grupo;
`RATE_LIMIT_ENABLED=false` apaga todos. Una regla mal escrita frena el arranque con un
mensaje que dice cuál es.

**Al pasarse**: 429 con `Retry-After` y un mensaje para el usuario, por ejemplo «Hiciste
muchas consultas a la IA seguidas. Esperá 1 minuto y volvé a intentar.» (o «Llegaste al
máximo de consultas a la IA de hoy» con la regla diaria). El log guarda el detalle:

```
WARNING Límite de uso: grupo=ia quien=ip:192.168.1.20 regla=20/60s ruta=POST /cotizar/stream reintentar_en=42s
```

**Cotizaciones simultáneas**: como mucho `MAX_CONCURRENT_QUOTES` (4) a la vez, para no
saturar el EPC; la siguiente recibe 429 con «Hay muchas cotizaciones en curso…» y
`Retry-After` de `MAX_CONCURRENT_QUOTES_RETRY_SECONDS` (5). Un cupo que nunca se libera
(un stream cortado antes de empezar) vence solo a los 10 minutos.

Alcanza con memoria porque el backend corre en un solo proceso (las sesiones de Playwright
lo obligan). Con varios procesos habría que pasar los contadores a Redis.

El front no reintenta solo las consultas rechazadas con 4xx (`shouldRetry` en `App.tsx`).

---

## 8. Salud del sistema

| Endpoint | Para quién | Qué responde |
|---|---|---|
| `GET /salud` | Un monitor externo (UptimeRobot, Better Stack…) | 200 si el backend responde y lee su base; 503 si la base no responde. No toca EPC, SAP ni Claude |
| `GET /salud/detalle` | Diagnóstico desde la PC del servidor | Base (y su latencia), sesiones de EPC, SAP y localizador sin consultarlos, frescura del stock del taller, gasto de la IA, atraso del event loop, PID y memoria del proceso, requests y cotizaciones en curso, registros pendientes. Con `estado: "degradado"` y la lista de `problemas` |

---

## 9. Recursos del proceso y event loop

**Atraso del event loop** (`LoopLagMonitor`): cada `LOOP_LAG_CHECK_SECONDS` (1 s) mide
cuánto tarda de más en despertar. Si pasa de `LOOP_LAG_WARN_MS` (200 ms), algo bloqueó el
loop (código sincrónico dentro de un endpoint async) y todos los requests esperaron. El
aviso dice qué requests estaban en curso:

```
WARNING Event loop: se atrasó 850 ms; algo lo está bloqueando (código sincrónico en un endpoint async). Requests en curso: POST /match (a3f9c2b1d4e5, 2.1s).
```

En Windows el atraso «normal» ronda los 15 a 30 ms por la resolución del reloj del
sistema: no es un problema.

**Muestras del proceso** (`RuntimeSampler`, con `psutil`): cada `RUNTIME_SAMPLE_SECONDS`
(60 s) guarda en `runtime_samples` la RAM del backend, la de sus procesos hijos (el
driver de Playwright y los Chromium de EPC y SAP), la CPU (en proporción de toda la
máquina), la RAM usada de la PC, hilos, tareas async, requests en curso y el atraso
máximo y promedio del loop. La RAM de los hijos suma la de cada proceso y cuenta dos veces
la memoria compartida: es una cota superior.

---

## 10. Perfiles a pedido

Para ver **en qué gasta el tiempo el backend por dentro**, sin reiniciarlo:
`scripts/perfil.ps1`, que usa el perfilador por muestreo de Python 3.15
(`profiling.sampling`). Se engancha desde afuera al proceso: el backend no se entera ni se
frena, y nada queda corriendo después.

Requisitos: el backend corriendo, y **PowerShell abierto como administrador** (Windows no
deja leer la memoria de otro proceso sin permisos). El script toma el PID de
`/salud/detalle` y usa el Python del proyecto (`uv run`), que es la misma versión que el
backend.

```powershell
cd Cotizador_BE
.\scripts\perfil.ps1                     # flame graph de 30 s (tiempo de reloj, todos los hilos)
.\scripts\perfil.ps1 -Segundos 60        # más largo: usá la app mientras graba
.\scripts\perfil.ps1 -Modo cpu           # solo tiempo de CPU (sin la espera de red)
.\scripts\perfil.ps1 -Modo tareas        # foto de ahora: qué espera cada tarea async
.\scripts\perfil.ps1 -Modo hilos         # foto de ahora: todos los hilos (EPC y SAP incluidos)
.\scripts\perfil.ps1 -Modo arbol         # árbol de tareas async (python -m asyncio pstree)
```

Los flame graphs quedan en `perfiles\` (ignorada por git) y se abren solos. Cómo leerlos:
cada barra es una función y su ancho, el tiempo que estuvo en la pila; las de arriba son
las que estaban corriendo. Una barra ancha de código propio es donde mirar; una ancha de
`select`/`poll` es espera (normal).

Cuándo usarlo: Monitoreo muestra que algo es lento y la cascada de etapas no lo explica,
o el monitor del loop avisa atrasos. Para cazar callbacks lentos en desarrollo también
sirve correr con `PYTHONASYNCIODEBUG=1` (loguea los que tardan más de 100 ms).

---

## 11. Pantalla Monitoreo

Ruta `/monitoreo` del front. **Hasta que exista el login, solo se ve desde la PC donde
corre el backend** (`MONITOR_ALLOWED_HOSTS`): desde otra PC el menú no muestra el acceso y
la API responde 403. Período: hoy, 7 o 30 días (en la hora local). Se refresca sola cada
minuto.

| Sección | Qué muestra | Endpoint |
|---|---|---|
| Rastrear un código | Todo lo de un `request_id` | `GET /monitoreo/requests/{codigo}` |
| Rendimiento | Requests, % de errores, p50/p95/p99 por ruta, rechazos por límite, los 20 más lentos y los errores recientes | `GET /monitoreo/rendimiento` |
| Costos IA | Gasto de hoy frente al tope, por día, operación, modelo y usuario; costo por cotización; caché | `GET /monitoreo/costos` |
| Calidad IA | Ítems sin resultado, al asesor, correcciones, confianza, tiempo hasta el primer repuesto, últimas cotizaciones | `GET /monitoreo/calidad-ia` |
| Integraciones | Logins y licencias, tiempos de cada consulta externa, fallas recientes, stock del taller | `GET /monitoreo/integraciones` |
| Negocio | Cotizaciones → presupuestos → aceptados → con pedido, montos, por día y por usuario | `GET /monitoreo/negocio` |
| Recursos | RAM del backend y de Chromium, CPU, atraso del loop | `GET /monitoreo/recursos` |
| Navegador | Web Vitals, errores de JavaScript, tiempos de cotización vistos por el vendedor | `GET /monitoreo/front` |
| Alertas | Activas e historial | `GET /monitoreo/alertas` |

Todos aceptan `?dias=N` (1 = hoy). Los percentiles: **p50** es la mediana; **p95**, lo que
tarda el 5 % más lento (la experiencia de los peores casos).

Un stream de cotización que falla después de responder 200 (el error viaja dentro del
stream) cuenta como error igual.

Ojo: detrás de un proxy en la misma PC todos los pedidos llegan desde `127.0.0.1` y
cualquiera vería Monitoreo; en ese caso, activar `--proxy-headers` en uvicorn o esperar al
login.

---

## 12. Alertas

`AlertManager` (`app/services/alerts.py`) revisa las reglas cada `ALERTS_CHECK_SECONDS`
(5 min). Una alerta avisa al dispararse, **vuelve a avisar cada `ALERTS_REPEAT_MINUTES`**
(60) mientras siga activa y avisa cuando se resuelve. Cada cambio queda en `alert_events`.

| Regla | Severidad | Se dispara cuando | Configuración |
|---|---|---|---|
| `errores_servidor` | crítica | Más del 5 % de los requests fallaron en los últimos 15 min (con al menos 20 requests) | `ALERT_ERROR_RATE_PCT`, `ALERT_ERROR_RATE_WINDOW_MINUTES`, `ALERT_MIN_REQUESTS` |
| `cotizaciones_lentas` | advertencia | El p95 de las cotizaciones pasó 20 s en la última hora (con al menos 5) | `ALERT_QUOTE_P95_SECONDS`, `ALERT_QUOTE_WINDOW_MINUTES`, `ALERT_QUOTE_MIN_SAMPLES` |
| `gasto_ia` | advertencia / crítica | El gasto del día llegó al 80 % / 100 % del tope | `LLM_BUDGET_WARN_RATIO`, `LLM_DAILY_BUDGET_USD` |
| `login_epc`, `login_sap`, `login_localizador` | crítica | Fallaron los últimos 3 logins seguidos | `ALERT_LOGIN_FAILURES` |
| `login_bloqueado_sap`, `login_bloqueado_localizador` | crítica | Se rechazó el usuario o la cuenta está bloqueada y se dejó de reintentar. **Avisa en el momento**, sin esperar la revisión | — |
| `stock_taller_desactualizado` | advertencia | El stock del DMS está desactualizado en horario de atención | `ALERT_DMS_BUSINESS_HOURS` (`08-19`), `ALERT_DMS_BUSINESS_DAYS` |
| `event_loop_bloqueado` | advertencia | El loop se atrasó más de 1 s | `ALERT_LOOP_LAG_MS` |
| `registros_perdidos` | advertencia | Se perdieron registros de observabilidad (la base no da abasto o falla) | — |

**Dónde avisan hoy**: en el log (`ERROR`/`WARNING` con el prefijo `ALERTA`, y `INFO
ALERTA resuelta` al resolverse) y en Monitoreo → Alertas. El canal externo (Telegram,
email, Slack) quedó pendiente de definir.

**Sumar un canal**: alcanza con registrar una función en el lifespan de `main.py`. Por
ejemplo, Telegram: crear el bot con @BotFather, sumar `telegram_bot_token` y
`telegram_chat_id` a `Settings` (y al `.env`), y:

```python
import threading

import httpx

def _send_telegram(text: str) -> None:
    httpx.post(
        f"https://api.telegram.org/bot{settings.telegram_bot_token}/sendMessage",
        json={"chat_id": settings.telegram_chat_id, "text": text},
        timeout=10,
    )

def telegram(alert, state):
    if state == "sigue":
        return  # solo disparadas y resueltas
    prefix = "Resuelta" if state == "resuelta" else alert.severity.upper()
    # En un hilo: los avisos inmediatos (login de SAP rechazado) salen desde el loop principal.
    threading.Thread(target=_send_telegram, args=(f"{prefix}: {alert.message}",), daemon=True).start()

alert_manager.add_notifier(telegram)
```

Si un canal falla, se loguea y los demás siguen.

---

## 13. Telemetría del navegador

El backend no ve lo que pasa en el navegador del vendedor. `src/lib/telemetry.ts` mide:

| Qué | Cómo | Para qué |
|---|---|---|
| **Web Vitals**: LCP (carga del contenido principal), INP (respuesta a clics y teclas), CLS (saltos de la pantalla), FCP, TTFB | Librería `web-vitals` | Saber si la pantalla se siente rápida en la PC real del vendedor |
| **Errores de JavaScript** | `window.onerror` y promesas sin manejar | Errores que nadie reporta |
| **Tiempos de la cotización**: hasta el primer repuesto y total | En `useQuote`, con el `request_id` del stream | La espera que siente el vendedor, cruzable con el backend |
| **Pedidos lentos o fallidos** | Los que tardan 5 s o más, fallan en el servidor o no llegan | Problemas de red o del backend vistos desde afuera |

Sale en lotes cada 10 s y al cerrar u ocultar la pestaña, con `navigator.sendBeacon` como
`text/plain` (no frena la navegación y no necesita el preflight de CORS) a `POST
/telemetria`, que lo guarda en `frontend_events`. Nunca manda texto de pedidos ni datos de
clientes; las rutas con ids se agrupan (`/presupuestos/:id`). Se apaga con
`VITE_TELEMETRY=false` en el front. El backend acepta hasta `TELEMETRY_MAX_EVENTS` (50)
eventos y `TELEMETRY_MAX_BYTES` (64 KB) por lote.

Monitoreo → Navegador muestra cada Web Vital en el percentil 75 con la calificación de
Google (por ejemplo, LCP es bueno hasta 2,5 s; INP, hasta 200 ms).

Además, cada pedido del front lleva su `X-Request-ID` (middleware de `openapi-fetch` y
`apiFetch` en `src/api/client.ts`), y los errores del servidor se muestran con el código
para soporte.

---

## 14. Trazas con OpenTelemetry (opcional)

Apagadas por defecto y sin costo. Se activan así:

1. Instalar las dependencias opcionales: `uv sync --extra otel`.
2. Levantar un colector. Para probar en la PC, Grafana con todo incluido:
   ```bash
   docker run --rm -p 3000:3000 -p 4317:4317 -p 4318:4318 grafana/otel-lgtm
   ```
3. En el `.env`: `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318` y reiniciar el backend.
4. Abrir Grafana en http://localhost:3000 → Explore → Tempo, y buscar por servicio
   `cotizador-backend`.

Qué genera: un span por request (FastAPI), uno por cada `step()` (EPC, SAP, Claude…),
uno por cada pedido de httpx (SAP y el localizador) y los del SDK de Anthropic **solo con
metadatos** (modelo, tokens, duración; nunca el texto). El `trace_id` aparece en cada log
JSON para saltar de un log a su traza. Con login, el span lleva `enduser.id`.

| Configuración | Por defecto | |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | vacío | Colector OTLP/HTTP; vacío = apagado |
| `OTEL_SERVICE_NAME` | `cotizador-backend` | |
| `OTEL_TRACES_SAMPLE_RATIO` | 1 | Proporción de requests trazados |
| `OTEL_EXCLUDED_URLS` | docs, salud, monitoreo, telemetría, `/epc/session`, `/sap/session` | Rutas que no se trazan |

Para Grafana Cloud: usar su endpoint OTLP y definir `OTEL_EXPORTER_OTLP_HEADERS` (la
autorización) como **variable de entorno del sistema**: el exportador la lee directo, no
del `.env`. Si el colector no responde, la app sigue andando: el exportador avisa en el log
y al apagar espera unos segundos a que venzan los reintentos.

---

## 15. Tablas, retención y consultas útiles

| Tabla | Qué guarda | Retención (días) |
|---|---|---|
| `observability.request_log` | Un request: ruta, estado, tiempos, etapas, atributos, error | `RETENTION_REQUESTS_DAYS` (90) |
| `observability.dependency_events` | Logins y licencias de EPC, SAP y el localizador | `RETENTION_EVENTS_DAYS` (90) |
| `app.llm_calls` | Cada llamada a Claude: modelos, tokens, costo, error | `RETENTION_LLM_CALLS_DAYS` (395: comparar un año con otro) |
| `app.quote_runs` | Cada cotización: pedido, chasis, resultado por ítem, costo | `RETENTION_QUOTES_DAYS` (90) |
| `observability.runtime_samples` | RAM, CPU, Chromium y atraso del loop | `RETENTION_RUNTIME_DAYS` (30) |
| `observability.alert_events` | Alertas disparadas, repetidas y resueltas | `RETENTION_ALERTS_DAYS` (90) |
| `observability.frontend_events` | Web Vitals, errores de JavaScript y tiempos del navegador | `RETENTION_FRONTEND_DAYS` (90) |

Todas tienen `created_at` en UTC y `user_id` (vacío hasta que exista el login). Una
limpieza automática borra lo vencido al minuto de arrancar y después una vez por día; 0
días = no se borra nunca.

**Telemetría y negocio van separados.** Lo que está en el schema `observability` se puede
perder sin afectar al negocio: no tiene foreign keys hacia las demás tablas y los respaldos
guardan su estructura pero no sus datos. `llm_calls` y `quote_runs` no son telemetría: son
costos y pedidos, y van en `app` con el resto del sistema. Los logs de texto nunca van a la base.

**`request_log` y `frontend_events` están particionadas por mes**
(`app/database/partitions.py`), porque son las que más crecen. Vencer un mes es un `DROP` de
su partición: instantáneo y sin filas muertas. Por eso una fila dura entre los días
configurados y un mes más. El backend crea el mes actual y los dos siguientes al arrancar y
cada día; lo que caiga fuera de esos meses va a una partición por defecto, que tiene que
estar vacía (ver la consulta de abajo).

Consultas útiles (`psql -h 127.0.0.1 -U cotizador_dev cotizador_dev`):

```sql
-- Gasto en Claude por día (hora UTC) y operación
SELECT (created_at AT TIME ZONE 'UTC')::date AS dia, operation,
       round(sum(cost_usd)::numeric, 4) AS usd, count(*) AS llamadas
FROM app.llm_calls GROUP BY dia, operation ORDER BY dia DESC;

-- Las 20 cotizaciones más caras
SELECT created_at, request_id, round(llm_cost_usd::numeric, 4) AS usd, items_total, text
FROM app.quote_runs ORDER BY llm_cost_usd DESC NULLS LAST LIMIT 20;

-- Pedidos que la IA no resolvió (para revisar y armar casos de prueba)
SELECT created_at, text, items_without_result FROM app.quote_runs
WHERE items_without_result > 0 ORDER BY created_at DESC;

-- Rutas más lentas de la última semana
SELECT route, count(*) AS n, round(avg(duration_ms)) AS promedio_ms, max(duration_ms) AS max_ms,
       percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms
FROM observability.request_log WHERE created_at >= now() - interval '7 days'
GROUP BY route ORDER BY promedio_ms DESC;

-- Logins fallidos
SELECT created_at, service, error_code, error FROM observability.dependency_events
WHERE operation = 'login' AND NOT ok ORDER BY created_at DESC;

-- Tiene que dar 0: si no, dejaron de crearse las particiones de cada mes
SELECT count(*) FROM observability.request_log_default;
```

---

## 16. Privacidad

- **Nunca se guardan ni se loguean**: contraseñas, tokens, cookies, la API key, los
  cuerpos de los requests ni datos del cliente final (nombre, teléfono). Los presupuestos
  sí tienen esos datos, pero en sus propias tablas, no en las de observabilidad.
- **Sí se guarda** el texto del pedido y el chasis en `quote_runs` (decidido para poder
  evaluar la IA con casos reales), 90 días.
- La IP de quien llama queda en `request_log` (hasta que haya login es lo único que
  distingue a quién se le cuentan los límites).
- Las trazas de OpenTelemetry no llevan el texto de los prompts.
- Con login, las tablas pasan a tener el usuario: ver
  [LOGIN_Y_OBSERVABILIDAD.md](LOGIN_Y_OBSERVABILIDAD.md).

---

## 17. Cuánto cuesta medir

- **Logs**: se encolan (microsegundos) y los escribe otro hilo.
- **request_log y demás tablas**: una inserción en lote cada 2 s desde otro hilo; los
  estados que el front consulta seguido no se guardan si salen bien.
- **Etapas y Server-Timing**: unos microsegundos por etapa.
- **psutil**: una muestra por minuto (decenas de milisegundos, en un hilo).
- **Monitor del loop**: despierta una vez por segundo.
- **Perfiles**: solo cuando se corre el script, y desde afuera del proceso.
- **OpenTelemetry**: apagado no cuesta nada; prendido, el envío es en lote y en otro hilo.
- **Telemetría del navegador**: un lote cada 10 s como mucho, con `sendBeacon`.

---

## 18. Configuración completa

Todas en el `.env` del backend (con su explicación en `.env.example` y en el docstring de
`Settings`, `app/core/config.py`):

| Grupo | Variables |
|---|---|
| Logs | `LOG_LEVEL`, `LOG_FORMAT`, `LOG_FILE`, `LOG_RETENTION_DAYS`, `LOG_QUIET_LOGGERS` |
| Requests | `OBS_SLOW_REQUEST_SECONDS`, `OBS_EXCLUDED_PATHS`, `OBS_QUIET_PATHS`, `OBS_WRITER_BATCH_SIZE`, `OBS_WRITER_FLUSH_SECONDS`, `OBS_WRITER_MAX_QUEUE`, `OBS_TIMEZONE`, `CORS_ORIGINS` |
| Costos de la IA | `ANTHROPIC_CACHE_TTL`, `LLM_DAILY_BUDGET_USD`, `LLM_BUDGET_WARN_RATIO`, `LLM_BUDGET_ACTION`, `LLM_PRICES_FILE` |
| Límites de uso | `RATE_LIMIT_ENABLED`, `RATE_LIMIT_IA`, `RATE_LIMIT_INTEGRACIONES`, `RATE_LIMIT_ESCRITURAS`, `RATE_LIMIT_TELEMETRIA`, `RATE_LIMIT_LOGIN`, `MAX_CONCURRENT_QUOTES`, `MAX_CONCURRENT_QUOTES_RETRY_SECONDS` |
| Recursos | `LOOP_LAG_CHECK_SECONDS`, `LOOP_LAG_WARN_MS`, `RUNTIME_SAMPLE_SECONDS`, `MONITOR_ALLOWED_HOSTS` |
| Alertas | `ALERTS_ENABLED`, `ALERTS_CHECK_SECONDS`, `ALERTS_REPEAT_MINUTES`, `ALERT_ERROR_RATE_PCT`, `ALERT_ERROR_RATE_WINDOW_MINUTES`, `ALERT_MIN_REQUESTS`, `ALERT_QUOTE_P95_SECONDS`, `ALERT_QUOTE_WINDOW_MINUTES`, `ALERT_QUOTE_MIN_SAMPLES`, `ALERT_LOGIN_FAILURES`, `ALERT_LOOP_LAG_MS`, `ALERT_DMS_BUSINESS_HOURS`, `ALERT_DMS_BUSINESS_DAYS` |
| Telemetría del front | `TELEMETRY_MAX_EVENTS`, `TELEMETRY_MAX_BYTES` (y `VITE_TELEMETRY` en el front) |
| Trazas | `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_TRACES_SAMPLE_RATIO`, `OTEL_EXCLUDED_URLS` |
| Retención | `RETENTION_REQUESTS_DAYS`, `RETENTION_EVENTS_DAYS`, `RETENTION_LLM_CALLS_DAYS`, `RETENTION_QUOTES_DAYS`, `RETENTION_DMS_UPLOADS_DAYS`, `RETENTION_RUNTIME_DAYS`, `RETENTION_ALERTS_DAYS`, `RETENTION_FRONTEND_DAYS` |

---

## 19. Problemas frecuentes

| Síntoma | Qué mirar |
|---|---|
| Monitoreo no aparece en el menú o responde 403 | Se está entrando desde otra PC. Hasta que haya login, solo desde la del servidor (o sumar la IP a `MONITOR_ALLOWED_HOSTS`) |
| Monitoreo no muestra lo último | Los registros se escriben cada 2 s; si sigue sin aparecer, `/salud/detalle` → `registros` (pendientes, descartados, lotes fallidos) y el log (`Observabilidad: no se pudieron guardar…`) |
| Los vendedores ven «Hiciste muchas consultas…» | Monitoreo → Rendimiento → columna Límite, y el log (`Límite de uso: grupo=…`). Subir `RATE_LIMIT_<GRUPO>` si el uso es legítimo |
| El asesor dejó de sugerir | Monitoreo → Costos IA: ¿se llegó al tope? Ajustar `LLM_DAILY_BUDGET_USD` o esperar a mañana |
| Avisos de «Event loop: se atrasó» | El aviso dice qué requests estaban en curso; correr `scripts\perfil.ps1 -Modo cpu` mientras se reproduce |
| `perfil.ps1` falla con permisos | Abrir PowerShell como administrador |
| No llegan trazas | ¿`uv sync --extra otel`? ¿`OTEL_EXPORTER_OTLP_ENDPOINT`? El log al arrancar dice «Trazas: activas…» o por qué no |
