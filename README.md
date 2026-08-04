# Machine-checked soundness for circomlib Poseidon(2)

A Lean 4 proof that the circomlib `Poseidon(2)` circuit is **sound and complete** against its
reference specification, built in the [cLean](https://github.com/Verified-zkEVM/clean)
framework. The proof carries no Poseidon-specific axioms.

**The point.** Veridise's formal verification of Semaphore v3
([report](https://semaphore.pse.dev/Semaphore_3.0.0_Audit.pdf), Jan 2023, §5) marks seven
specifications Verified. Every one of those Coq proofs closes with
`pose proof Poseidon.PoseidonHypo.poseidon_2_spec` — *PoseidonHypo*, a hypothesis. Poseidon at
arity 2 is assumed there, not proved, and that assumption carries the nullifier hash and every
node of the Merkle inclusion proof.

This repository proves the assumed statement.

- **The result and its honest scope:** [`POSEIDON2_RESULT.md`](POSEIDON2_RESULT.md)
- **The proof:** `Clean/Circomlib/Poseidon.lean`, namespace `Circomlib.Poseidon.Poseidon2`
- **How the target was chosen:** [`POSEIDON_T3_PLAN.md`](POSEIDON_T3_PLAN.md)
- **The one remaining `sorry` (pre-existing, BN254 primality) and how to close it:**
  [`BN254_PRIMALITY.md`](BN254_PRIMALITY.md)

## What is proved

The whole circuit: initial ARK, three full rounds, the P-matrix transition round, **57 partial
rounds** with sparse mixing, three more full rounds, final mix. 495 witnesses. Each component
carries its own soundness *and* completeness proof, and the top-level statement says the
constraint system admits no witness other than the true hash.

Validated against the unoptimized reference permutation and against circomlibjs's published
test vector `poseidon([1,2])`.

## Build

```bash
lake exe cache get
lake build Clean.Circomlib.Poseidon    # 1736 jobs; one warning: the pre-existing line-22 sorry
```

Lean 4.32.2. Upstream cLean README preserved as `README-clean-upstream.md`.

---

Built on [Verified-zkEVM/clean](https://github.com/Verified-zkEVM/clean), whose t=2 development
provided the proof architecture this mirrors.
