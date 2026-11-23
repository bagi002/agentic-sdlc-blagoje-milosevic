# Chapter 10: Quality Requirements

## 10.1 Quality Tree

The five quality goals from Chapter 1, ordered by priority, are decomposed into measurable sub-characteristics below.

```
Trading Card Game Quality Goals
│
├── 1. Correctness
│   ├── Rule fidelity          All game rules produce the right outcome every time
│   └── Edge-case coverage     Bleeding Out, Overload, Dud each handled correctly
│
├── 2. Testability
│   ├── Domain isolation       Domain layer exercisable without Flask or SQLite
│   └── BDD coverage           Every rule expressible as a Gherkin scenario
│
├── 3. Maintainability
│   ├── Bounded context clarity New rule confined to domain layer, one context
│   └── Ubiquitous language    Code names match kata terminology exactly
│
├── 4. Security
│   ├── Authentication         Only registered players can act in matches
│   └── Authorisation          A player cannot act on behalf of their opponent
│
└── 5. Usability
    ├── Turn guidance          SPA shows current hand, mana, and health at all times
    └── Error feedback         Invalid actions produce clear, actionable messages
```

---

## 10.2 Quality Scenarios

Each scenario follows the arc42 format: stimulus → environment → response → measure.

### Correctness

| ID | Stimulus | Environment | Response | Measure |
|---|---|---|---|---|
| QS-C1 | A player plays a card that costs exactly their remaining mana | Match in PLAY phase | Card is played; mana reaches 0; no error | pytest-bdd scenario passes; identical result on every run |
| QS-C2 | A player's deck is empty at the start of their turn | Match in AWAITING_START phase | Player takes 1 damage (Bleeding Out); no card drawn | pytest-bdd scenario passes; health decremented by exactly 1 |
| QS-C3 | A player holds 5 cards when a card would be drawn | Match in AWAITING_START phase | Drawn card is discarded; hand stays at exactly 5 | pytest-bdd scenario passes; hand size remains 5 |
| QS-C4 | A player plays a 0-mana card | Match in PLAY phase | Mana cost deducted (0); opponent health unchanged (Dud) | pytest-bdd scenario passes; opponent health unchanged |
| QS-C5 | A played card reduces opponent health to exactly 0 | Match in PLAY phase | Match status set to FINISHED; GameWon event raised | pytest-bdd scenario passes; no further cards can be played |
| QS-C6 | A player has 1 health and an empty deck at the start of their turn | Match in AWAITING_START phase | Bleeding Out fires: player takes 1 damage, health reaches 0; GameWon raised in `begin_turn`; match status set to FINISHED | pytest-bdd scenario passes; match finishes inside begin_turn; no card play phase is entered |

### Testability

| ID | Stimulus | Environment | Response | Measure |
|---|---|---|---|---|
| QS-T1 | Developer runs `pytest` against the domain package | No Flask server, no SQLite file on disk | All domain tests pass | 100% of TurnEngine and ComputerAI tests execute in under 2 seconds with no I/O |
| QS-T2 | Developer adds a new Gherkin scenario for a game rule | pytest-bdd configured | Scenario is auto-discovered and runs without framework changes | Zero new pytest configuration required |

### Maintainability

| ID | Stimulus | Environment | Response | Measure |
|---|---|---|---|---|
| QS-M1 | A developer unfamiliar with the codebase adds a new special rule | Existing domain layer | Rule implemented in `TurnEngine`, covered by a new BDD scenario; no Flask or SQLite files touched | Change contained to domain layer; completed within one working day |
| QS-M2 | A developer adds a third game mode | Existing bounded contexts | New mode added to the Game context without modifying the Identity context | Zero changes required in the Identity bounded context |

### Security

| ID | Stimulus | Environment | Response | Measure |
|---|---|---|---|---|
| QS-S1 | A request arrives at any match endpoint without an `Authorization` header | Running Flask API | Request rejected before any domain code executes | `401 Unauthorized` returned; no match state read or modified |
| QS-S2 | A player sends `POST /matches/{id}/cards` when it is the opponent's turn | Valid JWT, running Flask API | Request rejected by `MatchService` after authentication | `403 Forbidden` returned; match state unchanged |
| QS-S3 | A player registers with a username already in use | Running Flask API | Request rejected; no duplicate row created | `409 Conflict` returned; `players` table has exactly one row for that username |

### Usability

| ID | Stimulus | Environment | Response | Measure |
|---|---|---|---|---|
| QS-U1 | A player receives a `422` response for insufficient mana | React SPA | SPA displays which card was rejected and how much mana is missing | Error message names the card and the mana shortfall; no raw HTTP status shown to the player |
| QS-U2 | A player opens the match view | React SPA, match in PLAY phase | SPA shows the player's hand, current mana, mana slots, and opponent health without any additional action | All four values visible on initial render; no extra click required |
