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
