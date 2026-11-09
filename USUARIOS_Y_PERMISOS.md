# Usuarios, permisos y organizaciones — plan

Estado: plan acordado con el usuario el 8 de noviembre de 2026 y repasado el 9. Las etapas 1 a 5
se construyeron en la rama `feat/usuarios` de los tres repositorios y se mergearon a `main` el 9
de noviembre (ver [Cómo quedó la etapa 1](#cómo-quedó-la-etapa-1), [la 2](#cómo-quedó-la-etapa-2),
[la 3](#cómo-quedó-la-etapa-3), [la 4](#cómo-quedó-la-etapa-4) y
[la 5](#cómo-quedó-la-etapa-5)). La 6, publicar en internet y reforzar Postgres, está en curso
en la rama `feat/publicar` (ver [Cómo va la etapa 6](#cómo-va-la-etapa-6)). Lo que queda por
resolver está en [Abiertas](#abiertas) y lo que se dejó para después, en
[Pendientes](#pendientes). La configuración de productivo está en
[PRODUCCION.md](PRODUCCION.md).

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
  pide al loguearse: obligatorio para el administrador de plataforma (sin código configurado
  no entra), opcional para los demás. Si alguien pierde el celular, un Administrador de su
  organización le quita el código desde Administración (y se le cierran las sesiones); al de
  plataforma se le configura otro desde la consola de la PC del servidor
  (`python -m app.cli configurar-codigo`).
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
  primera PC, **tiene que ingresar de nuevo**, con la contraseña y el código si lo tiene; la
  otra queda afuera. Es a propósito: un usuario no se comparte entre varias PCs con solo tocar
  un botón, porque el concesionario paga un usuario por cada persona que cotiza (decisión 21;
  hasta la etapa 6 bastaba con «Usar acá»). Varias pestañas del mismo navegador son la misma
  sesión.
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
  código de verificación. Desde la web no se pueden crear administradores de plataforma, ni
  quitarles el código: si cambia de celular, `uv run python -m app.cli configurar-codigo`.
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
- Postgres repite el filtro como segunda barrera con Row Level Security, en la etapa 6 (ver
  [Seguridad en Postgres](#seguridad-en-postgres)).

## Seguridad en Postgres

Lo que se revisó el 9 de noviembre, sin cambiar nada:

- **Escucha en todas las direcciones de red** (`listen_addresses = '*'`, el puerto 5432 en
  `0.0.0.0` y `::`). Lo que hoy frena las conexiones desde la red es solo el firewall de Windows,
  que no tiene ninguna regla que abra el puerto. `pg_hba.conf` no se pudo leer sin el usuario
  `postgres`.
- **La app se conecta con `cotizador_dev`**, que es el dueño de todas las tablas, puede crear
  bases y tiene la contraseña igual al nombre. Con él se puede borrar cualquier tabla o el
  registro de seguridad.
- Sin límite de tiempo por consulta y sin Row Level Security. Las contraseñas de Postgres van con
  `scram-sha-256`, y sin SSL alcanza porque la app y la base están en la misma PC.

Lo que se hace en la etapa 6, antes de publicar:

1. **Solo conexiones desde esta PC:** `listen_addresses = 'localhost'` y `pg_hba.conf` solo con
   `127.0.0.1/32` y `::1/128` con `scram-sha-256`. Así no depende de que el firewall siga bien.
   Lo cambia el usuario, porque pide permisos de administrador y reiniciar el servicio.
2. **Un usuario de Postgres para cada cosa:**
   - **Dueño** (`cotizador_owner`): dueño de las tablas. Corre las migraciones, crea las
     particiones mensuales de la observabilidad y hace la limpieza por antigüedad. La app lo
     usa solo para eso, con su propia URL (`MIGRATIONS_DATABASE_URL`), no para atender pedidos.
   - **App** (`cotizador_app`): solo lee, agrega, cambia y borra filas de `app` y
     `observability`; no crea ni borra tablas, no crea bases y no es superusuario. Contraseña
     larga al azar en el `.env`.
   - **Sistema** (`cotizador_sistema`, con `BYPASSRLS`): para lo que no es de una sola
     organización (ver abajo). Los mismos permisos que la app.
   - **El registro de seguridad no se puede tocar desde la app:** en `audit_log` la app y el
     sistema solo agregan y leen filas. Ni una sesión robada ni un error del código lo borran.
   - **Límites para la app:** `statement_timeout`, `idle_in_transaction_session_timeout` y un
     máximo de conexiones.
3. **Row Level Security (RLS):** ver abajo.
4. **Disco y respaldos:** Postgres no cifra los datos en el disco. Windows 11 Home no trae
   BitLocker; si esta PC tiene «Cifrado del dispositivo», se prende. Respaldo diario con
   `pg_dump`, cifrado y guardado fuera de esta PC. Lo más sensible ya no depende del disco: las
   contraseñas de los usuarios y los tokens van con hash, y las cuentas de EPC, SAP y el
   localizador y el código de verificación, cifrados con `CREDENTIALS_KEY`, que está en el
   `.env` y no en la base.
5. **Detalles:** registrar en el log de Postgres los intentos de conexión (`log_connections`),
   una contraseña fuerte para `postgres` guardada fuera de la PC, y Postgres al día.

### Row Level Security

Es de la base, no de la API. La API ya filtra (primera barrera): cada repositorio recibe la
organización del usuario y lo de otra da 404. Con RLS, Postgres repite el filtro por su cuenta
en cada consulta, aunque el código se olvide del `WHERE`. Si la app pide
`SELECT * FROM estimates WHERE id = 57` y el 57 es de otra organización, Postgres agrega solo
`AND organization_id = <la del pedido>` y no devuelve nada.

- **Cómo sabe Postgres de qué organización es el pedido:** al empezar cada transacción, el
  backend fija `app.organization_id` (solo para esa transacción) con la organización del
  usuario. Lo hace un solo lugar (un evento de la sesión de SQLAlchemy que lee la
  organización del pedido desde una `ContextVar`), así los repositorios no cambian. Sin ese
  valor, las tablas del negocio no devuelven nada: si algo se olvida, falla cerrado.
- **Qué tablas:**
  - Por su organización: `organizations` (la propia), `branches`, `users`, `roles` (los del
    sistema, sin organización, se ven siempre), `estimates`, `insurers` y `audit_log`.
  - Por su sucursal: `deposits`, `dms_agents`, `dms_stock_uploads`, `loans`,
    `recent_vehicles` y `workshop_prices`.
  - Por su presupuesto, su depósito o su usuario: las líneas, pedidos, insumos y órdenes de
    los presupuestos, `dms_stock`, `sessions`, `user_roles`, `user_branches` e
    `integration_accounts`.
  - Sin RLS, porque son de todos: la lista de precios, las compatibilidades, los candidatos
    del catálogo, las sesiones guardadas de EPC y SAP (se comparten por huella) y la
    observabilidad, con `llm_calls` y `quote_runs` (la escribe el escritor de métricas, que no
    es de ningún pedido, y la API ya filtra Monitoreo por organización).
- **Lo que no es de una sola organización** usa la conexión de sistema, que se saltea RLS:
  el login y la lectura de la sesión de cada pedido (antes de saber quién es), la subida del
  agente del DMS (antes de saber de qué sucursal es), las alertas, el precalentado de las
  cuentas, lo que va por huella de una cuenta (cómo le fue y el último uso), la consola y la
  plataforma. Esa conexión se usa solo desde esos módulos, y un test lo controla. La limpieza
  de lo vencido y las particiones van con el dueño de las tablas.
- **Tests:** por cada tabla, una consulta sin filtro con la organización 1 no ve filas de la 2,
  y sin organización no ve ninguna.
- **El registro de seguridad de una organización** muestra al administrador de plataforma
  como «Administración de OEMIA»: RLS le oculta los usuarios de plataforma.
- **Contra qué protege y contra qué no:** protege de los errores del propio código (un filtro
  olvidado, una ruta nueva sin organización). No frena a quien logre ejecutar SQL arbitrario
  con el usuario de la app, porque podría cambiar `app.organization_id`; contra eso están las
  consultas con parámetros (SQLAlchemy), el usuario con permisos mínimos y Postgres solo local.
- **Costo:** chico. Las columnas `organization_id` y `branch_id` ya tienen índice, y las
  políticas por sucursal o por presupuesto hacen una subconsulta sobre tablas chicas.

## Acceso desde internet

Todos los usuarios, de cualquier sucursal y organización, entran a **esta PC por internet**, en
`app.oemia.com.ar` (confirmado el 9 de noviembre de 2026), así `oemia.com.ar` queda libre para
una web.

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
  llama viene en el header `CF-Connecting-IP`. La usan los límites de uso, el bloqueo del
  login, los logs y Monitoreo, desde un solo lugar (`app/core/client_ip.py`), y solo si el
  pedido viene de esta PC. `X-Forwarded-For` no se usa: lo puede inventar el usuario. Por eso
  el backend se levanta con `python -m app.server`, que apaga los «proxy headers» de uvicorn
  (hecho en la etapa 6).
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
6. **Publicar en internet y cierre de seguridad**, en bloques (ver
   [Cómo va la etapa 6](#cómo-va-la-etapa-6)):
   1. Backend listo para internet: entorno de productivo, el backend entregando el front,
      headers de seguridad, la IP real desde `CF-Connecting-IP`, apagar `/docs`, el «sigo» de
      Cotizar para Cloudflare, la alerta de muchos ingresos fallidos, los scripts de
      desarrollo y productivo, y [PRODUCCION.md](PRODUCCION.md) con toda la configuración.
   2. Código de verificación y sesiones: exigir al entrar el código de la plataforma, que un
      Administrador pueda quitarle el código a quien perdió el celular, y que volver a una PC
      desplazada pida ingresar de nuevo (decisión 21).
   3. La [seguridad en Postgres](#seguridad-en-postgres): un usuario para cada cosa, el
      registro que no se puede borrar y Row Level Security.
   4. Lo que hace el usuario: Cloudflare y el DNS, `cloudflared`, Postgres solo local, los
      usuarios y la base de productivo (`scripts/preparar_postgres.py productivo`), el backend
      al prender la PC, la PC como servidor y cargar la base.
   5. Pruebas por internet (DBSC, Cotizar sin error 524, el agente del DMS) y prender el túnel.

   El aviso de privacidad a los empleados salió de esta etapa (ver [Pendientes](#pendientes)).

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
  márgenes en `null` (los pedidos, desde la etapa 2).
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

## Cómo quedó la etapa 2

- **Migración 0003:** la organización y la sucursal entran obligatorias en las tablas del
  negocio, sin pasar datos (la base estaba vacía). `dealer_profile` y `dms_deposits` se borran.
- **Cada ruta recibe de quién son los datos** (`OrgScope` o `BranchScope`, en
  `app/core/auth.py`). Los repositorios filtran por la organización (o la sucursal); lo de otra
  organización da 404. Un usuario sin organización recibe 403 `sin_organizacion` y uno sin
  sucursal, 409 `sin_sucursal`.
- **Presupuestos:** número por organización (`uq_estimates_org_number`) con un bloqueo por
  organización; uno nuevo queda en la sucursal activa, con `created_by`; aceptarlo guarda
  `accepted_by`. La lista es de toda la organización, con la sucursal de cada uno y el filtro
  `sucursal`.
- **Datos del concesionario:** `GET /presupuestos/concesionario` arma la razón social, el CUIT y
  el logo de la organización con la dirección, el teléfono, el mail y la hora de mano de obra de
  la sucursal. El `PUT` cambia solo lo de la sucursal: lo de la organización se cambia en
  Administración.
- **Lista de precios:** una sola, la carga la plataforma (`POST /precios/cargar`, pide la
  contraseña de nuevo; `GET /precios/lista` para verla). El margen y la estrategia son de cada
  organización (`PATCH /precios/margen` y `/estrategia`, con `precios.gestionar`).
- **Pedidos:** sin `costos.ver`, la vista previa y el seguimiento no muestran el costo, y al
  confirmar el Excel el backend toma el costo de la lista de precios (no le cree al front).
  Los Excel guardan `created_by`.
- **Stock del taller:** un agente por sucursal con su token (`dms_agents`, se genera con
  `uv run python -m app.cli crear-agente-dms` hasta que esté Administración), depósitos por
  sucursal que se crean solos, y la consulta con la sucursal activa, las demás y el detalle de
  cada una. La frescura, la salud, Monitoreo y la alerta son por sucursal.
- **SAP** consulta con el código de la sucursal activa; **el localizador** mide desde su
  ubicación y marca como propias las sucursales de la organización.
- **El cupo:** `BranchRepository.seats_in_use` cuenta los usuarios activos con el rol Cotizador;
  el alta de usuarios lo va a respetar (etapa 5).
- **Observabilidad:** `request_log`, `dependency_events`, `frontend_events`, `llm_calls` y
  `quote_runs` guardan la organización (y los costos de la IA, la sucursal).
- **Salieron del `.env`:** `SAP_DEALER_CODE`, `LOCALIZADOR_ORIGEN_LAT`, `LOCALIZADOR_ORIGEN_LON`,
  `DMS_EXCLUDED_DEPOSITS`, `DMS_IGNORED_DEPOSITS` y `DMS_AGENT_TOKEN`.

## Cómo quedó la etapa 3

- **Migración 0004:** `integration_accounts` (una cuenta por usuario y servicio, con la
  contraseña cifrada atada a su fila, la huella, el estado y el último error) e
  `integration_sessions` (las cookies de SAP, cifradas, por huella).
- **Mi cuenta:** `GET /cuentas`, `PUT /cuentas/{servicio}` y `DELETE /cuentas/{servicio}`
  (`epc`, `sap`, `localizador`). Cargar y borrar piden la contraseña de nuevo y quedan en el
  registro de seguridad. La contraseña de la cuenta nunca vuelve al front.
- **Sesiones compartidas:** `app/core/integrations.py` arma un grupo de sesiones por servicio
  (`SessionPool`), una por huella. El EPC tiene un máximo (`EPC_MAX_SESSIONS`): al llegar se
  cierra la de más tiempo sin uso, y si todas tienen consultas en curso se avisa que hay que
  esperar (503 `sesiones_ocupadas`).
- **El cotizador y el asesor** usan el EPC sin saber de usuarios: cada llamada toma la cuenta
  del usuario del pedido. Sin cuenta, el EPC responde 409 `sin_cuenta` («Cargá tu cuenta de EPC
  en Mi cuenta») y el resto sigue andando. La vista previa del pedido sin cuenta de SAP avisa y
  no elige piezas solas.
- **Las píldoras** (`/epc/session`, `/sap/session`) muestran la sesión de la cuenta del usuario
  que mira, sin abrirla; `cuenta: false` en el EPC si no la cargó.
- **El estado de cada cuenta** (`conectada`, `rechazada`, `error`) se actualiza al usarla y se
  ve en Mi cuenta. Al arrancar se precalientan las cuentas usadas el último día.
- **Salieron del `.env`** `EPC_USERNAME`, `EPC_PASSWORD`, `SAP_USERNAME`, `SAP_PASSWORD`,
  `SAP_SESSION_FILE`, `SIMETRICAL_USERNAME` y `SIMETRICAL_PASSWORD`. En el `.env` de esta PC
  quedaron comentadas, sin borrar, hasta cargarlas en Mi cuenta. Entró `EPC_MAX_SESSIONS`.

## Cómo quedó la etapa 4

- **La API va bajo `/api`** (`app/core/paths.py`). Los límites de uso, los registros y la
  configuración siguen nombrando las rutas sin el prefijo. En desarrollo el servidor de Vite
  pasa `/api` al backend de desarrollo, hoy en `127.0.0.1:8001` (`vite.config.ts`, y `VITE_API_URL=/api` en
  `.env.development`): el front y la API quedan en el mismo nombre, como en productivo.
- **`npm run gen:api`** (`scripts/gen-api.mjs`) genera los tipos sin el prefijo, y el documento
  de `/telemetria` ya no rompe la generación.
- **El front arranca por `AuthGate`:** sin sesión, el ingreso (con el código de verificación si
  el usuario lo tiene); si otra PC tomó el usuario, «Tu usuario se abrió en otra PC» con «Usar
  acá», «No fui yo: cerrar esa sesión» y Salir; con la contraseña temporal, el cambio
  obligatorio. `/auth/me` se vuelve a leer cada 30 segundos.
- **El cliente** manda `X-OEMIA: 1`, renueva la cookie corta una sola vez cuando vence (aunque
  fallen varios pedidos a la vez) y repite el pedido.
- **El menú** muestra lo que permiten los roles, y cada pantalla pide su permiso. Arriba van el
  selector de sucursal (`PUT /auth/sucursal`), Mi cuenta y el aviso «Nuevo ingreso desde…
  ¿fuiste vos?».
- **Mi cuenta:** cambio de contraseña, código de verificación (la clave para escribirla en la app
  y el enlace `otpauth://`), Mis dispositivos con «Protegida» o «Sin protección» y las cuentas
  de EPC, SAP y el localizador. Lo sensible abre el diálogo para confirmar la contraseña.
- **Presupuestos:** la sucursal de cada uno y el filtro por sucursal, para quien trabaja en
  varias. **Cotizador:** el stock del taller suma «otras N» y el detalle de cada sucursal.
  **Lista de precios:** cargar la lista y las compatibilidades es de la plataforma; el margen y
  el criterio se editan con `precios.gestionar`. Las píldoras de EPC y SAP dicen «sin cuenta»
  si falta cargarla.
- **Para probar en esta PC sin tocar `cotizador_dev`:** `scripts/sembrar_desarrollo.py` arma
  `cotizador_usuarios_dev` con una organización, dos sucursales y el usuario `prueba.vendedor`.

## Cómo quedó la etapa 5

- **Rutas** bajo `/administracion` (`app/routers/admin.py`): `roles`, `organizaciones` (con sus
  `sucursales`), `sucursales/{id}` (con sus `depositos` y `agentes`), `usuarios` (con `roles`,
  `sucursales`, `desactivar`, `reactivar`, `contrasena`, `desbloquear`, `cerrar-sesiones` y
  `cuentas`) y `registro`. Un Administrador (`usuarios.gestionar`) ve y cambia solo lo de su
  organización: lo de otra da 404, también si la pide por `?organizacion=`. La plataforma ve
  todas y es la única que toca organizaciones, sucursales, depósitos y agentes.
- **Las reglas** están en `app/services/admin.py` y se revisan en una transacción con la fila
  de la organización bloqueada (`AdminRepository.organization_transaction`): dos altas a la
  vez no se pasan del cupo ni dejan a la organización sin Administrador.
  - **Cupo:** crear o reactivar un Cotizador, o darle el rol, por encima de
    `cotizador_seats` da 409 `sin_cupo` con el cupo y cómo ampliarlo. El cupo no se puede
    bajar por debajo de los que ya cotizan.
  - **Último Administrador:** no se le saca el rol ni se lo desactiva (409
    `ultimo_administrador`).
  - **El propio usuario:** nadie cambia sus roles, ni se desactiva ni se resetea la
    contraseña desde acá (409 `propio_usuario`); cerrar sus sesiones deja la actual.
  - Solo se asignan Cotizador, Encargado y Administrador, y los usuarios de la plataforma
    no aparecen. Un Cotizador necesita al menos una sucursal. Una sucursal desactivada se
    puede conservar, pero no sumar.
- **Piden la contraseña de nuevo:** crear usuarios, cambiar roles, desactivar, reactivar,
  resetear la contraseña, cerrar sesiones, cargar o borrar cuentas y todo lo de la
  plataforma. No la piden el nombre, las sucursales ni desbloquear.
- **Desactivar** corta todas las sesiones al instante (`usuario_desactivado`); **resetear** pone
  una contraseña temporal, que el usuario cambia al entrar, y también las cierra.
- **Registro de seguridad:** cada acción queda con quién la hizo, a quién y desde dónde. La
  migración 0005 suma lo de la plataforma: organizaciones y sucursales creadas o modificadas,
  depósitos y tokens de los agentes del DMS. Un test falla si el código escribe una acción
  que la base no acepta.
- **Monitoreo** se filtra por organización (`?organizacion=` en rendimiento, costos, calidad
  de la IA, integraciones, negocio y navegador); los recursos del servidor y las alertas son
  de todos.
- **El front** suma Administración con una pestaña por permiso: Usuarios, Organizaciones,
  Lista de precios, Registro de seguridad y Monitoreo (`/monitoreo` lleva ahí). El alta
  propone una contraseña temporal de cuatro palabras fácil de dictar; el token de un agente se
  muestra una sola vez, con un botón para copiarlo.
- **La lista de precios de la plataforma** lee `/precios/lista` (antes pedía `/precios/estado`,
  que exige organización) y cargarla abre el diálogo de la contraseña.
- **Mi cuenta** muestra las cuentas de EPC, SAP y el localizador solo a quien cotiza.
- **Para probar en esta PC:** `scripts/sembrar_desarrollo.py` suma `prueba.plataforma`, un
  administrador de plataforma que existe solo en la base de desarrollo (desde la etapa 6 entra
  con un código de prueba fijo, `DEV_TOTP_SECRET`).

## Cómo va la etapa 6

En la rama `feat/publicar` de los tres repositorios.

**Bloque 1, backend listo para internet (hecho):**

- **`ENVIRONMENT=produccion`** (acepta el nombre viejo, `OTEL_ENVIRONMENT`): sin `/docs`,
  `/redoc` ni `/openapi.json`; solo se atienden pedidos que vienen de esta PC (por
  `cloudflared`), los demás reciben 403; manda HSTS; y el backend no arranca si la
  configuración es insegura (`app/core/production.py`: `CORS_ORIGINS` con `*` o sin https,
  `SERVER_HOST` fuera de esta PC, sin el front compilado, sin `CREDENTIALS_KEY` o con los
  límites apagados).
- **`python -m app.server`** levanta el backend con lo importante fijo: `SERVER_HOST`
  (127.0.0.1), sin los «proxy headers» de uvicorn, sin el header `server` y hasta 30 segundos
  de espera al apagar. `--recargar` solo en desarrollo.
- **El backend entrega el front** desde `FRONTEND_DIST` (`app/core/frontend.py`): `assets/`
  con caché de un año, `index.html` sin caché, las pantallas devuelven `index.html` y una ruta
  de la API que no existe sigue siendo un 404 en JSON. Es un `Mount`: no pasa por la sesión
  ni por los límites. Esos archivos no se guardan en `request_log` (solo los 500).
- **Headers de seguridad** en todas las respuestas (`app/core/web_security.py`) y
  **Content-Security-Policy** en las páginas, con el hash del script del tema calculado del
  `index.html` compilado. `base-uri 'self'` y no `'none'`: `html-to-image` usa un `<base>`
  propio para leer las fuentes del PDF. Probado en el navegador con el front compilado: el
  tema oscuro y el PDF del presupuesto andan sin violaciones.
- **La IP real** desde `CF-Connecting-IP` (`app/core/client_ip.py`), también en el
  middleware de observabilidad, que antes tomaba la de la conexión.
- **El «sigo» de Cotizar** (`app/core/sse.py`, `SSE_KEEPALIVE_SECONDS`): el generador corre en
  una sola tarea aparte y una cola le pasa los eventos; si pasan 20 segundos sin eventos se
  manda un comentario SSE. Si el usuario cierra la pestaña, la tarea se cancela.
- **Alerta `ingresos_fallidos`:** `ALERT_USER_LOGIN_FAILURES` ingresos fallidos en
  `ALERT_USER_LOGIN_WINDOW_MINUTES`, de todas las organizaciones, dice desde cuántas IPs.
- **Scripts:** `scripts/desarrollo.ps1` (backend y front en dos ventanas; revisa Postgres y
  no arranca con un `.env` de productivo) y `scripts/produccion.ps1` (para la tarea
  programada; no arranca sin `ENVIRONMENT=produccion`).
- **[PRODUCCION.md](PRODUCCION.md):** toda la configuración de productivo, con su porqué.
- Los tests corren como desarrollo y sin el front compilado aunque el `.env` sea de
  productivo (`tests/conftest.py`).
- **Puertos:** `SERVER_PORT` vacío es 8000 en productivo y 8001 en desarrollo
  (`Settings.port`); el proxy de Vite va al 8001 (decisión 26).

**Bloque 2, código de verificación y sesiones (hecho):**

- **La plataforma entra siempre con el código.** Sin código configurado, el login responde
  403 `falta_configurar_codigo` (con la contraseña correcta: no suma para el bloqueo, pero
  queda en el registro como ingreso fallido con motivo `sin_codigo`).
- **`python -m app.cli configurar-codigo`:** código nuevo para un administrador de plataforma
  que cambió o perdió el celular; le cierra todas las sesiones y queda en el registro
  (`totp_activado` con origen `configurar-codigo`). Solo para usuarios de plataforma.
- **Quitar el código desde Administración** (`POST /administracion/usuarios/{id}/quitar-codigo`,
  con la contraseña recién escrita): para quien perdió el celular. Le cierra todas las
  sesiones y queda en el registro como `codigo_quitado` (migración 0006). No sirve para el
  propio usuario (409 `propio_usuario`), para quien no tiene código (409 `sin_codigo`) ni para la
  plataforma (404, no pertenece a ninguna organización). En el diálogo del usuario, con
  confirmación.
- **«Usar acá» pide la contraseña** (y el código si lo tiene), igual que el login (decisión
  21): una contraseña mal cuenta para el bloqueo y queda en el registro con motivo
  `usar_aca`, y la ruta entra en el límite de intentos del login. La PC desplazada ya no
  retoma sola cuando la otra sale (se sacó `take_free_slot`), y después de «No fui yo» también
  hay que escribir la contraseña.
- **El registro de seguridad** dice el motivo de cada ingreso fallido.
- `scripts/sembrar_desarrollo.py` le pone a `prueba.plataforma` el código de la clave fija
  `DEV_TOTP_SECRET`.
- Probado en el navegador: PC desplazada con contraseña mal y bien, quitar el código a un
  usuario, el registro y el ingreso de la plataforma con su código.

**Bloque 3, seguridad en Postgres (hecho):**

- **Tres usuarios de Postgres, tres URLs:** `DATABASE_URL` (la app, con RLS),
  `SYSTEM_DATABASE_URL` (el sistema, `BYPASSRLS`) y `MIGRATIONS_DATABASE_URL` (el dueño de las
  tablas: migraciones, particiones y limpieza de lo vencido). Vacías en desarrollo, las dos
  últimas usan `DATABASE_URL` (con un aviso de que RLS no se aplica); en productivo son
  obligatorias y con tres usuarios distintos.
- **`scripts/preparar_postgres.py`**, con el superusuario `postgres` (lo corre el usuario):
  - `desarrollo`: `cotizador_dev_app` y `cotizador_dev_sistema` para las bases de desarrollo
    y de los tests, con la contraseña igual al nombre, como `cotizador_dev`, que sigue siendo
    el dueño (solo sirven en esta PC y las bases no tienen datos de clientes). Corrido el 9
    de noviembre.
  - `productivo --env <.env de productivo>`: `cotizador_owner`, `cotizador_app` y
    `cotizador_sistema` con contraseñas largas al azar y la base `cotizador`, a la que solo
    se conectan ellos tres. Se corre en el bloque 4, cuando esté la carpeta de productivo.
  - Las contraseñas viajan ya cifradas (SCRAM) y no se muestran. Se puede volver a correr sin
    cambiarlas.
  - Límites de la app y del sistema: 60 segundos por consulta y por transacción abierta sin
    hacer nada (la carga de la lista de precios de junio de 2026 tarda 0,7 segundos; la
    consulta más lenta, 0,01) y, en productivo, 30 conexiones cada uno.
- **Los permisos los da el backend** cada vez que actualiza la base (`app/database/roles.py`):
  filas de `app` y `observability`, nada de cambiar tablas; en `audit_log`, solo agregar y
  leer; nada en la tabla de versiones de Alembic.
- **La organización de cada transacción** (`app/database/tenant.py`): la fija `authenticate`
  (ahora async, para que valga en todo el pedido): la del usuario, el sistema para la
  plataforma, ninguna sin sesión. `asyncio.to_thread` y el stream de Cotizar la conservan.
- **Migración 0007:** la función `app.current_organization_id()` y las políticas de las 23
  tablas, sin forzar RLS al dueño.
- **Al arrancar** se revisan los usuarios (`check_database_users`): en productivo no arranca
  si la app es dueña de tablas, superusuario, puede crear usuarios o bases o se saltea RLS, o
  si el sistema no se lo saltea.
- **Tests (809):** corren con los mismos tres usuarios (los de desarrollo). Por cada tabla con
  RLS, cada organización ve solo lo suyo y sin organización no se ve nada; escribir una fila
  de otra organización falla; la app y el sistema no pueden editar el registro ni cambiar
  tablas; toda tabla nueva tiene que quedar clasificada; la lista cerrada del sistema; y la
  organización en los hilos y en el stream. El código de los tests corre como sistema; los
  pedidos, como en productivo.
- Probado en el navegador con la base de prueba: vendedor (presupuestos, usuarios, registro,
  cuentas, crear y borrar una aseguradora) y plataforma (organizaciones, registro, salud y
  Monitoreo), sin errores en el log.

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
18. **El nombre público es `app.oemia.com.ar`.**
19. **El usuario de plataforma entra desde cualquier navegador**, con contraseña y el código de
    verificación de la app del celular (obligatorio). No se le exige la sesión atada a la PC.
20. **Postgres se refuerza antes de publicar, con Row Level Security incluido** (no se espera al
    segundo cliente): solo conexiones locales, un usuario de Postgres para cada cosa y el
    registro de seguridad que la app no puede borrar.
21. **Volver a una PC desplazada pide ingresar de nuevo** (contraseña y código si lo tiene), no
    alcanza con «Usar acá» (9 de noviembre de 2026). Es fricción a propósito: si bastara un
    botón, un solo usuario se usaría en varias PCs, y el concesionario paga por usuario.
22. **El aviso de privacidad a los empleados (Ley 25.326) sale de la etapa 6** y queda pendiente
    hasta definir quién es el responsable de los datos: OEMIA como empresa todavía no está
    constituida y al principio puede ser el dueño como monotributista.
23. **Restablecer la contraseña por mail** va después de publicar, junto con las alertas por
    mail: las dos necesitan un servicio de envío de mails y el DNS del dominio en Cloudflare.
    Mientras tanto, la resetea un Administrador.
24. **Productivo va en una PC dedicada**, no en la de desarrollo. La primera prueba en internet
    se hace en la de desarrollo, con Cloudflare apuntando a ella, y después se pasa (ver
    [PRODUCCION.md](PRODUCCION.md#dónde-vive-cada-cosa)).
25. **`main` es el código de productivo y se publica solo, de noche**, con una tarea en la PC
    de productivo que busca los cambios, respalda la base, actualiza y vuelve atrás si falla;
    lo urgente se publica a mano. Los tests en GitHub Actions como condición para publicar,
    más adelante (ver [PRODUCCION.md](PRODUCCION.md#publicar-una-versión-nueva)).
26. **Productivo usa el puerto de siempre (8000) y desarrollo uno fijo propio (8001)**, sin
    configurar nada (`SERVER_PORT` vacío): productivo no se toca, los dos pueden correr en la
    misma PC y el front de desarrollo nunca le habla a la base de los clientes.
27. **La base de productivo es una nueva, `cotizador`**, de `cotizador_owner` y con solo los
    tres usuarios del backend conectándose; `cotizador_dev` queda para desarrollo. Los
    usuarios de desarrollo son otros (`cotizador_dev_app` y `cotizador_dev_sistema`): no
    pueden entrar a la base de productivo aunque estén en el mismo Postgres.

## Abiertas

1. **El Administrador que no cotiza** ve los presupuestos sin poder editarlos (propuesto, ya
   anda así).

## Pendientes

Lo que se decidió dejar para después de la etapa 6:

1. **Aviso de privacidad a los empleados** (Ley 25.326): qué datos se guardan (usuario,
   nombre, IP, dispositivo, ingresos, cotizaciones, registro de seguridad), para qué, cuánto
   tiempo, quién es el responsable y cómo pedir acceso, corrección o borrado. Espera a que se
   defina el responsable (decisión 22). Conviene que lo revise un abogado.
2. **Restablecer la contraseña por mail** y **las alertas por mail** (decisión 23).
3. **La publicación automática de noche** y la deploy key de solo lectura (decisión 25).
4. **Los tests en GitHub Actions** como condición para publicar (decisión 25).
5. **Pasar productivo a la PC dedicada** después de la primera prueba (decisión 24).
