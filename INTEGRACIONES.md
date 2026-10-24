# Integraciones EPC, SAP y DMS del taller — cómo funcionan por dentro

Registro técnico de lo que se descubrió al analizar y reescribir las integraciones con
el EPC de Nissan (Microcat) y con el portal SAP de Nissan Argentina (octubre 2026).
Todo lo que figura como **medido** salió de pruebas reales contra los sistemas en producción;
lo marcado como **inferido** es interpretación que no se pudo confirmar.

Ninguna de las dos APIs está documentada ni tiene contrato: son las APIs internas que usan
las propias páginas web. Pueden cambiar sin aviso (ver [Mantenimiento](#mantenimiento)).

---

## Resumen

| Sistema | Antes | Ahora |
|---|---|---|
| **EPC** | Playwright manejaba la pantalla como una persona (tipear chasis, tipear familia, leer el resultado de la grilla), una familia por vez y con esperas fijas | Playwright **sólo hace el login** y deja la pestaña abierta; los datos se piden a la **API interna del EPC** con `fetch()` desde esa pestaña, en paralelo |
| **SAP** | Playwright hacía el login completo (esperando que cargue todo el portal) y el stock se pedía por HTTP, de a una consulta por vez | Igual esquema, pero el login corta apenas hay sesión, las cookies se guardan en disco, el re-login suele ser silencioso (SSO) y las consultas van en paralelo |

| Medición | Antes | Ahora |
|---|---|---|
| EPC, 8 ítems / 17 familias (sesión abierta) | 63 s | **3,3 s** |
| EPC, misma cotización repetida | 63 s | 0,07 s (caché de la app) |
| EPC, buscar vehículo / elegirlo | ≥3 s | 0,5–0,9 s / 2 ms |
| EPC, login | 17 s | ~11 s, y se hace solo al arrancar |
| Cotización de 11 ítems de punta a punta | ~1 min | 11,9 s (7,1 s son de Claude) |
| SAP, 8 consultas simultáneas | ~14 s (en fila) | **2,2 s** |
| SAP, login completo | 18 s | 11,4 s |
| SAP, primera consulta tras reiniciar el backend | 23 s | **2,1 s** (sin login) |
| SAP, re-login con la sesión de IAS viva | 18 s | 3 s (sin formulario) |

---

## Glosario

- **EPC / Microcat**: catálogo electrónico de repuestos de Nissan, provisto por Infomedia
  dentro del portal superservice.com. Es una app Angular que le habla a una API REST propia.
- **Licencia nominal**: la suscripción del EPC es para un usuario con *un dispositivo a la vez*.
  Abrir el EPC en otro lado le quita la licencia al anterior.
- **Heartbeat**: pedido periódico que la app del EPC le manda al servidor para avisar
  "sigo acá" y conservar la licencia.
- **JWT**: formato del token de acceso del EPC (`Authorization: Bearer eyJ…`). Lleva adentro
  su fecha de vencimiento (`exp`), que se puede leer sin llamar a nadie.
- **SAP BTP / Cloud Foundry**: la plataforma de SAP donde Nissan publica el portal de concesionarios.
- **Launchpad (Work Zone)**: el portal web de SAP (`/site/dealerportalarg`), hecho con SAPUI5.
- **Approuter**: el "portero" del launchpad. Guarda la sesión del portal (cookie `JSESSIONID`)
  y reenvía los pedidos a los sistemas de atrás (*destinations*).
- **XSUAA**: el servicio de autorización de BTP. Recibe la identidad desde IAS y le entrega
  un token OAuth al approuter.
- **IAS (SAP Cloud Identity Services – Identity Authentication)**: el proveedor de identidad
  donde se escribe usuario y contraseña (`argf04rxd.accounts.ondemand.com`). Tiene **su propia
  sesión**, separada de la del portal.
- **SAML**: protocolo con el que IAS le dice a XSUAA "este usuario ya se autenticó".
- **SSO silencioso**: cuando vence la sesión del portal pero la de IAS sigue viva, el portal
  manda al usuario a IAS, IAS lo reconoce y lo devuelve autenticado **sin mostrar el formulario**.
  Es lo mismo que pasa con "Iniciar sesión con Google" cuando ya estás logueado en Google.
- **CPI (Cloud Platform Integration)**: el middleware de SAP que traduce el pedido del portal
  y consulta el S/4HANA de Nissan (donde vive el stock).

---

## EPC (Microcat / superservice.com)

### Mapa

| Host | Qué es |
|---|---|
| `login.superservice.com` | Portal de login y landing ("Iniciar") |
| `microcat-america.superservice.com/content/microcat-epc/` | La app Angular del EPC (se abre en pestaña nueva) |
| `microcat-america.superservice.com/ver/microcat/epc-html/…` | **La API REST** del EPC |
| `presence-america.superservice.com` | Presencia/licenciamiento del portal |

### Login (medido)

| t (s) | Paso |
|---|---|
| 0,7 | Inicio |
| 1,2 | Página de login |
| 2,3 | `POST /api/v1/auth/login` → 303 |
| 2,8 | Landing (`/landingpage/…`) |
| 5,6 | Click en "Iniciar" → se abre la pestaña del EPC |
| 9,8 | Primer pedido autenticado de la app (`/v1/ui-config`) → **acá ya se pueden copiar los headers** |
| 10,1 | `POST /v1/sessions/device` (la app registra el dispositivo) |
| 13,9 | La app termina de inicializar la pantalla |
| 17,1 | El código viejo recién devolvía el control (esperaba 4 s más por si aparecía el diálogo de licencia) |

El código nuevo da la sesión por lista después de capturar los headers, ver el registro del
dispositivo y verificar la licencia: **~10,5–11 s**.

### Autenticación de la API

La app manda estos headers en cada pedido a `/ver/microcat/epc-html/…`:

| Header | Valor |
|---|---|
| `authorization` | `Bearer <JWT>`; vence a las **~20 h** (`exp` del JWT; el login API devuelve `expiresInSeconds: 72000`) |
| `x-ifm-session-id` | UUID generado por la pestaña; identifica al "dispositivo" para la licencia |
| `x-ifm-sid` | Id de la suscripción de la cuenta (fijo) |
| `x-ifm-franchise` | `NMC` (Nissan Microcat) |
| `x-ifm-encoding` | `gzip, deflate, br` |
| `accept`, `content-type` | JSON |

Además, los GET llevan el parámetro `afx`, un hash que **cambia con cada versión de la app**.
El valor fijo que tenía el viejo `epc_scraper_httpx.py` (`3fd25ca6`) ya estaba vencido en
octubre 2026. El código nuevo lo toma de los pedidos de la propia app.

### Endpoints

**Buscar vehículo** — `GET /v3/vehicle/search?searchTerm=<VIN completo o últimos 8>` + parámetros comunes.
Devuelve `vehicles[]` con `vin`, `id`, `label`, `shortName`, `catalogCode`, `buildDate` (`YYYYMMDD`),
`model.code`, `majorAttributes` (`ENGINE`, `GRADE`, `TRANS`, `BODY`, `BODY_COLOR`, `MODEL`).
Con 8 dígitos puede devolver varios vehículos. Medido: 0,5–0,9 s.

El `id` es un base64 opaco. Decodificado parece protobuf con el VIN, `NMC` y dos campos de
20 bytes (**inferido**: hashes). Se trata como opaco y se cachea por sesión.

**Buscar familia** — `GET /v1/globalsearch/search?q=<familia>&vehicleId=<id>` + parámetros comunes.
Medido: 0,3–1,3 s (≈0,7 s promedio). Los números están en:

```
matchResult.hasMatches
results[].vehicleResults[].partResults.partLists[].partSearch
        .catalogWithParts[].parts[].partIdentifications.partFormats[].value   → "16546-5RA0A"
```

El EPC también devuelve piezas relacionadas de **otras familias**, así que se filtran las que
empiezan con la familia buscada (igual que hacía el scraper de pantalla).

**Parámetros comunes** (los mismos que manda la app):
`market=AR, language=es, dataLanguage=es, useSynonyms=true, crossCatalog=false, byPassTsl=false,
includeNonApplicable=false, showOrbSupersession=false, interpretationAttributes=ALL_ATTRIBUTES,
region=GL, afx=<actual>`.

**Sesión y licencia**:
- `POST /v1/sessions/heartbeat` (cuerpo `{}`): 200 si tenemos la licencia; 403 + header
  `x-ifm-licensing-error` si la tiene otro dispositivo.
- `POST /v1/sessions/device` (cuerpo `{}`): la app registra el dispositivo al iniciar.
- `POST /v1/sessions/device?source=use_license`: **tomar la licencia** (lo que hace el botón
  "Usar licencia" / "Use Licence").

Otros endpoints que la app llama al iniciar (no se usan): `ui-config`, `settings`, `users/account`,
`applicationfeatures`, `features`, `pricing/*`, `catalog/*`, `whatsnew`, `history/*`, `job/list`,
`currentuser/subscriptions`, `searches/options`. También manda métricas a New Relic y
`/epc-html-usage-report/v1/usage-report/usage`.

### Licencia (medido + bundle de la app)

Lo siguiente sale de pruebas con dos sesiones compitiendo y de leer el JS de la app
(`main.<hash>.js`, ver [Cómo re-investigar](#cómo-re-investigar)):

- **Heartbeat cada 90 s** para suscripciones nominales (`NAMED_USERS_HEART_BEAT_INTERVAL=9e4`;
  las concurrentes usan 15 s). El primero sale 90 s después de iniciar la app.
- Para usuarios nominales **el heartbeat se corta cuando la ventana pierde el foco** (`onBlur`)
  y vuelve con `onFocus`. En headless no pasa; con `EPC_HEADFUL=true` y la ventana en segundo
  plano, sí.
- Mientras el **diálogo "Licencia en uso"** está abierto, la app **deja de mandar heartbeats**
  (`heartbeatEnabled = !displayUserLicense`). **Inferido**: esa era la causa de las sesiones
  que "se cerraban por inactividad" con el scraper viejo.
- La app chequea la licencia en el heartbeat y antes de ciertas acciones del usuario
  (`userActiveLicenceCheck`). Por eso el diálogo podía aparecer **tarde**, encima de la pantalla,
  y bloquear los clicks del scraper viejo (pasó 1 de cada 4 logins en las pruebas).
- El servidor avisa el conflicto con **403 + `x-ifm-licensing-error`** en cualquier pedido.
- El botón "Usar licencia" llama a `POST /v1/sessions/device?source=use_license` y después
  recarga la página. El botón queda deshabilitado con una cuenta regresiva.
- **Enfriamiento de ~60 s** (medido: `Retry-After: 56` y `48`): después de que un dispositivo
  toma la licencia, nadie puede retomarla durante ese lapso, **ni siquiera con un login nuevo**.
  Fuera de ese lapso, un login nuevo se queda con la licencia sin diálogo.
- En 18 pedidos simultáneos desde la misma pestaña no hubo ningún 403: el paralelismo dentro
  de la misma sesión no cuenta como "otro dispositivo".

### Duración de la sesión (medido)

- Con la pestaña abierta y los heartbeats andando, la sesión **seguía viva después de 34 min
  sin consultas** (la consulta tardó 1,75 s, sin re-login). Los 15 min de inactividad / 30 min
  máximo que suponía el código viejo no se cumplen.
- El token vence a las ~20 h. Al vencer, la API responde 401 y el backend re-loguea solo.

### Caché del lado del cliente (medido)

Dentro de una misma pestaña, repetir un pedido idéntico vuelve en ~8 ms aunque se use
`cache: 'no-store'`. **Inferido**: lo sirve el Service Worker de la app. Por eso repetir una
cotización del mismo vehículo tarda 0,07 s. Para medir tiempos reales hay que usar familias
que no se hayan pedido antes en esa pestaña.

### Por qué era lento el scraper de pantalla

- Esperas fijas: 3 s al identificar el chasis, 300 ms por familia y 500 ms por grupo.
- Cada familia **sin resultados** esperaba timeouts de hasta 5 s + 2 s.
- Todo en serie, con un lock global: una familia por vez, un usuario por vez.
- Cada búsqueda disparaba además ~10 pedidos de historial de la propia pantalla.

Medido con el mismo chasis y las mismas 17 familias: 31 s aislado y 63 s a través del backend.
Por API, secuencial: ~0,7 s por familia; 12 familias en paralelo: 1,36 s.
**Resultados idénticos** a los de la pantalla en 18 grupos de 2 chasis (incluido un grupo sin resultado).

### Por qué no "todo por HTTP sin navegador"

El intento anterior (`epc_scraper_httpx.py`, borrado) hacía el login por API y llamaba a la API
con httpx. Problemas:
- La cookie `Authorization` del dominio `.superservice.com` + el header → el EPC contaba dos
  dispositivos y respondía 403 "otro dispositivo" (según las notas del propio archivo).
- `afx` fijo y vencido.
- No resolvía la licencia ni el heartbeat.

Con `fetch()` desde la pestaña, los pedidos son indistinguibles de los de la app: mismas
cookies, mismo `x-ifm-session-id` y la app sigue manteniendo la licencia. La única ventaja
de sacar el navegador sería el login (~11 s), y eso ya pasa en segundo plano al arrancar.

---

## SAP (portal de concesionarios Nissan Argentina)

### Mapa

| Host | Qué es |
|---|---|
| `nissanargcf.launchpad.cfapps.us10.hana.ondemand.com` | Launchpad + approuter (la sesión que importa) |
| `nissanargcf.authentication.us10.hana.ondemand.com` | XSUAA |
| `argf04rxd.accounts.ondemand.com` | IAS (formulario de usuario y contraseña) |
| `sapui5.hana.ondemand.com`, `portal-service…`, `sapdas…` | Recursos del launchpad (no hacen falta para el stock) |

El stock es una ruta del approuter hacia la *destination* `CPI_NNA_S4H`:

```
POST /3689b859-d9d2-4e0f-bb15-ed4e675f11ba.ardprepstockdealer.arcomnissanardprepstockdealer
     /~67339fe8-7319-41a7-91d9-03453d9d7c9e~/destinations/CPI_NNA_S4H
     /AR/DealerPortal/Reporte/Stock/MaterialesSustitutos
```

### Login (medido, flujo viejo)

| t (s) | Paso |
|---|---|
| 1,3 | `GET /site/dealerportalarg` → 200, pero es una página que redirige por JS al login |
| 1,9 | XSUAA `/oauth/authorize` → `/login` → `/saml2/authenticate/<IAS>` |
| 4,4 | IAS muestra el formulario (primero el email, después la contraseña) |
| 7,5 | `POST` email |
| 10,4 | `POST` contraseña |
| 11,1 | IAS devuelve el SAML a XSUAA (`/saml/SSO/alias/nissanargcf.aws-live`) |
| **11,6** | **Launchpad `/login/callback` → la sesión del portal ya existe** |
| 12,0–18,1 | Carga del launchpad: 98 pedidos de SAPUI5, fuentes, CSS… **inútil para el stock** |

El código viejo además esperaba `networkidle` al abrir la página y tenía un `sleep` fijo de 2 s
entre email y contraseña. El código nuevo corta en `/login/callback`: **11,4 s** con formulario
y **3 s** con SSO silencioso.

### Cookies (medido)

| Cookie | Dominio | Vida | Para qué |
|---|---|---|---|
| `JSESSIONID`, `__VCAP_ID__`, `__VCAP_ID_META__` | launchpad | sesión | **La sesión del portal: es lo único que necesita el pedido de stock** |
| `JSESSIONID`, `IDP_USER`, `__HOST-IDP_J_COOKIE`, `__HOST-XSRF_COOKIE` | IAS | sesión | **La sesión de IAS**: permite el SSO silencioso |
| `__HOST-authIdentifierData` | IAS | ~400 días | Recuerda el usuario (puede saltear el paso del email) |
| `X-Uaa-Csrf`, `__Host-JSESSIONID`, `__VCAP_ID__` | XSUAA | 24 h / sesión | Flujo OAuth |

"Sesión" quiere decir que el navegador la borraría al cerrarse, pero el servidor la sigue
aceptando mientras no venza. El backend las guarda en `SAP_SESSION_FILE`
(`~/.cotizador/sap_session.json`, formato `storage_state` de Playwright). **Ese archivo da
acceso a SAP mientras las sesiones estén vivas: no compartirlo ni subirlo al repo.**

### Validar o mantener viva la sesión (medido)

| Pedido | Con sesión | Sin sesión |
|---|---|---|
| `GET /cp.portal/portal/rest/v1/personalization` + `X-Requested-With: XMLHttpRequest` | **200, vacío, ~190 ms** | **401** |
| `GET /site/dealerportalarg` (sin `X-Requested-With`) | 200 (38 KB) | 200 (783 B, redirección por JS) → **no sirve para validar** |
| `GET /user-api/currentUser` | 404 | 404 (no está configurado) |
| `GET` al path de stock | **500 de CPI** (deja un error registrado en CPI de Nissan) → **no usar** | 401 |

El backend usa `personalization` para validar las cookies guardadas y como keep-alive cada 5 min.

### Inactividad (medido con 4 sesiones, cada una chequeada una sola vez)

| Sin uso | Sesión del portal | Sesión de IAS |
|---|---|---|
| 12 min | viva | — |
| 18 min | viva | — |
| 25 min | viva | — |
| 35 min | **vencida** | **viva** → re-login sin formulario |

El timeout del portal está entre 25 y 35 min (**inferido**: 30, el típico del approuter).
No se midió cuánto dura la sesión de IAS más allá de 35 min.

### Consulta de stock

Cuerpo:

```json
{"HeaderSet": {"Header": {"Material": "", "Nav_Header_Materiales": {
  "Materiales": [{"Material": "620225RA0H", "Dealer": "232111"}]}}}}
```

Particularidades de la respuesta: `Materiales` viene como **objeto** si hay uno y como
**lista** si hay varios; `Cantna06` (stock en almacén 06) es un texto con espacios
(`"              0.000"`). Si la sesión venció puede volver 401/403 o un **200 con HTML** del login.

| Medición | Tiempo |
|---|---|
| Cliente HTTP nuevo por consulta (código viejo) | 1,9–3,6 s |
| Cliente persistente (código nuevo) | 1,3–2,0 s |
| 20 materiales en una consulta | 2,1 s (casi lo mismo que 3) |
| 8 consultas simultáneas | ~2,2 s en total |

El piso de ~1,3 s es del lado de Nissan (approuter → CPI → S/4).

---

## DMS del taller (AutoDealer)

El DMS del taller es AutoDealer (FBS), una aplicación VB6 sobre SQL Server. A diferencia
del EPC y de SAP no tiene web ni API: la única forma de leerlo es la base. Lo de esta sección
salió de dos scripts de diagnóstico de solo lectura corridos en el servidor
(`diagnostico_dms.ps1` y `diagnostico_dms_2.ps1`, en la raíz del workspace) y de los textos
del programa `ADNissan_Automatico.exe` (octubre 2026).

### Mapa (medido)

| Qué | Valor |
|---|---|
| Servidor | `WIN-HOLCOH2SPLV` (192.168.1.10): Windows Server 2012 R2, PowerShell 4.0, .NET 4.8 |
| Base | `Sai` en la instancia `SQLEXPRESS`: SQL Server 2012 **RTM** (11.0.2100, sin service packs), edición Standard |
| Acceso | TCP 1433 abierto en la red del taller; desde afuera no se llega |
| Logins | `AutoDealer` (el de la aplicación) es db_owner; el Administrador de Windows es sysadmin |

### De dónde sale el stock (medido)

No hay una columna de stock: se calcula con los movimientos de `KardexTR`, por repuesto y depósito.

```sql
SUM(CASE WHEN Tipo = 'I' THEN Cantidad WHEN Tipo = 'E' THEN -Cantidad ELSE 0 END)
```

- `Tipo`: `I` ingreso, `E` egreso, `N` no cuenta (no hay movimientos `N` desde 2024).
- Es la misma cuenta que hacen la vista propia del DMS `Existencias` y el archivo REFINV de
  `ADNissan_Automatico`: en 5 repuestos de muestra dieron igual. Conviene la cuenta directa
  porque tarda **0,13 s** para todo el stock (230 mil movimientos) contra 0,9 s de la vista,
  y sólo necesita leer `KardexTR`.
- Resultado: ~1.900 combinaciones repuesto/depósito con stock, ~5.800 unidades.
- Depósitos: CENTRAL, RACK, DEPOSITO 3, PAJARERA, PICKING y GARANTIAS (vacío hoy; se excluye
  por defecto). Las columnas de `Depositos` (`Garantia`, `Activo`, `Principal`) no los distinguen.
  Hay movimientos con depósito vacío (7 repuestos, 18 unidades).
- Reservas: el DMS las maneja como un depósito `RESERVA` (vista `StockDepositoReserva`), que en
  este taller no existe.
- Códigos: 44.446 de 44.726 repuestos tienen el formato del EPC sin guión (`165465RA0A`); el
  resto tiene guiones o espacios, que se quitan al comparar.
- `Repuestos.Reemplazoa` registra reemplazos (5.147 repuestos): **inferido**, sirve para sumar el
  stock de un número viejo cuando el EPC devuelve el nuevo. No se usa todavía.

`ADNissan_Automatico.exe` (tarea programada "Simetrical", todos los días a las 22:00) genera los
archivos para Simetrical en `C:\Buzon\Output` y los manda por FTP a la 1:00. Su REFINV es el
stock del día anterior: no sirve para cotizar.

### Por qué un agente que sube el stock

El backend corre en la nube. Conectarlo directo a la base exigiría publicar el puerto 1433 en
internet con un SQL Server sin parches desde 2012 (sólo TLS 1.0) y logins que pueden escribir.
En cambio, un agente en el servidor del taller (tarea programada en PowerShell 4, sin instalar
nada) calcula el stock cada pocos minutos y lo sube por HTTPS a `PUT /dms/stock`; sólo hay
conexiones salientes. Medido desde el servidor: sale a internet sin proxy, pero .NET usa
SSL 3/TLS 1.0 por defecto, así que el agente tiene que forzar TLS 1.2.

Sube siempre la foto completa, no las diferencias: si una subida falla, la siguiente corrige todo.

## Cómo está implementado

### `app/services/epc_scraper.py`

- `EPCScraper` corre Playwright en **un hilo propio con su loop**: en Windows uvicorn usa un
  loop que no soporta subprocesos y Playwright los necesita.
- `_open_session()`: login → pestaña del EPC → espera headers + registro del dispositivo →
  `_check_license()` (heartbeat propio; si da 403, `_claim_license()`).
- `_capture_api_headers()`: escucha todos los pedidos del contexto y se queda con los últimos
  headers, el `afx` y el vencimiento del JWT.
- `_watch_session_responses()`: si el heartbeat de la app da 403 o 401, marca la licencia o la
  sesión como perdidas. Se recuperan en la próxima consulta: el backend **no le roba la licencia
  a nadie en segundo plano**.
- `_fetch()`: ejecuta `fetch()` dentro de la pestaña; hasta 6 pedidos a la vez (`_MAX_PARALLEL_REQUESTS`).
- `_api_get()`: ante 403 de licencia, 401 o respuesta no JSON, recupera una vez y reintenta.
  Un contador de **generación** evita que varios pedidos que fallan juntos re-logueen varias veces.
- `search_group()`: consulta **en paralelo todas las familias equivalentes** y gana la primera,
  en orden, con resultado; cancela el resto.
- `identify_chassis()`: VIN → `id`, cacheado por sesión. Cada búsqueda indica su chasis:
  ya no existe el "chasis cargado en pantalla", así que dos cotizaciones simultáneas no se pisan.
- Precalentamiento al arrancar: si choca con el enfriamiento de la licencia, espera el
  `Retry-After` y reintenta una vez.
- `_release_when_idle()`: cierra la pestaña tras `SESSION_KEEPALIVE_HOURS` sin uso, para liberar la licencia.
- El conflicto de licencia dentro del enfriamiento **no se esquiva**: se informa
  "Reintentá en N s". Forzarlo armaría un ida y vuelta de licencia con quien la esté usando.

### `app/services/sap_stock.py`

- `_open_session()`: 1) cookies guardadas + chequeo con `personalization`; 2) si no sirven,
  login con Playwright **cargando las cookies guardadas** (así IAS puede hacer SSO silencioso).
- `_complete_login()`: bucle que completa lo que IAS pida (email y contraseña en dos pasos, en
  uno solo, o nada si hay SSO) y termina cuando ve `/login/callback`. Registra en el log si hubo
  formulario o SSO.
- Protección contra bloqueo de la cuenta: un rechazo de credenciales frena los reintentos hasta
  reiniciar con credenciales nuevas; SAP caído → 60 s sin reintentar.
- Un `httpx.AsyncClient` persistente con las cookies por dominio. **Sólo el login se serializa**:
  las consultas corren en paralelo.
- `_keep_alive()`: cada 5 min, mientras haya habido uso dentro de `SESSION_KEEPALIVE_HOURS`;
  si la sesión venció, la renueva en segundo plano.

### `app/services/cotizar_service.py`

- La llamada a Claude corre con `asyncio.to_thread`: el SDK es síncrono y antes bloqueaba
  todo el servidor mientras respondía.
- `/cotizar/stream` emite cada resultado del EPC apenas llega (el front lo ubica por `index`).

### `app/services/dms_stock.py`

- Guarda la última foto en memoria, indexada por código normalizado (sin guiones ni espacios,
  en mayúsculas), y la persiste en `DMS_STOCK_FILE`. Un archivo roto o ausente no es error:
  se arranca sin foto y el agente la repone.
- Al recibir una foto descarta los depósitos excluidos y las existencias nulas o negativas, y
  suma los depósitos repetidos de un mismo código.
- La antigüedad se mide con el reloj del backend al recibir la foto, no con el del servidor del taller.
- El router (`app/routers/dms.py`) exige `Authorization: Bearer <DMS_AGENT_TOKEN>` sólo en la
  subida; sin token configurado la subida responde 503.

### El agente (`Cotizador_BE/agente_dms/`)

Instalación y operación: ver el `README.md` de esa carpeta.

- `instalar_agente_dms.ps1` crea el login `cotizador_ro` con `SELECT` sobre `KardexTR` y `DENY` de
  escritura sobre el esquema `dbo` (le gana a cualquier permiso del rol `public`), y verifica con
  ese login que no pueda escribir antes de guardar la configuración. La contraseña es aleatoria y
  queda en `agente.config.json`, que sólo pueden leer Administradores y SYSTEM.
- La tarea programada corre como SYSTEM cada 5 minutos (disparador diario que se repite, porque la
  repetición "indefinida" falla en algunas versiones de Windows), no arranca otra corrida si la
  anterior sigue y corta a los 4 minutos.
- `agente_stock_dms.ps1` fuerza TLS 1.2, sube la foto en un solo `PUT` (~2.000 filas, menos de
  100 ms medido contra un backend local) y deja una línea por corrida en `logs\`. Con `-Prueba`
  guarda la foto en `stock_prueba.json` sin subirla.

### Configuración (`.env`)

| Variable | Default | Uso |
|---|---|---|
| `SAP_SESSION_FILE` | `~/.cotizador/sap_session.json` | Cookies de SAP entre reinicios |
| `SESSION_KEEPALIVE_HOURS` | `3` | Horas sin uso tras las que se liberan EPC y SAP |
| `WARMUP_ON_START` | `true` | Abrir ambas sesiones al arrancar el backend |
| `DMS_AGENT_TOKEN` | vacío | Token del agente del taller; vacío deshabilita la subida |
| `DMS_STOCK_FILE` | `data/dms_stock.json` | Última foto del stock del taller entre reinicios |
| `DMS_STALE_MINUTES` | `15` | Antigüedad a partir de la cual la foto está desactualizada |
| `DMS_EXCLUDED_DEPOSITS` | `GARANTIAS` | Depósitos que no cuentan como stock (separados por coma) |

### Trampa conocida: `uvicorn --reload` en Windows

Al recargar, uvicorn le manda Ctrl+C al proceso y **espera** a que termine el apagado
(`lifespan`). Si `close()` se cuelga, el backend queda muerto y no vuelve a recargar.
Pasó una vez al cancelar las tareas internas de Playwright: `browser.close()` quedaba esperando
para siempre. Por eso `close()` sólo cancela tareas propias y tiene un timeout de 10 s.
Con recargas seguidas (menos de 60 s entre una y otra) el precalentamiento del EPC choca con
el enfriamiento de la licencia y tarda un minuto más; es esperable.

---

## Mantenimiento

Qué puede romperse si Infomedia o Nissan cambian algo, y cómo se nota:

| Cambio | Síntoma | Dónde mirar |
|---|---|---|
| Pantalla de login de superservice o el botón "Iniciar" | Timeout en el login del EPC | `_open_epc_tab()` |
| Ruta de la API (`/ver/microcat/epc-html`) | "El EPC no terminó de iniciar" | `_API_PATH` |
| Forma de la respuesta de `globalsearch` | Todas las familias "sin resultados" | `_iter_raw_part_numbers()` |
| Parámetros de búsqueda | 400 en la API | `_SEARCH_PARAMS` (comparar con los pedidos de la app) |
| Formulario de IAS | `SAPAuthError` o "SAP no completó el login a tiempo" | selectores en `sap_stock.py` |
| Nueva versión de la app de stock en BTP (**inferido**: el segmento `~67339fe8…~` cambiaría) | 404 en el stock | `_SAP_STOCK_PATH` (copiarlo de las DevTools del portal) |
| El agente del taller deja de subir (servidor apagado, sin internet, tarea deshabilitada) | `desactualizado: true` | `GET /dms/stock/estado` y el log de la tarea en el servidor |
| Una actualización del DMS cambia `KardexTR` | El agente falla o el stock da distinto que la pantalla del DMS | Comparar la consulta del agente con la vista `Existencias` |

## Cómo re-investigar

- **Ver lo que manda la app**: con Playwright, `context.on("request", …)` y
  `await request.all_headers()` muestran URL, parámetros y headers reales de cada pedido.
  Filtrar los pseudo-headers de HTTP/2 (`:authority`, etc.) antes de reutilizarlos en `fetch()`.
- **Leer la lógica del EPC**: el bundle es público. `GET https://microcat-america.superservice.com/content/microcat-epc/`
  lista los scripts; el principal es `main.<hash>.js` (~7 MB, minificado en una sola línea).
  Buscar con Python (`str.find`), no con `grep -o '.{0,200}…'`: con regex sobre una línea de
  7 MB, `grep` tarda minutos. Términos útiles: `HEART_BEAT_INTERVAL`, `x-ifm-licensing-error`,
  `use_license`, `userActiveLicenceCheck`, `epc-roaming-user-dialog`.
- **Línea de tiempo de un login**: `context.on("response", …)` registrando `status`, método,
  tipo de recurso y URL con su tiempo relativo.
- **Medir sin caché**: familias que no se hayan pedido en esa pestaña (ver la caché de la app).
- **Medir inactividad**: una sesión por cada intervalo a probar, chequeada **una sola vez**,
  porque el propio chequeo cuenta como actividad.
- **No** probar con contraseñas incorrectas en SAP (bloqueo de cuenta), **no** hacer GET al path
  de stock (genera errores en CPI) y tener en cuenta que cada login del EPC le saca la licencia
  al backend que esté corriendo.
