# Revisión independiente

Antes de dar el sistema por terminado se le encargó a cuatro revisores, que no
participaron en la construcción, buscarle fallas:

| Revisor | Qué hizo |
|---|---|
| Código del backend | Leyó los tres servicios y la base, y escribió pruebas para demostrar cada sospecha |
| Infraestructura y penetración | Atacó el sistema en ejecución y revisó contenedores, redes y base de datos |
| Código del frontend | Revisó errores, seguridad del lado del cliente, accesibilidad y limpieza |
| Interfaz | Usó la aplicación en escritorio, tableta y celular, con ratón y con teclado |

Ninguno encontró una falla crítica ni una forma de saltarse la autenticación o
los permisos. Sí encontraron problemas reales. Este documento los cuenta: un
sistema seguro no es el que nunca tuvo fallas, sino el que las busca y las corrige.

## Lo más importante que se encontró y se corrigió

### 1. Cualquiera podía dejar al administrador por fuera

**Falla.** La cuenta se bloqueaba 15 minutos tras 5 contraseñas malas, sin importar
quién las enviara. Un extraño podía enviar 5 intentos cada 15 minutos contra
`admin` y mantenerlo bloqueado para siempre. Un control de seguridad convertido
en un ataque de denegación de servicio.

**Corrección.** El bloqueo ahora es por pareja **usuario + dirección IP**: quien
falla se bloquea a sí mismo. Además los fallos viejos caducan y los intentos
simultáneos se cuentan todos (antes, seis intentos a la vez contaban como uno).

### 2. Los tokens seguían valiendo después de salir o de ser desactivado

**Falla.** Un administrador degradado a cajero, o desactivado, podía seguir
cambiando precios durante una hora con el token que ya tenía. Cerrar sesión solo
borraba la cookie del navegador: una copia del token seguía funcionando.

**Corrección.** Los tokens ahora duran 10 minutos y el navegador los renueva
solo. Cada usuario tiene una versión de token que sube al cerrar sesión, cambiar
la contraseña o el rol: `auth-service` rechaza de inmediato los tokens viejos y
no los renueva. Queda una ventana de hasta 10 minutos en los otros dos
servicios, documentada como limitación.

### 3. Cerrar sesión podía dejar el pedido del cajero anterior

**Falla.** Si la red fallaba justo al pulsar "Salir", la pantalla quedaba a
medias: sin usuario, pero con el pedido en memoria y la cookie viva. En una caja
compartida, la siguiente persona heredaba el pedido.

**Corrección.** Un solo punto del código cierra la sesión local y siempre borra
usuario, pedido y avisos. Si no se pudo avisar al servidor, se dice claramente.

### 4. Un servicio comprometido podía reescribir las ventas

**Falla.** El rol de base de datos de `orders-service` podía modificar y borrar
ventas, aunque el servicio nunca lo hace. Se anunciaba "mínimo privilegio" pero
los permisos eran iguales para todos.

**Corrección.** Permisos por tabla según lo que cada servicio usa de verdad. El
servicio de ventas solo puede insertar y leer.

### 5. El contenedor del frontend podía saltarse el gateway

**Falla.** Todos los servicios compartían una red interna. Un contenedor
comprometido podía llamar directo a `auth-service` sin límite de peticiones y
falsificando su dirección IP en la auditoría.

**Corrección.** Una red por cada pareja de contenedores que necesita hablarse, y
los servicios solo le creen la IP del cliente al gateway.

### 6. Se podían falsificar líneas del log

**Falla.** Un nombre de producto con un salto de línea escribía una segunda
línea, inventada, en el log de auditoría. Un byte nulo en el usuario producía un
error 500 sin autenticación.

**Corrección.** Ningún texto admite caracteres de control, y el log escapa cada dato.

### 7. Un error de validación devolvía la contraseña enviada

**Falla.** Al rechazar una contraseña débil, la respuesta incluía la contraseña
tal cual. Terminaría en capturas, herramientas del navegador o logs.

**Corrección.** Los errores de validación dicen dónde y por qué, nunca el valor.

## Otras correcciones

**Backend**
- `orders-service` valida con límites lo que responde `products-service`: un
  precio negativo ya no llega a una venta.
- El menú se consulta en una sola llamada por venta, no una por producto.
- Una sesión vencida entre servicios se reporta como tal, no como "menú no disponible".
- El servicio no arranca si la base está vacía y no hay administrador configurado.
- Los parámetros de una consulta fallida ya no se escriben en el log.

**Gateway e infraestructura**
- Solo cifrados TLS con secreto hacia adelante; se rechazan los de tipo CBC y SHA-1.
- Una petición con cabecera `Host` ajena se corta (antes servía para armar
  redirecciones hacia otro dominio).
- Rotación de logs, límite de CPU y más memoria para el servicio que calcula Argon2.
- Dos productos no pueden llamarse igual cambiando solo mayúsculas.

**Interfaz**
- El botón "Registrar venta" siempre queda visible, por largo que sea el pedido.
- Una palabra muy larga ya no desborda las tarjetas ni ensancha la página en celular.
- La navegación ya no se corta en tableta y celular.
- No se pueden agregar productos mientras se registra una venta (antes se perdían).
- Si cambia el menú, el pedido en curso se actualiza y avisa qué se quitó.
- Vaciar el pedido y salir con un pedido a medias piden confirmación.
- Los errores se quedan en pantalla hasta cerrarlos y se muestran dentro de los diálogos.
- Cada producto conserva siempre su mismo dibujo.
- Foco de teclado más visible y bien ubicado al abrir diálogos y cambiar de pantalla.

## Lo que resistió

La prueba de penetración intentó, sin éxito:

- Falsificar tokens: sin firma, HS256 con la llave pública, llave propia,
  cabeceras `jwk`, `jku` y `kid`.
- Llegar a rutas internas (`/health`, `/docs`) con trucos de ruta y codificación.
- Falsificar la IP con `X-Forwarded-For`, `X-Real-IP` y `Forwarded`.
- Saltarse la defensa anti-CSRF cambiando el tipo de contenido.
- Escalar de cajero a administrador, o ver ventas de otro cajero.
- Imponer precios, totales o cantidades inválidas.
- Ejecutar código con XSS en nombres de productos y clientes.
- Contrabando de peticiones HTTP y cuerpos gigantes.
- Leer contraseñas o ejecutar programas desde el rol de base de datos.

## Lo que se decidió no corregir

Está en [Limitaciones conocidas](SEGURIDAD.md#limitaciones-conocidas), con el
porqué de cada una.

## Números

| | Antes de la revisión | Después |
|---|---|---|
| Pruebas automáticas del backend | 81 | 129 |
| Pruebas automáticas del frontend | 14 | 30 |
| Verificaciones de seguridad en vivo | 47 | 61 |
