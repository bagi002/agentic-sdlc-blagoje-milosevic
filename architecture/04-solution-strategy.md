# Chapter 4: Solution Strategy

## 4.1 Overview

The table below maps each top-priority quality goal to the architectural decision that addresses it.

| Quality Goal | Architectural Approach |
|---|---|
| **Correctness** | Pure-function domain layer; all game rules implemented as side-effect-free Python; every rule covered by pytest-bdd scenarios |
| **Testability** | Strict layered architecture — domain layer has zero imports from Flask or SQLite; exercised entirely in memory |
| **Maintainability** | Domain-Driven Design: one bounded context per concern; ubiquitous language in code matches the kata terminology |
| **Security** | Internal auth service issues session tokens (bcrypt + signed token); all match endpoints verify token and player identity before acting |
| **Usability** | React SPA handles all turn sequencing; players never interact with raw API responses |

## 4.2 Architectural Style: Layered + Domain-Driven

The system uses a **four-layer architecture** within a single deployable Flask application:

```
┌──────────────────────────────────┐
│         Presentation Layer       │  React 18 SPA (browser)
├──────────────────────────────────┤
│           API Layer              │  Flask routes + request/response serialisation
├──────────────────────────────────┤
│          Domain Layer            │  Pure Python — game rules, entities, domain events
├──────────────────────────────────┤
│       Infrastructure Layer       │  SQLite repositories
└──────────────────────────────────┘
```

The domain layer is the centrepiece. It knows nothing about HTTP, databases, or tokens. Flask and SQLite are infrastructure details that plug in around it.

## 4.3 Domain-Driven Design: Bounded Contexts

The domain is split into two bounded contexts. Each has its own model, its own database tables, and its own Flask blueprint.

| Bounded Context | Responsibility | Key Entities |
|---|---|---|
| **Identity** | Player registration, login, session token lifecycle | `Player`, `Session` |
| **Game** | Match lifecycle, turn engine, rule enforcement, computer AI | `Match`, `PlayerState`, `Card`, `Deck`, `Hand` |

The two contexts communicate in one direction only: the Game context reads the authenticated player identity (player ID) from the session token validated by the Identity context. No direct object references cross the boundary.

## 4.4 Technology Choices

| Concern | Choice | Reason |
|---|---|---|
| Backend language | Python 3.12 | TC-01 |
| REST framework | Flask | TC-02; minimal overhead for a small API surface |
| Frontend | React 18 SPA | TC-03 |
| Database | SQLite | TC-04; single file, zero server, sufficient for the low number of concurrent matches expected at kata scale |
| Password hashing | bcrypt | Industry-standard adaptive hash; resists brute-force as hardware improves |
| Session tokens | Signed JWT (HS256, `PyJWT`) | Stateless verification on every request; no server-side session store needed |
| Testing | pytest + pytest-bdd | TC-05; Gherkin scenarios double as living specification |
| Containerisation | Docker + Docker Compose | TC-06; one command starts the full stack locally |
| Computer opponent | Greedy AI (play the highest-cost affordable card each turn) | Simple, deterministic, correct; no external dependency |

## 4.5 Key Architectural Decisions (summary)

Full ADRs are in Chapter 9. The decisions with the widest impact are:

| Decision | Choice |
|---|---|
| Authentication mechanism | Internal bcrypt + JWT; no external IdP |
| Domain isolation | Domain layer is a pure Python package with no I/O imports |
| Persistence strategy | Repository pattern wrapping SQLite; domain objects never touch SQL directly |
| Computer opponent | Stateless greedy function; plug-in via the same `PlayerState` interface as a human |
| API design | REST; one blueprint per bounded context (`/auth`, `/matches`) |
