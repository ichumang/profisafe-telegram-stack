# Contributing

This is a **proprietary project** developed under employer NDA.

## Access

Contributions require:

1. A signed Non-Disclosure Agreement (NDA) with the project owner
2. Written approval from the project maintainer
3. Familiarity with IEC 61784-3-3 (PROFIsafe) and IEC 61508 (Functional Safety)

## Code Review

All changes to the safety-critical stack must pass:

- Full unit test suite (158 tests, 100% pass rate)
- Branch coverage ≥ 98%
- Static analysis (no warnings under `-Wall -Wextra -Werror -Wpedantic`)
- Manual review by a certified functional safety engineer (SIL 3 / CFSPE or equivalent)

## Coding Standards

- C99, no dynamic allocation, no recursion
- MISRA C:2012 compliance (Advisory + Required rules)
- All functions must have documented pre/post conditions
- Defensive coding per internal SIL 3 coding standard (SESE, default cases, assert macros)

## Contact

**Umang Panchal** — [GitHub](https://github.com/ichumang) · [github.com/ichumang](https://github.com/ichumang)
