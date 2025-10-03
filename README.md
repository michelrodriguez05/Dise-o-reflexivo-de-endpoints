# Diseño reflexivo de endpoints (App de transporte tipo Uber)


## Contexto

Se desea diseñar la API de un sistema similar a Uber, que permita conectar pasajeros y conductores, gestionar solicitudes de viajes, procesar pagos, registrar calificaciones y administrar la plataforma.

La aplicación debe manejar al menos los siguientes elementos:

- **Usuarios** (pasajeros y conductores).
- **Viajes** (solicitud, aceptación, estado, cancelación, finalización).
- **Pagos** (procesamiento, historial, recibos).
- **Calificaciones** (pasajero a conductor y viceversa).

También existe un rol **administrador**, encargado de la gestión general.

---
**1. ¿Qué entiendes por “endpoint” en el contexto de una API?**
Un endpoint como lo dice su nombre un punto final de un API o tambien un servicio web y es una URL (direccion) donde un cliente (navegador, app móvil, frontend, otro servidor) puede hacer una petición y resibir una respuesta del servidor.

---
**2. ¿Cuál es la diferencia entre un endpoint público y uno privado?**
A los endpoints Públicos cualquiera puede usarlos y no requieren autenticacion, lo riesgos que podria tener es que no deberian exponer datos confidenciales, porque cualquiera puede consultarlos.

A los endpoints Privados se requiere autenticación (usuario y contraseña, token, API key, OAuth, etc.) claraemnte  la ventaja que tienen es que protegen la privacidad y la seguridad del usuario.

---
**3.¿Qué información de un usuario consideras confidencial y no debería exponerse?**

Contraseñas, tokens, números de tarjeta completos, datos bancarios, ubicación en tiempo real sin consentimiento, documento de identidad completo, datos biométricos y correos/teléfonos sin protecciones.

---
**4. ¿Por qué es importante definir bien los métodos HTTP (GET, POST, PUT/PATCH, DELETE) en cada endpoint?**
Porque clarifica la intención (leer, crear, actualizar, borrar), ayuda a respetar idempotencia y caché, facilita el diseño RESTful y permite uso correcto de códigos HTTP.

---
**5. ¿Qué tipo de información requiere autenticación en este sistema?**
Crear/editar viajes, ver historial de pagos, acceder a ubicación en tiempo real de conductores/pasajeros, procesar pagos, valorar usuarios, panel de administración.

---
**6. ¿Cómo manejarías la seguridad de la ubicación de conductores y pasajeros?**
Sólo compartir ubicación en tiempo real tras consentimiento y durante un viaje activo; cifrar datos en tránsito (HTTPS/TLS); limitar frecuencia y retención (ej. retener 24–72 horas si es necesario por disputas); anonimizar cuando no sea esencial; permisos por scopes en tokens.

---
**7. ¿Qué pasaría si un viaje es solicitado y no hay conductores disponibles? ¿Cómo debería responder la API?**
Si un pasajero pide un viaje y no hay conductores cercanos, la API no debería dar un error, sino responder con un mensaje claro que indique que en ese momento no hay disponibilidad. Podría devolver un código 200 con un body tipo { "available": false } y, además, sugerirle al usuario opciones como volver a intentarlo más tarde, ampliar el área de búsqueda o mostrar un tiempo estimado de espera. También sería útil que la API registre estas solicitudes sin atender para analizarlas después y que, opcionalmente, se pueda activar una notificación o evento que avise al pasajero cuando aparezca un conductor disponible.

---
**8. ¿Cómo identificarías los recursos principales de esta aplicación?**

users, profiles, drivers, rides, payments, ratings, locations, vehicles, admin, notifications.

---
**9. ¿Qué ventajas tendría versionar la API (por ejemplo, /v1/...) desde el inicio?**
Permite cambios breaking sin romper clientes existentes, facilita migraciones, pruebas A/B y documentación por versión.

---
**10. ¿Por qué es importante documentar las respuestas de error y no solo las exitosas?**
Porque los clientes necesitan manejar fallos correctamente; ayuda a depurar, mejora experiencia de usuario y evita comportamientos indeterminados en el frontend.

---

## Roles y permisos
*Explica qué acciones puede realizar cada rol (pasajero, conductor, administrador).*
- **Pasajero (user:passenger)**

Crear/solicitar viajes, cancelar (según política), ver historial, pagar, calificar conductor, ver ubicación del conductor durante viaje, editar perfil y métodos de pago.

- **Conductor (user:driver)**

Recibir/aceptar viajes, actualizar estado de viaje (en ruta, llegó, inicié, finalizado), compartir ubicación en tiempo real, ver historial de viajes, recibir pagos/retiradas, calificar pasajero, gestionar vehículo y disponibilidad.

- **Administrador (admin)**

Ver/modificar usuarios y conductores, gestionar disputas, ver métricas, emitir reembolsos, inhabilitar cuentas, acceder a logs y reportes. Acceso restringido y auditado.


## Recursos principales
Lista de recursos que manejará tu API (ejemplo: `users`, `rides`, `payments`, `ratings`, etc.).

**Recursos principales**

- users (registro, login, perfiles)
- drivers (perfil conductor, vehículo, licencia)
- vehicles (info vehículo)
- rides (viajes/solicitudes/estados)
- payments (procesamiento, recibos, reembolsos)
- ratings (calificaciones y comentarios)
- locations (ubicaciones/geo)
- notifications (push/emails)
- admin (operaciones de backoffice)
- reports (reportes y métricas)

## Tabla de endpoints

|  # | Método | Ruta  | Descripción    | Parámetros (path / query body)                                                        | Auth                                    |                                    |                         |
|-:| :----:|:------------------------------- | :------------------------------------------------------------ | :-------------------------------------------------------------------------------------- | :-------------------------------------- | ---------------------------------- | ----------------------- |
|  1 |  POST  | `/v1/auth/register`              | Registrar nuevo usuario (pasajero o conductor)                | body: `{name,email,password,role (passenger                                             | driver),phone}`                         | público                            |                         |
|  2 |  POST  | `/v1/auth/login`                 | Obtener token (login)                                         | body: `{email,password}` → resp `{token,expires_in}`                                    | público                                 |                                    |                         |
|  3 |  POST  | `/v1/auth/logout`                | Invalidar token de sesión                                     | headers: `Authorization`                                                                | requiere token                          |                                    |                         |
|  4 |   GET  | `/v1/users/me`                   | Obtener perfil del usuario autenticado                        | headers token                                                                           | requiere token                          |                                    |                         |
|  5 |  PATCH | `/v1/users/me`                   | Actualizar perfil (nombre, teléfono, foto)                    | body: `{name,phone,photoUrl}`                                                           | requiere token                          |                                    |                         |
|  6 |  POST  | `/v1/drivers/verify`             | Enviar documentos para verificación de conductor              | body: `{licenseNumber,documents[]}`                                                     | requiere token, role=driver             |                                    |                         |
|  7 |   GET  | `/v1/drivers/:driverId`          | Ver perfil público de conductor (no ubicación en tiempo real) | path: `driverId`                                                                        | público                                 |                                    |                         |
|  8 |  POST  | `/v1/vehicles`                   | Registrar vehículo (conductor)                                | body: `{make,model,plate,color}`                                                        | requiere token, role=driver             |                                    |                         |
|  9 |   GET  | `/v1/rides/estimate`             | Obtener estimación de costo/tiempo antes de solicitar         | query: `pickupLat,pickupLng,destLat,destLng,serviceType`                                | público                                 |                                    |                         |
| 10 |  POST  | `/v1/rides`                      | Solicitar un viaje (crear solicitud)                          | body: `{pickup:{lat,lng,address},destination:{lat,lng,address},serviceType,promoCode?}` | requiere token, role=passenger          |                                    |                         |
| 11 |   GET  | `/v1/rides/:rideId`              | Obtener estado/detalle de un viaje                            | path: `rideId`                                                                          | requiere token (owner driver/passenger) |                                    |                         |
| 12 |  PATCH | `/v1/rides/:rideId/cancel`       | Cancelar viaje (política de cancelación)                      | body: `{reason,chargeCancellation?:boolean}`                                            | requiere token (owner)                  |                                    |                         |
| 13 |  PATCH | `/v1/rides/:rideId/accept`       | Conductor acepta la solicitud                                 | path: `rideId`                                                                          | requiere token, role=driver             |                                    |                         |
| 14 |  PATCH | `/v1/rides/:rideId/status`       | Actualizar estado (arribó, inició, finalizó)                  | body: `{status: "driver_arrived"                                                        | "in_progress"                           | "completed", location?:{lat,lng}}` | requiere token (driver) |
| 15 |  POST  | `/v1/rides/:rideId/track`        | Envío periódico de ubicación del conductor durante viaje      | body: `{lat,lng,timestamp}`                                                             | requiere token (driver)                 |                                    |                         |
| 16 |   GET  | `/v1/drivers/nearby`             | Buscar conductores disponibles en un radio                    | query: `lat,lng,radiusKm,serviceType`                                                   | público (limitado)                      |                                    |                         |
| 17 |  POST  | `/v1/payments/charge`            | Procesar pago de viaje                                        | body: `{rideId,paymentMethodId,amount}`                                                 | requiere token (passenger)              |                                    |                         |
| 18 |   GET  | `/v1/payments/:paymentId`        | Obtener recibo / detalle de pago                              | path: `paymentId`                                                                       | requiere token (owner or admin)         |                                    |                         |
| 19 |  POST  | `/v1/payments/refund`            | Solicitar reembolso / reembolso parcial                       | body: `{paymentId,amount,reason}`                                                       | requiere token (admin or system)        |                                    |                         |
| 20 |  POST  | `/v1/ratings`                    | Crear calificación entre usuario y conductor                  | body: `{rideId,score,comment,for: "driver"                                              | "passenger"}`                           | requiere token                     |                         |

---
## Flujos de uso 

**Flujo A — Solicitud, aceptación y finalización de un viaje**
**Pasajero**: calcula estimación → GET /v1/rides/estimate.

**Pasajero**: solicita viaje → POST /v1/rides con pickup y destino.

**Sistema**: busca conductores GET /v1/drivers/nearby y notifica varios.

**Conductor**: acepta → PATCH /v1/rides/:rideId/accept.

**Conductor**: envía ubicación periódica → POST /v1/rides/:rideId/track.

**Conductor**: marca driver_arrived → PATCH /v1/rides/:rideId/status.

**Conductor**: inicia viaje in_progress → PATCH /v1/rides/:rideId/status.

**Conductor**: finaliza viaje completed → PATCH /v1/rides/:rideId/status.

**Sistema**: crea cargo → POST /v1/payments/charge.

**Pasajero**: recibe recibo → GET /v1/payments/:paymentId.

**Ambos**: califican → POST /v1/ratings.

--- 

**Flujo B — Cancelación y política de devolución parcial**

**1.Pasajero**: solicita cancelación → PATCH /v1/rides/:rideId/cancel (antes de aceptación → sin cargo o con tarifa mínima según política).

**2**.Si conductor ya está en camino → PATCH con chargeCancellation=true y se crea cargo parcial POST /v1/payments/charge (monto de penalización).

**3.Sistema**: notifica a conductor, actualiza historial y emite recibo/refund si corresponde → POST /v1/payments/refund.

---
**Flujo C — Calificación y disputa**

**1**.Tras finalización, pasajero crea rating → POST /v1/ratings. 

**2.** Si hay disputa (conductor o pasajero): POST /v1/support/tickets con evidencia.

**3**. Admin revisa GET /v1/support/tickets/:ticketId, decide acción (reembolso, suspensión), aplica POST /v1/payments/refund o PATCH /v1/admin/users/:userId/status.


---
## Decisiones de diseño y justificación

- **Uso de los métodos HTTP**

Definí GET solo para consultas, POST para crear, PUT/PATCH para actualizaciones y DELETE para eliminaciones. Esto ayuda a que la API sea clara y fácil de entender.

- **Separación de recursos**

Cada recurso (ej. rides, payments, ratings) tiene sus propios endpoints. Así se evita confusión y se mantiene la API modular.

- **Autenticación y privacidad**

Marqué claramente qué endpoints son públicos (ej. registro, login) y cuáles requieren autenticación (ej. solicitar un viaje, pagar, calificar). Esto protege la información confidencial del usuario.

- **Seguridad de datos sensibles**

No expongo contraseñas, información financiera completa ni ubicación exacta sin permisos. La ubicación en tiempo real, por ejemplo, solo puede verse en un viaje activo.

- **Versionado desde el inicio**

Decidí usar /v1/... en todas las rutas. Esto facilita futuras actualizaciones sin romper la versión actual.

- **Respuestas consistentes**

Todas las respuestas siguen un mismo formato JSON con claves como success, data, message o error. Así, los clientes que consumen la API saben qué esperar siempre.

- **Manejo de casos especiales**

Ejemplo: si un pasajero solicita un viaje y no hay conductores disponibles, la API responde con un mensaje claro en lugar de un error.

--- 

## Manejo de errores

**400 — Parámetros inválidos**

- **Situación**: POST /v1/rides sin pickup o con coordenadas fuera de rango.

- **Respuesta**: 400, code: "INVALID_REQUEST".

**401 — Token expirado**

- **Situación**: token JWT expirado al llamar /v1/users/me.

- **Respuesta**: 401, code: "TOKEN_EXPIRED", detalle sugiere POST /v1/auth/refresh.

**403 — Rol insuficiente**

- **Situación**: pasajero intenta PATCH /v1/drivers/:id/status.

- **Respuesta**: 403, code: "INSUFFICIENT_ROLE".

**404 — Viaje no encontrado**

- **Situación**: consulta /v1/rides/abc con id inexistente.

- **Respuesta**: 404, code: "RIDE_NOT_FOUND".

**409 — Conflicto (race) / aceptado por otro conductor**

- **Situación**: dos conductores intentan aceptar la misma solicitud; uno ya la aceptó.

- **Respuesta**: 409, code: "RIDE_ALREADY_ACCEPTED".

```
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Descripción legible del error",
    "detail": "Opcional: más detalle técnico",
    "traceId": "uuid-1234"
  }
}
```


## Propuestas de mejora

- **1. Soporte multi-modal y pooling**: rutas y optimizaciones para viajes compartidos (pool) y rutas múltiples.

- **2. Sistema de ML para precios dinámicos**: surge/optimización de tarifas basadas en demanda/tiempo real.

- **3. Integración con KYC y verificación avanzada**: verificación de identidad por terceros y checks de seguridad.

- **4**. Programación de viajes y predicciones**: reservar viajes futuros, integración con calendario.

- **5. Marketplace de servicios**: añadir servicios adicionales (entregas, cargas) y promoción/promo codes avanzados.
