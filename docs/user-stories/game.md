# GAME-STORY-004: Play a Card

**Architecture References**: `architecture/05-building-block-view.md` — §5.3 Level 2 — Game Bounded Context (White-Box), Phase 2 — Card play; `architecture/06-runtime-view.md` — §6.3 Human Player Turn (Two-Phase Flow); `architecture/09-architecture-decisions.md` — ADR-005
**Pareto**: Core

```
AS A player whose turn it is
I WANT to play a card from my hand by spending its mana cost
SO THAT I deal damage to my opponent and advance toward winning the match
```

## Acceptance Criteria (end-to-end)

### SCENARIO 1: Playing an affordable card deducts mana and deals damage

**Scenario ID**: GAME-STORY-004-S1

**GIVEN**
* a match is `ACTIVE`, in `PLAY` phase
* it is alice's turn
* alice has `mana=3`, and her hand contains a card with `mana_cost=2` at index 0

**WHEN**
* alice submits `POST /matches/{id}/cards { "card_index": 0 }`

**THEN**
* the response status is `200 OK`
* alice's `mana` is now `1` (3 − 2)
* the opponent's `health` is reduced by `2`
* the card is removed from alice's hand
* the response body includes a `CardPlayed` event and a `PlayerDamaged` event

---

### SCENARIO 2: Playing a card the player cannot afford is rejected

**Scenario ID**: GAME-STORY-004-S2

**GIVEN**
* it is alice's turn, match is in `PLAY` phase
* alice has `mana=1`, and her hand contains a card with `mana_cost=4` at index 0

**WHEN**
* alice submits `POST /matches/{id}/cards { "card_index": 0 }`

**THEN**
* the response status is `422 Unprocessable Entity`
* the response body matches `{ "error": { "code": "INSUFFICIENT_MANA", ... } }`
* alice's mana and the opponent's health are unchanged
* the card remains in alice's hand

---

### SCENARIO 3: Playing a card out of turn is forbidden

**Scenario ID**: GAME-STORY-004-S3

**GIVEN**
* a match is `ACTIVE`, in `PLAY` phase
* it is alice's turn, not bob's

**WHEN**
* bob submits `POST /matches/{id}/cards { "card_index": 0 }` with his valid JWT

**THEN**
* the response status is `403 Forbidden`
* the match state is unchanged

---

### SCENARIO 4: Multiple cards may be played in sequence within one turn

**Scenario ID**: GAME-STORY-004-S4

**GIVEN**
* it is alice's turn, match in `PLAY` phase
* alice has `mana=5`, hand contains a card with `mana_cost=3` at index 0 and a card with `mana_cost=2` at index 1

**WHEN**
* alice submits `POST /matches/{id}/cards { "card_index": 0 }`, then `POST /matches/{id}/cards { "card_index": 0 }` (the second card is now at index 0 after the first was removed)

**THEN**
* after both requests alice's `mana` is `0` (5 − 3 − 2)
* the opponent's health has decreased by `5` total
* both `CardPlayed` events appear in the respective responses

---

## INFRA Stories

### GAME-INFRA-004.1: Deploy the Game service container

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/07-deployment-view.md` — §7.2 Containers; §7.3 Docker Compose Structure

The `api` container that serves match endpoints is the same container deployed in IDENTITY-INFRA-001.1. No additional container is needed. This story verifies that `POST /api/matches/{id}/cards` is reachable through the Nginx proxy on port 80 when a valid JWT is present.

#### SCENARIO 1: Card-play endpoint is reachable through the proxy

**Scenario ID**: GAME-INFRA-004.1-S1

**GIVEN**
* both `spa` and `api` containers are running (per IDENTITY-INFRA-001.1)
* the Nginx configuration in the `spa` container proxies `/api/*` to the `api` container on port 5000

**WHEN**
* `POST http://localhost:80/api/matches/{id}/cards` is submitted with an `Authorization: Bearer <token>` header and a valid JSON body

**THEN**
* a response is returned from the application (any HTTP status, confirming the full proxy path is live)

---

### GAME-INFRA-004.2: Create the `matches` table with an explicit migration

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/05-building-block-view.md` — §5.3 Level 2 — Game Bounded Context (MatchRepository component); `architecture/02-constraints.md` — CV-05

Create the `matches` SQLite table via an explicit migration that runs at container startup. The schema must store the full serialised `Match` aggregate as a JSON blob, with `match_id` (UUID, primary key) and `state` (text, not null) as the minimum columns needed for `MatchRepository` to serialise and deserialise aggregates.

#### SCENARIO 1: Migration creates the table on a fresh database

**Scenario ID**: GAME-INFRA-004.2-S1

**GIVEN**
* the `tcg_data` volume contains a `tcg.db` file with only the `players` table (from IDENTITY-INFRA-001.2)

**WHEN**
* the migration script runs at `api` container startup

**THEN**
* `PRAGMA table_info(matches)` returns at minimum columns `match_id` and `state`
* the `players` table and its existing data are unaffected

#### SCENARIO 2: Migration is idempotent on restart

**Scenario ID**: GAME-INFRA-004.2-S2

**GIVEN**
* `tcg.db` already contains the `matches` table with one row

**WHEN**
* the `api` container restarts and the migration runs again

**THEN**
* the existing row is still present
* no error is raised

---

### GAME-INFRA-004.3: In-process domain events wired to MatchService

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.5 Domain Events; `architecture/09-architecture-decisions.md` — ADR-004

`TurnEngine.play_card` accumulates `CardPlayed` and `PlayerDamaged` events on the `Match` aggregate. `MatchService` reads and serialises these events into the API response after each `POST /matches/{id}/cards` call. No message broker is used; events are cleared from the aggregate at the start of each request and are not persisted to the database.

#### SCENARIO 1: CardPlayed and PlayerDamaged events appear in the response

**Scenario ID**: GAME-INFRA-004.3-S1

**GIVEN**
* a match is `ACTIVE` in `PLAY` phase with the active player having sufficient mana

**WHEN**
* `POST /matches/{id}/cards` is processed successfully

**THEN**
* the response body contains an `events` list that includes a `CardPlayed` entry and a `PlayerDamaged` entry
* the `matches` table row for this match does **not** store the event list (events are transient)

#### SCENARIO 2: Events are cleared between requests

**Scenario ID**: GAME-INFRA-004.3-S2

**GIVEN**
* a first `POST /matches/{id}/cards` was processed, producing `CardPlayed` and `PlayerDamaged` events

**WHEN**
* a second `POST /matches/{id}/cards` is processed for the same match

**THEN**
* the second response contains only the events raised during the second request
* events from the first request do not appear in the second response

---

### GAME-INFRA-004.4: Structured logging for card-play requests

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/08-crosscutting-concepts.md` — §8.3 Logging

Each `POST /matches/{id}/cards` request must emit an `INFO` log entry. Business-rule violations (`InsufficientManaError`) must emit a `WARNING`. Unexpected failures must emit `ERROR` with a stack trace. No game state values that constitute sensitive player data (hand contents) are written at `INFO` level in production.

#### SCENARIO 1: Successful card play emits an INFO log

**Scenario ID**: GAME-INFRA-004.4-S1

**GIVEN**
* the `api` container is running with `FLASK_ENV=production`

**WHEN**
* a successful `POST /matches/{id}/cards` request is processed

**THEN**
* stdout contains a JSON log line at level `INFO` that includes the `match_id` and an event name (e.g. `card_played`)

#### SCENARIO 2: Insufficient mana emits a WARNING log

**Scenario ID**: GAME-INFRA-004.4-S2

**GIVEN**
* the `api` container is running with `FLASK_ENV=production`

**WHEN**
* a `POST /matches/{id}/cards` request is rejected with `InsufficientManaError`

**THEN**
* stdout contains a JSON log line at level `WARNING` that includes the `match_id`
* no stack trace is written (this is a business-rule violation, not an unexpected error)

---

## BE Stories

### GAME-BE-004.1: `TurnEngine.play_card` — pure function: mana deduction, damage, win check

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/05-building-block-view.md` — §5.3 Level 2 — Game Bounded Context (TurnEngine component); `architecture/10-quality-requirements.md` — §10.2 Quality Scenarios QS-C1, QS-C4, QS-C5

Implement `TurnEngine.play_card(match: Match, card: Card) -> Match` as a pure function (no I/O). It must: (1) raise `InsufficientManaError` if the active player's mana is less than the card's cost; (2) deduct the mana cost from the active player; (3) deal damage equal to the mana cost to the opponent — 0-mana cards deal no damage so `PlayerDamaged` is only appended when health actually decreases (the Dud rule is covered fully by GAME-STORY-008); (4) always append a `CardPlayed` event; (5) append a `PlayerDamaged` event only when mana_cost > 0; (6) if the opponent's health reaches ≤ 0, set `match.status = FINISHED` and append a `GameWon` event; (7) return the updated `Match`.

#### SCENARIO 1: Card cost equals remaining mana — card played, mana reaches 0, no error

**Scenario ID**: GAME-BE-004.1-S1

**GIVEN**
* a `Match` in `PLAY` phase with active player mana=3, opponent health=30
* a `Card` with `mana_cost=3`

**WHEN**
* `TurnEngine.play_card(match, card)` is called

**THEN**
* the returned match has active player mana=0
* the returned match has opponent health=27
* `match.events` contains a `CardPlayed` event followed by a `PlayerDamaged` event
* `match.status` remains `ACTIVE`
* no exception is raised

#### SCENARIO 2: Affordable card deducts mana and deals damage

**Scenario ID**: GAME-BE-004.1-S2

**GIVEN**
* a `Match` in `PLAY` phase with active player mana=3, opponent health=30
* a `Card` with `mana_cost=2`

**WHEN**
* `TurnEngine.play_card(match, card)` is called

**THEN**
* the returned match has active player mana=1
* the returned match has opponent health=28
* `match.events` contains a `CardPlayed` event followed by a `PlayerDamaged` event
* `match.status` remains `ACTIVE`

#### SCENARIO 3: Unaffordable card raises InsufficientManaError without modifying the match

**Scenario ID**: GAME-BE-004.1-S3

**GIVEN**
* a `Match` in `PLAY` phase with active player mana=1, opponent health=30
* a `Card` with `mana_cost=4`

**WHEN**
* `TurnEngine.play_card(match, card)` is called

**THEN**
* `InsufficientManaError` is raised
* the input match is not modified: active player mana remains 1, opponent health remains 30, `match.events` is unchanged

#### SCENARIO 4: Card that reduces opponent health to exactly 0 sets match FINISHED

**Scenario ID**: GAME-BE-004.1-S4

**GIVEN**
* active player mana=5, opponent health=3
* a `Card` with `mana_cost=3`

**WHEN**
* `TurnEngine.play_card(match, card)` is called

**THEN**
* the returned match has opponent health=0
* `match.status` is `FINISHED`
* `match.events` contains `CardPlayed`, `PlayerDamaged`, and `GameWon` events (in that order)

#### SCENARIO 5: Card that reduces opponent health below 0 also sets match FINISHED

**Scenario ID**: GAME-BE-004.1-S5

**GIVEN**
* active player mana=5, opponent health=2
* a `Card` with `mana_cost=5`

**WHEN**
* `TurnEngine.play_card(match, card)` is called

**THEN**
* the returned match has opponent health=-3
* `match.status` is `FINISHED`
* `match.events` contains `CardPlayed`, `PlayerDamaged`, and `GameWon` events

---

### GAME-BE-004.2: `MatchRepository` — load and save a Match aggregate

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/05-building-block-view.md` — §5.3 Level 2 — Game Bounded Context (MatchRepository component)

Implement `MatchRepository.save(match: Match) -> None` and `MatchRepository.load(match_id: UUID) -> Match`. `save` serialises the `Match` aggregate to JSON and upserts it into the `matches` table. `load` reads the row and deserialises it back to a `Match` dataclass. All SQL is confined to this class (CV-04).

#### SCENARIO 1: Save then load round-trips the full aggregate

**Scenario ID**: GAME-BE-004.2-S1

**GIVEN**
* an in-memory SQLite database with the `matches` table created
* a `Match` aggregate with two `PlayerState` objects, `status=ACTIVE`, `phase=PLAY`

**WHEN**
* `MatchRepository.save(match)` is called, then `MatchRepository.load(match.match_id)` is called

**THEN**
* the loaded `Match` has the same `match_id`, `status`, `phase`, and both `PlayerState` values as the original

#### SCENARIO 2: Save is idempotent — repeated saves update the row

**Scenario ID**: GAME-BE-004.2-S2

**GIVEN**
* a match has been saved once with `opponent health=30`

**WHEN**
* the match is mutated (opponent health set to 28) and saved again

**THEN**
* `MatchRepository.load(match_id)` returns the match with opponent health=28
* the `matches` table still contains exactly one row for this `match_id`

#### SCENARIO 3: Loading an unknown match_id raises MatchNotFoundError

**Scenario ID**: GAME-BE-004.2-S3

**GIVEN**
* the `matches` table contains no row for a given `match_id`

**WHEN**
* `MatchRepository.load(match_id)` is called

**THEN**
* `MatchNotFoundError` is raised

---

### GAME-BE-004.3: `MatchService.play_card` — orchestration: auth, phase guard, engine call, persist

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/05-building-block-view.md` — §5.3 Level 2 — Game Bounded Context (MatchService component); `architecture/08-crosscutting-concepts.md` — §8.1 Authorisation within a match; `architecture/06-runtime-view.md` — §6.3 Human Player Turn (Two-Phase Flow)

Implement `MatchService.play_card(match_id: UUID, player_id: UUID, card_index: int) -> Match`. It must: (1) load the match via `MatchRepository`; (2) raise `MatchNotFoundError` if absent; (3) raise `NotParticipantError` if `player_id` is not one of the two match participants; (4) raise `MatchFinishedError` if `match.status` is `FINISHED` — a finished match retains `phase=PLAY` from the last card that ended it, so a phase check alone is insufficient (see §6.3 error paths: "Match already finished → 409 Conflict"); (5) raise `NotYourTurnError` if it is not `player_id`'s turn; (6) raise `WrongPhaseError` if `match.phase` is not `PLAY`; (7) look up the card at `card_index` in the active player's hand, raising `InvalidCardIndexError` if out of range; (8) call `TurnEngine.play_card(match, card)`; (9) save the updated match via `MatchRepository`; (10) return the updated `Match`.

#### SCENARIO 1: Valid card play returns updated match

**Scenario ID**: GAME-BE-004.3-S1

**GIVEN**
* a match exists in `PLAY` phase for alice and bob
* it is alice's turn, alice has mana=3 and a card with mana_cost=2 at index 0

**WHEN**
* `MatchService.play_card(match_id, alice_id, card_index=0)` is called

**THEN**
* the returned match has alice's mana=1 and the opponent's health reduced by 2
* `MatchRepository.save` was called with the updated match

#### SCENARIO 2: Non-participant is rejected

**Scenario ID**: GAME-BE-004.3-S2

**GIVEN**
* a match exists for alice and bob

**WHEN**
* `MatchService.play_card(match_id, charlie_id, card_index=0)` is called

**THEN**
* `NotParticipantError` is raised
* `MatchRepository.save` is **not** called

#### SCENARIO 3: Out-of-turn play is rejected

**Scenario ID**: GAME-BE-004.3-S3

**GIVEN**
* a match is in `PLAY` phase and it is alice's turn

**WHEN**
* `MatchService.play_card(match_id, bob_id, card_index=0)` is called

**THEN**
* `NotYourTurnError` is raised
* `MatchRepository.save` is **not** called

#### SCENARIO 4: Card play on a FINISHED match is rejected

**Scenario ID**: GAME-BE-004.3-S4

**GIVEN**
* a match has `status=FINISHED` (and `phase=PLAY`, as left by the card that ended it)

**WHEN**
* `MatchService.play_card(match_id, alice_id, card_index=0)` is called

**THEN**
* `MatchFinishedError` is raised
* `MatchRepository.save` is **not** called

---

### GAME-BE-004.4: `Match Blueprint` — `POST /matches/{id}/cards` endpoint

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/05-building-block-view.md` — §5.3 Level 2 — Game Bounded Context (Match Blueprint component); `architecture/08-crosscutting-concepts.md` — §8.1 Request authentication; §8.2 Error Handling; §8.4 Input Validation

Implement the Flask route `POST /matches/{id}/cards`, protected by the `require_auth` decorator. It validates that `card_index` is present and is a non-negative integer, calls `MatchService.play_card`, and maps domain exceptions to HTTP status codes: `MatchNotFoundError` → 404, `NotParticipantError` → 403, `NotYourTurnError` → 403, `WrongPhaseError` → 409, `MatchFinishedError` → 409, `InsufficientManaError` → 422, `InvalidCardIndexError` → 400. The response body on success is the full serialised match state plus the event list.

#### SCENARIO 1: Valid card play returns 200 with updated state and events

**Scenario ID**: GAME-BE-004.4-S1

**GIVEN**
* the Flask test client is initialised with a valid JWT for alice
* `MatchService.play_card` returns an updated `Match` with `CardPlayed` and `PlayerDamaged` events

**WHEN**
* `POST /matches/{id}/cards` is called with `{ "card_index": 0 }`

**THEN**
* response status is `200 OK`
* response body contains the updated match state (mana, health) and an `events` list with `CardPlayed` and `PlayerDamaged`

#### SCENARIO 2: Insufficient mana returns 422

**Scenario ID**: GAME-BE-004.4-S2

**GIVEN**
* `MatchService.play_card` raises `InsufficientManaError`

**WHEN**
* `POST /matches/{id}/cards` is called

**THEN**
* response status is `422 Unprocessable Entity`
* response body matches `{ "error": { "code": "INSUFFICIENT_MANA", ... } }`

#### SCENARIO 3: Out-of-turn play returns 403

**Scenario ID**: GAME-BE-004.4-S3

**GIVEN**
* `MatchService.play_card` raises `NotYourTurnError`

**WHEN**
* `POST /matches/{id}/cards` is called

**THEN**
* response status is `403 Forbidden`
* response body matches `{ "error": { "code": "NOT_YOUR_TURN", ... } }`

#### SCENARIO 4: Non-integer card_index returns 400

**Scenario ID**: GAME-BE-004.4-S4

**GIVEN**
* the Flask test client is initialised with a valid JWT

**WHEN**
* `POST /matches/{id}/cards` is called with `{ "card_index": "two" }`

**THEN**
* response status is `400 Bad Request`
* response body names `card_index` as the offending field
* `MatchService.play_card` is **not** called

#### SCENARIO 5: Play on a finished match returns 409

**Scenario ID**: GAME-BE-004.4-S5

**GIVEN**
* `MatchService.play_card` raises `MatchFinishedError`

**WHEN**
* `POST /matches/{id}/cards` is called

**THEN**
* response status is `409 Conflict`
* response body matches `{ "error": { "code": "MATCH_FINISHED", ... } }`

---

## FE Stories

### GAME-FE-004.1: Card play UI — hand display and card selection

**Parent**: GAME-STORY-004
**Architecture References**: `architecture/05-building-block-view.md` — §5.1 Level 1 — System (White-Box) (React SPA container); `architecture/08-crosscutting-concepts.md` — §8.1 Authorisation within a match (Hand visibility); `architecture/10-quality-requirements.md` — §10.2 Quality Scenarios QS-U1, QS-U2

Implement the React match view's card-play interaction. The player's hand is displayed as a list of cards showing each card's mana cost. Clicking a card sends `POST /api/matches/{id}/cards { "card_index": N }` with the JWT in the `Authorization` header. After a successful response the UI updates the displayed mana, opponent health, and hand without a full page reload. The opponent's hand is never shown — only hand size is displayed. On a `422` response the UI shows which card was rejected and states how much mana is missing.

#### SCENARIO 1: Clicking a card submits the correct card_index

**Scenario ID**: GAME-FE-004.1-S1

**GIVEN**
* the player is on the match view, it is their turn, match is in `PLAY` phase
* the player's hand contains three cards with mana costs [1, 3, 2]

**WHEN**
* the player clicks the card with mana cost 3 (index 1)

**THEN**
* the SPA sends `POST /api/matches/{id}/cards` with `{ "card_index": 1 }` and the `Authorization: Bearer <token>` header

#### SCENARIO 2: Successful play updates mana, health, and hand in the UI

**Scenario ID**: GAME-FE-004.1-S2

**GIVEN**
* the player has mana=3, the opponent has health=30
* the player clicks a card with mana_cost=3
* the API returns `200 OK` with updated state (mana=0, opponent health=27) and events

**WHEN**
* the response is received

**THEN**
* the displayed mana updates to 0
* the displayed opponent health updates to 27
* the played card is removed from the displayed hand
* no full page reload occurs

#### SCENARIO 3: Insufficient mana response shows an actionable error message

**Scenario ID**: GAME-FE-004.1-S3

**GIVEN**
* the player attempts to play a card costing 4 mana but has only 2 mana
* the API returns `422` with `{ "error": { "code": "INSUFFICIENT_MANA", "message": "Card costs 4 mana but player only has 2." } }`

**WHEN**
* the response is received

**THEN**
* the UI displays a message that names the card and states the mana shortfall (e.g. "Card costs 4 mana but you only have 2.")
* the raw status code `422` is not shown to the player
* the hand and mana display remain unchanged
