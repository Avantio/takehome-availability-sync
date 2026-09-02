# Take-home — Servicio de sincronización de disponibilidad

Gracias por participar en el proceso de selección de Avantio. Este ejercicio está pensado para **3-4 horas de trabajo efectivo**. Tienes una semana para entregarlo, pero no esperamos ni valoramos que le dediques más tiempo del indicado: preferimos un alcance bien acotado y justificado a una solución exhaustiva.

## Sobre el uso de IA

En Avantio trabajamos a diario con agentes de código (Claude Code, Cursor, Copilot o similares). En este ejercicio su uso **no solo está permitido: es lo esperado**. Trabaja como lo harías en tu día a día. Lo que nos interesa evaluar es cómo diriges a la IA, qué decides delegar y cómo validas el resultado, no si eres capaz de escribir el código sin ayuda.

## Contexto

Avantio es una plataforma para gestores de alquiler vacacional. Cuando un gestor cambia la disponibilidad o el precio de un alojamiento para unas fechas, ese cambio debe propagarse a los portales externos donde el alojamiento está publicado (Booking, Airbnb, etc.).

En este ejercicio trabajarás con **un único portal externo ficticio, "Portal Sol"**, del que te proporcionamos un servidor listo para arrancar con Docker. Como todo portal real, tiene sus particularidades: limita el número de peticiones por minuto y a veces falla o tarda en responder. Su API está documentada en [`API.md`](./API.md); **no dispones de su código**, así que trátalo como tratarías a un portal de verdad.

## Tu tarea

Construye un servicio backend en **Node.js + TypeScript** que:

1. **Reciba actualizaciones de disponibilidad y precio** de alojamientos a través de una API HTTP:

   ```
   POST /updates
   {
     "accommodationId": "acc-1003",
     "from": "2026-10-01",
     "to": "2026-10-07",
     "available": true,
     "pricePerNight": 120.00
   }
   ```

2. **Sincronice esas actualizaciones con Portal Sol** a través de su API (documentada en `API.md`), garantizando que:
   - **Ninguna actualización se pierde**, aunque el portal falle o limite las peticiones.
   - El servicio **respeta los límites del portal** y no lo satura.
   - El resultado final en el portal refleja lo que el gestor pidió.

3. **Exponga el estado de sincronización** de un alojamiento:

   ```
   GET /accommodations/:id/sync-status
   ```

   Qué información devolver y con qué estructura es una decisión tuya; justifícala en la spec.

### Lo que no pedimos

- Base de datos real: la persistencia en memoria es aceptable. Si decides usar una, explica por qué.
- Autenticación del servicio, despliegue ni interfaz de usuario.
- Cobertura de tests del 100%: preferimos pocos tests que prueben lo importante a muchos que no prueben nada.

## Entregables

Los tres son obligatorios y tienen el mismo peso que el código en la evaluación:

1. **`SPEC.md`** — Escrita **antes de empezar a programar** y committeada como primer commit. Describe qué vas a construir, las decisiones de diseño principales y los criterios de aceptación. Puedes actualizarla durante el ejercicio, pero queremos ver el punto de partida en el historial.

2. **Código** con tests, en la carpeta `service/` de este mismo repositorio, con historial de commits (no un único commit final).

3. **`PROCESS.md`** — Media página basta. Cuéntanos:
   - Qué delegaste a la IA y qué hiciste a mano, y por qué.
   - Dónde la IA se equivocó o propuso algo que descartaste, y cómo lo detectaste.
   - Qué dejaste fuera por tiempo y qué harías a continuación.

## Cómo empezar

Necesitas Docker (o Docker Desktop) y Node.js 22 o superior.

```bash
# 1. Crea tu repositorio a partir de este template ("Use this template" en GitHub) y clónalo.

# 2. Arranca Portal Sol
docker compose up -d
curl http://localhost:4000/health          # → {"status":"ok",...}

#    Alternativa sin compose:
#    docker run --rm -p 4000:4000 ghcr.io/avantio/portal-sol:latest

# 3. Lee API.md y escribe SPEC.md. Haz tu primer commit.

# 4. Crea tu servicio en la carpeta service/
#    (estructura libre; incluye instrucciones para arrancarlo y lanzar los tests)
```

Portal Sol expone dos endpoints de administración que te resultarán útiles para verificar tu implementación:
`GET /__admin/requests` devuelve el registro de todas las peticiones que ha recibido (incluidas las rechazadas)
y `POST /__admin/reset` lo deja todo como al principio. Están descritos en `API.md`.

El estado del portal vive en memoria: si reinicias el contenedor, vuelve a los datos iniciales.

## Qué evaluamos

| Dimensión | Qué miramos |
|---|---|
| Especificación | Claridad, criterios de aceptación, decisiones justificadas |
| Calidad del código | Diseño, tests con valor real, mantenibilidad |
| Uso de la IA | Notas de proceso concretas y honestas; coherencia con el historial de commits |
| Juicio técnico | Trade-offs razonados, alcance acotado, qué dejaste fuera y por qué |

## Entrega

Cuando termines, comparte el repositorio (público o con acceso para el usuario que te indiquemos) respondiendo al correo del proceso. En la siguiente fase repasaremos contigo tu solución y la extenderemos juntos en directo.

¡Suerte!
