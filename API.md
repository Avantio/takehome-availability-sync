# Portal Sol — Documentación de la API

Portal Sol es un portal de alquiler vacacional (ficticio). Esta API permite a un channel manager
publicar la disponibilidad y el precio por noche de los alojamientos que tiene anunciados en el portal.

> Este documento es el **contrato completo** del portal. No dispones del código del servidor:
> trabaja únicamente con lo que aquí se describe y con el comportamiento observable.

## Arranque

El portal se distribuye como imagen Docker:

```bash
docker run --rm -p 4000:4000 ghcr.io/avantio/portal-sol:latest
```

URL base: `http://localhost:4000`. Comprueba que está arriba con `GET /health`.

El estado del portal vive en memoria: al reiniciar el contenedor se vuelve a los datos iniciales.

## Autenticación

Todas las rutas bajo `/api/` requieren la cabecera `X-Api-Key`.

```
X-Api-Key: sol-demo-key
```

Sin cabecera, o con una clave incorrecta, el portal responde `401 UNAUTHORIZED`.

## Convenciones

- Cuerpos y respuestas en JSON (`Content-Type: application/json`).
- Las fechas son **días naturales** en formato `YYYY-MM-DD`, sin hora ni zona horaria.
- Los rangos de fechas son **inclusivos** en ambos extremos: `from: 2026-10-01, to: 2026-10-07` cubre 7 noches.
- `pricePerNight` es un número decimal (EUR por noche), mayor o igual que 0.
- Los errores siempre tienen esta forma:

  ```json
  { "error": { "code": "RANGE_TOO_LARGE", "message": "Date range covers 32 days; ..." } }
  ```

## Límites y comportamiento del portal

Como cualquier portal real, Portal Sol tiene sus particularidades. Tenlas en cuenta al diseñar tu integración:

| Situación | Respuesta del portal | Qué se espera de tu integración |
|---|---|---|
| Superas el número de peticiones por minuto permitido | `429 RATE_LIMITED` con cabecera `Retry-After: <segundos>` | No reintentar antes de que pasen esos segundos. Las peticiones rechazadas no se procesan. |
| El portal no puede atender la petición en ese momento | `503 UNAVAILABLE` | La petición **no** se ha aplicado. Puede reintentarse más tarde. |
| El portal tarda en responder | Sin respuesta durante un tiempo | Configura un timeout razonable. **Una petición que expira por timeout puede haber sido procesada igualmente por el portal.** |
| Rango de fechas demasiado largo | `400 RANGE_TOO_LARGE` | Cada petición puede cubrir como máximo **31 días** (`from` y `to` inclusive). Rangos mayores deben dividirse en varias peticiones. |

Los `PUT` son **idempotentes**: repetir exactamente la misma petición deja el portal en el mismo estado.

## Endpoints

### `GET /api/v1/accommodations`

Lista los alojamientos publicados en el portal.

```json
{
  "accommodations": [
    { "id": "acc-1001", "name": "Apartamento Sol y Mar" },
    { "id": "acc-1002", "name": "Casa Rural Los Olivos" }
  ]
}
```

### `GET /api/v1/accommodations/:id`

Estado actual, día a día, de un alojamiento. Acepta los parámetros opcionales `from` y `to`
(`YYYY-MM-DD`, inclusivos) para acotar el resultado.

```
GET /api/v1/accommodations/acc-1003?from=2026-10-01&to=2026-10-03
```

```json
{
  "id": "acc-1003",
  "name": "Ático Centro Histórico",
  "days": [
    { "date": "2026-10-01", "available": true,  "pricePerNight": 120 },
    { "date": "2026-10-02", "available": true,  "pricePerNight": 140 },
    { "date": "2026-10-03", "available": false, "pricePerNight": 140 }
  ]
}
```

Errores: `404 NOT_FOUND` si el alojamiento no existe; `400 INVALID_DATE` si `from`/`to` no son fechas válidas.

### `PUT /api/v1/accommodations/:id/availability`

Fija disponibilidad y precio para todos los días del rango indicado.

Cuerpo:

```json
{
  "from": "2026-10-01",
  "to": "2026-10-07",
  "available": false,
  "pricePerNight": 120.0
}
```

| Campo | Tipo | Reglas |
|---|---|---|
| `from` | string `YYYY-MM-DD` | Obligatorio. Fecha válida. |
| `to` | string `YYYY-MM-DD` | Obligatorio. Igual o posterior a `from`. **Como máximo 31 días** contando ambos extremos. |
| `available` | boolean | Obligatorio. |
| `pricePerNight` | number | Obligatorio. `>= 0`. |

Respuesta `200`:

```json
{ "accommodationId": "acc-1003", "from": "2026-10-01", "to": "2026-10-07", "daysUpdated": 7 }
```

Errores: `400 INVALID_BODY`, `400 INVALID_DATE`, `400 RANGE_TOO_LARGE`, `404 NOT_FOUND`, además de los
`401`, `429` y `503` descritos más arriba.

### `GET /health`

Sin autenticación. Responde `200 { "status": "ok", ... }` cuando el portal está listo para recibir peticiones.

## Endpoints de administración

Estos endpoints **no forman parte de la API real** de un portal: existen para que puedas verificar tu
implementación (manualmente o desde tus tests). No requieren `X-Api-Key`, no cuentan para el límite de
peticiones y tu servicio de sincronización no debería depender de ellos en su funcionamiento normal.

### `GET /__admin/requests`

Registro de todas las peticiones recibidas bajo `/api/` (incluidas las rechazadas con `401`, `429` o `503`),
en orden de llegada. Se conservan las últimas 1000.

```json
{
  "count": 2,
  "requests": [
    {
      "id": 1,
      "timestamp": "2026-09-02T10:15:30.123Z",
      "method": "PUT",
      "path": "/api/v1/accommodations/acc-1003/availability",
      "body": { "from": "2026-10-01", "to": "2026-10-07", "available": false, "pricePerNight": 120 },
      "status": 200,
      "durationMs": 412
    },
    {
      "id": 2,
      "timestamp": "2026-09-02T10:15:31.001Z",
      "method": "GET",
      "path": "/api/v1/accommodations/acc-1003",
      "body": null,
      "status": 429,
      "durationMs": 2
    }
  ]
}
```

Si el cliente cerró la conexión antes de recibir respuesta, la entrada tiene `"status": null` y
`"clientDisconnected": true`. Eso no significa que la petición no se haya aplicado.

### `POST /__admin/reset`

Restaura los datos iniciales de todos los alojamientos, vacía el registro de peticiones y reinicia el
contador del límite por minuto. Responde `200 { "ok": true }`. Útil al principio de cada test.

## Códigos de error

| HTTP | `code` | Significado |
|---|---|---|
| 400 | `INVALID_BODY` | El cuerpo no es JSON válido o falta/sobra algún campo, o tiene un tipo incorrecto. |
| 400 | `INVALID_DATE` | Fecha mal formada, inexistente (p. ej. `2026-02-30`) o `to` anterior a `from`. |
| 400 | `RANGE_TOO_LARGE` | El rango cubre más de 31 días. |
| 401 | `UNAUTHORIZED` | Falta la cabecera `X-Api-Key` o la clave no es válida. |
| 404 | `NOT_FOUND` | El alojamiento (o la ruta) no existe. |
| 429 | `RATE_LIMITED` | Límite de peticiones por minuto superado. Consulta `Retry-After`. |
| 503 | `UNAVAILABLE` | El portal no puede atender la petición ahora. Nada se ha aplicado. |

## Alojamientos disponibles

El portal arranca con estos 10 alojamientos, cada uno con 90 días de disponibilidad a partir de la fecha
actual. Puedes consultarlos con `GET /api/v1/accommodations`.

| id | Nombre |
|---|---|
| `acc-1001` | Apartamento Sol y Mar |
| `acc-1002` | Casa Rural Los Olivos |
| `acc-1003` | Ático Centro Histórico |
| `acc-1004` | Villa Las Dunas |
| `acc-1005` | Estudio Playa Norte |
| `acc-1006` | Chalet Sierra Blanca |
| `acc-1007` | Loft Puerto Viejo |
| `acc-1008` | Cortijo El Almendral |
| `acc-1009` | Piso Familiar Ensanche |
| `acc-1010` | Cabaña Lago Azul |
