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
**Hard verifier rule (from the starter README): it rejects `sorry`, axioms,
opaque definitions, and noncomputable construction code outright** — this is
exactly why the leaderboard's other entry got REJECTED (used `axiom cheat`
instead of a constructed value). No amount of local `lake build` success
means anything if the submission leans on any of these; the local Lean
compiler is more permissive (accepts `sorry`) than the hosted axiom check.

Current record (= baseline, unbeaten): **9,699,931 bytes**, 0 verified
improvements as of 2026-09-17.

Kriterion's own `authorPrompt` (served by the API, meant to steer an AI agent
acting for a participant — i.e. exactly this workflow) says, verbatim: *"Do
not improve a metric by weakening correctness, security, or another
criterion. Do not submit production implementation code."* Treat that as
binding — it's the platform's own framing of scope, not something to argue
with.

## Where things live

- This fork: `~/work/scalar-multiplication` (branch-per-attempt, e.g.
  `lazar/attempt-1`), forked from `SebastianElvis/argomac-lean`. Push
  attempts here — `github.com/Lazar955/argomac-lean`.
- **This fork now has its own `lakefile.toml` + `lean-toolchain` at root**
  (added 2026-09-17) so it builds standalone, exactly as the hosted verifier
  will build it — it did NOT have these originally (baseline repos are meant
  to be vendored into a host project). The `lakefile.toml` requires
  `bn254-scalar-multiplication` from the canonical public challenge repo,
  pinned to the spec's current `library.commit` (check
  `kriterion challenge show scalar-multiplication` for the live value — it
  moves as organizers revise the challenge; was `88863ae0d8a46ab2a76ed22ab14b3a4c34cdac46`
  as of revision 6 / 2026-09-17). **Re-pin this if the CLI reports a newer
  `library.commit`** — confirmed via `git log` that at least the `formal/`
  obligation itself hasn't changed across recent revisions, but don't assume
  that holds forever.
- Canonical public challenge repo (obligation + starter, NOT where you work):
  `~/work/kriterion-challenge` (`Kriterion-cc/kriterion-challenge`). Its
  `formal/` directory is byte-identical to the internal platform monorepo's
  copy (diffed directly, 2026-09-17) — so everything derived from the
  monorepo checkout below is still accurate, just also cite this repo since
  it's what's actually public/authoritative.
- Internal platform monorepo (only needed if you want the *hosted* dev
  server, not for verifying a submission): `~/work/kriterion`
  (`babylonlabs-io/kriterion`), pinned to `origin/main` (`57f07dc` as of
  2026-09-15). Its `examples/bn254-scalar-multiplication/argomac-lean` is a
  git submodule of the baseline — useful as a second build path to
  cross-check, not required for submitting.
- Lean toolchain: installed via `elan` (`~/.elan/bin`), pinned automatically
  by `lean-toolchain` (v4.33.1). First `lake update` + build pulls mathlib +
  VCVio and the mathlib oleans cache (~8,700 files) — slow once, fine after.

```sh
export PATH="$HOME/.elan/bin:$PATH"
cd ~/work/scalar-multiplication   # this fork, standalone — matches what gets submitted
lake update   # first time / after re-pinning bn254-scalar-multiplication
lake build    # compiles Construction/Proof/Submission
```

Confirmed working end-to-end 2026-09-15 (via the vendored path in
`~/work/kriterion`): `lake build` completes clean (3666 jobs,
`Build completed successfully`) and reproduces the 9,699,931-byte baseline,
including `Submission`/`Proof`/`Correctness`/`AdaptivePrivacy`. Toolchain
setup itself is a solved problem — don't redo it. The *standalone* build via
this fork's own new `lakefile.toml` was added 2026-09-17 and should be
re-verified before relying on it (check `EXPERIMENTS.md` for the result).

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

## Live competition state (2026-09-17, check `kriterion board scalar-multiplication` for current)

This is Babylon Labs' internal "Kriterion Pilot Challenge," $500 pool,
announced in Slack (`C079KTWQKBL`, thread `1789488716.857219`), went live
2026-09-17T05:00 CEST. **A formal-definition loophole was found and exploited
within ~25 minutes of launch, confirmed by two independent people:**

- The adaptive-privacy game hands the simulator the *true output* and never
  bounds its running time. On a prime-order curve, `s·P` determines `s`
  uniquely from one valid pair, so an unbounded simulator can always
  brute-force-recover `s` and "match" the real distribution — the formalized
  privacy notion is vacuous on the valid-input branch (only actually
  enforced on the off-curve rejection branch). Root cause per Runchao Han:
  "the problem statement permits the simulator to be non-polynomial-time,
  which is not expected."
- Exploiting this, you can delete the entire garbled-multiplication pipeline
  and replace it with a tiny curve-membership gadget + a masked scalar
  pass-through — passes all 5 checks legitimately (no `sorry`/axiom, doesn't
  touch the security game), and wins on ciphertext bytes by ~99.6%.
- Board as of 2026-09-17 (`ciphertext_bytes`, lower ranks first):
  1. `marcin@babylonlabs.io`, **40,418 bytes** — same loophole
     (`Kriterion.TruncatedDirectDisclosure`)
  2. `liucheng@babylonlabs.io`, 40,768 bytes — the original exploit from the
     Slack thread
  3. baseline (`SebastianElvis/argomac-lean` @ `711689c`), 9,699,931 bytes
- **`entrySelection: "latest"` — only your most recent submission counts.**
  Marcin's earlier submission (`mpastecki/kriterion-BN254` @
  `c748541ae82d919c430add57a95a963972bd6b66`, **8,956,916 bytes, a genuine
  ~7.66% legitimate improvement**, review cites `Wire.garble_length`,
  `RCBComplete.perfectCorrectness`, `Security.concreteCircuitSimulator`) no
  longer scores — he overwrote it with the loophole entry 8 seconds later —
  but the commit is still public and is a live lead on a real technique that
  beats baseline, worth reading before/alongside the paper's sharing
  argument. It's independent evidence that more than the 13→9 family
  reduction alone may be available.

Team consensus in the thread: liucheng should be awarded under current
rules, and the statement will likely get patched (unbounded-simulator /
missing-blinding gap closed) — Runchao is already working a fix. **Do not
chase the loophole** — it's already claimed twice, adds no research value,
and will likely stop scoring once patched. Keep pursuing the legitimate
13→9 sharing-gap closure (see above); re-check `kriterion challenge show
scalar-multiplication`'s `revision` field before investing serious proof
effort, in case the statement changes underneath this work.

## Submitting

**Correction (2026-09-17): ignore any earlier note about `pnpm kriterion`
inside the platform monorepo — that's the internal/organizer path.** The
real *participant* path is the public standalone CLI from
[`Kriterion-cc/kriterion-cli`](https://github.com/Kriterion-cc/kriterion-cli),
already installed here at `~/.local/bin/kriterion` (universal Node script,
works with Node ≥18, no monorepo checkout needed):

```sh
source ~/.config/kriterion/env.sh   # sets KRITERION_API + KRITERION_TOKEN (local-only, not in any git repo)
kriterion whoami                    # sanity check auth
COMMIT=$(git -C ~/work/scalar-multiplication rev-parse HEAD)
kriterion submit \
  --challenge scalar-multiplication \
  --repo https://github.com/Lazar955/argomac-lean \
  --commit "$COMMIT" \
  --model claude-sonnet-5
kriterion submission show <submissionId-from-output>
kriterion board scalar-multiplication
```

**Use `--model <name>` whenever an AI helped write the solution** (per the
CLI docs: *"Use this command instead when an AI model helped create the
solution. Do not run both submission commands for the same commit."*) — this
work is Claude-assisted, so every real submission from this repo should
carry `--model`, not the bare `submit`. Exact string TBD (not yet confirmed
if Kriterion validates against a fixed model list) — use `claude-sonnet-5`
unless a submission gets rejected specifically for that field, in which case
check `docs/reference/cli.md` in `Kriterion-cc/kriterion-cli` for an
updated value.

Account status as of 2026-09-17: GitHub connected (`lazar955`), CLI token
live (`kriterion whoami` succeeds), `submission_access: "permissionless"`
confirmed directly from `kriterion challenge show scalar-multiplication` —
the settings page's "You have no role yet" text is a known stale UI
artifact (confirmed by Jenks, babylonlabs.io, Slack, 2026-09-16) and does
**not** gate submission. Submissions for this challenge opened
`2026-09-17T03:00:00Z` (already open as of this writing).

Freezes that exact commit; a verifier run checks the proof and measures
`ciphertextBytes`. Exit codes: `0` success, `1` usage/network error, `2`
policy/schema rejection (fix and resubmit, don't retry unchanged), `3` not
ready yet, `4` auth failure. `evaluation.state` on `submission show` is one
of `pending`/`running`/`rejected`/`complete` — read `evaluation.reason` on
`rejected`. Only submit a commit that builds clean locally first (`lake
build`, no `sorry`/axioms/opaque/noncomputable — see the hard verifier rule
above) — log every attempt in [EXPERIMENTS.md](EXPERIMENTS.md) regardless of
outcome, submitted or not.
