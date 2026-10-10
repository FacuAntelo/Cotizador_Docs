# Cotizador de Repuestos Nissan — Contexto del Proyecto

## Descripción general

Sistema para cotizar repuestos Nissan. Dado un texto libre del cliente (con jerga argentina, abreviaturas, etc.), identifica los repuestos solicitados y obtiene los números de pieza correctos consultando el catálogo EPC oficial de Nissan.

---

## Stack tecnológico

- **Backend:** Python 3.11+ + FastAPI, gestión de deps con `uv`
- **Frontend:** React + Vite (`frontend/src/App.jsx`)
- **LLM:** Claude API (claude-sonnet-4-20250514) vía Anthropic SDK
- **Automatización browser:** Playwright (sesión persistida en `~/.epc_session`)
- **Linting:** Ruff
- **Puerto backend:** 8000 | **Puerto frontend:** 5173

---

## Lo que ya está implementado

### 1. Matcher de repuestos — `POST /match`

Recibe texto libre y devuelve los repuestos identificados con sus familias:

```json
// Request
{ "texto": "pdi, fender dd, optica izquierda" }

// Response
[
  { "input": "pdi", "nombre": "Puerta Delantera Izquierda", "familias": ["80101", "H0101"], "comentario": null },
  { "input": "fender dd", "nombre": "Moldura de Guardabarros Delantera Derecha", "familias": ["63810"], "comentario": null },
  { "input": "optica izquierda", "nombre": "Óptica Delantera Izquierda", "familias": ["26060"], "comentario": null }
]
```

Claude entiende jerga argentina: pdi/pdd/pti/ptd, fender (=moldura plástica, NO el guardabarros metálico), optica, sapito, blower, di/dd/ti/td como sufijo, etc.

### 2. EPC Scraper — `POST /epc`

Consulta el EPC de Nissan en superservice.com usando Playwright.

**Flujo confirmado:**
1. Login en `https://login.superservice.com/login/es-es/`
2. Click en `<button><span>Iniciar</span></button>` → abre nueva pestaña Microcat EPC
3. Chasis → `#genericSearchInput` + Enter
4. Familia (5 chars) → `#globalSearchInput` + Enter
5. Resultados en ag-grid → `span#copyablePartNumberValue` (formato `16546-5RA0A`)
6. Se normaliza quitando guión: `165465RA0A`

**Grupos de familias equivalentes:** `80101` y `H0101` son el mismo repuesto en distintos catálogos. Si 80101 devuelve resultado, H0101 no se consulta.

```json
// Request
{ "chasis": "94DFCAP15MB110619", "grupos": [["80101", "H0101"], ["26060"]] }

// Response
[
  { "familia": "80101", "numeros": ["8010100Q0A"] },
  { "familia": "26060", "numeros": ["260601KA0A"] }
]
```

### 3. EPC Session y Estado — `POST /epc/connect` y `GET /epc/session`

Permite iniciar sesión manualmente de forma preventiva y obtener el estado visual de la sesión (tiempos restantes máximo y por inactividad). El Scraper mantiene internamente contadores `_session_start_time` y `_last_activity_time` para reportar estos datos al frontend.

### 4. Cotizador unificado — `POST /cotizar`

Endpoint principal. Una sola llamada desde texto libre a números de pieza completos.

**Flujo interno:**
1. `MatcherService.match(texto)` → identifica repuestos con sus familias equivalentes
2. `EPCScraper.search(chasis, grupos)` → consulta EPC solo para ítems con familias (las familias de cada ítem ya forman un grupo equivalente)
3. `CotizarService` correlaciona y fusiona ambos resultados

```json
// Request
{ "texto": "pdi, fender dd", "chasis": "94DFCAP15MB110619" }

// Response
[
  {
    "input": "pdi",
    "nombre": "Puerta Delantera Izquierda",
    "familia": "80101",
    "numeros": ["8010100Q0A"],
    "comentario": null
  },
  {
    "input": "fender dd",
    "nombre": "Moldura de Guardabarros Delantera Derecha",
    "familia": "63810",
    "numeros": ["638103YF0A"],
    "comentario": null
  }
]
```

Los repuestos no encontrados en el catálogo se incluyen con `nombre=null`, `familia=null`, `numeros=[]`.

### 5. Catálogo normalizado (143 repuestos)

Convención: `[Nombre funcional] [Superior/Inferior] [Posición] [Lado]`

Cada repuesto tiene: familias (códigos de 5 chars) y aliases (jerga/sinónimos). Vive en `app/services/matcher_service.py` como constante `_CATALOG_CONTEXT`.

---

## Estructura del proyecto

```
matcher-nissan/
├── CONTEXT.md
├── STANDARDS.md
├── backend/
│   ├── app/
│   │   ├── core/
│   │   │   └── config.py              # pydantic-settings, lee .env
│   │   ├── routers/
│   │   │   ├── match.py               # POST /match
│   │   │   ├── epc.py                 # POST /epc  (singleton scraper aquí)
│   │   │   └── cotizar.py             # POST /cotizar
│   │   ├── schemas/
│   │   │   ├── match.py               # MatchRequest, MatchResult
│   │   │   ├── epc.py                 # EPCRequest, EPCResult
│   │   │   └── cotizar.py             # CotizarRequest, CotizarResult
│   │   └── services/
│   │       ├── matcher_service.py     # lógica Claude + catálogo
│   │       ├── epc_scraper.py         # lógica Playwright
│   │       └── cotizar_service.py     # orquesta matcher + scraper
│   ├── .env                           # NO commitear
│   ├── .env.example                   # sí commitear
│   ├── pyproject.toml                 # deps + config ruff
│   └── main.py                        # FastAPI app + routers + lifespan
└── frontend/
    └── src/App.jsx
```

---

## Cómo correr el proyecto

**Backend:**
```bash
cd backend
pip install uv          # solo la primera vez
uv sync                 # instala todas las deps desde pyproject.toml
# Copiar .env.example → .env y completar valores
uv run uvicorn main:app --reload
```

**Frontend:**
```bash
cd frontend
npm install
npm run dev
```

**Linting (antes de cada commit):**
```bash
uv run ruff check . --fix
uv run ruff format .
```

---

## Variables de entorno (.env)

```
ANTHROPIC_API_KEY=sk-ant-...
EPC_USERNAME=facundo@antelo.com.ar
EPC_PASSWORD=...
EPC_URL=https://login.superservice.com/login/es-es/
EPC_SESSION_DIR=~/.epc_session
EPC_HEADFUL=false          # true para ver el browser (debug)
```

---

## Modelo de datos (diseñado, pendiente de implementar en DB)

### Entidades principales

**spare_part** — Repuesto como concepto genérico
- id, name (normalizado), category, notes

**part_family** — Familias del catálogo (primeros 5 chars)
- id, spare_part_id, family_code (varchar 5)

**spare_part_alias** — Sinónimos y jerga por repuesto
- id, spare_part_id, alias

**part_number** — Número de pieza completo (10 chars)
- id, part_family_id, full_number (varchar 10)
- source: manual | epc_confirmed
- notes

**vehicle_model** — Modelos (Nissan Note, Frontier, etc.)
- id, brand, model

**vehicle_version** — Versiones dentro de un modelo
- id, vehicle_model_id, version_name, year_from, year_to

**applicability** — Qué número de pieza va en qué vehículo
- id, spare_part_id, part_number_id
- vehicle_version_id (null = aplica a todas las versiones del modelo)
- confidence: confirmed | assumed | pending

**spare_part_spec** — Especificaciones clave-valor por repuesto
- id, spare_part_id, spec_key, spec_value

**part_composition** — Relación repuesto compuesto / componente
- id, parent_part_id, child_part_id, vehicle_version_id
- availability: only_together | also_separate

**spare_part_alternative** — Equivalente funcional si no existe para ese modelo
- id, spare_part_id, alternative_spare_part_id

---

## Decisiones de diseño importantes

1. **El LLM hace toda la normalización** — el catálogo completo va como contexto y Claude resuelve jerga, abreviaturas, lado dominante. Sin diccionarios de expansión.

2. **Familias equivalentes agrupadas** — las familias de un ítem del `/match` (ej: `["80101", "H0101"]`) ya forman el grupo equivalente para el EPC. Si la primera devuelve resultado, las demás no se consultan.

3. **Sesión Playwright persistida** — la sesión EPC se guarda en disco (`~/.epc_session`). Solo hace login la primera vez.

4. **Singleton scraper compartido** — el `EPCScraper` vive en `app/routers/epc.py` y es importado por `app/routers/cotizar.py` para garantizar que ambos endpoints usen la misma instancia y la misma sesión de browser.

5. **Dos modos de operación:**
   - Sin chasis: applicability desde BD (en el futuro)
   - Con chasis: consulta EPC para número exacto según versión

6. **Cache de resultados EPC** — cuando el EPC confirma un número, guardarlo en applicability con confidence = confirmed. La BD se enriquece sola.

---

## Próximos pasos en orden

1. **Base de datos** (SQLite/SQLAlchemy) con el modelo diseñado
2. **Cache de resultados EPC** en tabla `applicability`
3. **Tests unitarios** para `MatcherService` y `CotizarService` con mocks
4. **Pre-commit hooks** (ruff + pytest)
5. **Detección de repuestos en conjunto** via `part_composition`
6. **Equivalentes** via `spare_part_alternative`
