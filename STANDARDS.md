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

`.env.example` (este sí va al repo):
```
ANTHROPIC_API_KEY=
EPC_USERNAME=
EPC_PASSWORD=
EPC_URL=https://login.superservice.com/login/es-es/
DATABASE_URL=sqlite:///./nissan.db
```

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

- Mockear siempre dependencias externas: Anthropic API, Playwright, base de datos.
- Nombres de test descriptivos: `test_[qué]_[condición]_[resultado esperado]`.
- Un assert por test cuando sea posible.
- Los tests deben correr sin conexión a internet ni servicios externos.

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
- [ ] README actualizado si el cambio es visible para quien usa el proyecto
- [ ] Commit hecho con mensaje convencional descriptivo

---

## Contexto del dominio

Ver CONTEXT.md para la descripción completa del proyecto, decisiones de
diseño, modelo de datos y próximos pasos.
