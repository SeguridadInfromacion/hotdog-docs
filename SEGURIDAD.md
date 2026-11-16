# Controles de seguridad

Cada control indica **dónde está** para que se pueda abrir el archivo en clase.
Todos se pueden comprobar en vivo con
`bash scripts/security-checks.sh` en `hotdog-infra` (47 verificaciones).

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
| Bloqueo de la cuenta 15 min tras 5 intentos fallidos | `hotdog-auth-service/app/services.py` |
| Límite de 10 intentos de login por minuto por IP | `hotdog-infra/gateway/templates/default.conf.template` |
| JWT firmado con **RS256**; algoritmo fijo (rechaza `alg: none` y confusión HS256) | `app/security.py` de cada servicio |
| Solo `auth-service` tiene la llave privada; los demás solo pueden **verificar** | `hotdog-infra/docker-compose.yml` (sección `secrets`) |
| Token en cookie `HttpOnly; Secure; SameSite=Strict`: el JavaScript no puede leerlo | `hotdog-auth-service/app/routes.py` |
| Usuario desactivado pierde el acceso a `auth-service` de inmediato | `hotdog-auth-service/app/dependencies.py` |

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

## Gateway (nginx)

| Control | Dónde |
|---|---|
| HTTPS obligatorio, TLS 1.2 y 1.3, HTTP redirige a HTTPS | `hotdog-infra/gateway/templates/default.conf.template` |
| Cabeceras: HSTS, CSP estricta (sin `unsafe-inline`), `X-Frame-Options`, `nosniff`, `Referrer-Policy`, `Permissions-Policy` | ídem |
| Límite de peticiones por IP y tamaño máximo de cuerpo (64 KB) | ídem |
| `X-Forwarded-For` se sobrescribe: el cliente no puede falsificar su IP en la auditoría | ídem |
| Rutas internas (`/health`, `/docs`) no publicadas | ídem |

## Contenedores y red

| Control | Dónde |
|---|---|
| Procesos **sin root** | `Dockerfile` de cada servicio |
| Sistema de archivos de **solo lectura**, sin capacidades de Linux, sin escalamiento de privilegios | `hotdog-infra/docker-compose.yml` (`x-hardened`) |
| Límites de memoria y de procesos | ídem |
| Un solo puerto publicado (el gateway), por defecto solo en `127.0.0.1` | ídem |
| Redes segmentadas: cada base solo es alcanzable por su servicio; `backend` sin salida a internet | ídem (sección `networks`) |
| Imagen del frontend en dos etapas: sin Node ni código fuente en producción | `hotdog-frontend/Dockerfile` |

## Base de datos

| Control | Dónde |
|---|---|
| Una base por servicio | `hotdog-infra/docker-compose.yml` |
| Rol de aplicación con **mínimo privilegio**: no puede crear ni borrar tablas | `hotdog-database/common/90_roles.sh` |
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
| A10 SSRF | El único llamado saliente va a una dirección fija de configuración; `backend` no tiene salida a internet |

## Limitaciones conocidas

Un sistema seguro también dice con honestidad qué **no** cubre:

- **Certificado autofirmado.** El navegador muestra una advertencia. En
  producción se usaría un certificado de una autoridad reconocida.
- **Los tokens no se revocan.** Al desactivar un usuario, `auth-service` lo
  rechaza de inmediato, pero `products-service` y `orders-service` aceptan su
  token hasta que vence (60 minutos como máximo). La solución habitual es una
  lista de revocación compartida o tokens de vida más corta con renovación.
- **Secretos en archivos locales.** `.env` y `secrets/` viven en el disco de
  quien despliega. En producción irían en un gestor de secretos.
- **La auditoría no es inmutable frente al superusuario** de PostgreSQL, solo
  frente a la aplicación.
- **Sin TLS entre contenedores.** El tráfico interno va por redes privadas de
  Docker sin cifrar; en un despliegue en varias máquinas se usaría mTLS.
- **Una sola instancia de cada servicio.** No hay réplicas ni balanceo.
