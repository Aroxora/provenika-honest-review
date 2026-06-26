# Audit evidence

Raw output of the multi-agent audit behind [`../KNOWN-LIMITATIONS.md`](../KNOWN-LIMITATIONS.md). Every
inaccuracy was found by reading the (private) Provenika source, then **independently re-verified** by a
separate adversarial pass instructed to open the cited file and confirm / partly-confirm / refute /
mark-unverifiable. Tally: **37 confirmed, 5 partly, 0 refuted.**

## Files

- **`verified-limitations.txt`** — the publishable list: each verified finding with its `[CONFIRMED]` /
  `[PARTLY]` verdict, a one-sentence defensible statement, and the exact `file:line` evidence checked.
- **`audit-by-subsystem.txt`** — the eight per-subsystem audit reports (pipeline, cheminformatics, cost
  model, verify/provenance, structure/docking, docs, OSINT tools, real-world usefulness): summary,
  what-is-real, what-is-stub/wrapper, and every raw inaccuracy with claim / reality / evidence / severity /
  spec-mismatch flag.
- **`findings.json`** — the full machine-readable record: `audits` (per-subsystem) and `verifies`
  (per-finding verdicts + corrected statements).

## How to read it

- A finding marked **SPEC-MISMATCH** diverges from a real published specification (e.g. the Veber/Egan
  rules, AutoDock Vina's flags). The rest are accuracy/labeling/overstatement issues — figures that are
  reproducible but wrong-for-their-stated-purpose, or claims stronger than the code delivers.
- Line numbers are as of **2026-06-26** and will drift as fixes land. These files preserve the exact
  statements and citations as verified.
- The audit was deliberately adversarial: agents were told to refute where they could. The 5 `[PARTLY]`
  verdicts are findings where the verifier judged the original critique overstated and narrowed it — those
  nuances are kept verbatim so the record is not one-sided.

*Nothing here is medical advice.*
