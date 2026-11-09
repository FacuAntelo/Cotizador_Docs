# Usuarios, permisos y organizaciones — plan

Estado: plan acordado con el usuario el 8 de noviembre de 2026. Todavía no hay nada construido.
Lo que queda por resolver está en [Abiertas](#abiertas).

Cómo engancha el login con la observabilidad: [LOGIN_Y_OBSERVABILIDAD.md](LOGIN_Y_OBSERVABILIDAD.md).

## Qué se busca

- Que cada persona entre con su usuario y vea solo lo que su rol le permite.
- Que el sistema sirva para una **organización con varias sucursales**, cada una con sus
  depósitos, y más adelante para varias organizaciones.
- Un **administrador de plataforma** (el dueño del sistema) que da de alta organizaciones,
  sucursales, depósitos y usuarios, y es el único que ve **Monitoreo**.
- Que la **lista de precios** y los **costos** los vea solo quien tenga permiso.
- Cobrar **por sucursal**: cada sucursal trae un usuario incluido y los demás se pagan aparte.
- Que **cada usuario use sus propias cuentas de EPC, SAP y el localizador**, y que dos
  usuarios con la misma cuenta compartan una sola sesión.

## Cómo se organiza

```
Plataforma (administrador de plataforma)
└── Organización   razón social, CUIT, logo, usuarios extra contratados
    └── Sucursal   dirección, teléfono, mail, código de concesionario de Nissan,
                   ubicación para el localizador,
                   precios de taller, su servidor con el DMS y el agente que sube el stock
        └── Depósito   nombre en el DMS, si suma al stock por defecto, si se ignora
Usuario → una organización, un rol, una o más sucursales (una es la de por defecto),
          sus cuentas de EPC, SAP y localizador
```

## Tablas nuevas

Todas en el schema `app`. La base arrancó vacía, así que la organización y la sucursal entran
en las tablas actuales como columnas obligatorias desde el principio, sin pasar datos viejos.

| Tabla | Columnas principales |
|---|---|
| `organizations` | nombre, razón social, CUIT, logo, `extra_users` (usuarios contratados además de los incluidos), activa |
| `branches` | organización, nombre, dirección, teléfono, mail, `dealer_code` (código de concesionario de Nissan, puede repetirse entre sucursales), latitud y longitud, activa |
| `deposits` | sucursal, código en el DMS (único dentro de la sucursal: dos sucursales pueden tener su `PICKING`), nombre, `counts_by_default`, `ignored` |
| `users` | organización (vacía solo para el administrador de plataforma), usuario (único en todo el sistema), nombre a mostrar, hash de la contraseña, `must_change_password`, activo, intentos fallidos, bloqueado hasta, último ingreso, creado por |
| `roles` | organización (vacía para los roles del sistema), nombre, si es del sistema |
| `role_permissions` | rol, permiso. Los permisos válidos los define el código (un `CHECK` con la misma lista, como los estados) |
| `user_branches` | usuario, sucursal, si es la de por defecto |
| `sessions` | usuario, hash del token, creada, último uso, vence, sucursal activa, IP, navegador, revocada |
| `integration_accounts` | usuario, servicio (`epc`, `sap`, `localizador`), usuario de la cuenta, contraseña cifrada, huella, estado y último error |
| `dms_agents` | sucursal, nombre, hash del token, última subida |

Un usuario tiene **un solo rol** (`users.role_id`). Si algún día hace falta combinar roles, se
pasa a una tabla `user_roles`.

## De quién es cada dato

| Hoy | Pasa a ser de | Cambio |
|---|---|---|
| `estimates` y lo que cuelga (renglones, pedidos, insumos, orden del seguro) | Sucursal | El número es único **por organización** (`uq_estimates_org_number`) y el bloqueo que lo asigna, una clave por organización. El presupuesto dice de qué sucursal es |
| `loans` | Sucursal | — |
| `dms_stock` | Depósito | La clave pasa a ser (depósito, código) |
| `dms_stock_uploads` | Agente del DMS | — |
| `price_items`, `price_conflicts`, `price_variants`, `price_list` | Organización | `price_list` deja de ser una sola fila |
| `part_compatibilities`, `compatibility_list` | Organización | Ídem |
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
roles son grupos de permisos guardados en la base: arrancan los cuatro del sistema y más
adelante cada organización podría armar los suyos.

| Permiso | Qué habilita |
|---|---|
| `cotizar` | El cotizador: match, EPC, SAP, localizador y asesor |
| `presupuestos` | Ver, crear, editar y aceptar presupuestos |
| `pedidos` | Confirmar el Excel y registrar llegadas y separaciones |
| `prestamos` | Ver y cargar préstamos |
| `costos.ver` | Ver costo y márgenes en el cotizador, los presupuestos y los pedidos |
| `precios.gestionar` | La pantalla de lista de precios: subir la lista y las compatibilidades, cambiar margen y estrategia |
| `aseguradoras.gestionar` | Alta, baja y modificación de compañías de seguro |
| `sucursal.configurar` | Datos de la sucursal y precios de taller |
| `usuarios.gestionar` | Usuarios de su organización, dentro del cupo |
| `plataforma.administrar` | Organizaciones, sucursales, depósitos, agentes del DMS y usuarios de todas las organizaciones |
| `monitoreo.ver` | Monitoreo y la salud detallada |

| Permiso | Admin de plataforma | Admin de la organización | Encargado | Vendedor |
|---|:-:|:-:|:-:|:-:|
| `cotizar`, `presupuestos`, `pedidos`, `prestamos` | — | ✓ | ✓ | ✓ |
| `costos.ver` | — | ✓ | ✓ | — |
| `precios.gestionar` | — | ✓ | ✓ | — |
| `aseguradoras.gestionar`, `sucursal.configurar` | — | ✓ | ✓ | — |
| `usuarios.gestionar` | — | ✓ | — | — |
| `plataforma.administrar`, `monitoreo.ver` | ✓ | — | — | — |

El administrador de plataforma no pertenece a ninguna organización. Para usar el cotizador
necesita un usuario común dentro de una.

**Ocultar la pantalla no alcanza.** Hoy el cotizador recibe el costo y los márgenes de cada
repuesto (`/precios/consultar`) y los pedidos muestran el costo. Sin `costos.ver`, el backend
no manda esos campos; sin `precios.gestionar`, rechaza las rutas de la lista con 403. El precio
de venta lo ve todo el que cotiza.

**Cada ruta declara su permiso**, como hoy declara su grupo de límite de uso, y un test falla
si alguna ruta no lo dice. Las únicas sin permiso son `/auth/login`, `/salud` y la subida del
agente del DMS, que usa su propio token.

## Cupos: sucursales y usuarios

- **Los usuarios son de la organización**, no de una sucursal. Cada uno tiene habilitadas una
  o más sucursales y elige en cuál está trabajando.
- Se paga por dar de alta una sucursal y por dar de alta usuarios. El cupo de la organización
  es `sucursales activas × USERS_INCLUDED_PER_BRANCH + extra_users`, con un usuario incluido
  por sucursal.
- Cuentan los usuarios activos. El administrador de plataforma no cuenta.
- Crear o reactivar un usuario por encima del cupo se rechaza con un mensaje que dice el
  cupo y cómo ampliarlo. Desactivar una sucursal que dejaría más usuarios que cupo también se
  rechaza: primero hay que desactivar usuarios.
- Solo el administrador de plataforma crea sucursales y cambia `extra_users`. Lo que se cobra
  queda afuera del sistema: el sistema cuenta y pone el límite.
- Administración muestra por organización: sucursales, usuarios usados y cupo (ej. «3 de 4»).

## Login y sesiones

- **Usuario y contraseña.** El usuario es único en todo el sistema, así que al entrar no hay
  que elegir organización.
- **Contraseñas con Argon2id.** Mínimo 12 caracteres, sin reglas de composición.
- **Sesión en una cookie `HttpOnly`, `Secure` y `SameSite=Lax`**, con un token opaco cuyo hash
  se guarda en `sessions`. Salir revoca la sesión y desactivar un usuario corta todas las suyas.
  Nada de tokens en `localStorage`, que se pueden robar con un script.
- **Protección contra CSRF:** las rutas que escriben exigen un header propio del front, que un
  sitio ajeno no puede mandar sin pasar por CORS.
- **Vence a las 12 horas sin uso** y dura como máximo 7 días.
- **Fuerza bruta:** además del límite por IP que ya existe (`RATE_LIMIT_LOGIN`), bloqueo de
  15 minutos después de 5 fallos seguidos del mismo usuario. Siempre el mismo mensaje
  («usuario o contraseña incorrectos») y nunca la contraseña en el log.
- **Alta:** el admin crea el usuario con una contraseña temporal y el usuario la cambia al
  entrar. No hay mails: si alguien la olvida, un admin se la resetea.
- **El administrador de plataforma se crea con un comando** en la PC del servidor
  (`uv run python -m app.cli crear-admin`), que pide la contraseña ahí mismo. Desde la web no
  se pueden crear administradores de plataforma.
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
  contraseña solo se puede reemplazar. Un admin de la organización puede cargarla por un
  vendedor, con la misma regla.
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

- **Login** y cambio de contraseña obligatorio la primera vez.
- **Menú principal:** Cotizador, Presupuestos, Préstamos y Lista de precios (esta solo con
  `precios.gestionar`). Monitoreo sale de acá.
- **Selector de sucursal** arriba, para quien trabaja en más de una.
- **Mi cuenta:** cambiar la contraseña, cargar las cuentas de EPC, SAP y el localizador y ver
  si conectan, y salir.
- **Administración**, según los permisos:
  - Plataforma: organizaciones (con el cupo), sucursales con sus depósitos (si suman, si se
    ignoran) y el agente del DMS de cada una (generar el token, que se muestra una sola vez),
    y usuarios de todas.
  - Organización: sus usuarios (alta con contraseña temporal, rol, sucursales, desactivar,
    resetear la contraseña, cargar sus cuentas).
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

Entran: `CREDENTIALS_KEY`, `USERS_INCLUDED_PER_BRANCH`, `EPC_MAX_SESSIONS` y la duración de
las sesiones de login.

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
- **Nada escucha en la red:** Caddy en `127.0.0.1` sirve el front compilado y pasa `/api` al
  backend, que también escucha en `127.0.0.1`. `cloudflared` le manda todo a Caddy.
- **La IP real de cada pedido:** con el túnel todos llegan desde `127.0.0.1`, y la IP de quien
  llama viene en el header `CF-Connecting-IP`. Hay que usarla en los límites de uso, el bloqueo
  del login, los logs y Monitoreo; hoy se toma `request.client.host` (`app/core/access.py`,
  `app/core/rate_limit.py` y el middleware de observabilidad). Se le puede creer al header
  porque el backend solo escucha en `127.0.0.1`.
- **Nada se publica antes del login.** Hoy cualquiera que llegue cotiza con las cuentas de EPC
  y SAP, gasta la IA y ve los costos. Además `MONITOR_ALLOWED_HOSTS` deja ver Monitoreo desde
  `127.0.0.1`, y con el túnel todo llega desde ahí: Monitoreo quedaría abierto a internet. El
  túnel se prende con las etapas 1 a 4 en `main` y Monitoreo detrás de `monitoreo.ver`.
- **En productivo se apagan `/docs`, `/redoc` y `/openapi.json`.**
- **Los agentes del DMS** de cada sucursal suben el stock por internet a la dirección pública,
  con su token.
- **Cotizar responde en streaming** (`text/event-stream`). Hay que comprobar que pase por
  Cloudflare sin demorarse y que ningún pedido quede más de 100 segundos sin responder, que es
  lo que Cloudflare espera antes de cortar (error 524).
- **Opcional:** una regla del firewall de Cloudflare que deje entrar solo desde Argentina saca
  de encima a la mayoría de los bots. Un usuario de viaje no podría entrar.
- **Esta PC pasa a ser el servidor de todos.** Si se apaga, se reinicia por una actualización o
  se corta internet en Antelo, no cotiza nadie de ninguna sucursal. Hace falta que no se
  suspenda, que las actualizaciones reinicien fuera de horario, una UPS y un respaldo de la
  base fuera de esta PC.
- EPC, SAP y el localizador siguen logueando desde esta PC, como hoy.
- `CORS_ORIGINS` pasa a ser la dirección pública.

## Etapas

Las etapas 1 a 4 llegan a `main` juntas: un backend que pide login sin un front que lo tenga
deja la app inutilizable. Se trabajan en una rama de integración, con un commit por etapa para
revisarlas de a una.

1. **Login en el backend:** usuarios, roles, permisos, sesiones, `/auth/login`, `/auth/logout`,
   `/auth/me`, cambio de contraseña, el comando `crear-admin`, la dependencia de permisos en
   cada ruta y `bind_user`.
2. **Organizaciones, sucursales y depósitos:** las tablas, la organización y la sucursal en las
   tablas actuales, los repositorios filtrados, el número de presupuesto por organización, el
   agente del DMS por sucursal, los cupos y lo que sale del `.env`.
3. **Cuentas por usuario:** `integration_accounts`, el cifrado y las sesiones compartidas de
   EPC, SAP y el localizador.
4. **Front:** login, menú por permisos, selector de sucursal, Mi cuenta, stock por sucursal en
   el cotizador, costos y lista de precios según permiso, y Monitoreo dentro de Administración.
5. **Administración:** organizaciones, sucursales, depósitos, agentes del DMS y usuarios, con
   los cupos.
6. **Publicar en internet y cierre de seguridad:** el DNS en Cloudflare, `cloudflared` y Caddy,
   la IP real desde `CF-Connecting-IP`, apagar `/docs`, la alerta de muchos logins fallidos, la
   auditoría (`created_by`, `accepted_by`), Row Level Security antes del segundo cliente y el
   aviso de privacidad a los empleados (Ley 25.326). El túnel se prende al final de esta etapa.

## Decisiones tomadas

1. Existe el rol **administrador de la organización**, aunque al principio los usuarios los
   cree el administrador de plataforma.
2. El número de presupuesto es **uno por organización**; el presupuesto dice de qué sucursal es.
3. Un usuario puede trabajar en **varias sucursales**, eligiendo la activa arriba.
4. En el cotizador se ve el stock de **la sucursal propia, el resto de las sucursales y la
   planta**, y se puede expandir para ver cuánto tiene cada sucursal.
5. Se entra con **usuario**, no con mail.
6. **Todos entran por internet a esta PC**, con `oemia.com.ar` en Cloudflare y un Cloudflare
   Tunnel. El certificado es uno para el dominio, el de Cloudflare.
7. Se paga por dar de alta **sucursales** y **usuarios**. Los usuarios son de la organización
   y eligen en qué sucursal trabajan, entre las que tienen habilitadas.
8. **Cada usuario carga sus cuentas** de EPC, SAP y el localizador; misma cuenta, misma sesión.
9. El admin de la organización puede cargar las cuentas de un vendedor, sin poder verlas.
10. **Cada sucursal incluye un usuario**; los demás se pagan aparte.
11. El **código de concesionario de Nissan** va en cada sucursal y puede repetirse.

## Abiertas

1. **El nombre público:** `app.oemia.com.ar` u otro.
