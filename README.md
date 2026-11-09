# Cotizador de Repuestos Nissan

Dado un texto libre del cliente (jerga argentina, abreviaturas, etc.), identifica los repuestos solicitados y consulta el EPC oficial de Nissan para obtener los números de pieza completos.

## Endpoints disponibles

Todas piden login salvo las marcadas como públicas; cada una exige su permiso (ver
[USUARIOS_Y_PERMISOS.md](USUARIOS_Y_PERMISOS.md)). Las escrituras llevan el header
`X-OEMIA: 1` del front.

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/auth/login` | Pública. Usuario, contraseña y código de verificación → sesión en esta PC |
| POST | `/auth/salir` | Cerrar la sesión y olvidar esta PC |
| GET | `/auth/me` | Usuario, organización, sucursales, permisos y si otra PC tomó el usuario |
| POST | `/auth/renovar` | Pública. Renueva la cookie de la sesión en navegadores sin DBSC |
| POST | `/auth/dbsc/registro` · `/auth/dbsc/renovar` | Públicas. Las llama Chrome solo para atar la sesión a la PC |
| POST | `/auth/usar-aca` | Retomar el usuario en esta PC (la otra queda afuera) |
| POST | `/auth/confirmar-contrasena` | Volver a escribir la contraseña antes de algo sensible |
| PUT | `/auth/contrasena` | Cambiar la contraseña (cierra las demás PCs) |
| POST / DELETE | `/auth/codigo-verificacion` | Configurar o quitar el código de verificación (`/activar` lo activa) |
| GET / DELETE | `/auth/sesiones` | Mis dispositivos: las PCs recordadas y cerrar una |
| POST | `/match` | Texto libre → repuestos identificados + familias |
| POST | `/epc` | Chasis + familias → números de pieza completos |
| POST | `/cotizar` | **Endpoint principal:** texto libre + chasis → números de pieza |
| PUT | `/dms/stock` | Pública. El agente de una sucursal sube la foto completa de su stock (con su token) |
| POST | `/dms/stock` | Números de pieza → stock de la sucursal activa, de las demás y de cada una |
| GET | `/dms/stock/estado` | Cuándo llegó la última foto del stock de la sucursal activa |
| GET | `/presupuestos/siguiente-numero` | Número que le toca al próximo presupuesto |
| GET | `/presupuestos` | Presupuestos de toda la organización (`buscar` por número, cliente, teléfono, vehículo, patente o VIN; `sucursal` filtra) |
| GET | `/presupuestos/{numero}` | Un presupuesto guardado, con sus renglones |
| POST | `/presupuestos` | Guardar un presupuesto nuevo (sin número toma el siguiente; 409 si el número existe) |
| PUT | `/presupuestos/{numero}` | Actualizar un presupuesto guardado |
| GET / PUT | `/presupuestos/concesionario` | Datos de la organización y de la sucursal que encabezan los presupuestos (el PUT cambia los de la sucursal) |
| GET / POST | `/aseguradoras` | Compañías de seguro para el presupuesto (agregar no repite mayúsculas) |
| POST | `/presupuestos/{numero}/aceptado` | Aceptar el presupuesto guardado para habilitar pedidos, con las respuestas del cuestionario del seguro |
| GET / PUT | `/presupuestos/{numero}/aceptacion` | Franquicia, inspección final, pago del seguro y excluidos |
| POST / GET / DELETE | `/presupuestos/{numero}/orden-seguro` | Subir (PDF, reemplaza), ver o quitar la orden del seguro |
| GET | `/presupuestos/{numero}/pedidos/vista-previa` | Proponer compatibles por costo Nissan, planta y stock del taller, con el motivo de cada descarte |
| POST | `/presupuestos/{numero}/pedidos` | Confirmar el Excel y dejar para separar lo del stock del taller (null si no hay nada para pedir) |
| GET | `/presupuestos/{numero}/pedidos/{pedido_id}/excel` | Descargar el Excel de número de pieza y cantidad |
| POST | `/presupuestos/{numero}/pedidos/{pedido_id}/registrar` | Registrar fecha y número asignado por Nissan |
| POST | `/presupuestos/{numero}/pedidos/{pedido_id}/anular` | Anular un Excel que no se cargó en Nissan |
| GET | `/presupuestos/{numero}/abastecimiento` | Estados por ítem (pedido, para separar, separado), excluidos e historial |
| POST | `/presupuestos/{numero}/abastecimiento` | Pasar lo que falta de un ítem directo a pedido, para separar o separado |
| POST | `/presupuestos/{numero}/abastecimiento/{registro_id}/estado` | Cambiar el estado conservando origen, Nº de pedido y costo |
| POST | `/presupuestos/{numero}/abastecimiento/{registro_id}/anular` | Volver a «Sin pedir» sin borrar el historial |
| GET | `/prestamos` | Resumen y registros de préstamos de repuestos |
| POST | `/prestamos` | Crear un préstamo o pedido; cada número de pieza se guarda como una fila |
| PATCH | `/prestamos/{id}` | Actualizar el estado y la fecha de fin de un registro |
| GET | `/salud` · `/salud/detalle` | Salud del sistema (`/salud` es pública; el detalle, con `monitoreo.ver`) |
| GET | `/monitoreo/*` | Pantalla Monitoreo y rastreo de un código de error, con `monitoreo.ver` (ver OBSERVABILIDAD.md) |
| POST | `/telemetria` | Mediciones del navegador (Web Vitals, errores, tiempos) |

Ver documentación interactiva en http://127.0.0.1:8001/docs (solo en desarrollo; en productivo
está apagada)

## Requisitos

- Python 3.15 (si no está instalado, `uv sync` lo descarga)
- Node.js 18+
- [uv](https://docs.astral.sh/uv/) (gestor de paquetes Python)
- PostgreSQL 18 (`winget install PostgreSQL.PostgreSQL.18`), con `C:\Program Files\PostgreSQL\18\bin` en el PATH para usar `psql` y `pg_dump`
- Docker Desktop, para Grafana (trazas; en desarrollo es opcional)
- Credenciales de Anthropic y superservice.com

## Setup

### 1. Backend

```bash
cd backend

# Instalar uv (solo la primera vez)
pip install uv

# Instalar dependencias
uv sync

# Instalar browser de Playwright (solo la primera vez)
uv run playwright install chromium

# Configurar variables de entorno
copy .env.example .env
# Editar .env con tus credenciales
```

**Variables en `.env`:**
```
ANTHROPIC_API_KEY=sk-ant-...
CREDENTIALS_KEY=...
EPC_URL=https://login.superservice.com/login/es-es/
EPC_SESSION_DIR=~/.epc_session
EPC_HEADFUL=false
DATABASE_URL=postgresql+psycopg://cotizador_dev_app:cotizador_dev_app@127.0.0.1:5432/cotizador_dev
SYSTEM_DATABASE_URL=postgresql+psycopg://cotizador_dev_sistema:cotizador_dev_sistema@127.0.0.1:5432/cotizador_dev
MIGRATIONS_DATABASE_URL=postgresql+psycopg://cotizador_dev:cotizador_dev@127.0.0.1:5432/cotizador_dev
```

Una sola vez, el dueño y la base de desarrollo. Pide la contraseña de `postgres`; el
usuario `cotizador_dev` puede crear bases para que los tests armen la suya:

```powershell
psql -h 127.0.0.1 -U postgres -c "CREATE ROLE cotizador_dev LOGIN CREATEDB PASSWORD 'cotizador_dev'" -c "CREATE DATABASE cotizador_dev OWNER cotizador_dev LOCALE_PROVIDER icu ICU_LOCALE 'es-AR' TEMPLATE template0"
```

Y los usuarios de la app (con Row Level Security) y del sistema, que también usan los tests.
Pide la contraseña de `postgres` y escribe las tres URLs en el `.env`:

```powershell
uv run python scripts/preparar_postgres.py desarrollo
```

Postgres corre como servicio de Windows (`postgresql-x64-18`) y arranca solo con la PC.

Grafana corre en Docker y queda siempre prendido: se levanta una vez con
`docker compose up -d` y después vuelve solo con Docker Desktop. Ver
[OBSERVABILIDAD.md](OBSERVABILIDAD.md#14-trazas-con-opentelemetry-opcional).

`CREDENTIALS_KEY` cifra los secretos de la base (los códigos de verificación y las cuentas
de EPC, SAP y el localizador). Se genera una por PC y no se comparte:

```bash
uv run python -m app.cli generar-clave
```

El administrador de plataforma se crea solo desde la consola de la PC del servidor. Pide
la contraseña ahí mismo y configura el código de verificación con el celular:

```bash
uv run python -m app.cli crear-admin
```

Si un administrador de plataforma cambia o pierde el celular, se le configura un código nuevo
desde la misma consola (le cierra todas las sesiones):

```bash
uv run python -m app.cli configurar-codigo
```

```bash
# Levantar servidor (al arrancar crea o actualiza las tablas). La API queda bajo /api.
uv run --extra otel python -m app.server
# → http://127.0.0.1:8001/api/salud (en desarrollo; productivo usa el 8000)
# --recargar: se reinicia al guardar un cambio (vuelve a loguear EPC y SAP).
```

O el backend y el front juntos, cada uno en su ventana: `.\scripts\desarrollo.ps1`.
En productivo: [PRODUCCION.md](PRODUCCION.md).

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
# → http://localhost:5173
```

## Estructura

```
matcher-nissan/
├── CONTEXT.md               ← contexto del proyecto para el LLM
├── STANDARDS.md             ← estándares de desarrollo
├── INTEGRACIONES.md         ← cómo funcionan EPC, SAP y el DMS por dentro (hallazgos técnicos)
├── OBSERVABILIDAD.md        ← logs, métricas, costos de la IA, límites, alertas y perfiles
├── LOGIN_Y_OBSERVABILIDAD.md ← qué sumar a la observabilidad cuando exista el login
├── backend/
│   ├── app/
│   │   ├── core/config.py   ← configuración (pydantic-settings)
│   │   ├── database/        ← acceso a la base (engine, repositorios, migraciones)
│   │   ├── models/          ← tablas (SQLAlchemy)
│   │   ├── routers/         ← endpoints HTTP (match, epc, cotizar)
│   │   ├── schemas/         ← modelos Pydantic request/response
│   │   └── services/        ← lógica de negocio (Claude, Playwright, orquestación)
│   ├── migrations/          ← migraciones de Alembic
│   ├── alembic.ini
│   ├── .env.example
│   ├── pyproject.toml
│   └── main.py
└── frontend/
    └── src/App.jsx
```

## Desarrollo

```bash
# Linting y formato (antes de commitear)
uv run ruff check . --fix
uv run ruff format .

# Tests: crean y vacían su propia base (cotizador_test), nunca la de desarrollo
uv run pytest -v

# Migración nueva después de cambiar un modelo de app/models/
uv run alembic revision --autogenerate -m "descripcion"

# Consola de la base. Las tablas están en los schemas app y observability:
# app.estimates, o antes `SET search_path TO app, observability;`
psql -h 127.0.0.1 -U cotizador_dev cotizador_dev

# Respaldo: lleva la estructura de la telemetría, no sus datos (requests y demás), que se
# pueden perder y ocupan la mayor parte. Con -f y no con >: en PowerShell, > rompe el archivo.
pg_dump -h 127.0.0.1 -U cotizador_dev -Fc --exclude-table-data="observability.*" -f respaldo.dump cotizador_dev
```

Commits en formato convencional: `feat(cotizar): descripción`, `fix(epc): ...`, etc.
