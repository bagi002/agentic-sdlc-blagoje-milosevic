# Chapter 12: Glossary

Terms are listed alphabetically. Domain terms reflect the kata's ubiquitous language; architectural terms reflect the design decisions in this document.

---

| Term | Definition |
|---|---|
| **Active Player** | The player whose turn it is. At any point in a match exactly one player is active. |
| **Aggregate** | A DDD pattern: a cluster of domain objects treated as a single unit with one root entity. `Match` is the aggregate root of the Game bounded context. |
| **Bcrypt** | A password-hashing algorithm used to store player passwords. Adaptive: its work factor can be increased without invalidating stored hashes. |
| **Bleeding Out** | Special rule: if the active player's deck is empty when they would draw a card at the start of their turn, they take 1 damage instead of drawing. |
| **Bounded Context** | A DDD boundary within which a particular domain model and language apply. This system has two: Identity and Game. |
| **Card** | The fundamental game object. Each card has a mana cost. Playing a card deals damage equal to its mana cost (see also: Dud). |
| **ComputerAI** | The module that drives the computer opponent's card-play decisions. It uses a greedy strategy: on each step it picks the single highest-cost card affordable given the current mana, plays it, and repeats. |
| **Deck** | The 20-card draw pile each player starts with. Cards are drawn from the deck at the start of each turn. The fixed deck composition (mana costs: 0, 0, 1, 1, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 5, 5, 6, 6, 7, 8) is defined by the kata. |
| **Domain Event** | Something that happened in the domain, named in the past tense (e.g. `CardPlayed`, `TurnEnded`). Events are raised by `TurnEngine`, accumulated on the `Match` aggregate, and included in the API response. They are in-process and not persisted. |
| **Dud** | Special rule: a card with a mana cost of 0 deals no damage when played. The mana cost (0) is still deducted. |
| **Flask Blueprint** | A Flask mechanism for grouping related routes. The system uses two: `Auth Blueprint` (Identity context) and `Match Blueprint` (Game context). |
| **GameWon** | Domain event raised when a player's health reaches 0 or below. Can be raised by `TurnEngine.play_card` (after damage) or `TurnEngine.begin_turn` (after Bleeding Out). Causes `Match.status` to be set to `FINISHED`. |
| **Greedy Strategy** | The ComputerAI's decision rule: always play the highest-cost card currently affordable. See ComputerAI. |
| **Hand** | The cards a player holds and can play on their turn. Maximum 5 cards. Cards that would exceed this limit are discarded (see: Overload). |
| **Health** | A player's hit points. Each player starts with 30 HP. Health decreases when the opponent plays a card or when Bleeding Out fires. A player whose health reaches 0 or below loses. |
| **Identity (Bounded Context)** | The bounded context responsible for player registration, login, and JWT token lifecycle. Its key entity is `Player`. |
| **JWT (JSON Web Token)** | A signed, stateless token issued by `TokenService` on successful login. Encodes `player_id` and an expiry claim. Presented as a `Bearer` token on every protected request. |
| **Game (Bounded Context)** | The bounded context responsible for match lifecycle, turn engine, rule enforcement, and computer AI. Its aggregate root is `Match`. |
| **Mana** | The resource spent to play cards. At the start of each turn, a player's mana is refilled to their current number of mana slots. Playing a card costs mana equal to the card's mana cost. |
| **Mana Slot** | A permanent mana capacity unit. Each player starts with 0 mana slots. One slot is gained at the start of each turn, up to a maximum of 10. Mana is refilled to the slot count each turn. |
| **Match** | A single game between two players (PvP or PvC). The `Match` aggregate root tracks all game state: both players' health, mana, deck, hand, the current turn number, and the match phase and status. |
| **Match Blueprint** | The Flask Blueprint that exposes the Game context's REST endpoints: `GET /matches/{id}`, `POST /matches`, `POST /matches/{id}/join`, `POST /matches/{id}/cards`, `POST /matches/{id}/end-turn`. |
| **Opening Hand** | The 3 cards dealt to each player before the first turn begins. |
| **Overload** | Special rule: if a player's hand already holds 5 cards when a card would be drawn, the new card is discarded. The hand stays at 5. |
| **Phase** | The current state within an active turn. `AWAITING_START`: the active player's `begin_turn` has not yet run (used at match creation and briefly between turns). `PLAY`: `begin_turn` has completed and the active player may play cards. When `Match.status` is `FINISHED`, phase is irrelevant. |
| **Player** | A registered human user. Identified by a `player_id` (UUID). Participates in matches as either the active player or the opponent. |
| **PlayerState** | A value object within the `Match` aggregate that holds the state of one participant: health, mana slots, current mana, deck, and hand. |
| **Polling** | The mechanism by which the opponent's SPA discovers a turn change: it periodically calls `GET /matches/{id}` and checks whether the active player has changed. The response omits the opponent's hand. No push mechanism is implemented (see TD-01). |
| **PvC (Player vs Computer)** | A game mode in which one human player faces the computer opponent. The computer's turn runs synchronously inside the human's end-turn request. |
| **PvP (Player vs Player)** | A game mode in which two human players take turns. Each player uses their own browser session. |
| **Repository** | An infrastructure-layer class that abstracts database access. `PlayerRepository` and `MatchRepository` are the two repositories; they are the only classes that execute SQL. |
| **TurnEngine** | The domain module that implements all game rules as pure functions: `begin_turn`, `play_card`, and `end_turn`. It has no I/O imports and can be tested entirely in memory. |
| **Turn** | One player's complete action cycle: start of turn (mana slot gain, mana refill, card draw) followed by zero or more card plays, followed by end of turn. |
| **Win Condition** | A player wins when their opponent's health reaches 0 or below. Checked after every `play_card` call and after every `begin_turn` call (to catch Bleeding Out deaths at the start of a turn). |
