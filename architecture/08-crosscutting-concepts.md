# Chapter 8: Cross-Cutting Concepts

## 8.1 Authentication and Authorisation

### Registration and password storage

Passwords are hashed with **bcrypt** (via the `bcrypt` Python library) before being stored in SQLite. bcrypt is an adaptive hash: its work factor can be increased as hardware improves, without invalidating existing hashes. Plain-text passwords are never logged or stored.

### Session tokens

On successful login, `TokenService` issues a **signed JWT** (algorithm: HS256) using `PyJWT`. The token payload carries:

| Claim | Value |
|---|---|
| `sub` | `player_id` (UUID) |
| `exp` | Issue time + 24 hours |
| `iat` | Issue time |

The signing key (`JWT_SECRET`) is injected at runtime via an environment variable and never committed to the repository.

### Request authentication

Every endpoint in the Match Blueprint requires a valid JWT. The Auth Blueprint provides a `require_auth` decorator that:

1. Reads the `Authorization: Bearer <token>` header.
2. Calls `TokenService.verify_token(token)` → returns `player_id` or raises.
3. Attaches `player_id` to Flask's `g` object for the duration of the request.
4. Returns `401 Unauthorized` if the token is missing, malformed, or expired.

### Authorisation within a match

After authentication, `MatchService` enforces the following checks on every match endpoint:

- **Participant check (all endpoints including `GET /matches/{id}`):** the authenticated `player_id` must be one of the two participants of the requested match. Non-participants receive `403 Forbidden`.
- **Turn check (write endpoints only):** for `POST /matches/{id}/cards` and `POST /matches/{id}/end-turn`, `MatchService` additionally verifies it is the authenticated player's turn. Acting out of turn returns `403 Forbidden`.
- **Hand visibility (`GET /matches/{id}`):** the response omits the opponent's hand. Each player may only see their own cards; the opponent's hand size is included but not the card values.

---

## 8.2 Error Handling

All error responses use a consistent JSON envelope:

```json
{
  "error": {
    "code": "INSUFFICIENT_MANA",
    "message": "Card costs 4 mana but player only has 2."
  }
}
```

### HTTP status code mapping

| Situation | Status code |
|---|---|
| Missing or invalid request fields | `400 Bad Request` |
| Missing, malformed, or expired JWT | `401 Unauthorized` |
| Valid JWT but wrong player or wrong turn | `403 Forbidden` |
| Resource not found (match, player) | `404 Not Found` |
| Duplicate username on registration | `409 Conflict` |
| Match already finished | `409 Conflict` |
| Business rule violation (e.g. insufficient mana) | `422 Unprocessable Entity` |
| Unexpected server error | `500 Internal Server Error` |

### Domain exceptions

The domain layer raises typed Python exceptions (e.g. `InsufficientManaError`, `NotYourTurnError`). The Flask layer catches these in a centralised error handler registered with `app.register_error_handler` and maps them to the appropriate HTTP status codes and JSON envelopes. The domain layer itself never constructs HTTP responses.

### Unhandled exceptions

A catch-all handler logs the full stack trace (see §8.3) and returns `500 Internal Server Error` with a generic message. Stack traces are never included in API responses.

---

## 8.3 Logging

The system uses Python's standard `logging` module configured at application startup.

| Level | Used for |
|---|---|
| `DEBUG` | Detailed domain state changes (disabled in production) |
| `INFO` | Incoming requests, match lifecycle events (match created, turn started, game won) |
| `WARNING` | Recoverable anomalies (e.g. a client sending a request out of sequence) |
| `ERROR` | Caught exceptions with stack traces |
| `CRITICAL` | Startup failures (e.g. missing `JWT_SECRET`) |

Log lines are written to stdout in JSON format in production (structured logging, one JSON object per line) and in human-readable format during local development. The `FLASK_ENV` environment variable controls the format.

Sensitive values (passwords, JWT tokens) are never written to logs.

---

## 8.4 Input Validation

Request validation happens at the Flask layer, before any domain code runs, using a lightweight schema check (e.g. via `marshmallow` or manual checks):

- Required fields are present and non-empty.
- Field types match expectations (e.g. `card_index` is a non-negative integer).
- String lengths are within bounds (e.g. username: 3–32 characters).

Validation failures return `400 Bad Request` with a JSON body listing the offending fields. The domain layer assumes its inputs are already valid and raises typed domain exceptions only for business-rule violations.

---

## 8.5 Domain Events

Domain events are raised by `TurnEngine` and accumulated on the `Match` aggregate during a single request. They are **in-process and synchronous**: no message broker is involved.

| Event | Raised by | Consumed by |
|---|---|---|
| `CardPlayed` | `TurnEngine.play_card` | `MatchService` (included in API response) |
| `PlayerDamaged` | `TurnEngine.play_card` | `MatchService` (included in API response) |
| `CardDiscarded` | `TurnEngine.begin_turn` | `MatchService` (included in API response) |
| `BleedingOut` | `TurnEngine.begin_turn` | `MatchService` (included in API response) |
| `TurnEnded` | `TurnEngine.end_turn` | `MatchService` (included in API response) |
| `GameWon` | `TurnEngine.play_card` (opponent health ≤ 0 after damage) or `TurnEngine.begin_turn` (active player health ≤ 0 after Bleeding Out) | `MatchService` — sets `Match.status = FINISHED`; included in API response |

After `MatchService` saves the updated `Match`, it serialises the accumulated event list and includes it in the API response. The SPA uses the event list to animate what happened during the turn (damage dealt, cards discarded, etc.) without having to diff two state snapshots.

Events are cleared from the aggregate at the start of each new request; they are not persisted to the database.
