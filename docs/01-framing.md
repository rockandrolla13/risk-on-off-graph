# Phase 1 — Framing

**Status:** draft for approval · **Date:** 2026-10-07 · **Decided before this phase:** graph type A (market network)

---

## The answer in one paragraph

Build a **market network** each day from rolling, volatility-adjusted co-movement between about 30 cross-asset series. Read risk-off as the network **tightening**: one dominant common factor, a shrinking tree, risky assets moving as one block, and safe havens pulling away from it. Turn a few such graph measures into one **continuous risk-appetite score**, built without any labels. Use it first as a **regime filter for corporate bond signals**, sampled on each signal's formation date. Judge it against plain baselines (VIX, HY spread, realised volatility, a simple z-score composite) and against a list of stress episodes fixed in advance. The design separates **correlation** from **volatility** on purpose, because the main risk is building an expensive copy of the VIX.

---

## Q1. The graph (type A — decided)

### What the nodes are

About 25–40 daily series, one per node, grouped by role. The exact list depends on data availability (Phase 3).

| Group | Examples | Expected role |
|---|---|---|
| Equity | US, Europe, Japan, EM indices; equity sectors (optional) | risky |
| Credit | IG and HY spreads (cash and CDS index) | risky (spread changes enter with sign flipped, so "up" means risk-on) |
| Rates | 2y and 10y government yields, curve slope (US, Germany) | mixed — role changes over time |
| FX | broad USD, JPY, CHF, EM FX basket, commodity currencies | JPY, CHF: haven; EM FX, AUD: risky |
| Commodities | gold, oil, industrial metals | gold: haven; oil, metals: risky |
| Volatility | equity, rates and FX implied vol | see note below |

**Note on volatility series as nodes [D].** If the VIX is a node and also a baseline, the comparison becomes circular. Proposal: build the graph **without** implied-volatility nodes, and keep volatility for the baselines and as a conditioning variable. A second variant with them included is run only as a check.

### What the edges are

An edge is the co-movement between two nodes over a rolling window, measured on **volatility-adjusted** returns: each series' daily change divided by its own trailing volatility. Adjusting first matters, because raw correlations rise in every sell-off simply because volatility rises. Kinlaw & Turkington (2013) show that the correlation part of market "turbulence" can be separated from the volatility part, and that it carries forward-looking information after controlling for volatility.

Options for the edge estimator:

| Estimator | What it captures | Pros | Cons | Proposal |
|---|---|---|---|---|
| Rolling Pearson correlation with Ledoit–Wolf shrinkage | total co-movement | simple, stable with ~30 nodes, well understood (Ledoit & Wolf 2004) | every edge partly reflects the common factor | **v1** |
| Exponentially weighted correlation | same, with recent days weighted more | reacts faster | one more parameter (half-life) | variant |
| Partial correlation via graphical lasso | direct links after removing all others | sparse graph; removes edges that exist only through a common factor (Friedman, Hastie & Tibshirani 2008) | penalty choice; less stable day to day | **v2**, if v1 passes |
| Rank (Spearman/Kendall) correlation | co-movement robust to outliers | fewer single-day distortions | slower to compute | robustness check |

Window: **63 trading days** as the default [D], with 21, 126 and 252 in the robustness grid. Step: daily.

### From a correlation matrix to a graph

| Representation | What it keeps | Source | Proposal |
|---|---|---|---|
| Full weighted graph | every pair | — | used for spectral measures |
| Minimum spanning tree | the strongest N−1 links, via distance $d_{ij}=\sqrt{2(1-\rho_{ij})}$ | Mantegna (1999); Onnela et al. (2003) | used for topology measures |
| Threshold graph | links with \|ρ\| above a cut-off | — | not proposed: results depend on an arbitrary cut-off |
| Topologically constrained (planar) graphs | more links than a tree, still sparse | Aste, Shaw & Di Matteo (2010) | later variant |

### Mixed closing times

Asian markets close before US markets open. A same-day correlation between Japan and the US therefore mixes today in Tokyo with today in New York, which are different information sets. Options [D]:

1. Use **two-day (overlapping) returns** for all pairs. Simple; the timing mismatch averages out.
2. Align everything to one cut-off (e.g. London close) and lag series that close after it by one day.
3. Use weekly returns. Clean, but slow and fewer observations per window.

Proposal: option 1 for v1, option 2 as a check. Final choice in Phase 3, when close times are known.

### What risk-off should look like in the graph (hypotheses to test)

| # | Hypothesis | Graph measure | Basis |
|---|---|---|---|
| H1 | Markets become tightly coupled | **Absorption ratio**: share of total variance explained by the top *k* eigenvectors | Kritzman, Li, Page & Rigobon (2011) define it and argue tightly coupled markets are more fragile |
| H2 | The tree contracts | MST length; mean occupation layer around the most connected node | Onnela et al. (2003): the asset tree shrinks during crashes |
| H3 | Risky assets move as one block and havens move against it | Average correlation inside the risky block minus the risky–haven correlation | [D] hypothesis; haven list to confirm |
| H4 | The graph behaves unusually | Correlation-surprise style distance between today's matrix and its history, with volatility removed | Kinlaw & Turkington (2013) |
| H5 | Clusters merge | Number of communities; size of the largest | [D] hypothesis |
| H6 | The structure changes fast | Distance between consecutive days' matrices or trees | [D] hypothesis |

These are hypotheses. Phase 6 fixes how each will be tested before any result is seen.

### Why A first

A is the simplest graph to compute, explain and test. A lead-lag network (type B) needs more data per estimate and is noisier on daily data; it stays a later extension if A beats the baselines. A computation graph (type C) is an engineering structure we will have anyway, inside the code. A hybrid (type D) is where A naturally goes next, if a regime model on top of the graph features proves useful.

---

## Q2. What "risk-on / risk-off" means as a target

There is no observed label for risk appetite. Options:

| Option | What it is | Risk |
|---|---|---|
| **Unsupervised continuous score** | combine graph measures into one number; no labels used in construction | needs careful evaluation, since nothing tells it what "right" is |
| Latent regime model (e.g. hidden Markov model) on returns and volatility | states estimated from data | fitted states use the whole sample unless filtered; can just rediscover volatility |
| Rule-based labels (e.g. VIX and HY spread percentiles) | simple, transparent | **circular** if the indicator is then judged against the same variables |
| Episode list | known stress periods | small sample; for evaluation only |

**Recommendation.** Build an **unsupervised continuous score**. Use no labels in construction. Evaluate it three ways:
1. against a **pre-registered list of stress episodes** that you fix before any results;
2. against the **baselines**, to show it adds information beyond them;
3. against a **hidden Markov model on the baselines**, using **filtered** state probabilities only (no smoothing, which uses future data), as a reference regime series, never as a training label.

**Avoiding circularity.** The main graph excludes implied-volatility nodes (see Q1). A "graph minus baselines" test regresses the score on the baselines and asks whether the residual still has predictive value.

---

## Q3. The indicator's job

| Job | What it needs | How to judge it |
|---|---|---|
| **Regime filter for bond signals** | a state on each signal's formation date | does conditioning on the state improve the bond signal out of sample? |
| Continuous feature | a score per date | does it add to a bond-return model? |
| Early warning | must lead stress | lead–lag against episodes and baselines |
| Description | coincident is fine | not useful enough on its own |

**Recommendation.** Primary job: **regime filter for corporate bond signals**. Secondary: continuous feature. Computed daily, sampled on each signal's formation date, using only data available at that date. Early warning is tested but not required. Being coincident but **cleaner than the VIX** would already be useful for a filter.

What this implies:
- The final test is **out-of-sample bond-signal performance by regime**, so we need at least one named bond signal to condition (open question Q-5).
- Most bond signals in this work are monthly, so the score's month-end behaviour matters more than daily wiggles.

---

## Scope of version 1

**In:** nodes as above, without implied-vol nodes; volatility-adjusted two-day returns; 63-day shrunk correlation; full matrix and MST; measures H1–H6; one combined score; baselines; evaluation per Phase 6.

**Out (later, only if v1 passes):** graphical lasso (v2); lead-lag edges (type B); learned causal graphs; graph neural networks; intraday data.

---

## Open questions for Andreas

Copied to `docs/open-questions.md`.

1. **Q-1. Nodes.** Which of the series in the node table do you have, and from which source? (Phase 3 needs this.)
2. **Q-2. Cut-off.** What daily cut-off time and time zone should define "today"? London close, New York close, or another?
3. **Q-3. Sample.** What start date? Several credit and FX series begin late, and longer history means more stress episodes to test on.
4. **Q-4. Episodes.** Which stress episodes, with exact start and end dates, should be pre-registered? The list must be fixed before any results. Widely studied candidates include the 2008 financial crisis, the 2011 euro-area crisis, the 2013 taper tantrum, the August 2015 renminbi devaluation, Q4 2018, March 2020 and the 2022 rate shock. You choose the list and the dates.
5. **Q-5. Bond signal.** Which corporate bond signal(s) should the filter be tested on first?
6. **Q-6. Havens.** Confirm the haven set for H3: Treasuries, Bunds, gold, JPY, CHF?
7. **Q-7. Approve or change** the three recommendations above: unsupervised score, regime-filter job, implied-vol nodes excluded from the main graph.

---

## References (verified via OpenAlex, 2026-10-07)

- Aste, T., Shaw, W. T., & Di Matteo, T. (2010). Correlation structure and dynamics in volatile markets. *New Journal of Physics, 12*(8), 085009. https://doi.org/10.1088/1367-2630/12/8/085009
- Friedman, J. H., Hastie, T., & Tibshirani, R. (2008). Sparse inverse covariance estimation with the graphical lasso. *Biostatistics*. https://doi.org/10.1093/biostatistics/kxm045
- Kinlaw, W., & Turkington, D. (2013). Correlation surprise. *Journal of Asset Management*. https://doi.org/10.1057/jam.2013.27
- Kritzman, M., & Li, Y. (2010). Skulls, financial turbulence, and risk management. *Financial Analysts Journal, 66*(5), 30–. https://doi.org/10.2469/faj.v66.n5.3
- Kritzman, M., Li, Y., Page, S., & Rigobon, R. (2011). Principal components as a measure of systemic risk. *Journal of Portfolio Management, 37*(4), 112–. https://doi.org/10.3905/jpm.2011.37.4.112
- Ledoit, O., & Wolf, M. (2004). A well-conditioned estimator for large-dimensional covariance matrices. *Journal of Multivariate Analysis*. https://doi.org/10.1016/s0047-259x(03)00096-4
- Mantegna, R. N. (1999). Hierarchical structure in financial markets. *European Physical Journal B, 11*(1), 193–. https://doi.org/10.1007/s100510050929
- Onnela, J.-P., Chakraborti, A., Kaski, K., & Kertész, J. (2003). Dynamics of market correlations: Taxonomy and portfolio analysis. *Physical Review E, 68*(5), 056110. https://doi.org/10.1103/physreve.68.056110

Not used in v1, for later phases: Billio, Getmansky, Lo & Pelizzon (2012), *JFE*, https://doi.org/10.1016/j.jfineco.2011.12.010 (Granger-causality networks); Diebold & Yilmaz (2014), *Journal of Econometrics 182*(1), https://doi.org/10.1016/j.jeconom.2014.04.012 (variance-decomposition connectedness). Both belong to graph type B.

Content claims above come from each paper's abstract. Methods details will be read from full texts in Phase 5.
