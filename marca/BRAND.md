# Marca OEMIA

Resumen de uso de la marca. La versión para mostrar y compartir es [guia-de-marca.pdf](guia-de-marca.pdf), y las aplicaciones están en [tablero-marca.html](tablero-marca.html) y en `tablero/`.

## La idea

OEM por los números de pieza originales, IA por el asesor que entiende el pedido. El símbolo es una tuerca con forma de globo de chat y una chispa en el centro: el repuesto y la conversación con el cliente en una sola forma. La O de la palabra repite la tuerca.

## Archivos

Los originales están en `Cotizador_FE/brand/logo/`, en SVG:

| Versión | Archivo | Cuándo usarla |
| --- | --- | --- |
| Horizontal | `oemia-horizontal-*.svg` | Versión principal |
| Apilada | `oemia-stacked-*.svg` | Espacios cuadrados |
| Símbolo | `oemia-symbol-*.svg` | Ícono y avatar |
| Símbolo chico | `oemia-symbol-small-*.svg` | Debajo de 24 px |
| Palabra | `oemia-wordmark-*.svg` | Cuando el símbolo ya está cerca |
| Ícono de app | `oemia-app-icon.svg` | Instalación y accesos directos |

Cada versión viene en `color`, `on-blue`, `on-dark`, `black` y `white`. La pestaña del navegador usa `favicon.svg`, en azul fijo con base blanca para que se vea en pestañas claras y oscuras.

## Espacio y tamaño

Dejá libre una distancia X en todos los lados del logo, donde X es la mitad de la altura del símbolo.

| Versión | Pantalla | Impreso |
| --- | --- | --- |
| Horizontal | 96 px de ancho | 25 mm de ancho |
| Apilada | 64 px de ancho | 18 mm de ancho |
| Palabra | 64 px de ancho | 18 mm de ancho |
| Símbolo | 24 px | 6 mm |
| Símbolo chico | 16 px | No se usa |

## Color

| Nombre | HEX | RGB | CMYK aproximado | Uso |
| --- | --- | --- | --- | --- |
| Azul carbónico | `#25227A` | 37 34 122 | 70 72 0 52 | Logo y barra superior |
| Rojo numerador | `#E5391C` | 229 57 28 | 0 75 88 10 | Chispa e "IA" de la palabra |
| Índigo de acción | `#2F2BB0` | 47 43 176 | 73 76 0 31 | Botones y enlaces |
| Lavanda | `#EEEDF9` | 238 237 249 | 4 5 0 2 | Fondo de página |
| Tinta | `#16163A` | 22 22 58 | 62 62 0 77 | Texto |

Los valores CMYK son una conversión directa. Antes de imprimir hay que ajustarlos con la imprenta y elegir los Pantone, que todavía no están definidos.

**Copias del remito:** blanco para En stock, canario `#FFF2AE` para Disponible en planta y rosado `#FFD9DF` para Diferido. Estos colores significan disponibilidad y nada más, y siempre van con forma y texto.

## Tipografía

- **Barlow:** títulos, texto, importes y datos.
- **JetBrains Mono:** solo lo que se compara carácter por carácter: números de pieza, chasis y presupuestos.

Las dos tienen licencia SIL Open Font. La palabra del logo está dibujada: no se escribe con ninguna fuente.

## Voz

- Usa las palabras del mostrador: Diferido, En stock, Disponible en planta, paños, franquicia.
- Habla en español rioplatense y va al grano.
- No vende la IA: el asesor propone y el vendedor decide.

## Qué no hacer

No estirar, rotar ni recolorear el logo. No agregarle sombras ni efectos. No usarlo sobre fondos cargados. No reescribir la palabra con otra letra.

## Pendiente

- **Búsqueda de marca:** hay que hacer una búsqueda profesional antes de cualquier uso público. Esta guía no la reemplaza ni asegura que el nombre o el logo estén disponibles.
- **Pantone:** definirlos con la imprenta si se va a imprimir.
- **Tablero:** las tarjetas, el cartel y el sticker son simulaciones para ver el logo en contexto. No son piezas producidas.

## Relación con el producto

El sistema visual completo de la app está en `Cotizador_FE/DESIGN.md` y el movimiento en `Cotizador_FE/MOTION.md`. El presupuesto formal no usa esta marca: es un documento del concesionario, con sus datos y su logo.
