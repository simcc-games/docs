<!--
locale: en
rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: unresolved-rulings
-->

# Maths Mastermind — Open Questions

The archived implementation supplies complete answer-validation behaviour, but it does not define
a SIMCC tournament format.

## 1. What is the official event scoring format?

Round length, number of cards, difficulty mix, hint or reveal penalties, and tie-breaks are not
specified. The platform must not invent them before an event is configured.

## 2. Are multiple distinct solutions part of competition scoring?

The validator accepts any correct expression, while each generated card stores one known solution.
The current solo mode does not award additional credit for finding alternatives.

## 3. Should generated fractions be restricted further?

Upper Division cards use exact reduced fractions with denominators up to 12. No source establishes whether an
official competition would allow the full generated range.
