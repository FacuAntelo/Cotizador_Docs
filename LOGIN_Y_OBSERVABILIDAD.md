# Login y observabilidad — qué tener en cuenta al sumar usuarios

Guía de alto nivel para cuando se implemente el login: qué ya está preparado, qué hay que
agregar y qué decidir. El sistema de observabilidad está en
[OBSERVABILIDAD.md](OBSERVABILIDAD.md). El plan de usuarios, roles, permisos y organizaciones
está en [USUARIOS_Y_PERMISOS.md](USUARIOS_Y_PERMISOS.md).

## Estado (9 de noviembre de 2026, rama `feat/usuarios`)

Hechos con la etapa 1 del login: `bind_user` en la dependencia global de autenticación
(`app/core/auth.py`), que corre antes de los límites de uso, así cuentan por usuario;
Monitoreo y `/salud/detalle` detrás de `monitoreo.ver`; el bloqueo por usuario, el mensaje
único y los logins fallidos sin la contraseña en el log (puntos 1 a 4). La telemetría usa la
cookie de la sesión, que viaja sola porque el front y la API están en el mismo nombre
(punto 5). Falta la alerta de muchos logins fallidos, en la etapa 6.

## Lo que ya está preparado

- **`user_id` en todas las tablas de observabilidad** (`request_log`, `llm_calls`,
  `quote_runs`, `dependency_events`, `frontend_events`), con índice. No hace falta migrar.
- **Un solo punto de enganche**: `bind_user(user_id)` (`app/core/observability`). Desde ahí,
  el usuario aparece solo en los logs, en cada fila guardada, en el costo de la IA, en los
  límites de uso y en las trazas (`enduser.id`).
- **Los límites de uso ya cuentan por usuario** cuando hay uno (`usuario:<id>`), y por IP
  si no. La ruta `POST /auth/login` ya tiene su grupo (`RATE_LIMIT_LOGIN=5/15min`).
- **Monitoreo ya agrupa por usuario**: costos por usuario, cotizaciones por vendedor.

## Qué agregar

1. **Llamar a `bind_user` en la dependencia de autenticación**, en cada request, apenas se
   valida la sesión o el token. El contexto del request lo crea el middleware antes, así
   que alcanza con llamarla dentro de la dependencia; el stream de la cotización corre en
   el mismo contexto y queda a nombre del usuario.
2. **Usar un id estable y opaco** (el id interno del usuario), no el email ni el nombre: los
   logs y las métricas no tienen que tener datos personales de más.
3. **Acceso a Monitoreo**: reemplazar `require_local_access` (`app/core/access.py`) por el
   permiso `monitoreo.ver`, que tiene solo el administrador de plataforma (decidido el
   8 de noviembre de 2026; ver USUARIOS_Y_PERMISOS.md).
4. **Login y fuerza bruta**: el grupo `login` ya limita intentos por IP; sumar un bloqueo
   temporal por usuario tras varios fallos, responder siempre el mismo mensaje («usuario o
   contraseña incorrectos») y loguear los intentos fallidos con el usuario pedido pero
   **nunca la contraseña** (el filtro de secretos ayuda, pero la regla es no pasarla al log).
   Una alerta nueva de «muchos logins fallidos» se arma igual que las de EPC y SAP.
5. **Telemetría del navegador**: `sendBeacon` no permite mandar el header
   `Authorization`. Con sesión por cookie, el beacon la lleva sola (CORS necesita
   `allow_credentials=True`, que ya tiene orígenes explícitos). Con token en header, pasar
   la telemetría a `fetch(..., { keepalive: true })` con el header.
6. **Tope de la IA por usuario** (opcional): hoy el tope es global. Para uno por vendedor,
   llevar el gasto en `LLMUsageTracker` por `user_id` (o sumarlo desde `llm_calls`) y una
   configuración `LLM_USER_DAILY_BUDGET_USD`.
7. **Auditoría del negocio** (quién creó, aceptó o pidió cada presupuesto): no es
   observabilidad; va en las tablas del negocio (`created_by`, `accepted_by`) con su
   migración, junto con el login.

## Perfiles y diagnóstico por usuario

El perfilador (`scripts/perfil.ps1`) mira el proceso entero, no a un usuario. Para ver qué
le pasó a un vendedor: filtrar `request_log` por `user_id`, rastrear el código que le
mostró el front, o buscar en las trazas por `enduser.id`.

## Privacidad

Con login, la actividad de cada vendedor queda registrada: es información personal de los
empleados. Definir quién puede verla (rol administrador), cuánto se guarda (las
retenciones ya son configurables) y avisarlo internamente. Revisar los requisitos de la
Ley 25.326 de protección de datos personales con quien corresponda.

## Tests a sumar

- `bind_user` en la dependencia de autenticación deja el usuario en `request_log` y `llm_calls`.
- Los límites de uso cuentan por usuario y no por IP cuando hay login.
- Monitoreo responde 403 a un usuario sin rol de administrador.
- El login fallido no deja la contraseña en el log.
