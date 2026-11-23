# Chapter 9: Architecture Decisions

---

## ADR-001: Technology Stack (Flask, React 18, SQLite, pytest/pytest-bdd, Docker)

**Status:** Accepted

**Context:**
The project is a kata-scale web application with a small, well-defined domain. The course mandates Python 3.12 (TC-01), React 18 (TC-03), pytest + pytest-bdd (TC-05), and Docker (TC-06). The backend REST framework, database, and testing approach beyond the mandated tools are left to the team's discretion. The stack must support rapid local setup, isolated domain testing, and a reproducible CI environment.

**Decision:**
Use Python 3.12 + Flask for the backend, React 18 for the frontend SPA, SQLite for persistence, pytest + pytest-bdd for testing, and Docker + Docker Compose for containerisation.

**Rationale:**
- Flask is minimal and low-ceremony; it is the right fit for a small REST surface with two blueprints and no need for a full framework's conventions.
- SQLite requires no separate server process, eliminating setup friction. A single `.db` file on a Docker volume is sufficient for the low number of concurrent matches expected at kata scale.
- pytest-bdd lets Gherkin scenarios serve as both specification and executable tests, satisfying OC-04.
- Docker Compose gives a single-command startup and reproducible environments across machines, satisfying TC-06.

**Consequences:**
- SQLite's single-writer constraint is acceptable at kata scale but would require replacement (e.g. PostgreSQL) if concurrent write throughput became a requirement.
- Flask's lack of built-in async support is not a concern here; all operations are synchronous and fast.

---

## ADR-002: No AWS

**Status:** Accepted

**Context:**
AWS services (Lambda, DynamoDB, Cognito, API Gateway) are unavailable for this project (TC-07). The architecture must run entirely on a local machine or a generic host.

**Decision:**
Do not use any AWS services. All infrastructure is provided by Docker containers running on a standard host.

**Rationale:**
- The constraint is hard: AWS is not available.
- Docker Compose provides equivalent local reproducibility without cloud dependency.

**Consequences:**
- The system cannot leverage managed scaling, CDN, or cloud-native auth. These are not requirements at kata scale.
- When a hosting target is decided, the Docker Compose artefact is portable to any container-capable host without AWS-specific changes.

---

## ADR-003: Internal Authentication with bcrypt and JWT

**Status:** Accepted

**Context:**
Players must be identified to enforce turn order and prevent one player from acting for another. The original design delegated this to an external Identity Provider (OAuth2/JWT). That decision was reversed: the system now owns the full player lifecycle.

**Decision:**
Implement registration and login inside the Trading Card Game itself. Store hashed passwords in SQLite using bcrypt. Issue signed JWTs (HS256, 24-hour expiry) on login via `TokenService`. Verify JWTs on every protected endpoint using the `require_auth` decorator.

**Rationale:**
- Removing the external IdP eliminates a runtime dependency and simplifies local setup.
- bcrypt is an industry-standard adaptive hash; it is the correct choice for password storage.
- Stateless JWTs avoid a server-side session store, keeping the Flask API stateless between requests.
- HS256 is sufficient for a single-service system where the signing key never leaves the server.

**Consequences:**
- The system is now responsible for credential security (hashing, token expiry, secret rotation). This is manageable at kata scale with a well-chosen `JWT_SECRET` and bcrypt work factor.
- Token revocation is not supported (no blocklist). A 24-hour expiry is the only revocation mechanism. This is acceptable for the kata; a production system would need a revocation strategy.

---

## ADR-004: Modular Monolith with DDD Bounded Contexts and In-Process Domain Events

**Status:** Accepted

**Context:**
The domain has two distinct concerns: identity (registration, login) and game (match lifecycle, rules). These could be separate services, but the kata scale does not justify the operational overhead of microservices.

**Decision:**
Deploy a single Flask application (modular monolith) with two DDD bounded contexts: **Identity** and **Game**. Each context has its own Flask blueprint, its own Python modules, and its own database tables. Domain events are raised in-process and accumulated on the `Match` aggregate; they are not published to a message broker.

**Rationale:**
- A modular monolith enforces the same bounded-context boundaries as microservices but without network hops, distributed tracing, or separate deployment pipelines.
- In-process events are sufficient here: only `MatchService` consumes them (to build the API response). No cross-context reactions are needed.
- The module boundaries mean the system can be split into separate services later if scale demands it, without redesigning the domain model.

**Consequences:**
- Both contexts share one SQLite file and one process. A failure in one context takes down the other. Acceptable at kata scale.
- In-process events cannot fan out to external consumers. If future requirements add notifications or analytics, a message broker would need to be introduced.

---

## ADR-005: Two-Phase Turn Flow

**Status:** Accepted

**Context:**
The kata's turn sequence is: gain mana slot → refill mana → draw card → play cards → end turn. A single-request design (send all cards at once) forces the player to choose cards before seeing the drawn card and the refreshed mana, which is not faithful to the kata and creates a poor user experience.

**Decision:**
Split the turn into two phases. Phase 1 (start of turn: mana slot gain, mana refill, card draw) runs automatically when the previous turn ends, via `TurnEngine.begin_turn`. Phase 2 (card play) is driven by the player: one `POST /matches/{id}/cards` per card, followed by `POST /matches/{id}/end-turn`.

**Rationale:**
- The player sees the drawn card and refreshed mana before making decisions, which matches the kata's intent.
- Persisting state after every card means a disconnection mid-turn does not lose progress.
- The computer opponent follows the same two-phase contract, keeping the domain logic uniform.

**Consequences:**
- A PvP turn now requires at least two round-trips (one for the start-of-turn state, one or more for card plays, one for end-turn) instead of one. This is correct behaviour, not a drawback.
- The `Match` aggregate needs a `phase` field (`AWAITING_START` / `PLAY`) and a `status` field (`ACTIVE` / `FINISHED`) to guard against out-of-sequence requests. `phase` governs whether card play is permitted; `status` signals that the match is over.

---

## ADR-006: Greedy Computer Opponent

**Status:** Accepted

**Context:**
Player vs Computer mode requires an automated opponent. The opponent must be deterministic (testable), correct (follows all game rules including mana deduction per card), and simple to implement and maintain.

**Decision:**
Implement the computer opponent as a stateless loop in `ComputerAI`. On each iteration it calls `next_card(player_state)`, which returns the single highest-cost card the computer can still afford given its current mana, or `None` if none is affordable. `MatchService` calls `TurnEngine.play_card` for the returned card (which deducts mana), then repeats until `next_card` returns `None` or the win condition is met.

**Rationale:**
- Picking one card at a time and re-checking mana after each `play_card` call is correct: mana decreases with each card played, so choosing all cards up front from the starting mana would allow plays that are not actually affordable.
- The loop is deterministic: given the same `PlayerState`, it always produces the same sequence, making it fully testable with pytest-bdd.
- `ComputerAI` calls `TurnEngine.play_card` through the same code path as a human player, so rule enforcement is not duplicated.
- Implementing a more sophisticated AI (minimax, MCTS) would add complexity without contributing to the kata's architectural goals.

**Consequences:**
- The greedy opponent is beatable with simple strategies (e.g. holding low-cost cards). This is acceptable; gameplay balance is not a requirement of the kata.
- If a more capable AI is needed later, `next_card` is the single function to replace; the rest of the turn machinery is unchanged.
