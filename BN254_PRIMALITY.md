# Discharging the BN254 primality `sorry`

`Clean/Circomlib/Poseidon.lean:22` carries the only remaining `sorry` in the Poseidon
development:

```lean
instance : Fact (Nat.Prime BN254_PRIME) := ⟨by sorry⟩
```

It predates this work and is independent of the Poseidon soundness argument, but it is
worth closing: it is the last admitted statement in the file.

## Why the obvious routes fail

- `decide` / `native_decide`: the decidability instance for `Nat.Prime` is trial division to
  sqrt(p) ~ 1.5e38 operations. Not feasible.
- `norm_num`: attempted and fails. Its prime extension is also trial-division based and is
  intended for small numbers.

## The route that works: Lucas primality with a Pratt certificate

Mathlib provides (`Mathlib/NumberTheory/LucasPrimality.lean:39`):

```lean
theorem lucas_primality (p : ℕ) (a : ZMod p) (ha : a ^ (p - 1) = 1)
    (hd : ∀ q : ℕ, q.Prime → q ∣ p - 1 → a ^ ((p - 1) / q) ≠ 1) : p.Prime
```

### Certificate (computed and numerically verified)

p = 21888242871839275222246405745257275088548364400416034343698204186575808495617

p - 1 factors completely as:

```
2^28 · 3^2 · 13 · 29 · 983 · 11003 · 237073 · 405928799
     · 1670836401704629 · 13818364434197438864469338081
```

Witness: **a = 5**. Verified numerically:
- 5^(p-1) = 1 (mod p)
- 5^((p-1)/q) != 1 (mod p) for every prime factor q listed above

**Trap:** a = 3 satisfies the Fermat condition but *fails* the order condition, so it cannot
be used. This costs an afternoon if discovered late.

## Remaining proof obligations

1. `(5 : ZMod p) ^ (p - 1) = 1` — a modular exponentiation; `native_decide` handles this
   size comfortably since `ZMod p` arithmetic is efficient.
2. The factorization is complete: `2^28 * 3^2 * 13 * ... * 13818364434197438864469338081 = p - 1`
   — pure arithmetic, `norm_num` or `native_decide`.
3. Every listed factor is prime. Small ones fall to `norm_num`. The two large ones need
   their own Lucas certificates:
   - 1670836401704629
   - 13818364434197438864469338081
   This is the recursive part of a Pratt certificate and the bulk of the work.
4. Given 2 and 3, any prime q dividing p-1 must be one of the listed factors
   (`Nat.Prime.eq_of_dvd_of_prime` style reasoning over the factorization).
5. For each listed q, `(5 : ZMod p) ^ ((p - 1) / q) != 1` — ten modular exponentiations,
   each `native_decide`-sized.

## Estimate

Obligations 1, 2 and 5 are mechanical. Obligation 3 (the two nested certificates) and
obligation 4 (the enumeration argument) are the real work: plausibly a day for someone
comfortable in Mathlib, less if a Pratt-certificate tactic already exists in the ecosystem
worth searching for first.

This is a clean, self-contained contribution to the upstream repo, independent of Poseidon.
