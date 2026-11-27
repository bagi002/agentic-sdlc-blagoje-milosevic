# Trading Card Game — User Story Inventory

## Bounded Contexts (Domains)

| Domain | Prefix | Source |
|--------|--------|--------|
| Identity | `IDENTITY` | `architecture/05-building-block-view.md` §5.2 |
| Game | `GAME` | `architecture/05-building-block-view.md` §5.3 |

---

## Pareto Analysis

> **20% of stories = 80% of system value.**
> 14 stories total → 20% = ~3 stories in the core set.
> Core is selected by applying all four criteria: (1) needed for MVP, (2) blocks other stories, (3) delivers the most user value, (4) exercises the most architecture components.

### Core Stories (~20% → ~80% value)

| ID | Title | Domain | Criteria met |
|----|-------|--------|--------------|
| IDENTITY-STORY-001 | Player Registration | Identity | MVP · blocks IDENTITY-002 + all GAME · no account = nothing works · Auth Blueprint + AuthService + PlayerRepository + bcrypt + SQLite |
| IDENTITY-STORY-002 | Player Login & JWT Issuance | Identity | MVP · blocks all GAME · no token = no match actions · TokenService + JWT |
| GAME-STORY-004 | Play a Card (mana deduction + damage) | Game | MVP · blocks GAME-005/006–009 · highest gameplay value · TurnEngine.play_card + MatchService + MatchRepository + CardPlayed + PlayerDamaged events |

### Supporting Stories (~80% → ~20% incremental value)

| ID | Title | Domain |
|----|-------|--------|
| IDENTITY-STORY-003 | Authenticated Request Rejection (missing/expired/invalid JWT) | Identity |
| GAME-STORY-001 | Create a Match | Game |
| GAME-STORY-002 | Join a Match | Game |
| GAME-STORY-003 | Start-of-Turn Phase (mana gain, refill, card draw) | Game |
| GAME-STORY-005 | End a Turn & Advance Active Player | Game |
| GAME-STORY-006 | Bleeding Out (draw from empty deck → 1 damage) | Game |
| GAME-STORY-007 | Overload (5-card hand → drawn card discarded) | Game |
| GAME-STORY-008 | Dud (0-mana card → no damage) | Game |
| GAME-STORY-009 | Win Condition (health ≤ 0 → match FINISHED) | Game |
| GAME-STORY-010 | Computer Opponent Turn (PvC greedy loop) | Game |
| GAME-STORY-011 | View Match State (hand, mana, health) | Game |

---

## Story Bundles

| ID | File | Status |
|----|------|--------|
| **IDENTITY-STORY-001** | [identity.md](identity.md) | ✅ Done |
| **IDENTITY-STORY-002** | [identity.md](identity.md) | ✅ Done |
| **GAME-STORY-004** | [game.md](game.md) | ✅ Done |
| IDENTITY-STORY-003 | [identity.md](identity.md) | ⏳ Pending |
| GAME-STORY-001 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-002 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-003 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-005 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-006 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-007 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-008 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-009 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-010 | [game.md](game.md) | ⏳ Pending |
| GAME-STORY-011 | [game.md](game.md) | ⏳ Pending |

---

## Pareto Progress

📊 **Pareto Progress**: 3/3 core stories written (100% of core stories)
🎯 **Core functionality coverage**: ~80% of 80% target  _(formula: 3/3 × 80%)_
