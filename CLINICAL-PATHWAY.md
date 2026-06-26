# From this tool to a patient — the only real path

This is the honest answer to "how does something like Provenika reach clinical use?" The short
answer: **the software never does — and that is by design.** A drug it helps triage *might*, over
a decade and through experiments and trials the software does not and cannot run. Conflating the
two is the exact mistake the whole project exists to prevent.

There are two completely different questions hiding in "take it to clinical use," with two
different honest answers.

> This document mirrors `docs/CLINICAL-PATHWAY.md` in the (private) Provenika codebase, published
> here so the clinical stance is fully visible. File references below (e.g. `cad/validate.py`)
> point into that private codebase.

---

## Question A — "can the *software* be used in patient care?"

**Answer: no, and it must not be, in its current form.** Provenika is a hypothesis-generation /
OSINT-triage tool. As the rest of this review documents with file-level evidence, it is **not
validated** and several of its figures don't match their stated specs. Software that informs a
diagnosis, prognosis, or treatment is a **medical device** ("Software as a Medical Device", SaMD)
and is regulated as one:

- **US (FDA):** SaMD requires risk classification, a 510(k)/De Novo/PMA pathway as applicable,
  clinical validation, and a Quality Management System. (Under the 21st Century Cures Act, some
  clinical-decision-support software is *excluded* from device regulation **only if** the clinician
  can independently review the basis for the recommendation — a bar a black-box, non-validated
  triage score does not clear for treatment decisions.)
- **EU (MDR 2017/745):** most clinical software is Class IIa or higher, requiring a notified body,
  clinical evaluation, and CE marking.
- **Quality/standards:** ISO 13485 (QMS), IEC 62304 (software lifecycle), ISO 14971 (risk
  management), plus clinical-validation studies on the intended population.

None of that is a code change. It is an organisation, a quality system, evidence, and a regulatory
submission. The project therefore keeps a hard line — no per-patient output, no dosing, no
prognosis, no treatment recommendation — enforced in CI by a medical-safety audit that fails the
build if any tool emits clinical content without a disclaimer.

---

## Question B — "can a *drug* this tool helped find reach patients?"

**Answer: yes — but only through the standard, human-run, decade-long, regulated pipeline,** where
this tool legitimately occupies just the cheap in-silico *front*. Each stage is gated by
experiment, not software:

| Stage | What actually happens | Where this tool fits | Who runs it |
|------|----------------------|---------------------|-------------|
| **0. Target & hypothesis** | Pick a disease-linked, druggable target; gather measured ligands | ✅ **This tool** — keyless OSINT triage, cited dossiers, ligand shortlists, docking-box setup | Researcher (in-silico) |
| **1. Hit finding / lead** | Assays, SAR, medicinal chemistry; confirm binding & selectivity in vitro | ⚠️ tool output is a *starting list to test*, not a result | Med-chemists + wet lab |
| **2. Preclinical** | Potency/selectivity, ADME, in-vivo PK, and **toxicology** in animals | ❌ out of scope | Pharmacologists, tox labs |
| **3. IND** | File an Investigational New Drug application; FDA allows first-in-human | ❌ | Sponsor + regulators |
| **4. Phase 1** | Safety/dose in a small cohort, under IRB | ❌ | Clinical investigators |
| **5. Phase 2** | Efficacy signal + dosing in patients | ❌ | Clinical investigators |
| **6. Phase 3** | Large randomised controlled trials for efficacy & safety | ❌ | Multi-site trials |
| **7. NDA/BLA → approval** | Regulatory review; label; manufacturing (GMP) | ❌ | Sponsor + FDA/EMA |
| **8. Clinical use + Phase 4** | Prescribing; post-market surveillance | ❌ — and only a *drug* gets here, never this software | Clinicians |

Industry base rates (the same ones the cost-benefit model cites): a program entering Phase 1 has on
the order of an **~8% chance** of eventual approval, over **~10 years** and **~$1–2B**. The
in-silico front this tool addresses is a few percent of that cost and **does not de-risk the
expensive, decisive steps** — it only helps choose where to *start*.

---

## So what does "make it validated" actually mean here?

Not a badge — **evidence**, produced by the validation harness (`cad/validate.py` in the private
codebase):

1. **Redocking pose accuracy** — re-dock known co-crystal ligands and measure RMSD to the
   crystallographic pose (≤2 Å = correct). Reports per-complex RMSD, mean, and success rate.
2. **Retrospective enrichment** — rank known actives mixed with decoys by the triage score; report
   ROC AUC and enrichment factor.

These are *necessary, not sufficient*: reproducing known answers earns cautious trust in
prospective triage; it does **not** establish clinical validity, which only trials provide. The
harness is gated on real binaries (Vina/Open Babel) and **never fabricates a number** — if it
can't measure, it says so. **A harness is not a result:** until those benchmarks are run and
reported, the pipeline remains unvalidated.

---

## The honest bottom line

- The **software** stays a research/triage aid. It is not, and should not be made to look like, a
  clinical tool. The path to clinical software is regulatory, not a commit.
- A **molecule** it helps surface can reach patients only by passing every wet-lab, preclinical,
  and clinical-trial gate above — none of which this project performs.
- The best, most honest thing this project can do toward "real" impact is exactly what it claims:
  produce **cited, re-verifiable, validated-where-measurable** research leads, and hand them to the
  scientists and clinicians who run the regulated process. Nothing here is medical advice.
