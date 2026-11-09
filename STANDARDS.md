# Estándares de Desarrollo — Cotizador Repuestos Nissan

Seguí estos estándares en todo momento. No escribas código sin aplicarlos.
Antes de arrancar cualquier tarea, leé este archivo completo.

---

## Clean Code

- Nombres descriptivos y en inglés para código (variables, funciones, clases).
  El español solo para strings visibles al usuario y comentarios de dominio.
- Una función = una responsabilidad. Si necesitás escribir "y" para describir
  lo que hace una función, separarla en dos.
- Sin código muerto, sin TODOs sin ticket, sin comentarios que explican el "qué".
  Los comentarios solo para el "por qué" cuando no es obvio.
- Máximo 3 niveles de indentación. Si hay más, refactorizar con early returns
  o extrayendo funciones.
- Evitar números mágicos y strings hardcodeados. Usar constantes con nombre.

---

## Arquitectura

Estructura de carpetas del backend:

```
backend/
├── app/
│   ├── routers/        # endpoints HTTP, sin lógica de negocio
│   ├── services/       # lógica de negocio pura, sin saber de HTTP ni DB
│   ├── database/       # acceso a datos, queries, ORM
│   ├── models/         # entidades SQLAlchemy
│   ├── schemas/        # Pydantic schemas (request/response)
│   └── core/           # config, dependencias, constantes
├── tests/
│   ├── unit/           # tests de services con mocks
│   └── integration/    # tests de endpoints
├── .env                # variables reales (no commitear)
├── .env.example        # variables sin valores (sí commitear)
├── pyproject.toml      # definición del proyecto y dependencias
└── main.py
```

Reglas de dependencia entre capas:
- routers → services → database → models
- Las capas internas no importan nada de las externas
- Los services no importan FastAPI. Los database no importan schemas Pydantic.

---

## Stack y herramientas modernas

### Package management: uv (no pip directo)

```bash
# Instalar uv si no está
pip install uv

# Iniciar proyecto
uv init

# Agregar dependencia
uv add fastapi
uv add --dev pytest ruff

# Correr el proyecto
uv run uvicorn main:app --reload

# Sincronizar entorno desde lockfile
uv sync
```

uv genera `uv.lock` automáticamente — commitear siempre este archivo.
No usar pip freeze manual. Las dependencias quedan en `pyproject.toml`.

### Linting y formato: Ruff

Configurar en `pyproject.toml`:

```toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM"]

[tool.ruff.format]
quote-style = "double"
```

Correr antes de cada commit:
```bash
uv run ruff check . --fix
uv run ruff format .
```

### Pre-commit hooks (configurar una sola vez)

```bash
uv add --dev pre-commit
uv run pre-commit install
```

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: tests
        name: Run tests
        entry: uv run pytest tests/unit
        language: system
        pass_filenames: false
```

Una vez configurado, se ejecuta automáticamente antes de cada `git commit`.

---

## Variables de entorno

Toda configuración sensible en `.env`. Nunca hardcodear credenciales.

`.env.example` va al repo y lista **todas** las variables del backend, agrupadas por tema y con
su valor por defecto; las claves quedan vacías. El `.env` de cada PC sigue el mismo orden, con
todas las variables escritas, para que se vea qué usa sin abrir el código.
`tests/unit/test_config.py` falla si una variable de `Settings` no está en `.env.example` o si
el valor del ejemplo no coincide con el del código.

Leer con pydantic-settings en `app/core/config.py`:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    anthropic_api_key: str
    epc_username: str
    epc_password: str
    epc_url: str
    database_url: str

    class Config:
        env_file = ".env"

settings = Settings()
```

Lo que se pueda ajustar sin tocar el código (modelos, umbrales, límites, retenciones, tiempos
de espera de los servicios externos, direcciones de esos servicios) va como campo de `Settings`,
con su valor por defecto, su explicación en el docstring y su línea en `.env.example`. Se
quedan en el código, como constantes con nombre, los detalles de implementación que nadie
cambiaría al instalar (largos de truncado, intervalos internos, rutas de las APIs de Nissan,
selectores de las páginas) y los textos de los prompts.

---

## Observabilidad y errores

El sistema completo está en OBSERVABILIDAD.md. Reglas al escribir código nuevo:

### Logs

- Un logger por módulo: `logger = logging.getLogger(__name__)`. Nunca `print`.
- Niveles: `INFO` lo normal que vale la pena ver, `WARNING` lo que se recupera solo o es
  previsto (licencia en uso, límite de uso), `ERROR` lo que falló y alguien tiene que
  mirar, `DEBUG` el detalle para diagnosticar.
- Con traceback: `logger.exception(...)` dentro del `except`, no `traceback.format_exc()`.
- **Nunca loguear** contraseñas, tokens, cookies ni datos del cliente final (nombre,
  teléfono). El filtro de secretos es una red de seguridad, no un permiso.
- El código del request se agrega solo: no hace falta pasarlo al log.

### Medir

```python
from app.core.observability import annotate, step, track

with step("epc.familia", familia="16546"):   # una etapa: tiempo y error en el request
    ...
with track("sap", "login"):                  # además queda en dependency_events
    ...
annotate(items=12, items_sin_resultado=1)    # datos de negocio del request
```

- `step()` para cada llamada a un servicio externo, con el servicio como prefijo.
- `track()` para operaciones pocas e importantes de una integración (logins, licencias).
- En los atributos, nada de secretos ni datos personales.
- **Toda llamada a Claude pasa por `llm_call()`** (`app/services/llm_usage.py`): sin eso no
  se registra el costo ni se respeta el tope diario.

```python
with llm_call("match", settings.anthropic_model) as call:
    call.response = client.messages.create(...)
```

### Errores

- El usuario recibe un mensaje claro, en sus palabras y sin tecnicismos; el detalle
  técnico va al log. Nunca `detail=str(e)` con una excepción cualquiera.
- En un router, `server_error()` (o `report_error()` si la respuesta se arma a mano):
  loguean, guardan el error en el request y devuelven el mensaje para el usuario.

```python
except Exception as e:
    raise server_error(e, "No se pudo consultar el EPC. Probá de nuevo.", log=logger) from e
```

- Una excepción cuyo mensaje está escrito para el vendedor hereda de `UserFacingError`
  (`app/core/errors.py`) y se muestra tal cual.

### Rutas nuevas

- Cada ruta cae en un grupo de límites (`ROUTE_GROUPS` en `app/core/rate_limit.py`): los
  POST/PUT/PATCH/DELETE son escrituras por defecto. Una lectura por POST que el front
  dispara por cada fila se marca sin límite, o una cotización grande choca con él.
- Una ruta que el front consulta seguido va en `OBS_QUIET_PATHS`.
- Una pantalla interna usa `require_local_access` (con login, un rol de administrador).

### Tablas nuevas de observabilidad

Con `created_at`, `user_id` y `request_id` indexados, y su retención en
`RETENTION_SETTINGS` (`app/core/observability/retention.py`). Si se puede perder sin
afectar al negocio va en el schema `observability`, sin foreign keys; si crece mucho,
particionada por mes (`app/database/partitions.py`). Un dato del negocio no es telemetría
aunque lo genere la medición (los costos de la IA van en `app`, con el resto de las tablas).

### Tablas nuevas del negocio

- Van en `app`, el schema por defecto de los modelos. La base no tiene `public`: una
  tabla o una consulta sin schema falla en vez de quedar en otro lado.
- Un índice por consulta real que lo necesite, y uno en cada foreign key.
- Un `CHECK` en cada columna con valores fijos, igual al `Literal` del schema.
- `numeric` para importes y cantidades. Nunca `float`.
- Nada de borrar y reinsertar una tabla entera en una tarea periódica: escribir solo lo
  que cambió.

---

## Documentación

### Docstrings en todas las funciones públicas

Formato Google style:

```python
def match_parts(text: str) -> list[MatchResult]:
    """Identifica repuestos en texto libre y los mapea al catálogo.

    Usa Claude para interpretar jerga argentina, abreviaturas y
    referencias ambiguas. Infiere el lado dominante si no está explícito.

    Args:
        text: Texto libre del cliente con los repuestos solicitados.
              Puede venir separado por comas, enters o espacios.

    Returns:
        Lista de MatchResult con nombre normalizado, familias y comentarios.
        Los ítems no encontrados tienen nombre=None.

    Raises:
        AnthropicError: Si falla la llamada a la API de Claude.
        ValueError: Si el texto está vacío.
    """
```

### Endpoints FastAPI documentados

Cada endpoint debe tener summary, description, response_model y posibles
errores documentados. Así /docs queda completa automáticamente:

```python
@router.post(
    "/match",
    response_model=list[MatchResponse],
    summary="Matchear repuestos desde texto libre",
    description="""
    Recibe texto libre del cliente e identifica los repuestos solicitados
    mapeándolos al catálogo normalizado de Nissan.
    
    Entiende jerga argentina: pdi, fender, optica, sapito, etc.
    Infiere el lado (izq/der) cuando no está explícito usando contexto.
    """,
    responses={
        400: {"description": "Texto vacío"},
        500: {"description": "Error en la API de Claude"},
    }
)
```

### README.md

Mantener actualizado con cada cambio relevante. Secciones mínimas:
- Descripción del proyecto
- Requisitos
- Setup paso a paso
- Variables de entorno requeridas
- Cómo correr tests
- Estructura de carpetas
- Endpoints disponibles (o link a /docs)

---

## Control de versiones

### Commits convencionales

Formato: `tipo(scope): descripción en imperativo, minúscula`

Tipos:
- `feat`: nueva funcionalidad
- `fix`: corrección de bug
- `refactor`: cambio de código sin cambiar comportamiento
- `docs`: solo documentación
- `test`: agregar o corregir tests
- `chore`: tareas de mantenimiento (deps, config)

Ejemplos correctos:
```
feat(epc): agrega endpoint /epc para consultar superservice
fix(matcher): corrige inferencia de lado cuando no hay contexto
refactor(services): extrae lógica de login en EPCService
docs(readme): actualiza instrucciones de setup con uv
chore(deps): agrega playwright y pydantic-settings
```

### Reglas

- Un commit por tarea lógica completa. No commits de "wip" o "arreglos".
- Nunca commitear: .env, __pycache__, .venv, uv.lock NO (sí commitear uv.lock).
- Siempre commitear: .env.example, uv.lock, pyproject.toml.
- Antes de cada commit: `git status` y `git diff` para revisar qué se commitea.

---

## Testing

### Estructura

```python
# tests/unit/test_matcher_service.py

from unittest.mock import patch, MagicMock
from app.services.matcher_service import MatcherService

def test_match_returns_correct_family_for_pdi():
    """Verifica que 'pdi' se mapea a Puerta Delantera Izquierda."""
    with patch("app.services.matcher_service.anthropic_client") as mock_client:
        mock_client.messages.create.return_value = MagicMock(
            content=[MagicMock(text='[{"input":"pdi","nombre":"Puerta Delantera Izquierda","familias":["80101"],"comentario":null}]')]
        )
        service = MatcherService()
        result = service.match("pdi")
        assert result[0].nombre == "Puerta Delantera Izquierda"
        assert "80101" in result[0].familias
```

### Reglas

- Mockear siempre dependencias externas: Anthropic API, Playwright. La base no se mockea:
  los tests corren contra Postgres en una base propia (`cotizador_test`), porque otro motor
  no detecta lo que Postgres rechaza (largos, foreign keys, tipos).
- Nombres de test descriptivos: `test_[qué]_[condición]_[resultado esperado]`.
- Un assert por test cuando sea posible.
- Los tests deben correr sin conexión a internet ni servicios externos, salvo el Postgres
  local.

Correr tests:
```bash
uv run pytest tests/unit -v
uv run pytest tests/ -v  # todos
```

---

## Checklist antes de terminar cada tarea

- [ ] El código corre sin errores
- [ ] `uv run ruff check .` sin warnings
- [ ] Tests unitarios pasan: `uv run pytest tests/unit -v`
- [ ] Docstrings completos en funciones públicas nuevas
- [ ] .env.example actualizado si se agregó variable nueva
- [ ] Llamadas externas medidas con `step()`, y a Claude con `llm_call()`
- [ ] Errores con mensaje para el usuario y detalle técnico en el log
- [ ] Rutas nuevas en el grupo de límites que corresponde
- [ ] README actualizado si el cambio es visible para quien usa el proyecto
- [ ] Commit hecho con mensaje convencional descriptivo

---

## Contexto del dominio

Ver CONTEXT.md para la descripción completa del proyecto, decisiones de
diseño, modelo de datos y próximos pasos.
