# Provenika — known limitations

Every item below was found by reading the (private) Provenika source and **adversarially re-verified**
against the cited file. Status: **37 confirmed, 5 partly-confirmed, 0 refuted.** Severity is the audit's,
from the standpoint of "could this mislead a researcher who trusts the output." Line numbers are as of
2026-06-26 and will drift as fixes land. Raw per-finding evidence: [`audit-evidence/`](audit-evidence/).

Legend: 🔴 high · 🟠 medium · 🟡 low · ✅ fixed · ⏳ open

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
- ⏳ 🟠 **Box center and size use inconsistent reference points.** Center is the arithmetic *mean* of ligand
  atoms while size is the *(max−min) extent* + padding, so the box is not symmetric about the ligand's true
  extent; the 8 Å pad usually absorbs it, but a skewed atom distribution on an axis >16 Å can clip the pocket.
  *(`cad/binding_site.py:81-82`.)*
- ⏳ 🟡 **Box padding is a fixed 8 Å/side (16 Å/axis).** Derived from ligand extent, not detected pocket
  geometry — larger than tight boxes recommended for focused redocking. *(`cad/binding_site.py:79,82`.)*
- ⏳ 🔴 **"Potent activities = N" applies no potency threshold and counts records, not molecules.** The dossier
  figure counts every ChEMBL activity with a non-null pChEMBL (weak measurements included) and reads
  `total_count` (records), yet is labeled "potent measured ligands." It also contradicts the repo's *own*
  triage stage, which does enforce a pChEMBL floor and assay-type filter. *(`cad/target_report.py:61,62,75,187`;
  propagates to `run_pipeline.py:179`, `README.md:54`.)*
- ⏳ 🔴 **Ligand triage ranks an arbitrary, unordered slice and calls it "the most potent."** The activity
  query has no `order_by` and scans only the first `--scan` records (default 4000, fewer if a 40 s budget
  truncates), so for high-data targets (e.g. EGFR) the ranking is best-of-an-arbitrary early slice and can vary
  between runs. *(`cad/virtual_triage.py:117,123-131,286-288,382`.)*
- ⏳ 🟠 **Potency conflates assay types and cherry-picks the best value.** IC50/Ki/Kd/EC50 are pooled into one
  pChEMBL axis and only the single *maximum* (most potent) value per molecule is kept, with no assay-type,
  data-validity, or confidence filter. *(`cad/virtual_triage.py:45,139,147-149`.)*
- ⏳ 🔴 **Target resolution differs across stages — the three lookups can describe different proteins.**
  `target_report` (ChEMBL) has no exact gene-symbol guard while `virtual_triage` does; the two UniProt lookups
  take the first fuzzy `gene OR protein_name` match (`size:1`, `results[0]`). So the dossier counts, the
  triaged ligands, and the docking-box PDB can belong to *different* proteins for one input (e.g. AKT1 →
  AKT1S1), and the manifest never cross-checks the accessions. *(`cad/target_report.py:52-56,79-81`;
  `cad/virtual_triage.py:81-101`; `cad/fetch_structure.py:55-56`.)*
- ⏳ 🟠 **Gene→UniProt takes the top relevance hit with no exact-symbol check.** A short/ambiguous/substring name
  can silently resolve to the wrong protein and propagate through the structure/box/dock chain.
  *(`cad/fetch_structure.py:55-60`.)*
- ⏳ 🔴 **"Docking feasible: yes" is inferred from the existence of any PDB.** `pdb_count > 0` counts all PDB
  cross-references indiscriminately — apo, NMR, fragment, mutant, partial-domain — so it indicates only that
  *some* structure exists, not that a dockable pocket structure is available. *(`cad/target_report.py:93,99,174`;
  `cad/run_pipeline.py:175`.)*
- ⏳ 🟠 **"Best" PDB optimizes resolution only — ignores coverage, domain, apo/holo, mutation.** Can return a
  high-resolution structure of the wrong domain, an apo form, or a point mutant (e.g. EGFR T790M) with no flag;
  on an apo pick the pipeline silently skips box generation rather than seeking a holo structure.
  *(`cad/fetch_structure.py:65-72,98-105,135-145`.)*
- ⏳ 🟠 **AlphaFold fallback never reads pLDDT and only fetches fragment F1.** A low-confidence or truncated
  predicted model is handed off identically to a good one; the "validate pLDDT before docking" string is
  advice, not an enforced gate, and proteins >~2700 residues lose everything past F1.
  *(`cad/fetch_structure.py:83-84,88,158-171`.)*
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

- ⏳ 🟠 **Reads as target-specific but is a pure modality × phase benchmark.** `probability_of_approval` is a
  static lookup × oncology factor with *zero* target/molecule/biology input — identical for every program at a
  given phase. *(partly: the BCR verdict does vary with user market inputs, and the docstring discloses it.)*
  *(`cad/cost_benefit.py:76,105`.)*
- ⏳ 🟡 **Preclinical approval prior is invented but attributed to BIO.** BIO/Informa CDSR reports only
  clinical-phase rates; the preclinical 6% is an author estimate surfaced under a citation that does not contain
  it. *(`cad/cost_benefit.py:35-37,121`.)*
- ⏳ 🟡 **Wong et al. 2019 cited for per-phase cost it does not provide.** Wong estimated success rates and
  durations, not development costs; the encoded costs are DiMasi-lineage. *(`cad/cost_benefit.py:19-20,47-49,121`.)*
- ⏳ 🟠 **"Revenue" is actually risk-adjusted gross profit.** The output field/print labeled "revenue" is
  multiplied by an 0.80 gross-margin factor, so the benefit/cost ratio compares profit to cost.
  *(`cad/cost_benefit.py:86-87,109,139`.)*
- ⏳ 🟠 **No time-value discounting over a ~9.5-year horizon.** Future peak-sales profit is valued at face value
  with no NPV/discount parameter, inflating the ratio vs a proper rNPV. *(`cad/cost_benefit.py:45-51,87,90`.)*
- ⏳ 🟡 **Asymmetric risk treatment.** Revenue is probability-weighted by LOA; "expected remaining cost" is the
  full deterministic sum of all remaining phases, unweighted and undiscounted — so benefit and cost are
  risk-adjusted on inconsistent bases. *(`cad/cost_benefit.py:81 vs :87`.)*
- ⏳ 🟠 **Go/no-go thresholds (BCR ≥5 / ≥1.5 / ≥0.8) have no cited or derived basis.** Hardcoded round numbers
  with no source or sensitivity analysis, unlike every other constant in the file, yet they drive the headline
  verdict. *(`cad/cost_benefit.py:92-99`.)*
- ⏳ 🟡 **Triage drug-likeness score double-counts Lipinski space.** It linearly combines QED (which already
  encodes MW/logP/HBD/HBA) with a separate Ro5-violation term. Explicitly flagged non-validated in code, so the
  risk is presentational. *(`cad/virtual_triage.py:238-239,244,247-248`.)*

---

## 3. Provenance & verification — the audit trail overclaims

`verify.py` is a real tamper-detector and reproducibility gate (it genuinely re-fetches counts, does an
independent raw-HTTP SMILES check, and recomputes several artifacts, failing CI on mismatch). But the headline
claims — "every figure re-pulled," "fabrication impossible to hide," "no value originates from a model" —
are stronger than the code delivers.

- ⏳ 🔴 **Most ligand-shortlist numbers are never re-pulled.** For each top hit only the canonical SMILES is
  re-fetched; potency (`best_pchembl`), `qed`, `ro5_violations`, similarity, and MW/alogp/etc. are read from the
  file and fed unchanged into the score recompute — so the score proves only that the column is *internally
  consistent with its own row*, not that the inputs are real. *(`cad/verify.py:189-205`; only SMILES re-pulled at
  `:155-180`; overstated at `verify.py:7`, `:397`, `README.md:35,135`.)*
- ⏳ 🟠 **A self-consistent fabrication passes.** Edit potency *and* its matching score together and the row
  reproduces and passes; only an isolated score-column edit is caught. *(`cad/verify.py:189-205`.)*
- ⏳ 🟠 **DRIFT silently blesses changed numbers and still prints "No fabrications."** Any saved-vs-live
  difference within `max(2, 10%)` (symmetric — a count going *down* counts too) is a non-failing DRIFT that
  exits 0. *(`cad/verify.py:83-85,413,418`.)*
- ⏳ 🟠 **A wrong-but-stable query passes by design.** Count checks re-run the *same* query logic, proving
  reproducibility, not query-design correctness — and `potent_activity_records` is itself the mislabeled,
  unfiltered query from §1. *(`cad/verify.py:16-19`; self-disclosed in the docstring.)*
- ⏳ 🟡 **The SMILES check proves ID↔SMILES consistency, not that the molecule is a target hit.** *(partly: the
  repo states this limitation explicitly elsewhere.)* And the "byte-equal to ChEMBL" pull reads the same
  `canonical_smiles` field the pipeline read — a separate code path, not an orthogonal data source. *(partly.)*
  *(`cad/verify.py:155-180`.)*
- ⏳ 🟠 **`provenance.py` validates only the label string, not the value's source.** `Figure.__post_init__`
  raises on any origin other than `fetched`/`computed` (so a literal `model` label *does* raise), but nothing
  binds the value to the cited source — a model-produced number passed as `manifest.fetched(...)` is accepted.
  The "no value originates from a model" guarantee is a labeling convention, not enforcement. *(partly.)*
  *(`cad/provenance.py:93-98,108-113`; `README.md:34`.)*
- ⏳ 🟡 **The manifest is never verified against the artifacts.** `verify.py` re-checks the underlying figures but
  never reads `provenance.json`, so a manifest edited in isolation would not FAIL. *(`cad/verify.py:343-383`.)*
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

- ⏳ 🟠 **`clean` ignores Brenk alerts.** `clean = Ro5 ∧ Veber ∧ no-PAINS`; Brenk alerts are computed and shown
  but never gate the flag, so **175 committed compounds are `clean:true` while carrying Brenk alerts** (e.g.
  BRAF CHEMBL500659, `brenkAlerts=3`). *(`cad/cheminformatics.py:113`; `web/public/data/cheminformatics/BRAF.json`.)*
- ⏳ 🟡 **Veber rule omits the HBD+HBA≤12 alternative clause.** Only the TPSA branch is implemented, so a
  high-TPSA but low-H-bond molecule is misreported as failing Veber. *(`cad/cheminformatics.py:91`.)*
- ⏳ 🟡 **Egan rule is a rectangular approximation of an ellipsoidal model**, using Crippen MolLogP where the
  paper uses AlogP98 — corner cases misclassify. *(`cad/cheminformatics.py:92`.)*
- ⏳ 🟡 **PAINS/Brenk name arrays are truncated to 5 while the count is full** — the listed alerts can be fewer
  than the reported total. *(`cad/cheminformatics.py:110-111`.)*
- ⏳ 🟠 **Committed `/explore` JSON is a non-reproducible live-ChEMBL snapshot (2026-06-23).** The compound set is
  pulled live with a truncating time budget, so re-running yields different molecules; a failed per-target
  refresh leaves the stale file in place while `index.json`'s `generated` date advances — advertising a
  freshness some files lack. *(`cad/precompute_site_data.py:88-89,130,158`; `cad/virtual_triage.py:60-62`.)*
- ⏳ 🟡 **`saScore` is schema-fragile.** It is emitted only when RDKit's optional Contrib `SA_Score` tree is
  installed; regenerating on a Contrib-less RDKit silently drops the field from every entry, diverging from the
  38 committed files that all include it. *(`cad/cheminformatics.py:54-56,100`; `precompute_site_data.py:111`.)*

---

## 5. Scope & framing overclaims

- ⏳ 🟠 **"63 OSINT tools" is an accurate count but a misleading frame.** Only ~21 hit free public APIs keyless;
  ~13 are Tavily wrappers that throw without a paid `TAVILY_API_KEY`, and ~25 are static curated datasets or
  template scaffolds (oral-drug lists, gap analysis, CAR-T/mRNA/IND "design", treatment/biomarker heuristics)
  that query nothing live. The per-tool docstrings are mostly honest; the aggregate README framing is not.
  *(`src/tools/cancer/index.ts`; `cicd/generate_readme.py:31`.)*
- ⏳ 🟠 **Confident headlines vs buried caveats.** The deeper docs (`ANTI-HALLUCINATION.md`, `REAL-CAD-ROADMAP.md`)
  are candid and already enumerate most limitations, but the top-level framing ("auditable evidence engine,"
  "fabrication impossible to hide," docking "implemented") sits one or two layers above the caveats that qualify
  it (docking is wrapper-only and not run in CI; the binding box quietly fails on apo/AlphaFold structures).

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
6. ⏳ Verifier honesty: scope the "every figure" headline; re-pull potency/QED; verify `provenance.json`;
   make DRIFT/SKIP non-silent; raise or document the top-5/25 caps.
7. ✅ Fix the `--autobox` Vina path — *done.* ⏳ Make `clean` include Brenk; complete Veber.
8. ⏳ Reconcile the README aggregate framing with the per-tool reality and the buried caveats.

*Nothing here is medical advice. Provenika is research tooling for the in-silico front of discovery, not a
validated pipeline and not a clinical instrument.*
