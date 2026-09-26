<!--
locale: en
rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: unresolved-rulings
-->

# Maths Warriors — Open Questions

Points where the source is genuinely ambiguous, where a rule appears to be *implicit* rather than
enforced, or where two authorities in the repository disagree. **Nothing here has been guessed at
in the rulebook or the tournament reference.** Open items need a human ruling; closed items remain
here as an audit trail and are labelled explicitly.

Things that look like implementation defects rather than rules are in `SUSPECTED-BUGS.md`. A few
items are cross-referenced where the distinction is unclear.

---

## 1. Does the First Player Penalty apply to Mind attacks?

**Priority: HIGH — this will be asked at a tournament.**

The code applies the strict-`>` requirement to **Strength attacks only**
(`js/engine.js:100-101`, `server/game-logic.js:184-195`). Neither engine imposes any condition on
a Mind attack against the opponent's last die (`js/engine.js:102-106`,
`server/game-logic.js:240-268`). The move generator agrees (`js/engine.js:141-153` applies it;
`js/engine.js:156-176` does not).

The project's own help page states the opposite: *"This applies to both Strength and Mind Attacks
on the last remaining opponent die"* (`how-to-play.html`, "First Player Penalty" section).

A Mind attack requires *exact* equality, so "strictly greater" has no natural meaning for it —
which may be why it was omitted. But then the intended rule might be that the first player simply
**cannot** win by Mind attack, or must exceed by producing a value the target cannot equal. Both
are coherent; neither is implemented.

**Needs:** an authoritative statement of the original SIMOC rule.

---

## 2. Is the timeout win condition inverted?

**Priority: HIGH.**

Both engines award the timeout win to the player with **fewer of their own dice remaining**
(`js/timer.js:46-48`, `server/game-logic.js:371-375`, whose comment reads `// fewer remaining =
better`). The help page states the winner is *"the player with fewer remaining opponent dice"* —
the opposite.

Because both engines agree, this is not a transcription slip in one place; it is either the
intended rule or a defect replicated during the port. Under the rule as coded, deliberately
sacrificing your own dice near time expiry is winning play, which is hard to defend as design.

Cross-referenced in `SUSPECTED-BUGS.md` item 8.

**Needs:** the original rule. If the code is wrong, both engines require the fix.

---

## 3. Is there a stalemate or no-legal-move rule?

There is **no handling anywhere** for a human player with no legal move. The engines never test
for it; `legalMoves` is only invoked for the AI and for review (`js/ai.js:165`,
`js/search.js:2`). The AI skips when the list is empty (`js/ai.js:34-41`); a human simply
receives a disabled attack button and must skip manually
(`js/expression.js:279`).

Consequently two players can skip indefinitely and the game is decided by clock expiry.

**Needs:** does the physical game have a stalemate rule, an auto-skip, or a forced-loss condition?
Note also that with an unlimited game clock (permitted offline, `js/timer.js:21-25`) a
mutual-skip game never terminates at all.

---

## 4. Is there a draw?

No code path anywhere produces a draw. Clock expiry with equal dice counts resolves to the
non-starter (`js/timer.js:48`), and ELO has no draw term (`server/supabase.js:100-103`).

**Needs:** confirmation that the physical game genuinely has no draw, or a definition of one.

---

## 5. Is "best of 3" part of the game or a UI wrapper?

Offline play wraps games in a best-of-3 match: 3 rounds, 2 to win (`js/state.js:2`,
`js/game.js:287, 298`). The server has no such concept — each online game is standalone
(`server/rooms.js:225-256`), with repeat play only via the rematch handshake
(`server/rooms.js:453-484`).

**Needs:** is a SIMOC "match" one game or a best-of-3? This determines whether the online mode is
missing a feature or the offline mode has an extra one. (The client also creates a `matchState`
for online games, which misbehaves — `SUSPECTED-BUGS.md` item 5.)

---

## 6. Should the game clock pause?

The move clock pauses while the AI is thinking (`js/timer.js:66`). The game clock does not
(`js/timer.js:11-20`) — it runs through AI think time (600–2000 ms per turn, `js/config.js:2-7`),
the AI's step-by-step move visualisation (~500 ms per step, `js/ai.js:66, 148`), and all dice
animations (~700 ms, `js/game.js:170, 200`). Against Impossible over a long game this is on the
order of a minute of shared clock consumed by presentation.

Online, the server clock ticks independently of any client animation
(`server/rooms.js:264-303`).

**Needs:** is the game clock intended to be pure wall-clock, or thinking time only?

---

## 7. Is `alwaysFirst` a legitimate setting?

`settings.alwaysFirst` forces Player 1 to move first, bypassing dice comparison entirely
(`js/state.js:37`), and is exposed in the UI (`js/main.js:257-259`) and persisted
(`js/main.js:976, 1001-1004`).

It is offline-only and has no server equivalent.

**Needs:** is this a practice/teaching aid, an accessibility feature, or a debug control that was
never removed? Judges need to know whether to require it off.

---

## 8. Does board sort order form part of the rules?

`diceSort` (`js/state.js:1`) is presented as a display preference, but offline it feeds directly
into first-player determination, which compares the arrays *in board order*
(`js/state.js:38`). With `diceSort === 'type'` the comparison is D4-vs-D4, D6-vs-D6, … rather
than sorted-value-vs-sorted-value. The server always sorts by value first
(`server/game-logic.js:61-62`).

If the original rule is "compare sorted values," the client is wrong (`SUSPECTED-BUGS.md`
item 3). If the original rule is "compare like die against like die," the *server* is wrong.

**Needs:** the original comparison procedure. These give different first players on real
positions.

---

## 9. What is the "tiebreaker roll"?

The help page states that if all six pairs are equal, *"a tiebreaker roll decides."* Both engines
instead return Player 1 deterministically (`js/state.js:39`, `server/game-logic.js:67`).

**Needs:** does the physical game roll off? If so, both engines are missing it. (The probability
of a full six-way tie is small but not negligible; it happens.)

---

## 10. Must every selected die be used in a Mind attack expression?

The server makes this structurally impossible by deriving the dice *from* the expression
(`server/game-logic.js:251`). The client tracks selection and expression as independent lists,
validates the expression, and then rerolls the *selection* (`js/engine.js:95, 103, 119-121`).

The stock UI keeps them in sync (`js/ui.js:792-798`), so this cannot normally be observed. But it
means the offline engine would accept a Mind attack where a selected die contributes nothing yet
is still rerolled.

**Needs:** is "select it, must use it" a rule, or merely an artefact of the interface? See
`SUSPECTED-BUGS.md` item 1 for the mechanical divergence.

---

## 11. Are parentheses intended to be available to both sides?

Human players can bracket freely (`js/expression.js:22-51`). The AI's move generator emits only
flat `die op die op die …` sequences and never produces a parenthesis
(`js/engine.js:167-171`). The same limitation applies to the post-game review engine
(`js/search.js:2`), so a review may report a human's bracketed move as a "Blunder" purely because
the analyser could not find the bracketed alternative.

**Needs:** is this a deliberate handicap, or an unimplemented capability? It affects how "best
move" annotations should be read.

---

## 12. Is undo permitted, and what is its scope?

Undo is offline-only, one level deep (`js/undo.js:8-40`), and — because the snapshot is taken
before the reroll (`js/game.js:53`) and each attempt draws a fresh seed
(`js/game.js:230`) — it allows a player to re-draw an unfavourable reroll, not merely to correct a
misclick.

**Needs:** is undo a teaching aid only? Should rated offline play disable it?

---

## 13. Does move-timer expiry count as a move?

Offline, a move-timer expiry swaps the player without incrementing `moveCount` or writing a game
log entry (`js/timer.js:76-82`). Online, it routes through `skipTurn` and does increment
(`server/rooms.js:288`, `server/game-logic.js:338`).

Move count feeds post-game statistics (`js/stats.js:106-114`) and the review log
(`js/game.js:235-252`), so the two modes produce different records of the same game.

**Needs:** should a timed-out turn be recorded as a move? (Mechanically flagged in
`SUSPECTED-BUGS.md` item 6.)

---

## 14. Closed — the puzzle uses the game's arithmetic

**Status: CLOSED as an implementation defect.** Phase 1 replaced the archived puzzle's
left-to-right evaluator with the same standard-precedence evaluator used by match play. The
current puzzle trainer consumes the shared TypeScript evaluator, so puzzle and game arithmetic
agree. No tournament rule was changed.

---

## 15. Is unlimited game time a supported format?

Offline permits a game clock of 0 = unlimited (`js/main.js:815-816`, `js/timer.js:21-25`). Online
clamps the clock to 60–3600 s, so unlimited is unreachable (`server/rooms.js:143`).

**Needs:** is untimed play a legitimate format? If so, note that it combines with item 3 to
produce genuinely non-terminating games.

---

## 16. Should online opponents see your expression as you build it?

Online mode broadcasts your in-progress dice selection, operators and target to your opponent in
real time (`js/ui.js:391-394`, `js/multiplayer.js:553-555`, `server/rooms.js:437-448`), rendered
on their board (`js/ui.js:395-451`).

This is clearly deliberate and fits a perfect-information game, but it has competitive
consequences: an opponent sees your candidate arithmetic before you commit, including expressions
you abandon.

**Needs:** confirmation that live preview is intended for rated play.

---

## 17. Is the D20's absence from the first-player comparison intended?

First-player determination compares raw face values across dice of different types
(`js/state.js:38`, `server/game-logic.js:63-66`). Sorted ascending, position 6 is almost always
the D20 or D12 — so the comparison is dominated by which player rolled a low value on their
largest die, and a D4 showing 4 is treated as equal to a D20 showing 4.

This is internally consistent and may well be the original rule, but it is worth confirming that
the comparison is on *values* rather than on normalised values or on die type.

**Needs:** confirmation.
