# Provenika — known limitations

Every item below was found by reading the (private) Provenika source and **adversarially re-verified**
against the cited file. Status: **37 confirmed, 5 partly-confirmed, 0 refuted.** Severity is the audit's,
from the standpoint of "could this mislead a researcher who trusts the output." Line numbers are as of
2026-06-26 and will drift as fixes land. Raw per-finding evidence: [`audit-evidence/`](audit-evidence/).

Legend: 🔴 high · 🟠 medium · 🟡 low · ✅ fixed · ◐ partly · ⏳ open

> **Remediation status (updated):** **most items below have since been FIXED** in the private codebase
> (docking-box selection + geometry, `--autobox`, mmCIF fetch, coverage-aware structure pick + AlphaFold
> pLDDT, exact-gene-symbol resolution, "bioactivity records"/"docking feasible" relabels, potency-ordered
> triage, Veber/`clean`/Brenk fidelity, verifier + README de-overclaiming plus a deeper verify
> (`provenance.json` cross-check + independent QED re-fetch), "63 tools" framing, staleness
> surfacing) plus a new validation harness and clinical-pathway doc. See the **README's "Status" section**
> for the per-item summary; that is the current source of truth. Individual entries below may still read in
> the present tense as originally found. What remains is inherent (apo/holo + mutation can only be flagged,
> not auto-resolved by a triage tool) or cosmetic (one JSON key rename).

---

## 1. Scientific correctness — CAD data not matched to its specifications

This is the category the project most needs to confront: numbers that are *computed*, *re-verifiable*, and
*wrong-for-the-stated-purpose*, because the computation does not match the specification or label it invokes.

- ✅ 🔴 **Docking box was the largest hetero group, not the inhibitor.** The box was centered on whichever
  non-solvent hetero residue had the most atoms; the `IGNORE` blocklist excluded only water/ions/cryo-additives,
  not cofactors (HEME ~43, FAD ~53, NAD ~44 atoms), so a cofactor could outrank a small inhibitor and the box
  never checked it occupied the relevant pocket. *(`cad/binding_site.py:74`, `:31-35`; described as fact at
  `README.md:118`, `run_pipeline.py:156-157`.)* **Fixed:** selection now scores by protein burial (≤4.5 Å
  contact) + drug-like heavy-atom size and warns on uncertainty.
- ✅ 🟠 **Box center and size used inconsistent reference points.** Center was the arithmetic *mean* of ligand
  atoms while size is the *(max−min) extent* + padding, so the box was not symmetric about the ligand's true
  extent. *(`cad/binding_site.py:81-82`.)* **Fixed:** center is now the bounding-box midpoint `(min+max)/2`,
  matching how Vina treats the box as center ± size/2.
- ⏳ 🟡 **Box padding is a fixed 8 Å/side (16 Å/axis).** Derived from ligand extent, not detected pocket
  geometry — larger than tight boxes recommended for focused redocking. *(`cad/binding_site.py:79,82`.)*
- ✅ 🔴 **"Potent activities = N" applied no potency threshold and counted records, not molecules.** The dossier
  figure counts every ChEMBL activity with a non-null pChEMBL and reads `total_count` (records), yet was labeled
  "potent measured ligands." *(`cad/target_report.py`; `run_pipeline.py`; `README.md`.)* **Fixed:** relabelled
  honestly everywhere as "ChEMBL bioactivity records (any pChEMBL, not potency-filtered, records not distinct
  molecules)". (JSON key kept stable for the verifier.)
- ✅ 🔴 **Ligand triage ranked an arbitrary, unordered slice and called it "the most potent."** The activity
  query had no `order_by` and scanned only the first `--scan` records (truncated by a 40 s budget), so for
  high-data targets the ranking was best-of-an-arbitrary early slice. *(`cad/virtual_triage.py`.)* **Fixed:**
  the query is now `order_by=-pchembl_value` and stops at the pChEMBL floor, so a bounded/truncated scan keeps
  the genuinely most-potent records.
- ◐ 🟠 **Potency conflates assay types and cherry-picks the best value.** IC50/Ki/Kd/EC50 are pooled into one
  pChEMBL axis and only the single *maximum* value per molecule is kept, with no assay-type/validity/confidence
  filter. *(`cad/virtual_triage.py`.)* **Documented:** the docstring now states the pooling + most-favorable
  selection plainly; the pooling itself is unchanged (a deliberate triage simplification).
- ✅ 🔴 **Target resolution differed across stages — the lookups could describe different proteins.**
  "AKT1" could resolve to AKT1S1 in the dossier. *(`cad/target_report.py`, `virtual_triage.py`,
  `fetch_structure.py`.)* **Fixed on all three:** `target_report` (ChEMBL) and `fetch_structure` (UniProt) now
  apply the SAME exact gene-symbol guard `virtual_triage` had; and the verifier now cross-checks the UniProt
  accession recorded in the manifest against the dossier (see §3), so a cross-stage mismatch is caught.
- ✅ 🟠 **Gene→UniProt took the top relevance hit with no exact-symbol check.** *(`cad/fetch_structure.py`.)*
  **Fixed:** `resolve_uniprot` requests `gene_primary` (size 5) and prefers the entry whose primary gene symbol
  exactly matches the query, falling back to the first hit only when none match.
- ✅ 🔴 **"Docking feasible: yes" was inferred from the existence of any PDB.** `pdb_count > 0` counted all PDB
  cross-references (apo, NMR, fragment, mutant) so it meant only that *some* structure exists.
  *(`cad/target_report.py`, `run_pipeline.py`.)* **Fixed:** relabelled "structures exist — confirm one is holo
  and covers the site (not auto-checked)" in both the dossier and the SUMMARY.
- ◐ 🟠 **"Best" PDB optimized resolution only — ignored coverage, domain, apo/holo, mutation.** Could return a
  high-resolution structure of the wrong domain, an apo form, or a point mutant with no flag.
  *(`cad/fetch_structure.py`.)* **Mostly fixed:** selection is now **coverage-aware** (prefers structures
  spanning ≥50% of the protein over fragments) AND **holo-aware** for the docking path — `pick_structure`
  probes the top candidates and prefers one with a bound ligand, so the box step isn't handed an apo entry
  (and flags apo when no holo is found). **Inherent residue:** mutation/construct identity still can't be
  auto-confirmed without lab context — now stated explicitly.
- ✅ 🟠 **AlphaFold fallback never read pLDDT and only fetched fragment F1.** A low-confidence or truncated
  predicted model was handed off identically to a good one. *(`cad/fetch_structure.py`.)* **Fixed:** the
  AlphaFold path now reads mean/percent-confident **pLDDT** (B-factor column) and states only fragment F1 is
  fetched (large proteins truncated), instead of a static advisory string.
- ✅ 🟠 **Blind-box docking used `--autobox`, which stock AutoDock Vina does not support.** The no-center branch
  passed a flag that exists only in smina/gnina (and even there it is `--autobox_ligand`), so the documented
  blind-docking path errored out instead of running. *(`cad/dock.py:160-163`.)* **Fixed:** the blind path now
  computes a real whole-receptor box (bounding-box midpoint + extent + margin) and passes explicit
  `--center/--size`, since Vina has no auto-box mode; a regression test guards against the flag returning.

---

## 2. The feasibility / cost-benefit model

The clinical-phase approval priors and per-phase costs are correctly transcribed from BIO/Informa and DiMasi —
that part is real. The problem is everything around them, presented under one "sources" line that blends
sourced and invented numbers.

> ◐ **Partly fixed.** The human-facing readout and docstring now state these honestly: P(approval) is flagged
> as a phase×modality benchmark identical for every program; "Risk-adjusted revenue" is relabelled **gross
> profit**; the BCR is flagged undiscounted + asymmetric; the verdict bands are flagged unsourced; and the
> BIO (preclinical estimate) / Wong (success rates, not costs) / DiMasi (costs) attribution is corrected.
> The `risk_adjusted_revenue_musd` JSON key has now also been **renamed** to
> `risk_adjusted_gross_profit_musd`. The only inherent residue is that P(approval) is a phase×modality
> benchmark with no target-specific signal — which the readout now states outright.

- ⏳ 🟠 **Reads as target-specific but is a pure modality × phase benchmark.** `probability_of_approval` is a
  static lookup × oncology factor with *zero* target/molecule/biology input — identical for every program at a
  given phase. *(partly: the BCR verdict does vary with user market inputs, and the docstring discloses it.)*
  *(`cad/cost_benefit.py:76,105`.)*
- ✅ 🟡 **Preclinical approval prior was invented but attributed to BIO.** *(`cad/cost_benefit.py`.)* **Fixed:**
  the docstring + dict comment now state the preclinical 6% is a rough author estimate NOT in the BIO report.
- ✅ 🟡 **Wong et al. 2019 cited for per-phase cost it does not provide.** *(`cad/cost_benefit.py`.)* **Fixed:**
  attribution corrected — per-phase cost is DiMasi-lineage; Wong is for success rates/durations.
- ✅ 🟠 **"Revenue" was actually risk-adjusted gross profit** (multiplied by an 0.80 gross-margin factor).
  *(`cad/cost_benefit.py`.)* **Fixed:** the readout says "Risk-adj. gross profit" AND the JSON key is renamed
  `risk_adjusted_revenue_musd` → `risk_adjusted_gross_profit_musd`, with every reader (the verifier, the
  SUMMARY) updated in lockstep.
- ✅ 🟠 **No time-value discounting over a ~9.5-year horizon.** *(`cad/cost_benefit.py`.)* **Fixed (disclosed):**
  the BCR line now states it is undiscounted (so it overstates a true rNPV).
- ✅ 🟡 **Asymmetric risk treatment** (revenue LOA-weighted, cost not). *(`cad/cost_benefit.py`.)* **Fixed
  (disclosed):** the readout flags the asymmetry explicitly.
- ✅ 🟠 **Go/no-go thresholds (BCR ≥5 / ≥1.5 / ≥0.8) have no cited basis.** *(`cad/cost_benefit.py`.)* **Fixed
  (disclosed):** the verdict line + a code comment now flag the bands as unsourced heuristics.
- ◐ 🟡 **Triage drug-likeness score double-counts Lipinski space** (QED + a separate Ro5 term). Already flagged
  non-validated in code, so the risk is presentational. *(`cad/virtual_triage.py`.)* (Unchanged; a deliberate,
  disclosed heuristic.)

---

## 3. Provenance & verification — the audit trail overclaims

`verify.py` is a real tamper-detector and reproducibility gate (it genuinely re-fetches counts, does an
independent raw-HTTP SMILES check, and recomputes several artifacts, failing CI on mismatch). But the headline
claims — "every figure re-pulled," "fabrication impossible to hide," "no value originates from a model" —
are stronger than the code delivers.

> ◐ **Largely fixed — coverage widened substantially.** The overclaiming WORDING is corrected, AND the real
> gaps have since closed: each top hit's **SMILES, QED, AND potency (`best_pchembl`) are now all independently
> re-fetched** from ChEMBL (commits `ae5d774`, `ae21871`), **`provenance.json` is cross-checked** against the
> artifacts (`e3fbeb5`), and the hits check now covers **all 25 shortlist rows** (the top-5 cap was raised). So
> a self-consistent potency+score fabrication no longer passes. **And the descriptors are now re-fetched too**
> (mw/alogp/TPSA, commit `04a1726`), so **EVERY ChEMBL-sourced shortlist figure** — SMILES, QED, potency, and
> descriptors — is independently re-pulled across all 25 rows; the original "every figure re-pulled" claim is now
> literally true for the shortlist. **Sole residue (by design):** **DRIFT exits 0** — a within-tolerance change
> in a living database is legitimate growth, not fabrication; it is now shown, not silent.

- ✅ 🔴 **Every ChEMBL-sourced ligand figure is now re-pulled.** Each shortlist hit's **SMILES, QED, potency
  (`best_pchembl`), AND mw/alogp/TPSA are independently re-fetched from ChEMBL** and compared (commits
  `ae5d774`, `ae21871`, `04a1726`), across all 25 rows — so the original "every figure re-pulled" claim is now
  literally true for the shortlist. *(`cad/verify.py`.)*
- ✅ 🟠 **A self-consistent potency fabrication is now caught.** Editing `best_pchembl` + its matching score no
  longer slips through: potency is independently re-fetched from ChEMBL and compared (live < saved FAILs), as
  are SMILES and QED. *(`cad/verify.py`.)*
- ◐ 🟠 **DRIFT exits 0 (by design) — now disclosed, not silent.** A saved-vs-live difference within
  `max(2, 10%)` is a non-failing DRIFT, because a living database legitimately gains/loses records; failing CI
  on that would break on every real DB update. The banner/docstring now state DRIFT is symmetric and exits 0,
  so it is visible rather than blessed silently. *(`cad/verify.py`.)*
- ⏳ 🟠 **A wrong-but-stable query passes by design.** Count checks re-run the *same* query logic, proving
  reproducibility, not query-design correctness — and `potent_activity_records` is itself the mislabeled,
  unfiltered query from §1. *(`cad/verify.py:16-19`; self-disclosed in the docstring.)*
- ⏳ 🟡 **The SMILES check proves ID↔SMILES consistency, not that the molecule is a target hit.** *(partly: the
  repo states this limitation explicitly elsewhere.)* And the "byte-equal to ChEMBL" pull reads the same
  `canonical_smiles` field the pipeline read — a separate code path, not an orthogonal data source. *(partly.)*
  *(`cad/verify.py:155-180`.)*
- ◐ 🟠 **`provenance.py`'s write-time check validates only the label string, not the value's source.**
  `Figure.__post_init__` raises on any origin other than `fetched`/`computed` (so a literal `model` label *does*
  raise), but nothing at write time binds the value to the cited source — a model-produced number passed as
  `manifest.fetched(...)` is accepted. The "no value originates from a model" guarantee is a labeling
  convention, not write-time enforcement. **Narrowed:** `verify.py`'s new `verify_manifest` now re-validates
  every figure's origin and cross-checks the figures that *mirror an artifact value* against that artifact (see
  the §3 last item), so an isolated manifest edit is caught after the fact — but a figure with no mirrored
  artifact is still trusted on its label alone. *(`cad/provenance.py:93-98,108-113`; `cad/verify.py:280-345`.)*
- ✅ 🟡 **The manifest was never verified against the artifacts.** `verify.py` imported provenance only for URLs
  and never read `provenance.json` back. *(`cad/verify.py`.)* **Fixed:** new `verify_manifest` re-validates every
  figure's origin (an illegal `model` origin FAILs) and cross-checks the figures that mirror an artifact value
  (UniProt accession, PDB/activity/drug counts, P(approval), benefit/cost, docking box) against
  dossier/cost_benefit/binding_site — FAIL on any disagreement. Offline-tested.
- ⏳ 🟡 **cost-benefit "reproduces exactly" only checks internal consistency.** It recomputes from the saved
  assumptions; it does not verify those assumptions or that the hardcoded priors match the cited source.
  *(`cad/verify.py:216-221`.)*
- ⏳ 🟡 **Only the top 5 hits and first 25 liabilities are verified.** Rows past those fixed caps are never
  checked. *(`cad/verify.py:142-147,270-290`.)*
- ⏳ 🟡 **Deterministic checks degrade to SKIP, not FAIL.** *(partly:* an absent RDKit or a blocked PDB fetch
  downgrades the liabilities/box integrity checks to a non-failing SKIP; the count and SMILES checks do fail
  closed.) *(`cad/verify.py:252-253,279-282,385,394`.)*

---

## 4. Cheminformatics rule-set fidelity & data freshness

The chemistry is real RDKit and faithful — these are fidelity caveats, not fakes.

- ✅ 🟠 **`clean` ignored Brenk alerts** (`clean = Ro5 ∧ Veber ∧ no-PAINS`), so 175 committed compounds were
  `clean:true` while carrying Brenk alerts. *(`cad/cheminformatics.py`.)* **Fixed:** `clean` now also requires
  zero Brenk alerts; the committed `/explore` data is regenerated to match (0 such compounds remain).
- ✅ 🟡 **Veber rule omitted the HBD+HBA≤12 alternative clause** (PSA-only). *(`cad/cheminformatics.py`.)*
  **Fixed:** the H-bond alternative clause is implemented, with a test.
- ✅ 🟡 **Egan rule is a rectangular approximation of an ellipsoidal model** (Crippen MolLogP for AlogP98).
  *(`cad/cheminformatics.py`.)* **Fixed (documented):** now explicitly commented as a permissive rectangular
  approximation so it is not mistaken for the exact ellipse.
- ✅ 🟡 **PAINS/Brenk name arrays were truncated to 5 while the count was full.** *(`cad/cheminformatics.py`.)*
  **Fixed:** the arrays are no longer truncated.
- ◐ 🟠 **Committed `/explore` JSON is a live-ChEMBL snapshot** (compound set pulled live, so not byte-reproducible),
  and a failed per-target refresh could leave a stale file while `index.json`'s date advanced.
  *(`cad/precompute_site_data.py`, `virtual_triage.py`.)* **Fixed (staleness surfaced):** each index entry now
  carries its OWN refresh date, so a stale per-target file is visible (entry date < index date); the data was
  also just regenerated. Inherent: it is still a dated snapshot of a living database.
- ◐ 🟡 **`saScore` is schema-fragile** (emitted only when RDKit's Contrib `SA_Score` is installed).
  *(`cad/cheminformatics.py`, `precompute_site_data.py`.)* **Mitigated:** `verify.py` already only compares SA
  when both sides computed it, so a Contrib-less environment does not produce a false FAIL; the field can still
  be absent on such a regen (disclosed).

---

## 5. Scope & framing overclaims

- ✅ 🟠 **"63 OSINT tools" was an accurate count but a misleading frame** (only ~21 are keyless live lookups; ~13
  need a paid Tavily key; ~25 are static datasets/templates). *(`src/tools/cancer/`, `cicd/generate_readme.py`.)*
  **Fixed:** the README now states "~21 are keyless live-API lookups; the rest need a Tavily key or are static
  reference datasets / templates."
- ◐ 🟠 **Confident headlines vs buried caveats.** The deeper docs are candid, but the top-level framing sat above
  the caveats. **Partly fixed:** "fabrication impossible to hide" → "easy to catch", the verifier banner and the
  docking-feasible/potency labels are corrected, and `REAL-CAD-ROADMAP` already flags docking as wrapper-only.
  **Still open:** a full top-level reconciliation pass of the older docs' headline framing.

---

## 6. Real-world adoption — utility is entirely prospective

- ⏳ 🔴 **No evidence of any real oncologist or researcher using Provenika.** The CADD core and the committed
  EGFR sample run are genuinely real and re-runnable, but nothing has shipped to a real user.
- ⏳ 🟠 **The "outreach" system is investor email automation, and it is not live.** Not clinician/patient
  outreach. `web/public/data/outreach/status.json`: `overall: fail`, `send_enabled: false`, `sent: 0`; all four
  stored contacts are VCs in "drafted" status; the public outreach log has zero entries.
- ✅ **To the project's credit:** it does *not* fabricate traction — pitch figures are explicit `[fill in]`
  placeholders and the business verdict is a candid "pass as a venture investment." The honesty risk is the gap
  between an impressive provenance/pitch apparatus and the fact that nobody is using it yet.

---

## Prioritized remediation queue

1. ✅ Docking-box ligand selection (buried + drug-like) — *done.*
2. ⏳ `saScore` schema fragility — make the verifier tolerant; stabilize the field.
3. ⏳ `.cif`-only structure fetch — fall back from `.pdb` to mmCIF.
4. ⏳ Holo/coverage/mutation-aware structure selection; surface pLDDT; fetch all AlphaFold fragments.
5. ⏳ Re-label "potent activities" (apply a real threshold or rename) and the feasibility verdict
   (target-independent, gross-profit, undiscounted).
6. ◐ Verifier honesty: headline scoped ✅, QED re-pulled ✅, `provenance.json` cross-checked ✅. STILL OPEN —
   re-pull potency/descriptors, make DRIFT/SKIP non-silent, and lift the top-5/25 caps.
7. ✅ Fix the `--autobox` Vina path — *done.* ⏳ Make `clean` include Brenk; complete Veber.
8. ⏳ Reconcile the README aggregate framing with the per-tool reality and the buried caveats.

*Nothing here is medical advice. Provenika is research tooling for the in-silico front of discovery, not a
validated pipeline and not a clinical instrument.*
