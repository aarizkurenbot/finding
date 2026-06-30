# Finding — Reglas del Juego

> Documento de reglas de negocio y flujo de partida. Única fuente de verdad para el comportamiento del juego.

---

## Concepto

Juego diario de observación visual. Cada día se publica una "carta" con 5 niveles. En cada nivel, un grid de imágenes sobre una temática común contiene una imagen intrusa que no encaja. El objetivo es encontrarla en el menor tiempo posible.

---

## Ciclo Diario

- **Una carta por día**. Todos los jugadores ven la misma carta.
- **Un intento por jugador por día**. No hay reintentos si fallas.
- **Reset a las 00:00 UTC** (o zona horaria configurada). La carta anterior se archiva; la nueva se activa.

---

## Vidas (Fallos)

- **2 fallos permitidos por partida** (global, no por nivel).
- Al tercer fallo: **GAME OVER inmediato**.
- Partida terminada. No se puede jugar de nuevo hasta mañana.

---

## Estructura de una Partida

### Niveles (5 en total)

| Nivel | Grid      | Dificultad esperada                           |
|-------|-----------|-----------------------------------------------|
| 1     | 3×3       | Temática obvia, intrusa visible a simple vista |
| 2     | 4×4       | Temática clara, intrusa un poco más sutil     |
| 3     | 4×4       | Imágenes más parecidas entre sí                 |
| 4     | 5×5       | Intrusa que casi encaja con el tema             |
| 5     | 5×5       | Máxima sutileza o distractor adicional          |

- El jugador debe completar los 5 niveles en orden.
- No se puede saltar niveles.

### Timer

- El **timer comienza** cuando se renderiza el primer nivel.
- El **timer se pausa** durante las transiciones entre niveles (1-2 segundos).
- El **timer se detiene** al completar el nivel 5 (victoria) o al tercer fallo (game over).
- El tiempo mostrado es el **tiempo efectivo de juego**, sin contar transiciones.

---

## Estados del Jugador

Un jugador, para una carta del día dada, puede estar en uno de estos estados:

### 1. NO_JUGADO
- Nunca ha iniciado la partida de hoy.
- Landing: botón **"Jugar hoy"**.

### 2. EN_PROGRESO (partida abandonada)
- Inició la partida pero cerró la app/navegador antes de completarla o de hacer game over.
- Landing: botón **"Continuar partida"**.
- **Aviso obligatorio**: esta partida no entra en leaderboard ni registra tiempo.
- Puede continuar las veces que quiera hasta completarla o hacer game over.
- El timer no se muestra ni se registra.

### 3. COMPLETADO
- Terminó los 5 niveles.
- Landing: resultado (tiempo, fallos, ranking) + cuenta atrás hasta la siguiente carta + acceso a leaderboard.
- El tiempo solo cuenta para leaderboard si la partida se completó **de una sentada** (sin abandonos).

### 4. GAME_OVER
- Llegó a 3 fallos.
- Landing: "Partida terminada. Nivel X — 3 fallos." + cuenta atrás hasta mañana.
- **No hay botón de reintentar.**
- Puede ver el leaderboard pero no volver a jugar hasta el día siguiente.

---

## Regla Crítica: Partida Contínua vs. Interrumpida

|                                 | Partida contínua        | Partida interrumpida (abandonada) |
|---------------------------------|------------------------|-----------------------------------|
| Timer                           | Sí, cuenta para ranking | No, no se muestra ni registra     |
| Entrada en leaderboard          | Sí                     | No                                |
| Fallos permitidos               | 2 (3º = game over)     | 2 (3º = game over)                |
| Puede reanudar                  | N/A                    | Sí, infinitas veces               |
| Puede reintentar tras game over | No                     | No                                |

**Cómo se determina si es contínua:**
- Al iniciar la partida, el servidor crea un `GameAttempt` con `interrupted = false`.
- Si el frontend detecta que la pestaña/app se cerró (o hubo inactividad extrema), marca el attempt como `interrupted = true`.
- Una vez `interrupted = true`, nunca vuelve a `false`.
- El jugador puede seguir jugando desde donde iba, pero ese flag queda fijado.

---

## Flujo de una Partida (Diagrama)

```
LANDING
  │
  ├── NO_JUGADO ──→ [Pulsa "Jugar"] ──→ INSTRUCCIONES ──→ NIVEL 1
  │                                                   │
  ├── EN_PROGRESO ──→ [Pulsa "Continuar"] ──────────────┘ (sin timer)
  │
  ├── COMPLETADO ──→ Resultado + Leaderboard
  │
  └── GAME_OVER ──→ "Hasta mañana" + Leaderboard


DENTRO DE LA PARTIDA (niveles 1-5):

  NIVEL N ──→ toca imagen ──→ ¿Acierto?
                                    │
                       Sí ←─────────┘───→ No
                       │                    │
                       │              +1 fallo global
                       │                    │
                       │              ¿3 fallos?
                       │                    │
             NIVEL N+1 (si N<5)      Sí ←──┘──→ No
                       │                    │
               VICTORIA (si N=5)      sigue en NIVEL N
                       │
              Fin. Timer para.
              Guarda tiempo.
              Entra en leaderboard.
```

---

## Leaderboard

- **Solo entran partidas COMPLETADAS y CONTÍNUAS** (`status = completed` + `interrupted = false`).
- Ordenado por: (1) tiempo total ascendente, (2) fallos totales ascendente.
- Se actualiza en tiempo real (o casi). Un jugador puede bajar de puesto durante el día.
- Cualquiera puede ver el leaderboard, incluso sin haber jugado hoy o habiendo abandonado.

---

## Compartir Resultado

- Disponible en la pantalla de victoria.
- Texto precompuesto, sin spoilers de imágenes ni temática.
- Ejemplo: *"Hoy resolví 5 niveles en 45s con 3 fallos en Finding. ¿Me superas?"*

---

## Entidades de Datos (Resumen)

```yaml
GameAttempt:
  id: UUID
  user_id: UUID
  card_date: YYYY-MM-DD
  status: in_progress | completed | game_over
  total_time_ms: integer | null  # null si interrupted
  levels_reached: 0..5
  total_fails: 0..3
  failed_at_level: 1..5 | null
  interrupted: boolean            # true si hubo cierre/abandono
  eligible_for_leaderboard: boolean  # true solo si completed + !interrupted
  started_at: timestamp
  completed_at: timestamp | null
  abandoned_at: timestamp | null

LeaderboardEntry (vista/materializada):
  card_date: YYYY-MM-DD
  user_id: UUID
  total_time_ms: integer
  total_fails: 0..2
  rank: integer
  created_at: timestamp
```

---

## Decisiones de Diseño Aclaradas

| Decisión | Valor elegido |
|----------|---------------|
| ¿Timer en instrucciones? | No. Empieza al renderizar nivel 1. |
| ¿Penalización por fallo? | No. Solo se cuentan. |
| ¿Reintentos tras game over? | No. Un intento por día. |
| ¿Imágenes se reordenan al fallar? | No. Fijas. |
| ¿Se muestran intrusas al final? | Sí, galería de las 5 encontradas. |
| ¿Leaderboard visible tras jugar? | Sí, siempre. |
| ¿Qué pasa si cierra a mitad? | Marca como interrupted, puede continuar sin timer ni leaderboard. |
| ¿Timer pausado en transiciones? | Sí, solo cuenta tiempo efectivo. |

---

## Backlog (fuera del MVP)

- Patrocinios de cartas (marcas eligen temática e intrusa).
- Sistema de premios/sorteos.
- Streaks de días jugados.
- Estadísticas personales históricas.
- Gráficas de evolución.
- Panel de administración para curar cartas manualmente.
- Antifraude (detección de tiempos imposibles, multi-cuenta).
