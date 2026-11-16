# Guion de la demo (10 a 15 minutos)

## Antes de la clase

```bash
cd hotdog-infra
docker compose -f docker-compose.setup.yml run --rm setup   # solo la primera vez
docker compose up -d --build
```

- Abre `https://localhost:8443` y acepta la advertencia del certificado
  autofirmado (Avanzado → Continuar).
- Las contraseñas de `admin` y `cajero` están en `hotdog-infra/.env`.
- Deja abiertas dos terminales en `hotdog-infra`.

## 1. Es un sistema distribuido (2 min)

```bash
docker compose ps
```

Muestra los 8 contenedores. Señala que **solo el gateway publica puertos**.
Abre el diagrama del [README](README.md).

Para que se vea que son piezas independientes, tumba una:

```bash
docker compose stop orders-service
```

En el navegador: el menú sigue cargando y se puede iniciar sesión, pero al
registrar una venta aparece un aviso claro. En **Auditoría** se ven dos
bitácoras y un aviso de que falta la de Ventas. Vuelve a subirlo:

```bash
docker compose start orders-service
```

## 2. Uso normal (3 min)

1. Entra como **cajero**. Toca dos o tres perros, escribe un nombre de cliente y
   registra la venta. Muestra el recibo.
2. Señala que el cajero no ve Menú, Usuarios ni Auditoría.
3. Sal y entra como **admin**. En **Menú** agrega un producto, edítale el precio
   y elimínalo.
4. Abre **Auditoría**: ahí está todo lo que acabas de hacer, con usuario, hora e IP.

## 3. Atacar el sistema en vivo (5 min)

```bash
bash scripts/security-checks.sh
```

Son 61 ataques y verificaciones reales contra el sistema en ejecución. Mientras
corre, comenta los más vistosos:

| Sección | Qué decir |
|---|---|
| 3 | "Intento inyección SQL en el login: solo responde credenciales inválidas." |
| 4 | "El cajero intenta crear un producto llamando a la API directamente: 403. El frontend no es la defensa; el servidor sí." |
| 5 | "Fabrico un token que dice que soy admin, sin firma. Rechazado. Tomo un token real de cajero y le cambio el rol. Rechazado: la firma ya no coincide." |
| 6 | "Envío una venta con precio de 1 peso. Rechazada: el precio lo pone el servidor." |
| 6b | "Copio el token de un cajero. Funciona. El cajero cierra sesión. El token copiado ya no sirve: salir invalida el token, no solo borra la cookie." |
| 8 | "Los contenedores no corren como root y su disco es de solo lectura." |
| 9 | "El servicio de ventas no puede ni ver la base de productos, y el frontend no puede saltarse el gateway. El usuario de la base no puede borrar la auditoría ni las ventas." |
| 10 | "25 intentos de login seguidos: el gateway frena la mayoría con 429." |

Después de la sección 10 espera un minuto antes de volver a iniciar sesión.

## 4. Mostrarlo a mano (opcional)

**El JavaScript no puede leer la sesión.** En el navegador, con sesión iniciada,
abre la consola (F12) y escribe:

```js
document.cookie
```

Sale vacío. En la pestaña Aplicación → Cookies sí aparece `access_token`,
marcada `HttpOnly`, `Secure` y `SameSite=Strict`.

**La base guarda hashes, no contraseñas.**

```bash
docker compose exec db-auth psql -U postgres -d auth -c "SELECT username, left(password_hash, 40) FROM users"
```

**La aplicación no puede borrar su rastro.**

```bash
docker compose exec db-orders psql -U orders_app -d orders -c "DELETE FROM audit_log"
```

Responde `permission denied`.

**Las cabeceras de seguridad.**

```bash
curl -skI https://localhost:8443/ | grep -iE "strict|content-security|x-frame"
```

## 5. Cierre (1 min)

- La seguridad está en **capas**: gateway, servicio, base de datos, contenedor y red.
- Cada capa **desconfía** de la anterior.
- Lo que no se cubrió está escrito en
  [Limitaciones conocidas](SEGURIDAD.md#limitaciones-conocidas). Saber qué no
  protege un sistema también es seguridad.

## Apagar

```bash
docker compose down        # conserva los datos
docker compose down -v     # borra también las bases
```
