# Controles de seguridad

Cada control indica **dónde está** para que se pueda abrir el archivo en clase.
Todos se pueden comprobar en vivo con
`bash scripts/security-checks.sh` en `hotdog-infra` (61 verificaciones).

El sistema pasó además por una revisión independiente de código, una prueba de
penetración y una crítica de interfaz; lo que encontraron y cómo se corrigió está
en [REVISION.md](REVISION.md).

## Tríada CID

| Propiedad | Cómo se protege aquí |
|---|---|
| **Confidencialidad** | HTTPS, contraseñas con Argon2id, cookie `HttpOnly`, bases de datos en redes internas, control de acceso por rol, un cajero solo ve sus ventas. |
| **Integridad** | Precios y totales calculados en el servidor, validación estricta de entradas, tokens firmados (RS256), bitácora de solo-agregar, restricciones `CHECK` en la base. |
| **Disponibilidad** | Límite de peticiones, límites de memoria y procesos por contenedor, *healthchecks* y reinicio automático, tiempos de espera entre servicios, la interfaz sigue funcionando por partes si un servicio cae. |

## Autenticación

| Control | Dónde |
|---|---|
| Contraseñas con hash **Argon2id** (nunca en texto plano) | `hotdog-auth-service/app/security.py` |
| Política de contraseñas (10+ caracteres, letras y números) | `hotdog-auth-service/app/schemas.py` |
| Mismo mensaje y mismo tiempo de respuesta si el usuario no existe (no se pueden enumerar usuarios) | `hotdog-auth-service/app/services.py` |
| Bloqueo de 15 min tras 5 intentos fallidos, **por pareja usuario + IP**: quien falla a propósito se bloquea a sí mismo, no al dueño de la cuenta | `hotdog-auth-service/app/services.py` |
| Los intentos simultáneos se cuentan todos (fila bloqueada con `FOR UPDATE`) | `hotdog-auth-service/app/repositories.py` |
| Límite de 10 intentos de login por minuto por IP | `hotdog-infra/gateway/templates/default.conf.template` |
| JWT firmado con **RS256**; algoritmo fijo (rechaza `alg: none` y confusión HS256) | `app/security.py` de cada servicio |
| Solo `auth-service` tiene la llave privada; los demás solo pueden **verificar** | `hotdog-infra/docker-compose.yml` (sección `secrets`) |
| Token en cookie `HttpOnly; Secure; SameSite=Strict`: el JavaScript no puede leerlo | `hotdog-auth-service/app/routes.py` |
| Token de vida corta (10 min) que el navegador renueva solo; la sesión tiene un tope absoluto de 10 horas | `hotdog-auth-service/app/routes.py` (`/api/auth/refresh`) |
| Cerrar sesión, cambiar la contraseña o el rol, o desactivar al usuario **invalida sus tokens** (versión de token) | `hotdog-auth-service/app/dependencies.py`, `services.py` |

## Autorización

| Control | Dónde |
|---|---|
| Roles `admin` y `cashier` verificados **en el servidor, en cada servicio** | `app/dependencies.py` de cada servicio |
| Confianza cero entre servicios: `products-service` vuelve a verificar el token que le reenvía `orders-service` | `hotdog-orders-service/app/products_client.py` |
| Un cajero solo ve sus propias ventas | `hotdog-orders-service/app/services.py` (`_scope`) |
| Un administrador no puede desactivarse ni degradarse a sí mismo | `hotdog-auth-service/app/services.py` |
| La guarda de rutas del frontend es solo comodidad; no es un control de seguridad | `hotdog-frontend/src/router/index.js` |

## Entradas y lógica de negocio

| Control | Dónde |
|---|---|
| Validación estricta con Pydantic: tipos, longitudes y rangos | `app/schemas.py` de cada servicio |
| Campos desconocidos rechazados (sin asignación masiva) | `extra="forbid"` en `app/schemas.py` |
| Sin inyección SQL: solo consultas parametrizadas por el ORM | `app/repositories.py` de cada servicio |
| El cliente no envía precios: el total lo calcula el servidor | `hotdog-orders-service/app/services.py` |
| Sin XSS: Vue escapa todo el texto y `v-html` está prohibido por el linter | `hotdog-frontend/eslint.config.js` |
| Anti-CSRF: `SameSite=Strict` + cabecera obligatoria en peticiones que modifican | `app/dependencies.py` (`require_csrf_header`) |
| Errores sin trazas ni detalles internos hacia el cliente | `app/main.py` de cada servicio |
| Los errores de validación nunca devuelven el valor enviado (podría ser una contraseña) | `app/main.py` de cada servicio |
| Sin caracteres de control en los textos: no se pueden falsificar líneas del log ni provocar errores en la base | `app/schemas.py` (`PRINTABLE`) |
| Tampoco se confía a ciegas en otro servicio: el menú que responde `products-service` se valida con límites | `hotdog-orders-service/app/products_client.py` |

## Gateway (nginx)

| Control | Dónde |
|---|---|
| HTTPS obligatorio, TLS 1.2 y 1.3, solo cifrados ECDHE con GCM o ChaCha20, HTTP redirige a HTTPS | `hotdog-infra/gateway/templates/default.conf.template` |
| Peticiones con cabecera `Host` ajena: se corta la conexión | ídem |
| Cabeceras: HSTS, CSP estricta (sin `unsafe-inline`), `X-Frame-Options`, `nosniff`, `Referrer-Policy`, `Permissions-Policy` | ídem |
| Límite de peticiones por IP y tamaño máximo de cuerpo (64 KB) | ídem |
| `X-Forwarded-For` se sobrescribe, y los servicios solo le creen esa cabecera al gateway: nadie puede falsificar su IP en la auditoría | ídem y `FORWARDED_ALLOW_IPS` en `docker-compose.yml` |
| Rutas internas (`/health`, `/docs`) no publicadas | ídem |

## Contenedores y red

| Control | Dónde |
|---|---|
| Procesos **sin root** | `Dockerfile` de cada servicio |
| Sistema de archivos de **solo lectura**, sin capacidades de Linux, sin escalamiento de privilegios | `hotdog-infra/docker-compose.yml` (`x-hardened`) |
| Límites de memoria, CPU y procesos; logs con rotación | ídem |
| Un solo puerto publicado (el gateway), por defecto solo en `127.0.0.1` | ídem |
| Una red por cada pareja que necesita hablarse (9 redes): el frontend no puede llamar a un servicio saltándose el gateway y ningún servicio alcanza la base de otro. Solo el gateway tiene salida | ídem (sección `networks`) |
| Imagen del frontend en dos etapas: sin Node ni código fuente en producción | `hotdog-frontend/Dockerfile` |

## Base de datos

| Control | Dónde |
|---|---|
| Una base por servicio | `hotdog-infra/docker-compose.yml` |
| Rol de aplicación con **mínimo privilegio**: no puede crear ni borrar tablas, con tope de conexiones y de tiempo por consulta | `hotdog-database/common/20_role.sh` |
| Permisos por tabla según lo que cada servicio usa: `orders-service` no puede editar ni borrar ventas; `auth-service` no puede borrar usuarios | `hotdog-database/*/30_grants.sql` |
| Bitácora de auditoría de **solo-agregar** para la aplicación | ídem |
| Restricciones `CHECK` como última línea de defensa | `hotdog-database/*/01_schema.sql` |

## Secretos y cadena de suministro

| Control | Dónde |
|---|---|
| Ningún secreto en los repositorios; se generan al azar en cada instalación | `hotdog-infra/setup/generate.sh`, `.gitignore` |
| Dependencias con versión fija | `requirements.txt`, `package-lock.json` |
| CI en cada cambio: linter, pruebas, `bandit`, `pip-audit` / `npm audit` | `.github/workflows/ci.yml` de cada repositorio |

## Trazabilidad

Cada servicio registra quién hizo qué, cuándo y desde qué IP: inicios de sesión
(correctos y fallidos), bloqueos, cambios de usuarios, cambios del menú y ventas.
El administrador ve las tres bitácoras juntas en la pantalla **Auditoría**.

## OWASP Top 10 (2021)

| Riesgo | Mitigación principal |
|---|---|
| A01 Pérdida de control de acceso | Roles verificados en cada servicio; ventas filtradas por cajero |
| A02 Fallas criptográficas | HTTPS, Argon2id, RS256, llaves fuera del repositorio |
| A03 Inyección | ORM parametrizado, validación con Pydantic, Vue escapa la salida, CSP |
| A04 Diseño inseguro | Precios en el servidor, confianza cero entre servicios, mínimo privilegio |
| A05 Configuración insegura | Contenedores endurecidos, cabeceras, `/docs` apagado, versiones ocultas |
| A06 Componentes vulnerables | Versiones fijas, `pip-audit` y `npm audit` en CI |
| A07 Fallas de autenticación | Bloqueo de cuenta, límite de intentos, mensajes uniformes, cookie segura |
| A08 Fallas de integridad | Tokens firmados, `npm ci` con lockfile, bitácora de solo-agregar |
| A09 Fallas de registro y monitoreo | Auditoría en base de datos y en los logs de cada servicio |
| A10 SSRF | El único llamado saliente va a una dirección fija de configuración, sin seguir redirecciones; los servicios no tienen salida a internet |

## Limitaciones conocidas

Un sistema seguro también dice con honestidad qué **no** cubre:

- **Certificado autofirmado.** El navegador muestra una advertencia. En
  producción se usaría un certificado de una autoridad reconocida.
- **Ventana de revocación de hasta 10 minutos.** Al desactivar un usuario o
  cerrar su sesión, `auth-service` lo rechaza de inmediato y ya no le renueva el
  token. `products-service` y `orders-service` no consultan a `auth-service` en
  cada petición, así que aceptan el token que ya tenía hasta que vence (10
  minutos como máximo). Cerrar esa ventana exigiría una lista de revocación
  compartida o consultar a `auth-service` en cada petición.
- **El servicio de ventas ve el token del usuario.** Lo reenvía a
  `products-service` para consultar el menú. Si `orders-service` fuera
  comprometido, podría reutilizar ese token contra los otros servicios mientras
  siga vigente. La solución de fondo es un token propio de servicio, de solo
  lectura, o audiencias distintas por servicio.
- **En Docker Desktop (Windows y macOS) todos los clientes comparten IP.** El
  gateway ve la dirección de la red virtual de Docker, no la real: el límite de
  peticiones, el bloqueo por IP y la IP de la auditoría no distinguen un cliente
  de otro. En un servidor Linux sí se ve la IP real.
- **Sin protección contra ventas duplicadas.** Si se pierde la respuesta de una
  venta, la interfaz avisa que hay que revisar en Ventas antes de reintentar,
  pero el servidor no tiene una llave de idempotencia.
- **Las contraseñas iniciales quedan en el entorno** del contenedor
  `auth-service` (solo se usan en el primer arranque) y las de las bases en el
  de cada servicio. Quien tenga acceso a Docker en la máquina puede leerlas.
- **Secretos en archivos locales.** `.env` y `secrets/` viven en el disco de
  quien despliega. En producción irían en un gestor de secretos.
- **La auditoría no es inmutable frente al superusuario** de PostgreSQL, solo
  frente a la aplicación.
- **Sin TLS entre contenedores.** El tráfico interno va por redes privadas de
  Docker sin cifrar; en un despliegue en varias máquinas se usaría mTLS.
- **Imágenes fijadas por etiqueta, no por huella** (`postgres:17-alpine`...).
  Fijarlas por `sha256` evitaría que cambien sin aviso.
- **Una sola instancia de cada servicio.** No hay réplicas ni balanceo.
