# Plan de seguridad

Lo que salió de la revisión de seguridad del 16 de noviembre de 2026: qué se arregló, qué falta
y cómo hacerlo. La revisión fue de solo lectura sobre el código de `menu-usuario` y sobre la
carpeta de productivo `C:\Oemia`; no se probó nada contra `app.oemia.ar`.

Todo el código está en la rama `menu-usuario` de los tres repositorios, así que llega a
productivo cuando se publique esa rama.

Cuando un punto se termina, se borra de acá: lo que hay que recordar ya está en `PRODUCCION.md`,
`USUARIOS_Y_PERMISOS.md` u `OBSERVABILIDAD.md`.

## Hecho en `menu-usuario` (lote 1)

Falta publicarlo. Decidido el 16 de noviembre de 2026.

| Qué | Cómo quedó | Dónde está escrito |
|---|---|---|
| Pedidos gigantes sin sesión | Cada ruta tiene un tamaño máximo que se revisa antes de leer (1 MB lo común) | `PRODUCCION.md`, «Tamaño de los pedidos» |
| El bloqueo confirmaba la contraseña | Con el usuario bloqueado el login responde «usuario bloqueado» con la contraseña bien o mal, sin mirarla | `USUARIOS_Y_PERMISOS.md`, «Fuerza bruta» |
| Confirmar contraseña ignoraba el bloqueo | Confirmar y cambiar la contraseña respetan el bloqueo; confirmar tiene el límite del login | Ídem |
| Volver a atar una sesión ya atada | Una sesión se ata a la PC una sola vez; el segundo registro se rechaza y queda en el log | `USUARIOS_Y_PERMISOS.md`, DBSC |
| Gasto de la IA sin tope real | Largos máximos (10.000 caracteres el pedido) y tope diario por organización | `OBSERVABILIDAD.md`, «Tope de gasto diario» |
| Grafana sin usuario | El compose apaga el acceso sin usuario (ver punto 2) | `OBSERVABILIDAD.md`, sección 14 |
| Sitios que se atienden | `ALLOWED_HOSTS`, apagado hasta que se ponga en el `.env` (ver punto 1) | `PRODUCCION.md`, «Sitios que se atienden» |
| Dependencias del front | `npm audit fix`: sin avisos | `PRODUCCION.md`, «Publicar una versión nueva» |

Tres cosas quedaron distintas de lo planeado:

- **Cambiar la contraseña no entró al límite del login.** Ahí se cuentan también los intentos
  con una contraseña nueva que no cumple las reglas, y a los 5 dejaba al usuario esperando 15
  minutos. Lo frena el bloqueo: 5 contraseñas actuales mal y no se puede seguir probando.
- **El registro rechazado de DBSC queda en el log y no en el Registro de seguridad.** Sumarlo
  al registro pide una migración (la lista de hechos permitidos está en una restricción de la
  base). Queda para cuando haya otra migración de `audit_log`.
- **El PDF de la orden del seguro quedó como estaba.** La subida ya exige que empiece con
  `%PDF-` y todas las respuestas llevan `nosniff`. Sumarle `Content-Security-Policy: sandbox`
  puede hacer que Chrome no lo muestre.

## Al publicar `menu-usuario`

### 1. `.env` de productivo

- `ALLOWED_HOSTS=app.oemia.ar`. Antes, mirar en el panel de Cloudflare que la ruta del túnel no
  tenga otro valor en «HTTP Host Header». Si queda mal, todos los pedidos reciben 400 y el log
  dice qué sitio llegó: se corrige o se vacía, y se reinicia.
- `LLM_ORG_HARD_BUDGET_USD`: sin ponerlo vale 3 veces `LLM_DAILY_BUDGET_USD`. Con el diario en
  5, cada organización se queda sin IA al gastar 15 dólares en un día.
- `GRAFANA_ADMIN_PASSWORD`: solo hace falta en una instalación nueva de Grafana.

### 2. Grafana

Hoy, en esta PC, Grafana deja entrar como administrador sin usuario ni contraseña (comprobado
el 16 de noviembre: `http://127.0.0.1:3000/api/org` responde sin sesión). Solo escucha en la
PC, así que el riesgo es el mismo del punto 9: cualquier programa de la PC.

1. Con el `docker-compose.yml` nuevo en `C:\Oemia\Cotizador_BE`: `docker compose up -d`. Recrea
   el contenedor sin perder los datos.
2. Entrar a `http://127.0.0.1:3000`: ahora pide usuario. Probar `admin` / `admin` y cambiar la
   contraseña. Si no entra, ponerle una nueva con el comando de `OBSERVABILIDAD.md`, sección 14.

## Lo que hacés vos

### 3. Cloudflare: regla de límite para el login

En el panel, dominio `oemia.ar` → Security → WAF → Rate limiting rules → Create rule.

- Expresión:
  `(http.host eq "app.oemia.ar" and http.request.uri.path eq "/api/auth/login" and http.request.method eq "POST")`
- Contar por IP. El plan gratis trae una sola regla y deja elegir pocos períodos: usar el más
  corto que ofrezca, con 10 pedidos como máximo, y acción Block.
- Para qué sirve: frena las ráfagas antes de que lleguen a la PC. El límite fino (5 cada 15
  minutos) lo sigue poniendo el backend.

### 4. Cloudflare: revisar el panel

Lo que la revisión no pudo ver. Si entrás al panel en el navegador de Claude, lo puede mirar
con vos sin cambiar nada.

- La regla de país: `(http.host eq "app.oemia.ar" and ip.src.country ne "AR")` → Block, activa.
- El túnel: una sola ruta pública, `app.oemia.ar` → `http://127.0.0.1:8000`, sin rutas viejas y
  sin «HTTP Host Header» cambiado (punto 1).
- El Worker `oemia-mantenimiento`: rutas `app.oemia.ar/*`, y `/api/*` afuera.
- SSL/TLS en Full (strict), TLS mínimo 1.2 y «Always Use HTTPS» prendido.
- Tu cuenta de Cloudflare con verificación en dos pasos.
- API tokens: borrar los que no se usen.

### 5. Firewall de Windows

En PowerShell como administrador. Lista las reglas de entrada que dejan pasar los puertos de
productivo; lo esperado es que no salga ninguna.

```powershell
Get-NetFirewallRule -Direction Inbound -Enabled True -Action Allow | ForEach-Object { $p = $_ | Get-NetFirewallPortFilter; if ($p.LocalPort -match '^(8000|5432|3000|4318)$') { "{0} | {1} | {2}" -f $_.DisplayName, $p.Protocol, $p.LocalPort } }
```

Aunque saliera alguna, hoy todo escucha solo en `127.0.0.1`. Es la segunda barrera.

### 6. Deploy key de solo lectura

Hoy `C:\Oemia` baja el código con tus credenciales de GitHub, que también pueden subir. Una
llave por repositorio (GitHub no deja repetirla):

```powershell
ssh-keygen -t ed25519 -f C:\Oemia\.ssh\cotizador_be -C "oemia-prod-be"
```

```powershell
ssh-keygen -t ed25519 -f C:\Oemia\.ssh\cotizador_fe -C "oemia-prod-fe"
```

1. En GitHub, en cada repositorio: Settings → Deploy keys → Add deploy key. Pegar el `.pub` y
   dejar «Allow write access» sin tildar.
2. En cada carpeta de `C:\Oemia`, apuntar el remoto a SSH y fijar su llave:

   ```powershell
   git -C C:\Oemia\Cotizador_BE remote set-url origin git@github.com:DUEÑO/REPO_BE.git
   ```

   ```powershell
   git -C C:\Oemia\Cotizador_BE config core.sshCommand "ssh -i C:/Oemia/.ssh/cotizador_be -o IdentitiesOnly=yes"
   ```

3. Probar con `git -C C:\Oemia\Cotizador_BE fetch` y repetir para el front.

### 7. Respaldos cifrados y fuera de la PC

Windows 11 Home no trae BitLocker, así que se cifra cada respaldo. Con `age` la PC puede crear
respaldos pero no abrirlos: la clave para abrirlos vive fuera.

1. Instalar: `winget install FiloSottile.age`.
2. En otra PC o en el celular, generar el par con `age-keygen` y guardar la clave privada en
   el gestor de contraseñas. A la PC de productivo va solo la pública (`age1...`).
3. Después de cada `pg_dump -f respaldo.dump`:

   ```powershell
   age -r age1TU_CLAVE_PUBLICA -o respaldo.dump.age respaldo.dump
   ```

   y borrar `respaldo.dump`.
4. Copiar el `.age` fuera de la PC (disco externo o nube).
5. Hacer lo mismo con `C:\Oemia\respaldos\cotizador_antes_de_0012.dump`, que hoy está sin
   cifrar.

Claude puede dejar un script `respaldar.ps1` que haga los pasos 3 y 4 de una vez.

## Para después, con el cómo

### 8. Row Level Security en la telemetría

**Qué falta.** Las tablas de `observability` que tienen `organization_id` (`request_log`,
`dependency_events`, `llm_calls`, `quote_runs`, `quote_item_feedback`, `frontend_events`) no
tienen políticas. Hoy solo las lee la plataforma desde Monitoreo, así que es una segunda
barrera y no un agujero abierto: empieza a importar el día que una pantalla de un
concesionario lea de ahí (por ejemplo, «mis cotizaciones» o «mi gasto de IA»).

**Cómo hacerlo.**

1. **Quién escribe y quién lee, primero.** Hoy el escritor de telemetría y
   `ObservabilityRepository` usan `session_factory`, el usuario de la app, sin organización
   puesta en la sesión. Con políticas, sus `INSERT` fallarían. Los dos pasan a
   `system_session_factory` (el usuario `cotizador_sistema`, que se saltea RLS), en `main.py`:
   `start_observability(system_session_factory, owner_session_factory, ...)` y
   `ObservabilityRepository(system_session_factory)`. También `usage_tracker.load_today`, que
   suma el gasto de todas las organizaciones.
2. **Buscar cualquier otra lectura o escritura** de esas tablas con el usuario de la app:
   `grep -rn "RequestLogRecord\|LLMCallRecord\|QuoteRunRecord\|FrontendEventRecord\|DependencyEventRecord" app`.
   Lo que escriba con la sesión del pedido tiene que tener la organización puesta
   (`app/database/tenant.py`) o pasar al usuario del sistema.
3. **Una migración nueva** (la `0019` ya es la de los alias de las bieletas), copiando la forma de `0007_row_level_security.py`: en cada una de las
   seis tablas, `ENABLE ROW LEVEL SECURITY` y `FORCE ROW LEVEL SECURITY`, y una política para
   `cotizador_app` con `organization_id = current_setting('app.organization_id')::bigint` en
   `USING` y en `WITH CHECK`. Las filas sin organización (pedidos sin sesión) quedan solo para
   el usuario del sistema. Las tablas están particionadas por mes: la política va en la tabla
   madre y vale para las particiones, pero hay que probarlo con una partición nueva
   (`app/database/partitions.py`).
4. **Tests.** Sumar las seis tablas a `tests/integration/test_row_level_security.py` (una
   organización no ve las filas de otra, y no puede insertar con el `organization_id` de otra)
   y correr la suite entera antes y después: el riesgo es una escritura olvidada que empiece
   a fallar en silencio, porque el escritor de telemetría descarta lo que no puede guardar y
   solo lo cuenta en `registros_perdidos`.
5. **Al publicar**, mirar esa alerta (`registros_perdidos`) el primer día.

Es medio día de trabajo. Lo hace Claude cuando se decida.

### 9. Productivo separado de tu usuario de Windows

**Qué falta.** Es el riesgo más grande que queda: cualquier programa que corra con tu usuario
(un paquete de npm, una extensión, una herramienta) lee `C:\Oemia\Cotizador_BE\.env`, con la
clave de Anthropic, `CREDENTIALS_KEY` y las tres contraseñas de Postgres; puede hablarle al
backend en `127.0.0.1:8000` inventando la IP de origen; y, hasta el punto 2, entra a Grafana.

**La solución de fondo es la PC dedicada**, ya acordada el 9 de noviembre. Al pasar a esa PC:

- Clave de Anthropic nueva, y la vieja revocada en la consola de Anthropic.
- Contraseñas de Postgres nuevas (las genera `scripts/preparar_postgres.py`).
- `CREDENTIALS_KEY`: si se genera una nueva, hay que volver a cargar las cuentas de EPC, SAP y
  el localizador y volver a configurar los códigos de verificación. Si se lleva la actual, va
  por un medio seguro y se borra de esta PC.
- En esa PC no se instala nada más: ni Node de desarrollo, ni editores, ni navegador de uso
  diario. Ahí también van los puntos 6 y 7.

**Si la PC dedicada se demora, en esta PC**, en PowerShell como administrador:

1. Crear un usuario local sin privilegios, con una contraseña que no se use en otro lado:

   ```powershell
   New-LocalUser -Name oemia -Password (Read-Host -AsSecureString "Contraseña") -PasswordNeverExpires -Description "Corre OEMIA productivo"
   ```

2. Darle permiso para correr tareas programadas: `secpol.msc` no existe en Windows Home, así
   que se hace al crear la tarea (paso 5), que lo otorga sola al guardar la contraseña.
3. Parar el backend y darle la carpeta a ese usuario, sacándosela al tuyo. Los administradores
   siguen entrando, así que vas a poder editarla desde una consola de administrador:

   ```powershell
   icacls C:\Oemia /inheritance:r /grant "Administradores:(OI)(CI)F" "SYSTEM:(OI)(CI)F" "oemia:(OI)(CI)M"
   ```

4. Entrar una vez a Windows con `oemia` y preparar lo que se guarda por usuario:
   - `uv sync --frozen --extra otel` en `C:\Oemia\Cotizador_BE` (Python y el caché de uv).
   - `uv run playwright install chromium` (los navegadores de EPC y SAP).
   - `EPC_SESSION_DIR` del `.env` apunta a `~/.epc_session`: con otro usuario es otra carpeta,
     así que la primera consulta vuelve a loguear el EPC.
   - Git: `git config --global --add safe.directory C:/Oemia/Cotizador_BE` (y el del front), y
     la deploy key del punto 6, porque tus credenciales de GitHub no están en ese usuario.
5. Cambiar la tarea «OEMIA backend productivo» para que corra como `oemia` («Ejecutar tanto si
   el usuario inició sesión como si no», con su contraseña). Los pasos de la tarea están en
   `PRODUCCION.md`, «La tarea programada».
6. `cloudflared` y Postgres no cambian: ya corren como servicios, con sus propios usuarios.
7. Probar: `https://app.oemia.ar/api/salud`, una cotización con EPC y SAP, y que desde tu
   usuario `Get-Content C:\Oemia\Cotizador_BE\.env` diga «Acceso denegado».

Lo que no arregla: un programa con permisos de administrador entra igual, y tu usuario es
administrador de la PC si acepta los carteles de UAC. Por eso es un paliativo y no reemplaza a
la PC dedicada. Es medio día de trabajo, y los pasos 4 y 5 se repiten en la PC dedicada.

### 10. Lo que faltó revisar

- **EPC, SAP, localizador y matcher, línea por línea.** A buscar: texto del cliente que termine
  dando órdenes a la IA, URLs armadas con datos del usuario, y qué se guarda de las sesiones
  de Playwright. Lo hace Claude; sale un informe corto con hallazgos o sin ellos.
- **Base de productivo.** Dos consultas de solo lectura con el usuario dueño, para confirmar
  que productivo coincide con las migraciones:

  ```sql
  SELECT rolname, rolsuper, rolbypassrls, rolcreatedb, rolcreaterole
  FROM pg_roles WHERE rolname LIKE 'cotizador%';

  SELECT n.nspname, c.relname, c.relrowsecurity, c.relforcerowsecurity
  FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE n.nspname IN ('app', 'observability') AND c.relkind IN ('r', 'p')
  ORDER BY 1, 2;
  ```

## Lo que queda como está

- **Bloquear a propósito a un usuario conocido** (5 intentos cada 15 minutos con `f.apellido`).
  Es el costo de tener bloqueo. Lo acotan la regla del punto 3 y la alerta de ingresos
  fallidos que ya existe.
- **Un programa de la misma PC puede inventar `CF-Connecting-IP`** y saltear el límite por IP.
  No tiene arreglo dentro de la misma PC; se va con el punto 9.

## Dependencias

Revisadas el 16 de noviembre de 2026.

- **Backend:** `uv audit --frozen` sobre `uv.lock`: 76 paquetes, ninguna vulnerabilidad
  conocida. El comando es experimental en uv 0.13.
- **Front:** `npm audit` daba 9 avisos moderados por `minimatch` (GHSA-g3hv-75wg-m249), que
  entra por `eslint` y `typescript-eslint`: herramientas de desarrollo que no van en lo que se
  publica. Arreglado con `npm audit fix`; hoy no da ninguno.
