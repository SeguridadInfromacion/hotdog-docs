# Modelo de amenazas

## Qué se protege

| Activo | Por qué importa |
|---|---|
| Contraseñas de los usuarios | Dan acceso a la caja; la gente las reutiliza en otros sitios |
| Sesiones activas | Quien roba una sesión actúa como ese usuario |
| El menú y sus precios | Cambiarlos es robar o sabotear |
| El registro de ventas | Es el dinero del negocio |
| La bitácora de auditoría | Es la evidencia cuando algo sale mal |
| La llave privada de firma | Con ella se puede fabricar cualquier identidad |

## Quién podría atacar

- **Un extraño en la red**: no tiene cuenta; intenta entrar o tumbar el servicio.
- **Un cajero deshonesto**: tiene cuenta legítima; intenta hacer más de lo permitido.
- **Un sitio web malicioso** abierto en el navegador de un usuario con sesión.
- **Alguien que compromete un contenedor**: intenta moverse hacia los demás.

## Fronteras de confianza

1. Internet → gateway (lo único expuesto).
2. Gateway → servicios (red interna `backend`).
3. Servicio → servicio (`orders-service` → `products-service`).
4. Servicio → su base de datos (red interna propia).

En cada frontera se vuelve a verificar: nadie confía en que el anterior ya revisó.

## Amenazas y mitigaciones (STRIDE)

| # | Categoría | Amenaza | Mitigación |
|---|---|---|---|
| 1 | **S**uplantación | Adivinar contraseñas por fuerza bruta | Límite de 10 intentos/min por IP en el gateway; bloqueo de cuenta tras 5 fallos; Argon2id hace lento cada intento |
| 2 | Suplantación | Averiguar qué usuarios existen | Mismo mensaje y mismo tiempo de respuesta exista o no el usuario |
| 3 | Suplantación | Fabricar un token (`alg: none`, HS256 con la llave pública, firma propia) | Algoritmo fijo RS256; firma, emisor, audiencia y vencimiento obligatorios |
| 4 | Suplantación | Robar la sesión con XSS | Cookie `HttpOnly`; CSP estricta; Vue escapa todo; `v-html` prohibido |
| 5 | Suplantación | Un sitio ajeno actúa con la sesión del usuario (CSRF) | `SameSite=Strict`; cabecera obligatoria que otro origen no puede enviar; sin CORS |
| 6 | Manipulación (**T**) | Enviar un precio o total falso al vender | El servidor ignora precios del cliente: campos extra se rechazan y el total se calcula con el menú oficial |
| 7 | Manipulación | Inyección SQL | Consultas parametrizadas; ids validados como UUID; el rol de la base no puede hacer DDL |
| 8 | Manipulación | Alterar datos en tránsito | TLS 1.2+ y HSTS |
| 9 | Manipulación | Cambiar ventas históricas al editar el menú | Cada venta guarda copia del nombre y precio del momento |
| 10 | **R**epudio | "Yo no fui" | Bitácora con usuario, acción, hora e IP; la aplicación no puede borrarla ni editarla |
| 11 | Repudio | Falsificar la IP en la bitácora con `X-Forwarded-For` | El gateway sobrescribe la cabecera con la IP real |
| 12 | Divulgación (**I**) | Leer contraseñas si se filtra la base | Solo hay hashes Argon2id con sal |
| 13 | Divulgación | Un cajero mira las ventas de otro | El servidor filtra por el id del token |
| 14 | Divulgación | Errores que revelan detalles internos | Respuesta genérica al cliente; el detalle va al log |
| 15 | Divulgación | Acceso directo a una base de datos | Sin puertos publicados; redes internas; una contraseña distinta por base |
| 16 | Denegación (**D**) | Inundar de peticiones | Límite por IP; cuerpo máximo 64 KB; tiempos de espera; límites de memoria y procesos |
| 17 | Denegación | Un servicio caído tumba todo | Cada servicio falla por separado; respuestas 503 claras; reinicio automático |
| 18 | **E**levación | Un cajero ejecuta acciones de administrador | El rol se lee del token firmado y se verifica en cada ruta, en cada servicio |
| 19 | Elevación | Asignarse un rol enviando campos extra | `extra="forbid"`: campos desconocidos se rechazan |
| 20 | Elevación | Saltar de un contenedor comprometido a otro | Sin root, solo lectura, sin capacidades; redes segmentadas; solo `auth-service` puede firmar tokens |

## Riesgos aceptados

Están en [Limitaciones conocidas](SEGURIDAD.md#limitaciones-conocidas): certificado
autofirmado, tokens sin revocación, secretos en disco local, sin TLS interno y
una sola instancia por servicio. Son decisiones conscientes para mantener el
proyecto pequeño, no olvidos.
