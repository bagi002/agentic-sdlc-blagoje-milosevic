# Chapter 1: Introduction and Goals

## 1.1 Purpose

This document describes the architecture of the **Trading Card Game** (TCG) system, a web-based implementation of the [TCG kata](https://codingdojo.org/kata/TradingCardGame/). Two players fight with damage cards; each card costs mana and deals damage equal to its mana cost. The player whose health reaches zero or below first loses.

## 1.2 Requirements Overview

### Core Game Rules

| Rule | Detail |
|------|--------|
| Starting health | 30 HP per player |
| Starting mana | 0 slots; +1 slot per turn (max 10), refilled fully each turn |
| Deck | 20 cards with mana costs: 0, 0, 1, 1, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 5, 5, 6, 6, 7, 8 |
| Opening hand | 3 cards drawn before the first turn |
| Turn sequence | Gain mana slot → refill mana → draw card → play any affordable cards → end turn |
| Damage | A played card deals damage equal to its mana cost |
| Win condition | Opponent health ≤ 0 |

### Special Rules

| Rule | Trigger | Effect |
|------|---------|--------|
| Bleeding Out | Active player draws from an empty deck | Takes 1 damage instead of drawing |
| Overload | Hand already holds 5 cards when a card would be drawn | Drawn card is discarded (hand stays at 5) |
| Dud | A 0-mana card is played | No damage dealt |

### Game Modes

- **Player vs Player (PvP):** two authenticated human players take turns in real time.
- **Player vs Computer (PvC):** one authenticated human player faces a computer-controlled opponent.

### Authentication

Players register and log in through the Trading Card Game itself. The system stores accounts in SQLite with hashed passwords (bcrypt). On login the system issues a session token, which the client presents on every subsequent request. There is no external Identity Provider.

## 1.3 Quality Goals

The following quality attributes drive key architecture decisions, ordered by priority:

| Priority | Quality Attribute | Scenario |
|----------|-------------------|----------|
| 1 | **Correctness** | All game rules (including edge cases: Bleeding Out, Overload, Dud) produce the same outcome regardless of how many times a scenario is replayed. |
| 2 | **Testability** | The game engine can be exercised in full isolation via pytest and pytest-bdd without a running server or database. |
| 3 | **Maintainability** | A developer unfamiliar with the codebase can add a new game rule or mode within one working day. |
| 4 | **Security** | Only authenticated players can create or participate in matches; a player cannot act on behalf of their opponent. |
| 5 | **Usability** | The React SPA guides players through a match without requiring knowledge of the underlying rules. |

## 1.4 Stakeholders

| Role | Concern |
|------|---------|
| **Player (Human)** | Fair, rule-correct matches; smooth turn-taking UI; secure authentication |
| **Developer** | Clean domain model; isolated, fast test suite; easy local setup via Docker |
| **Course Instructor** | Demonstrates arc42 + C4 + DDD + BDD applied to a bounded kata |
