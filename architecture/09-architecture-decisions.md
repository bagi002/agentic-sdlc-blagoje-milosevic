# 9. Architecture Decisions

## ADR-001: Technology Stack for the Trading Card Game

**Status:** Accepted

**Context:**
The Trading Card Game needs a backend that enforces the game rules (health,
mana, deck, hand, turns, damage, win conditions), a web frontend where players
play matches (player vs player or player vs computer), a way to store matches
and players, and an automated test setup that supports TDD and BDD. The
application must be packaged so it can be built, tested and deployed the same
way on every machine and in CI.

**Decision:**
- Backend: Python 3.12 with Flask, exposing a REST API over HTTPS
- Frontend: React 18 single-page application
- Data storage: SQLite
- Testing: pytest for unit tests, pytest-bdd for BDD acceptance scenarios
- Packaging: Docker

**Rationale:**
- Python 3.12, React 18, pytest and pytest-bdd are the course tech stack
- Flask is lightweight and simple, which fits a small REST API for one game
- SQLite needs no separate database server and stores everything in one file
- pytest-bdd turns the game rules into executable Given/When/Then scenarios
- Docker gives the same environment locally, in CI and in deployment

**Consequences:**
- SQLite allows only one writer at a time, which limits many simultaneous matches
- The SQLite file must be kept on persistent storage (a Docker volume) in deployment
- Flask provides less out of the box than larger frameworks, so validation
  and API structure must be added explicitly
- Two languages (Python and JavaScript) must be maintained, each with its own tooling
