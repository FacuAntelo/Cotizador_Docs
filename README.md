# Cotizador de Repuestos Nissan

Dado un texto libre del cliente (jerga argentina, abreviaturas, etc.), identifica los repuestos solicitados y consulta el EPC oficial de Nissan para obtener los números de pieza completos.

## Endpoints disponibles

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/match` | Texto libre → repuestos identificados + familias |
| POST | `/epc` | Chasis + familias → números de pieza completos |
| POST | `/cotizar` | **Endpoint principal:** texto libre + chasis → números de pieza |
| PUT | `/dms/stock` | El agente del taller sube la foto completa del stock (con token) |
| POST | `/dms/stock` | Números de pieza → stock del taller por depósito |
| GET | `/dms/stock/estado` | Cuándo llegó la última foto del stock del taller |
| GET | `/presupuestos/siguiente-numero` | Número que le toca al próximo presupuesto |
| GET | `/presupuestos` | Presupuestos guardados (`buscar` por número, cliente, teléfono, vehículo, patente o VIN) |
| GET | `/presupuestos/{numero}` | Un presupuesto guardado, con sus renglones |
| POST | `/presupuestos` | Guardar un presupuesto nuevo (sin número toma el siguiente; 409 si el número existe) |
| PUT | `/presupuestos/{numero}` | Actualizar un presupuesto guardado |
| GET / PUT | `/presupuestos/concesionario` | Datos y logo del concesionario que encabezan los presupuestos |
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

Ver documentación interactiva en http://localhost:8000/docs

## Requisitos

- Python 3.11+
- Node.js 18+
- [uv](https://docs.astral.sh/uv/) (gestor de paquetes Python)
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
EPC_USERNAME=tu@email.com
EPC_PASSWORD=...
EPC_URL=https://login.superservice.com/login/es-es/
EPC_SESSION_DIR=~/.epc_session
EPC_HEADFUL=false
DATABASE_URL=sqlite:///data/cotizador.db   # opcional: es el valor por defecto
```

```bash
# Levantar servidor (al arrancar crea o actualiza la base en data/cotizador.db)
uv run uvicorn main:app --reload
# → http://localhost:8000
```

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

# Tests
uv run pytest -v

# Migración nueva después de cambiar un modelo de app/models/
uv run alembic revision --autogenerate -m "descripcion"
```

Commits en formato convencional: `feat(cotizar): descripción`, `fix(epc): ...`, etc.
