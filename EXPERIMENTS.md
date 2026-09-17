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

## 2026-09-17 — competition went live, loophole exploited by others (not us)

- Branch/commit: n/a — observation only, no work of ours.
- Approach: n/a.
- Changed: nothing in this repo. Read the Slack announcement thread
  (`C079KTWQKBL`/`1789488716.857219`) and the live board.
- Local build: n/a.
- Submitted: no.
- Outcome: n/a for us. For the record: two people (liucheng, then marcin)
  independently found the adaptive-privacy definition hands the simulator
  an unbounded/true-output view, making privacy vacuous on the valid-input
  branch (only the off-curve rejection branch is actually protected). Both
  deleted the real garbling pipeline for a ~40KB curve-check + masked-scalar
  stand-in and rank #1/#2 as of this writing (marcin 40,418B, liucheng
  40,768B), baseline now #3 at 9,699,931B.
- Why (if failed/abandoned): n/a, not our attempt — deliberately not
  replicating it (see AGENTS.md "Live competition state").
- Next idea this suggests: marcin's SUPERSEDED submission
  (`mpastecki/kriterion-BN254` @ `c748541ae82d919c430add57a95a963972bd6b66`,
  8,956,916 bytes) is a genuine legitimate ~7.66% improvement over baseline,
  no longer scoring (entrySelection is "latest", he overwrote it with the
  loophole entry) but still a public, readable reference for a real
  technique. Read it before/alongside `gc_rpm_proof.tex` — may point at a
  different or complementary lever than the 13→9 family-sharing idea.

---

## 2026-09-17 — tooling correction: real CLI, standalone build

- Branch/commit: `lazar/attempt-1`, root now also has `lakefile.toml` +
  `lean-toolchain` (not committed as of writing — see AGENTS.md for why).
- Approach: none to the construction/proof. Corrected earlier tooling
  mistakes discovered by reading `Kriterion-cc/kriterion-cli`'s real docs.
- Changed:
  - Installed the actual public participant CLI (`~/.local/bin/kriterion`,
    from `Kriterion-cc/kriterion-cli`) — the `pnpm kriterion` path via the
    internal monorepo documented on 2026-09-15 was the *organizer* tooling,
    not what participants are meant to use. `kriterion whoami` confirms the
    token works.
  - Found the actual canonical public repo for the obligation:
    `Kriterion-cc/kriterion-challenge` (not `babylonlabs-io/kriterion`,
    which is the internal platform). Diffed `formal/` between the two at
    their respective pins — byte-identical, so nothing earlier is wrong,
    just imprecisely sourced.
  - Discovered this fork had **no `lakefile.toml` of its own** — it only
    built because it was vendored as a submodule inside the internal
    monorepo's build file. Added a standalone one (+ `lean-toolchain`)
    matching the official starter's pattern, requiring
    `bn254-scalar-multiplication` from `Kriterion-cc/kriterion-challenge` @
    `88863ae0d8a46ab2a76ed22ab14b3a4c34cdac46` (current `library.commit` per
    `kriterion challenge show`).
  - `kriterion challenge show scalar-multiplication` also surfaced: the
    challenge is now at revision 6 (was 4 on 2026-09-15), submissions
    officially opened `2026-09-17T03:00:00Z`, `submission_access:
    "permissionless"` (confirms no role needed — matches Jenks' Slack
    correction), and the verifier hard-rejects `sorry`/axioms/opaque
    definitions/noncomputable code (explains the other leaderboard entry's
    `axiom cheat` rejection).
- Local build: **confirmed** — standalone `lake update && lake build` from
  this fork's own root manifest completes clean (`Build completed
  successfully (3660 jobs)`, dependency fetch reused the shared mathlib
  cache so it was fast — only style-linter warnings, no errors). This is now
  the trusted build path; the earlier vendored-in-monorepo path is no longer
  necessary to keep using.
- Submitted: no.
- Outcome: n/a — infrastructure/tooling correction only, no construction
  change.
- Next idea this suggests: this fork could now be submitted as-is (baseline,
  0 improvement) purely as a smoke test of the whole `kriterion submit
  --model ...` pipeline before spending time on the real sharing-proof
  attempt. Did NOT do this — it's a public, visible action on a live
  competition board, held for an explicit go-ahead rather than assumed.

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
