# Team Submission — Bengali Hallucination Detection Pipeline

This repository contains our team's submission notebook for the competition.

## ⚠️ Important: Use the Kaggle Version, Not This GitHub Copy

**The authoritative, working version of this notebook lives on Kaggle, not here on GitHub.**

GitHub does not reliably preserve the dataset links attached to a Kaggle notebook when it's pushed/exported to a repo. As a result, the notebook file in this repo may be missing one or more linked input datasets, and **may fail to run out of the box**.

👉 **For grading, review, or re-running our pipeline, please use the Kaggle notebook directly:**

**[Insert Kaggle notebook link here]**

If you need a local/offline copy for reference, copy the notebook cells over from the Kaggle version rather than relying on the GitHub copy as the source of truth — treat this repo as a mirror/backup, not the primary artifact.

## If You Must Run From This GitHub Copy

If you choose to run the notebook from this repository instead of Kaggle, please note:

1. **Manually verify every dataset link** listed below (and any listed in the notebook's metadata/input section) resolves and is attached before running. Do not assume the links carried over correctly from Kaggle.
2. Re-attach any datasets that failed to transfer, using the exact versions specified.
3. Confirm the runtime environment (GPU/accelerator, package versions) matches what's specified in the notebook's setup cell(s) — some installs are Kaggle-environment-specific and are intentionally *not* run automatically inside pipeline functions (see setup cell notes in the notebook).
4. Run cells top-to-bottom in order; do not skip the setup/install cell.

### Dataset Links Used in This Notebook

*(Fill in / verify this list matches the actual "Input Data" panel on the Kaggle notebook before submitting — do not rely on memory or the repo history.)*

- [ ] Dataset 1 — `<name>` — `<link>`
- [ ] Dataset 2 — `<name>` — `<link>`
- [ ] Dataset 3 — `<name>` — `<link>`
- [ ] ...

## Summary

This notebook implements our team's pipeline for the competition, combining:

- **Phase A** — deterministic/statistical feature engineering and a lightweight classifier
- **Phase B** — a closed-book / LLM-judge stage for deeper reasoning over flagged rows

Please refer to in-notebook markdown cells (on Kaggle) for detailed methodology, ablations, and design decisions.

## Contact

If any dataset link is broken or missing, please reach out to the team before assuming the pipeline is non-functional — it is very likely a GitHub export artifact, not a bug in the pipeline itself.

---

## JD-no-monogatari — Pipeline Overview

A four-stage waterfall for Bangla LLM hallucination detection. Each stage only ever sees the rows the stages before it couldn't already resolve, so compute is spent roughly in proportion to how hard a row actually is.

### Architecture

1. **Oracle stage — answer-key lookup**
   Runs first, before anything else. Fuzzy-matches incoming questions against three scraped answer-key corpora (a BCS exam bank, titulm-bangla-mmlu, and a LiveMCQ job-solutions bank). Where a confident match exists, the verdict is decided by literal answer comparison — an equivalence cascade that handles digit/script normalization, number-set matching with month/calendar-system awareness, and cross-script (Bangla↔English) equivalence — with an LLM adjudicator called only for genuinely ambiguous matches. Resolves a substantial share of the test set before any reasoning model runs.

2. **Phase A — grounded rows** (`context != [NULL]`)
   For rows with a supporting passage: a deterministic feature layer (numeric/date consistency, containment, sentence-relevance scoring) plus a multilingual NLI scorer plus an LLM judge, combined in a regime-routed ensemble with F1-optimal thresholding and high-precision hard-rule overrides. Grammar-classification questions and idiom/synonym/antonym questions with decoy contexts are kicked out here and handled downstream instead, since a grounded passage carries no real signal for those question types.

3. **Phase B — closed-book rows** (`context == [NULL]`)
   For rows with no passage: hybrid dense+sparse retrieval (BGE-M3 + BM25, RRF fusion) over a multi-segment Bangla corpus (Wikipedia, NCTB/TigerLLM textbooks, Bangladesh law), a deterministic routing gate + fine-tuned BanglaBERT classifier (MATH / DERIVABLE / LOOKUP), and a two-stage two-GPU verdict pass — a smaller model handles LOOKUP rows with retrieved evidence and a three-way ENTAILED/CONTRADICTED/SILENT verifier, then a larger model handles MATH/DERIVABLE rows plus re-judges uncertain LOOKUP rows as an escalation step. Idioms and synonym/antonym questions get dedicated dictionary-backed evidence layers with a deterministic exact-match fast path.

4. **Phase G — grammar questions**
   A fully separate deterministic-solver-first pass (sandhi/samas/etc.) with a rule-table-only LLM fallback for anything the solvers can't resolve outright.

**Merge.** The four outputs are concatenated by row id with a hard exclusivity check — every row must come from exactly one stage — before writing the final submission.

### Design principles

- **Evidence before inference.** Every stage prefers checking a claim against something concrete (an answer key, a passage, retrieved text, a dictionary) over asking a model to judge from parametric knowledge alone.
- **Abstain rather than guess.** The three-way SILENT option and hard-rule gating exist specifically so uncertain evidence doesn't get laundered into a confident wrong verdict.
- **Offline-compliant by construction.** No paid APIs; every model loads from local weights; every heavy component runs in an isolated subprocess so the pipeline degrades gracefully (and loudly logs when it does) rather than failing silently.
