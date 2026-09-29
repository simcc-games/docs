<!--
locale: en
rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: current-implementation
-->

# Maths Mastermind — Rules Changelog

## 2026.09.29 — The UD Team deck

- Added SIMCC's printed UD Team cards as a fixed deck ("UDT cards" on the solo board). The rules
  are unchanged: use at least two of the four numbers, each once.
- Each deck card keeps its printed number, layout (top, right, bottom, left) and answer. The board
  shows the card as printed and turns it over to the answer once it is solved or revealed.
- 19 of the 20 sample cards are in the deck. UDT 5 is left out: its printed answer, (3+5)×8−2,
  uses a 2 that is not on the card (3, 4, 5, 8), and no two or more of those numbers make 62.
- Deck results are checked on the server against the deck, and rated as medium cards.

## 2026.07.31 — Shared-module release

- Ported the archived rational evaluator and seeded card generator to the pure `GameModule` package.
- Preserved the minimum-two-slots rule, exact fractions, standard precedence and retryable wrong
  answers.
- Made generated card IDs seeded and versioned instead of time-dependent.
- Added the solo web board, keyboard controls and rendered rule documents.
- No official tournament scoring or timing format was created.

## Version policy

Published rules use a `YYYY.MM.DD` tag. Event-specific scoring and timing must name the rules
version they extend.
