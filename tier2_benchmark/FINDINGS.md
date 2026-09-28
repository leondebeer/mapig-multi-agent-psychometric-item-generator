# MAPIG Tier 2 Replication Validation — Findings Report

**Date:** 2026-09-27
**Prepared by:** Leon De Beer (NTNU) with Hermes Agent (automated replication harness)
**Scope:** Tier 2 replication benchmark of MAPIG's pseudo-factor analysis (PFA) and item-generation pipeline against four published instruments.

---

## 1. Summary

MAPIG's PFA **recovers published factor structures with high accuracy** (40/41 items, 97.6%), and its loadings achieve **Tucker congruence of 0.91–1.00** against published loading matrices — reproducing the Varrasi et al. (2026) ">.90" benchmark the paper cites. The full-pipeline facet decomposition is also correct for all four constructs **when the construct is explicitly flagged multi-dimensional**.

Running the pipeline end-to-end surfaced **eight concrete issues** — three of which (factor-order misalignment, reverse-keyed sign handling, overlapping-facet cross-contamination) map directly onto the paper's own Gap list (factor-sign/order indeterminacy, discriminant validity). Full detail in §4.

---

## 2. Method

### 2.1 Instruments

| Instrument | Source | Factors | Items |
|---|---|---|---|
| SWLS — Satisfaction With Life Scale | Diener et al. (1985) | 1 | 5 |
| Grit-S — Short Grit Scale | Duckworth & Quinn (2009) | 2 | 8 |
| UWES-9 — Utrecht Work Engagement Scale | Schaufeli et al. (2006) | 3 | 9 |
| CBI — Copenhagen Burnout Inventory | Kristensen et al. (2005) | 3 | 19 |

Instruments chosen for open-access items + published factor structures, off MAPIG's publisher blocklist, spanning a factor-complexity gradient (1/2/3/3).

### 2.2 Three analyses

1. **Factor recovery** — feed the *published items* into `run_pfa(items, facet_mapping, n_factors)`; score item→factor assignment against the published structure (with factor-order alignment via Hungarian assignment).
2. **Tucker congruence vs published loadings** — compare PFA-recovered loadings to published loading matrices (aligned), per factor.
3. **Full-pipeline generation** — run the complete pipeline on each construct's definition, check facet decomposition + item quality + PFA recovery.

### 2.3 Model routing

- Claude (facet mapper, item writer, reviewers, meta-editor) via OpenRouter.
- GPT-5.4-mini / GPT-5.2 (analytics) + `text-embedding-3-large` (embeddings) via OpenAI.
- Perplexity academic search **not configured** (no key) — degrades gracefully; evidence gathering runs at reduced fidelity.

---

## 3. Results

### 3.1 Factor recovery (published items → PFA)

| Instrument | Recovery | Tucker vs expected (one-hot) |
|---|---|---|
| SWLS | 5/5 (100%) | 0.994 |
| Grit-S | 8/8 (100%) | 0.963 / 0.938 |
| UWES-9 | 9/9 (100%) | 0.926 / 0.945 / 0.974 |
| CBI | 18/19 (94.7%) | 0.973 / 0.786 / 0.954 |
| **Total** | **40/41 (97.6%)** | |

The single miss is CBI item 13 ("Do you have enough energy for family and friends during leisure time?"), a reverse-keyed work-life item that is a documented weak item in the CBI literature.

### 3.2 Tucker congruence vs published loadings

| Instrument | Published source | Per-factor congruence | Mean |
|---|---|---|---|
| SWLS | Diener et al. (1985), Table 1 | 1.00 | 1.00 |
| Grit-S | Duckworth & Quinn (2009), Fig 1 | 0.96 / 0.94 | 0.95 |
| UWES-9 | Sinval et al. (2018), Fig 1 | 0.93 / 0.95 / 0.97 | 0.95 |
| CBI | Fiorilli et al. (2015), Fig 1 | 0.97 / 0.81 / 0.96 | 0.91 |

All instruments ≥ 0.91 mean congruence (Lorenzo-Seva & ten Berge: >.95 near-identity, .85–.94 fair similarity). The only sub-.85 value is CBI's work-related factor (0.81), driven entirely by the reverse-keyed item 13 (see §4.2).

### 3.3 Full-pipeline generation

| Construct | Facets identified | Result |
|---|---|---|
| SWLS (1f) | ✓ correct | 5 items, PFA recovery 1.0 |
| Grit (2f) | ✓ correct (4+4) | recovery 1.0, congruence .995/.996 |
| UWES (3f) | ✓ correct (3+3+3) | recovery 1.0, congruence .98–.99, *force-accepted* |
| Burnout (3f) | ✓ correct (2+4+3) | **PFA recovery 0.0, congruence ~0, max item-cosine .90** |

Facet decomposition is correct for all four. Item-generation quality degrades monotonically with facet overlap: Grit (distinct facets) clean → UWES (moderate) force-accepted → Burnout (semantically-close "exhaustion" facets) fails, with cross-contaminated items and a PFA that cannot separate the factors.

---

## 4. Findings (bugs & limitations)

### 4.1 PFA metrics do not align factor order *(maps to Gap 4: factor-order indeterminacy)*
`factor_recovery_rate` and `tuckers_congruence` score recovered factors against the expected *order*. Oblique rotation returns factors in arbitrary order for 3+ factors, so a correctly-recovered structure can report recovery = 0.0 and congruence ≈ 0. Reproduced on CBI: the three factors were recovered perfectly but in permuted order `{personal→1, work→2, client→0}`, and MAPIG's internal metrics reported `recovery_rate = 0.0`, `tuckers_congruence = [0.04, 0.025, 0.09]`.
**Fix:** align recovered factors to expected (Hungarian assignment on |loadings|, or the existing DAAL labels) before scoring.

### 4.2 Reverse-keyed items recover with flipped sign *(maps to Gap 4: factor-sign indeterminacy)*
CBI item 13 (reverse-keyed) recovered with loading −0.38 on its factor, vs published +0.42 — opposite sign. Sign alignment appears to handle factor-level reflection but not per-item reverse-keying consistently through the PFA.
**Fix:** verify the polarity flip is applied before/after embedding consistently.

### 4.3 `is_unidimensional` defaults to `true` — silent facet collapse
`UserRequest.is_unidimensional` defaults to `True`, and the facet-mapper prompt then outputs exactly one facet regardless of the definition. Grit (2 facets) and UWES (3 facets) both collapsed to a single undifferentiated pool until the flag was set to `false` explicitly. Sub-constructs are merely listed as `flagged_sub_constructs` ("consider a separate run"), never generated.
**Fix:** auto-detect dimensionality from the definition, or warn when the definition names multiple facets but the flag is `true`.

### 4.4 Content-reviewer `issue` field capped at 210 chars — run crash
`ContentReviewResponse.comments[].issue` has `max_length=210`. Claude occasionally exceeds it, raising a Pydantic `ValidationError`; the structured-output fallback then also fails (JSONDecodeError) and the run crashes. Reproduced during the burnout run (`"Facet balance: Personal ...personal burnout items."`).
**Fix:** loosen the cap or truncate/coerce in the fallback parser.

### 4.5 Literal duplicate items pass through
The pipeline emitted the literal duplicate `"I'm satisfied with my job."` twice in an earlier run, plus near-synonym substitution items. MAPIG *flagged* the resulting redundancy (6 flags, pseudo-α too high) but did not deduplicate.
**Fix:** string-similarity dedup before final output.

### 4.6 Discriminant failure for overlapping facets *(matches shipped weakness)*
For semantically-close facets (personal vs work-related burnout), the item writer produces cross-contaminated items — "personal burnout" items still mention "work" (e.g. "I feel worn out from my *work* and daily responsibilities") — and the PFA cannot separate the factors (recovery 0.0). This is the same "internal consistency too high — possible item redundancy" weakness MAPIG's own shipped `eval_results.json` admits.
**Fix:** stronger negative-space / contrastive prompts between adjacent facets.

### 4.7 Broken dependency pins
`requirements.txt` pins `langgraph-checkpoint==3.0.3` (does not exist on PyPI — versions jump 3.0.1 → 4.0.0) and `langchain-core==1.2.8` (too old for `langchain-anthropic>=1.3.4`, which resolves to 1.7.4 requiring `langchain-core>=1.6.4`). Installation fails out of the box.
**Fix:** remove the checkpoint pin (code uses in-memory `MemorySaver`); bump `langchain-core>=1.6.4,<2.0.0`.

### 4.8 Test-suite claim mismatch
The paper cites "~290 unit tests, zero-warning gate," but the published repo contains **no Python tests** and CI explicitly skips pytest when `tests/` is absent (only 7 frontend Vitest tests + Playwright E2E exist).
**Fix:** commit the backend test suite, or soften the claim to "Tier 1 partial."

---

## 5. Methodological notes

1. **Published loading matrices are not uniformly available.** UWES-9 (Schaufeli et al., 2006) and CBI (Kristensen et al., 2005) do **not** publish per-item loading matrices in their original papers (UWES reports only CFA fit indices; CBI's three scales are *a priori* sub-dimensions with only α + item-total ranges). Loading matrices had to be sourced from later validation studies (Sinval et al. 2018; Fiorilli et al. 2015). Consequence for the paper: **"Tucker congruence vs published loadings" is only executable where the source publishes loadings; factor-recovery (item→factor assignment) is the generalizable Tier 2 metric**, with loading-congruence as a secondary check where available.

2. **CBI's Fiorilli et al. (2015) source is an adaptation** — Italian teachers, "client" renamed "student," and two work-related items dropped. This is a caveat to state, not hide.

---

## 6. Reproducibility

All benchmark artifacts live in `tier2_benchmark/` in the repo:

| File | Purpose |
|---|---|
| `instruments.json` | Published items, facets, polarities for the 4 instruments |
| `published_loadings.json` | Published loading matrices (4 sources, cited) |
| `run_tier2.py` | Factor-recovery benchmark (PFA + Hungarian factor alignment) |
| `run_tier2_published.py` | Tucker congruence vs published loadings (NaN-aware) |
| `run_generation.py` | Full-pipeline generation on the 4 construct definitions |

Run with `venv/bin/python tier2_benchmark/<script>.py` (venv at repo root; dependencies installed per the fixed `requirements.txt`).

---

## 7. Recommendations

1. **Fix 4.1, 4.2, 4.4 first** — these are correctness bugs that produce misleading metrics (4.1) or crash runs (4.4) and affect the Tier 2 evidence directly.
2. **Reframe the Tier 2 metric** in the paper around factor-recovery (generalizable) with loading-congruence as a secondary check, and note the 2/4 instrument loading-availability caveat explicitly.
3. **Add the overlapping-facet case (Burnout) as a known boundary condition** — it is a *feature* of the validation that it localizes exactly where the generator is weak, not a reason to hide the result.
4. **Tier 3 (blinded expert ratings)** is the natural next evaluation; the Tier 2 evidence here is sufficient to proceed.
