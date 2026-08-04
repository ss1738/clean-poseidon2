# Poseidon t=3 (arity 2) verification — working plan

## Why this target
Veridise's formal verification of Semaphore v3 (Jan 2023, 16 person-weeks) marks 7 specs
"Verified" — but every proof closes with `Poseidon.PoseidonHypo.poseidon_2_spec`.
Poseidon at arity 2 is **axiomatized, not proved**. That axiom carries the nullifier hash
and every node of the Merkle inclusion proof. Discharging it hardens the most-cited
formally-verified circuit in the Circom ecosystem.

Source: https://semaphore.pse.dev/Semaphore_3.0.0_Audit.pdf (Section 5, Formal Verification)

## State of the repo (Verified-zkEVM/clean, branch poseidon-t3)
- `Clean/Specs/Poseidon.lean` — reference spec `poseidon2` (arity 2, t=3) exists, line 145,
  with a circomlibjs test vector at line 226. Constants `C_t3`, `M_t3` present.
- `Clean/Specs/PoseidonOptimized.lean` — `poseidon2Opt` (line 210), `mixS_t3` (line 40),
  `P_t3`, `S_t3` present.
- `Clean/Circomlib/Poseidon.lean` — 479 lines, complete circuit + soundness/completeness
  chain for **t=2 only** (arity 1): Sigma, Ark, Mix, MixS, MixLast, FullRounds,
  PartialRounds (with `partialRoundsOpt_t2` induction), PoseidonEx, Poseidon.
  **Zero t=3 content.** This is the gap.

## Environment (verified working, 2026-08-04)
- elan + Lean 4.32.2, Mathlib cache fetched
- `lake build Clean.Circomlib.Poseidon` → 1736 jobs, ~4s incremental
- Only warning: the `sorry` at line 22 (BN254 primality)

## Side quest: discharge the BN254 primality `sorry`
`norm_num` cannot do a 254-bit primality proof (attempted, fails).
Correct route is `Mathlib.NumberTheory.LucasPrimality.lucas_primality` with a Pratt
certificate. Certificate computed and checked numerically:

- p = 21888242871839275222246405745257275088548364400416034343698204186575808495617
- p - 1 = 2^28 · 3^2 · 13 · 29 · 983 · 11003 · 237073 · 405928799
          · 1670836401704629 · 13818364434197438864469338081
  (factorization verified complete)
- Lucas witness **a = 5**: 5^(p-1) = 1 (mod p), and 5^((p-1)/q) ≠ 1 for every prime
  factor q above. (Note: a = 3 satisfies Fermat but fails the order condition — do not
  use 3.)
- Remaining work: each factor needs its own primality proof; the largest,
  13818364434197438864469338081, needs a nested certificate.

## Progress (2026-08-04)
Done and building (`lake build Clean.Circomlib.Poseidon`, 1736 jobs, no errors):
- `ark_t3_eq`, `mix_matrix_t3_eq` — foundational rewriting lemmas
- `InitialArk` — state [0, in0, in1] + C[0..2], soundness + completeness
- `FullRound_t3` — 3x S-box -> ARK -> MIX(M), soundness + completeness
- `PartialRoundOpt_t3` — S-box on element 0 -> ARK -> sparse MixS, soundness + completeness
- `partialRoundsOpt_t3_induction` — the chain lemma for the 57-round fold
- `ApplyPartialRoundsOpt_t3` — all 57 partial rounds folded, sound via the induction lemma

Remaining: ApplyFullRounds_t3 (fold of 3 full rounds, twice), the P_t3 transition round
(reuses FullRound_t3 with matrix P_t3), MixLast (final S-box + mix with no ARK), and the
top-level `Poseidon2` assembling all of it against `poseidon2Opt`, then the bridge to
`Specs.Poseidon.poseidon2` and the circomlibjs test vector.

## Main work plan
1. Mirror the t=2 chain at t=3. Order: Sigma (reuse) → Ark_t3 → Mix_t3 → MixS_t3 →
   MixLast_t3 → FullRounds_t3 → PartialRounds_t3 (the induction lemma is the hard part;
   pattern-match `partialRoundsOpt_t2`, line 357) → PoseidonEx_t3 → Poseidon2.
2. Soundness target: circuit output = `Specs.PoseidonOptimized.poseidon2Opt`, then bridge
   to `Specs.Poseidon.poseidon2` via the existing optimized↔reference equivalence.
3. Validate against the circomlibjs test vector already in the spec file.

## Deliverable
A public branch/PR with `Poseidon2` proved sound, plus a short write-up stating exactly
which axiom in the Semaphore verification it discharges. That artifact is simultaneously
the EF ESP application and the introduction to Veridise / Nethermind / zkSecurity.
