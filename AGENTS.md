# Kriterion `scalar-multiplication` — working notes

This repo is a fork of the baseline submission (`SebastianElvis/argomac-lean`)
for the Kriterion challenge at <https://kriterion.cc/c/scalar-multiplication>.
Multiple agents/sessions work on this over time, not concurrently. Read this
file and [EXPERIMENTS.md](EXPERIMENTS.md) before trying anything — both to
avoid repeating a dead end and to see what's already partially working.

The verifier only reads `Submission.lean`, `Construction.lean`, `Proof.lean`,
`Construction/`, and `Proof/` from whatever commit is submitted. Everything
else at root (this file included) is free — it doesn't affect scoring and
stays visible in the public repo, which fits how Kriterion works (every
verified entry is meant to be a base the next person builds from).

## The challenge, in one paragraph

Build a computable garbled circuit for fixed-scalar multiplication on BN254
(`Submission.solution : Kriterion.Solution`), prove correctness, a constant
public byte layout, 100-bit adaptive privacy (two-stage game, VCV-io random
oracle model), Lamport compatibility (128-bit label pair per bit of `x || y`,
254 little-endian bits/coordinate), and HASH160-compatible encoded labels.
Score = `Kriterion.Benchmark.ciphertextBytes`, lower is better. Verified only
via Lean kernel (`leanprover/lean4:v4.33.1`, `mathlib v4.33.1`) — no partial
credit, it either type-checks against `Kriterion.Solution` or it's rejected.
**Constraint from `challenge.yaml`: you may use VCV-io primitives,
intermediate games, and reductions, but must not change the final security
game, public oracle access, or the work count (`q0 + q1 + q2 + 1`).**

Current record (= baseline, unbeaten): **9,699,931 bytes**, 0 verified
improvements as of 2026-09-15.

## Where things live

- This fork: `~/work/scalar-multiplication` (branch-per-attempt, e.g.
  `lazar/attempt-1`). Push attempts here.
- Kriterion platform monorepo (has the obligation + local verifier):
  `~/work/kriterion`, pinned to `origin/main` (`57f07dc` as of 2026-09-15).
  Challenge spec: `examples/bn254-scalar-multiplication/`.
- To verify locally: `examples/bn254-scalar-multiplication/argomac-lean` in
  the kriterion checkout is a **git submodule** pointing at
  `SebastianElvis/argomac-lean`. Point it at this fork/branch (or just
  `rsync`/copy this working tree over it) before running `lake build`.
- Lean toolchain: installed via `elan` (`~/.elan/bin`), pinned automatically
  by `lean-toolchain` (v4.33.1). First `lake update` + build pulls mathlib +
  VCVio and the mathlib oleans cache (~8,700 files) — slow once, fine after.

```sh
export PATH="$HOME/.elan/bin:$PATH"
cd ~/work/kriterion/examples/bn254-scalar-multiplication
lake build   # compiles Kriterion + Construction/Proof/Submission + tests
```

Confirmed working 2026-09-15: `lake build` completes clean (3666 jobs,
`Build completed successfully`) and reproduces the 9,699,931-byte baseline
end to end, including `Submission`/`Proof`/`Correctness`/`AdaptivePrivacy`.
Toolchain setup is a solved problem — don't redo it, just rebuild.

`Proof/CiphertextSize.lean`'s `ciphertextSize` theorem is the one that pins
the byte count — if you change the construction, that theorem's RHS literal
must change to match, and the proof needs re-deriving accordingly.

## Where the 9,699,931 bytes actually come from

Confirmed by reading `Proof/CiphertextSize.lean` + `Construction/ArgoMAC/Encoding.lean`:

```
9,699,931 = 40,736 (curve membership table, ~0.4%)  +  9,659,195 (pointMAC table, ~99.6%)
```

`pointMAC` = 91 output "digits" (`FieldMacToECMac.outputMacCount`) × 3
coordinate tables (X, Y, Z). Each coordinate table (`Biquadratic.Table`) has
6 optional field coefficients (33 bytes if `Some`, 1 if `None`) and 5
optional "digit adaptor" vectors — `x7, x9, y6, y8, y10` — each costing
**8,129 bytes if `Some`** (254 bit-positions × 32-byte garbled row + 1 tag
byte), 1 byte if `None`.

Per row, `garbleX` populates 4/5 digit vectors, `garbleY` populates 4/5,
`garbleZ` populates 5/5 → **13 populated digit-adaptor tables per row** out
of 15 possible. `91 × 13 × 8,129 ≈ 9.6M` — that's essentially the whole
ciphertext.

## The concrete opportunity (and why baseline didn't take it)

This repo's own README has a "Paper correspondence" table: the BaBe paper
needs only **9** point-adaptor families per bucket (shared across X/Y/Z via
a tweak); this entry needs **13** (see above). Closing that gap alone:
`(13 − 9) × 91 × 8,128 ≈ 2.96M bytes` → **~6.74M**, a verified win over
baseline if it holds.

I checked whether the 4 extra copies are free-to-dedupe duplicates — **they
are not**, as currently coded. `Construction/ArgoMAC/FieldMacToECMac.lean`
gives X, Y, Z fully independent `Biquadratic.Oracles` and randomness
(`RowOracles.x/y/z`, `RowRandomness.x/y/z` are separate structs; the shared
field names like `r3`/`r5` are coincidental naming, not shared values). So
each coordinate's `y6`/`y8`/etc. table is a genuinely distinct garbled
object today.

**Why**: the paper shares one garbled table across coordinate uses via block
formula `π(L⊕t) ⊕ (L⊕t)` (see `BitAdaptor.lean`'s Davies–Meyer construction
for the entry's current formula, `π(L⊕t) ⊕ L`, which drops the `⊕t`
feed-forward). This repo's own README states plainly why they didn't use
the paper's formula:

> "One shared translation cannot equate the formulas for two different
> tweaks and every label... The repository therefore proves this variant
> directly. It does not claim that the two shared-bucket constructions have
> identical distributions."

**So the highest-leverage target is**: prove that sharing one garbled digit
table across coordinate uses (via a tweak) preserves the required adaptive-
privacy distribution — either by switching to the paper's
`π(L⊕t) ⊕ (L⊕t)` formula and redoing the paper's own bucket-sharing
argument (`gc_rpm_proof.tex` in BaBe.latex), or finding another formula/
argument that does formalize. This is a real proof-engineering problem, not
a quick patch — see `Proof/Privacy/PaperConstruction.lean` for the closest
existing attempt and exactly where it stops short.

Secondary/fallback ideas if that's a dead end (unexplored, no evidence
either way yet): tag-byte overhead in the `Option` encoding, whether all 91
output digits are load-bearing, curve-table encoding slack (small, ~0.4%
of total, low priority).

## Submitting

Via the `kriterion` CLI in the platform monorepo (`~/work/kriterion`), not a
web form:

```sh
source ~/.config/kriterion/env.sh   # sets KRITERION_API + KRITERION_TOKEN (local-only, not in any git repo)
cd ~/work/kriterion
pnpm install   # first time only
pnpm kriterion submit \
  --challenge scalar-multiplication \
  --repo https://github.com/Lazar955/argomac-lean \
  --commit <full 40-char commit hash on this fork>
```

Account status as of 2026-09-16: GitHub connected (`lazar955`), a CLI token
exists (labeled `cli`, stored in `~/.config/kriterion/env.sh`). The
settings page still shows "You have no role yet. A challenge organizer can
invite you" — per Jenks (babylonlabs.io, Slack, 2026-09-16) that's a stale
"ghost process" left in the UI and does **not** actually gate submission.
No invite needed; the CLI command above should just work with the token.

Freezes that exact commit; a verifier run checks the proof and measures
`ciphertextBytes`. Only push/submit a commit that builds clean locally first
— log every attempt in [EXPERIMENTS.md](EXPERIMENTS.md) regardless of
outcome.
