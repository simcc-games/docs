<!--
locale: en
rules-version: 2026.07.31
event-rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: current-implementation
-->

# Maths Warriors — Tournament Reference

**Audience:** competition judges and floor officials.
**Purpose:** lookup, not reading. Find your section, get a ruling, move on.
**Basis:** reverse-engineered from the archived web implementation, then checked against the
shared TypeScript engine used by the current client and server. Legacy `js/` and `server/`
citations remain as audit evidence. Nothing here is inferred from tradition or from the game's
own prose pages.

---

## 0. READ THIS FIRST — one shared rules engine

The current platform has **one rules engine**:
`packages/games/maths-warriors/src/engine.ts`. Local play, AI, review, puzzles and the Node game
server consume that same module. Online play remains server-authoritative: the client sends move
intent and the server validates it with the shared engine before broadcasting the resulting
state.

The archived vanilla application had separate client and server engines. Their differences were
removed during the shared-engine migration and are recorded in §12 for audit purposes. They are
not mode-specific rulings in the current platform.

---

## 1. Setup

| Item | Ruling | Source |
|---|---|---|
| Dice per player | Exactly 6: D4, D6, D8, D10, D12, D20 | `js/config.js:1`, `server/game-logic.js:6-13` |
| Initial values | Each die independently uniform over `1..sides` | `js/state.js:3-5`, `server/game-logic.js:18-20` |
| Dice order shown | By value ascending in local and online play | `packages/games/maths-warriors/src/engine.ts` |
| Both players' dice rolled at setup | Yes, independently | `js/state.js:25-26` |

### 1.1 First player determination

Compare the two dice arrays **position by position**. First position where the values differ:
the player with the **lower** value moves first. If all six positions are equal, **Player 1**
moves first.

- Shared engine: `packages/games/maths-warriors/src/engine.ts`
- Archived agreement path: `server/game-logic.js:60-68`

**There is no tiebreaker roll.** All-equal resolves deterministically to Player 1
(`js/state.js:39`, `server/game-logic.js:67`). The game's help page claims a tiebreaker roll;
the code has none.

The archived client's `alwaysFirst` practice setting is not part of the current platform or the
tournament rules.

---

## 2. Turn structure

A turn consists of exactly one of: Strength attack, Mind attack, or Skip. The turn then passes
(`js/engine.js:129`, `server/game-logic.js:215, 292, 339`).

| Question | Ruling | Source |
|---|---|---|
| Captures per turn | Exactly one; both attack types capture exactly one target | `js/engine.js:111` |
| Can a turn capture zero dice? | Only by skipping | `js/engine.js:82-89` |
| Is the turn consumed if the attack is rejected? | No. The shared engine returns an explicit error, leaves state untouched, and keeps the same player on move | `packages/games/maths-warriors/src/engine.ts` |

---

## 3. Strength attack

| Rule | Ruling | Source |
|---|---|---|
| Number of attacking dice | Exactly 1. Two or more is rejected | `js/engine.js:99` |
| Threshold (normal) | `attacker.value >= target.value` | `js/engine.js:101`, `server/game-logic.js:192-194` |
| Threshold (first-player final capture) | `attacker.value > target.value` | `js/engine.js:100-101`, `server/game-logic.js:187-190` |
| Target must be uncaptured | Yes | `js/engine.js:93`, `server/game-logic.js:178` |
| Attacker must be uncaptured and owned by mover | Yes | `js/engine.js:95`, `server/game-logic.js:177` |
| Effect on attacker | Rerolled | `js/engine.js:119-121`, `server/game-logic.js:202` |

---

## 4. Mind attack

### 4.1 Structural requirements

| Rule | Ruling | Source |
|---|---|---|
| Minimum dice | 2 | `js/engine.js:103`, `server/game-logic.js:252` |
| Maximum dice | 6 (no explicit cap; bounded by dice owned) | — |
| Expression must equal target **exactly** | Yes | `js/engine.js:106`, `server/game-logic.js:268` |
| Die used twice in one expression | Rejected (`Die used twice`) in every mode | `packages/games/maths-warriors/src/engine.ts` |
| Die values tamper-check | Token value must match the real die value in every mode | `packages/games/maths-warriors/src/engine.ts` |
| Selected-but-unused dice | Impossible — attacking dice are derived from the expression | `packages/games/maths-warriors/src/engine.ts` |
| Expression token limit | 64 tokens (online, silently truncated) | `server/rooms.js:34, 42` |
| Effect on used dice | All rerolled | `js/engine.js:119-121`, `server/game-logic.js:274-280` |

### 4.2 Arithmetic — the evaluator

Both engines use the same shunting-yard algorithm (`js/expression.js:84-170`,
`server/game-logic.js:79-156`).

| Question | Ruling | Source |
|---|---|---|
| **Operators permitted** | `+ − × ÷` only | `js/expression.js:79`, `server/rooms.js:35` |
| **Precedence** | `* /` = 2, `+ -` = 1. Standard | `js/expression.js:79` |
| **Associativity** | Left, for equal precedence (`>=` in the pop condition) | `js/expression.js:148` |
| **Parentheses** | Fully supported, arbitrarily nested. Unbalanced → rejected | `js/expression.js:126-142, 162` |
| **Is division restricted to exact results?** | **No.** Non-integer *intermediate* values are permitted without restriction. Only the **final** result is required to be an integer | `js/expression.js:168`, `server/game-logic.js:155` |
| **Worked case** | `3 / 2 * 4` = 6 → **LEGAL**. `7 / 2` = 3.5 → **REJECTED** | as above |
| **Are negative intermediates permitted?** | **Yes, without restriction.** No sign check exists at any point. `2 - 5 + 8` = 5 is legal | `js/expression.js:102-117`; asserted in `tests/game-core.test.js:28` |
| **Can the final result be negative or zero?** | Structurally it may evaluate so, but it can never match a target: die values are `1..sides` (`js/state.js:4`), so any non-positive result simply fails the equality check | `js/engine.js:106` |
| **Division by zero** | Rejected, `Division by zero` | `js/expression.js:107`, `server/game-logic.js:98` |
| **Non-finite results** | Rejected | `js/expression.js:114, 167` |
| **Malformed expression** | Rejected: trailing operator, two adjacent values, empty | `js/expression.js:122, 144, 160, 166` |

### 4.3 Client-side input restrictions (offline UI)

These constrain what a player can *build* on screen. They are UI guards, not engine rules — a
crafted client is bounded only by §4.1 and §4.2.

| Behaviour | Source |
|---|---|
| An operator cannot be placed as the first token, or directly after `(` | `js/expression.js:9, 13` |
| Typing a second operator **replaces** the trailing one | `js/expression.js:14` |
| `(` only allowed at the start, after an operator, or after another `(` | `js/expression.js:29-36` |
| `)` only allowed after a value or another `)`, and only if an `(` is outstanding | `js/expression.js:40-44` |
| Selecting a die auto-inserts `+` if the previous token was a value or `)` | `js/ui.js:794-797` |
| Clicking an already-selected die **deselects** it and surgically removes it plus one adjacent operator | `js/ui.js:783-791` |
| Empty `()` pairs and dangling operators are auto-stripped | `js/expression.js:52-78` |
| Attack button enabled only if: target set, and (Strength: exactly 1 die meeting threshold) or (Mind: ≥2 dice and expression == target) | `js/expression.js:261-280` |
| Switching Strength→Mind carries a single already-selected die into the expression | `js/ui.js:696-705` |
| Keyboard: number keys select own dice by value; Shift+number selects target | `js/keyboard.js:11-26` |
| Keyboard: pressing an operator auto-switches to Mind mode | `js/keyboard.js:109-115` |

---

## 5. The First Player Penalty

**Statement.** When the mover is the first player AND the opponent has exactly **one** active die
remaining, a **Strength** attack requires `>` rather than `>=`.

- Client: `js/engine.js:100-101`
- Server: `server/game-logic.js:184-195`
- Move generator: `js/engine.js:141, 144`
- UI gate: `js/expression.js:272-274`
- UI highlight of legal attackers: `js/ui.js:316-322`
- On-screen warning banner: `js/ui.js:673-683`

**It does NOT apply to Mind attacks.** Neither engine applies any strict-inequality condition to
a Mind attack (`js/engine.js:102-106`, `server/game-logic.js:240-268`). The first player may win
by Mind attack with an expression exactly equal to the last die's value.

> **Judges: this is the single most likely dispute at a tournament.** The implementation's own
> help page (`how-to-play.html`) explicitly states the penalty "applies to both Strength and Mind
> Attacks on the last remaining opponent die." The code does not. Rule in advance and announce
> it. See `docs/OPEN-QUESTIONS.md` item 1.

**Trigger condition is "opponent has 1 die left," not "this move wins."** Since every capture
removes exactly one die, these coincide.

---

## 6. Skipping and stalemate

| Question | Ruling | Source |
|---|---|---|
| When is skipping legal? | **Always**, on your own turn. No legality precondition is checked | `js/game.js:205-221`, `server/game-logic.js:334-351` |
| Is skipping ever forced? | **Never by rule.** Forced only mechanically by move-timer expiry (§7.2) | `js/timer.js:76-82`, `server/rooms.js:286-292` |
| May a player skip while holding a legal attack? | Yes. Nothing prevents it | as above |
| What if a player has **no** legal move? | **No rule exists.** The engines do not detect this condition for a human player. There is no auto-skip, no stalemate, no draw | — (absence; nearest handling is AI-only, `js/ai.js:34-41`) |
| Can both players skip indefinitely? | Yes. The game runs until the game clock expires and §8.2 decides it | `js/timer.js:45-51` |
| Does a skip increment the move counter? | Offline skip: yes (`js/engine.js:83`). Offline move-timer expiry: **no** (§12.6). Online, both: yes (`server/game-logic.js:338`) |
| Is there a draw outcome? | **No.** No code path produces a draw | — (absence) |

---

## 7. Clocks

### 7.1 Game clock

| Item | Offline | Online |
|---|---|---|
| Default | 12 min = 720 s (`js/config.js:13`) | 720 s (`server/game-logic.js:25`) |
| Configurable range | 1–60 min, or blank/0 for unlimited (`index.html:345`, `js/main.js:815-816`) | Clamped 60–3600 s. **Unlimited is not possible online** (`server/rooms.js:143`) |
| Tick | Client `setInterval` 1 s (`js/timer.js:11-20`) | Server `setInterval` 1 s, authoritative (`server/rooms.js:264-303`) |
| Sync to clients | n/a | `time_sync` broadcast every 5 s (`server/rooms.js:296-302`) |
| Runs during opponent/AI thinking? | **Yes** — the game clock never pauses (`js/timer.js:11-20`) | Yes |
| Warning states | ≤180 s "warning", ≤60 s "critical", audible tick in final 10 s (`js/timer.js:15, 41-42`) | — |

**On expiry** → §8.2.

### 7.2 Move clock

| Item | Offline | Online |
|---|---|---|
| Default | **Off** (0) (`js/config.js:14`, `index.html:356`) | 30 s (`server/game-logic.js:25`) |
| Configurable range | 0–120 s (`index.html:356`) | Clamped 0–120 s (`server/rooms.js:144`) |
| Pauses while AI is thinking | **Yes** (`js/timer.js:66`) | n/a |
| On expiry | Turn passes to opponent. **Does not** route through the skip path: `moveCount` is not incremented and no game-log entry is written (`js/timer.js:76-82`) | Routed through `skipTurn`; `moveCount` **is** incremented, `turn_skipped` broadcast with `reason: 'timeout'` (`server/rooms.js:286-292`, `server/game-logic.js:338`) |
| Reset on each move | Yes (`js/timer.js:55-63`, `server/game-logic.js:216, 293, 340`) |

**Divergence:** see §12.6.

---

## 8. Game end

### 8.1 Win by capture

Opponent has 0 active dice → game over, mover wins, `winReason: 'all_captured'`
(`js/engine.js:124-127`, `server/game-logic.js:356-363`).

Evaluated **after** the capture and **before** the turn would pass; a winning move does not pass
the turn (`js/engine.js:124-130`).

### 8.2 Win on game-clock expiry

```
p1Remaining = count of P1's uncaptured dice
p2Remaining = count of P2's uncaptured dice

if p1Remaining < p2Remaining  → Player 1 wins
if p2Remaining < p1Remaining  → Player 2 wins
if equal                      → the player who did NOT move first wins
```

- Offline: `js/timer.js:45-51`
- Online: `server/game-logic.js:368-382`, `winReason: 'timeout'`

> ⚠️ **The player with FEWER of their OWN dice remaining wins.** This is written identically in
> both engines, so it is not a single-site slip — but it inverts the intuitive reading and
> directly contradicts `how-to-play.html`, which states the winner is "the player with fewer
> remaining opponent dice." **Judges must announce which reading governs before the event.**
> See `docs/OPEN-QUESTIONS.md` item 2 and `docs/SUSPECTED-BUGS.md` item 8.

### 8.3 Win by forfeit (online only)

See §9.

### 8.4 Match structure

Each game is standalone in local, AI and online play. Best-of-3 state belonged to the archived
client wrapper and was deliberately not carried into the current platform. An event that wants a
multi-game match must define that event format separately; it does not change the rules of an
individual game.

---

## 9. Disconnection, reconnection, forfeit (online only)

| Event | Handling | Source |
|---|---|---|
| Socket closes / heartbeat fails | Player marked disconnected; opponent notified `opponent_disconnected` | `server/rooms.js:499-513`, `server/index.js:107-110, 124-133` |
| Heartbeat interval | 30 s ping; no pong → terminate + disconnect | `server/index.js:124-133` |
| Disconnect during **lobby** (`status: 'waiting'`) | Room destroyed immediately, no result recorded | `server/rooms.js:504-508` |
| Disconnect during **game** | **30-second** reconnect window | `server/rooms.js:516, 531` |
| Reconnect within window | Requires matching `code`, `playerId`, and `sessionToken`. Full state snapshot resent; opponent notified `opponent_reconnected` | `server/rooms.js:537-580` |
| Window expires | Game over. `winner` = opponent, `winReason: 'forfeit'`. Result recorded to the ladder | `server/rooms.js:516-531` |
| Room teardown after forfeit | +60 s | `server/rooms.js:530` |
| If **both** players are gone | Forfeit still resolves and is recorded; nobody receives `game_over` | `server/rooms.js:518-527` |
| Server restart | `server_shutdown` broadcast; clients disconnect. **In-flight games are lost — rooms are in-memory only** | `server/index.js:141-158`, `server/rooms.js:10` |

### 9.1 "Claim Victory" — judges read this

The disconnect overlay presents a **Claim Victory** button (`index.html:1100`).

> **The button does nothing.** Its handler plays a click sound and closes the overlay. It sends
> no message to the server (`js/main.js:686-690`).

Forfeit is awarded **solely** by the server's 30-second timer. A player who clicks "Claim
Victory" has not claimed anything; a player who does not click it loses nothing. The countdown
displayed alongside it is a purely cosmetic client-side counter that stops at 0 and takes no
action (`js/multiplayer.js:589-606`).

**Ruling:** treat "Claim Victory" as a no-op. Forfeits are automatic and server-determined.

### 9.2 Rematch

Both players must request/accept; a new game starts only when both have
(`server/rooms.js:460-477`). Decline clears the request set (`server/rooms.js:478-483`).

### 9.3 Room lifecycle

| Condition | Action | Source |
|---|---|---|
| Waiting room older than 5 min | Destroyed | `server/rooms.js:624, 629-631` |
| Finished game, older than 2 min, no rematch pending, nobody connected | Destroyed | `server/rooms.js:625, 632-641` |
| Server room cap | 50 concurrent rooms | `server/rooms.js:111-117` |
| Room codes | 5 chars from `23456789ABCDEFGHJKMNPQRSTUVWXYZ` — no `0/O/1/I/L` | `server/rooms.js:13, 75-84` |

---

## 10. Ranked play and integrity

| Item | Ruling | Source |
|---|---|---|
| Ranked requires authentication | Both creator and joiner must be signed in | `server/rooms.js:120-122, 184-186` |
| Same account both sides | Rejected: "Ranked matches require two different accounts" | `server/rooms.js:187-189` |
| Guest identity format | `Guest_XXXX`, validated server-side | `server/rooms.js:86-88` |
| Rating applied | Only when `playStyle === 'ranked'` **and both players authenticated** | `server/supabase.js:196-199` |
| Casual online games | Logged to match history with `elo_delta: null` | `server/supabase.js:105-119` |
| Offline games (vs AI / local P2) | Logged as **casual** to the signed-in player's history; never rated | `js/game.js:301-309`, `js/auth.js:164-192` |
| Move rate limit | 10 messages/sec/client, excess dropped with an error | `server/index.js:16, 85-96` |
| Expression sanitisation | Type/operator/paren whitelist; anything unrecognised voids the whole expression | `server/rooms.js:38-58` |
| Origin check | WebSocket connections verified against `CORS_ORIGIN` | `server/index.js:32-35, 63-70` |

**Undo** (`js/undo.js`): available **offline only** (`js/undo.js:33`). One snapshot deep, taken
before each attack (`js/game.js:53`), usable on your own turn or while the AI is thinking
(`js/undo.js:36-39`). Cleared at game start (`js/game.js:19`). Keyboard `U` (`js/keyboard.js:90`).
**Judges: undo has no server equivalent and should be disallowed in any rated offline event.**

---

## 11. ELO

| Item | Value | Source |
|---|---|---|
| Starting rating | 1000 | `sql/supabase_auth_leaderboard_setup.sql:59, 289` |
| K-factor | 24 | `server/supabase.js:9`, `sql/…:153` |
| Expected score | `E = 1 / (1 + 10^((loserElo − winnerElo)/400))` | `server/supabase.js:101` |
| Delta | `round(24 × (1 − E))`, applied `+delta` to winner and `−delta` to loser (zero-sum) | `server/supabase.js:102, 152-158` |
| Rating floor | 100 | `server/supabase.js:152, 156`, `sql/…:156, 161` |
| Rating ceiling | None | — |
| No draw handling | Correct — no draws exist | — |
| Preferred path | Postgres RPC `record_ranked_match`, row-locked (`FOR UPDATE`) | `sql/…:119-174` |
| Fallback path | Read-modify-write from the Node server if the RPC is missing. **Not atomic** | `server/supabase.js:136-183` |
| Forfeits rated? | **Yes** — `finalizeGame` is called on forfeit with the same path as a normal win | `server/rooms.js:523`, `server/supabase.js:190-205` |
| Timeouts rated? | **Yes** | `server/rooms.js:277` |

Rank tiers are cosmetic and derived from ELO (or, for signed-out users, local win count):
Beginner 0 / Bronze 800 / Silver 1000 / Gold 1200 / Platinum 1400 / Diamond 1600 / Master 1800 /
Grandmaster 2000 (`js/ranks.js:3-12`).

---

## 12. Shared-engine parity record

The Phase 2 migration removed the archived client/server rules divergence. These choices are now
applied consistently in local, AI and online play:

| Historical item | Shared ruling |
|---|---|
| 12.1 Selected-but-unused dice | Dice are derived from the expression; an unused selection cannot exist |
| 12.2 Same die used twice | Rejected |
| 12.3 First-player comparison order | Values are sorted before comparison; open question 8 remains flagged |
| 12.6 Move-timer expiry | Routed through skip and increments the move count |
| 12.7 Reroll RNG | Seeded; the seed travels with the move so replay verification is deterministic |
| 12.8 Rejected move handling | Explicit error, state unchanged |
| 12.10 Die value tampering | Expression values must match the real dice |

Unlimited clock support, match wrappers and undo are product or event-format questions rather
than alternate rules engines. They must not be used to produce different arithmetic or capture
rulings by mode.

---

## 13. Common judge questions — fast answers

| Question | Answer | Source |
|---|---|---|
| Can a Mind attack use only one die? | No. Minimum two | `js/engine.js:103` |
| Can a Strength attack use two dice? | No. Exactly one | `js/engine.js:99` |
| Can I attack my own die? | No. Targets come from the opponent's array only | `js/engine.js:93` |
| Can I target an already-captured die? | No | `js/engine.js:93` |
| Does `3/2*4 = 6` count? | Yes — legal | `js/expression.js:168` |
| Does `7/2 = 3.5` count? | No | `js/expression.js:168` |
| Are negative intermediates legal? | Yes, unrestricted | `js/expression.js:102-117` |
| Do brackets work? | Yes, fully nested | `js/expression.js:126-142` |
| Do captured dice come back? | Never | `js/engine.js:111` |
| Does the capturer gain the captured die? | No. It is removed from play | `js/engine.js:111` |
| Are unused dice rerolled? | No — only dice used in the attack | `js/engine.js:119-121` |
| Is the target die rerolled before capture? | No | `js/engine.js:111` |
| Can I skip when I have a legal move? | Yes | `js/game.js:205` |
| What if I have no legal move? | No rule. You may skip; nothing forces or detects it | — |
| Is there a draw? | No | — |
| Does the first-player penalty apply to Mind attacks? | **Not in code.** Disputed — see §5 | `js/engine.js:102-106` |
| Who wins on timeout? | Fewer own dice remaining; tie → non-starter. **See the §8.2 warning** | `js/timer.js:46-48` |
| Does "Claim Victory" do anything? | No | `js/main.js:686-690` |
| Is a forfeit rated? | Yes | `server/rooms.js:523` |
| Can a player rejoin after 30 s? | No — the game is already forfeited | `server/rooms.js:516-531` |

---

## 14. Out of scope

**Math Mastermind ("3M")** — `js/mastermind-core.js`, `js/mastermind-page.js`,
`math-mastermind/index.html` — is a **separate game** shipped in the same repository. It shares
no rules with Maths Warriors and is not covered here.

**Puzzle trainer** is a single-player practice feature, not a match format. Its expressions use
the same shared evaluator and standard precedence as the game. Puzzle outcomes do not create
tournament match results.

**Tutorial** — `js/tutorial.js` — scripted; gates player actions and suppresses the victory
modal (`js/game.js:275`). Not a play mode.
