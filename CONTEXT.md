# Cotizador de Repuestos Nissan — Contexto del Proyecto

## Descripción general

Sistema para cotizar repuestos Nissan. Dado un texto libre del cliente (con jerga argentina, abreviaturas, etc.), identifica los repuestos solicitados y obtiene los números de pieza correctos consultando el catálogo EPC oficial de Nissan.

---

## Stack tecnológico

- **Backend:** Python 3.15 + FastAPI, gestión de deps con `uv`
- **Frontend:** React + TypeScript + Vite, SPA con TanStack Query y tipos generados del OpenAPI (ver `Cotizador_FE/README.md`)
- **LLM:** Claude API vía Anthropic SDK: el matcher con claude-sonnet-5-5 (esfuerzo bajo) y el asesor con claude-opus-5-5
- **Automatización browser:** Playwright, sólo para los logins (EPC y SAP); los datos se piden a las APIs
- **Base de datos:** Postgres 18 con SQLAlchemy 2, psycopg y migraciones de Alembic (ver [Base de datos](#base-de-datos))
- **Observabilidad:** logs estructurados, medición de cada request, costos de la IA, límites de uso, alertas y pantalla Monitoreo, sin infraestructura externa; trazas OpenTelemetry opcionales (ver [OBSERVABILIDAD.md](OBSERVABILIDAD.md))
- **Linting:** Ruff
- **Puerto backend:** 8001 en desarrollo y 8000 en productivo (`SERVER_PORT` vacío) | **Puerto frontend:** 5173

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

Permite forzar un login nuevo y obtener el estado de la sesión: `time_left_seconds` es lo que le queda al token del EPC (se lee del JWT) e `inactivity_left_seconds` lo que falta para que el backend libere la sesión por falta de uso. Al arrancar el backend la sesión se abre sola en segundo plano (`WARMUP_ON_START`). Si alguien cierra a mano la ventana del EPC o todo el navegador, el estado pasa a inactivo y el próximo login abre un navegador nuevo.

### 3b. Stock SAP — `POST /sap/stock` y `GET /sap/session`

Playwright hace el login SAML (IAS) y corta apenas el portal emite la sesión (`/login/callback`); el stock se pide con httpx a un endpoint CPI del launchpad. Las cookies se guardan en `SAP_SESSION_FILE`, así un reinicio del backend no re-loguea, y con la sesión de IAS viva el re-login no pasa por el formulario (SSO). Un keep-alive cada 5 min (`GET /cp.portal/portal/rest/v1/personalization`, 200 con sesión / 401 sin ella) la mantiene viva mientras haya uso reciente. Medido: la sesión del portal sobrevive 25 min sin uso y vence antes de los 35; a los 35 min la sesión de IAS seguía viva y el re-login fue silencioso. Las consultas corren en paralelo; sólo el login se serializa. `GET /sap/session` devuelve el estado de la sesión sin consultar SAP, y la barra del front lo muestra junto al del EPC.

| Escenario | Tiempo |
|---|---|
| Login completo + consulta | ~14 s |
| Reinicio con cookies vigentes | ~2 s |
| Portal vencido, IAS vivo (SSO) | ~6,5 s |
| Consulta con sesión abierta | ~1,3–2 s (8 en paralelo ≈ 2,2 s) |

### 3c. Stock del taller — `PUT /dms/stock`, `POST /dms/stock` y `GET /dms/stock/estado`

> Detalle del DMS (servidor, fórmula del stock, por qué un agente): ver `INTEGRACIONES.md`.

El stock propio del taller vive en el DMS (AutoDealer, SQL Server en la red del taller), al que el backend en la nube no llega. Un agente en el servidor del taller (`Cotizador_BE/agente_dms/`) calcula el stock cada pocos minutos y sube la foto completa con `PUT /dms/stock` y `Authorization: Bearer <DMS_AGENT_TOKEN>`; la foto reemplaza a la anterior y se guarda en la base (tablas `dms_stock` y `dms_stock_uploads`). `POST /dms/stock` responde como `/sap/stock` (`{"materiales": [...]}`), con el total y el detalle por depósito, comparando códigos sin guiones. Si la foto tiene más de `DMS_STALE_MINUTES`, la respuesta trae `desactualizado: true`. Se guardan los depósitos con su ubicación (si el agente la manda), pero los de `DMS_EXCLUDED_DEPOSITS` (por defecto `GARANTIAS,PICKING`) vienen con `incluido_por_defecto: false` y no suman a `disponible`. Los de `DMS_IGNORED_DEPOSITS` (por defecto `MINOZZI`) se descartan al recibir la foto: no se guardan ni se muestran.

En el front, la columna **Stock** de la tabla (y el bloque "Stock" de las sugerencias del asesor) muestra las unidades de cada N° de pieza. El número y la **(i)** abren un popover con cada depósito: unidades, ubicación y una casilla para sumarlo o no. La consulta incluye el padrón completo enviado por el agente desde `dbo.Depositos.Descripcion`, guardado en `dms_deposits`, con cero donde no hay existencias. Primero aparecen los depósitos con stock y luego los demás en orden alfabético. `GARANTIAS` y `PICKING` se incluyen también desde la configuración de exclusiones. El campo opcional `depositos` de `PUT /dms/stock` reemplaza el padrón; omitirlo conserva el anterior para agentes antiguos. Para que el agente del taller mande el padrón hay que copiar los scripts nuevos y volver a correr el instalador, que da el permiso de lectura sobre `Depositos`. Las casillas valen para todos los repuestos (destildar RACK lo saca de todos) y quedan guardadas en el navegador (`localStorage`, sólo las diferencias con lo que viene por defecto). Código: `Cotizador_FE/src/features/shop-stock/`.

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

### 3d. Localizador de partes de la red — `GET /localizador/{numero_parte}`

> Detalle de Simetrical (login, endpoints, datos mal cargados): ver `INTEGRACIONES.md`.

Qué concesionarios de la red Nissan tienen un número de pieza, según el localizador de partes de Simetrical. El backend se loguea con httpx (sin navegador) en la primera consulta y pide todas las sucursales en un solo POST (~0,3 s). Devuelve el total de la red y las sucursales agrupadas por provincia: **CABA** primero, después **Provincia de Buenos Aires** y luego el resto de las provincias, siempre de la más cercana a la más lejana al concesionario propio (`LOCALIZADOR_ORIGEN_LAT/LON`). La provincia y la ubicación de cada sucursal salen de una tabla propia (`app/services/locator_branches.py`), porque el sitio las trae mal cargadas. Una sucursal con stock en varios depósitos se suma. Sin `SIMETRICAL_USERNAME` / `SIMETRICAL_PASSWORD` responde 503.

En el front, cada línea de la columna **Stock en planta** tiene una lupa que abre un popover con el total en red y las provincias: CABA y Provincia de Buenos Aires abiertas con cada concesionario, el resto plegadas con su total y la distancia a la más cercana. El propio aparece marcado como "Tu concesionario". La consulta se hace al abrir la lupa y se reutiliza 5 minutos. Como la lupa va en cada número, aparece también en cada alternativa compatible al expandir ⇄. Código: `Cotizador_FE/src/features/part-locator/`.

```json
// GET /localizador/1520865F0E (datos del 2026-10-25, recortado)
{
  "numero_parte": "1520865F0E",
  "total_red": 6076,
  "provincias": [
    { "provincia": "CABA", "total": 1306,
      "concesionarios": [
        { "nombre": "ANTELO", "sucursal": null, "existencia": 122, "distancia_km": 0, "propio": true },
        { "nombre": "NISSAN CAPITAL", "sucursal": "Villa Luro", "existencia": 299, "distancia_km": 3, "propio": false }
      ] },
    { "provincia": "Provincia de Buenos Aires", "total": 1507, "concesionarios": ["…"] },
    { "provincia": "Santa Fe", "total": 1207, "concesionarios": ["…"] }
  ]
}
```

### 3e. Presupuesto formal — `/presupuestos`

En la tabla de **Repuestos identificados** cada fila tiene una casilla, y todas vienen tildadas: el vendedor destilda lo que no quiere. Lo que se suma después también entra tildado (un repuesto que se valida con el asesor IA, las piezas elegidas de un kit). Con otra cotización vuelve a estar todo tildado. Con los repuestos tildados, el vendedor puede:

- **Copiar para WhatsApp**: un renglón por repuesto con descripción, precio final con IVA y disponibilidad (y el total, si se tilda "Con total").
- **Generar presupuesto**: abre a la derecha la hoja del presupuesto formal, copiada de la planilla del concesionario: logo del concesionario a la izquierda, número en el centro, logo Nissan a la derecha, franja con el nombre del concesionario y "Datos del cliente", dirección / CUIT / teléfono / e-mail, vehículo / VIN / nombre / patente / teléfono del cliente, fecha y validez, la tabla (descripción, unidades, precio unitario sin IVA, subtotal y total con IVA) y los totales. Con la hoja abierta, tildar o destildar una fila la suma o la saca, y la hoja sigue a la tabla: si en la fila se elige otro número (⇄ o el detalle del precio), se cambia la cantidad o el criterio de cotización, el renglón toma el número, el precio, el sugerido y la disponibilidad nuevos. Con el mismo número se respeta el precio que haya ajustado el vendedor. Los repuestos que entran tildados con la hoja abierta (validados, piezas de un kit o los de una cotización nueva) se suman como renglones, y un renglón que se armó antes de que llegaran el precio o el stock los toma cuando llegan.

Todo se edita sobre la hoja: textos, cantidades, precio unitario sin IVA, total con IVA del renglón y el total general (que reparte el cambio entre los renglones en proporción). Al tocar un renglón aparece una fila de ajustes: −10 / −5 / +5 / +10 %, "De lista" (precio de la lista del concesionario), "Sugerido Nissan" (SUG con IVA), disponibilidad y ocultar, y una X roja en el margen izquierdo del renglón que lo saca del presupuesto (y lo destilda en la tabla). "Ocultar" cierra la fila hasta que se vuelve a tocar un renglón. Arriba de la hoja están los ajustes para todos los precios, un porcentaje libre y "Precios de lista" / "Sugeridos Nissan". La opción "Incluir N.º de pieza" agrega la columna. Se puede agregar un ítem a mano (ej. mano de obra) con el botón o con **Enter** en cualquier campo de un renglón, que agrega uno vacío debajo y deja el cursor en su descripción, como en una planilla. **Backspace** en la descripción de un renglón que no tiene nada (sin descripción, número ni precio) lo borra y deja el cursor al final del anterior. En los datos del cliente o del concesionario, Enter no agrega nada. Los renglones que quedan vacíos (sin descripción, número ni precio) se descartan solos al guardar. El precio de lista y el sugerido de cada renglón se guardan con el presupuesto, así siguen disponibles al reabrirlo. Si falta algo para guardar (hoja vacía, ítems sin descripción), aparece un aviso en el centro de la pantalla y los ítems incompletos quedan marcados en rojo.

Reglas:

- El precio que se guarda y se edita es el **unitario con IVA** (2 decimales); el sin IVA se calcula dividiendo por 1,21. Así un total redondo con IVA (ej. $ 2.181.600) da el mismo unitario que la planilla ($ 1.802.975,21). El subtotal suma la columna sin IVA y el total la columna con IVA.
- **Disponibilidad**: "En stock" si el taller tiene la cantidad pedida (con los depósitos tildados), "Disponible en planta" si la tiene SAP, y **diferido** si no alcanza en ninguno de los dos. Si el taller lo tiene, no es diferido aunque SAP lo marque así. Diferido se imprime "(DIFERIDO)" en rojo junto a la descripción. Se puede cambiar a mano en cada renglón.
- **Validez**: fin de mes por defecto, o de 1 a 4 semanas; se imprime con la fecha de vencimiento.
- **Número**: el último guardado más uno; se puede cambiar hasta guardarlo (para seguir la numeración de la planilla). Si al guardar otro vendedor ya tomó el número propuesto, la base asigna el siguiente; si el número lo escribió el vendedor y existe, responde 409.
- **Guardar y PDF** guarda (o actualiza) el presupuesto y descarga "Presupuesto 008709 - Cliente.pdf" en A4. Para que no se pueda copiar ni editar el texto, cada página es una **imagen** de la hoja (`html-to-image`, a ~290 ppp) y el PDF va cifrado con permiso de solo imprimir (`jspdf`; se abre sin contraseña). Las páginas se cortan entre renglones. Es una traba para el uso normal, no una garantía: una imagen se puede retocar con un editor de imágenes. Código: `Cotizador_FE/src/features/estimate/estimatePdf.ts`.
- Los datos del concesionario que se usen al guardar quedan como los de los próximos presupuestos (`PUT /presupuestos/concesionario`). El logo se cambia haciendo clic en él (PNG, JPG, WebP o SVG de hasta 1 MB) y se guarda como data URL. Sin datos guardados, el front usa los de la planilla y el logo recortado de ella (`Cotizador_FE/src/features/estimate/assets/`), que conviene reemplazar por el original.

**Mano de obra y chapa y pintura.** Debajo de la tabla, junto a "+ Agregar ítem", hay dos botones:

- **+ Agregar MO**: suma un renglón "Mano de obra" de 1 hora al **precio por hora del taller**. Ese precio (sin IVA) se carga en el campo "Hora de MO sin IVA" de la misma fila y queda guardado con los datos del concesionario (`dealer_profile.labor_hourly_rate`, `precio_hora_mano_obra` en `PUT /presupuestos/concesionario`). Sin precio cargado, el renglón entra en cero. Se pueden agregar varios.
- **+ Agregar chapa y pintura** (**Editar chapa y pintura** si ya hay): abre un pop up donde el repuestero carga cada tarea con su **descripción**, sus **paños** y sus **días de chapa** (con decimales, "0,5"). La primera vez propone "Reemplazo y pintado <repuesto>" con un paño por unidad para cada pieza de carrocería que se reemplaza (paragolpes, guardabarros, puerta, capot, baúl/portón, techo, zócalo, aleta, tapa de combustible; no cuentan molduras, soportes, ópticas ni herrajes). Enter en una descripción agrega una tarea debajo. Al confirmar, la hoja muestra:
  - **Paños (p)** con la suma de los paños y **Días de chapa (c)** con la suma de los días. El precio unitario de cada uno se carga en la hoja y se conserva al volver a editar; lo que suma cero no aparece. Esas cantidades son la suma de las tareas: tocarlas abre el pop up.
  - Al final, sin cantidad ni precio, **Detalle chapa y pintura**: una tarea por línea con su aclaración, ej. "Reemplazo y pintado paragolpes delantero (2p, 1c)", "Reparación guardabarros dd (1p, 1c)", "Pintado cacha der (0,5p)". La casilla "Mostrar los paños y días de cada tarea" oculta las aclaraciones (y los renglones pasan a llamarse "Paños" y "Días de chapa").

  Las tareas se guardan con el presupuesto (`chapa_pintura`, columna `estimates.bodywork`), así el pop up se reabre con lo cargado. Quitar el detalle de la hoja, o "Quitar chapa y pintura" en el pop up, borra las tareas. Un detalle armado antes del pop up se abre como tareas sin paños ni días. Código: `Cotizador_FE/src/features/estimate/bodywork.ts` y `BodyworkDialog.tsx`.

La cantidad de mano de obra, días de chapa y paños admite **decimales** (hasta dos, "2,5"); la de los repuestos sigue siendo entera. Cada renglón guarda su tipo (`tipo`: `repuesto`, `mano_obra`, `dias_chapa`, `panos`, `detalle_chapa_pintura`). Los pedidos y el seguimiento de abastecimiento sólo consideran los renglones de tipo `repuesto`.

**Compañía de seguros.** En los datos del cliente, **Seguro** es un desplegable: **Particular** (sin compañía) o una de las compañías cargadas. La lista arranca vacía: "+ Agregar compañía…" escribe una nueva, que queda para los próximos presupuestos y no se repite con otras mayúsculas (`GET` / `POST /aseguradoras`, tabla `insurers`). Se imprime en la hoja y el PDF ("Seguro: Sancor Seguros" o "Particular") y se ve en la lista de guardados.

La pantalla **Presupuestos** (`/presupuestos`) busca los guardados (por número, cliente, teléfono, vehículo, patente o VIN) y los reabre para editarlos o volver a descargar el PDF; también arranca uno en blanco. Código: `Cotizador_FE/src/features/estimate/` y `Cotizador_BE/app/routers/estimates.py`.

### 3f. Generador de pedidos y completitud del presupuesto

En el panel compartido por Cotizador y Presupuestos, **Aceptado** aparece cuando el presupuesto
está guardado. Después de marcar la aceptación se habilita **Generar pedido**, que abre una vista
previa y no realiza ninguna operación en el portal de Nissan.

Se consulta la matriz de reemplazos vigente hacia adelante, los costos PNC sin IVA de la lista
Nissan, el stock de planta y la foto del DMS. Se elige el número compatible de menor costo con
cantidad suficiente en planta. El costo cero de la lista se trata como faltante; si hay varios
costos distintos para un mismo código, no se elige uno sin revisión.

El stock del taller usa los depósitos incluidos por defecto; PICKING nunca cubre este pedido. Si
el stock alcanza parcialmente, se propone solo el faltante. Las existencias del taller y de planta
se reparten entre renglones repetidos para no usarlas dos veces. Lo ya pedido o apartado se
descuenta del faltante.

La vista tiene dos tablas:

- **Repuestos a pedir**: solo número de pieza, descripción (con las unidades si son más de una)
  y costo total con IVA, más el total del pedido. **Quitar** lo pasa a Descartados.
- **Descartados**: número, descripción y **motivo** en una línea ("En stock del taller: 1 u.",
  "Diferido: sin stock suficiente en planta Nissan", "Sin costo en la lista Nissan", "Varios
  costos en la lista Nissan", "Stock en planta Nissan sin confirmar", "Sin número de pieza",
  "Quitado a mano"). **Incorporar** lo pasa a pedir; si le falta costo se completa ahí mismo
  (con los costos de la lista si hay varios) y, si tiene compatibles, se elige el número. Los
  diferidos tienen la lupa del localizador de la red.

Sin foto del taller no se prepara una propuesta automática; un SAP caído se informa como
disponibilidad desconocida, nunca como stock cero confirmado.

**Confirmar y descargar Excel** guarda una exportación inmutable antes de descargarla. La hoja
Pedido contiene únicamente **Número de pieza** y **Cantidad**; los códigos son texto para conservar
ceros iniciales y los repetidos se agrupan sumando cantidades. Los costos quedan en el sistema, no
en el Excel. Al confirmar, lo que cubre el stock del taller queda **Para separar** (si un cubierto
se incorpora para pedirlo, su stock no se aparta). Si todo está en stock no hay Excel: el botón es
**Confirmar para separar**.

Después de generar aparece **Excel generado · falta registrar Nº de pedido**. Se puede volver a
descargar o anular si no se cargó en Nissan; anularlo también libera lo que se dejó para separar.
Mientras está pendiente no se genera otro. Al registrar el número, cada ítem del Excel queda
**Pedido** con ese número y fecha. La exportación usa un identificador de reintento y las
mutaciones comparan la revisión del presupuesto, evitando duplicados cuando trabajan dos
operadores.

**Completitud** es el indicador del presupuesto: porcentaje de unidades **separadas** sobre las de
los ítems con número de pieza (los ítems sin número no cuentan). Al desplegarlo (**Ver estados**)
se ve cada ítem con sus estados y el repuestero los cambia a medida que llegan:

| Estado | Cuándo |
|---|---|
| Sin pedir | Nadie lo pidió ni apartó |
| Pedido | Está en un Excel; muestra Nº y fecha de Nissan (o "falta registrar el Nº") |
| Para separar | Llegó lo pedido (guarda la fecha de llegada) o está en el stock del taller |
| Separado | Apartado para el cliente; guarda la fecha. Es lo único que suma al porcentaje |

Los estados nacen del pedido, pero el desplegable de cada ítem **siempre** deja cambiarlo:

- Lo **Sin pedir** pasa directo a cualquier estado (`POST /presupuestos/{n}/abastecimiento`):
  Pedido queda como pedido por fuera del Excel ("Sin Nº de Nissan" y la fecha); Para separar y
  Separado, como stock del taller.
- Un registro pasa a cualquier estado (lo del stock que no estaba y se pidió pasa a Pedido) o
  vuelve a **Sin pedir**: se anula y entra en el próximo pedido (reemplaza a "No está en stock").
- Lo que está en un Excel sin registrar cambia de estado después de registrar el número.

Nada de esto escribe movimientos ni reservas en el DMS.

**Aceptación con seguro.** Si el presupuesto tiene compañía, **Aceptado** abre un cuestionario de
tres pasos; cada uno se puede omitir y lo omitido queda "sin indicar":

1. **Franquicia**: con franquicia (y el monto) o sin franquicia.
2. **¿Lleva inspección final?** Sí o no.
3. **¿Se aceptó el pago completo como figura en el presupuesto?** Tres respuestas:
   - **Sí, todo**: completa repuestos y mano de obra con IVA (la mano de obra incluye paños y
     días de chapa) y el **total a abonar por la compañía** = repuestos + mano de obra −
     franquicia.
   - **Sí, todo, pero por otro monto**: no excluye nada y no completa nada; se carga lo que paga
     el seguro (se muestra lo del presupuesto como referencia). Con repuestos y mano de obra el
     total se calcula solo; si no, se escribe a mano.
   - **No, hay cosas que no dio**: lista todos los ítems para marcar lo que no dio el seguro:
     quedan **excluidos** y los montos se recalculan sin ellos (se pueden corregir a mano).

Sin compañía, Aceptado acepta directo, como antes. Al aceptar se habilita la segunda pestaña,
**Aceptación** (la primera es **Presupuesto**), donde todo se edita y se guarda al momento:
franquicia (sin indicar / con franquicia y monto / sin franquicia), inspección final (sin indicar /
sí / no), **orden del seguro** (un PDF de hasta `MAX_INSURANCE_ORDER_MB`, 10 MB por defecto; subir otro la reemplaza), **pago de la
compañía** (repuestos, mano de obra y total; el total se calcula solo si están los dos, si no se
escribe a mano; "Completar con lo del presupuesto" los llena sin los excluidos) y **Excluidos**
("Se aceptó el presupuesto completo" si no hay; si no, la lista con **Revertir** y **Agregar
excluidos**). En un presupuesto particular solo aparece Excluidos.

En la pestaña Presupuesto, arriba de la hoja y fuera del PDF, se ve **Tiene inspección final** en
rojo, **Inspección final sin confirmar** en amarillo (o "Sin inspección final"), la franquicia y
cuántos ítems se excluyeron. Los excluidos salen **tachados** en la hoja y en el PDF; el total del
presupuesto no cambia. Un repuesto excluido no se pide ni cuenta para la Completitud (se informa
aparte); para excluir uno que ya está pedido o apartado, primero hay que pasarlo a Sin pedir.
Endpoints: `GET` / `PUT /presupuestos/{n}/aceptacion`, `POST` / `GET` / `DELETE
/presupuestos/{n}/orden-seguro`; `POST /presupuestos/{n}/aceptado` recibe además `aceptacion`
con las respuestas del cuestionario.

Los renglones del presupuesto conservan su ID al guardar y reordenar. Si un ítem tiene historial
de pedidos o abastecimiento no se permite borrarlo ni cambiar su número o cantidad; se puede
editar su descripción y precio. Una necesidad adicional se agrega como otro renglón.
Los registros cargados a mano antes (comprado, préstamo, separado)
se siguen mostrando. Código: BE app/{routers,database,services,schemas}/{orders,acceptance}.py,
app/models/orders.py; FE src/features/orders/, src/features/acceptance/, src/api/orders.ts y
src/api/acceptance.ts.

### 3f. Préstamos de repuestos — `/prestamos`

La pantalla **Préstamos** permite registrar movimientos entre concesionarios. Arriba muestra dos
contadores de unidades pendientes: **Debés** suma los registros de tipo `pedido` y **Te deben**
suma los registros de tipo `prestado`. El formulario permite elegir el tipo, cargar uno o varios
números de pieza (uno por línea, separados en filas al guardar), cantidad (1 por defecto),
concesionario y fecha de inicio (hoy por defecto).

La grilla muestra número de pieza, cantidad, tipo, concesionario, estado, fecha de inicio y fecha
de fin. Los estados son **pendiente**, **devuelto** y **comprado**; al pasar un registro a
devuelto o comprado se guarda la fecha de fin del día.
Código: BE `app/{routers,database,models,schemas}/loans.py`; FE `src/pages/LoansPage.tsx` y
`src/api/loans.ts`.

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

Lo que el catálogo no resuelve pasa directamente por el asesor IA (sección 6): el ítem que el
matcher no encontró, tenga `interpretacion` o no, y el que encontró pero sin números en el EPC
para el vehículo. Esos ítems vuelven con `nombre=null`, la `sugerencia` del asesor y, en
`interpretacion`, la descripción que había propuesto el matcher (su interpretación o el nombre
del catálogo). Si el EPC está caído, el ítem del catálogo queda sin número y no se consulta al
asesor, que tampoco podría buscar.

**Agregar un repuesto por número** — `GET /precios/repuestos?numero=…` (permiso `cotizar`).
Debajo de la tabla, el vendedor escribe un número de pieza, completo o una parte (con guiones,
espacios o minúsculas, da igual). El backend busca en la lista de precios de Nissan (unos 43.000
repuestos, más los códigos repetidos): primero los que empiezan con el texto y después los que
lo contienen, hasta 20, y avisa si hay más (`hay_mas`). Desde 3 caracteres, con una consulta por
pausa. El elegido se suma al final de la tabla con la descripción de la lista, la marca
«Agregado por número», su stock de SAP y del taller, sus reemplazos y su precio, y sin
«Corregir IA». Un número completo de 10 caracteres que no está en la lista se puede agregar
igual, con el número como nombre. Lo agregado a mano queda al cotizar otro pedido para el mismo
vehículo: las filas del pedido nuevo van antes, y cambiar de vehículo lo saca. Mientras el
matcher lee un pedido nuevo no se puede agregar.

**Historial del chasis** — `GET /vehiculos/{chasis}/repuestos` (con los días que se guarda,
`dias`), `POST` para anotar y `DELETE /vehiculos/{chasis}/repuestos/{numero}` para quitar uno
(permiso `cotizar`). El cotizador recuerda los números cotizados para cada chasis, con la
descripción y lo que dice el EPC (`epc`: verificado, fuera de catálogo o sin verificar), en la
tabla `vehicle_parts` de cada organización. Se anotan solos: los del catálogo con números del
EPC al terminar la cotización (con el nombre del catálogo), los del asesor IA cuando el
vendedor los valida (con la descripción del asesor) y los agregados por número que el EPC
verificó. Lo fuera de catálogo EPC o sin verificar entra solo si el vendedor toca «Agregar al
historial» en la fila (la primera vez se le avisa, con «No volver a mostrar», que se recuerda
por usuario con `POST /auth/avisos-ocultos`). Lo que el asesor sugiere sin validar no se anota.
Con el vehículo elegido, debajo de «Repuestos identificados» está la sección «Historial de este
chasis»: la lista a la vista, la última consulta primero, con la etiqueta de lo que no está en
el EPC; se elige con casillas y **Cotizar N** los suma a la tabla con la marca «Del historial»
y su precio, stock y reemplazos de ese momento. Los que ya están en la cotización aparecen
marcados y no se vuelven a sumar; cada uno se puede quitar con la X. Cada consulta renueva la
fecha; lo que no se cotiza en `VEHICLE_PARTS_DAYS` días (180 por defecto) deja de mostrarse y
se borra en la próxima anotación de la organización.

**Agregar repuesto** — el botón «+ Agregar repuesto» (también con la tabla vacía, si hay un
vehículo) abre una ventana con la búsqueda en la lista de precios. El número elegido se
verifica con `POST /epc/verificar`: se busca en el EPC, para el chasis, su familia (los
primeros 5) y se compara. Si está, entra; si el EPC tiene otro y el escrito figura compatible
(Nissan o usuarios), entra con «Compatible con X del EPC»; si el EPC no tiene nada, entra con
«Fuera de catálogo EPC»; sin EPC (sin cuenta o caído), con «Sin verificar en EPC». Si el EPC
tiene otros que no figuran compatibles, se avisa y se elige entre «Agregar el del EPC» y
«Agregar el mío y guardar la compatibilidad» (con nota opcional).

**Compatibilidades de los usuarios** — `POST /compatibilidades/agregadas` (y `DELETE` para
borrar una que quedó mal, solo las del concesionario). Van aparte de la lista de Nissan (que se
reemplaza entera en cada carga), en `user_compatibilities`, en el sentido en que se cargan, con
nota opcional. Valen para el concesionario de quien la cargó; la plataforma, en Administración
→ Catálogo, ve quién la cargó, de dónde, la nota y cuántas veces se ofreció, y decide si vale
para todos. `POST /compatibilidades/consultar` devuelve las de Nissan primero y después las de
los usuarios, con su nota en `agregadas`. «Ver alternativas» aparece siempre en la fila,
aunque no haya compatibles: tiene un campo para sumar otro número que sirve en su lugar, y el
número sumado se puede elegir para cotizar como las demás alternativas.

### 4b. Precio en el cotizador — `POST /precios/consultar`

Todos los importes del Excel de Nissan son **sin IVA**: el **PNC** es el costo del
concesionario y el **SUG**, el precio sugerido de Nissan. En pantalla se muestran con IVA del
21%. El cálculo vive en `app/services/prices.py` (`breakdown` y `dealer_price`).

- **Sin margen** configurado: `SUG × 1,21`.
- **Con margen**: `PNC × 1,21 / (1 − margen)`. El margen de la organización es **sobre el precio
  de venta**, no un recargo sobre el costo: con 37,5%, el costo es el 62,5% del precio. Ejemplo:
  PNC $ 11.794,79 → $ 18.871,66 sin IVA → $ 22.834,71 con IVA. Se redondea a centavos el precio
  sin IVA y el con IVA sale de multiplicarlo por 1,21. Con 0% se cotiza al costo con IVA.
- El margen va de 0% a menos de 100% (con 100% el precio sería infinito): lo valida la API
  (`PATCH /precios/margen`, 422 si no) y una restricción de la base
  (`ck_organizations_dealer_margin_percent`, migración 0011). Hasta noviembre de 2026 era un
  recargo, `PNC × (1 + margen) × 1,21`, y aceptaba hasta 1000%; la migración se detiene si una
  organización tiene un margen fuera del rango nuevo.
- El margen de Nissan es `(SUG − PNC) / SUG`, también sobre el precio de venta. Se toma de la
  columna Cod Venta; si falta, se calcula.
- La estrategia de cotización (más caro, más barato con stock, etc.) ordena por el precio final
  con IVA que manda el backend; el front no recalcula precios. Como el margen es el mismo para
  todos los números de la organización, el orden entre candidatos no cambia con la fórmula.

### 5. Catálogo normalizado (143 repuestos)

Convención: `[Nombre funcional] [Superior/Inferior] [Posición] [Lado]`

Cada repuesto tiene: familias (códigos de 5 chars) y aliases (jerga/sinónimos). Vive en la base, en `catalog_entries` (de todos, sin Row Level Security): la primera lectura de una base nueva la llena con `SEED_CATALOG` de `app/services/matcher_service.py`. Lo cambia solo la plataforma, en Administración → Catálogo: al aprobar una familia candidata, como repuesto nuevo con el nombre editable o al final de las familias de uno que ya está (si varias traen números, gana la primera), o a mano en **Catálogo del matcher** (`GET`/`POST /catalogo/entradas`, `PUT`/`DELETE /catalogo/entradas/{entrada_id}`): se busca por nombre o por el principio de una familia, se cambia el nombre o las familias y su orden, y se suman o borran entradas. Dos entradas no pueden llamarse igual (el catálogo inicial trae algunos nombres repetidos, que quedan hasta que se cambien), el nombre no lleva «|» porque separa el nombre de las familias en el texto que lee la IA, y la última entrada no se borra: sin ninguna, la próxima lectura volvería a llenarlo con `SEED_CATALOG`. El matcher y el asesor usan el cambio desde la próxima cotización. Borrar una entrada no cambia el estado de las candidatas aprobadas en ella. Cuando la corrección que validó el vendedor era un número de pieza, la candidata guarda como nombre técnico la descripción de ese número en la lista de precios, no el número.

El matcher responde con salida estructurada (`output_config.format` con un esquema JSON de `items`): la API garantiza la forma, porque una vez Sonnet devolvió algo que no era el JSON; si igual llega algo que no se puede leer, el texto queda en el log. Reglas del matcher que importan: no se le puede bajar la temperatura (el SDK 2.x ya no acepta `temperature` y los modelos actuales la rechazan), así que lo que lo hace parejo son las reglas: «puerta del izq» a veces salía como no encontrada por la regla vieja de «posición desconocida = no encontrado»; solo asigna un repuesto del catálogo si es exactamente ese (nunca uno parecido); **nunca supone lado ni posición**: si el catálogo separa el repuesto por lado o por posición y el ítem no lo dice, cotiza todas las variantes que faltan, una por ítem con cantidad 1, y lo avisa en el comentario ("espejo" = izquierdo y derecho; "paragolpes" = delantero y trasero; "puerta" sola = las cuatro), sin deducirlo del resto del pedido; "del" y "tras" después del repuesto son la posición ("puerta del izq" = puerta delantera izquierda); un plural o "ambos" de un repuesto que el catálogo no separa por lado (AMORTIGUADOR TRASERO, TERMOSTATO) es un ítem con cantidad 2; y a los repuestos reales que no están en el catálogo les completa `interpretacion` con su nombre técnico.

Si ese ítem con cantidad 2 vino de un plural o de "ambos", sin una cantidad escrita, después de encontrar el número en el EPC el asesor IA revisa si el vehículo lleva dos piezas distintas: en un Tiida con MR18DE, «ambos termostatos» son el principal (21200) y el sub-termostato (21230). Antes de preguntarle al asesor se miran las otras familias de la misma entrada: TERMOSTATO tiene la 21200 (el principal) y la 21230 (el sub-termostato), el EPC las prueba todas y se queda con la primera que trae números; si otra también trae para el vehículo, esa es la otra pieza, sin IA (lo guardado del EPC ya las tiene). Si el matcher no asignó el repuesto, «ambos termostatos» llega entero al asesor (también vale «ambas», «los dos», «las dos» o un plural que el matcher dejó en 2, sin cantidad escrita): si propone dos piezas distintas del catálogo para ir juntas (`seleccion` «varias»), entran las dos directo, una por fila; si elige una del catálogo, se buscan las otras familias de su entrada igual que arriba. Un kit de piezas del catálogo sin «ambos» sigue dejando elegir al vendedor. Si encuentra otro número, el del catálogo queda con cantidad 1 y el otro llega como un ítem aparte, con el mismo pedido (evento `extra_match` del stream, numerado a continuación de los del matcher). Los eventos de `/cotizar/stream` (`phase`, `matches`, `extra_match`, `epc_result`, `done`, `error`) están en el contrato de la API como `QuoteStreamEvent` (`app/schemas/cotizar.py`): el front toma sus tipos de `npm run gen:api`, y los tests del stream validan cada evento contra ese esquema. Cuando el asesor reconoce una pieza del catálogo que el matcher no asignó, busca esas familias en el EPC igual, en lugar de pedirle al vendedor que lo corrija. Un kit, juego o service que agrupa repuestos distintos ("kit de filtros", "kit de distribución") se separa en un ítem por repuesto, todos con el mismo `input` y con `kit` = nombre del kit; si se vende como una sola pieza (KIT JUNTAS REPARACION MOTOR) o son varias unidades del mismo repuesto ("juego de bujías" = bujía × 4), es un solo ítem.

### 6. Asesor IA para repuestos fuera de catálogo

Todo ítem que el catálogo no resuelve (sección 4) va directamente al asesor: `PartAdvisorService` (`app/services/part_advisor.py`) hace un ida y vuelta con un segundo modelo (`ANTHROPIC_ADVISOR_MODEL`, por defecto `claude-opus-5-5`), en paralelo con el resto de la cotización:

1. **Propuesta**: el modelo recibe el texto del cliente y la descripción completa que propuso el matcher: el texto solo puede ser jerga o un kit entero («kit de distribución»), y la descripción dice qué pieza buscar («Tensor de cadena de distribución»). Si el ítem es una pieza de un kit, se le pide solo esa pieza; si el texto del cliente y la descripción no coinciden, sigue al cliente y lo cuenta en `aclaracion`. También recibe el vehículo (del EPC) y los repuestos del catálogo que comparten palabras con el pedido: alguno puede ser la pieza, porque al matcher se le pueden pasar. Si el ítem era del catálogo pero el EPC no tenía números de sus familias, en cambio, se le dice cuáles se buscaron, no se las vuelve a proponer y se buscan otras (puede que en ese vehículo la pieza lleve otra familia o que no la lleve). Hasta el 10 de noviembre de 2026 el asesor recibía solo el texto del cliente, y los ítems sin `interpretacion` o sin números del EPC quedaban «No encontrado» hasta que el vendedor tocaba «Corregir IA». Hasta el 11 de noviembre de 2026 las familias del catálogo se descartaban, con la idea de que el matcher ya las habría encontrado; no era así («pata de motor inferior»: el asesor propuso 11360 y 11320 y quedaron afuera), y ahora se buscan como las demás y, si elige una, va directo. Devuelve familias en una sola de dos listas: `familias_vehiculo` (hasta 6 que podrían ser de este vehículo) o `familias_otros_modelos` (si cree que el vehículo no lleva la pieza: hasta 6 habituales en otros modelos Nissan). Más una `aclaracion` para el vendedor y `es_kit`. Si el nombre sirve para piezas distintas ("volante": de dirección o del motor), propone familias de todas en lugar de elegir una; si es un kit que el matcher no separó, propone las de cada pieza del kit (pueden ser del catálogo).
2. **EPC**: se buscan esas familias para el chasis.
3. **Elección** (misma conversación, con los números completos del EPC): una opción si corresponde una sola pieza, varias con una `pregunta` para el cliente si son piezas distintas (ej. sonda delantera o trasera), o ninguna. Se descartan números que el EPC no devolvió. `seleccion` dice cómo se eligen: `"una"` si son alternativas, `"varias"` si son piezas de un kit; en ese caso el front muestra casillas y las piezas marcadas pasan a la tabla, cada una en su fila.

Si es de `familias_otros_modelos` y el EPC no devuelve nada, se informa que el vehículo probablemente no lleva esa pieza. **Si elige una sola pieza y su familia es del catálogo** (el matcher no la reconoció: «parrilla» es PARRILLA RADIADOR PARAGOLPES), el ítem entra directo como ese repuesto, con el nombre de la entrada (`nombre` en `epc_result` y en `/cotizar/asesor`) y sin sugerencia: no se valida ni queda candidata. Si ofrece varias, cada opción del catálogo lleva el nombre de su entrada (`catalogo`) y, al elegirla, pasa a la tabla sin validar. En el front, el resto de estos ítems van en su propia sección "Asesor IA", debajo de la tabla, con la interpretación, el resultado, la confianza, las opciones y los botones **Validar** / **Descartar**: validar pasa el ítem a la tabla de repuestos identificados y llama a `POST /catalogo/candidatos`, que guarda la familia en la tabla `catalog_candidates` como **candidata pendiente** y cada validación en `catalog_validations` (qué pidió el cliente, la interpretación, el chasis, los números, la organización y el usuario). Los números validados quedan además entre los consultados del chasis. Una familia que ya está en el catálogo no queda candidata (`POST /catalogo/candidatos` devuelve null y solo anota los números); en la lista de candidatas, las que ya están en el catálogo dicen en qué entrada (`en_catalogo`) y **Aprobar** solo cambia su estado, sin el diálogo.

**Revisar las candidatas** — Administración → **Catálogo**, solo el administrador de plataforma (`GET /catalogo/candidatos?estado=…` y `PATCH /catalogo/candidatos/{familia}`). Lista las familias con su descripción, cómo las pidieron, el nombre técnico y cuántas veces se validaron, filtradas por estado (pendientes por defecto). «Ver validaciones» muestra cada una: cuándo, el chasis, el número validado, el pedido y quién (las últimas 50). Las validadas antes del 10 de noviembre de 2026 solo tienen las listas juntas de la candidata, sin el par chasis y número. **Aprobar**, **Descartar** y **Volver a pendiente** guardan el estado con quién y cuándo. Aprobar la suma al catálogo del matcher (ver arriba); volver a pendiente o descartar una aprobada no la saca: eso se hace en Catálogo del matcher.

**Pedidos ambiguos.** Si el texto nombra piezas distintas y el pedido no aclara cuál, el matcher no elige: lo marca con `ambiguo: true` y lo manda al asesor (aunque alguna esté en el catálogo), que ofrece una opción por pieza con su `pregunta`. En ese caso el asesor sí puede proponer familias del catálogo.

**Corregir al asesor** — `POST /cotizar/asesor`. El vendedor puede corregir cualquier ítem (una sugerencia del asesor o un repuesto que el matcher asignó mal): escribe qué pieza es ("es la pantalla del electroventilador") y el asesor vuelve a hacer el ida y vuelta con todas sus indicaciones y las familias que ya descartó. Como en la primera consulta, puede proponer familias del catálogo. Al validar, la última indicación se guarda como `interpretacion` de la candidata.

### 7. Observabilidad — `/salud`, `/monitoreo/*`, `/telemetria`

Cada request tiene un código (`request_id`, header `X-Request-ID`) que viaja por los logs, la base y las trazas; el front lo muestra en los errores del servidor («Código para soporte: …»). El detalle completo está en [OBSERVABILIDAD.md](OBSERVABILIDAD.md); en resumen:

- **Logs:** consola legible y `logs/cotizador.log` en JSON con rotación diaria, con secretos tapados. Se escriben desde otro hilo.
- **Requests:** `ObservabilityMiddleware` (ASGI puro) mide headers, primer byte y final, agrega `Server-Timing` y deja una fila en `request_log` con las etapas (`step()`) y los datos de negocio (`annotate()`).
- **Errores:** el usuario recibe un mensaje claro; el detalle técnico queda en el log y en `request_log.error`. Las excepciones escritas para el vendedor heredan de `UserFacingError`.
- **Costos de la IA:** toda llamada a Claude pasa por `llm_call()`: tokens y costo con el modelo que respondió en `llm_calls`, costo por cotización y **tope diario** (`LLM_DAILY_BUDGET_USD`), que por defecto apaga solo el asesor.
- **Cotizaciones:** `quote_runs` guarda el texto del pedido, el chasis y el resultado por ítem, para medir la calidad de la IA y armar casos de prueba.
- **Límites de uso:** por grupo de rutas, por usuario o por IP (`RATE_LIMIT_*`), y un máximo de cotizaciones simultáneas (`MAX_CONCURRENT_QUOTES`). Responden 429 con `Retry-After` y un mensaje para el usuario.
- **Salud y recursos:** `/salud` para un monitor externo, `/salud/detalle` para diagnosticar, monitor del atraso del event loop y muestras de RAM y CPU del backend y de Chromium.
- **Perfiles a pedido:** `scripts/perfil.ps1` usa `profiling.sampling` de Python 3.15 sin reiniciar el backend.
- **Pantalla Monitoreo** (Administración → Monitoreo en el front): rendimiento, costos, calidad de la IA, integraciones, embudo de ventas, recursos, navegador y alertas, más el rastreo de un código de error, de todas las organizaciones o de una. Solo con el permiso `monitoreo.ver`.
- **Alertas:** reglas revisadas cada 5 minutos que avisan en el log y en Monitoreo. El canal externo queda pendiente.
- **Telemetría del navegador:** Web Vitals, errores de JavaScript y tiempos de cotización, en lotes a `POST /telemetria`.
- **Trazas:** OpenTelemetry opcional, apagado por defecto (`uv sync --extra otel`).

Todo está preparado para el login: las tablas tienen `user_id` y alcanza con llamar a `bind_user()`. Qué sumar en ese momento está en [LOGIN_Y_OBSERVABILIDAD.md](LOGIN_Y_OBSERVABILIDAD.md).

---

## Estructura del proyecto

```
matcher-nissan/
├── CONTEXT.md
├── STANDARDS.md
├── backend/
│   ├── app/
│   │   ├── core/
│   │   │   ├── config.py              # pydantic-settings, lee .env
│   │   │   ├── observability/         # contexto del request, logs, middleware, escritor en lote, recursos, trazas
│   │   │   ├── errors.py              # UserFacingError y report_error
│   │   │   ├── http_errors.py         # server_error para los routers
│   │   │   ├── rate_limit.py          # límites de uso y cupos simultáneos
│   │   │   ├── access.py              # acceso a Monitoreo y a la salud detallada
│   │   │   └── llm_pricing.py         # precios de Claude y costo de cada llamada
│   │   ├── routers/
│   │   │   ├── match.py               # POST /match
│   │   │   ├── epc.py                 # POST /epc  (singleton scraper aquí)
│   │   │   ├── cotizar.py             # POST /cotizar
│   │   │   ├── health.py              # GET /salud y /salud/detalle
│   │   │   ├── monitoring.py          # GET /monitoreo/*
│   │   │   └── telemetry.py           # POST /telemetria
│   │   ├── schemas/
│   │   │   ├── match.py               # MatchRequest, MatchResult
│   │   │   ├── epc.py                 # EPCRequest, EPCResult
│   │   │   └── cotizar.py             # CotizarRequest, CotizarResult
│   │   └── services/
│   │       ├── matcher_service.py     # lógica Claude + catálogo
│   │       ├── epc_scraper.py         # sesión EPC (Playwright) + consultas a su API
│   │       ├── sap_stock.py           # login SAP (Playwright) + stock por httpx
│   │       ├── cotizar_service.py     # orquesta matcher + scraper
│   │       ├── llm_usage.py           # llm_call(): costo de cada llamada y tope diario
│   │       ├── quote_runs.py          # registro de cada cotización
│   │       ├── monitoring.py          # cálculos de la pantalla Monitoreo
│   │       ├── alerts.py              # reglas de alertas y canales de aviso
│   │       └── telemetry.py           # guarda lo que mide el navegador
│   ├── scripts/perfil.ps1             # perfiles del backend en vivo
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
uv run --extra otel python -m app.server   # --recargar para reiniciar al guardar
```

O los dos juntos: `.\scripts\desarrollo.ps1` (en `Cotizador_BE`). Productivo: [PRODUCCION.md](PRODUCCION.md).

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
EPC_URL=https://login.superservice.com/login/es-es/
EPC_SESSION_DIR=~/.epc_session
EPC_HEADFUL=false          # true para ver el browser (debug)
DATABASE_URL=postgresql+psycopg://cotizador_dev_app:cotizador_dev_app@127.0.0.1:5432/cotizador_dev
SYSTEM_DATABASE_URL=...    # la misma base con el usuario de sistema (se saltea RLS)
MIGRATIONS_DATABASE_URL=... # la misma base con el dueño de las tablas
CREDENTIALS_KEY=...        # cifra las cuentas de EPC, SAP y el localizador de cada usuario
LLM_DAILY_BUDGET_USD=1     # tope de gasto diario en Claude; 0 = sin tope
LOG_FILE=logs/cotizador.log  # vacío: solo consola
```

Las variables de observabilidad (logs, límites de uso, alertas, retención, trazas) tienen valores por defecto y están explicadas en `.env.example` y en [OBSERVABILIDAD.md](OBSERVABILIDAD.md#18-configuración-completa).

---

## Base de datos

Postgres 18, con SQLAlchemy 2, el driver psycopg y migraciones de Alembic en `migrations/`. En desarrollo es el Postgres instalado en Windows (servicio `postgresql-x64-18`, puerto 5432, solo acepta conexiones de esta PC): la base `cotizador_dev`. Los tests crean y vacían las suyas (`cotizador_test` y `cotizador_migraciones_test`). Todas usan la collation `es-AR`, que se elige al crear la base y no se puede cambiar después. Productivo usa la base `cotizador`, con sus propios usuarios. El backend aplica las migraciones al arrancar (`prepare_database()` en el lifespan de `main.py`) y, si Postgres no responde, no arranca y dice cómo levantarlo.

Un usuario de Postgres para cada cosa (`app/database/engine.py`; los crea `scripts/preparar_postgres.py`):

- **La app** (`DATABASE_URL`, `session_factory`): solo filas, con Row Level Security. Cada transacción dice su organización (`app/database/tenant.py`), y Postgres deja ver y escribir solo sus filas; sin organización, nada. En desarrollo, `cotizador_dev_app`.
- **El sistema** (`SYSTEM_DATABASE_URL`, `system_session_factory`): se saltea RLS. Solo para lo que no es de una sola organización (el login, el agente del DMS, las alertas, el precalentado, la consola y la plataforma); un test controla qué módulos lo usan. En desarrollo, `cotizador_dev_sistema`.
- **El dueño de las tablas** (`MIGRATIONS_DATABASE_URL`, `owner_session_factory`): migraciones, particiones mensuales y limpieza de lo vencido. En desarrollo, `cotizador_dev`.

Los permisos sobre las tablas los da el backend al actualizar la base (`app/database/roles.py`): la app y el sistema solo tocan filas, y en `audit_log` solo agregan y leen. Una tabla nueva en `app` se clasifica: con RLS (su política va en la migración que la crea, como en `0007_row_level_security`) o de todos; `tests/integration/test_row_level_security.py` falla hasta que se decide. Un hilo propio (`threading.Thread`, `ThreadPoolExecutor`) no hereda la organización: usar `asyncio.to_thread` o decirla con `as_organization`. Las tablas están en `app/models/` y el acceso en los repositorios de `app/database/`, que son sincrónicos y los routers llaman con `asyncio.to_thread`.

Hay dos schemas: `app`, con el negocio (el de todos los modelos que no digan otro), y `observability`, con la telemetría. `public` no existe: la migración inicial lo elimina, así que toda tabla y toda consulta dice su schema (`app.estimates`). En `psql`, `SET search_path TO app, observability;` evita escribirlo en cada consulta. Una extensión nueva se instala en un schema explícito (`CREATE EXTENSION pg_trgm SCHEMA app`).

La historia anterior (24 migraciones, de la época de SQLite) quedó en git: `0001_esquema_inicial` es el punto de partida en Postgres. El 6 de noviembre de 2026 la instalación pasó a Postgres arrancando de cero: los datos de SQLite no se copiaron y el 8 de noviembre se borró `data/cotizador.db`.

Las tablas sin prefijo están en `app`.

| Tabla | Qué guarda |
|---|---|
| `catalog_candidates` | Familias que sugirió el asesor y validó el vendedor, con su estado (pendiente, aprobada o descartada) y quién la revisó. |
| `catalog_validations` | Cada validación de una candidata: pedido, interpretación, chasis, números, organización y usuario. |
| `vehicle_parts` | Números de pieza cotizados para cada chasis de una organización, con su descripción y la última consulta (`VEHICLE_PARTS_DAYS`). |
| `dms_stock` | Última foto del stock del taller, por código y depósito, con la ubicación si el agente la manda |
| `dms_stock_uploads` | Cada subida del agente del taller: cuándo llegó, cuántos repuestos y unidades (30 días) |
| `estimates` | Presupuestos formales de una organización, con la sucursal donde se hicieron y quién los hizo y aceptó: número (único en la organización), fecha, validez, cliente, datos del concesionario al emitirlo, total con IVA, compañía de seguros, tareas de chapa y pintura y lo acordado al aceptar (franquicia, inspección final, pago del seguro) |
| `estimate_lines` | Renglones de cada presupuesto, en orden, con el precio unitario con IVA, la disponibilidad y si quedó excluido al aceptar |
| `insurers` | Compañías de seguro de cada organización para elegir en los presupuestos |
| `estimate_insurance_orders` | La orden del seguro (PDF) de cada presupuesto aceptado |
| `estimate_orders` | Excel confirmados, contenido y costo congelados, número de pedido y anulaciones |
| `estimate_supplies` | Lo pedido o apartado por ítem: origen (pedido o stock), Nº de pedido, costo con IVA, fechas de llegada y separación, anulaciones |
| `loans` | Registros de préstamos o pedidos de repuestos de cada sucursal: pieza, cantidad, concesionario, fechas y estado |
| `observability.request_log` | Cada request: ruta, estado, tiempos, etapas, atributos y error |
| `observability.dependency_events` | Logins y tomas de licencia de EPC, SAP y el localizador |
| `llm_calls` | Cada llamada a Claude: modelos, tokens, duración, costo y error |
| `quote_runs` | Cada cotización: texto del pedido, chasis, resultado por ítem, tiempos y costo |
| `observability.runtime_samples` | Muestras de RAM, CPU, procesos de Chromium y atraso del event loop |
| `observability.alert_events` | Alertas disparadas, repetidas y resueltas |
| `observability.frontend_events` | Web Vitals, errores de JavaScript y tiempos medidos en el navegador |

Convenciones del esquema, en `app/models/base.py`:

- **Fechas:** `timestamptz`, siempre en UTC (`UTCDateTime`, que rechaza fechas sin zona horaria).
- **JSON:** `JSONB` (`Json`). Un `None` es NULL de SQL y las claves no conservan su orden: lo que depende del orden va en una lista.
- **Claves primarias numéricas:** `BIGINT` con `IDENTITY`.
- **Costos y existencias:** `numeric`, nunca `float`: se suman y tienen que dar exacto.
- **Restricciones:** con nombres fijos (`uq_estimates_number`, `fk_...`), para que una migración pueda referirse a ellas. Los estados llevan un `CHECK` con los mismos valores que los `Literal` de `app/schemas/` (un test comprueba que coinciden).
- **Índices:** solo los que usa una consulta. Cada uno se paga en cada escritura.

Para cambiar una tabla: modificar el modelo y crear la migración con `uv run alembic revision --autogenerate -m "descripcion"`. Un test falla si un modelo no coincide con las migraciones.

### Qué va en la base y qué no

| Qué | Dónde | Por qué |
|---|---|---|
| Presupuestos, pedidos, stock, precios, costos de la IA (`llm_calls`) y cotizaciones (`quote_runs`) | `app` | Son datos del negocio: se respaldan y van a tener organización |
| Requests, integraciones, recursos, alertas y telemetría del navegador | `observability` | Se pueden perder sin afectar al negocio: no tienen foreign keys y sus datos no entran en los respaldos. Ver [OBSERVABILIDAD.md](OBSERVABILIDAD.md#15-tablas-retención-y-consultas-útiles) |
| Logs de texto | Consola y `logs/` | Nunca en la base. En la nube, `LOG_FILE` vacío: el proveedor junta la consola |

Si el volumen algún día lo justifica, la telemetría puede pasar a otra base o a una herramienta dedicada (Grafana, Sentry) sin tocar el negocio: no tiene foreign keys y ninguna consulta la cruza con las tablas de `app`. Hoy usa la misma base que el negocio (`session_factory` para escribir y leer, y el dueño de las tablas para la limpieza diaria). Para llevarla a otra base hace falta una segunda URL, y que el writer y la limpieza diaria, que también escriben y limpian `llm_calls` y `quote_runs`, elijan la conexión según el schema de la tabla. Para una herramienta dedicada, además, Monitoreo tendría que leer de ella.

Dos escrituras están pensadas para muchos usuarios:

- **Stock del taller:** el agente sube la foto entera cada 5 minutos, pero solo se escribe lo que cambió. Borrar y reinsertar todo dejaría una fila muerta por repuesto en cada subida.
- **Número de presupuesto:** se asigna con un bloqueo de Postgres (`ESTIMATE_NUMBER_LOCK`), que se libera al terminar la transacción. Con reintentos, varios vendedores simultáneos pueden agotarlos.

### Lo que cambia con organizaciones, sucursales y usuarios

El plan está en [USUARIOS_Y_PERMISOS.md](USUARIOS_Y_PERMISOS.md) («De quién es cada dato»). Desde
la etapa 2 cada dato dice de quién es: los presupuestos y las aseguradoras, de la organización;
los préstamos, los precios del taller, los vehículos recientes y los depósitos con su stock, de
la sucursal; la lista de precios y las compatibilidades de Nissan, compartidas. El perfil del
concesionario se repartió entre la organización (razón social, CUIT y logo) y la sucursal
(dirección, teléfono, mail y hora de mano de obra). Las tablas de observabilidad suman
`organization_id`, para filtrar Monitoreo.

Desde la etapa 6, Postgres repite ese filtro con Row Level Security (ver arriba): el backend atiende los pedidos con un usuario que no es el dueño de las tablas, porque RLS no se aplica al dueño (no se fuerza) ni a un superusuario.

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

7. **Observabilidad sin infraestructura nueva** — los datos quedan en Postgres, en un schema aparte del negocio, y en `logs/`, y se ven en la pantalla Monitoreo. Medir no frena: logs y registros se escriben desde otros hilos, y si la base falla se pierden métricas pero la app sigue andando. OpenTelemetry queda listo para sumar Grafana sin tocar el código.

8. **Límites de uso en memoria** — alcanza porque el backend corre en un solo proceso (las sesiones de Playwright lo obligan). Con varios procesos habría que pasar los contadores a Redis.

9. **El tope de la IA no frena el mostrador** — al llegar al tope se apaga el asesor (Opus, lo caro) y el matcher sigue: se puede seguir cotizando lo que está en el catálogo.

---

## Próximos pasos en orden

1. **Modelo de catálogo en la base** (la base ya existe: Postgres + Alembic) con el modelo diseñado
2. **Cache de resultados EPC** en tabla `applicability`
3. **Tests unitarios** para `MatcherService` y `CotizarService` con mocks
4. **Pre-commit hooks** (ruff + pytest)
5. **Detección de repuestos en conjunto** via `part_composition`
6. **Equivalentes** via `spare_part_alternative`
7. **Login** de usuarios, con la observabilidad por usuario (ver [LOGIN_Y_OBSERVABILIDAD.md](LOGIN_Y_OBSERVABILIDAD.md))
8. **Canal de alertas** (Telegram, email o Slack; ver [OBSERVABILIDAD.md](OBSERVABILIDAD.md#12-alertas))
9. **Optimizar el costo de la IA** con los datos de Monitoreo: caché de prompts, esfuerzo y modelo, evaluados con casos de `quote_runs`
