# Machine-checked soundness for circomlib Poseidon(2)

**Result.** The circomlib `Poseidon(2)` circuit (two field inputs, state width t=3) is proved
sound and complete in Lean 4 against its optimized reference specification, in the
[cLean](https://github.com/Verified-zkEVM/clean) framework. The proof carries no
Poseidon-specific axioms.

**Why this specific circuit.** Veridise's formal verification of Semaphore v3
([audit report](https://semaphore.pse.dev/Semaphore_3.0.0_Audit.pdf), January 2023, Section 5)
marks seven specifications Verified — `CalculateSecret`, `CalculateIdentityCommitment`,
`CalculateNullifierHash`, the top-level `Semaphore`, `MerkleTreeInclusionProof`, `MultiMux1`,
and Poseidon determinism. Every one of those Coq proofs closes with:

```coq
pose proof Poseidon.PoseidonHypo.poseidon_2_spec.
```

`PoseidonHypo` is a *hypothesis*. Poseidon at arity 2 is assumed there, not proved — and that
assumption carries the nullifier hash and every node of the Merkle inclusion proof. This work
proves the assumed statement.

## What is proved

In `Clean/Circomlib/Poseidon.lean`, namespace `Circomlib.Poseidon.Poseidon2`:

| Component | Statement | Status |
|---|---|---|
| `ark_t3_eq`, `mix_matrix_t3_eq` | round-constant addition and 3x3 MDS mixing, closed form | proved |
| `InitialArk` | state `[0, in0, in1]` + `C_t3[0..2]` | soundness + completeness |
| `FullRound_t3` | 3 S-boxes, ARK, MIX(M) | soundness + completeness |
| `PartialRoundOpt_t3` | S-box on element 0, ARK, sparse MixS | soundness + completeness |
| `partialRoundsOpt_t3_induction` | an n-step state chain equals the spec fold | proved |
| `ApplyPartialRoundsOpt_t3` | all **57** partial rounds | soundness + completeness |
| `ApplyFullRounds_t3` | 3-round full-round fold | soundness + completeness |
| **`Poseidon2.circuit`** | **the whole circomlib `Poseidon(2)`** | **soundness + completeness** |

The top-level statement: for any environment satisfying the circuit's constraints, the output
equals `Specs.PoseidonOptimized.poseidon2Opt` of the inputs — i.e. the constraint system admits
no witness other than the true hash.

Circuit structure proved, matching circomlib: initial ARK, three full rounds, the P-matrix
transition round, 57 partial rounds with sparse mixing, three full rounds, final mix. 495
witnesses.

## Validation

The specification is anchored to the real hash, not merely to itself:

- `poseidon2Opt` agrees with the unoptimized reference `Specs.Poseidon.poseidon2` on tested
  inputs (`native_decide`).
- `poseidon2Opt #v[1, 2] = 0x115cc0f5e7d690413df64c6b9662e9cf2a3617f2743245519e19607a4417189a`,
  the published circomlibjs test vector.

## Honest scope

- The proof is against the **optimized** permutation (the structure circomlib implements),
  bridged to the reference specification by evaluation on test vectors rather than by a general
  equivalence theorem. Proving `poseidon2Opt = poseidon2` for all inputs is worthwhile follow-up
  work.
- One `sorry` remains in the file, at line 22: `Fact (Nat.Prime BN254_PRIME)`. It predates this
  work and is independent of the Poseidon argument. A Pratt certificate is recorded in
  `POSEIDON_T3_PLAN.md` (Lucas witness a = 5, with the full factorization of p-1); discharging it
  via `Mathlib.NumberTheory.LucasPrimality` is a self-contained follow-up.
- This proves the *circuit* sound relative to the *specification*. It says nothing about
  Poseidon's cryptographic strength, which is a separate question.
- Semaphore v4 replaced Poseidon identity commitments with EdDSA/BabyJubjub, dropped formal
  verification, and shipped two circuit-level Criticals in the new code
  ([v4 audit](https://semaphore.pse.dev/Semaphore_4.0.0_Audit.pdf)). This result applies to the
  v3 lineage and to any circuit depending on circomlib Poseidon at arity 2 — which remains
  widely deployed.

## Reproducing

```bash
git clone <this repo> && cd clean && git checkout poseidon-t3
lake exe cache get
lake build Clean.Circomlib.Poseidon      # 1736 jobs; one warning, the line-22 sorry
```

Environment: Lean 4.32.2, Mathlib via lake cache.
