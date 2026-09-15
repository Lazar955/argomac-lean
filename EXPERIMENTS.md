# Experiment log

Append-only. One entry per attempt, in either direction (verified or not).
Newest first. See [AGENTS.md](AGENTS.md) for the challenge/repo context.

Entry template:

```
## YYYY-MM-DD — <short name>

- Branch/commit:
- Approach:
- Changed:
- Local build: pass / fail (paste the first real error if fail)
- Submitted: yes/no — if yes, verifier result + ciphertextBytes
- Outcome: verified win / verified no-improvement / rejected / abandoned
- Why (if failed/abandoned):
- Next idea this suggests:
```

---

## 2026-09-15 — baseline reproduction, no change yet

- Branch/commit: `lazar/attempt-1` @ `711689c` (= upstream baseline, unmodified)
- Approach: none — getting the toolchain and verifier working locally before
  attempting any construction change.
- Changed: nothing in `Construction/`/`Proof/`. Forked the repo, set up this
  log and AGENTS.md (both outside the verified paths).
- Local build: in progress (`lake build` against pinned mathlib v4.33.1 +
  VCVio, in `~/work/kriterion/examples/bn254-scalar-multiplication`).
- Submitted: no.
- Outcome: n/a.
- Findings worth keeping (see AGENTS.md for full detail): the 9,699,931
  bytes is ~99.6% the `pointMAC` table, driven by 13 populated 8,129-byte
  digit-adaptor tables per row (91 rows) where the BaBe paper's construction
  only needs 9 via cross-coordinate table sharing. The baseline's own README
  says they couldn't prove the shared-table distribution equivalence for
  their block formula and used independent tables per coordinate instead.
- Next idea this suggests: read `gc_rpm_proof.tex` (BaBe.latex) and
  `Proof/Privacy/PaperConstruction.lean` to scope whether the paper's
  sharing argument is provable in Lean for this construction, or whether
  it's genuinely blocked (in which case try smaller/orthogonal savings
  instead — see "Secondary/fallback ideas" in AGENTS.md).
