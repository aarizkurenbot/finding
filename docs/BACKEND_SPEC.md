# Finding — Especificación del Backend

> API REST en Go. Fuente única de verdad para contratos, endpoints y modelo de datos.

---

## Stack

| Componente | Librería / Herramienta |
|------------|------------------------|
| Lenguaje | Go 1.24+ |
| Router HTTP | `go-chi/chi/v5` |
| Driver PostgreSQL | `jackc/pgx/v5` |
| Generador de código SQL | `kyleconroy/sqlc` |
| Migrations | `golang-migrate/migrate` |
| Auth / JWT | `golang-jwt/jwt/v5` |
| Config / env | Variables de entorno (sin librería externa) |

---

## Estructura de Carpetas

```
backend/
├── main.go                  # Entry point: configura router y arranca servidor
├── go.mod / go.sum
├── handlers/
│   ├── health.go            # GET /api/health
│   ├── auth.go              # POST /api/auth/anonymous
│   ├── game.go              # GET /api/game/today, POST /api/game/start, POST /api/game/event
│   ├── leaderboard.go       # GET /api/leaderboard/:date
│   └── admin.go             # POST /api/admin/rotate-card
├── db/
│   ├── queries/             # Archivos .sql para sqlc
│   │   ├── users.sql
│   │   ├── cards.sql
│   │   ├── game_attempts.sql
│   │   └── leaderboard.sql
│   ├── sqlc.yaml
│   └── migrations/
│       ├── 001_init.up.sql
│       └── 001_init.down.sql
├── internal/
│   ├── config.go            # Carga de variables de entorno
│   ├── jwt.go               # Firma y validación de tokens
│   ├── db.go                # Pool de conexiones pgx
│   └── middleware/
│       ├── auth.go           # Verifica JWT en requests protegidos
│       └── admin.go          # Verifica ADMIN_SECRET en endpoints admin
└── models.go                # Structs compartidas (requests/responses JSON)
```

---

## Autenticación

### Modo anónico (MVP)

1. El frontend genera un UUID v4 en `localStorage`.
2. `POST /api/auth/anonymous` → backend crea o recupera el usuario asociado a ese UUID, genera un alias anónimo si es nuevo y devuelve JWT junto con el alias.
3. El frontend guarda el JWT y lo manda en cada request:
   ```
   Authorization: Bearer <jwt>
   ```
4. Si el usuario borra el navegador, pierde el token → crea otro usuario anónimo nuevo. Aceptable para el MVP.

### JWT Payload

```json
{
  "sub": "user_uuid",
  "iat": 1234567890,
  "exp": 1234571490
}
```

### Modo registrado (post-MVP)

Username + password, o OAuth. Se añadirá tabla `credentials` más adelante sin tocar el flujo actual.

---

## Endpoints API

### Públicos (sin auth)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/health` | Comprueba que el servidor Go responde y que la conexión a PostgreSQL funciona (`SELECT 1`). Devuelve `200` con `{"status":"ok","database":"connected"}` o `503` con `{"status":"ok","database":"disconnected"}`. |
| GET | `/api/leaderboard/:date` | Devuelve el top 100 de una fecha (YYYY-MM-DD) y la posición del jugador autenticado si está fuera del top. Requiere JWT para incluir la posición propia. |
| GET | `/api/leaderboard/today` | Igual que el endpoint por fecha, usando la carta activa de hoy. |

### Auth

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/anonymous` | Crea usuario anónimo y devuelve JWT. Body: `{ "client_id": "uuid" }`. |

### Juego (requiere JWT)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/game/today` | Devuelve la carta activa de hoy con sus 5 niveles, grids y URLs públicas de imágenes. No incluye `intruder_index`. |
| POST | `/api/game/start` | Crea un nuevo GameAttempt (`in_progress`). Devuelve `attempt_id`. El intento se considera iniciado explícitamente al renderizar el nivel 1 y enviar `level_started`. |
| POST | `/api/game/event` | Recibe eventos del juego (`level_started`, `answer_submitted`, `game_abandoned`). |
| GET | `/api/game/status` | Estado actual del usuario para la carta de hoy (para saber si reanuda). |

### Admin (requiere `X-Admin-Secret`)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/admin/rotate-card` | Recibe la carta completa, valida sus 5 niveles, archiva la anterior y activa la nueva. Llamado por cron diario o por una herramienta editorial. |

---

## Eventos del Juego (`POST /api/game/event`)

El cliente nunca declara que una respuesta es correcta, que ha fallado o que ha completado un nivel. Envía la selección realizada y el backend determina el resultado consultando `intruder_index`.

El body base es:

```json
{
  "event_id": "uuid",
  "attempt_id": "uuid",
  "event_type": "answer_submitted",
  "payload": {
    "level": 3,
    "selected_index": 7,
    "client_timestamp": "2026-07-01T09:15:31Z"
  }
}
```

**Tipos de evento aceptados:**

| `event_type` | Payload | Efecto en el backend |
|--------------|---------|----------------------|
| `level_started` | `{ level, client_timestamp }` | Registra el inicio del nivel. El servidor controla el tiempo efectivo. |
| `answer_submitted` | `{ level, selected_index, client_timestamp }` | Compara `selected_index` con `intruder_index`. Si coincide, completa el nivel; si no, suma un fallo global. |
| `game_abandoned` | `{ client_timestamp }` | Marca `interrupted = true` y `abandoned_at`. El attempt sigue como `in_progress` y puede reanudarse. |

El backend responde a `answer_submitted` con el resultado calculado, por ejemplo:

```json
{
  "correct": true,
  "level_completed": true,
  "game_status": "in_progress",
  "current_level": 4,
  "levels_reached": 3,
  "total_fails": 1
}
```

Si la respuesta incorrecta provoca el tercer fallo, responde `game_status = "game_over"` y no permite más respuestas. Si la respuesta correcta completa el nivel 5, responde `game_status = "completed"`, calcula el tiempo solo cuando `interrupted = false` y crea o actualiza la entrada del leaderboard.

**Validaciones del backend:**
- El `attempt_id` existe y pertenece al usuario del token.
- El attempt está en estado `in_progress`.
- El `level` coincide con el nivel actual del attempt.
- `selected_index` está dentro de los límites del grid del nivel.
- El evento no se procesa dos veces: el cliente debe enviar un `event_id` UUID idempotente.
- `client_timestamp` no es futuro; sirve para diagnóstico, no para calcular el tiempo oficial.
- El servidor calcula el tiempo con sus propios timestamps, nunca con el reloj del cliente.
- Cada evento debe incluir un `event_id` UUID único por intento.
- El backend persiste los eventos procesados en `game_events` con una restricción `UNIQUE(event_id)`.
- Si llega de nuevo un `event_id` ya procesado, devuelve exactamente la respuesta original sin repetir sus efectos.

### Inicio y abandono de la partida

Se adopta un inicio explícito y un abandono explícito:

- `POST /api/game/start` crea el `GameAttempt` en estado `in_progress`.
- El intento empieza a contar tiempo cuando el cliente recibe la carta, renderiza el nivel 1 y envía `level_started`.
- El frontend envía `game_abandoned` mediante `pagehide` o `visibilitychange` cuando detecta que la partida se interrumpe.
- Si el evento de abandono no llega por un cierre brusco o una pérdida de conexión, el intento permanece `in_progress` y `GET /api/game/status` lo presenta como reanudable.
- No se usa heartbeat en el MVP.
- Un intento reanudable no vuelve a ser continuo: si se marca `interrupted = true`, ese valor es permanente.

### Contrato de `GET /api/game/today`

La carta diaria se entrega completa en una única respuesta. Las imágenes se sirven mediante URLs públicas, sin firmas temporales.

### Almacenamiento de imágenes

Se adopta almacenamiento en el propio servidor, servido directamente por Caddy:

- Las imágenes viven en un volumen persistente del despliegue, por ejemplo `/data/images`.
- PostgreSQL guarda la ruta relativa o la clave lógica de cada imagen, no sus bytes.
- Caddy sirve los archivos estáticos directamente, sin pasar por el backend Go.
- El backend construye las URLs públicas a partir de la configuración de la aplicación.
- El volumen de imágenes debe incluirse en la estrategia de backups del VPS.
- La estructura de rutas debe abstraer el proveedor para permitir una migración posterior a object storage sin cambiar el contrato JSON público.

No se usan URLs firmadas ni se almacenan imágenes como `bytea` en PostgreSQL durante el MVP.

Ejemplo de respuesta:

```json
{
  "card_date": "2026-07-01",
  "card_id": "uuid",
  "levels": [
    {
      "level": 1,
      "grid_size": 3,
      "theme": "frutas",
      "images": [
        { "index": 0, "url": "https://example.com/images/card-uuid/l1-0.webp" },
        { "index": 1, "url": "https://example.com/images/card-uuid/l1-1.webp" }
      ]
    }
  ]
}
```

- La respuesta contiene los cinco niveles y todas sus imágenes.
- Cada imagen incluye un índice estable dentro de su nivel para `selected_index`.
- La respuesta no incluye `intruder_index` ni ningún campo equivalente.
- Las URLs no contienen información que revele cuál es la intrusa.
- El cliente puede precargar las imágenes de los niveles siguientes.

### Idempotencia de eventos

Se adopta una tabla de eventos procesados como registro de idempotencia y auditoría. La inserción del evento y la aplicación de sus efectos sobre `game_attempts` ocurren en la misma transacción.

- Un `event_id` solo puede pertenecer a un `attempt_id`.
- Un reintento con el mismo `event_id` y el mismo `attempt_id` devuelve la respuesta persistida originalmente.
- Un `event_id` repetido con otro `attempt_id` se rechaza con `409 Conflict`.
- La respuesta original se persiste como JSON para poder devolverla sin recalcular el evento.
- El procesamiento debe ser atómico: si falla la actualización del intento, tampoco queda registrado el evento.

### Tiempo oficial de juego

Para una partida continua, el backend mide el tiempo efectivo por nivel:

- `level_started`: guarda `level_started_at` usando el reloj del servidor.
- `answer_submitted`: guarda `answered_at` usando el reloj del servidor.
- El tiempo del nivel es `answered_at - level_started_at`.
- El intervalo entre la respuesta correcta y el siguiente `level_started` no cuenta.
- `total_time_ms` es la suma de los tiempos efectivos de los cinco niveles.
- Si `interrupted = true`, `total_time_ms` queda en `NULL` y la partida no puede entrar en el leaderboard.
- `client_timestamp` solo se conserva para diagnóstico; nunca se usa para el leaderboard.

---

## Modelo de Datos

### `users`

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username TEXT UNIQUE,
    anonymous_alias TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Los usuarios anónimos reciben un alias generado por el backend al crearse, con formato `Jugador-XXXX` (cuatro caracteres alfanuméricos en mayúsculas). El alias no contiene información personal ni deriva de forma reversible del UUID. Si por colisión ya existe el alias generado, el backend regenera otro antes de insertar el usuario.

El alias se mantiene mientras el usuario conserve su UUID y puede cambiarse en el futuro sin modificar la identidad técnica del usuario.

### `daily_cards`

```sql
CREATE TABLE daily_cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_date DATE UNIQUE NOT NULL,
    payload_hash TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### `card_levels`

```sql
CREATE TABLE card_levels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_id UUID REFERENCES daily_cards(id) ON DELETE CASCADE,
    level_number INT NOT NULL CHECK (level_number BETWEEN 1 AND 5),
    grid_size INT NOT NULL CHECK (grid_size IN (3, 4, 5)),
    theme TEXT NOT NULL,
    images JSONB NOT NULL,
    intruder_index INT NOT NULL,
    UNIQUE(card_id, level_number)
);
```

### `game_attempts`

```sql
CREATE TABLE game_attempts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    card_date DATE NOT NULL,
    status TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN ('in_progress', 'completed', 'game_over', 'expired')),
    total_time_ms INT,
    levels_reached INT NOT NULL DEFAULT 0 CHECK (levels_reached BETWEEN 0 AND 5),
    total_fails INT NOT NULL DEFAULT 0 CHECK (total_fails BETWEEN 0 AND 3),
    failed_at_level INT CHECK (failed_at_level BETWEEN 1 AND 5),
    interrupted BOOLEAN NOT NULL DEFAULT FALSE,
    eligible_for_leaderboard BOOLEAN GENERATED ALWAYS AS (
        status = 'completed' AND NOT interrupted
    ) STORED,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    abandoned_at TIMESTAMPTZ,
    UNIQUE(user_id, card_date, status)
);
```

Nota: La restricción `UNIQUE(user_id, card_date, status)` permite tener un `in_progress` y más tarde un `completed` o `game_over`, pero solo uno de cada tipo por día.

### `game_events`

```sql
CREATE TABLE game_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL UNIQUE,
    attempt_id UUID NOT NULL REFERENCES game_attempts(id) ON DELETE CASCADE,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    response JSONB NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`event_id` garantiza la idempotencia. La inserción de `game_events` y la modificación de `game_attempts` deben ejecutarse dentro de la misma transacción.

### `leaderboard_entries`

```sql
CREATE TABLE leaderboard_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_date DATE NOT NULL,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    total_time_ms INT NOT NULL,
    total_fails INT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(card_date, user_id)
);
```

El ranking se calcula ordenando por `total_time_ms ASC`, después `total_fails ASC` y finalmente `created_at ASC` como desempate estable. La API devuelve como máximo 100 entradas en `entries` y, si el request lleva un JWT válido, una entrada `current_player` con la posición propia aunque esté fuera del top 100. Los endpoints públicos siguen siendo consultables sin JWT, pero en ese caso `current_player` es `null`.

Ejemplo de respuesta:

```json
{
  "card_date": "2026-07-01",
  "entries": [
    {
      "position": 1,
      "alias": "Jugador-7F3A",
      "total_time_ms": 45000,
      "total_fails": 1
    }
  ],
  "current_player": {
    "position": 137,
    "alias": "Jugador-A12C",
    "total_time_ms": 98200,
    "total_fails": 2
  }
}
```

`position` es siempre única por el desempate temporal; no se generan posiciones compartidas. Cada entrada del leaderboard muestra únicamente `position`, `alias`, `total_time_ms` y `total_fails`. El nivel alcanzado, el estado de la partida y la fecha de finalización quedan fuera de este contrato.

---

## Flujo de una Partida (desde el backend)

```
1. GET /api/game/today
   → Devuelve la carta activa + 5 niveles + imágenes.
   → No incluye `intruder_index`; esa respuesta solo la conoce el backend.
   → Cada imagen conserva un índice estable dentro de su nivel para que el cliente envíe `selected_index`.

2. POST /api/game/start
   → Crea GameAttempt (status: in_progress).
   → Devuelve attempt_id.

3. Durante el juego, frontend envía POST /api/game/event:
   - level_started → backend valida y marca
   - fail → +1 a total_fails. Si 3 → status = game_over
   - level_completed → levels_reached++. Si 5 → game_completed
   - game_completed → calcula total_time_ms, escribe en leaderboard
   - game_abandoned → marca abandoned_at (sticky attempt, puede reanudar)

4. Al completar:
   → Backend calcula rank INSERT INTO leaderboard_entries.
   → Devuelve resultado al frontend.
```

---

## Reanudación de Partida Interrumpida

**Estrategia: Sticky Attempt (Opción A)**

- El attempt `in_progress` persiste en BD. No se crea uno nuevo al reanudar.
- Al volver, `GET /api/game/status` devuelve el estado actual:
  - `attempt_id`, `levels_reached`, `total_fails`, `interrupted = true`, nivel actual.
- El frontend reanuda desde ese nivel.
- El backend no resetea nada. El flag `interrupted` ya está fijado a `true` y nunca vuelve a `false`.
- Si el usuario llega a completar los 5 niveles, `eligible_for_leaderboard = false` porque `interrupted = true`.

---

## CORS

No se configuran cabeceras CORS en el backend Go. Caddy sirve el frontend estático y actúa como proxy reverso hacia el backend, por lo que todas las peticiones del navegador son same-origin. Si en el futuro se sirve el frontend desde otro dominio o se añade una webview en una app móvil, se activará una lista de orígenes permitidos mediante variable de entorno.

---

## Rotación Diaria de Carta (Cron)

**Trigger:** GitHub Actions (`schedule: cron '0 0 * * *'`, UTC) → POST `/api/admin/rotate-card`

**Autenticación administrativa:**

Se usan dos secretos independientes, ambos guardados en variables de entorno y nunca en el repositorio:

- `ROTATION_SECRET`: usado por el cron o proceso automático de rotación.
- `EDITOR_SECRET`: usado por la herramienta editorial o futuro dashboard.

El endpoint acepta el secreto correspondiente mediante:

```
X-Admin-Secret: <secret_from_env>
```

El backend identifica qué secreto coincide y aplica autorización por operación. Un secreto revocado no afecta al otro.

**Permisos:**

| Secreto | Operaciones permitidas |
|---------|------------------------|
| `ROTATION_SECRET` | Rotar y activar cartas mediante `POST /api/admin/rotate-card`. |
| `EDITOR_SECRET` | Crear, validar y consultar cartas en endpoints editoriales. No puede activar una carta en producción. |

El MVP implementa únicamente `POST /api/admin/rotate-card`; los endpoints editoriales quedan definidos para la siguiente fase y responderán `404` hasta estar implementados.

### Auditoría administrativa

Se registra cada operación administrativa relevante, tanto si termina correctamente como si falla:

- `id`: UUID del registro.
- `operation`: operación ejecutada, por ejemplo `rotate_card`.
- `actor`: origen lógico de la credencial (`rotation` o `editor`), nunca el secreto.
- `card_date`: fecha de la carta afectada, si aplica.
- `request_id`: identificador de correlación de la petición.
- `success`: indica si la operación terminó correctamente.
- `error_code`: código interno del error, si lo hubo; no se guardan secretos ni payloads sensibles.
- `created_at`: timestamp UTC del servidor.

La auditoría es append-only para la aplicación: no se actualizan ni eliminan registros desde los endpoints administrativos. Los logs pueden conservar el `request_id`, pero nunca `X-Admin-Secret`.

### `admin_audit_log`

```sql
CREATE TABLE admin_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation TEXT NOT NULL,
    actor TEXT NOT NULL CHECK (actor IN ('rotation', 'editor')),
    card_date DATE,
    request_id UUID NOT NULL,
    success BOOLEAN NOT NULL,
    error_code TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

La escritura de auditoría de una operación de rotación forma parte de la misma transacción que la activación de la carta cuando la operación llega a la fase transaccional. Los errores de autenticación previos a identificar una credencial válida se registran sin almacenar el secreto.
**Response body:**

En una creación normal:

```json
{
  "success": true,
  "card_date": "2026-07-01",
  "card_id": "uuid",
  "levels_created": 5,
  "expired_attempts": 12,
  "idempotent": false
}
```

En un reintento idempotente, devuelve el mismo resumen de la operación original con `idempotent: true`, sin modificar la carta ni volver a expirar intentos.

**Request body:**

```json
{
  "card_date": "2026-07-01",
  "levels": [
    {
      "level": 1,
      "grid_size": 3,
      "theme": "frutas",
      "images": [
        { "index": 0, "path": "card-2026-07-01/l1-0.webp" },
        { "index": 1, "path": "card-2026-07-01/l1-1.webp" }
      ],
      "intruder_index": 1
    }
  ]
}
```

**Acciones del endpoint:**
1. Valida el body antes de modificar la base de datos.
2. Rechaza la carta si no contiene exactamente 5 niveles numerados del 1 al 5.
3. Valida el `grid_size`, el número de imágenes, la unicidad de sus índices y que `intruder_index` apunte a una imagen existente.
4. Verifica que las rutas de imagen estén dentro del volumen permitido y no permitan traversal (`..`).
5. En una única transacción, archiva la carta activa anterior, crea `daily_cards` y sus 5 `card_levels`.
6. Si la fecha ya tiene una carta:
   - Si el payload es idéntico al ya registrado, devuelve éxito idempotente sin modificar nada.
   - Si el payload es diferente, devuelve `409 Conflict` y no sobrescribe la carta.

La carta se prepara fuera del backend; este endpoint valida y activa, pero no selecciona imágenes ni inventa contenido editorial.

**Idempotencia de la rotación:** El backend calcula una huella determinista del payload normalizado (`card_date` y niveles, imágenes, rutas e `intruder_index`). Esa huella se guarda con la carta. Un reintento con la misma huella devuelve la respuesta original; una huella distinta para la misma fecha se rechaza. La operación idempotente se registra en `admin_audit_log` como éxito, sin volver a expirar intentos ni modificar la carta.

**Fallo de rotación:** Se adopta un fallo duro. Si la validación, la escritura o la activación de la nueva carta falla, la transacción completa hace rollback y la carta anterior no se modifica. El endpoint devuelve un error estructurado y no se crea ninguna carta parcial ni de emergencia. El frontend mostrará que no hay carta disponible para la fecha nueva hasta que la rotación se ejecute correctamente.

**Errores estructurados:**

```json
{
  "success": false,
  "error": {
    "code": "INVALID_LEVELS",
    "message": "La carta debe contener exactamente 5 niveles",
    "request_id": "uuid"
  }
}
```

- `code`: código interno estable, en mayúsculas y sin espacios, que identifica el tipo de error.
- `message`: descripción legible para el cliente o el operador, sin exponer detalles internos del servidor.
- `request_id`: UUID de correlación que permite localizar la entrada correspondiente en `admin_audit_log` y en los logs del servidor.
- En errores de validación con múltiples problemas, se incluye `error.validation_errors` como lista opcional de códigos y mensajes sin exponer rutas internas ni secretos.

Códigos de error previstos para el MVP:

| `code` | HTTP | Cuándo |
|--------|------|--------|
| `UNAUTHORIZED` | 401 | Falta o es inválido `X-Admin-Secret`. |
| `INVALID_LEVELS` | 400 | No hay exactamente 5 niveles o no están numerados del 1 al 5. |
| `INVALID_GRID` | 400 | `grid_size` no es 3, 4 o 5; o el número de imágenes no coincide. |
| `INVALID_INTRUDER` | 400 | `intruder_index` fuera de rango o no apunta a una imagen existente. |
| `INVALID_PATH` | 400 | Una ruta de imagen contiene `..` o está fuera del volumen permitido. |
| `CONFLICT` | 409 | Ya existe una carta para esa fecha con un payload diferente. |
| `INTERNAL_ERROR` | 500 | Cualquier fallo no cubierto por los anteriores. |

**Intentos incompletos al cambiar el día:** Al rotar la carta, dentro de la misma transacción se marcan como `expired` los intentos `in_progress` de la carta anterior. El backend registra la transición durante la rotación y `GET /api/game/status` devuelve el estado final junto con la carta y fecha originales. Los intentos expirados no pueden reanudarse, no se convierten en `game_over` y no entran en el leaderboard. El intento conserva la carta, fecha y eventos originales para consulta histórica.

**Protección:** El endpoint rechaza requests sin `X-Admin-Secret` válido. No depende de IP ni de auth de usuario.

---

## Validaciones y Límites

### Rate limiting por endpoint

| Endpoint | Límite | Ámbito |
|----------|--------|--------|
| `POST /api/auth/anonymous` | 3 por hora | Por IP |
| `POST /api/game/start` | 1 cada 5 segundos | Por IP |
| `POST /api/game/event` | 10 por minuto | Por usuario (JWT) |
| `GET /api/leaderboard/*` | 30 por minuto | Por IP |
| `POST /api/admin/rotate-card` | 5 por minuto | Por IP |
| Resto de endpoints | 60 por minuto | Por IP |

El rate limiting se implementa con un middleware en `chi` que usa un contador en memoria del backend. No requiere Redis ni almacenamiento externo para el MVP. Al superar el límite, el backend devuelve `429 Too Many Requests` con una cabecera `Retry-After` indicando los segundos de espera.

### Otras validaciones

| Regla | Comportamiento |
|-------|----------------|
| Tiempos absurdos | Si `total_time_ms < 5000` para 5 niveles, se marca como sospechoso en logs pero NO se rechaza en MVP. |
| Eventos fuera de orden | Si llega `answer_submitted` para nivel 3 pero `levels_reached = 1`, devuelve HTTP 400. |
| Timestamp futuro | Si el evento tiene `client_timestamp` futuro, devuelve HTTP 400. |
| Un intento por día | Si ya existe un attempt `completed` o `game_over` para hoy, `POST /api/game/start` devuelve 409 Conflict. |
| Reanudación ilimitada | Un attempt `in_progress` (con `interrupted = true`) puede reanudarse infinitas veces. |

---

*Documento vivo. Si cambiamos contratos de API o schema, se actualiza aquí primero.*
