# Prompt Set: Building a Graph for Risk-On / Risk-Off Indicators

A sequence of prompts for directing an AI coding agent (Claude Code or similar) through the project, from first framing to a tested, reviewed indicator. Written 2026-10-07.

---

## How to use this

1. **Paste the Project Charter (Prompt 0) at the start of every session**, or save it as the project's `CLAUDE.md`. It carries the rules the agent must never forget.
2. **Run the phases in order.** Each phase ends with a document. That document is the input to the next phase. Do not let the agent skip ahead to code.
3. **One phase per session where possible.** Start the next session with: the charter, plus the previous phase's output file.
4. **You approve every gate.** Each phase prompt ends with "stop and wait for my approval". Read the output before saying go.
5. **Fill the `{{placeholders}}`** before sending. If you don't know an answer, write "unknown — ask me" and the agent will list it as an open question.
6. **Steering prompts (Section S)** are for mid-session use: when the agent drifts, guesses, or over-builds.

Phase map:

| Phase | Prompt | Output file | Gate |
|---|---|---|---|
| 0 | Project charter | `CLAUDE.md` | — |
| 1 | Frame the problem: what is the graph, what is "risk-on" | `docs/01-framing.md` | You pick the graph type and target |
| 2 | Stress-test the framing | `docs/02-challenge.md` | You accept or revise |
| 3 | Data contract | `docs/03-data-contract.md` | You confirm what data exists |
| 4 | Architecture | `docs/04-architecture.md` | You approve the modules |
| 5 | Algorithm cards | `docs/05-algorithms.md` | You approve each estimator |
| 6 | Evaluation protocol | `docs/06-evaluation.md` | You approve the tests *before* any result exists |
| 7 | Implementation plan | `docs/07-plan.md` | You approve the task list |
| 8 | Build, task by task | code + tests | Each task's tests pass |
| 9 | Validation run | `docs/09-validation.md` | You read the results |
| 10 | Reviews | `docs/10-review.md` | Fix or accept findings |
| 11 | Handover | `README.md`, `docs/11-handover.md` | Done |

---

## Prompt 0 — Project Charter (paste every session, or save as CLAUDE.md)

```text
PROJECT: Risk-on / risk-off indicators built from a graph of markets.

GOAL
Build a research tool that turns a set of market time series into a graph, measures
how that graph changes over time, and produces daily risk-on / risk-off indicators.
The indicators will be used as features and regime filters for corporate bond signals.

MY ROLE AND YOURS
- I own the decisions. You propose, I choose. When a choice is mine, stop and ask.
- Write designs before code. No code until I approve the plan (Phase 7).
- Explain things in plain English. Short sentences. Lead with the answer.

NON-NEGOTIABLE RULES
1. No look-ahead. A value at date t uses only data published before t's cut-off time.
   Every input carries a publication lag. Every fitted object (graph weights, scalers,
   regime models, thresholds) is fitted on data ending before t and then frozen.
2. Point-in-time data. Never use revised data as if it were known at the time.
   Use as-of joins (latest value at or before the cut-off), never forward-fill across a lag.
3. No invented facts. Do not invent data fields, vendor names, tickers, API calls,
   paper results or parameter values. If you don't know, say "unknown" and add it to
   docs/open-questions.md.
4. Every parameter has a source tag: [P] from a named paper (with section),
   [D] a default we chose, [Q] an open question.
5. Simple baselines first. Every indicator is compared with a plain baseline
   (e.g. the VIX alone, or an equal-weight z-score composite of the same inputs).
   If the graph adds nothing over the baseline, we say so.
6. Tests before code. Every module gets tests that would fail without it, including
   a look-ahead test (truncate inputs at t; outputs at t must not change).
7. Determinism. Same inputs and config give identical outputs. Seeds are fixed and logged.
8. Small steps. One task per commit. Never mix refactoring with new behaviour.

DEFINITIONS (fill in as phases settle them)
- Universe of nodes: {{e.g. ~30 cross-asset series: equity indices, rates, credit spreads, FX, commodities, vol}}
- Frequency: {{daily close, with a fixed cut-off time and time zone}}
- Sample: {{start date – end date}}
- Graph type: {{decided in Phase 1}}
- Target meaning of "risk-on": {{decided in Phase 1}}

WHERE THINGS LIVE
- Designs: docs/   Code: src/   Tests: tests/   Config: config/   Open questions: docs/open-questions.md
```

---

## Phase 1 — Frame the problem

```text
Read the charter. We are at Phase 1. Write no code.

Write docs/01-framing.md answering three questions. For each, give 2–4 options,
the trade-offs in plain English, and your recommendation with the reason.

Q1. WHAT IS THE GRAPH?
Consider at least these options:
  A. Market network: nodes are assets or factors; edges measure co-movement
     (e.g. rolling correlation, partial correlation, graphical lasso).
     Risk-off shows up as changes in the graph's shape: higher density, one dominant
     cluster, rising centrality of safe havens.
  B. Lead-lag / causal network: directed edges for "X helps predict Y"
     (e.g. Granger tests, VAR, transfer entropy, a learned DAG from time series).
     Risk-off shows up as changes in who leads whom.
  C. Indicator dependency graph (a computation DAG): nodes are raw inputs and
     derived indicators; edges are transformations. This is an engineering structure,
     not a market model.
  D. Hybrid: build A or B, extract graph features, feed them to a regime model.
For each, say what risk-on/off would look like in the graph, what data it needs,
how noisy it is with daily data, and how hard it is to test.

Q2. WHAT DOES "RISK-ON / RISK-OFF" MEAN AS A TARGET?
There is no observed label. Options include:
  - a continuous score with no labels (unsupervised);
  - labels from a latent regime model (e.g. hidden Markov model on returns/vols);
  - labels from simple rules on observable variables (e.g. VIX and credit spread
    percentiles);
  - an event list of known stress episodes, used only for evaluation.
Say which you recommend and how we avoid circularity (building the indicator from
the same variables used to label it).

Q3. WHAT IS THE INDICATOR'S JOB?
Options: a regime filter for bond signals (on/off), a continuous feature, an early
warning (leads stress), or a description (coincident). The answer changes how we
evaluate it. Recommend one primary job.

Background you may use (verify before relying on it):
- A sell-side EM regime indicator classified risk-seeking / neutral / risk-averse
  regimes from term premia, a funding-stress spread, IG and HY credit spreads,
  rates volatility, FX volatility and the VIX.
- Hidden Markov models are commonly used to detect latent risk-on/off states.
- Methods exist to learn DAGs from time series with instantaneous and lagged effects
  (structural VAR based).

End with: a one-paragraph recommended design, and a list of open questions for me.
Stop and wait for my approval.
```

---

## Phase 2 — Stress-test the framing

```text
Phase 2. Write no code. Act as a sceptical reviewer of docs/01-framing.md and the
choices I approved: {{paste my decisions}}.

Write docs/02-challenge.md:
1. The five strongest reasons this design could fail or mislead. For each: how we
   would detect it, and what we would change.
   Must cover:
   - "It's just the VIX": the graph indicator is a noisy copy of one input.
   - Non-stationarity: correlations rise in every sell-off, so "density up" may only
     restate realised volatility.
   - Window choice: results that exist for one window length only.
   - Circular evaluation: labels and indicator built from the same series.
   - Asynchronous closes: markets in different time zones make same-day
     correlations wrong (e.g. Asia closes before the US opens).
2. Simpler alternatives that might do the same job, and what evidence would make
   us drop the graph and use them instead.
3. A kill criterion: a test result, written now, that would make us stop.
Stop and wait for my approval.
```

---

## Phase 3 — Data contract

```text
Phase 3. Write no code. Do not search my machine for data; ask me.

Write docs/03-data-contract.md with one row per input series:
| node id | description | asset class | source/table | field | frequency |
| close time + time zone | publication lag | history start | point-in-time? |
| known breaks (e.g. benchmark discontinued, methodology change) | status |

Candidate inputs to propose (mark each "available?" for me to confirm):
equity indices (US, Europe, Japan, EM), equity implied vol, rates (2y/10y, curve slope),
rates implied vol, IG and HY credit spreads (cash and CDS index), FX (USD, JPY, CHF,
EM FX basket), FX implied vol, gold, oil, funding/liquidity spreads, term premium
estimates, bond-equity correlation.

Rules:
- Where a benchmark was discontinued (for example LIBOR-based spreads), propose a
  replacement and mark the splice date.
- State the daily cut-off time we will use, and for each series whether its close is
  before or after it. Flag every series whose close falls after the cut-off.
- List missing-data rules: holidays, half days, stale prints. Never forward-fill a
  value across its own publication lag.

End with a short list of questions for me: what exists, where, and how it is keyed.
Stop and wait for my answers.
```

---

## Phase 4 — Architecture

```text
Phase 4. Write no code. Using docs/01–03, write docs/04-architecture.md.

1. Modules, one line each on what they own and what they must not know about.
   Expected shape (change it if you have a reason):
     ingest      -> raw series, as-of aligned to the daily cut-off
     transform   -> returns, changes, standardisation (fitted on past only)
     graph       -> node set + edge estimator + window -> graph per date
     features    -> graph statistics per date (density, centrality, clustering,
                    spectral measures, community structure)
     indicator   -> combine features into risk-on/off score(s)
     regime      -> optional latent-state model or labelling rule
     evaluate    -> tests from docs/06 (written later)
     report      -> tables and charts
2. The data flow as a diagram (text is fine). It must be a DAG: no module imports
   one downstream of it.
3. Interfaces: the input and output of each module, with types and the date index
   convention. One object per date or one panel? Say which and why.
4. Config: what is configurable (universe, windows, estimator, thresholds) and how a
   run records its full config and data snapshot ids.
5. Storage: cache graphs per date? Size estimate for {{N nodes}} × {{years}} daily.
6. Where look-ahead could leak in each module, and the guard for each.
Keep it small. Prefer functions over classes unless state is needed.
Stop and wait for my approval.
```

---

## Phase 5 — Algorithm cards

```text
Phase 5. Write no code. Write docs/05-algorithms.md: one card per algorithm we
approved. Each card has exactly these fields:

  Name / What it measures (one sentence) / Inputs / Window and step /
  Formula (LaTeX) / Parameters with source tags [P]/[D]/[Q] /
  Output and its sign convention (what "more risk-off" looks like) /
  Fitting: what is fitted, on what window, how often refitted /
  Look-ahead guard / Failure modes / A synthetic test that should recover a known answer.

Cards to write (drop any we rejected, add any we chose):
- Edge estimators: rolling correlation (with shrinkage), partial correlation /
  graphical lasso, lead-lag (Granger or VAR), and any causal-graph learner we chose.
- Graph features: edge density above a threshold, average absolute correlation,
  eigenvector centrality of safe-haven nodes, largest-eigenvalue share (absorption
  ratio style), number and size of communities, graph distance between consecutive
  dates (how fast the graph is changing).
- Indicator construction: how features become one score (z-score average, first
  principal component, regime model), and how thresholds become on/off states.
- Regime model, if used: states, emissions, fitting window, how we avoid using
  smoothed (future-informed) state probabilities — use filtered probabilities only.

Do not quote numbers from papers unless I give you the paper; write [Q] instead.
Stop and wait for my approval.
```

---

## Phase 6 — Evaluation protocol (before any results)

```text
Phase 6. Write no code. We fix the tests BEFORE we see any result, so we cannot
tune to them. Write docs/06-evaluation.md:

1. Baselines every indicator is compared with: VIX level, HY spread level, an
   equal-weight z-score composite of the same raw inputs, realised volatility.
2. Questions and tests:
   a. Does it add information beyond the baselines? (regressions / horse races,
      incremental R², correlation with each baseline)
   b. Timing: does it lead, coincide with, or lag stress? (cross-correlations at
      leads/lags; event studies around a pre-registered list of episodes)
   c. Usefulness for its job (from Phase 1): e.g. as a regime filter, does it change
      the risk-adjusted performance of {{named bond signal}} out of sample?
   d. Stability: same conclusion across window lengths, node sets, estimators and
      sub-periods? Report the full grid, not the best cell.
3. Data split: training / validation / untouched test period. The test period is
   opened once, at the end.
4. Multiple testing: how many variants we try, and how we correct for it.
5. The pre-registered episode list for event studies: {{dates I give you}}.
   No episodes may be added after seeing results.
6. The kill criterion from Phase 2, restated as a concrete test.
Stop and wait for my approval.
```

---

## Phase 7 — Implementation plan

```text
Phase 7. Write no code yet. Turn docs/04–06 into docs/07-plan.md:

- An ordered task list. Each task: id, goal, files it creates or changes, tests it
  adds, acceptance check, and which tasks it depends on.
- Tasks are small (one sitting each) and touch disjoint files where possible, so
  independent tasks can run in parallel.
- First tasks build the skeleton and the look-ahead test harness, before any estimator.
- Include synthetic-data fixtures: (a) a planted two-regime market where correlations
  jump on known dates; (b) a planted lead-lag pair; (c) pure noise. Every estimator
  must recover (a) and (b) and find nothing in (c).
- Mark which tasks need my data and cannot start until docs/03 is confirmed.
Stop and wait for my approval.
```

*(If you use the `orchestrate` tool for parallel builds, ask the agent to also write the plan in its conductor format and run `orchestrate conductor check` before anything executes.)*

---

## Phase 8 — Build one task (repeat per task)

```text
Phase 8. Implement task {{task id}} from docs/07-plan.md, and only that task.

Steps:
1. Restate the task's goal and acceptance check in two lines.
2. Write the tests first. Run them and show they fail for the right reason.
3. Write the smallest code that makes them pass. Match the style of the codebase.
4. Run the full test suite, including the look-ahead test.
5. Report: files changed, tests added, test results (paste the summary line),
   and anything you assumed. Add new unknowns to docs/open-questions.md.
6. Commit with a message that says what changed and why.

Do not start the next task. Do not refactor unrelated code. If the task turns out
bigger than planned, stop and propose a split.
```

---

## Phase 9 — Validation run

```text
Phase 9. Run the evaluation from docs/06-evaluation.md exactly as written, on the
training and validation periods only. Do not open the test period.

Write docs/09-validation.md:
1. One-paragraph answer first: does the graph indicator add anything over the
   baselines? Yes / no / unclear, and why.
2. Results for each test in docs/06, with the full grid of variants (not the best one).
3. Where it fails, say so plainly.
4. Any deviation from docs/06 and why it was unavoidable.
5. Charts: indicator vs baselines over time with the pre-registered episodes marked.
Then stop. I decide whether we open the test period.
```

---

## Phase 10 — Reviews

**10a. Code review**
```text
Review the code changed since {{commit/branch}}. Look for: look-ahead leaks
(fitting on full sample, forward-fills across lags, same-day use of late closes),
silent NaN handling, non-determinism, untested branches, and code that does not match
docs/05. List findings most-severe first with file:line and a failing scenario.
Do not fix anything yet.
```

**10b. Adversarial review of the indicator**
```text
Argue against the indicator in docs/09. Try to show it is (1) a relabelled VIX or
volatility, (2) an artefact of one window or one node, (3) driven by a few episodes,
(4) unusable in real time because of publication lags. For each, run or propose the
test that would show it, and say what the result means.
```

**10c. Architecture review**
```text
Review the structure against docs/04. Are module boundaries still clean? Does any
module know about something downstream? What will be painful to change when we add
new nodes or a new edge estimator? Recommend at most three changes, ranked.
```

---

## Phase 11 — Handover

```text
Write README.md and docs/11-handover.md:
- What the indicator is, in five plain sentences.
- How to run it end to end (one command), how to add a node, how to add an estimator.
- The config that produced the reported results, with data snapshot ids.
- Known limitations and open questions.
- What we tried that did not work, and why (so nobody repeats it).
```

---

## Section S — Steering prompts (use any time)

| Situation | Prompt |
|---|---|
| Agent starts coding too early | `Stop. We are in Phase {{n}}; no code yet. Put what you were about to build into the design document as a proposal.` |
| Agent guesses a fact | `You stated {{X}}. Where did it come from? If it isn't from my data, a file, or a paper I gave you, mark it [Q] and add it to open questions.` |
| Scope creep | `That's outside the current task. Add it to docs/ideas.md and continue with {{task}} only.` |
| Too much output | `Give me the answer in three sentences, then the one decision you need from me.` |
| Suspected look-ahead | `Prove this has no look-ahead: truncate all inputs at {{date}}, rerun, and show the outputs up to {{date}} are identical.` |
| Result looks too good | `Before we believe this: show the same result with the next two window lengths, without the top 3 episodes, and against the VIX baseline.` |
| Lost context in a new session | `Read CLAUDE.md, docs/07-plan.md and docs/open-questions.md. Tell me in five lines where we are, what's done, and the next task.` |
| Agent wants a big refactor | `List the refactor as a separate task in docs/07-plan.md with its own tests. Finish the current task first.` |
| Unclear failure | `Don't patch yet. State the symptom, three hypotheses, and the smallest test that separates them.` |

---

## Defaults worth starting from

These are suggestions, tagged [D], to be confirmed in the relevant phase.

- **Graph type:** start with **A (market network)** using shrunk rolling correlation on daily returns, then add **B (lead-lag)** only if A beats the baselines. A is easier to test and explain. [D]
- **Asynchronous closes:** use returns over two days or weekly returns for cross-time-zone pairs, or align everything to one cut-off and lag late closes by one day. [D]
- **First features:** average absolute correlation, largest-eigenvalue share, safe-haven centrality (gold, JPY, CHF, Treasuries), and day-to-day graph change. [D]
- **Target:** an unsupervised continuous score, evaluated against a pre-registered episode list and a hidden Markov model on the baselines, using **filtered** probabilities only. [D]
- **Kill criterion:** if the graph score adds no out-of-sample information beyond an equal-weight z-score of the same inputs, drop the graph. [D]

Related material in this wiki: `concepts/hidden-markov-models`, `concepts/regime-switching-models`, `concepts/graph-signal-processing`, `concepts/structural-vector-autoregression`, `sources/misiakos-2025-dag-tfrc` (DAGs from time series), `sources/ms-2018-07-09-em-risk-indicator-regime-switching` (a regime indicator's input set), `analyses/credit-universe-topology-and-representation` (graph and sheaf representation of a credit universe).
