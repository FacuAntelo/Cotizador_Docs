# El resolvedor: cotizar sin IA

Desde noviembre de 2026 el cotizador reconoce los repuestos del pedido en el navegador,
mientras se escribe, y solo manda a la IA lo que no reconoció. El objetivo es casi no usar el
matcher y que el repuestero cotice más rápido. Código: `Cotizador_FE/src/features/resolver/`,
`Cotizador_BE/app/routers/resolver.py` y `app/services/resolver.py`.

## Cómo se lee el pedido

Los repuestos van separados por **comas o enters**, indistintamente. Cada uno se resuelve por
separado. Al cerrarlo con una coma, Enter o **Tab** queda en el recuadro como un **elipse con su
cruz**, y lo que se está escribiendo va al final. El estado se ve en el elipse y en la línea de
abajo, que cuenta solo «identificados · números de pieza · ya cotizados» (en pantalla no se
habla de IA):

| Estado | Cómo se ve | Qué pasa al cotizar |
|---|---|---|
| Identificado | Elipse verde, siempre con tilde | Va directo al EPC, sin IA |
| Número de pieza | Elipse azul, con tilde si la lista de precios lo tiene | Se verifica en el EPC para el chasis, como «Agregar repuesto» |
| Sin identificar | Elipse de borde punteado | Es lo único que recibe el matcher |
| Ya cotizado | El mismo elipse | No se vuelve a cotizar |

- **Un texto que nombra varios repuestos se abre en un elipse por repuesto** al cerrarlo,
  como una abreviatura: «2 puertas del» queda «2 puerta delantera derecha» y «2 puerta
  delantera izquierda». Solo si todos son del catálogo y cada nombre vuelve a dar ese mismo
  repuesto; lo ya cotizado no se abre (cambiaría su clave y se volvería a cotizar).
- **El mismo repuesto no entra dos veces**: mismo texto, mismas piezas con la misma cantidad
  o mismo número. Queda el primero y el repetido se descarta sin aviso.
- **Retroceso con el campo vacío no hace nada**: para sacar un repuesto está su cruz.
- **Alto**: antes de la primera cotización el recuadro crece con el pedido; después tiene alto
  máximo y su barra, y queda abajo de todo para seguir agregando.

El pedido sigue siendo un solo texto: `requestText.ts` lo cierra y le saca un repuesto, y
`requestClosing.ts` abre los de varios repuestos y saca los repetidos. Los elipses son cómo se
muestra.

Qué reconoce, en este orden:

1. **Un número de pieza** (5 + 5 caracteres, con o sin guion, con algún dígito), si la lista
   de precios lo tiene: se consulta `GET /precios/repuestos` por cada uno. Si la lista no lo
   tiene, es «Sin identificar» y al cotizar va como texto, no como fila. Sin lista cargada o si
   la consulta falla, se toma como número, como antes.
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
- **La cruz de un elipse lo saca del recuadro, no de la tabla.** Si era un repuesto ya
  cotizado, su fila se va recién al tocar **Cotizar** de nuevo (sin popup; queda anotado como
  eliminado). Si sigue pedido de otra forma («pdi» fuera, pero queda su nombre), la fila sigue.
- Cada fila tiene además **Eliminar**, que pregunta «Eliminar» o «Eliminar y reportar error»
  (con un texto para explicarlo). Eliminada la fila, ese repuesto se puede volver a cotizar.
- **Nunca dos filas iguales.** El mismo repuesto escrito de dos formas, o que ya está en la
  tabla, no se manda de nuevo; y el backend saca lo que el matcher devuelve repetido
  (`_without_duplicates`: mismo repuesto del catálogo o mismo pedido sin catálogo).
- **Lo que resuelve el asesor IA también está en la tabla**, al final: lo que se pidió, en qué
  quedó (consultando, no encontrado, sugerido por IA y falta validar, descartado) y «Ver en
  Asesor IA», que baja a su tarjeta. Validado, pasa a ser una fila más.
- **El backend recibe lo reconocido por id de entrada** (`resueltos` en `POST /cotizar/stream`)
  y busca él las familias: no viajan al navegador. `texto` trae solo lo que tiene que leer la
  IA y puede venir vacío. Los ítems llegan con `local: true`.
- **«Agregar repuesto»** acepta un número (lo busca en la lista de precios, como antes) o una
  descripción: con Enter se suma al pedido de arriba y se cotiza. Pueden ser varios, con comas.
- **Números que el EPC no reconoce para el chasis**: si para un número escrito el EPC tiene
  otro que no figura compatible, sale un cartel por cada número del EPC, de a uno, con
  «1 de N» en el título. Los escritos que dan el mismo número del EPC (dos de la misma pieza)
  van en el mismo cartel: «Agregar el del EPC» suma ese una vez, y «Agregar los míos» suma los
  escritos y guarda la compatibilidad de cada uno. «No agregar» pasa al siguiente
  (`differentNumbers.ts`).

## Mientras se escribe

- **Abreviaturas**: al escribir una y tocar **espacio** (o Tab), se reemplaza por sus
  repuestos, cada uno en su elipse con el nombre completo: «sds» → «soporte delantero
  izquierdo», «soporte delantero derecho». Valen las del usuario, las globales y los alias de
  una sola palabra que no nombran ninguna pieza («pdi»).
- **Sugerencias**: hasta cuatro repuestos cuyas palabras empiezan con lo escrito, primero los
  que más cotiza el concesionario. La palabra que se está escribiendo nunca se ignora: «para»
  es el principio de «paragolpes». `Tab` acepta, `↑` `↓` elige, `Shift+Enter` marca varios y
  `Tab` los acepta, `Shift+Tab` acepta todos, `Esc` cierra. Con el mouse, clic en el nombre
  acepta y clic en la casilla (o Ctrl+clic) marca.
- **Tab**, en orden: expande la abreviatura escrita; si hay sugerencias, acepta; si no, cierra
  lo escrito como un elipse; con el campo vacío pasa al siguiente control.
- **Números de pieza**: si el catálogo no sugiere nada y lo escrito parece el principio de un
  número (5 caracteres o más, con algún dígito), se ofrecen hasta seis de la lista de precios,
  con su descripción. Es la misma búsqueda que «Agregar repuesto».
- **Sin recomendaciones**: desde noviembre de 2026 debajo del campo no se ofrece nada («Lo
  más pedido» ni «Suele ir con»): no acertaban. `recommend()` sigue en `resolver.ts`, con sus
  tests, por si vuelven.
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
(«pds» = paragolpes delantero y sus dos soportes). El diálogo muestra arriba las del sistema:
las globales, las de posición y lado, y las siglas del catálogo («pdi», «gtd»). Los sinónimos
de una palabra («sapito») y los alias largos reconocen igual, pero no se listan.

**Abreviaturas globales** (`global_abbreviations`, de todos, sin Row Level Security): las carga
el administrador de plataforma en Administración → Catálogo. Lo que el sistema ya entiende para
todos son esas, las de posición (del, tras, izq…) y los alias del catálogo:

- Un usuario **no puede cargar** una que el sistema ya tiene (409 `ya_es_global`).
- Si después aparece una global **igual** a la de un usuario (mismos repuestos), la del usuario
  se borra la próxima vez que lee el catálogo: pasa a usar la de todos.
- Si aparece una global **distinta**, la del usuario sigue valiendo para él (manda sobre la
  global) y vuelve con `conflicto`: se le muestra en rojo, con la recomendación de quitarla.
- Una global no puede repetir un alias del catálogo (409 `ya_es_alias`): eso se cambia desde
  la entrada.

Las primeras globales son `pr` (parabrisas), `pgd` y `pgt` (paragolpes delantero y trasero) y
`pgds` y `pgts` (cada paragolpes con sus dos soportes). Una base que ya tenía catálogo las
recibe de la migración 0015; una nueva, al sembrar el catálogo (`SEED_GLOBAL_ABBREVIATIONS`).

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
