# Finding — Configuracion del Agente

> Reglas operativas para el asistente al trabajar en este proyecto. Commits, convenciones, stack.

## Formato de commits

- Mensaje corto: `tipo: frase de maximo 50 caracteres`
- Tipos validos: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`
- Ejemplo: `docs: add wireframes v2 — 9 screens total`

## Stack

- Frontend: Svelte 5 + Vite + CSS puro scopado (sin Tailwind)
- Backend: Go 1.24+ (`chi`, `pgx`, `sqlc`, JWT)
- Infra: Docker Compose (Caddy + Go + Postgres) en Hetzner VPS
- Dev en Raspberry Pi, deploy a VPS

## Reglas del proyecto

- Docs son fuente unica de verdad — se actualizan antes que el codigo
- Specs primero, codigo despues
- Siempre 2-3 opciones con trade-offs antes de [DECISION]
- Sin ORMs magicos, sin Tailwind, sin React

## Auth

- Usuarios anonimos: UUID en localStorage + JWT
- Usuarios registrados: post-MVP (tabla `credentials`)

## Contacto

- Autor: Aker (Asier)
- Email: athilha+pi@gmail.com
- GitHub: aarizkurenbot
