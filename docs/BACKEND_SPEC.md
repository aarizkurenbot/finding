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
2. `POST /api/auth/anonymous` → backend crea usuario con ese UUID, devuelve JWT.
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
| GET | `/api/health` | Estado del servidor y conexión a BD. |
| GET | `/api/leaderboard/:date` | Ranking de una fecha (YYYY-MM-DD). |
| GET | `/api/leaderboard/today` | Leaderboard de la carta activa hoy. |

### Auth

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/anonymous` | Crea usuario anónimo y devuelve JWT. Body: `{ "client_id": "uuid" }`. |

### Juego (requiere JWT)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/game/today` | Devuelve la carta activa de hoy con sus 5 niveles. |
| POST | `/api/game/start` | Crea un nuevo GameAttempt (in_progress). Devuelve `attempt_id`. |
| POST | `/api/game/event` | Recibe eventos del juego (level_started, fail, level_completed, game_completed, game_abandoned). |
| GET | `/api/game/status` | Estado actual del usuario para la carta de hoy (para saber si reanuda). |

### Admin (requiere `X-Admin-Secret`)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/admin/rotate-card` | Cierra la carta anterior, crea la nueva. Llamado por cron diario. |

---

## Eventos del Juego (`POST /api/game/event`)

Un solo endpoint para todos los eventos. El body incluye el tipo:

```json
{
  "attempt_id": "uuid",
  "event_type": "level_completed",
  "payload": {
    "level": 3,
    "fails": 1,
    "timestamp": "2026-07-01T09:15:31Z"
  }
}
```

**Tipos de evento:**

| `event_type` | Payload | Efecto en el backend |
|--------------|---------|----------------------|
| `level_started` | `{ level, timestamp }` | Marca nivel actual. Timer corre. |
| `fail` | `{ level, timestamp }` | +1 a `total_fails`. Si llega a 3 → `status = game_over`. |
| `level_completed` | `{ level, timestamp }` | Incrementa `levels_reached`. Avanza al siguiente nivel. |
| `game_completed` | `{ timestamp }` | `status = completed`. Calcula `total_time_ms`. Escribe en leaderboard. |
| `game_abandoned` | `{ timestamp }` | Marca `abandoned_at`. El attempt sigue como `in_progress` (sticky). |

**Validaciones del backend:**
- El `attempt_id` existe y pertenece al usuario del token.
- El attempt está en estado `in_progress`.
- El `timestamp` no es futuro ni del año pasado.
- El nivel del evento es coherente con `levels_reached` (no puedes completar el nivel 3 si solo has llegado al 1).

---

## Modelo de Datos

### `users`

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username TEXT UNIQUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### `daily_cards`

```sql
CREATE TABLE daily_cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_date DATE UNIQUE NOT NULL,
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
    images TEXT[] NOT NULL,
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
    status TEXT NOT NULL DEFAULT 'in_progress',
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

### `leaderboard_entries`

```sql
CREATE TABLE leaderboard_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_date DATE NOT NULL,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    total_time_ms INT NOT NULL,
    total_fails INT NOT NULL,
    rank INT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(card_date, user_id)
);
```

---

## Flujo de una Partida (desde el backend)

```
1. GET /api/game/today
   → Devuelve la carta activa + 5 niveles + imágenes.
   → El frontend conoce la intrusa por índice (no se envía al cliente, pero está en el array).

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

## Rotación Diaria de Carta (Cron)

**Trigger:** GitHub Actions (`schedule: cron '0 0 * * *'`) → POST `/api/admin/rotate-card`

**Header requerido:**
```
X-Admin-Secret: <secret_from_env>
```

**Acciones del endpoint:**
1. Marca la carta de ayer (si existe) como `archived`.
2. Crea nueva fila en `daily_cards` con `card_date = CURRENT_DATE` y `status = 'active'`.
3. Crea los 5 registros en `card_levels` asociados a esa carta.
4. En el MVP, los datos de niveles pueden ser de prueba o hardcodeados.

**Protección:** El endpoint rechaza requests sin `X-Admin-Secret` válido. No depende de IP ni de auth de usuario.

---

## Validaciones y Límites

| Regla | Comportamiento |
|-------|----------------|
| Rate limiting en `/api/game/start` | Máximo 1 llamada cada 5 segundos por IP. |
| Tiempos absurdos | Si `total_time_ms < 5000` para 5 niveles, se marca como sospechoso en logs pero NO se rechaza en MVP. |
| Eventos fuera de orden | Si llega `level_completed` para nivel 3 pero `levels_reached = 1`, devuelve HTTP 400. |
| Timestamp futuro | Si el evento tiene fecha futura, devuelve HTTP 400. |
| Un intento por día | Si ya existe un attempt `completed` o `game_over` para hoy, `POST /api/game/start` devuelve 409 Conflict. |
| Reanudación ilimitada | Un attempt `in_progress` (con `interrupted = true`) puede reanudarse infinitas veces. |

---

*Documento vivo. Si cambiamos contratos de API o schema, se actualiza aquí primero.*
