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

Plan propuesto el 2026-11-02, sin acordar ni empezar. Las ideas están abiertas a discusión:

1. Registrar qué elige el vendedor y lo que devuelve el EPC, y comparar Haiku con Sonnet.
2. El catálogo en la base y agregar un repuesto por número: ya están hechos.
3. Un matcher propio que corre a la sombra, sin mostrar lo suyo.
4. Prenderlo de a poco, regla por regla.
5. Compatibilidades aprendidas y abreviaturas de cada usuario.
