# Productivo

La PC de productivo atiende a todos los usuarios por internet, en `app.oemia.ar`. Acá
está **toda la configuración de productivo**: cada valor, dónde se pone y por qué. Muchos problemas de
seguridad vienen de una configuración mal puesta, así que la regla es simple: **si se cambia
algo de productivo, se cambia también acá** (y se anota abajo, en [Cambios](#cambios)).

Las contraseñas y claves **no van en este documento**: solo dónde están guardadas.

Estado: se arma en la etapa 6 de [USUARIOS_Y_PERMISOS.md](USUARIOS_Y_PERMISOS.md). Lo marcado
como *pendiente* todavía no está hecho.

## Cómo llega un pedido

```
Usuario ──https──▶ Cloudflare ──túnel cifrado──▶ cloudflared ──▶ backend ──▶ Postgres
                   (certificado,                 (esta PC)        127.0.0.1   127.0.0.1:5432
                    solo Argentina)                               puerto 8000
```

- En el router no se abre ningún puerto: `cloudflared` sale hacia Cloudflare.
- El backend escucha solo en `127.0.0.1` y, en productivo, **rechaza con 403 todo pedido que no
  venga de esta misma PC**. Aunque alguien abriera el puerto en el firewall, no entra.
- El backend entrega también el front (la carpeta `dist`) y la API bajo `/api`.

## Dónde vive cada cosa

**Productivo va en una PC dedicada**, distinta de la de desarrollo: así reiniciar, probar algo
pesado o romper el entorno de desarrollo no afecta a los concesionarios. En esa PC corren solo
Postgres, el backend y `cloudflared`, con los repositorios en `main`, que es el código de
productivo.

**La primera prueba se hace en la PC de desarrollo** (la de Antelo), con el túnel de Cloudflare
apuntando a ella; después se pasa a la PC dedicada y se cambia el túnel. Desarrollo usa el
puerto 8001 y productivo el 8000, así que no se cruzan. Pero cada uno necesita su `.env`: para
tener los dos prendidos a la vez, productivo corre desde otra carpeta (una copia de los
repositorios en `main`) con el `.env` de productivo (`scripts\desarrollo.ps1` no arranca con
uno de productivo).

| Qué | Dónde | Estado |
|---|---|---|
| Código de productivo | El backend y el front en `main`, clonados de GitHub en `C:\OEMIA` (`Cotizador_BE` y `Cotizador_FE`; la documentación no hace falta) | en la PC de desarrollo, para la prueba (9 de noviembre) |
| Permisos de `C:\OEMIA` | Solo el usuario de Windows que corre el backend, SYSTEM y Administradores (ver abajo). Por defecto, cualquier usuario de la PC puede leer y cambiar lo que se crea en `C:\` | hecho (9 de noviembre) |
| Configuración del backend | `.env` del backend en esa PC (no se sube a git) | |
| Acceso a GitHub | Una «deploy key» de solo lectura por repositorio (son privados): la PC baja código y nunca puede subirlo | pendiente |
| Copia de las claves | Fuera de la PC de productivo (gestor de contraseñas o papel guardado) | pendiente |
| Base de datos | Postgres 18 en la PC de productivo, servicio `postgresql-x64-18` | |
| Logs del backend | `logs\` del backend, un archivo por día, 30 días | |

**Armar la carpeta** (lo de abajo, en este orden):

1. `git clone --branch main` del backend y del front en `C:\OEMIA`, `npm ci` y `npm run build`
   en el front.
2. El `.env` del backend sale de `.env.example` con los valores de [la tabla](#el-env-de-productivo)
   y una `CREDENTIALS_KEY` nueva (`uv run python -m app.cli generar-clave`).
3. Cerrar la carpeta, en PowerShell (no hace falta administrador):

   ```powershell
   icacls C:\OEMIA /inheritance:r /grant:r "${env:USERNAME}:(OI)(CI)F" "*S-1-5-18:(OI)(CI)F" "*S-1-5-32-544:(OI)(CI)F"
   ```

   (`S-1-5-18` es SYSTEM y `S-1-5-32-544`, Administradores.) Revisar con
   `icacls C:\OEMIA\Cotizador_BE\.env` que no aparezcan «Usuarios» ni «Usuarios
   autentificados».
4. `uv sync --frozen --extra otel` en el backend, desde la terminal del usuario (no desde la
   app de Claude: las instalaciones de Python que hace la app no las ve la terminal).
5. Los usuarios y la base: ver [Postgres](#postgres).

Al pasar a la PC dedicada: EPC, SAP y el localizador loguean desde la conexión a internet de
esa PC, `CREDENTIALS_KEY` se copia a su `.env` (sin ella no se leen las cuentas guardadas) y la
base se pasa con un respaldo de `pg_dump`.

## El `.env` de productivo

Solo lo que cambia respecto de los valores por defecto de `.env.example` o lo que conviene
tener presente. El resto queda como en `.env.example`.

| Variable | Valor | Por qué |
|---|---|---|
| `ENVIRONMENT` | `produccion` | Apaga `/docs`, atiende solo lo que llega por el túnel, manda HSTS y revisa la configuración al arrancar. |
| `SERVER_HOST` | `127.0.0.1` | Nadie de la red entra directo. El backend no arranca con otro valor. |
| `SERVER_PORT` | vacío (por defecto) | Vacío es 8000 en productivo (y 8001 en desarrollo). Tiene que coincidir con el túnel. |
| `FRONTEND_DIST` | `../Cotizador_FE/dist` (por defecto) | El front compilado. Sin él, el backend no arranca. |
| `CORS_ORIGINS` | `https://app.oemia.ar` | Con el front y la API en la misma dirección no hace falta CORS. Nunca `*` ni `http`. |
| `ALLOWED_HOSTS` | `app.oemia.ar` | El backend atiende solo los pedidos que nombran ese sitio (y los de esta PC por su nombre local). Antes de ponerlo, ver «Sitios que se atienden». |
| `MAX_BODY_KB` | Sin poner (1024) | Tamaño máximo de un pedido común a la API. Ver «Tamaño de los pedidos». |
| `CREDENTIALS_KEY` | clave secreta | Cifra las cuentas de EPC, SAP, el localizador y los códigos de verificación. Si se pierde, cada usuario vuelve a cargar sus cuentas. Copia fuera de la PC. |
| `ANTHROPIC_API_KEY` | clave secreta | La IA. |
| `DATABASE_URL` | `cotizador_app` en la base `cotizador` | Solo filas, con Row Level Security. La escribe `scripts/preparar_postgres.py`. |
| `SYSTEM_DATABASE_URL` | `cotizador_sistema` | Lo que no es de una sola organización (login, agente del DMS, alertas, consola, plataforma). Se saltea RLS. |
| `MIGRATIONS_DATABASE_URL` | `cotizador_owner` | El dueño de las tablas: migraciones, particiones y limpieza. El backend no arranca si las tres no usan usuarios distintos. |
| `RATE_LIMIT_ENABLED` | `true` | Sin límites, cualquiera prueba contraseñas sin parar o gasta la IA. El backend no arranca con `false`. |
| `RATE_LIMIT_LOGIN` | `5/15min` | Intentos de ingreso por IP (la IP real, ver abajo). |
| `AUTH_LOCKOUT_FAILURES` / `AUTH_LOCKOUT_MINUTES` | `5` / `15` | Contraseñas mal seguidas que bloquean el usuario, y por cuánto. |
| `SSE_KEEPALIVE_SECONDS` | `20` | Cloudflare corta a los 100 segundos sin recibir nada: Cotizar manda un «sigo» si no hay novedades. |
| `ALERT_USER_LOGIN_FAILURES` / `ALERT_USER_LOGIN_WINDOW_MINUTES` | `20` / `15` | Alerta de muchos ingresos fallidos (en Monitoreo y en el log). |
| `EPC_HEADFUL` / `SAP_HEADFUL` | `false` | Los navegadores de EPC y SAP sin ventana. |
| `LOG_LEVEL` | `INFO` | `DEBUG` llena el disco y muestra más de lo necesario. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://127.0.0.1:4318` | Las trazas van al Grafana de esta PC (`docker-compose.yml`). |
| `LLM_DAILY_BUDGET_USD` | `1` (el de `.env.example`) | Tope diario de la IA en dólares. Revisarlo cuando haya varios concesionarios cotizando. |
| `LLM_ORG_HARD_BUDGET_USD` | Sin poner (3 veces el diario) | Tope diario de cada organización: al llegar, queda sin IA hasta el otro día. Ver OBSERVABILIDAD.md, «Tope de gasto diario». |
| `GRAFANA_ADMIN_PASSWORD` | contraseña | La lee `docker-compose.yml`, no el backend: contraseña de `admin` en Grafana la primera vez que arranca con el volumen vacío. |
| `VEHICLE_PARTS_DAYS` | Sin poner (180) | Días que se recuerdan los números cotizados de cada chasis desde la última consulta. |
| `RECENT_NUMBERS_DAYS` | Sin poner (3) | Días del historial de números cotizados sin chasis. |
| `ANTHROPIC_MAX_RETRIES` | Sin poner (4) | Reintentos cuando Anthropic falla de su lado (503, sobrecarga, conexión) antes de avisar que la IA no responde. |

## Lo que hace el backend por su cuenta

Esto no se configura: está en el código y vale siempre que corresponde.

- **Headers de seguridad en todas las respuestas** (`app/core/web_security.py`):
  `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY` y `Referrer-Policy: same-origin`.
  En la API, además, `Cache-Control: no-store`: los datos de los clientes no quedan guardados
  ni en el navegador ni en Cloudflare.
- **Content-Security-Policy en las páginas del front** (`app/core/frontend.py`): scripts solo de
  esta dirección (y el del tema, por su hash), nada dentro de iframes ajenos, sin plugins.
  Se calcula sola desde el `index.html` compilado.
- **En productivo:**
  - sin `/docs`, `/redoc` ni `/openapi.json`;
  - solo pedidos que vienen de esta PC (o sea, por `cloudflared`); los demás, 403 y un aviso en
    el log;
  - `Strict-Transport-Security: max-age=31536000`: el navegador entra solo por https durante
    un año.
- **La IP real de cada pedido** sale de `CF-Connecting-IP`, y solo si el pedido viene de esta
  PC (`app/core/client_ip.py`). `X-Forwarded-For` no se usa nunca: lo puede inventar el
  usuario. Esa IP es la que usan los límites de intentos, el registro de seguridad, los logs y
  Monitoreo.
- **La ubicación de cada inicio de sesión** sale de los headers que agrega Cloudflare
  (`cf-ipcity`, `cf-region`, `cf-region-code`, `cf-ipcountry`), con la misma regla: solo si el
  pedido viene de esta PC (`app/core/client_location.py`). Se guarda en la sesión y la ve el
  usuario en Mis dispositivos, en lugar de la IP.
- **Archivos del front:** los de `assets/` (con hash en el nombre) se guardan un año en caché;
  `index.html` y los íconos se revalidan en cada visita, así una versión nueva llega sola.
- **Tamaño máximo de cada pedido** (`app/core/body_limit.py`): lo que se pasa se rechaza con
  413 antes de leerlo, también sin sesión. Ver «Tamaño de los pedidos».

### Tamaño de los pedidos

FastAPI lee el cuerpo entero de un pedido antes de mirar la sesión y los límites de uso, y
Cloudflare deja pasar hasta 100 MB. Sin un tope, alguien sin usuario podría llenar la memoria
del backend con pedidos gigantes al login. Por eso cada ruta tiene un máximo, que se revisa
antes de leer:

| Qué | Rutas | Máximo | Configuración |
|---|---|---|---|
| Listas que se suben | `POST /precios/cargar`, `/compatibilidades/cargar`, `/promociones/cargar` | 15 MB + 1 MB del formulario | `MAX_UPLOAD_MB` |
| Orden del seguro | `POST /presupuestos/{numero}/orden-seguro` | 10 MB + 1 MB | `MAX_INSURANCE_ORDER_MB` |
| Foto del stock del taller | `PUT /dms/stock` | 20 MB | `MAX_DMS_SNAPSHOT_MB` |
| Telemetría del front | `POST /telemetria` | 64 KB | `TELEMETRY_MAX_BYTES` |
| Pedidos con el logo | `POST` y `PATCH /administracion/organizaciones`, `PUT /presupuestos/concesionario` | 1,5 MB del logo + el común | `MAX_LOGO_CHARS` en `app/schemas/estimates.py` |
| Todo lo demás | | 1 MB | `MAX_BODY_KB` |

- Si el pedido dice cuánto mide (`Content-Length`) y se pasa, recibe el 413 sin que se guarde
  nada. Si no lo dice, se cuentan los bytes a medida que llegan y se rechaza al pasar el máximo.
- Lo que sobra se lee y se tira, sin guardarlo en memoria, hasta 32 MB: si el backend
  contestara con el navegador todavía mandando, la conexión se cortaría y el usuario vería un
  error de red en vez del mensaje. Un pedido que declara más de 32 MB se rechaza sin leer nada
  y se le corta la conexión.
- El 413 trae el código `pedido_muy_grande` y queda en Monitoreo y en el log («Pedido
  rechazado por tamaño»).
- Los controles de cada ruta (`read_limited` en las subidas y el de `/telemetria`) siguen
  estando: dan el mensaje preciso («El archivo supera los 15 MB») cuando se pasa por poco.

**Para aceptar algo más grande en una ruta nueva** (por ejemplo, fotos de un repuesto o de un
siniestro):

1. Crear la configuración con el máximo, como `MAX_PHOTO_MB`, en `app/core/config.py` y en
   `.env.example`.
2. Agregar el renglón de la ruta a `ROUTE_LIMITS` en `app/core/body_limit.py`, con su método y
   su dirección sin `/api`. Para un archivo en un formulario, con `_upload(...)`, que suma el
   margen del formulario.
3. En la ruta, leer el archivo con `read_limited` (`app/core/uploads.py`) y revisar que sea lo
   que dice ser por sus primeros bytes, como `is_pdf` en `app/services/acceptance.py`. Una
   imagen se revisa igual: JPEG empieza con `FF D8 FF`, PNG con `89 50 4E 47`, WebP con
   `RIFF....WEBP`. No alcanza con la extensión ni con el tipo que declara el navegador.
4. Al devolverla, mandar el tipo exacto (`image/jpeg`), nunca uno que venga del usuario. SVG no:
   puede traer scripts.
5. Dos cosas a pensar antes: dónde se guardan (hoy la orden del seguro va a la base; muchas
   fotos conviene que vayan a disco o a un almacenamiento aparte) y un límite de uso para la
   ruta en `app/core/rate_limit.py`, porque subir fotos sin parar llena el disco.
6. No subir `MAX_BODY_KB` para que entren: ese máximo vale para todas las rutas, incluido el
   login sin sesión.

### Sitios que se atienden

Con `ALLOWED_HOSTS`, el backend atiende solo los pedidos cuyo header `Host` nombra uno de esos
sitios. Los pedidos a `127.0.0.1` o `localhost` pasan siempre, para revisar la salud desde la
PC. Sirve contra una página cualquiera abierta en un navegador de la PC de productivo que haga
apuntar su propio nombre a `127.0.0.1` para hablarle al backend sin pasar por Cloudflare.

Vacío no revisa nada. Antes de ponerlo en productivo hay que confirmar con qué `Host` llegan los
pedidos por el túnel: `cloudflared` manda el del sitio (`app.oemia.ar`) salvo que la ruta del
túnel tenga configurado otro en «HTTP Host Header». Si quedara mal, todos los pedidos reciben
400 y el log dice «pide el sitio '…', que no está en ALLOWED_HOSTS»: se corrige el valor o se
vacía, y se reinicia.

### Lo que revisa al arrancar

En productivo, si algo de esto falla, **el backend no arranca** y el log dice qué cambiar
(`app/core/production.py`):

- `CORS_ORIGINS` sin `*` y solo con direcciones `https`.
- `SERVER_HOST` es `127.0.0.1`.
- Está el front compilado en `FRONTEND_DIST`.
- Está `CREDENTIALS_KEY`.
- `RATE_LIMIT_ENABLED` prendido.
- Están `SYSTEM_DATABASE_URL` y `MIGRATIONS_DATABASE_URL`, y las tres URLs de la base usan
  usuarios distintos.
- Ya conectado a Postgres (`app/database/roles.py`): el usuario de la app no es dueño de
  tablas, no es superusuario, no puede crear usuarios ni bases y no se saltea Row Level
  Security; el de sistema sí se lo saltea.

`CORS_ORIGINS` con `*` no arranca ni en desarrollo. En desarrollo, lo de Postgres solo avisa en
el log.

## Cómo se levanta

- **Siempre con `scripts\produccion.ps1`** (o `python -m app.server`), **nunca con
  `uvicorn main:app`**. El comando propio deja fijo lo que importa: escuchar solo en
  `127.0.0.1`, apagar los «proxy headers» de uvicorn (si no, uvicorn le cree a
  `X-Forwarded-For` y se puede esquivar el límite de intentos del login), no anunciar
  `server: uvicorn` y esperar hasta 30 segundos a las cotizaciones en curso al apagar.
  `produccion.ps1` además se niega a arrancar si el `.env` no dice `ENVIRONMENT=produccion` y
  usa las dependencias exactas de `uv.lock`.
- Si alguien lo levantara con `uvicorn main:app`, uvicorn cambiaría la IP de la conexión y el
  backend de productivo rechazaría todos los pedidos con 403: falla a la vista, no en silencio.
- **Al prender la PC:** la tarea programada «OEMIA backend productivo» (ver abajo).
- **Publicar una versión nueva:** ver [Publicar una versión nueva](#publicar-una-versión-nueva).

### La tarea programada

Hecha el 9 de noviembre de 2026 (bloque 4). La registra el usuario en PowerShell como
administrador:

```powershell
$tarea = "OEMIA backend productivo"
$accion = New-ScheduledTaskAction -Execute "powershell.exe" -Argument '-NoProfile -WindowStyle Hidden -File "C:\OEMIA\Cotizador_BE\scripts\produccion.ps1"' -WorkingDirectory "C:\OEMIA\Cotizador_BE"
$alPrender = New-ScheduledTaskTrigger -AtStartup
$alPrender.Delay = "PT1M"
$cadaMinuto = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) -RepetitionInterval (New-TimeSpan -Minutes 1)
$usuario = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -LogonType S4U -RunLevel Limited
$ajustes = New-ScheduledTaskSettingsSet -ExecutionTimeLimit ([TimeSpan]::Zero) -MultipleInstances IgnoreNew -StartWhenAvailable -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -Priority 4
Register-ScheduledTask -TaskName $tarea -Description "Backend de productivo de OEMIA (C:\OEMIA). Si se cae, lo vuelve a levantar en un minuto. Ver PRODUCCION.md." -Action $accion -Trigger $alPrender, $cadaMinuto -Principal $usuario -Settings $ajustes
```

| Qué | Cómo y por qué |
|---|---|
| Cuándo arranca | Un minuto después de prender la PC (así Postgres y la red ya están). Y cada minuto Windows mira si sigue corriendo: si se cayó, lo levanta; si corre, no hace nada (`IgnoreNew`). |
| Con qué usuario | El de Windows dueño de `C:\OEMIA`, sin administrador y aunque nadie haya iniciado sesión (`S4U`: no guarda la contraseña, y no puede entrar a carpetas compartidas de la red, que no usa). Es el usuario que tiene `uv`, Python y los navegadores de EPC. Corre en la sesión 0, apartado: los programas de la sesión del usuario sin administrador no pueden cerrarlo. |
| Límite de tiempo | Ninguno (`PT0S`). Por defecto Windows corta las tareas a las 72 horas. |
| Prioridad | Normal (4). Por defecto las tareas corren con prioridad baja. |

**Pararlo para mantenimiento** (PowerShell como administrador). Primero se deshabilita, si no
la tarea lo vuelve a levantar al minuto:

```powershell
Disable-ScheduledTask -TaskName "OEMIA backend productivo"
Stop-ScheduledTask -TaskName "OEMIA backend productivo"
# Si el 8000 sigue ocupado, cerrar el que escucha:
Get-NetTCPConnection -LocalPort 8000 -State Listen -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

**Volver a prenderlo:** `Enable-ScheduledTask -TaskName "OEMIA backend productivo"` y
`Start-ScheduledTask -TaskName "OEMIA backend productivo"`.

**Ver si anda:** `Get-ScheduledTask -TaskName "OEMIA backend productivo"` (estado `Running`) y
el log del backend en `C:\OEMIA\Cotizador_BE\logs\`.

## Publicar una versión nueva

Hecho el 19 de noviembre de 2026. **`main` es el código de productivo**: lo que se mergea a
`main` se publica esa noche, si los tests pasan. Lo que no está listo queda en su rama.

**Cómo funciona:**

1. En cada push a `main`, GitHub Actions corre los tests de cada repo
   (`.github/workflows/tests.yml`): en el backend ruff y la suite completa con un Postgres 18;
   en el front typecheck, lint, tests y build. Si pasan, mueve la rama **`publicable`** a ese
   commit. `npm run check:api` no corre allá (necesita el backend, que es otro repo privado):
   se corre en la PC antes de pushear.
2. A las 3 de la mañana, la tarea programada **«OEMIA publicar»** corre
   `C:\OEMIA\Cotizador_BE\scripts\publicar.ps1`, que sale a buscar los cambios por SSH (nada
   de afuera entra a la PC):
   - publica solo si en el backend **y** en el front `publicable` está en el mismo commit que
     `main`. Si alguno tiene commits sin tests en verde, esa noche no publica nada (nunca sale
     una mitad) y lo deja en el log;
   - si no hay nada nuevo, termina sin tocar nada;
   - respalda la base con `pg_dump` en `C:\OEMIA\respaldos\cotizador_publicar_*.dump`
     (quedan los últimos 10; los respaldos hechos a mano no se borran);
   - baja el código, corre `uv sync` si cambió `uv.lock` y `npm ci` si cambió
     `package-lock.json`, y compila el front guardando los archivos viejos;
   - reinicia el backend y espera hasta 4 minutos a que `/api/salud` responda con el proceso
     nuevo y la página pida el front recién compilado;
   - si anduvo, pone el tag con la fecha (`2026.11.19`, o `2026.11.19.2` si ya hubo uno ese
     día) en los dos repos y lo sube a GitHub. Al llegar el tag, GitHub Actions
     (`.github/workflows/release.yml`) crea el **Release** de cada repo, con los commits desde
     la publicación anterior: ahí se ve qué salió cada noche;
   - si no anduvo, vuelve el código a la versión anterior, recompila y reinicia. **Las
     migraciones no se deshacen solas**: el log dice qué respaldo restaurar si hace falta.
3. Si la PC estaba apagada a las 3, no publica al prenderla (sería en horario de atención):
   espera a la noche siguiente.

**El log:** `C:\OEMIA\Cotizador_BE\logs\publicar-AAAA-MM.log`.

**A mano** (urgente, o para probar), en PowerShell como administrador:

```powershell
C:\OEMIA\Cotizador_BE\scripts\publicar.ps1 -Prueba   # dice qué haría, sin cambiar nada
C:\OEMIA\Cotizador_BE\scripts\publicar.ps1           # publica ya
```

Los tests tienen que estar en verde igual: un hotfix es mergear a `main`, esperar a que GitHub
Actions termine (unos minutos; se ve en la pestaña Actions) y correr el script.

**Para dejarlo andando (una sola vez):**

1. **Claves SSH**, una por repo, con el usuario de Windows dueño de `C:\OEMIA` (PowerShell
   común):
   ```powershell
   ssh-keygen -t ed25519 -f "$HOME\.ssh\oemia_be" -N '""' -C "oemia-productivo-backend"
   ssh-keygen -t ed25519 -f "$HOME\.ssh\oemia_fe" -N '""' -C "oemia-productivo-front"
   ```
2. En GitHub, en cada repo: **Settings → Deploy keys → Add deploy key**, con el contenido del
   `.pub` correspondiente y **Allow write access** tildado (hace falta para subir el tag).
3. En `$HOME\.ssh\config`:
   ```
   Host github-oemia-be
     HostName github.com
     User git
     IdentityFile ~/.ssh/oemia_be
     IdentitiesOnly yes

   Host github-oemia-fe
     HostName github.com
     User git
     IdentityFile ~/.ssh/oemia_fe
     IdentitiesOnly yes
   ```
4. Los repos de productivo pasan a SSH y firman los tags con un nombre propio:
   ```powershell
   git -C C:\OEMIA\Cotizador_BE remote set-url origin git@github-oemia-be:FacuAntelo/Cotizador_BE.git
   git -C C:\OEMIA\Cotizador_FE remote set-url origin git@github-oemia-fe:FacuAntelo/Cotizador_FE.git
   ssh -T git@github-oemia-be   # la primera vez pregunta por la huella de GitHub: «yes»
   ssh -T git@github-oemia-fe
   git -C C:\OEMIA\Cotizador_BE config user.name "OEMIA productivo"
   git -C C:\OEMIA\Cotizador_BE config user.email "productivo@oemia.local"
   git -C C:\OEMIA\Cotizador_FE config user.name "OEMIA productivo"
   git -C C:\OEMIA\Cotizador_FE config user.email "productivo@oemia.local"
   ```
5. **La tarea**, en PowerShell como administrador y con ese mismo usuario:
   ```powershell
   C:\OEMIA\Cotizador_BE\scripts\publicar.ps1 -Registrar
   ```
   Queda todos los días a las 3, con el mismo usuario que el backend, aunque nadie haya
   iniciado sesión (`S4U`), y con privilegios altos: el backend corre en la sesión 0 y sin
   ellos no se lo puede cerrar.
6. **Probarla:** `publicar.ps1 -Prueba`. Tiene que decir qué publicaría, o que no hay nada
   nuevo. Si dice que main todavía no tiene los tests en verde, GitHub Actions no terminó (o
   falló) sobre ese commit: se mira en la pestaña Actions del repo.

Las claves de despliegue con escritura también podrían pushear a `main`: si la PC se
compromete, se borran desde GitHub (Settings → Deploy keys) y no queda ningún otro acceso.

## Si alguien pierde el celular

- **Un usuario de una organización:** un Administrador le quita el código en Administración →
  Usuarios → «Quitar código». Se le cierran todas las sesiones; entra con la contraseña y, si
  quiere, configura un código nuevo desde el menú de su usuario.
- **Un administrador de plataforma:** desde la consola de la PC de productivo,
  `uv run python -m app.cli configurar-codigo`. Le cierra todas las sesiones. Desde la web no
  se puede: la plataforma entra siempre con el código.

## Postgres

Detalle y motivos en [Seguridad en Postgres](USUARIOS_Y_PERMISOS.md#seguridad-en-postgres).

**Usuarios y base** (bloque 3, hecho; en productivo se corre en el bloque 4). Con la carpeta de
productivo y su `.env` (que dice `ENVIRONMENT=produccion`), desde esa carpeta:

```powershell
uv run python scripts/preparar_postgres.py productivo --env C:\ruta\de\productivo\Cotizador_BE\.env
```

Pide la contraseña de `postgres` (no se muestra ni se guarda) y:

| Qué | Cómo queda |
|---|---|
| `cotizador_owner` | Dueño de la base `cotizador` y de sus tablas. Migraciones, particiones mensuales y limpieza de lo vencido. Hasta 5 conexiones, sin límite de tiempo (una migración puede tardar). |
| `cotizador_app` | Solo leer, agregar, cambiar y borrar filas, con Row Level Security. En `audit_log`, solo agregar y leer. Nada de crear o borrar tablas, usuarios ni bases. 60 segundos por consulta y por transacción sin hacer nada; hasta 30 conexiones. |
| `cotizador_sistema` | Lo mismo que la app, pero se saltea Row Level Security. |
| Base `cotizador` | UTF-8 y orden en español (ICU `es-AR`). Solo se conectan esos tres usuarios. |
| Contraseñas | Largas al azar, en el `.env` de productivo y en ningún otro lado. Volver a correr el script no las cambia. Copia fuera de la PC, junto con `CREDENTIALS_KEY`. |

Los permisos sobre las tablas los da el backend cada vez que arranca, y antes de atender
revisa los usuarios: si la app es dueña de tablas, superusuario o se saltea RLS, o si el
sistema no se lo saltea, no arranca.

**Solo en esta PC** (bloque 4, lo hace el usuario como administrador), en
`C:\Program Files\PostgreSQL\18\data`:

| Qué | Valor | Estado |
|---|---|---|
| `postgresql.conf` | `listen_addresses = 'localhost'`: escucha solo en `127.0.0.1` y `::1`, nadie de la red llega al 5432 | hecho (9 de noviembre) |
| `postgresql.conf` | `log_connections = on` (sin el `#` adelante): cada conexión queda en el log de Postgres | hecho (9 de noviembre) |
| `pg_hba.conf` | Solo `host all all 127.0.0.1/32 scram-sha-256` y `host all all ::1/128 scram-sha-256`; el resto, con `#` | hecho (9 de noviembre) |
| Usuario `postgres` | Contraseña fuerte, guardada fuera de la PC (`psql -h 127.0.0.1 -U postgres -c "\password postgres"`) | hecho (9 de noviembre) |

Después de cambiar los archivos: `Restart-Service postgresql-x64-18` (PowerShell como
administrador). En Docker corre solo Grafana, que no usa Postgres.

## Cloudflare

**Dominios** (plan gratis): `oemia.ar` es el de la app (`app.oemia.ar`) y `oemia.com.ar`
redirige a `oemia.ar`. Los dos están delegados a Cloudflare desde NIC Argentina (hecho el 9
de noviembre de 2026). Hasta que haya una web en `oemia.ar`, `oemia.ar` y `www.oemia.ar`
llevan a la app.

**Redirección de `oemia.com.ar`** (hecha el 9 de noviembre de 2026), en la zona `oemia.com.ar`:

1. **DNS → Registros:** un registro `A` para `@` y otro para `www`, los dos con la IP
   `192.0.2.1` y **con proxy** (nube naranja). Esa IP no es de nadie: Cloudflare contesta antes
   de llegar a ella. Si al pasar el dominio se importaron registros `A`, `AAAA` o `CNAME` para
   `@` o `www`, se reemplazan; los `MX` y `TXT` (mail) no se tocan.
2. **Reglas → Crear regla → Regla de redirección** (Redirect Rule), llamada «A oemia.ar»:
   - Si coincide: **Todas las solicitudes entrantes**.
   - Entonces: **Dinámico**, con la expresión
     `concat("https://oemia.ar", http.request.uri.path)`, código **301** y **Conservar la
     cadena de consulta** marcado.
3. **SSL/TLS → Certificados perimetrales:** «Usar siempre HTTPS» prendido. El certificado de
   `oemia.com.ar` y `www.oemia.com.ar` lo pone Cloudflare solo (puede tardar unos minutos).
4. Probar: `https://oemia.com.ar/prueba?x=1` y `http://www.oemia.com.ar/prueba?x=1` tienen que
   terminar en `https://oemia.ar/prueba?x=1`. Mientras `oemia.ar` no tenga una web, ahí se ve
   un error de Cloudflare: es lo esperado.

**Redirección de `oemia.ar` a la app** (temporal, hasta que haya una web), en la zona
`oemia.ar`:

1. **DNS → Registros:** un registro `A` para `@` y otro para `www`, los dos con la IP
   `192.0.2.1` y **con proxy**, como en `oemia.com.ar`. El de `app` (el del túnel) no se toca.
2. **Reglas → Crear regla → Regla de redirección**, llamada «oemia.ar a la app (temporal)»:
   - Si coincide: **Expresión de filtro personalizada**, **Nombre de host** **está en**
     `oemia.ar` y `www.oemia.ar` (`(http.host in {"oemia.ar" "www.oemia.ar"})`). **No** «Todas
     las solicitudes entrantes»: entraría `app.oemia.ar` y la app se redirigiría a sí misma
     sin fin.
   - Entonces: **Estático**, `https://app.oemia.ar/`, código **302**, sin conservar la cadena
     de consulta. Es 302 y no 301 porque el 301 el navegador lo recuerda para siempre: cuando
     esté la web, quien ya entró seguiría yendo a la app.
3. Probar: `https://oemia.ar`, `http://www.oemia.ar` y `https://oemia.com.ar` tienen que
   terminar en `https://app.oemia.ar/`.

Cuando esté la web, se borra la regla y los registros `A` de `@` y `www` pasan a apuntar a
ella.

**Página de mantenimiento** (Worker `oemia-mantenimiento`, código en
`Cotizador_FE/cloudflare/mantenimiento/`). Si alguien abre la app y el servidor no contesta
(PC apagada o túnel caído: error 1033; backend caído o reiniciándose: 502), en vez del error
de Cloudflare ve la página de OEMIA con la imagen, que se fija cada 20 segundos y abre la app
sola cuando vuelve. El plan gratis no deja cambiar las páginas de error de Cloudflare: lo
hace un Worker delante de `app.oemia.ar`.

- Solo cambia las páginas que se abren, y solo si Cloudflare o el túnel dicen que no llegaron
  al backend (502, 503, 504, 520 a 524 y 530). Con el servidor andando no aparece: un 500 del
  backend es un error del código y pasa tal cual.
- Lo demás pasa tal cual y la API no pasa por el Worker (ruta sin Worker): Cotizar, el agente
  del DMS y los límites por IP no se enteran. Con la app ya abierta, si el backend se cae, los
  pedidos a la API dan error en la app; la página aparece al recargar.
- La página lleva adentro la letra, el ícono y la imagen: no le pide nada al servidor caído.
  No tiene dirección propia: solo se ve con el servidor caído.
- Plan gratis: 100.000 pedidos por día al Worker, sin contar la API. Si se pasara, la ruta en
  **Fail open** sigue llevando a la app, solo que sin la página.

Para subirla o cambiar la imagen (desde la PC de desarrollo, en una terminal propia, no la de
Claude):

1. Dejar una sola imagen (png, jpg, webp o gif, menos de 1 MB; hoy `mantenimiento.jpeg`) en
   `Cotizador_FE/cloudflare/mantenimiento/`. Si es otra, cambiar `IMAGE_ALT` en `armar.mjs`
   por lo que muestra y dice. Después correr ahí `node armar.mjs`.
2. La primera vez, `npx wrangler@4.153.0 login` (abre Cloudflare para dar permiso).
3. `npx wrangler@4.153.0 deploy`: sube el Worker y crea la ruta `app.oemia.ar/*`.
4. La primera vez, en la zona `oemia.ar` → **Rutas de Workers**:
   - en `app.oemia.ar/*`, **Request limit failure mode** en **Fail open (proceed)**;
   - **Agregar ruta** `app.oemia.ar/api/*` con Worker **Ninguno**.
5. Probar: la app anda igual y Monitoreo sigue mostrando la IP real. Para ver la página, al
   reiniciar el backend abrir la app en ese minuto: tiene que aparecer y volver sola.

**La app en `app.oemia.ar`** (bloque 4, hecho el 9 de noviembre de 2026 en la PC de
desarrollo, para la prueba), en la zona `oemia.ar`:

| Qué | Cómo quedó |
|---|---|
| `cloudflared` | Instalado con `winget install --id Cloudflare.cloudflared`; servicio de Windows `cloudflared` (automático, como LocalSystem) con la llave del túnel. La llave no se anota en ningún lado: si se pierde, se reinstala desde el panel. |
| Túnel | Uno por PC, creado en Zero Trust → Networks → Tunnels. Public hostname `app.oemia.ar` → `HTTP` `127.0.0.1:8000`. Al pasar a la PC dedicada se crea otro túnel, se prueba, se le pasa `app.oemia.ar` y se borra el de esta PC (así su llave deja de servir). |
| Firewall | Security → WAF → Custom rules: `(http.host eq "app.oemia.ar" and ip.src.country ne "AR")` → **Block**. |
| HTTPS | SSL/TLS → Edge Certificates: «Always Use HTTPS» prendido y «Minimum TLS Version» 1.2. El HSTS de Cloudflare, **apagado**: lo manda el backend solo para `app.oemia.ar`; el de Cloudflare vale para toda la zona. |
| Ubicación | Rules → Settings → Managed Transforms: «Add visitor location headers» prendido (15/11/2026). Le pasa al backend la ciudad, la provincia y el país de cada pedido, para la ubicación de Mis dispositivos. Apagado, todo sigue andando sin ubicación. |
| Sin tocar | Bot Fight Mode y «Under Attack» apagados (frenan al agente del DMS). Sin reglas de caché para `/api` (el backend igual manda `no-store`). |

Probado por internet: `http` pasa a `https` con 301, `/api/salud` trae HSTS, `nosniff`, `DENY`
y `no-store` (`cf-cache-status: DYNAMIC`), `/docs` muestra la app, `/openapi.json` da 404, TLS
1.1 se rechaza y Monitoreo guarda la IP pública real.

**Lo que falta** (bloque 5): que ningún pedido pase 100 segundos sin respuesta (error 524),
DBSC por https y el agente del DMS.

## Windows

*Pendiente (bloque 4).*

- Que no se suspenda ni hiberne.
- Actualizaciones que reinicien fuera del horario de atención.
- El firewall sin reglas que abran el 5432 ni el puerto del backend.
- «Cifrado del dispositivo», si esta PC lo tiene.
- UPS.
- Respaldo diario de la base, cifrado y fuera de esta PC.

## Revisión rápida

Después de cualquier cambio de configuración:

1. `https://app.oemia.ar/docs` muestra la app, no la documentación de la API.
2. `https://app.oemia.ar/api/salud` responde y trae `strict-transport-security`,
   `x-content-type-options` y `cache-control: no-store`.
3. Ingresar con una contraseña mal y ver en Administración → Registro de seguridad que la IP es
   la real y no `127.0.0.1`.
4. Desde otra PC de la red, `http://<IP de la PC de productivo>:5432` y `:8000` no responden.
5. El log del arranque no dice «Los usuarios de Postgres no sirven» ni «Row Level Security no
   se aplica».

*Pendiente:* un comando que revise todo esto solo, incluido lo de Postgres y Windows.

## Cambios

| Fecha | Cambio |
|---|---|
| 2026-11-19 | Publicación automática: GitHub Actions corre los tests y mueve `publicable`; `scripts/publicar.ps1` publica de noche con respaldo, vuelta atrás y tag con la fecha. Falta dejarla andando en la PC (claves SSH y la tarea). |
| 2026-11-16 | Revisión de seguridad (`PLAN_SEGURIDAD.md`), en la rama `menu-usuario`: tope de tamaño de los pedidos, `ALLOWED_HOSTS`, tope de IA por organización, el bloqueo ya no confirma la contraseña, una sesión se ata a la PC una sola vez y Grafana pide usuario. |
| 2026-11-15 | Cloudflare manda la ubicación de cada pedido («Add visitor location headers»), para Mis dispositivos. La guarda la migración 0017 desde que se publique la rama `menu-usuario`. |
| 2026-11-10 | `oemia.ar` lleva a la app (302, hasta que haya una web) y página de mantenimiento con un Worker de Cloudflare. |
| 2026-11-10 | Repuestos consultados por chasis y la revisión de las candidatas del catálogo: la migración 0009 corre sola al arrancar. |
| 2026-11-10 | Arreglo de Precios con la matriz de reemplazos cargada y el front llamando a la API solo con el cliente tipado (`npm run check:api`); cómo publicar a mano. |
| 2026-11-09 | Tarea programada «OEMIA backend productivo»: arranca al prender la PC y lo vuelve a levantar si se cae. |
| 2026-11-09 | Túnel de Cloudflare `app.oemia.ar` → `127.0.0.1:8000` desde la PC de desarrollo, solo Argentina, TLS 1.2 como mínimo. |
| 2026-11-09 | Usuarios de productivo y base `cotizador` creados; primer arranque del backend de productivo, con la revisión en orden. |
| 2026-11-09 | Carpeta de productivo `C:\OEMIA` (backend y front en `main`) con su `.env` y `CREDENTIALS_KEY` nueva; las trazas, al Grafana de la PC. |
| 2026-11-09 | Etapa 6, bloque 4: `oemia.com.ar` redirige a `oemia.ar` y Postgres escucha solo en esta PC (`listen_addresses`, `pg_hba.conf`, contraseña nueva de `postgres`). |
| 2026-11-09 | El nombre público pasa a `app.oemia.ar`; `oemia.com.ar` redirige a `oemia.ar`. Los dos dominios, en Cloudflare. |
| 2026-11-09 | Etapa 6, bloque 3: tres usuarios de Postgres (app con Row Level Security, sistema y dueño), `scripts/preparar_postgres.py` y la base nueva `cotizador`. |
| 2026-11-09 | Etapa 6, bloque 2: la plataforma entra siempre con el código, `configurar-codigo`, quitar el código desde Administración y «Usar acá» con contraseña. |
| 2026-11-09 | El puerto sale del entorno: 8000 en productivo, 8001 en desarrollo. |
| 2026-11-09 | Productivo va en una PC dedicada; la primera prueba, en la de desarrollo. Se publica solo de noche desde `main` (pendiente). |
| 2026-11-09 | Etapa 6, bloque 1: `ENVIRONMENT`, `python -m app.server`, headers de seguridad, CSP, IP real desde Cloudflare, front servido por el backend, aviso «sigo» en Cotizar, alerta de ingresos fallidos y revisión de la configuración al arrancar. |
