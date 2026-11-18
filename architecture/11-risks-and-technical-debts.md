# Chapter 11: Risks and Technical Debts

## 11.1 Risks

Risks are items that could prevent the system from meeting its goals if left unaddressed.

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | **SQLite concurrent writes** — SQLite allows only one writer at a time. Under concurrent load (multiple matches being played simultaneously), write requests queue behind a lock. At kata scale this is unlikely to cause failures, but any spike in traffic could produce timeouts. | Low | Medium | Accept for now. If concurrent matches become a requirement, replace SQLite with PostgreSQL behind the same repository interface. No domain code changes needed. |
| R-02 | **JWT without revocation** — A stolen token remains valid until its 24-hour expiry. There is no blocklist or immediate invalidation mechanism. | Low | Medium | Accept for now. Mitigate partially by keeping the expiry short. A production system would add a token blocklist (e.g. a Redis set of revoked `jti` claims) or switch to shorter-lived tokens with refresh. |
| R-03 | **Hosting not decided** — There is no production deployment target. The system cannot be shared with players outside a local machine until this is resolved. | High (certainty) | High (blocks delivery) | Decision required before any shared deployment. The Docker Compose artefact is portable; the decision is organisational, not technical. |
| R-04 | **No TLS in development** — The Docker Compose setup runs on plain HTTP port 80. JWTs and passwords travel in clear text in development. | Medium (developer machines only) | Low (dev-only risk) | Development-only exposure is acceptable. TLS must be added (e.g. via a reverse proxy with a certificate) before any public deployment. |
| R-05 | **Single process, no fault isolation** — Both bounded contexts (Identity and Game) run in the same Flask process. An unhandled exception in one context can crash the entire application. | Low | High | The catch-all error handler (§8.2) reduces the chance of a crash propagating. A production deployment should run multiple Gunicorn worker processes so a single worker failure does not take down the service. |

---

## 11.2 Technical Debts

Technical debts are known shortcuts or missing pieces that are acceptable now but will need addressing if the system evolves.

| ID | Debt | Consequence if not addressed | Remediation path |
|---|---|---|---|
| TD-01 | **Polling instead of push** — P2 discovers it is their turn by polling `GET /matches/{id}`. There is no server-push mechanism (WebSocket, SSE). | Unnecessary API calls; turn transitions feel slow if the poll interval is long. | Replace polling with WebSocket or Server-Sent Events when real-time responsiveness becomes a priority. The API contract and domain layer do not change; only the delivery mechanism does. |
| TD-02 | **No explicit SQLite migration tool** — CV-05 requires explicit migrations, but no migration library (e.g. Alembic) has been integrated yet. | Schema changes require manual coordination; CI cannot verify schema correctness automatically. | Integrate Alembic or a lightweight alternative; add a migration step to the Docker entrypoint. |
| TD-03 | **No rate limiting** — The API has no rate limiting on authentication endpoints. A malicious client can attempt unlimited login requests. | Brute-force attacks on player credentials are unconstrained. | Add rate limiting per IP on `POST /auth/login` (e.g. via Flask-Limiter) before any public deployment. |
| TD-04 | **Computer opponent strategy is not configurable** — The greedy AI difficulty is fixed. There is no way to select a different strategy without changing code. | Players cannot choose a difficulty level; the game is always beatable by the same simple strategy. | Introduce a strategy interface; wire the chosen strategy at match-creation time and persist the choice on the `Match` aggregate. |
| TD-05 | **No match history** — Completed matches are kept in SQLite but there is no API endpoint or UI to list or review past matches. | Players have no record of previous games; debugging a reported rule error requires reading the database directly. | Add a `GET /matches` endpoint filtered by player and status; expose match history in the SPA when needed. |
