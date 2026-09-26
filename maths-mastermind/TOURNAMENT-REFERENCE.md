<!--
locale: en
rules-version: 2026.07.31
event-rules-version: unassigned
last-reviewed: 2026-07-31
document-status: implementation-reference
-->

# Maths Mastermind — Tournament Reference

## 1. Valid answer checklist

A submission is valid when all of these are true:

1. It uses at least two number slots.
2. No slot is used more than once.
3. It uses only `+`, `−`, `×`, `÷` and parentheses.
4. Its syntax is complete and parentheses are balanced.
5. It never divides by zero.
6. Its exact rational value equals the card target.

Repeated printed values are judged by slot identity. If a card contains four `1/3` slots, up to
four copies of `1/3` may be used—one from each slot.

## 2. Evaluation

Apply standard operator precedence. Multiplication and division have equal precedence and associate
left to right; addition and subtraction behave the same at their lower precedence. Parentheses take
priority. Do not round fractions.

## 3. Event format is not yet specified

The software does not define an official round length, number of cards, tie-break, hint penalty or
scoring formula. An event organiser must publish those choices with the event rules version. The
valid-answer checklist above is the current implementation reference, not a complete tournament
format.
