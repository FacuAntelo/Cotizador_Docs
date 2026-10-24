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
```

```bash
# Levantar servidor
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
│   │   ├── routers/         ← endpoints HTTP (match, epc, cotizar)
│   │   ├── schemas/         ← modelos Pydantic request/response
│   │   └── services/        ← lógica de negocio (Claude, Playwright, orquestación)
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
uv run pytest tests/unit -v
```

Commits en formato convencional: `feat(cotizar): descripción`, `fix(epc): ...`, etc.
