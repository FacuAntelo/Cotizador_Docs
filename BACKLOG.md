# Backlog

Lo que falta hacer en OEMIA, para ir trabajándolo de a uno. Cada pedido tiene un número fijo
para nombrarlo («hagamos el 3»): los nuevos toman el siguiente y no se renumera. Antes de
anotar uno se aclaran las dudas, y lo decidido queda escrito en el pedido. Cuando se termina,
se borra de acá: queda en el historial de git y en el documento del tema.

## La app

### 19. Agregar repuesto: probarlo con el EPC de verdad

Está hecho: el botón «+ Agregar repuesto», la verificación con el EPC por familia, las
etiquetas, «Agregar al historial» y el aviso con «No volver a mostrar». Falta probar con una
cuenta de EPC que la búsqueda por familia (los primeros 5 del número) traiga todos los números
de esa familia para el vehículo, que es contra lo que se compara el número escrito.

### 21. Cotizar por número de pieza, sin chasis

Pedido el 12/11/2026. **Hecho en `costos` el 14/11/2026, falta probarlo en pantalla y
publicarlo**; cómo funciona, en `CONTEXT.md` («Cotizar sin chasis»).

- Un selector arriba del panel, fuera de él, como en las otras pantallas: «Con chasis» (un auto
  de frente) y «Por N° de pieza» (una caja). La columna del chasis se cierra con una animación
  y el recuadro pasa a «Ingresá los números de pieza». Cambiar de modo no borra nada.
- Cada número se cotiza al cerrarlo (con espacio, coma, Enter o Tab) y va directo a la tabla,
  con lo último arriba. Las opciones salen desde la primera tecla, con casillas y tope de 20.
- Los números quedan como elipses en el recuadro; «Limpiar» (o Ctrl+Backspace) los borra todos.
- La misma tabla sin el EPC, con «Eliminar» sin cartel, «Copiar para
  WhatsApp» y «Generar presupuesto», con su propia hoja.
- Lo que no está en la lista de precios entra con aviso y sin consultar SAP, salvo que se pida.
- Historial de 3 días, compartido por la organización y con «Solo los míos». Se despliega con
  el botón «Historial».
- **A mirar en pantalla:** la animación (en Chrome debería interpolar `grid-template-columns`;
  si no, la columna se cierra de golpe), el recuadro a todo el ancho, la hoja al lado de la
  tabla, 390 px y tema oscuro. El backend de desarrollo tiene que estar reiniciado: sin eso no
  hay `/numeros/recientes` ni búsqueda desde 1 carácter.

### 22. Pegar el pedido del cliente tal cual

Pedido el 17/11/2026. **Hecho en `pegar-pedido` el 17/11/2026 (BE, FE y Docs), falta
publicarlo**; cómo funciona, en `RESOLVEDOR.md`. El repuestero copia el pedido como lo escribió
el cliente y lo pega, sin tiempo de elegir del predictivo. Ejemplo real: «paragolpes del y
soportes, guadabaros di con guqrdaplast y bieletas x2».

Lo que se hizo, con lo decidido:

- **Elipse «con opciones»**: lo que no se identificó pero el predictivo tiene qué ofrecer (un
  error de tipeo, una pieza ambigua). Ámbar, borde lleno de 2px y la flecha de un desplegable,
  contra el rojo punteado del sin identificar. Al tocarlo se elige de su desplegable; si no se
  toca, va a la IA como lo sin identificar. El título dice «Elegí cuál es», sin hablar de la IA.
- **Errores de tipeo**: nunca quedan verdes solos, quedan con opciones.
- **La «y» separa repuestos** como la coma, y «con» pasa la posición y el lado a lo que va con
  la pieza. La cantidad es de cada repuesto.
- **Al pegar, todo queda en elipses**, también el último.
- **«bieletas x2» ya no es un número de pieza**: las palabras no se juntan.
- **Bieletas**: sin más son las delanteras, izquierda y derecha, por alias en el catálogo
  (`SEED_ALIASES` y la migración 0019, que los carga en productivo al publicar). Un alias
  puede estar en los dos lados de un repuesto, no en dos repuestos.
- **Cantidad con el lado sin decir**: se reparte, «2 bieletas» son una de cada lado.

Probado en pantalla el 17/11/2026 con la base de desarrollo: pegar el ejemplo, elegir del
desplegable con Enter, Esc, tema claro y oscuro y 390 px. Sin probar: cotizarlo de punta a punta
(desarrollo no tiene cuenta de EPC) y con el catálogo de productivo, que puede tener otros
nombres y alias que el inicial.

### 23. El asesor valida en la fila de la tabla, no en una tarjeta abajo

Pedido el 17/11/2026. **Hecho en `asesor-en-fila` el 19/11/2026 (BE, FE y Docs), falta
publicarlo.** Probado en pantalla con el EPC y el asesor simulados (un proxy local con las
cuatro respuestas): marcar tocando la fila, agregar varias, cambiar elección, rechazar,
agregar el kit, «¿Por qué?», descartar y reemplazar por número, en tema claro y oscuro y a
390 px. Sin probar con el asesor real.

Ajustes pedidos el 19/11/2026 al probarlo, ya hechos:

- **Para elegir**: el grupo lleva un borde punteado ámbar animado y tres puntos que laten
  (quietos con movimiento reducido), y «N.º a confirmar» en lugar de «—». Tocar la fila la
  marca y se pueden marcar varias; el botón del encabezado dice «Rechazar opciones» (rojo) sin
  nada marcado y «Agregar N» (verde) con algo. Rechazar deja «No encontrado». Lo agregado lleva
  «Cambiar elección», que vuelve al grupo en el lugar del pedido.
- **Indicar el número de pieza** va junto a la descripción, como botón, no al final de la fila.
- **Historial del chasis**: cerrado de entrada, con el título «N repuestos cotizados
  previamente para este chasis»; se abre tocándolo.
- **Enter no acepta del desplegable**: solo Tab. Con texto cierra el elipse tal cual; con el
  recuadro vacío cotiza.
- **Catálogo** (migración 0020): ESPEJO RETROVISOR CENTRAL (INTERIOR), 96321, y TABLERO DE
  INSTRUMENTOS, 24810. «espejo retrovisor» ya no da solo los de puerta: queda para elegir. Si
  en productivo el tablero ya estaba con esa familia, la migración no lo duplica y lo de que
  no aparecía en el predictivo tendría otra causa (sin verificar: no se pudo leer esa base). Antes, lo que resolvía el asesor
iba resumido en la tabla («Sugerido por IA · falta validar» y «Ver en Asesor IA») y todo lo
demás estaba en una tarjeta debajo: poco intuitivo.
Se eligió entre tres maquetas (A, B y C) mostradas en el chat; quedan A y C:

- **Una sola sugerencia (A)**: el número sugerido entra a la tabla como una fila más, con su
  disponibilidad y su precio. Se distingue por la marca «IA» en rojo (el rojo es solo lo que
  propone el asesor) y una línea debajo del nombre: «Sugerido por IA · confianza media ·
  pediste «…»». En la columna de acciones, Validar, Descartar y Corregir IA. «¿Por qué?»
  despliega debajo de la fila «Cómo lo interpretó» y «Resultado en el EPC». «Probablemente no
  la lleva» va en esa misma línea roja.
- **Varias opciones o un kit (C)**: el pedido queda como encabezado de un grupo, con la
  pregunta para el cliente, y debajo una sub-fila por pieza posible con su número,
  disponibilidad y precio, para comparar antes de elegir. **Elegir ya valida**: un clic, sin
  Validar aparte, y se aprende el alias como hoy al validar. En un kit cada pieza lleva casilla
  y «Agregar N» suma las marcadas.
- **Descartar no saca la fila**: queda como «Descartado», con Corregir IA para indicarle al
  asesor qué pieza es y con una opción para **reemplazarla por un número de pieza** escrito a
  mano (como «Agregar repuesto»: se busca en la lista de precios y se verifica en el EPC).
- **Buscando y No encontrado** también quedan en su fila: «Consultando «…»…» con el spinner, y
  «No encontrado» con Corregir IA y el reemplazo por número.
- **Corregir IA** abre su formulario debajo de la fila, como ya pasa con los identificados.
- La tarjeta y la sección del asesor debajo de la tabla desaparecen.

## Integraciones

### 16. Confirmar en productivo que dos usuarios con la misma cuenta de EPC no se molestan

En productivo se abrieron dos sesiones para la misma cuenta de EPC (las cuentas 1 y 5) y se
sacaron la licencia entre ellas. Lo más probable era el usuario escrito con otras mayúsculas:
desde la migración 0010 la huella compara el usuario sin mayúsculas y las huellas de
productivo se recalculan al publicar.

- **Después de publicar:** que con los dos usuarios cotizando haya una sola «sesión nueva para
  la cuenta» en el log de productivo, y que no salte el aviso.
- **Si sigue pasando:** comparar las dos cuentas sin mostrar usuarios ni contraseñas (un script
  que diga solo si son iguales; lo corre el usuario, porque leer la base de productivo está
  bloqueado para Claude). Otras causas: contraseñas distintas que el EPC acepta las dos, o el
  EPC abierto en el navegador de alguien.

### 18. Cuántas sesiones de EPC, SAP y el localizador aguanta el servidor

Hecho: el máximo del EPC pasó a 10 (`EPC_MAX_SESSIONS`), cada sesión cierra la pestaña del login
y, si están todas ocupadas, el pedido espera su turno hasta 60 s (`EPC_QUEUE_SECONDS`) en vez
de avisar enseguida. Cada Chrome del EPC ocupa unos 300 MB.

- **En la PC dedicada:** medir de nuevo y ajustar el máximo con margen.
- **A revisar:** qué pasa si muchas cuentas inician sesión juntas, por ejemplo al arrancar el
  backend, que vuelve a abrir las usadas el último día. Y si Nissan (EPC o SAP) o Simetrical
  limitan los inicios de sesión o los pedidos desde una misma IP.
- **Si algún día los Chrome van en varias PC**, o molesta que publicar una versión cierre las
  sesiones del EPC: los Chrome en un proceso aparte del backend.

## Etapa 6: publicar en internet

### 2. La PC como servidor (bloque 4, paso 9)

Lo de la sección «Windows» de [PRODUCCION.md](PRODUCCION.md): que no se suspenda,
actualizaciones que reinicien fuera del horario, el firewall sin el 5432 ni el 8000, el cifrado
del dispositivo si la PC lo tiene, y la UPS. Más el respaldo diario de la base con `pg_dump`,
cifrado y fuera de esta PC.

Falta decidir del respaldo: dónde va la copia, con qué se cifra, a qué hora corre y cuántas se
guardan.

### 3. Pruebas por internet (bloque 5)

- DBSC de verdad: Chrome en Windows con TPM, por HTTPS.
- Cotizar a través de Cloudflare sin el error 524 (corta a los 100 segundos sin datos).
- El agente del DMS de una sucursal subiendo el stock a `app.oemia.ar`.

### 4. Ver la página de mantenimiento con una caída real

En el próximo reinicio del backend o corte del túnel, abrir la app: tiene que aparecer la
página de mantenimiento y, cuando el servidor vuelve, abrirse la app sola.

### 5. Un comando para la revisión rápida

Que revise solo lo de «Revisión rápida» en [PRODUCCION.md](PRODUCCION.md), incluido lo de
Postgres y Windows.

## Después de la etapa 6

Se decidió dejarlos para después de publicar. Las decisiones están en
[USUARIOS_Y_PERMISOS.md](USUARIOS_Y_PERMISOS.md#decisiones-tomadas).

### 6. Aviso de privacidad a los empleados

Ley 25.326: qué datos se guardan (usuario, nombre, IP, dispositivo, ingresos, cotizaciones,
registro de seguridad), para qué, cuánto tiempo, quién es el responsable y cómo pedir acceso,
corrección o borrado. Espera a que se defina el responsable (decisión 22). Conviene que lo
revise un abogado.

### 7. Restablecer la contraseña por mail y alertas por mail

Decisión 23. Solo para quien tiene mail; a los demás se la restablece un Administrador
(decisión 29).

### 8. La publicación automática de noche

Con la deploy key de solo lectura (decisión 25).

### 9. Los tests en GitHub Actions

Como condición para publicar (decisión 25).

### 10. Pasar productivo a la PC dedicada

Después de la primera prueba en esta PC (decisión 24).

## Matcher

### 12. Que el cotizador aprenda y dependa menos de la IA

Hecho en la rama `costos` (noviembre de 2026): el resolvedor local, las píldoras, cotizar solo
lo nuevo, eliminar filas con reporte, autocompletado, recomendaciones, pre-cotización, alias
aprendidos, abreviaturas por usuario y el recorrido de los repuestos sin resultado en
Monitoreo. Cómo funciona y sus límites: [RESOLVEDOR.md](RESOLVEDOR.md). El catálogo
normalizado y lo que queda por revisar: [CATALOGO_NORMALIZACION.md](CATALOGO_NORMALIZACION.md).

Falta:

- **Probarlo con el EPC y la IA de verdad.** En desarrollo se probó con lo guardado del EPC y
  sin cuenta de EPC ni de SAP: no se vio un pedido mixto (parte reconocida, parte leída por la
  IA) ni un número de pieza escrito en el pedido contra el EPC.
- **Revisar con un repuestero** los alias iniciales y qué pedidos de todos los días no se
  reconocen todavía (Monitoreo → Sin resultado dice cuántos ítems leyó la IA).
- **Que la pre-cotización adelante también precio y stock**, no solo el EPC.
- **Recomendaciones con las ventas** del DMS, cuando estén, y con los presupuestos guardados.
- **Tolerancia a errores de tipeo** en el resolvedor y en las sugerencias.
- **Que la plataforma vea los alias aprendidos** de cada concesionario y pueda pasarlos al
  catálogo, para todos.
- **Atributos por número de pieza** (óptica LED o halógena, temperatura del termostato) y una
  lista editable de palabras de posición y lado. La posición y el lado siguen en la descripción
  de la entrada.
- **El asesor**: reutilizar sus respuestas por pedido y modelo de vehículo, y probar un modelo
  más barato con los casos guardados. Ahora se guarda su respuesta completa, que es la base.
