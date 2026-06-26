# Provenika — honest review *(how far it has actually come, and where it falls short)*

This repository is a **candid, evidence-grounded accounting** of [Provenika](https://provenika.com) —
an "auditable oncology evidence engine" that turns public databases into ranked, cited, re-verifiable
drug-discovery hypotheses. The Provenika codebase itself is **private**; this public repo exists so the
project's real state — what genuinely works, who could use it today, and exactly where it is wrong or
overstated — is on the record rather than buried under a confident README.

Every claim below is traced to a specific file and line in the private codebase and was produced by a
multi-agent audit whose findings were **adversarially re-verified** (37 confirmed, 5 partly-confirmed,
0 refuted). The raw, per-finding evidence is in [`audit-evidence/`](audit-evidence/); the consolidated,
prioritized list of problems is in [**`KNOWN-LIMITATIONS.md`**](KNOWN-LIMITATIONS.md).

> **The one-line verdict.** Provenika is a *real, honestly-scoped, keyless research-triage front-end*
> for the cheap in-silico start of oncology drug discovery — and a genuinely useful one for a
> med-chemist or computational researcher who treats it as a cited starting point. It is **not** a
> validated CADD pipeline: several of its scientific figures are computed in ways that do not match the
> specifications they invoke, and its headline "auditable / fabrication-impossible" framing is stronger
> than the code delivers. **It has no clinical use and no real users yet.**

---

## ⚠️ Not for patient care

Provenika does not diagnose, treat, or advise on any patient, and neither does this review. Nothing here
is medical advice. A computational hit is a hypothesis for the wet lab, never evidence a therapy works.

---

## How far it has actually come — what genuinely works

These parts are real, run on live public data, and would reproduce in a standard environment. Credit
where it is due:

- **Keyless OSINT one-liners (the strongest part).** Six commands and a self-test dispatch straight to
  live public APIs with **no API key** — PubMed, ClinicalTrials.gov, cBioPortal, ChEMBL, UniProt, KEGG,
  Open Targets — and print cited, re-verifiable results. The `--self-test` honestly probes 8 live
  endpoints. For fast literature/target/trial triage this is legitimately handy.
- **Real RDKit cheminformatics.** PAINS/Brenk via the real RDKit `FilterCatalog`, real Bickerton **QED**,
  real Ertl–Schuffenhauer **SA score**, standard descriptors, ECFP4 Tanimoto, and Butina clustering — a
  thin, faithful wrapper, not a fake (verified: aspirin SA≈1.58, QED≈0.550, genuine catalog alert names).
- **Cited ligand shortlists.** A 25-compound, SMILES + ChEMBL-linked shortlist (CSV/SDF) you can hand
  straight to real docking/ADMET tools — a real time-saver *if* treated as an unvalidated triage start.
- **Structure + docking-box bootstrap.** Fetches a real experimental PDB (or AlphaFold model) and computes
  a Vina-ready box, so a real docking run can begin. (The box selection was recently hardened — see below.)
- **A real provenance/verify discipline.** `verify.py` re-fetches ChEMBL/UniProt counts, does an
  independent raw-HTTP SMILES byte-equality check against ChEMBL, and deterministically recomputes several
  artifacts, failing CI on mismatch. As a *tamper-detector and reproducibility gate* this is more than
  most pipelines offer.
- **Honest by construction, mostly.** The in-code disclaimers are candid, a CI "medical-safety" audit
  fails the build on any dosing/treatment/prognosis output, and the pitch docs leave traction as explicit
  `[fill in]` placeholders rather than inventing it. The project does **not** fabricate adoption.

**Who it helps today:** a med-chemist or computational/oncology *researcher* who wants a fast, cited,
keyless first pass at "is this target worth pursuing, and what chemotypes have measured activity?" —
and who will sanity-check every number. **Who it does not help:** a practicing oncologist or anyone
making a patient-facing decision. That is out of scope by design, and the repo says so.

---

## Where it falls short — the downfalls

Full detail, with `file:line` evidence and severity, is in
[**`KNOWN-LIMITATIONS.md`**](KNOWN-LIMITATIONS.md). The headline problems:

### Inaccurate CAD data — not matched to the specifications it invokes
- **"Potent activities = N" applies no potency threshold** and counts *records, not molecules* — the word
  "potent" is unearned (`cad/target_report.py:61`).
- **The "most potent" ligand ranking scans an arbitrary, unordered first slice** (~4000 records, truncated
  by a wall-clock budget), **pools IC50/Ki/Kd/EC50**, and keeps the single most-favorable value per
  molecule — so it is best-of-an-arbitrary-slice, not a global potency ranking (`cad/virtual_triage.py`).
- **The docking box was centered on the largest hetero group** — which can be a cofactor/glycan/peptide,
  not the inhibitor — and its center (atom mean) and size (min/max extent) use inconsistent reference
  points (`cad/binding_site.py`). *(Selection is now fixed — see "Status" below.)*
- **"Docking feasible: yes" is inferred from the mere existence of any PDB** — apo, mutant, wrong-domain,
  or NMR all count (`cad/target_report.py:174`, `cad/run_pipeline.py:175`).
- **Structure selection optimizes resolution only**, ignoring apo/holo, domain coverage, and mutations;
  the AlphaFold fallback grabs only the **F1 fragment** and never reads the **pLDDT** confidence it tells
  you to check (`cad/fetch_structure.py`).
- **Target resolution differs across stages** — three independent fuzzy lookups can describe *different
  proteins* for one input, with no cross-check (`target_report.py`, `virtual_triage.py`, `fetch_structure.py`).
- **The docking wrapper's blind-box fallback passes `--autobox`**, a flag stock AutoDock Vina does not
  support — that path errors instead of docking (`cad/dock.py:163`).

### The "feasibility verdict" is a generic benchmark dressed as a prediction
- `probability_of_approval` is a **static phase × modality × oncology lookup with zero target- or
  molecule-specific signal** — identical for every program at the same phase. The preclinical prior is an
  invented number mis-attributed to BIO; "revenue" is actually gross profit; there is **no time
  discounting**; benefit is risk-weighted but cost is not; and the go/no-go thresholds are unsourced round
  numbers (`cad/cost_benefit.py`).

### The "auditable / fabrication-impossible" framing overstates the code
- "Every figure re-pulled from source" is not true: **most numeric columns in the ligand shortlist are
  never re-fetched** — the score is recomputed from the same unverified row. **DRIFT silently blesses**
  changed counts within a tolerance and still prints "No fabrications." Only the **top 5 hits / 25
  liabilities** are checked. `provenance.py` only validates the *label string* `fetched`/`computed`, not
  that the value truly came from the source, and `verify.py` **never reads `provenance.json`** (`cad/verify.py`).

### Cheminformatics fidelity caveats
- The `clean` flag **ignores Brenk alerts** (175 served compounds are `clean:true` yet carry Brenk
  alerts); **Veber** omits the HBD+HBA≤12 clause; **Egan** is a rectangular approximation of an
  ellipsoidal model; PAINS/Brenk name arrays are truncated to 5 while the count is full; and the served
  `web/public/data/cheminformatics/*.json` is a **non-reproducible 2026-06-23 ChEMBL snapshot** that a
  "never-shrink" guard can leave stale while the index date advances (`cad/cheminformatics.py`, `precompute_site_data.py`).

### Scope / framing overclaims
- **"63 OSINT tools"** is a literally-accurate count, but only ~21 are live keyless lookups; ~13 silently
  need a paid Tavily key and ~25 are static curated datasets or template scaffolds (`src/tools/cancer/`).
- **No real adoption.** The "outreach" system is investor email automation, not clinician outreach, and it
  is not live (`status.json`: `overall: fail`, `send_enabled: false`, `sent: 0`; all contacts are VCs in
  "drafted"). Utility is entirely prospective — **nobody is using this yet.**

---

## Status — what is being fixed

The private codebase is under active remediation. As of this review:

- ✅ **Docking-box ligand selection** — now picks the **buried, drug-like** ligand (protein-contact +
  heavy-atom-size scoring) instead of the largest hetero group, surfaces the runners-up, and warns when
  the pick is uncertain; the pipeline/verifier share one selection function so a saved box still
  re-verifies exactly. *(Tested, committed.)*
- ✅ **Docking blind-box no longer passes `--autobox`** — stock AutoDock Vina has no auto-box mode, so the
  no-`--center` path now builds a real whole-receptor box and passes explicit `--center/--size`, instead of
  erroring out. *(Tested, committed.)*
- ✅ **Structure fetch now handles mmCIF** — many modern/large PDB entries have no legacy `.pdb` file, so the
  docking-box stage used to silently 404 on them; it now falls back to `.cif` and parses the mmCIF atom loop.
  *(Tested, committed.)*
- ✅ **A validation *harness* now exists** (`cad/validate.py`) — redocking pose-RMSD (≤2 Å = correct) plus
  retrospective enrichment (ROC AUC / EF), so the pipeline's accuracy can be **measured**, not asserted. The
  metric math is offline-tested in CI. **Important honesty caveat:** a harness is not a result — the pipeline
  remains **unvalidated** until those benchmarks are actually run (the docking step needs Vina/Open Babel).
  Building the measurement is the honest first step toward "validated," nothing more.
- ✅ **Structure selection is now coverage-aware** — `pick_best_pdb` prefers structures that span a
  substantial fraction of the protein over tiny high-resolution domain fragments, reads the **AlphaFold pLDDT**
  confidence it used to ignore, and flags that only fragment F1 is fetched + that apo/holo and mutations are
  unchecked. *(Tested.)*
- ✅ **The cost-benefit readout no longer overclaims** — it now states P(approval) is a phase×modality
  benchmark identical for every program (no target signal), relabels gross profit (was "revenue"), flags the
  undiscounted/asymmetric ratio and the unsourced verdict bands, and corrects the BIO/Wong/DiMasi attribution.
  *(Labels/attribution only — the JSON is unchanged; the field-name rename is deferred.)*
- ✅ **Cross-stage target resolution + dossier labels** — the target dossier now applies the same exact
  gene-symbol guard as the other stages (so "AKT1" no longer resolves to AKT1S1); "potent activities" is
  relabelled an honest "ChEMBL bioactivity records (any pChEMBL, not potency-filtered, records not molecules)"
  count; and "docking feasible: yes" becomes "structures exist — confirm one is holo and covers the site
  (not auto-checked)". *(Tested.)*
- ✅ **The verifier and README stop overclaiming** — the `verify.py` banner/headline and the README no longer
  say "every figure re-pulled" / "fabrication impossible to hide". They now state exactly what's re-checked
  (dossier counts + top-5 SMILES live, deterministic artifacts recomputed), that DRIFT is symmetric and exits
  0, and that potency/QED columns and `provenance.json` are **not** independently re-verified (a self-consistent
  fabrication can pass). "Impossible to hide" → "a fabricated number is easy to catch".
- ✅ **Docking box is now geometrically consistent** — centered on the bounding-box midpoint (was the atom
  mean, a different reference than the size), so the box is symmetric about the ligand.
- ✅ **Cheminformatics rule fidelity** — Veber now includes the omitted HBD+HBA≤12 clause; `clean` now
  requires zero **Brenk** alerts (175 served compounds were "clean" while carrying Brenk alerts); PAINS/Brenk
  name arrays are no longer truncated to 5; Egan is documented as a rectangular approximation.
- ✅ **Ligand triage is no longer an arbitrary slice** — the ChEMBL scan is `order_by=-pchembl_value`, so a
  bounded/budget-truncated scan keeps the genuinely most-potent records; the assay-type pooling is documented.
- ✅ **Honest "63 tools" framing + staleness surfacing** — the README states only ~21 of the 63 tools are
  keyless live lookups; the precompute index stamps each target's own refresh date so a stale `/explore` file
  is visible rather than masked by the advancing index date.
- ✅ **Cost-benefit JSON key renamed** — `risk_adjusted_revenue_musd` → `risk_adjusted_gross_profit_musd`, every
  reader (verifier, SUMMARY) updated in lockstep; the verifier gate stays green. The committed `/explore` data
  has been **regenerated** so all 38 targets match the fixed cheminformatics rules and potency-ordered triage.
- ⏳ **In progress:** **holo-aware structure selection** — the coverage-aware pick can land on an *apo*
  structure (e.g. EGFR now resolves to apo 4UV7), so a preference for a ligand-bound structure on the docking
  path is being added; and a **deeper verifier pass** (re-pulling more hit columns + cross-checking
  `provenance.json`).

**Nearly every concrete downfall this review documents has now been fixed** — see the commit history of the
(private) codebase. What remains is either actively in progress (above) or inherent: a triage tool cannot
confirm a structure's apo/holo or mutation state, or that a molecule works, without lab work. The honest
headline is unchanged: useful **research triage**, still **not a validated CADD pipeline** and **never a
clinical tool**.

## On "validation" and "clinical use"

Two questions worth answering plainly, because they are easy to conflate:

- **"Is it validated?"** Validation is *measured evidence*, not a badge. The new harness lets you produce
  that evidence (redocking RMSD, enrichment); until those benchmarks are run and reported, the honest answer
  stays **no — it is unvalidated**. Reproducing known answers would earn cautious trust in *triage*, never
  proof of a prospective or clinical result.
- **"Can it be used clinically?"** **No, and it should not be.** Software that informs diagnosis, prognosis,
  or treatment is a regulated medical device (FDA SaMD / EU MDR) requiring clinical-validation studies, a
  quality system, and clearance — not a code change. This tool stays a research/triage aid; a CI medical-
  safety audit fails the build on any clinical output. A *molecule* it helps surface can reach patients only
  through the full wet-lab → preclinical → IND → trials → approval pipeline, none of which this repo performs.

The full, honest map of that route — and exactly where this tool may and may not go — is in
[**`CLINICAL-PATHWAY.md`**](CLINICAL-PATHWAY.md).

---

## Methodology & reproducibility

- **Audit:** eight parallel readers, one per subsystem (pipeline, cheminformatics, cost model,
  verify/provenance, structure/docking, docs, OSINT tools, real-world usefulness), each grounding every
  finding in `file:line`.
- **Verification:** each claimed inaccuracy was handed to an independent adversarial verifier instructed to
  open the cited file and confirm, partly-confirm, refute, or mark unverifiable. Only confirmed/partly
  findings appear here. Result: **37 confirmed, 5 partly, 0 refuted.**
- **Caveat:** line numbers reflect the codebase at audit time (2026-06-26) and will drift as the code is
  fixed. The `audit-evidence/` files preserve the exact statements and citations as verified.

---

*This is a self-review of the authors' own project, published in the same spirit the project claims for
itself: compute or cite, never assert. Nothing here is medical advice.*
