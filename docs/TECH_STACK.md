# Finding — Stack Técnico y Arquitectura

> Decisión de stack para el MVP. Objetivo: simple, autocontenido, fácil de desplegar en un VPS sin magia negra.

---

## Filosofía

- Un solo VPS, Docker Compose, Caddy como proxy.
- El backend compila a un binario y ocupa pocos megas. No necesita un runtime pesado.
- El frontend es estático. No SSR, no hidratación, no framework pesado.
- La base de datos corre en un contenedor con volumen persistente. Backups explícitos.
- Deploy vía SSH desde GitHub Actions. Cero clicks manuales en producción.

---

## Componentes

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| Frontend | Svelte 5 + Vite | Reactividad sin runtime pesado, componentes scopados, transiciones built-in. Cubre juego + dashboard admin. |
| UI | CSS puro (scoped por componente) | Svelte scopea automáticamente. CSS nativo 2025 tiene nesting, custom properties, `@layer`. Sin Tailwind, sin SCSS. |
| Bundler / Dev server | Vite | Es el estándar actual. Rápido, sencillo, HMR sin dolor. |
| Backend | Go 1.24+ | Binario nativo, bajo consumo de RAM, sin dependencias de runtime. Para un CRUD + auth + timer, es directo. |
| Router HTTP | `chi` | Minimalista, compatible con middleware estándar de Go. Si prefieres algo más completo, `echo` también vale. |
| Driver PostgreSQL | `pgx/v5` | Driver moderno, typed, mejor que `lib/pq` (deprecated). |
| Query builder / ORM | `sqlc` | Escribes SQL puro, genera Go tipado. Sin ORM mágico, sin reflexión. |
| Migrations | `golang-migrate` | Corre en el init del contenedor. Schema versionado en archivos `.sql`. |
| Auth / Sesión | JWT (`golang-jwt/jwt/v5`) | Tokens de sesión para usuarios anónimos o registrados. Sin cookies sesión-server. |
| Base de datos | PostgreSQL 16 | Contenedor con volumen Docker. Para el MVP, más que suficiente. |
| Proxy + HTTPS | Caddy | Auto-HTTPS con Let's Encrypt. Config de una línea. Sirve también el front estático. |
| Contenedores | Docker + Docker Compose | Un solo `docker-compose.yml` levanta todo. |
| CI / Deploy | GitHub Actions | Build + test + SSH al VPS + `docker compose up --build -d`. |
| Cron diario | GitHub Actions (`schedule`) | Llama a `POST /api/admin/rotate-card` a las 00:00 UTC. |
| Backups DB | `pg_dump` + script | Diario a un bucket S3-compatible (Backblaze B2, Wasabi). |

---

## Diagrama de Arquitectura

```
                           Internet
                              │
                              ▼
                    ┌─────────────────┐
                    │   Caddy Server  │  ← Puerto 80/443, auto-HTTPS
                    │   (reverse proxy + sirve estáticos)
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │               │
              ▼              ▼               ▼
         ┌────────┐   ┌──────────┐   ┌────────────┐
         │ /api/* │   │ /* (SPA) │   │ /static/*  │
         │   →    │   │   →      │   │    →       │
         │  Go    │   │  build/  │   │  build/    │
         │ :8080  │   │ (index.html, JS, CSS, imgs)
         └────────┘   └──────────┘   └────────────┘
              │
              │ (red Docker interna)
              ▼
         ┌──────────┐
         │ Postgres │  ← Puerto 5432, solo interno
         │   :5432  │
         └──────────┘
```

---

## Estructura del Repositorio

```
finding/
├── docker-compose.yml          # Levanta Caddy, backend, postgres
├── .github/
│   └── workflows/
│       ├── deploy.yml            # Build + deploy en push a main
│       └── rotate-card.yml       # Cron diario: POST /api/admin/rotate-card
│
├── frontend/
│   ├── package.json              # Vite + TypeScript
│   ├── vite.config.ts
│   ├── index.html
│   ├── src/
│   │   ├── main.ts               # Entry point
│   │   ├── router.ts             # Navegación simple (landing → game → result)
│   │   ├── game/
│   │   │   ├── Game.ts           # Lógica de partida (timer, niveles, fallos)
│   │   │   ├── LevelRenderer.ts  # Renderizado del grid
│   │   │   └── types.ts
│   │   ├── api.ts                # Cliente fetch tipado
│   │   └── styles/
│   │       └── main.css          # Tailwind o CSS puro
│   └── dist/                     # Output del build (Caddy sirve esto)
│
├── backend/
│   ├── go.mod
│   ├── main.go                   # Entry point: configura router y arranca servidor
│   ├── handlers/
│   │   ├── game.go               # GET /api/game/today, POST /api/game/attempt, etc.
│   │   ├── leaderboard.go        # GET /api/leaderboard/:date
│   │   ├── auth.go               # POST /api/auth/register (anónimo), /login
│   │   └── admin.go              # POST /api/admin/rotate-card (protegido)
│   ├── db/
│   │   ├── queries/              # Archivos .sql para sqlc
│   │   │   ├── users.sql
│   │   │   ├── game_attempts.sql
│   │   │   └── leaderboard.sql
│   │   ├── sqlc.yaml             # Config de generación
│   │   └── migrations/           # Archivos .sql versionados
│   │       ├── 001_init.up.sql
│   │       └── 001_init.down.sql
│   └── internal/
│       ├── config.go             # Carga de variables de entorno
│       ├── jwt.go                # Firma y validación de tokens
│       └── models.go             # Structs compartidas (si no generadas por sqlc)
│
└── docs/
    ├── GAME_RULES.md
    └── TECH_STACK.md             # Este archivo
```

---

## Flujo de Deploy

**1. Push a `main`**

```
git push origin main
```

**2. GitHub Actions ejecuta `.github/workflows/deploy.yml`**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Build del frontend
      - name: Build frontend
        run: |
          cd frontend
          npm ci
          npm run build

      # Build del backend
      - name: Build backend
        run: |
          cd backend
          go build -o finding-server ./main.go

      # Copiar al VPS vía SSH (rsync o scp)
      - name: Deploy to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          source: "frontend/dist/,backend/finding-server,docker-compose.yml,migrations/"
          target: "/opt/finding"

      # SSH al VPS y levantar contenedores
      - name: Restart services
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/finding
            docker compose up --build -d
```

**3. En el VPS**

Caddy se reinicia con la nueva config. El backend compila y arranca. Postgres ya estaba corriendo con su volumen persistente.

---

## Variables de Entorno (`.env` en el VPS, nunca en el repo)

```bash
# Postgres
POSTGRES_USER=finding
POSTGRES_PASSWORD=...
POSTGRES_DB=finding
POSTGRES_URL=postgres://finding:...@postgres:5432/finding?sslmode=disable

# Backend
JWT_SECRET=...
ADMIN_SECRET=...          # Para proteger /api/admin/rotate-card
PORT=8080

# Caddy
DOMAIN=find-game.example.com
```

---

## Decisiones de Diseño Técnico

### ¿Por qué no React / Next.js / Remix?

El flujo del juego es secuencial, sí, pero el dashboard de admin (tablas de cartas, formularios de creación, estadísticas de uso) justifica un framework. Svelte cubre ambos casos: el juego es trivial en Svelte (estado reactivo con variables normales, transiciones built-in) y el admin también (tablas, formularios, bindings bidireccionales sin boilerplate). No necesitamos React para esto.

### ¿Por qué `sqlc` y no GORM / ent / Prisma?

`sqlc` te escribe el Go por ti a partir de SQL que ya sabes escribir. No hay DSL de ORM que aprender, no hay migraciones incompatibles, no hay queries N+1 ocultas. Si sabes SQL, sabes `sqlc`. El código generado es Go puro y plano.

### ¿Por qué JWT en vez de sesiones server-side?

El backend es stateless. No necesita Redis ni tabla de sesiones. El token JWT va en el header `Authorization`. El frontend lo guarda en `localStorage`. Si el usuario borra el navegador, pierde el token y crea otro usuario anónimo. Para el MVP, eso es aceptable.

### ¿Por qué Caddy y no nginx?

Config de HTTPS en una línea: `tls internal` o auto-Let's Encrypt. Config de reverse proxy en dos líneas. Sirve archivos estáticos por defecto. Para un solo VPS con un par de servicios, Caddy elimina fricción.

### ¿Por qué no un CDN externo para las imágenes?

En el MVP las imágenes van en Supabase Storage, S3, o incluso en el propio VPS (`/opt/finding/images/` servidas por Caddy). Si el juego tiene éxito, se migra a Cloudflare R2 o similar sin tocar el backend. Prematuro optimizar ahora.

---

## Backlog Técnico (post-MVP)

| Item | Cuándo |
|------|--------|
| CDN para imágenes | Cuando el tráfico de imágenes supere el ancho de banda del VPS |
| Redis para cache de leaderboard | Cuando el leaderboard tenga >1000 entries y las queries se noten lentas |
| Observabilidad (logs, métricas) | Cuando haya usuarios reales reportando bugs |
| Test suite E2E (Playwright) | Cuando el frontend crezca y haya regresiones |
| Migración a React (opcional) | Si el equipo crece y necesita un framework de equipo |

---

## Notas sobre el VPS (Hetzner)

**Recomendación de instancia para el MVP:**
- CPX11 (2 vCPU, 4GB RAM, 40GB NVMe) → ~5€/mes. Sobra para el MVP.
- Si necesitas más, escalas verticalmente en minutos.

**SO recomendado:** Ubuntu 24.04 LTS. Instalar Docker y Docker Compose (`docker.io` + `docker-compose-plugin` del repo de Docker, no el de Ubuntu).

**Seguridad básica:**
- Firewall UFW: solo 22 (SSH), 80 (HTTP), 443 (HTTPS).
- SSH solo por key, password login deshabilitado.
- Fail2ban para bloquear brute-force.
- Docker no expone Postgres al exterior (solo red interna).

**Backup de base de datos:**
```bash
# Script diario via cron en el VPS
pg_dump "${POSTGRES_URL}" | gzip > /backups/finding-$(date +%Y%m%d).sql.gz
# Subir a B2/Wasabi con rclone o similar
```

---

## Próximos Pasos

1. Crear el repo con la estructura de carpetas.
2. `go mod init finding` en `/backend`.
3. `npm create vite@latest frontend -- --template vanilla-ts`.
4. Escribir `docker-compose.yml` con Caddy + Go + Postgres.
5. Escribir la primera migration (`001_init.sql`) y generar código con `sqlc`.
6. Implementar el endpoint mínimo: `GET /api/game/today` que devuelva la carta activa.
7. Implementar la landing del frontend que consuma ese endpoint.

---

*Documento vivo. Si cambiamos algo del stack, se actualiza aquí antes de tocar código.*
