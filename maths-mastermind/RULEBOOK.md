<!--
locale: en
rules-version: 2026.07.31
last-reviewed: 2026-07-31
document-status: current-implementation
-->

# Maths Mastermind — Rulebook

Maths Mastermind is a solo arithmetic challenge. Each card gives you four numbers and one target.
Build an expression that equals the target exactly.

## 1. The card

A card contains:

- four number slots;
- one target from 0 to 99 on Easy and Medium, or 1 to 99 on Hard; and
- a difficulty: Easy, Medium or Hard.

Repeated values occupy separate slots. Two slots that both show `1/3` are two usable numbers, not
one number printed twice by mistake.

## 2. Building an expression

Use **at least two** of the four number slots. A slot may be used at most once in one answer. You
do not have to use all four slots.

The available operations are addition (`+`), subtraction (`−`), multiplication (`×`) and division
(`÷`). Parentheses may be used to control the order of evaluation.

Standard order of operations applies: multiplication and division before addition and subtraction,
unless parentheses say otherwise. Division by zero is not allowed.

## 3. Exact arithmetic

Answers use exact rational arithmetic. Fractions are not converted to rounded decimals, so `1/3 +
1/3` is exactly `2/3`.

Your final value must equal the target exactly. A complete wrong answer stays open so you can undo,
clear or revise it and try again.

## 4. Difficulty

- **Easy:** whole-number cards and whole-number targets.
- **Medium:** may include one fraction; targets are usually whole numbers.
- **Hard:** includes one or two fractions and excludes a zero target.

Every generated card carries a verified solution. Other valid solutions may exist and are accepted.

## 5. Controls

Select card numbers with the on-screen controls or keys `1`–`4`. Operator keys, parentheses,
Backspace, Escape and Enter work directly. `H` shows a first-step hint and `N` loads a new card.

Revealing the stored solution does not assert that it is the only possible solution.
