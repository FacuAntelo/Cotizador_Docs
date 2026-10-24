# Cotizador de Repuestos Nissan — Contexto del Proyecto

## Descripción general

Sistema para cotizar repuestos Nissan. Dado un texto libre del cliente (con jerga argentina, abreviaturas, etc.), identifica los repuestos solicitados y obtiene los números de pieza correctos consultando el catálogo EPC oficial de Nissan.

---

## Stack tecnológico

- **Backend:** Python 3.11+ + FastAPI, gestión de deps con `uv`
- **Frontend:** React + TypeScript + Vite, SPA con TanStack Query y tipos generados del OpenAPI (ver `Cotizador_FE/README.md`)
- **LLM:** Claude API (claude-haiku-5-5) vía Anthropic SDK
- **Automatización browser:** Playwright, sólo para los logins (EPC y SAP); los datos se piden a las APIs
- **Base de datos:** SQLite (`data/cotizador.db`) con SQLAlchemy 2 y migraciones de Alembic; Postgres más adelante (ver [Base de datos](#base-de-datos))
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

> Detalle técnico completo de EPC y SAP (endpoints, licencia, cookies, tiempos medidos): ver `INTEGRACIONES.md`.

Consulta el EPC de Nissan (Microcat, superservice.com). Esquema mixto:
Playwright sólo hace el login y deja abierta la pestaña del EPC; los datos se
piden a la API interna del EPC con `fetch()` desde esa pestaña, reutilizando los
headers que manda la propia app (`authorization`, `x-ifm-session-id`, `x-ifm-sid`, …).

**Flujo:**
1. Login en `https://login.superservice.com/login/es-es/` → click en `Iniciar` → abre la pestaña del EPC
2. Se capturan los headers del primer pedido autenticado de la app y se verifica la licencia (`POST /v1/sessions/heartbeat`)
3. Vehículo → `GET /v3/vehicle/search?searchTerm=<VIN o últimos 8>` → `id` interno (cacheado por sesión)
4. Familia → `GET /v1/globalsearch/search?q=<familia>&vehicleId=<id>` → números (formato `16546-5RA0A`)
5. Se normaliza quitando guión (`165465RA0A`) y se descartan piezas de otras familias

**Rendimiento medido (8 ítems / 17 familias):** UI por Playwright 31–63 s → API 3,3 s.
Hasta 6 pedidos en paralelo; el EPC no los toma como "otro dispositivo".

**Licencia:** es nominal (un dispositivo a la vez). Si otro dispositivo la toma, la API
responde 403 con el header `x-ifm-licensing-error` y se recupera con
`POST /v1/sessions/device?source=use_license` (lo mismo que el botón "Usar licencia").
El EPC impone ~60 s de espera (`Retry-After`) tras cada cambio de dispositivo, incluso
a un login nuevo: en ese lapso la consulta falla con un mensaje que indica los segundos.

**Sesión:** la app manda un heartbeat cada 90 s mientras la pestaña está abierta y el
token dura ~20 h. Medido: tras 34 min sin consultas la sesión seguía viva (consulta en
1,75 s, sin re-login). El backend libera la sesión tras `SESSION_KEEPALIVE_HOURS` sin uso.

**Grupos de familias equivalentes:** `80101` y `H0101` son el mismo repuesto en distintos catálogos. Se consultan en paralelo y gana la primera, en orden, que devuelve resultado.

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

Permite forzar un login nuevo y obtener el estado de la sesión: `time_left_seconds` es lo que le queda al token del EPC (se lee del JWT) e `inactivity_left_seconds` lo que falta para que el backend libere la sesión por falta de uso. Al arrancar el backend la sesión se abre sola en segundo plano (`WARMUP_ON_START`).

### 3b. Stock SAP — `POST /sap/stock`

Playwright hace el login SAML (IAS) y corta apenas el portal emite la sesión (`/login/callback`); el stock se pide con httpx a un endpoint CPI del launchpad. Las cookies se guardan en `SAP_SESSION_FILE`, así un reinicio del backend no re-loguea, y con la sesión de IAS viva el re-login no pasa por el formulario (SSO). Un keep-alive cada 5 min (`GET /cp.portal/portal/rest/v1/personalization`, 200 con sesión / 401 sin ella) la mantiene viva mientras haya uso reciente. Medido: la sesión del portal sobrevive 25 min sin uso y vence antes de los 35; a los 35 min la sesión de IAS seguía viva y el re-login fue silencioso. Las consultas corren en paralelo; sólo el login se serializa.

| Escenario | Tiempo |
|---|---|
| Login completo + consulta | ~14 s |
| Reinicio con cookies vigentes | ~2 s |
| Portal vencido, IAS vivo (SSO) | ~6,5 s |
| Consulta con sesión abierta | ~1,3–2 s (8 en paralelo ≈ 2,2 s) |

### 3c. Stock del taller — `PUT /dms/stock`, `POST /dms/stock` y `GET /dms/stock/estado`

> Detalle del DMS (servidor, fórmula del stock, por qué un agente): ver `INTEGRACIONES.md`.

El stock propio del taller vive en el DMS (AutoDealer, SQL Server en la red del taller), al que el backend en la nube no llega. Un agente en el servidor del taller (`Cotizador_BE/agente_dms/`) calcula el stock cada pocos minutos y sube la foto completa con `PUT /dms/stock` y `Authorization: Bearer <DMS_AGENT_TOKEN>`; la foto reemplaza a la anterior y se guarda en la base (tablas `dms_stock` y `dms_stock_uploads`). `POST /dms/stock` responde como `/sap/stock` (`{"materiales": [...]}`), con el total y el detalle por depósito, comparando códigos sin guiones. Si la foto tiene más de `DMS_STALE_MINUTES`, la respuesta trae `desactualizado: true`. Se guardan todos los depósitos con su ubicación (si el agente la manda), pero los de `DMS_EXCLUDED_DEPOSITS` (por defecto `GARANTIAS,PICKING`) vienen con `incluido_por_defecto: false` y no suman a `disponible`.

En el front, la columna **Stock** de la tabla (y el bloque "Stock" de las sugerencias del asesor) muestra las unidades de cada N° de pieza. El número y la **(i)** abren un popover con cada depósito: unidades, ubicación y una casilla para sumarlo o no. La consulta incluye el padrón completo enviado por el agente desde `dbo.Depositos.Descripcion`, guardado en `dms_deposits`, con cero donde no hay existencias. Primero aparecen los depósitos con stock y luego los demás en orden alfabético. `GARANTIAS` y `PICKING` se incluyen también desde la configuración de exclusiones. El campo opcional `depositos` de `PUT /dms/stock` reemplaza el padrón; omitirlo conserva el anterior para agentes antiguos. Hay que actualizar el script del agente en el taller para descubrir depósitos vacíos nuevos. Las casillas valen para todos los repuestos (destildar RACK lo saca de todos) y quedan guardadas en el navegador (`localStorage`, sólo las diferencias con lo que viene por defecto). Código: `Cotizador_FE/src/features/shop-stock/`.

```json
// POST /dms/stock → { "materiales": ["1520865F0E"] }
{
  "actualizado": "2026-10-24T01:52:10Z",
  "desactualizado": false,
  "items": [
    { "material": "1520865F0E", "disponible": 122,
      "depositos": [
        { "deposito": "RACK", "existencia": 84, "ubicacion": null, "incluido_por_defecto": true },
        { "deposito": "CENTRAL", "existencia": 38, "ubicacion": null, "incluido_por_defecto": true }
      ] }
  ]
}
```

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

Reglas del matcher que importan: solo asigna un repuesto del catálogo si es exactamente ese (nunca uno parecido); un plural ("amortiguadores") son los dos lados (un ítem por lado si el catálogo los separa, o uno con cantidad 2) con la posición inferida del resto del pedido; y a los repuestos reales que no están en el catálogo les completa `interpretacion` con su nombre técnico. Un kit, juego o service que agrupa repuestos distintos ("kit de filtros", "kit de distribución") se separa en un ítem por repuesto, todos con el mismo `input` y con `kit` = nombre del kit; si se vende como una sola pieza (KIT JUNTAS REPARACION MOTOR) o son varias unidades del mismo repuesto ("juego de bujías" = bujía × 4), es un solo ítem.

### 6. Asesor IA para repuestos fuera de catálogo

Si el matcher marca un ítem con `interpretacion` (existe pero no está en el catálogo), `PartAdvisorService` (`app/services/part_advisor.py`) hace un ida y vuelta con un segundo modelo (`ANTHROPIC_ADVISOR_MODEL`, por defecto `claude-opus-5-5`), en paralelo con el resto de la cotización:

1. **Propuesta**: el modelo recibe el texto del cliente tal cual (no la interpretación del matcher: si es errónea lo arrastra a otra pieza), el vehículo (del EPC) y los repuestos del catálogo que comparten palabras con el pedido, marcados como descartados. Las familias que pertenecen al catálogo se filtran también en el código: si el pedido fuera una de ellas, el matcher ya la habría encontrado. Devuelve familias en una sola de dos listas: `familias_vehiculo` (hasta 6 que podrían ser de este vehículo) o `familias_otros_modelos` (si cree que el vehículo no lleva la pieza: hasta 6 habituales en otros modelos Nissan). Más una `aclaracion` para el vendedor y `es_kit`. Si el nombre sirve para piezas distintas ("volante": de dirección o del motor), propone familias de todas en lugar de elegir una; si es un kit que el matcher no separó, propone las de cada pieza del kit (pueden ser del catálogo).
2. **EPC**: se buscan esas familias para el chasis.
3. **Elección** (misma conversación, con los números completos del EPC): una opción si corresponde una sola pieza, varias con una `pregunta` para el cliente si son piezas distintas (ej. sonda delantera o trasera), o ninguna. Se descartan números que el EPC no devolvió. `seleccion` dice cómo se eligen: `"una"` si son alternativas, `"varias"` si son piezas de un kit; en ese caso el front muestra casillas y las piezas marcadas pasan a la tabla, cada una en su fila.

Si es de `familias_otros_modelos` y el EPC no devuelve nada, se informa que el vehículo probablemente no lleva esa pieza. En el front, estos ítems van en su propia sección "Asesor IA", debajo de la tabla, con la interpretación, el resultado, la confianza, las opciones y los botones **Validar** / **Descartar**: validar pasa el ítem a la tabla de repuestos identificados y llama a `POST /catalogo/candidatos`, que guarda la familia en la tabla `catalog_candidates` como **candidata pendiente**: el catálogo se actualiza a mano después de revisarlas (`GET /catalogo/candidatos`).

**Pedidos ambiguos.** Si el texto nombra piezas distintas y el pedido no aclara cuál, el matcher no elige: lo marca con `ambiguo: true` y lo manda al asesor (aunque alguna esté en el catálogo), que ofrece una opción por pieza con su `pregunta`. En ese caso el asesor sí puede proponer familias del catálogo.

**Corregir al asesor** — `POST /cotizar/asesor`. El vendedor puede corregir cualquier ítem (una sugerencia del asesor o un repuesto que el matcher asignó mal): escribe qué pieza es ("es la pantalla del electroventilador") y el asesor vuelve a hacer el ida y vuelta con todas sus indicaciones y las familias que ya descartó. Con una corrección también puede proponer familias del catálogo. Al validar, la última indicación se guarda como `interpretacion` de la candidata.

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
│   │       ├── epc_scraper.py         # sesión EPC (Playwright) + consultas a su API
│   │       ├── sap_stock.py           # login SAP (Playwright) + stock por httpx
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
DATABASE_URL=sqlite:///data/cotizador.db   # default; Postgres más adelante
```

---

## Base de datos

SQLite en `data/cotizador.db` (`DATABASE_URL`), con SQLAlchemy 2 y migraciones de Alembic en `migrations/`. El backend aplica las migraciones al arrancar (`upgrade_database()` en el lifespan de `main.py`). Las tablas están en `app/models/` y el acceso en los repositorios de `app/database/`, que son sincrónicos y los routers llaman con `asyncio.to_thread`. Para pasar a Postgres alcanza con instalar el driver y cambiar `DATABASE_URL`.

| Tabla | Qué guarda |
|---|---|
| `catalog_candidates` | Familias que sugirió el asesor y validó el vendedor. Antes vivían en `data/catalogo_candidatos.json`: la migración inicial las importa una vez si encuentra el archivo |
| `dms_stock` | Última foto del stock del taller, por código y depósito, con la ubicación si el agente la manda |
| `dms_stock_uploads` | Cada subida del agente del taller: cuándo llegó, cuántos repuestos y unidades |

Las fechas se guardan en UTC (`UTCDateTime` en `app/models/base.py`: SQLite no guarda la zona horaria). Para cambiar una tabla: modificar el modelo y crear la migración con `uv run alembic revision --autogenerate -m "descripcion"`.

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

2. **Familias equivalentes agrupadas** — las familias de un ítem del `/match` (ej: `["80101", "H0101"]`) ya forman el grupo equivalente para el EPC. Se consultan en paralelo y gana la primera, en orden, con resultado.

3. **Playwright sólo para los logins** — EPC: la pestaña queda abierta para que la app mantenga la sesión y la licencia, y los datos se piden a su API con `fetch()` desde esa pestaña (sin tocar el DOM). SAP: el browser se cierra apenas se obtiene la sesión; las cookies se guardan en `SAP_SESSION_FILE`. Ambas sesiones se precalientan al arrancar y se liberan tras `SESSION_KEEPALIVE_HOURS` sin uso.

4. **Singleton scraper compartido** — el `EPCScraper` vive en `app/routers/epc.py` y es importado por `app/routers/cotizar.py` para garantizar que ambos endpoints usen la misma instancia y la misma sesión de browser. Los pedidos al EPC pueden ir en paralelo y no dependen de un "chasis cargado": cada búsqueda indica su chasis.

5. **Dos modos de operación:**
   - Sin chasis: applicability desde BD (en el futuro)
   - Con chasis: consulta EPC para número exacto según versión

6. **Cache de resultados EPC** — cuando el EPC confirma un número, guardarlo en applicability con confidence = confirmed. La BD se enriquece sola.

---

## Próximos pasos en orden

1. **Modelo de catálogo en la base** (la base ya existe: SQLite + Alembic) con el modelo diseñado
2. **Cache de resultados EPC** en tabla `applicability`
3. **Tests unitarios** para `MatcherService` y `CotizarService` con mocks
4. **Pre-commit hooks** (ruff + pytest)
5. **Detección de repuestos en conjunto** via `part_composition`
6. **Equivalentes** via `spare_part_alternative`
