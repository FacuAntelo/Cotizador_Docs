# Usuarios, permisos y organizaciones — plan

Estado: plan acordado con el usuario el 8 de noviembre de 2026 y repasado el 9. Se construye en
la rama `feat/usuarios` de los tres repositorios: la etapa 1 está hecha (ver
[Cómo quedó la etapa 1](#cómo-quedó-la-etapa-1)).
Lo que queda por resolver está en [Abiertas](#abiertas).

Cómo engancha el login con la observabilidad: [LOGIN_Y_OBSERVABILIDAD.md](LOGIN_Y_OBSERVABILIDAD.md).

## Qué se busca

- Que cada persona entre con su usuario y vea solo lo que sus roles le permiten.
- Que el sistema sirva para una **organización con varias sucursales**, cada una con sus
  depósitos, y para varias organizaciones.
- Un **administrador de plataforma** (el dueño del sistema) que da de alta organizaciones,
  sucursales, depósitos y usuarios, carga la lista de precios de Nissan y es el único que ve
  **Monitoreo**.
- Que los **costos** y el **margen** los vea y los cambie solo quien tenga permiso.
- Cobrar **por usuario que cotiza**: los usuarios que solo administran no se cobran.
- Que **cada usuario use sus propias cuentas de EPC, SAP y el localizador**, y que dos
  usuarios con la misma cuenta compartan una sola sesión.
- Que un usuario se use **en un solo lugar a la vez**, sin tener que loguearse todos los días.

## Cómo se organiza

```
Plataforma (administrador de plataforma): la lista de precios y las compatibilidades de Nissan
└── Organización   razón social, CUIT, logo, usuarios que cotizan contratados, margen y
                   estrategia de cotización
    └── Sucursal   dirección, teléfono, mail, código de concesionario de Nissan,
                   ubicación para el localizador,
                   precios de taller, su servidor con el DMS y el agente que sube el stock
        └── Depósito   nombre en el DMS, si suma al stock por defecto, si se ignora
Usuario → una organización, uno o más roles, una o más sucursales (una es la de por defecto),
          sus cuentas de EPC, SAP y localizador
```

## Tablas nuevas

Todas en el schema `app`. La base arrancó vacía, así que la organización y la sucursal entran
en las tablas actuales como columnas obligatorias desde el principio, sin pasar datos viejos.

| Tabla | Columnas principales |
|---|---|
| `organizations` | nombre, razón social, CUIT, logo, `cotizador_seats` (usuarios que cotizan contratados), `dealer_margin_percent`, `quote_strategy`, activa |
| `branches` | organización, nombre, dirección, teléfono, mail, `dealer_code` (código de concesionario de Nissan, puede repetirse entre sucursales), latitud y longitud, activa |
| `deposits` | sucursal, código en el DMS (único dentro de la sucursal: dos sucursales pueden tener su `PICKING`), nombre, `counts_by_default`, `ignored` |
| `users` | organización (vacía solo para el administrador de plataforma), usuario (único en todo el sistema), nombre a mostrar, hash de la contraseña, `must_change_password`, código de verificación (secreto cifrado y si está activo), `active_session_id`, activo, intentos fallidos, bloqueado hasta, último ingreso, creado por |
| `roles` | organización (vacía para los roles del sistema), nombre, si es del sistema |
| `role_permissions` | rol, permiso. Los permisos válidos los define el código (un `CHECK` con la misma lista, como los estados) |
| `user_roles` | usuario, rol |
| `user_branches` | usuario, sucursal, si es la de por defecto |
| `sessions` | una por PC o navegador: usuario, hash del token de la cookie y del de renovación, si está atada a la PC (DBSC) y su clave pública, creada, último uso, vence, sucursal activa, IP, navegador, revocada |
| `audit_log` | cuándo, organización, quién, qué hizo, sobre quién o qué, IP, detalle |
| `integration_accounts` | usuario, servicio (`epc`, `sap`, `localizador`), usuario de la cuenta, contraseña cifrada, huella, estado y último error |
| `dms_agents` | sucursal, nombre, hash del token, última subida |

Un usuario puede tener **varios roles** (`user_roles`), y sus permisos son la suma.

## De quién es cada dato

| Hoy | Pasa a ser de | Cambio |
|---|---|---|
| `estimates` y lo que cuelga (renglones, pedidos, insumos, orden del seguro) | Organización, con la sucursal donde se hizo | El número es único **por organización** (`uq_estimates_org_number`) y el bloqueo que lo asigna, una clave por organización. Se ven los de toda la organización |
| `loans` | Sucursal | — |
| `dms_stock` | Depósito | La clave pasa a ser (depósito, código) |
| `dms_stock_uploads` | Agente del DMS | — |
| `price_items`, `price_conflicts`, `price_variants`, `price_list` | Compartida | La lista de Nissan es la misma para todos y la carga el administrador de plataforma. El margen y la estrategia salen de `price_list` y pasan a `organizations` |
| `part_compatibilities`, `compatibility_list` | Compartida | Ídem: las carga el administrador de plataforma |
| `insurers` | Organización | `uq_insurers_name` pasa a ser por organización |
| `dealer_profile` | Se reparte | Razón social, CUIT y logo en `organizations`; dirección, teléfono y mail en `branches` |
| `workshop_prices` | Sucursal | — |
| `recent_vehicles` | Sucursal | — |
| `catalog_candidates` | Compartida | Es conocimiento del catálogo de Nissan y sirve para que el sistema aprenda. Suma quién la validó |
| `llm_calls`, `quote_runs` | Organización, sucursal y usuario | Ya tienen `user_id` |
| Tablas de `observability` | Suman `organization_id` | Sin foreign key, como el resto de la telemetría |

Auditoría del negocio junto con el login: `created_by` y `accepted_by` en los presupuestos,
`created_by` en préstamos y pedidos.

## Roles y permisos

Los permisos son textos fijos que define el código, porque es el código el que los revisa. Los
roles son grupos de permisos guardados en la base: arrancan los del sistema y más adelante cada
organización podría armar los suyos.

| Permiso | Qué habilita |
|---|---|
| `cotizar` | El cotizador: match, EPC, SAP, localizador y asesor |
| `presupuestos` | Crear, editar y aceptar presupuestos |
| `presupuestos.ver` | Ver y descargar los presupuestos de toda la organización |
| `pedidos` | Confirmar el Excel y registrar llegadas y separaciones |
| `prestamos` | Ver y cargar préstamos |
| `costos.ver` | Ver costo y márgenes en el cotizador, los presupuestos y los pedidos |
| `precios.gestionar` | Margen y estrategia de cotización de la organización |
| `aseguradoras.gestionar` | Alta, baja y modificación de compañías de seguro |
| `sucursal.configurar` | Datos de la sucursal y precios de taller |
| `usuarios.gestionar` | Usuarios de su organización, dentro del cupo, y su registro de seguridad |
| `plataforma.administrar` | Organizaciones, sucursales, depósitos, agentes del DMS, usuarios de todas las organizaciones, la lista de precios y las compatibilidades de Nissan, y el registro de seguridad completo |
| `monitoreo.ver` | Monitoreo y la salud detallada |

Los roles de una organización se combinan:

| Permiso | Cotizador | Encargado | Administrador | Plataforma |
|---|:-:|:-:|:-:|:-:|
| `cotizar`, `presupuestos`, `pedidos`, `prestamos` | ✓ | — | — | — |
| `presupuestos.ver` | ✓ | — | ✓ | — |
| `costos.ver`, `aseguradoras.gestionar`, `sucursal.configurar` | — | ✓ | ✓ | — |
| `precios.gestionar`, `usuarios.gestionar` | — | — | ✓ | — |
| `plataforma.administrar`, `monitoreo.ver` | — | — | — | ✓ |

- Un repuestero es **Cotizador**.
- Un jefe de repuestos es **Cotizador + Encargado**.
- El dueño que cotiza es **Cotizador + Administrador**.
- Alguien que administra y no cotiza es solo **Administrador**, y no ocupa cupo.

El rol **Plataforma** no se combina con los demás: ese usuario no pertenece a ninguna
organización. El dueño del sistema tiene dos usuarios, el de plataforma y uno común para
cotizar: el poderoso se usa poco y se expone menos.

**Ocultar la pantalla no alcanza.** Hoy el cotizador recibe el costo y los márgenes de cada
repuesto (`/precios/consultar`) y los pedidos muestran el costo. Sin `costos.ver`, el backend
no manda esos campos; sin `precios.gestionar`, rechaza el cambio de margen y estrategia con
403. El precio de venta lo ve todo el que cotiza.

**Cada ruta declara su permiso**, como hoy declara su grupo de límite de uso, y un test falla
si alguna ruta no lo dice. Las únicas sin permiso son `/auth/login`, `/salud` y la subida del
agente del DMS, que usa su propio token.

**Nadie se queda sin administrador:** no se puede sacar el rol Administrador ni desactivar al
último que tiene la organización, y nadie puede cambiarse sus propios roles.

## Cupos: usuarios que cotizan

- **Los usuarios son de la organización**, no de una sucursal. Cada uno tiene habilitadas una
  o más sucursales y elige en cuál está trabajando.
- **Se cobra por usuario que cotiza**, que es lo que cuesta (la IA y las sesiones de EPC).
  Cuentan los usuarios activos con el rol Cotizador. Los que solo administran y el
  administrador de plataforma no cuentan.
- El cupo es `organizations.cotizador_seats`, y solo lo cambia el administrador de plataforma.
- Crear o reactivar un usuario Cotizador, o darle el rol a uno que no lo tenía, por encima del
  cupo se rechaza con un mensaje que dice el cupo y cómo ampliarlo.
- Las sucursales las crea solo el administrador de plataforma. Si se cobran, se cobra afuera
  del sistema: el sistema no las limita.
- Administración muestra por organización las sucursales y el cupo (ej. «cotizan 3 de 4»).

## Login y sesiones

- **Usuario y contraseña.** El usuario es único en todo el sistema, así que al entrar no hay
  que elegir organización.
- **Contraseñas con Argon2id.** Mínimo 12 caracteres, sin reglas de composición. Se rechazan
  las muy comunes y las que contienen el usuario.
- **Código de verificación** de 6 dígitos de una app (Google Authenticator o similar), que se
  pide al loguearse: obligatorio para el administrador de plataforma, opcional para los
  Administradores.
- **Cada PC queda recordada**, como en Mercado Libre, y la sesión se protege contra el robo
  de cookies: los virus que las copian del navegador son hoy la forma más común de robar
  cuentas.
  - **Sesiones con estado, no JWT:** la cookie lleva un token al azar de 256 bits que no dice
    nada por sí mismo, y en cada pedido el backend busca su hash en `sessions`. Así una sesión
    se corta al instante (salir, otra PC, robo detectado, usuario desactivado) y un cambio de
    roles vale desde el pedido siguiente. Un JWT no se puede cortar antes de que venza sin
    consultar la base igual, y con un solo servidor no hay nada que ganar: cuesta una consulta
    chica por pedido.
  - **Cookie corta:** la de la sesión dura 10 minutos y se renueva sola. Va con `__Host-` (vale
    solo para el nombre exacto y ningún subdominio la puede pisar), `HttpOnly`, `Secure`,
    `SameSite=Strict` y `Path=/`. `Strict` alcanza porque el front y la API están en el mismo
    nombre.
  - **Cookie de renovación** (solo sin DBSC): `__Secure-`, `HttpOnly`, `Secure`,
    `SameSite=Strict` y el `Path` de la ruta de renovación, así no viaja en los demás pedidos.
  - Al loguearse siempre se crea un token nuevo. Nada de tokens en `localStorage`, y la base
    guarda solo hashes.
  - **HSTS:** el navegador no vuelve a intentar entrar por `http`.
  - **Atada a la PC (DBSC)** en Chrome sobre Windows con TPM (desde Chrome 146): al loguearse,
    el navegador crea una clave en el chip de seguridad que no se puede copiar, y cada
    renovación va firmada con ella. Una cookie robada sirve como mucho 10 minutos y no se
    renueva desde otra máquina.
  - **Sin DBSC** (Firefox, Safari, celulares y PCs sin TPM): la renovación usa un token que
    cambia cada vez. Si aparece uno ya usado, hay dos con la misma sesión: se cierra para los
    dos, queda en el registro de seguridad y se avisa en Monitoreo. Mientras el dueño la usa,
    un robo se detecta en minutos; si el dueño no la usa, el ladrón sigue hasta que vuelva.
  - **Duración:** atadas a la PC, 30 días sin uso y 90 como máximo. Sin DBSC, 7 días sin uso y
    30 como máximo. El administrador de plataforma, 12 horas sin uso y 7 días como máximo.
  - Cada sesión guarda si quedó atada, sin mirar el nombre del navegador, y Mis dispositivos
    lo muestra («Protegida» o «Sin protección»).
- **Lo sensible pide la contraseña de nuevo** (vale 10 minutos): cambiar la contraseña o el
  código de verificación, cargar cuentas de EPC, SAP o el localizador, crear usuarios, cambiar
  roles o cerrar sesiones de otros, generar tokens del DMS y todo lo de plataforma. Con una
  sesión robada no se puede dejar afuera al dueño ni darse permisos.
- **Aviso de ingreso nuevo:** «Nuevo ingreso desde Chrome en Windows, IP …: ¿fuiste vos?», con
  un botón para cerrar esa sesión.
- **Lo que no se evita:** si el virus sigue en la PC, puede usar el navegador ahí mismo o
  capturar la contraseña al escribirla. Ayudan Windows Defender prendido, vendedores sin
  permisos de administrador en la PC y Chrome al día. Una PC infectada se limpia, se cierran
  las sesiones del usuario y se cambia la contraseña.
- **Un solo lugar a la vez.** El usuario tiene una sola sesión en uso (`users.active_session_id`).
  Al entrar desde otra PC, la anterior deja de andar y muestra «Tu usuario se abrió en otra
  PC»; el front se entera en su próximo pedido o en menos de 30 segundos. Si vuelve a la
  primera PC, toca **Usar acá** y la retoma sin escribir la contraseña, porque esa PC sigue
  recordada; la otra queda afuera. Varias pestañas del mismo navegador son la misma sesión.
  Si alguien entra con la contraseña de otro, el dueño lo nota porque lo sacan.
- **Mis dispositivos**, en Mi cuenta: cada PC recordada con navegador, IP y último uso, y un
  botón para cerrarla. Un Administrador puede cerrar todas las de un usuario de su
  organización.
- Cambiar o resetear la contraseña cierra todas las demás sesiones. Desactivar un usuario
  corta todas las suyas.
- **Protección contra CSRF:** las rutas que escriben exigen un header propio del front, que un
  sitio ajeno no puede mandar sin pasar por CORS.
- **Fuerza bruta:** además del límite por IP que ya existe (`RATE_LIMIT_LOGIN`), bloqueo de
  15 minutos después de 5 fallos seguidos del mismo usuario; un Administrador puede
  desbloquearlo antes. Siempre el mismo mensaje («usuario o contraseña incorrectos») y nunca la
  contraseña en el log.
- **Alta:** el admin crea el usuario con una contraseña temporal y el usuario la cambia al
  entrar. No hay mails: si alguien la olvida, un admin se la resetea.
- **El administrador de plataforma se crea con un comando** en la PC del servidor
  (`uv run python -m app.cli crear-admin`), que pide la contraseña ahí mismo y configura el
  código de verificación. Desde la web no se pueden crear administradores de plataforma.
- **Registro de seguridad** (`audit_log`): los logins buenos y fallidos, los bloqueos, los
  cambios y reseteos de contraseña, las altas, bajas y cambios de rol, las sesiones cerradas y
  quién cargó cada cuenta de EPC, SAP o el localizador. Lo ve el administrador de plataforma
  completo y cada Administrador el de su organización. Se guarda `RETENTION_AUDIT_DAYS` (365 por
  defecto).
- `GET /auth/me` devuelve el usuario, su organización, sus sucursales, la activa y sus
  permisos. El front arma el menú con eso, y el backend vuelve a revisar el permiso en cada
  ruta.
- `bind_user` se llama en la dependencia de autenticación y deja al usuario en los logs, en
  la telemetría, en los límites de uso y en las trazas.

## Cuentas de EPC, SAP y localizador por usuario

Hoy hay una sola sesión de cada servicio, con las cuentas del `.env`. Pasa a haber una por
cuenta, y cada usuario carga las suyas en **Mi cuenta**.

- **La contraseña se guarda cifrada**, no con hash, porque hay que usarla para loguearse. La
  clave de cifrado está en el `.env` (`CREDENTIALS_KEY`), nunca en la base: un respaldo de la
  base sin el `.env` no deja leer ninguna. Si se pierde la clave, cada usuario vuelve a cargar
  sus cuentas.
- **Nadie la vuelve a ver:** la pantalla muestra el usuario de la cuenta y si conecta, pero la
  contraseña solo se puede reemplazar. Un Administrador puede cargarla por un usuario de su
  organización, con la misma regla.
- **Dos usuarios con la misma cuenta comparten una sola sesión.** Las sesiones se identifican
  por una huella de servicio, usuario **y contraseña** (un HMAC con la clave de cifrado). Si
  bastara el usuario, cualquiera podría escribir la cuenta de otro con una contraseña
  inventada y usar su sesión. Con una contraseña distinta, intenta su propio login; si está
  mal, falla y se le avisa.
- **EPC:** cada cuenta distinta es un navegador abierto con su licencia. Compartir la sesión
  evita además que dos usuarios con la misma cuenta se saquen la licencia entre ellos. Cada
  navegador ocupa memoria, así que hay un máximo de sesiones de EPC abiertas
  (`EPC_MAX_SESSIONS`). Al llegar al máximo se cierra la que lleva más tiempo sin uso; si
  están todas en uso, se avisa que hay que esperar.
- **SAP:** el navegador se abre solo para loguear. Las cookies de cada cuenta se guardan
  cifradas en la base, en lugar de `SAP_SESSION_FILE`.
- **Localizador:** un cliente HTTP por cuenta, sin navegador.
- Las sesiones se cierran tras `SESSION_KEEPALIVE_HOURS` sin uso. Al arrancar se precalientan
  las cuentas que se usaron en el último día.
- Las píldoras de EPC y SAP de arriba muestran la sesión de la cuenta del usuario que mira.
- Sin una cuenta cargada, esa parte del cotizador avisa «Cargá tu cuenta de EPC en Mi cuenta»
  y el resto sigue andando.
- Los logs, las alertas y Monitoreo identifican la cuenta por su id: nunca el usuario ni la
  contraseña de la cuenta.
- `EPC_USERNAME`, `EPC_PASSWORD`, `SAP_USERNAME`, `SAP_PASSWORD`, `SIMETRICAL_USERNAME`,
  `SIMETRICAL_PASSWORD` y `SAP_SESSION_FILE` salen del `.env`.

## Stock en el cotizador

Cada renglón muestra, sin salir del cotizador:

- **Tu sucursal:** lo que hay en los depósitos de la sucursal activa que suman por defecto.
- **Otras sucursales:** la suma de las demás sucursales de la organización.
- **Planta:** el stock de SAP, como hoy, con la lupa del localizador para la red.

**El código de concesionario es de cada sucursal.** Nissan le da uno a cada sucursal: en general los
primeros cuatro dígitos son del concesionario y los dos últimos de la sucursal (Sendai Lanús es
`231111`, Quilmes `231132`, Lobos `231141`; Antelo, `232111`). A veces varias sucursales comparten
uno (las tres de Femotors usan `230912`). Lo revisamos el 9 de noviembre en el localizador:
cada sucursal de la red viene con su código (`ClientIDAlpha`). Por eso:

- La consulta a SAP manda el código de la sucursal activa.
- El localizador marca como propias las sucursales con un código de la organización.

Al expandirlo se ve cuánto tiene cada sucursal, y en cada una sus depósitos y la ubicación si
el DMS la manda. El vendedor sigue pudiendo sumar o sacar depósitos, como hoy; los de por
defecto salen de `deposits.counts_by_default`.

Cada sucursal tiene su propio servidor con su DMS, y su agente sube el stock de esa sucursal
con su propio token. El token dice de qué sucursal es, así que todo lo que sube queda en esa
sucursal sin que nadie tenga que asignarlo. Un depósito que aparece por primera vez se crea
solo en esa sucursal y suma por defecto. Desde Administración se puede marcar que no sume
(como hoy `GARANTIAS` y `PICKING`) o que se ignore y no se guarde (como `MINOZZI`).

El stock desactualizado (`DMS_STALE_MINUTES`) se mide por sucursal: si se apaga el servidor
de una, solo esa aparece desactualizada, en el cotizador y en Monitoreo.

## Pantallas

- **Login**, con el código de verificación si el usuario lo tiene, y cambio de contraseña
  obligatorio la primera vez.
- **Menú principal:** Cotizador, Presupuestos, Préstamos y Lista de precios, cada uno según los
  permisos. Monitoreo sale de acá.
- **Presupuestos:** los de toda la organización, con la sucursal de cada uno y un filtro por
  sucursal. Uno nuevo queda en la sucursal activa.
- **Lista de precios** de la organización: de cuándo es la lista de Nissan, y el margen y la
  estrategia (con `precios.gestionar`).
- **Selector de sucursal** arriba, para quien trabaja en más de una.
- **Mi cuenta:** cambiar la contraseña, el código de verificación, Mis dispositivos, cargar las
  cuentas de EPC, SAP y el localizador y ver si conectan, y salir.
- **Administración**, según los permisos:
  - Plataforma: organizaciones (con el cupo), sucursales con sus depósitos (si suman, si se
    ignoran) y el agente del DMS de cada una (generar el token, que se muestra una sola vez),
    usuarios de todas, la lista de precios y las compatibilidades de Nissan, y el registro de
    seguridad.
  - Organización: sus usuarios (alta con contraseña temporal, roles, sucursales, desactivar,
    resetear la contraseña, desbloquear, cerrar sus sesiones, cargar sus cuentas) y su registro
    de seguridad.
  - **Monitoreo**, filtrable por organización. Reemplaza el acceso solo desde esta PC
    (`MONITOR_ALLOWED_HOSTS`).

## Lo que deja de estar en el `.env`

| Variable | Pasa a |
|---|---|
| `SAP_DEALER_CODE` | Cada sucursal (`branches.dealer_code`) |
| `LOCALIZADOR_ORIGEN_LAT`, `LOCALIZADOR_ORIGEN_LON` | Cada sucursal |
| `DMS_EXCLUDED_DEPOSITS`, `DMS_IGNORED_DEPOSITS` | Cada depósito |
| `DMS_AGENT_TOKEN` | Un agente por sucursal, con el token guardado como hash |
| `MONITOR_ALLOWED_HOSTS` | El permiso `monitoreo.ver` |
| Cuentas y sesión de EPC, SAP y Simetrical | Las cuentas de cada usuario |

Entran: `CREDENTIALS_KEY`, `EPC_MAX_SESSIONS`, `RETENTION_AUDIT_DAYS` y la duración de las
sesiones de login.

## Aislamiento entre organizaciones

- Cada repositorio recibe la organización (y la sucursal cuando corresponde) del usuario y
  filtra por ella. Pedir algo de otra organización da 404, no 403, para no confirmar que
  existe. Hay tests de eso por cada recurso.
- Antes de sumar una segunda organización se activa Row Level Security como segunda barrera.
  Necesita que el backend se conecte con un usuario de Postgres que no sea el dueño de las
  tablas, que es lo previsto para productivo: las migraciones corren con el dueño y la app con
  un usuario con permisos mínimos.

## Acceso desde internet

Todos los usuarios, de cualquier sucursal y organización, entran a **esta PC por internet**, en
un nombre de `oemia.com.ar` (propuesto: `app.oemia.com.ar`, para dejar `oemia.com.ar` libre
para una web).

- **Cloudflare Tunnel, sin abrir puertos.** El programa `cloudflared` corre como servicio de
  Windows y abre una conexión de salida hacia Cloudflare. Los usuarios entran a Cloudflare, y
  Cloudflare les pasa los pedidos a esta PC por esa conexión. En el router no se abre nada, la
  IP de la oficina no queda a la vista y anda aunque el proveedor cambie la IP. Para eso el DNS
  de `oemia.com.ar` pasa a Cloudflare (cuenta gratis, cambiando los servidores de DNS en NIC
  Argentina).
- **El certificado es uno para el dominio y lo pone Cloudflare**, gratis y renovado solo. De
  Cloudflare a esta PC el túnel ya va cifrado.
- **El backend entrega también el front**, sin Caddy: la carpeta `dist` que arma
  `npm run build` y la API bajo `/api` (por ejemplo `/api/presupuestos`), así no choca con
  las direcciones de las pantallas. El backend escucha solo en `127.0.0.1` y `cloudflared` le
  manda todo. En productivo quedan prendidos tres servicios: Postgres, el backend y
  `cloudflared`. Cloudflare guarda copias de los archivos del front y los comprime.
- **En desarrollo, también una sola dirección:** el servidor de Vite pasa `/api` al backend,
  así la cookie viaja igual que en productivo.
- **Publicar una versión nueva:** `npm run build` y reiniciar el backend.
- **La IP real de cada pedido:** con el túnel todos llegan desde `127.0.0.1`, y la IP de quien
  llama viene en el header `CF-Connecting-IP`. Hay que usarla en los límites de uso, el bloqueo
  del login, los logs y Monitoreo; hoy se toma `request.client.host` (`app/core/access.py`,
  `app/core/rate_limit.py` y el middleware de observabilidad). Se le puede creer al header
  porque el backend solo escucha en `127.0.0.1`.
- **Nada se publica antes del login.** Hoy cualquiera que llegue cotiza con las cuentas de EPC
  y SAP, gasta la IA y ve los costos. Además `MONITOR_ALLOWED_HOSTS` deja ver Monitoreo desde
  `127.0.0.1`, y con el túnel todo llega desde ahí: Monitoreo quedaría abierto a internet. El
  túnel se prende con las etapas 1 a 5 en `main` y Monitoreo detrás de `monitoreo.ver`.
- **En productivo se apagan `/docs`, `/redoc` y `/openapi.json`.**
- **Los agentes del DMS** de cada sucursal suben el stock por internet a la dirección pública,
  con su token.
- **Cotizar responde en streaming** (`text/event-stream`). Hay que comprobar que pase por
  Cloudflare sin demorarse y que ningún pedido quede más de 100 segundos sin responder, que es
  lo que Cloudflare espera antes de cortar (error 524).
- **Solo desde Argentina:** una regla del firewall de Cloudflare (las reglas propias son
  gratis, hasta 5) saca de encima a la mayoría de los bots. Un usuario de viaje no podría
  entrar. Por provincia no se puede en el plan gratis (el campo es de los planes Business y
  Enterprise), y la provincia que se deduce de una IP no es confiable, sobre todo con datos
  del celular.
- **Esta PC pasa a ser el servidor de todos.** Si se apaga, se reinicia por una actualización o
  se corta internet en Antelo, no cotiza nadie de ninguna sucursal. Hace falta que no se
  suspenda, que las actualizaciones reinicien fuera de horario, una UPS y un respaldo de la
  base fuera de esta PC.
- EPC, SAP y el localizador siguen logueando desde esta PC, como hoy.
- `CORS_ORIGINS` pasa a ser la dirección pública.

## Etapas

Las etapas 1 a 5 llegan a `main` juntas: un backend que pide login sin un front que lo tenga
deja la app inutilizable, y sin Administración los usuarios solo se podrían crear desde la
consola. Se trabajan en una rama de integración, con un commit y sus tests por etapa para
revisarlas de a una.

1. **Login en el backend:** usuarios, roles combinables, permisos, sesiones por PC con un solo
   lugar a la vez, la protección contra el robo de sesión (DBSC, la renovación que detecta
   reusos, la contraseña para lo sensible y el aviso de ingreso nuevo), el código de
   verificación, `/auth/login`, `/auth/logout`, `/auth/me`, cambio
   de contraseña, el registro de seguridad, el comando `crear-admin`, la dependencia de
   permisos en cada ruta y `bind_user`.
2. **Organizaciones, sucursales y depósitos:** las tablas, la organización y la sucursal en las
   tablas actuales, los repositorios filtrados, el número de presupuesto por organización, la
   lista de precios compartida con el margen y la estrategia por organización, el agente del
   DMS por sucursal, el cupo y lo que sale del `.env`.
3. **Cuentas por usuario:** `integration_accounts`, el cifrado y las sesiones compartidas de
   EPC, SAP y el localizador.
4. **Front:** la API bajo `/api` con el proxy de Vite, login, menú por permisos, selector de
   sucursal, Mi cuenta con Mis dispositivos, «Usar acá», presupuestos de toda la organización,
   stock por sucursal en el cotizador, costos y lista de precios según permiso, y Monitoreo
   dentro de Administración.
5. **Administración:** organizaciones, sucursales, depósitos, agentes del DMS, usuarios con sus
   roles y el cupo, la carga de la lista de precios y las compatibilidades, y el registro de
   seguridad.
6. **Publicar en internet y cierre de seguridad:** el DNS en Cloudflare, `cloudflared`, el
   backend entregando el front, la IP real desde `CF-Connecting-IP`, apagar `/docs`, la regla
   de solo Argentina, la alerta de muchos logins fallidos, la auditoría (`created_by`, `accepted_by`), Row Level Security antes
   del segundo cliente y el aviso de privacidad a los empleados (Ley 25.326). El túnel se prende
   al final de esta etapa.

## Cómo quedó la etapa 1

- **Rutas:** `/auth/login`, `/auth/salir`, `/auth/me`, `/auth/renovar`, `/auth/usar-aca`,
  `/auth/confirmar-contrasena`, `/auth/contrasena`, `/auth/codigo-verificacion` (y `/activar`),
  `/auth/sesiones` (Mis dispositivos) y `/auth/avisos-vistos`. Chrome llama solo a
  `/auth/dbsc/registro` y `/auth/dbsc/renovar`.
- **Cookies:** `__Host-oemia_sesion` (10 minutos, `Path=/`) y `__Secure-oemia_renovacion`
  (solo para `/auth/renovar`, y se borra cuando la sesión queda atada a la PC).
- **CSRF:** las escrituras exigen el header `X-OEMIA: 1`. No lo piden las de DBSC (las manda
  Chrome por su cuenta) ni `/telemetria` (`sendBeacon` no puede agregar headers).
- **Cada ruta declara su acceso** con `requires(...)`, `signed_in()` o `public()`
  (`app/core/auth.py`), y `tests/integration/test_route_access.py` falla si alguna no lo hace.
  Son públicas solo el login, las dos renovaciones, las de DBSC, `/salud`, la subida del agente
  del DMS y `/telemetria`.
- **Roles del sistema:** los escribe el backend al arrancar desde `app/core/permissions.py`.
- **Costos:** sin `costos.ver`, `/precios/consultar` y `/precios/estado` mandan el costo y los
  márgenes en `null`. Falta lo mismo en los pedidos, que hoy reciben el costo del front: va con
  la etapa 2.
- **Una PC que quedó afuera** recibe 409 `sesion_desplazada` con la otra PC (`otra_sesion`) y
  puede salir, tocar «Usar acá» (sin contraseña, como estaba propuesto) o cerrar la otra («No
  fui yo»). Si la otra PC sale, esta retoma sola.
- **El registro de DBSC se ofrece solo al loguearse** y vale 5 minutos: si se ofreciera en
  cualquier respuesta, alguien con la cookie copiada de un navegador sin DBSC podría atar la
  sesión a su PC.
- **DBSC solo anda con HTTPS.** En desarrollo (`http://localhost`) las sesiones quedan sin atar
  y usan la renovación con el token que cambia. Se prueba de verdad con el túnel, en la etapa 6.
- **`MONITOR_ALLOWED_HOSTS` salió del `.env`**; entraron `CREDENTIALS_KEY` (se genera con
  `uv run python -m app.cli generar-clave`), las duraciones de las sesiones, el bloqueo y
  `RETENTION_AUDIT_DAYS`.
- **Las claves de `Settings` no se imprimen** (`repr=False`): antes, un error que mostraba la
  configuración mostraba también las contraseñas de SAP y Simetrical.

## Decisiones tomadas

1. **Roles combinables:** Cotizador, Encargado y Administrador; los permisos se suman.
2. El número de presupuesto es **uno por organización**, y **se ven los presupuestos de toda la
   organización**, con la sucursal de cada uno.
3. Un usuario puede trabajar en **varias sucursales**, eligiendo la activa arriba.
4. En el cotizador se ve el stock de **la sucursal propia, el resto de las sucursales y la
   planta**, y se puede expandir para ver cuánto tiene cada sucursal.
5. Se entra con **usuario**, no con mail.
6. **Todos entran por internet a esta PC**, con `oemia.com.ar` en Cloudflare y un Cloudflare
   Tunnel. El certificado es uno para el dominio, el de Cloudflare. Solo desde Argentina.
7. **Se cobra por usuario que cotiza.** Los que solo administran no se cobran. Las sucursales
   las crea solo el administrador de plataforma.
8. **Cada usuario carga sus cuentas** de EPC, SAP y el localizador; misma cuenta, misma sesión.
9. Un Administrador puede cargar las cuentas de un usuario de su organización, sin poder verlas.
10. El **código de concesionario de Nissan** va en cada sucursal y puede repetirse.
11. **Código de verificación** obligatorio para el administrador de plataforma y opcional para
    los Administradores.
12. El dueño del sistema tiene **dos usuarios**: el de plataforma y uno común para cotizar.
13. **La lista de precios y las compatibilidades de Nissan las carga el administrador de
    plataforma**, una sola para todos. Cada organización elige su margen y su estrategia.
14. **Registro de seguridad** de logins, contraseñas, roles, sesiones y cuentas.
15. **Cada PC queda recordada** y el usuario se usa **en un solo lugar a la vez**.
16. **Protección contra el robo de sesión:** sesión atada a la PC (DBSC) en Chrome sobre
    Windows; en los demás navegadores, renovación que detecta reusos y sesiones más cortas.
    Lo sensible pide la contraseña de nuevo.
17. Las etapas 1 a 5 llegan juntas a `main`, y recién después se publica en internet.

## Abiertas

1. **El nombre público:** `app.oemia.com.ar` u otro.
2. **Volver a una PC desplazada:** propuesto «Usar acá» sin contraseña; la otra opción es pedir
   la contraseña de nuevo.
3. **El Administrador que no cotiza** ve los presupuestos sin poder editarlos (propuesto).
4. **El usuario de plataforma solo entra con la sesión atada a la PC** (Chrome en Windows), y
   los demás usuarios pueden usar cualquier navegador (propuesto).
