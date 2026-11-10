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
| Código de productivo | Los tres repositorios en `main`, en la PC de productivo | pendiente (por ahora, la prueba en la de desarrollo) |
| Configuración del backend | `.env` del backend en esa PC (no se sube a git) | |
| Acceso a GitHub | Una «deploy key» de solo lectura por repositorio (son privados): la PC baja código y nunca puede subirlo | pendiente |
| Copia de las claves | Fuera de la PC de productivo (gestor de contraseñas o papel guardado) | pendiente |
| Base de datos | Postgres 18 en la PC de productivo, servicio `postgresql-x64-18` | |
| Logs del backend | `logs\` del backend, un archivo por día, 30 días | |

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
- **Archivos del front:** los de `assets/` (con hash en el nombre) se guardan un año en caché;
  `index.html` y los íconos se revalidan en cada visita, así una versión nueva llega sola.

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
- **Al prender la PC:** una tarea programada de Windows corre `produccion.ps1` y lo reinicia si
  se cae (pendiente, bloque 4).
- **Publicar una versión nueva:** ver abajo.

## Publicar una versión nueva

*Pendiente.* Acordado el 9 de noviembre de 2026:

- **`main` es el código de productivo.** Lo que se mergea a `main` se publica; lo que no está
  listo queda en su rama.
- **Se publica solo, de noche**, no en cada merge: reiniciar en horario de atención corta las
  cotizaciones en curso y vuelve a loguear EPC y SAP, y si algo sale mal le pega a todos a la
  vez sin nadie mirando. De noche hay tiempo de arreglarlo antes de abrir.
- **Lo hace una tarea programada en la PC de productivo** (`publicar.ps1`, a las 3 de la mañana
  salvo que se elija otro horario), que sale a buscar los cambios: nada de afuera entra a la PC.
  Si `main` no cambió en el backend ni en el front, no hace nada. Si cambió:
  1. respaldo de la base (las migraciones no se deshacen solas);
  2. baja el código nuevo, instala las dependencias exactas y compila el front;
  3. reinicia el backend y revisa que `/api/salud` responda;
  4. si no responde, vuelve a la versión anterior y lo deja en el log.
- **Lo urgente, a mano:** `publicar.ps1 -Ahora`.
- **Backend y front se mergean el mismo día** cuando un cambio necesita a los dos: si no, a la
  noche se publica una mitad.
- **Más adelante:** que GitHub Actions corra los tests en cada push y que la tarea de la noche
  publique solo commits con los tests en verde.

## Si alguien pierde el celular

- **Un usuario de una organización:** un Administrador le quita el código en Administración →
  Usuarios → «Quitar código». Se le cierran todas las sesiones; entra con la contraseña y, si
  quiere, configura un código nuevo en Mi cuenta.
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
| `postgresql.conf` | `log_connections = on` (sin el `#` adelante): cada conexión queda en el log de Postgres | pendiente |
| `pg_hba.conf` | Solo `host all all 127.0.0.1/32 scram-sha-256` y `host all all ::1/128 scram-sha-256`; el resto, con `#` | hecho (9 de noviembre) |
| Usuario `postgres` | Contraseña fuerte, guardada fuera de la PC (`psql -h 127.0.0.1 -U postgres -c "\password postgres"`) | hecho (9 de noviembre) |

Después de cambiar los archivos: `Restart-Service postgresql-x64-18` (PowerShell como
administrador). En Docker corre solo Grafana, que no usa Postgres.

## Cloudflare

**Dominios** (plan gratis): `oemia.ar` es el de la app (`app.oemia.ar`) y `oemia.com.ar`
redirige a `oemia.ar`. Los dos están delegados a Cloudflare desde NIC Argentina (hecho el 9
de noviembre de 2026).

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

**Lo que falta del lado de Cloudflare** (bloques 4 y 5):

- Túnel `app.oemia.ar` → `http://127.0.0.1:8000`, con `cloudflared` como servicio de Windows.
  Primero en la PC de desarrollo, para la prueba; después se instala en la PC de productivo y se
  apaga en la otra.
- Regla del firewall: solo Argentina.
- «Always Use HTTPS» prendido. HSTS ya lo manda el backend.
- No crear reglas de caché para `/api` (el backend igual manda `no-store`).
- Ningún pedido puede pasar 100 segundos sin respuesta (error 524).

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
| 2026-11-09 | Etapa 6, bloque 4: `oemia.com.ar` redirige a `oemia.ar` y Postgres escucha solo en esta PC (`listen_addresses`, `pg_hba.conf`, contraseña nueva de `postgres`). |
| 2026-11-09 | El nombre público pasa a `app.oemia.ar`; `oemia.com.ar` redirige a `oemia.ar`. Los dos dominios, en Cloudflare. |
| 2026-11-09 | Etapa 6, bloque 3: tres usuarios de Postgres (app con Row Level Security, sistema y dueño), `scripts/preparar_postgres.py` y la base nueva `cotizador`. |
| 2026-11-09 | Etapa 6, bloque 2: la plataforma entra siempre con el código, `configurar-codigo`, quitar el código desde Administración y «Usar acá» con contraseña. |
| 2026-11-09 | El puerto sale del entorno: 8000 en productivo, 8001 en desarrollo. |
| 2026-11-09 | Productivo va en una PC dedicada; la primera prueba, en la de desarrollo. Se publica solo de noche desde `main` (pendiente). |
| 2026-11-09 | Etapa 6, bloque 1: `ENVIRONMENT`, `python -m app.server`, headers de seguridad, CSP, IP real desde Cloudflare, front servido por el backend, aviso «sigo» en Cotizar, alerta de ingresos fallidos y revisión de la configuración al arrancar. |
