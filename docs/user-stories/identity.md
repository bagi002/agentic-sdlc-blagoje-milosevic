# IDENTITY-STORY-001: Player Registration

**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (White-Box); `architecture/06-runtime-view.md` — §6.1 Player Registration
**Pareto**: Core

```
AS A new player
I WANT to register an account with a username and password
SO THAT I can log in and participate in matches
```

## Acceptance Criteria (end-to-end)

### SCENARIO 1: Successful registration

**Scenario ID**: IDENTITY-STORY-001-S1

**GIVEN**
* the system is running
* no account with username `alice` exists

**WHEN**
* the player submits `POST /auth/register` with `{ "username": "alice", "password": "Secret123" }` <!-- pragma: allowlist secret -->

**THEN**
* the response status is `201 Created`
* a `Player` row exists in the `players` table for username `alice`
* the stored password is a bcrypt hash, not the plain-text value
* no JWT token is returned in the response

---

### SCENARIO 2: Duplicate username rejected

**Scenario ID**: IDENTITY-STORY-001-S2

**GIVEN**
* an account with username `alice` already exists

**WHEN**
* a second `POST /auth/register` is submitted with the same username `alice`

**THEN**
* the response status is `409 Conflict`
* the response body contains `{ "error": { "code": "USERNAME_TAKEN", ... } }`
* the `players` table still has exactly one row for `alice`

---

### SCENARIO 3: Missing field rejected

**Scenario ID**: IDENTITY-STORY-001-S3

**GIVEN**
* the system is running

**WHEN**
* `POST /auth/register` is submitted with `{ "username": "alice" }` (no `password` field)

**THEN**
* the response status is `400 Bad Request`
* the response body names the missing `password` field

---

## INFRA Stories

### IDENTITY-INFRA-001.1: Deploy the Identity service container

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/07-deployment-view.md` — §7.2 Containers; §7.3 Docker Compose Structure

Deploy the `api` container (Python 3.12-slim + Gunicorn) and the `spa` container (nginx:alpine) via Docker Compose so that `POST /api/auth/register` is reachable from the host through the Nginx reverse proxy on port 80.

#### SCENARIO 1: Both containers start and the register endpoint is reachable

**Scenario ID**: IDENTITY-INFRA-001.1-S1

**GIVEN**
* `docker-compose.yml` defines the `spa` service (nginx:alpine, port 80, `depends_on: api`) and the `api` service (python:3.12-slim, Gunicorn, `JWT_SECRET` environment variable, `tcg_data` volume mounted at `/data`, internal port 5000)
* the `spa` Nginx configuration reverse-proxies all `/api/*` requests to the `api` container on port 5000

**WHEN**
* `docker compose up --build` is executed

**THEN**
* both the `spa` and `api` containers are running
* `POST http://localhost:80/api/auth/register` with a valid JSON body returns a response (any HTTP status from the application, confirming the full proxy path is live)

---

### IDENTITY-INFRA-001.2: Create the `players` table with an explicit migration

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context; `architecture/02-constraints.md` — CV-05 Explicit migrations

Create the `players` SQLite table via an explicit migration script that runs at container startup. The schema must include `player_id` (UUID, primary key), `username` (text, unique, not null), and `hashed_password` (text, not null).

#### SCENARIO 1: Migration creates the table on a fresh database

**Scenario ID**: IDENTITY-INFRA-001.2-S1

**GIVEN**
* the `tcg_data` volume contains no `tcg.db` file (first run)

**WHEN**
* the `api` container starts and executes the migration script

**THEN**
* `tcg.db` exists on the volume
* `PRAGMA table_info(players)` returns columns `player_id`, `username`, `hashed_password`
* a `UNIQUE` constraint exists on `username`

#### SCENARIO 2: Migration is idempotent on restart

**Scenario ID**: IDENTITY-INFRA-001.2-S2

**GIVEN**
* `tcg.db` already contains the `players` table with one row

**WHEN**
* the `api` container is restarted and the migration script runs again

**THEN**
* the existing row is still present
* no error is raised (`CREATE TABLE IF NOT EXISTS` semantics)

---

### IDENTITY-INFRA-001.3: Domain events — not applicable for registration

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.5 Domain Events; `architecture/09-architecture-decisions.md` — ADR-004

The registration flow does not raise any domain events. Domain events are defined for the Game bounded context only (`CardPlayed`, `TurnEnded`, `GameWon`, etc.). No event infrastructure is required for this story.

---

### IDENTITY-INFRA-001.4: Structured logging for registration requests

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.3 Logging

Configure Python's `logging` module at `api` startup to write `INFO`-level entries for each registration request and `ERROR`-level entries with stack traces for unexpected failures. In production (`FLASK_ENV=production`) log lines are JSON objects on stdout; in development they are human-readable.

#### SCENARIO 1: Successful registration emits an INFO log

**Scenario ID**: IDENTITY-INFRA-001.4-S1

**GIVEN**
* the `api` container is running with `FLASK_ENV=production`

**WHEN**
* a successful `POST /auth/register` request is processed

**THEN**
* stdout contains a JSON log line at level `INFO` that includes the event name (e.g. `player_registered`) and does **not** include the plain-text password or any hash

#### SCENARIO 2: Passwords are never logged

**Scenario ID**: IDENTITY-INFRA-001.4-S2

**GIVEN**
* the `api` container is running

**WHEN**
* any registration request (successful or failed) is processed

**THEN**
* no log line at any level contains the plain-text password or the bcrypt hash

---

## BE Stories

### IDENTITY-BE-001.1: `PlayerRepository` — save and unique-check

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (PlayerRepository component)

Implement `PlayerRepository.save(player: Player) -> None` and `PlayerRepository.exists_by_username(username: str) -> bool`. `save` inserts a new row into `players`; `exists_by_username` queries for an existing row. All SQL is confined to this class (CV-04).

#### SCENARIO 1: Save a new player

**Scenario ID**: IDENTITY-BE-001.1-S1

**GIVEN**
* an in-memory SQLite database with the `players` table created
* a `Player` entity with `player_id=UUID("...")`, `username="alice"`, `hashed_password="$2b$..."`

**WHEN**
* `PlayerRepository.save(player)` is called

**THEN**
* a row exists in `players` with the correct `player_id`, `username`, and `hashed_password`

#### SCENARIO 2: Exists check returns True for a known username

**Scenario ID**: IDENTITY-BE-001.1-S2

**GIVEN**
* the `players` table contains a row for `alice`

**WHEN**
* `PlayerRepository.exists_by_username("alice")` is called

**THEN**
* the return value is `True`

#### SCENARIO 3: Exists check returns False for an unknown username

**Scenario ID**: IDENTITY-BE-001.1-S3

**GIVEN**
* the `players` table is empty

**WHEN**
* `PlayerRepository.exists_by_username("alice")` is called

**THEN**
* the return value is `False`

---

### IDENTITY-BE-001.2: `AuthService.register` — orchestration, hashing, uniqueness

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (AuthService component); `architecture/08-crosscutting-concepts.md` — §8.1 Registration and password storage

Implement `AuthService.register(username: str, password: str) -> None`. It calls `PlayerRepository.exists_by_username`; raises `UsernameTakenError` if taken; otherwise hashes the password with bcrypt and calls `PlayerRepository.save`. The domain layer never constructs HTTP responses.

#### SCENARIO 1: Register a new player

**Scenario ID**: IDENTITY-BE-001.2-S1

**GIVEN**
* `PlayerRepository.exists_by_username("alice")` returns `False`

**WHEN**
* `AuthService.register("alice", "Secret123")` is called

**THEN**
* `PlayerRepository.save` is called once with a `Player` whose `hashed_password` satisfies `bcrypt.checkpw("Secret123", hashed_password)`
* no exception is raised

#### SCENARIO 2: Duplicate username raises `UsernameTakenError`

**Scenario ID**: IDENTITY-BE-001.2-S2

**GIVEN**
* `PlayerRepository.exists_by_username("alice")` returns `True`

**WHEN**
* `AuthService.register("alice", "Secret123")` is called

**THEN**
* `UsernameTakenError` is raised
* `PlayerRepository.save` is **not** called

---

### IDENTITY-BE-001.3: `Auth Blueprint` — `POST /auth/register` endpoint

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (Auth Blueprint component); `architecture/08-crosscutting-concepts.md` — §8.2 Error Handling; §8.4 Input Validation

Implement the Flask route `POST /auth/register`. It validates the request body (username: 3–32 chars, password: non-empty), calls `AuthService.register`, and maps `UsernameTakenError` → `409` and validation errors → `400`. All responses use the standard JSON envelope.

#### SCENARIO 1: Valid request returns 201

**Scenario ID**: IDENTITY-BE-001.3-S1

**GIVEN**
* the Flask test client is initialised
* `AuthService.register` is configured to succeed (no duplicate)

**WHEN**
* `POST /auth/register` is called with `{ "username": "alice", "password": "Secret123" }` <!-- pragma: allowlist secret -->

**THEN**
* response status is `201 Created`
* response body is `{}`

#### SCENARIO 2: Duplicate username returns 409

**Scenario ID**: IDENTITY-BE-001.3-S2

**GIVEN**
* `AuthService.register` raises `UsernameTakenError`

**WHEN**
* `POST /auth/register` is called with the duplicate username

**THEN**
* response status is `409 Conflict`
* response body matches `{ "error": { "code": "USERNAME_TAKEN", ... } }`

#### SCENARIO 3: Missing password field returns 400

**Scenario ID**: IDENTITY-BE-001.3-S3

**GIVEN**
* the Flask test client is initialised

**WHEN**
* `POST /auth/register` is called with `{ "username": "alice" }` (no `password`)

**THEN**
* response status is `400 Bad Request`
* response body names the `password` field as missing

---

## FE Stories

### IDENTITY-FE-001.1: Registration form

**Parent**: IDENTITY-STORY-001
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.2 Error Handling (HTTP status code mapping); `architecture/01-introduction-and-goals.md` — §1.3 Quality Goals (Usability, priority 5)

Implement a React registration form with `username` and `password` fields and a submit button. On success (`201`) it navigates the user to the login page. On `409` it displays "Username already taken." On `400` it displays the field-level error returned by the API. The raw HTTP status code is never shown to the player.

#### SCENARIO 1: Successful registration navigates to login

**Scenario ID**: IDENTITY-FE-001.1-S1

**GIVEN**
* the user is on the `/register` page
* the API will return `201`

**WHEN**
* the user enters username `alice`, password `Secret123`, and clicks Submit

**THEN**
* the form makes `POST /api/auth/register` with the correct body
* the browser navigates to `/login`

#### SCENARIO 2: Duplicate username shows inline error

**Scenario ID**: IDENTITY-FE-001.1-S2

**GIVEN**
* the user is on the `/register` page
* the API will return `409`

**WHEN**
* the user submits the form

**THEN**
* the message "Username already taken." appears on the page
* the user remains on `/register`
* the HTTP status `409` is not visible in the UI

#### SCENARIO 3: Empty username prevents submission

**Scenario ID**: IDENTITY-FE-001.1-S3

**GIVEN**
* the user is on the `/register` page

**WHEN**
* the user leaves the username field blank and clicks Submit

**THEN**
* no API request is made
* a client-side validation message appears near the username field

---

# IDENTITY-STORY-002: Player Login & JWT Issuance

**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (White-Box); `architecture/06-runtime-view.md` — §6.2 Player Login; `architecture/09-architecture-decisions.md` — ADR-003
**Pareto**: Core

```
AS A registered player
I WANT to log in with my username and password
SO THAT I receive a JWT that lets me create and participate in matches
```

## Acceptance Criteria (end-to-end)

### SCENARIO 1: Successful login returns a signed JWT

**Scenario ID**: IDENTITY-STORY-002-S1

**GIVEN**
* a player account for `alice` exists in the `players` table

**WHEN**
* `POST /auth/login` is submitted with `{ "username": "alice", "password": "Secret123" }` <!-- pragma: allowlist secret -->

**THEN**
* the response status is `200 OK`
* the response body is `{ "token": "<jwt>" }` where `<jwt>` is a valid HS256-signed JWT
* the JWT payload contains claim `sub` equal to alice's `player_id` (UUID)
* the JWT payload contains claim `exp` set to approximately 24 hours from the issue time
* no plain-text password appears anywhere in the response

---

### SCENARIO 2: Wrong password rejected without username enumeration

**Scenario ID**: IDENTITY-STORY-002-S2

**GIVEN**
* a player account for `alice` exists

**WHEN**
* `POST /auth/login` is submitted with `{ "username": "alice", "password": "WrongPass" }` <!-- pragma: allowlist secret -->

**THEN**
* the response status is `401 Unauthorized`
* the response body matches `{ "error": { "code": "INVALID_CREDENTIALS", ... } }`
* no token is returned

---

### SCENARIO 3: Unknown username rejected with the same response as a wrong password

**Scenario ID**: IDENTITY-STORY-002-S3

**GIVEN**
* no account for `unknown_user` exists

**WHEN**
* `POST /auth/login` is submitted with `{ "username": "unknown_user", "password": "anything" }` <!-- pragma: allowlist secret -->

**THEN**
* the response status is `401 Unauthorized`
* the response body is identical in structure to SCENARIO 2 (no enumeration of valid usernames)

---

### SCENARIO 4: Missing field rejected

**Scenario ID**: IDENTITY-STORY-002-S4

**GIVEN**
* the system is running

**WHEN**
* `POST /auth/login` is submitted with `{ "username": "alice" }` (no `password` field)

**THEN**
* the response status is `400 Bad Request`
* the response body names the missing `password` field

---

## INFRA Stories

### IDENTITY-INFRA-002.1: Deployment — shared with IDENTITY-STORY-001

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/07-deployment-view.md` — §7.2 Containers; §7.3 Docker Compose Structure

The `api` and `spa` containers required for login are the same containers deployed in IDENTITY-INFRA-001.1. No additional deployment work is needed; `POST /api/auth/login` is served through the same Nginx proxy on port 80.

#### SCENARIO 1: Login endpoint is reachable through the proxy

**Scenario ID**: IDENTITY-INFRA-002.1-S1

**GIVEN**
* both containers are running (per IDENTITY-INFRA-001.1)

**WHEN**
* `POST http://localhost:80/api/auth/login` is submitted with a valid JSON body

**THEN**
* a response is returned (any HTTP status from the application, confirming the proxy path is live)

---

### IDENTITY-INFRA-002.2: Data store — `players` table already created by IDENTITY-INFRA-001.2

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context; `architecture/02-constraints.md` — CV-05

Login reads from the `players` table created in IDENTITY-INFRA-001.2. No additional schema migration is required for this story.

#### SCENARIO 1: Login query resolves against the existing players table

**Scenario ID**: IDENTITY-INFRA-002.2-S1

**GIVEN**
* the `players` table exists and contains a row for `alice` (created via registration)

**WHEN**
* `PlayerRepository.find_by_username("alice")` is called during login

**THEN**
* the matching `Player` entity is returned without a schema error

---

### IDENTITY-INFRA-002.3: Domain events — not applicable for login

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.5 Domain Events; `architecture/09-architecture-decisions.md` — ADR-004

The login flow does not raise any domain events. Domain events are defined for the Game bounded context only. No event infrastructure is required for this story.

---

### IDENTITY-INFRA-002.4: Structured logging for login requests

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.3 Logging

Login requests must emit `INFO` log entries for successful logins and `ERROR` entries with stack traces for unexpected failures. JWT tokens and passwords must never appear in any log line.

#### SCENARIO 1: Successful login emits an INFO log

**Scenario ID**: IDENTITY-INFRA-002.4-S1

**GIVEN**
* the `api` container is running with `FLASK_ENV=production`

**WHEN**
* a successful `POST /auth/login` request is processed

**THEN**
* stdout contains a JSON log line at level `INFO` that includes an event name (e.g. `player_logged_in`) and the `player_id`
* the log line does not contain the plain-text password, the bcrypt hash, or the issued JWT token

#### SCENARIO 2: Tokens and passwords are never logged

**Scenario ID**: IDENTITY-INFRA-002.4-S2

**GIVEN**
* the `api` container is running

**WHEN**
* any login request (successful or failed) is processed

**THEN**
* no log line at any level contains the plain-text password, the bcrypt hash, or the JWT token string

---

## BE Stories

### IDENTITY-BE-002.1: `PlayerRepository.find_by_username` — load a player by username

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (PlayerRepository component)

Implement `PlayerRepository.find_by_username(username: str) -> Player | None`. It queries the `players` table by username and returns the `Player` entity if found, or `None` if not.

#### SCENARIO 1: Returns the Player entity for a known username

**Scenario ID**: IDENTITY-BE-002.1-S1

**GIVEN**
* the `players` table contains a row with `username="alice"`, `player_id=<uuid>`, `hashed_password="$2b$..."`

**WHEN**
* `PlayerRepository.find_by_username("alice")` is called

**THEN**
* a `Player` dataclass is returned with the matching `player_id`, `username`, and `hashed_password`

#### SCENARIO 2: Returns None for an unknown username

**Scenario ID**: IDENTITY-BE-002.1-S2

**GIVEN**
* the `players` table contains no row for `"unknown_user"`

**WHEN**
* `PlayerRepository.find_by_username("unknown_user")` is called

**THEN**
* `None` is returned

---

### IDENTITY-BE-002.2: `TokenService.issue_token` — issue a signed HS256 JWT

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (TokenService component); `architecture/08-crosscutting-concepts.md` — §8.1 Session tokens; `architecture/09-architecture-decisions.md` — ADR-003

Implement `TokenService.issue_token(player_id: UUID) -> str`. It creates a JWT with claims `sub` (player_id as string), `iat` (now), and `exp` (now + 24 hours), signed with HS256 using `JWT_SECRET`, and returns the encoded token string.

#### SCENARIO 1: Issued token carries the correct claims

**Scenario ID**: IDENTITY-BE-002.2-S1

**GIVEN**
* `JWT_SECRET` is set to a non-empty string
* `player_id` is a known UUID

**WHEN**
* `TokenService.issue_token(player_id)` is called

**THEN**
* the returned string is a valid HS256 JWT decodable with the same `JWT_SECRET`
* the decoded payload contains `sub` equal to `str(player_id)`
* the decoded payload contains `exp` approximately 24 hours after `iat`

#### SCENARIO 2: Token issued with a different secret is rejected on decode

**Scenario ID**: IDENTITY-BE-002.2-S2

**GIVEN**
* a token was issued with `JWT_SECRET="correct-secret"` <!-- pragma: allowlist secret -->

**WHEN**
* the token is decoded with `JWT_SECRET="wrong-secret"` <!-- pragma: allowlist secret -->

**THEN**
* `PyJWT` raises a signature verification error

---

### IDENTITY-BE-002.3: `AuthService.login` — credential verification and token issuance

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (AuthService component); `architecture/06-runtime-view.md` — §6.2 Player Login

Implement `AuthService.login(username: str, password: str) -> str`. It loads the `Player` via `PlayerRepository.find_by_username`; raises `InvalidCredentialsError` if the player is not found or if `bcrypt.checkpw` fails; on success calls `TokenService.issue_token` and returns the token string.

#### SCENARIO 1: Correct credentials return a token string

**Scenario ID**: IDENTITY-BE-002.3-S1

**GIVEN**
* `PlayerRepository.find_by_username("alice")` returns a `Player` whose `hashed_password` matches `"Secret123"`

**WHEN**
* `AuthService.login("alice", "Secret123")` is called

**THEN**
* a non-empty JWT string is returned
* `TokenService.issue_token` was called with alice's `player_id`

#### SCENARIO 2: Wrong password raises InvalidCredentialsError

**Scenario ID**: IDENTITY-BE-002.3-S2

**GIVEN**
* `PlayerRepository.find_by_username("alice")` returns a valid `Player`
* `bcrypt.checkpw("WrongPass", hashed_password)` returns `False`

**WHEN**
* `AuthService.login("alice", "WrongPass")` is called

**THEN**
* `InvalidCredentialsError` is raised
* `TokenService.issue_token` is **not** called

#### SCENARIO 3: Unknown username raises InvalidCredentialsError

**Scenario ID**: IDENTITY-BE-002.3-S3

**GIVEN**
* `PlayerRepository.find_by_username("unknown_user")` returns `None`

**WHEN**
* `AuthService.login("unknown_user", "anything")` is called

**THEN**
* `InvalidCredentialsError` is raised
* the error message does not distinguish "user not found" from "wrong password"

---

### IDENTITY-BE-002.4: `Auth Blueprint` — `POST /auth/login` endpoint

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/05-building-block-view.md` — §5.2 Level 2 — Identity Bounded Context (Auth Blueprint component); `architecture/08-crosscutting-concepts.md` — §8.2 Error Handling; §8.4 Input Validation

Implement the Flask route `POST /auth/login`. It validates the request body (username and password present and non-empty), calls `AuthService.login`, and maps `InvalidCredentialsError` → `401` and validation errors → `400`. All responses use the standard JSON envelope.

#### SCENARIO 1: Valid credentials return 200 with token

**Scenario ID**: IDENTITY-BE-002.4-S1

**GIVEN**
* the Flask test client is initialised
* `AuthService.login` is configured to return a token string

**WHEN**
* `POST /auth/login` is called with `{ "username": "alice", "password": "Secret123" }` <!-- pragma: allowlist secret -->

**THEN**
* response status is `200 OK`
* response body is `{ "token": "<jwt_string>" }`

#### SCENARIO 2: Invalid credentials return 401

**Scenario ID**: IDENTITY-BE-002.4-S2

**GIVEN**
* `AuthService.login` raises `InvalidCredentialsError`

**WHEN**
* `POST /auth/login` is called with wrong credentials

**THEN**
* response status is `401 Unauthorized`
* response body matches `{ "error": { "code": "INVALID_CREDENTIALS", ... } }`

#### SCENARIO 3: Missing password field returns 400

**Scenario ID**: IDENTITY-BE-002.4-S3

**GIVEN**
* the Flask test client is initialised

**WHEN**
* `POST /auth/login` is called with `{ "username": "alice" }` (no `password`)

**THEN**
* response status is `400 Bad Request`
* response body names the `password` field as missing

---

## FE Stories

### IDENTITY-FE-002.1: Login form

**Parent**: IDENTITY-STORY-002
**Architecture References**: `architecture/05-building-block-view.md` — §5.1 Level 1 — System (White-Box) (React SPA container); `architecture/08-crosscutting-concepts.md` — §8.1 Session tokens; `architecture/01-introduction-and-goals.md` — §1.3 Quality Goals (Usability, priority 5)

Implement a React login form with `username` and `password` fields and a submit button. On success (`200`) the SPA stores the returned JWT in memory and navigates to the page where the player can create or join a match (GAME-STORY-001 / GAME-STORY-002). On `401` it displays "Invalid username or password." without indicating which field was wrong. The raw HTTP status is never shown to the player.

#### SCENARIO 1: Successful login stores the token and navigates to the create/join page

**Scenario ID**: IDENTITY-FE-002.1-S1

**GIVEN**
* the user is on the `/login` page
* the API will return `200 OK` with `{ "token": "<jwt>" }`

**WHEN**
* the user enters username `alice`, password `Secret123`, and clicks Login

**THEN**
* the SPA makes `POST /api/auth/login` with `{ "username": "alice", "password": "Secret123" }` <!-- pragma: allowlist secret -->
* the returned token is stored in memory (not in localStorage or a cookie)
* the browser navigates to the page where the player can create or join a match (GAME-STORY-001 / GAME-STORY-002)

#### SCENARIO 2: Invalid credentials show an inline error

**Scenario ID**: IDENTITY-FE-002.1-S2

**GIVEN**
* the user is on the `/login` page
* the API will return `401 Unauthorized`

**WHEN**
* the user submits the form

**THEN**
* the message "Invalid username or password." appears on the page
* the user remains on `/login`
* the response does not indicate whether the username or the password was wrong

#### SCENARIO 3: Empty password field prevents submission

**Scenario ID**: IDENTITY-FE-002.1-S3

**GIVEN**
* the user is on the `/login` page

**WHEN**
* the user leaves the password field blank and clicks Login

**THEN**
* no API request is made
* a client-side validation message appears near the password field
