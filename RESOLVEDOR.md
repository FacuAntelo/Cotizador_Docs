# El resolvedor: cotizar sin IA

Desde noviembre de 2026 el cotizador reconoce los repuestos del pedido en el navegador,
mientras se escribe, y solo manda a la IA lo que no reconoció. El objetivo es casi no usar el
matcher y que el repuestero cotice más rápido. Código: `Cotizador_FE/src/features/resolver/`,
`Cotizador_BE/app/routers/resolver.py` y `app/services/resolver.py`.

## Cómo se lee el pedido

Los repuestos van separados por **comas o enters**, indistintamente. Cada uno se resuelve por
separado y queda en uno de estos estados, que se ven en el campo y en la línea de abajo
(«3 reconocidos · 1 lo lee la IA»):

| Estado | Cómo se ve | Qué pasa al cotizar |
|---|---|---|
| Reconocido | Píldora verde | Va directo al EPC, sin IA |
| Número de pieza | Píldora azul | Se verifica en el EPC para el chasis, como «Agregar repuesto» |
| Lo lee la IA | Subrayado punteado | Es lo único que recibe el matcher |
| Ya cotizado | Píldora gris | No se vuelve a cotizar |

Qué reconoce, en este orden:

1. **Un número de pieza** (5 + 5 caracteres, con o sin guion, con algún dígito).
2. **La cantidad escrita**: «2 pdi», «x4 bujías», «filtro de aceite x3». Otro número en el
   texto lo lee la IA.
3. **El pedido entero**, si es una abreviatura del usuario, algo que el concesionario ya
   resolvió (alias aprendido) o un alias del catálogo («pdi»). Las abreviaturas del usuario
   mandan sobre lo demás.
4. **El nombre del catálogo o de un alias**, palabra por palabra. Las palabras de la pieza
   tienen que ser exactamente las del nombre, sin sobrar ni faltar: «volante» no es ni el de
   dirección ni el del motor, y lo lee la IA. No distingue mayúsculas, acentos, género ni
   plural. Lo que el nombre lleva entre paréntesis no hace falta pedirlo.
5. **Posición y lado**: delantero / del, trasero / tras, superior / sup, inferior / inf,
   izquierdo / izq, derecho / der, y di, dd, ti, td. «del» es «delantero» solo después de la
   pieza y si no sigue otra palabra de ella («tapa del motor» es una preposición).
6. **«X con Y»**: Y hereda la pieza y la posición de X. «paragolpes del con soportes» son el
   paragolpes delantero y sus dos soportes.
7. **«ambos», «los dos», «todos los»**: van todas las variantes. De un repuesto que el
   catálogo no separa por lado («ambos amortiguadores traseros») son dos unidades, y el
   backend revisa si el vehículo lleva dos piezas distintas, como antes.

**Nunca supone lado ni posición**: si el catálogo separa el repuesto y el pedido no lo dice,
van todas las variantes con la nota «No indicaste el lado: se cotizan todas las variantes».
«puerta» sola son las cuatro.

## Al cotizar

- **Solo lo nuevo.** Lo que ya está en la tabla no se vuelve a cotizar; lo que se agrega al
  texto se suma después. Un repuesto editado entra como nuevo.
- **Borrar texto no quita nada.** Cada fila tiene **Eliminar**, que pregunta «Eliminar» o
  «Eliminar y reportar error» (con un texto para explicarlo). Eliminada la fila, ese repuesto
  se puede volver a cotizar.
- **El backend recibe lo reconocido por id de entrada** (`resueltos` en `POST /cotizar/stream`)
  y busca él las familias: no viajan al navegador. `texto` trae solo lo que tiene que leer la
  IA y puede venir vacío. Los ítems llegan con `local: true`.
- **«Agregar repuesto»** acepta un número (lo busca en la lista de precios, como antes) o una
  descripción: con Enter se suma al pedido de arriba y se cotiza. Pueden ser varios, con comas.

## Mientras se escribe

- **Sugerencias**: hasta cuatro repuestos cuyas palabras empiezan con lo escrito, primero los
  que más cotiza el concesionario. `Tab` acepta, `↑` `↓` elige, `Shift+Enter` marca varios y
  `Tab` los acepta, `Shift+Tab` acepta todos, `Esc` cierra. Con el mouse, clic acepta y
  Ctrl+clic marca.
- **Recomendaciones**: debajo del campo. Según lo que ya está en el pedido y en la tabla: lo
  que el concesionario suele cotizar junto, las piezas que van con esa (los soportes de ese
  paragolpes), la del otro lado y las de la misma zona. Sin nada cargado, lo más cotizado,
  una por tipo de pieza. Clic la suma al pedido.
- **Pre-cotización**: 0,7 s después de reconocer un repuesto, con un vehículo elegido, se
  buscan sus familias en el EPC (`POST /cotizar/precotizar`). La respuesta queda en lo guardado
  del EPC, que es de todos; no toca el historial del chasis ni cuenta como cotización.

## Lo que aprende

`learned_aliases`, por concesionario (Row Level Security). Se guarda sola, sin validación:

- **Al terminar una cotización**, lo que la IA resolvió con el catálogo y tuvo números en el
  EPC: el pedido, sin mayúsculas ni acentos, queda apuntando a esas entradas. Un pedido se
  aprende entero o nada («espejo» son el izquierdo y el derecho: si uno no tuvo números, no se
  guarda). No se aprende lo que llevó una cantidad, lo ambiguo, lo que ya resolvió el
  cotizador, ni la pieza que «ambos» sumó después.
- **Al validar una sugerencia del asesor**: el pedido queda apuntando a esa familia. Si la
  familia todavía no está en el catálogo, se cotiza por la familia con la descripción del
  asesor; cuando la plataforma la aprueba, pasa a valer su entrada.
- **«Corregir IA»** olvida lo aprendido para ese pedido. Si la corrección da con una pieza del
  catálogo, esa es la que se aprende; si da una sugerencia, se aprende al validarla.
- **«Eliminar y reportar error»** olvida lo aprendido para ese pedido y ese repuesto.
  «Eliminar» a secas no olvida nada: el cliente puede simplemente no querer la pieza.

**Abreviaturas** (`user_abbreviations`), por usuario: botón «Abreviaturas» sobre el pedido. El
repuestero escribe la abreviatura y elige del catálogo, por descripción, uno o varios repuestos
(«pds» = paragolpes delantero y sus dos soportes). Las generales ya vienen como alias.

**Frecuencia y lo que va junto**: sale de los repuestos cotizados para cada chasis
(`vehicle_parts`) del último año, por concesionario, y se recalcula cada 10 minutos.

## Lo que queda registrado

- `quote_runs.items_local`: cuántos ítems de cada cotización se resolvieron sin IA. De cada
  ítem se guarda la respuesta completa del asesor (aclaración, explicación, pregunta y cada
  propuesta con sus números).
- `quote_item_feedback`: lo que hizo el repuestero con cada ítem (validado, descartado,
  corregido, eliminado, reportado, con su explicación).
- `catalog_validations.advisor_text`: lo que dijo el asesor de lo validado. Se ve en
  Administración → Catálogo → Ver validaciones.
- **Monitoreo → Sin resultado** (`GET /monitoreo/sin-resultado`): porcentaje resuelto sin IA,
  y de cada repuesto que no quedó resuelto, qué escribió el repuestero, qué entendió el
  matcher, qué respondió el asesor y qué hizo después. Lo que el asesor encontró y el
  repuestero validó cuenta como encontrado. Aparte, los errores reportados.

## Límites conocidos

- **La pre-cotización adelanta solo el EPC.** El precio y el stock se siguen pidiendo al
  cotizar.
- **Un plural solo** de un repuesto sin lado («amortiguadores traseros», sin «ambos») lo lee
  la IA: no se sabe si son dos unidades.
- **No tolera errores de tipeo**: «paragolpe» sí (singular), «paragolpez» no.
- **Lo ambiguo no tiene estado propio**: lo que nombra piezas distintas lo lee la IA, que lo
  marca ambiguo como antes.
- **Las recomendaciones** usan lo cotizado por chasis. Los presupuestos guardados y las ventas
  todavía no entran.
- **Los alias aprendidos son de cada concesionario.** No hay pantalla para que la plataforma
  los vea o los generalice.
- **`POST /cotizar` (sin stream)** no usa `resueltos`: el front usa solo el stream.
