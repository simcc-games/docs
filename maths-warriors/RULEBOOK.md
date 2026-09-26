<!--
locale: en
rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: current-implementation
-->

# Maths Warriors — Rulebook

**Learn to play.** Read this once and you'll be ready for your first game.

Every rule below has a small source note in brackets. Paths beginning with `js/` or `server/`
refer to the archived implementation from which the rules were reconstructed. The current web
client and game server share the pure TypeScript engine in
`packages/games/maths-warriors/src`. You can ignore the notes while learning — they remain so a
teacher or judge can audit how the rule was established.

---

## 1. What you need

Two players. Each player gets **six dice**, one of each kind (`js/config.js:1`):

| Die | Numbers it can show |
|-----|---------------------|
| D4  | 1–4   |
| D6  | 1–6   |
| D8  | 1–8   |
| D10 | 1–10  |
| D12 | 1–12  |
| D20 | 1–20  |

That's it. No board, no cards, no score sheet.

---

## 2. Setting up

1. **Everyone rolls everything.** All twelve dice are rolled — your six and your opponent's six
   (`js/state.js:25-26`). Whatever number each die lands on is its **value** for now.
2. **Line them up.** Each player's dice are sorted so they're easy to read
   (`js/state.js:31-34`). By default they're sorted from lowest value to highest.
3. **Decide who starts.** Compare the two lines of dice, one position at a time, left to right.
   The first time you find a difference, whoever has the **smaller** number goes first
   (`js/state.js:36-40`). If every single pair is equal, Player 1 goes first
   (`js/state.js:39`).

> ⚠️ **All-equal tiebreak is disputed.** The code deterministically gives the start to Player 1,
> but the game's own help page promises *"a tiebreaker roll decides."* Neither engine implements
> a roll-off. See `docs/OPEN-QUESTIONS.md` item 9 — **ask your judge before a rated game.**

> **Going first is not purely an advantage.** There's a small handicap attached to it, called the
> First Player Penalty. See section 6.

---

## 3. What you're trying to do

**Capture all six of your opponent's dice.** The moment their last die is captured, you win
(`js/engine.js:124-127`).

A captured die is gone for good. It doesn't come back, it doesn't get rerolled, and you don't
get to keep it or use it — it just leaves the game (`js/engine.js:111`).

---

## 4. A turn

Players take turns. On your turn you do **exactly one** of three things:

1. A **Strength attack**
2. A **Mind attack**
3. **Skip** (do nothing)

Then it's the other player's turn (`js/engine.js:129`).

You can only ever capture **one** die per turn, because both attack types capture exactly one
target (`js/engine.js:111`).

### Strength attack — the simple one

Pick **one** of your dice and **one** of your opponent's dice.

> **Your die's value must be equal to or bigger than theirs.** (`js/engine.js:101`)

If it is, you capture their die. Then **your attacking die is rerolled** — it gets a brand new
random value (`js/engine.js:119-121`).

*Example:* your D12 shows **9**. Their D8 shows **7**. `9 ≥ 7`, so you capture their D8. Your
D12 rerolls and might now show anything from 1 to 12.

### Mind attack — the clever one

Pick **two or more** of your dice (`js/engine.js:103`) and join them together with maths to make
a sum that is **exactly** equal to the value of the die you're attacking (`js/engine.js:105-106`).

You may use:

- `+` add, `−` subtract, `×` multiply, `÷` divide (`js/expression.js:79`)
- brackets `(` `)` to control what happens first (`js/expression.js:22-51`)

If your sum lands exactly on their number, you capture that die — **and every die you used in
the sum gets rerolled** (`js/engine.js:119-121`).

*Example:* their D20 shows **18**. You have a D6 showing **3** and a D8 showing **6**.
`3 × 6 = 18`. Exact match — their D20 is captured, and your D6 and D8 both reroll.

### Skipping

You can always choose to skip your turn (`js/game.js:205-221`). Nothing happens; play passes to
your opponent. Skipping is never forced on you, and you're allowed to skip even when you *do*
have a legal attack available — the game does not check (`server/game-logic.js:334-351`).

---

## 5. The maths rules, precisely

These are the rules the game uses when it checks your sum (`js/expression.js:84-170`). They
matter, so read them once carefully.

**Normal order of operations.** `×` and `÷` happen before `+` and `−`, and things in brackets
happen first (`js/expression.js:79`, `js/expression.js:143-153`). So `2 + 3 × 4` is **14**, not
20. If you want 20, write `(2 + 3) × 4`.

**Left to right for ties.** `10 − 3 − 2` is `(10 − 3) − 2 = 5`, not `10 − (3 − 2)`
(`js/expression.js:148`).

**Your final answer must be a whole number.** If your sum works out to 3.5, it's rejected
(`js/expression.js:168`).

**But halfway through, anything goes.** Fractions and negative numbers are completely fine as
long as the *final* answer is a whole number (`js/expression.js:84-170` — there is no check on
intermediate values).

- `3 ÷ 2 × 4` = **6** ✔ legal (it passes through 1.5 on the way)
- `2 − 5 + 8` = **5** ✔ legal (it passes through −3 on the way)
- `7 ÷ 2` = 3.5 ✘ rejected

**You can't divide by zero** (`js/expression.js:107`) — though since dice never show 0, this
almost never comes up.

**Each die can be used only once** in a single sum (`server/game-logic.js:259`). Your D6 showing
4 cannot become "4 + 4".

**You must actually use every die you picked.** Picking a die and then not putting it in the sum
isn't how the game is meant to work (`server/game-logic.js:251`) — and on the screen, picking a
die automatically drops it into your sum (`js/ui.js:792-798`).

---

## 6. The First Player Penalty

The player who moved first has one small handicap, and it only ever matters on the very last
capture of the game.

> When the first player attacks their opponent's **last remaining die** using a **Strength
> attack**, their die's value must be **strictly greater** than the target — `>` instead of `≥`
> (`js/engine.js:100-101`, `server/game-logic.js:184-195`).

So if you went first and your opponent has one die left showing **6**, a die of yours showing 6
is *not* enough. You need 7 or more.

This only applies to the player who went first, and only to Strength attacks. Everything else
stays the same.

*(In the game's own help page this penalty is described as also applying to Mind attacks. The
code does not do that — see `docs/OPEN-QUESTIONS.md`, item 1. Ask your judge which one your
event is using.)*

---

## 7. Clocks

There are two optional clocks (`js/timer.js`).

**Game clock.** One clock for the whole game, shared. The default is **12 minutes**
(`js/config.js:13`, `index.html:345`). If it runs out before anyone has been wiped out, the game
ends immediately and the winner is decided by counting dice — see below (`js/timer.js:45-51`).

**Move clock.** An optional per-turn clock, **off by default** (`js/config.js:14`,
`index.html:356`). If it runs out on your turn, your turn is skipped automatically
(`js/timer.js:76-82`).

---

## 8. How the game ends

There are two ways to win.

**By capture.** You captured all six of your opponent's dice (`js/engine.js:124-127`). This is
the normal way to win.

**On the clock.** The game clock ran out. The winner is the player with **fewer dice left on
their own side**; if both players have the same number of dice left, the player who did *not*
go first wins (`js/timer.js:45-51`, `server/game-logic.js:368-382`).

> ⚠️ That second rule reads strangely — you'd expect *more* dice left to be better. It's written
> the same way in both halves of the code, so it's not a typo in one place, but it does
> contradict the game's own help page. It's logged in `docs/OPEN-QUESTIONS.md` (item 2) and
> `docs/SUSPECTED-BUGS.md` (item 8). **Ask your judge before a rated game.**

There is no draw. There is no rule for what happens if neither player can move.

---

## 9. A full worked turn

Here's one complete turn, start to finish.

**The position.** It's your turn. You did *not* go first, so the First Player Penalty doesn't
apply to you at all.

| | D4 | D6 | D8 | D10 | D12 | D20 |
|---|---|---|---|---|---|---|
| **You** | 2 | 4 | 6 | 3 | 11 | 7 |
| **Opponent** | 1 | 5 | 8 | 10 | 10 | 19 |

**Step 1 — Look for a target.** Their D20 showing **19** is the scariest die on the board. You'd
love to remove it.

**Step 2 — Can you take it by Strength?** Your biggest die is the D12 showing 11. `11 ≥ 19` is
false, so no (`js/engine.js:101`). Strength won't work on that one.

**Step 3 — Try a Mind attack.** You need a sum that hits exactly **19**, using two or more of
your dice (`js/engine.js:103`).

Have a look at what you've got: 2, 4, 6, 3, 11, 7.

- `11 + 7 = 18` — one short.
- `11 + 6 + 2 = 19` — **that works.**

**Step 4 — Check it against the rules.**

- Two or more dice? Three dice. ✔ (`js/engine.js:103`)
- Every die used once only? Yes — D12, D8 and D4, each once. ✔ (`server/game-logic.js:259`)
- Whole number at the end? 19. ✔ (`js/expression.js:168`)
- Exactly equal to the target's 19? Yes. ✔ (`js/engine.js:106`)
- First Player Penalty? Doesn't apply — you weren't the first player
  (`js/engine.js:100`). And even if you *were*, they have six dice left, not one, so it still
  wouldn't apply.

**Step 5 — Attack.** Their D20 is captured and leaves the game permanently
(`js/engine.js:111`).

**Step 6 — Reroll.** The three dice you used — your D12, D8 and D4 — are all rerolled
(`js/engine.js:119-121`). Say they come up 4, 1 and 3. Your D6, D10 and D20 are untouched,
because you didn't use them.

**The position now:**

| | D4 | D6 | D8 | D10 | D12 | D20 |
|---|---|---|---|---|---|---|
| **You** | 3 | 4 | 1 | 3 | 4 | 7 |
| **Opponent** | 1 | 5 | 8 | 10 | 10 | — |

**Step 7 — Pass the turn.** It's your opponent's move (`js/engine.js:129`).

Notice the cost: to take their best die you spent your three best numbers, and the rerolls came
back low. That trade-off is the heart of the game.

---

## 10. Quick reference

| | |
|---|---|
| Dice per player | 6 — D4, D6, D8, D10, D12, D20 (`js/config.js:1`) |
| Goal | Capture all 6 opponent dice (`js/engine.js:124-127`) |
| Strength attack | 1 die, value `≥` target (`js/engine.js:101`) |
| Mind attack | 2+ dice, sum `=` target exactly (`js/engine.js:103-106`) |
| Operators | `+ − × ÷` and brackets (`js/expression.js:79`, `js/expression.js:22-51`) |
| Order of operations | Standard — brackets, then `× ÷`, then `+ −` (`js/expression.js:143-153`) |
| Final answer | Must be a whole number (`js/expression.js:168`) |
| Halfway answers | Negatives and fractions allowed (`js/expression.js:84-170`) |
| Dice used in an attack | Rerolled (`js/engine.js:119-121`) |
| Dice captured | Removed permanently (`js/engine.js:111`) |
| Skipping | Always allowed (`js/game.js:205-221`) |
| First Player Penalty | Strength only, last enemy die only, `>` not `≥` (`js/engine.js:100-101`) |
| Default game clock | 12 minutes (`js/config.js:13`) |
| Default move clock | Off (`js/config.js:14`) |

---

*This rulebook was reconstructed from the source code of the web implementation. Where the code
was unclear or appeared to contradict itself, nothing was invented — those points are recorded
in `docs/OPEN-QUESTIONS.md` and `docs/SUSPECTED-BUGS.md`.*
