# Project Charter: Risk-On / Risk-Off Indicators from a Market Graph

## Goal

Build a research tool that turns a set of daily cross-asset time series into a graph, measures how that graph changes over time, and produces risk-on / risk-off indicators. The indicators will be used as regime filters and features for corporate bond signals.

## Roles

- Andreas owns the decisions. The agent proposes; Andreas chooses. When a choice is his, stop and ask.
- Designs come before code. No code until the implementation plan (Phase 7) is approved.
- Write in plain English. Short sentences. Lead with the answer.

## Non-negotiable rules

1. **No look-ahead.** A value at date *t* uses only data published before *t*'s cut-off time. Every input carries a publication lag. Every fitted object (correlation estimates, shrinkage targets, scalers, regime models, thresholds) is fitted on data ending before *t* and frozen.
2. **Point-in-time data.** Never use revised data as if it were known at the time. Use as-of joins (latest value at or before the cut-off). Never forward-fill a value across its own publication lag.
3. **No invented facts.** Do not invent data fields, vendors, tickers, API calls, paper results or parameter values. If unknown, write "unknown" and add it to `docs/open-questions.md`.
4. **Source tags on every parameter:** **[P]** from a named paper (with section), **[D]** a default we chose, **[Q]** an open question.
5. **Baselines first.** Every indicator is compared with plain baselines: the VIX alone, the HY spread alone, realised volatility, and an equal-weight z-score composite of the same raw inputs. If the graph adds nothing, we say so.
6. **Tests before code.** Every module gets tests that fail without it, including a look-ahead test: truncate inputs at *t*; outputs at *t* must not change.
7. **Determinism.** Same inputs and config give identical outputs. Seeds fixed and logged.
8. **Small steps.** One task per commit. Never mix refactoring with new behaviour.
9. **No proprietary data in git.** Raw and derived data stay out of the repository.

## Decisions so far

| Item | Decision | Phase |
|---|---|---|
| Graph type | **A — market network**: nodes are cross-asset series, edges measure co-movement | 1 (decided by Andreas, 2026-10-07) |
| Target meaning of risk-on/off | proposed in `docs/01-framing.md` | 1 (awaiting approval) |
| Indicator's job | proposed in `docs/01-framing.md` | 1 (awaiting approval) |
| Universe, frequency, cut-off, sample | open — `docs/open-questions.md` | 3 |

## Where things live

| What | Where |
|---|---|
| Phase prompts | `docs/PROMPTS.md` |
| Design documents | `docs/NN-*.md` |
| Open questions | `docs/open-questions.md` |
| Code / tests / config | `src/`, `tests/`, `config/` (from Phase 8) |
