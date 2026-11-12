# Normalización del catálogo del matcher (noviembre de 2026)

Registro de lo que cambió en el catálogo con la migración `0012` de `Cotizador_BE`, para
revisarlo después con más profundidad. Los cambios se aplican solos al publicar, y solo a las
entradas que siguen como en el catálogo inicial: lo que la plataforma editó a mano no se toca
(queda avisado en el log de la migración).

El catálogo pasó de 424 a 418 entradas, sin nombres repetidos, y cada entrada puede tener
**alias**: otras formas de pedir ese repuesto. Se editan en Administración → Catálogo.

## Para revisar en profundidad

### Entradas que se unieron

Se unieron porque tenían el mismo nombre o eran el mismo repuesto. Si alguna resulta ser una
pieza distinta, hay que volver a separarla en Catálogo del matcher.

| Queda | Familias, en orden | Se unió |
|---|---|---|
| TAPA DE VALVULAS | 11046, 13264 | TAPA DE VALVULAS (13264) |
| JUNTA BOMBA DE AGUA | 11062, 21014 | JUNTA BOMBA DE AGUA (21014) |
| LATERAL DE CARROCERIA DERECHO | 76032, 76022, 76030 | LATERAL CARROCERIA DERECHO (76032) |
| LATERAL DE CARROCERIA IZQUIERDO | 76033, 76023, 76031 | LATERAL CARROCERIA IZQUIERDO (76033) |
| BISAGRA CAPOT DERECHA | 65400, 65470 | AMORTIGUADOR DE CAPOT DERECHO (65470) |
| BISAGRA CAPOT IZQUIERDA | 65401, 65471 | AMORTIGUADOR DE CAPOT IZQUIERDO (65471) |

- **Orden de las familias:** si varias traen números para un vehículo, gana la primera. En las
  unidas quedó primero la familia de la entrada que estaba antes en el catálogo. Falta
  confirmar que ese orden es el correcto en cada una.
- **Bisagra y amortiguador de capot:** son el mismo repuesto; algunos vehículos llevan
  amortiguador en lugar de bisagra. «Amortiguador de capot derecho / izquierdo» quedó como
  alias de la bisagra.

### Termostato: dos piezas distintas

Estaba dos veces con el mismo nombre (familias 21200, 21230 y 21230, 21202). Son dos piezas
que no se reemplazan entre sí, y algunos vehículos llevan las dos (Tiida 1.8 MR18DE):

| Entrada | Familias | Qué es |
|---|---|---|
| TERMOSTATO ENTRADA DE AGUA MOTOR | 21200 | El termostato: en la entrada de agua del motor, con la manguera inferior del radiador. Empieza a abrir a 82 °C (21200-ET01A). |
| TERMOSTATO SALIDA DE AGUA MOTOR | 21230, 21202 | La válvula de control de agua: en la salida de agua del motor, con la manguera superior. Empieza a abrir a 95 °C (21230-EL00B, reemplazada por 21230-6N20A). |

- **La familia 21202** estaba junto a la 21230 y quedó ahí. No se sabe qué pieza es: falta
  verificarla.
- **Consecuencia a revisar:** antes «termostato» traía una sola pieza (la primera familia con
  números). Ahora son dos entradas, y si el pedido no dice cuál, el matcher puede cotizar las
  dos; en un vehículo que lleva una sola, la otra no tiene números en el EPC y pasa al asesor
  IA. No está medido cuántas veces pasa.
- Las temperaturas y la ubicación salen de una consulta a un agente, no del EPC.

### Lado o posición corregidos por deducción

Eran entradas repetidas con familias distintas. El lado se dedujo de la numeración de Nissan
(el número menor suele ser el derecho) y no se verificó en el EPC.

| Familias | Antes | Ahora |
|---|---|---|
| 82430, 82431 | LIMITADOR PUERTA DELANTERA | LIMITADOR PUERTA TRASERA |
| 62256 | EMBELLECEDOR FARO AUXILIAR DELANTERO IZQUIERDO | EMBELLECEDOR FARO AUXILIAR DELANTERO DERECHO |
| 68761 | REJILLA AIRE ACONDICIONADO LATERAL DERECHA | REJILLA AIRE ACONDICIONADO LATERAL IZQUIERDA |
| 96374 | CACHA TAPA ESPEJO DERECHO (SIN PINTAR) | CACHA TAPA ESPEJO IZQUIERDO (SIN PINTAR) |
| 96312, 96318 | CUBRE ESQUINA ESPEJO | CUBRE ESQUINA ESPEJO DERECHO |
| F2520, 62520 | SEDE DE OPTICA DELANTERA IZQUIERDA | SEDE DE OPTICA DELANTERA DERECHA |
| F2521, 62521 | SEDE DE OPTICA TRASERA IZQUIERDA | SEDE DE OPTICA DELANTERA IZQUIERDA |

### La única ambigüedad real: 84890

La familia 84890 está en **LEYENDAS** y en **EMBLEMA BAUL PORTON TRASERO**: con los primeros
cinco caracteres no se puede saber cuál es. Quedó como estaba.

## Correcciones de nombre

Sin cambio de familias: género («PUERTA DELANTERA DERECHO» → «PUERTA DELANTERA DERECHA»),
plurales, ortografía y orden de las palabras (la posición y el lado, al final). Los adjetivos
concuerdan con la pieza que tiene esa posición: «CERRADURA PUERTA DELANTERA IZQUIERDA».

- Puertas (4), molduras de puerta (4), ventanillas (4), burletes (8), cerraduras de puerta (4)
  y cintas calco de marco e interior de puerta (10): «DELANTERO / TRASERO IZQUIERDO / DERECHO»
  → «DELANTERA / TRASERA IZQUIERDA / DERECHA».
- OPTICA DELANTERO IZQUIERDO / DERECHO → OPTICA DELANTERA IZQUIERDA / DERECHA.
- RULEMAN DE RUEDA DELANTERO / TRASERO → DELANTERA / TRASERA; MAZA DE RUEDA DELANTERO →
  DELANTERA.
- SOPORTES PARAGOLPES TRASERO IZQUIERDO / DERECHO → SOPORTE PARAGOLPES TRASERO …
- BISAGRAS CAPOT IZQUIERDO → BISAGRA CAPOT IZQUIERDA; BISAGRA CAPOT DERECHO → DERECHA.
- PASTILLAS FRENO TRASERAS → PASTILLAS DE FRENO TRASERAS.
- VALVULA ADMISION → VALVULA DE ADMISION; VALVULAS ESCAPE → VALVULA DE ESCAPE.
- TAPA DE SUMINSTRO DE COMBUSTIBLE → TAPA DE SUMINISTRO DE COMBUSTIBLE.
- SOPORTE DE ALMA DELANTERO DERECHO / IZQUIERDO → SOPORTE DE ALMA DELANTERA …; SOPORTE DERECHO
  / IZQUIERDO ALMA TRASERA → SOPORTE DE ALMA TRASERA DERECHO / IZQUIERDO.
- BISAGRA IZQUIERDA / DERECHA BAUL PORTON TRASERO → BISAGRA BAUL PORTON TRASERO IZQUIERDA /
  DERECHA.
- SENSOR IZQUIERDO / DERECHO PUNTO CIEGO → SENSOR PUNTO CIEGO IZQUIERDO / DERECHO.

La lista completa, entrada por entrada, es `RENAMED` en
`Cotizador_BE/migrations/versions/0012_alias_y_catalogo_normalizado.py`.

## Alias iniciales

Cincuenta entradas arrancan con alias, sacados de la jerga que ya conocía el matcher y del
mostrador: `pdi`, `pdd`, `pti`, `ptd`, `gdi`, `gdd`, `gti`, `gtd`, «fender» y «overfender»
(molduras de guardabarros), «faro delantero» (óptica), «antiniebla» (faro auxiliar), «sapito»,
«blower», «encauzador de radiador» (deflectores), «pantalla / encauzador de electroventilador»
(bóveda), «retrovisor», «pasarruedas» (guardaplast), «portón», «precap», entre otros. La lista
es `SEED_ALIASES` en `Cotizador_BE/app/services/matcher_service.py`.

Son un punto de partida escrito a mano, sin revisar uno por uno con un repuestero. Conviene
repasarlos en Administración → Catálogo (se busca por alias).
