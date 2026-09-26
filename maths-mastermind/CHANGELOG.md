<!--
locale: en
rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: current-implementation
-->

# Maths Mastermind — Rules Changelog

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
