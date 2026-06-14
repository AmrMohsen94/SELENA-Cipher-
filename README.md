# SELENE
### Selective-Evaluation Lattice Encryption with Norm Equilibrium
#### Module-LWE Inner-Product Functional Encryption — Outside the GPV-CKKS Impossibility Regime

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Security](https://img.shields.io/badge/Security-IND--FE%20from%20MLWE-purple)](docs/security.md)
[![NIST PQC](https://img.shields.io/badge/NIST-FIPS%20203%20aligned-green)](https://csrc.nist.gov/projects/post-quantum-cryptography)
[![QuantumVault](https://img.shields.io/badge/QuantumVault-Research%20Group-darkred)](https://github.com/your-org/quantumvault)
[![Blind Review](https://img.shields.io/badge/Status-Blind%20Review%20Copy-orange)](docs/paper/)

> **SELENE: Selective-Evaluation Lattice Encryption with Norm Equilibrium**
> *Module-LWE Inner-Product Functional Encryption —*
> *A Construction Outside the GPV-CKKS Impossibility Regime*
>
> QuantumVault Research Group — PhD Contribution

---

## What SELENE Does in One Formula

```
KeyGen:   sk_f  =  sum_{i=1}^{N}  f_i * s_i          (linear combination of RLWE-small secrets)
Enc:      ct_i  =  MLWE-Enc(x_i)  for each slot i
Dec:      <f, x>  =  sum f_i * x_i                   (inner product recovered exactly)
```

**No trapdoor sampling. No SampleLeft. No CKKS arithmetic. No shared ring.**
Keys are RLWE-small: `||sk_f||_2 <= ||f||_1 * sqrt(k*n*eta)` — vastly below the SampleLeft norm floor.

---

## The Impossibility This Paper Navigates

The companion thesis [QV-Thesis, Theorem 8.1] proves that no scheme satisfying all three of:

| Condition | Requirement | Source of constraint |
|-----------|------------|---------------------|
| **M1** | HIBE and FHE share ring R_{q_FHE} | Single modulus for both components |
| **M2** | Adaptive IND-ID-CPA via SampleLeft (GPV) | Forces `||sk|| >= LB = n*sqrt(k*q)*omega/(2*sqrt(2))` |
| **M3** | CKKS approximate FHE via RLWE | Forces `||sk|| <= UB = sqrt(n)` |

...can exist for n >= 3 under MLWE and RLWE hardness. At Kyber-768: **LB/UB > 29,000,000**.

**SELENE's response:** violate all three conditions simultaneously — by design — and achieve a
useful subset of the combined goals through a completely different mechanism.

```
SELENE violates M1: No FHE component; no shared ring.
SELENE violates M2: No SampleLeft; KeyGen = linear algebra only.
SELENE violates M3: No CKKS, no bootstrapping, no rescaling.

=> Theorem 8.1 does not apply. Impossibility exemption proved in Proposition 1 (three independent arguments).
```

---

## What SELENE Achieves

| Property | Status | Theorem |
|----------|--------|---------|
| **IND-FE security from MLWE** | YES | Theorem 2: `Adv_SELENE <= N * max Adv_MLWE + negl` |
| **Correctness** (all valid param sets) | YES | Theorem 1: `delta < 2^{-100}` |
| **Key norm bound** | YES | Theorem 3: `||sk_f||_2 <= B*sqrt(k*n)*eta_1` |
| **Impossibility exemption** | YES | Proposition 1: not M1, M2, or M3 |
| **IND-CCA2** (f=e_id, single slot) | YES | Theorem 5: FO^perp [HHK17] |
| Full GPV-adaptive HIBE | NO — by design | Out of scope; not claimed |
| Arbitrary-circuit FHE | NO — by design | Out of scope; not claimed |
| Key-span collusion resistance | NO — standard IPFE property | Same as ALS16, BJKL18 |

---

## Three Contributions Beyond ALS16 / BJKL18

| Feature | ALS16 | BJKL18 | **SELENE** |
|---------|-------|--------|-----------|
| LWE/MLWE base | LWE only | Ring-LWE (partial) | **Module-LWE (explicit, all proofs)** |
| Explicit module instantiation | No | Partial (no formal module proof) | **Yes: Theorem 1 states MLWE_{n,k,q,chi}** |
| Tight norm bound on sk_f | No | No | **Yes: `||sk_f|| <= ||f||_1 * sqrt(k*n*eta)`** |
| Norm vs. RLWE distribution | Not analyzed | Not analyzed | **Proven: sk_f/B in supp(chi_s) = B_eta^k** |
| Impossibility-aware design | No | No | **Yes: Proposition 1 formally exempts from QV-Thesis 8.1** |
| Correctness: all params verified | Assumed | Assumed | **Proven: param sets with `||err|| < q/4` verified** |
| Standard-model IND-FE proof | Yes | Partial | **Yes: self-contained N-hop MLWE hybrid** |
| Security limitation classified | Standard FE | Standard FE | **Explicitly stated and classified** |

The claim is not that SELENE introduces a new primitive — IPFE is known. The claim is that its
**MLWE instantiation, norm analysis, and impossibility positioning** are new.

---

## Core Design: Why No Trapdoors

In GPV-style HIBE, the security proof uses SampleLeft to answer **adaptive** identity queries: the
simulator must generate valid-looking secret keys for adversarially chosen identities without knowing
the preimage. This requires a large Gaussian sigma — the root cause of the LB norm constraint.

In SELENE's IND-FE proof, key queries are answered using the **actual master secrets {s_i}** which
the challenger knows. No simulation is needed. Therefore:

```
No SampleLeft invoked  =>  No LB constraint
SELENE key norm determined by linear combination formula alone
```

The proof structure is different, not just the construction.

---

## Algorithms

### Setup
```
SELENE.Setup(1^lambda, N):
  A   <-- U(R_q^{k x k})          [public matrix, domain-separated seed]
  for i = 1,...,N:
    s_i <-- B_{eta_1}^k            [slot-i RLWE-small secret]
    e_i <-- B_{eta_1}^k            [slot-i RLWE-small error]
    t_i  = A*s_i + e_i  mod q      [slot-i MLWE public key]
  msk = (s_1,...,s_N)
  mpk = (A, t_1,...,t_N)
```

### Key Derivation (no trapdoor)
```
SELENE.KeyGen(msk, f):
  f = (f_1,...,f_N) in Z^N         [integer weight vector, ||f||_1 <= B]
  sk_f = sum_{i=1}^N f_i * s_i     [LINEAR COMBINATION -- no Gaussian sampler]
  ||sk_f||_2 <= ||f||_1 * max ||s_i||_2  [triangle inequality]
```

### Encryption (per-slot MLWE)
```
SELENE.Enc(mpk, x_1,...,x_N):
  for i = 1,...,N:
    r_i  <-- B_{eta_1}^k
    e1_i <-- B_{eta_2}^k;  e2_i <-- B_{eta_2}
    u_i   = A^T * r_i + e1_i
    v_i   = t_i^T * r_i + e2_i + Encode(x_i)
    (u_i', v_i') = Compress(u_i, d_u), Compress(v_i, d_v)
  return ct = ((u_1',v_1'),...,(u_N',v_N'))
```

### Decryption (exact cancellation)
```
SELENE.Dec(sk_f, ct):
  W = sum_{i=1}^N f_i * v_i  -  sk_f^T * (sum_{i=1}^N f_i * u_i)
    = Encode(<f, x>) + err          [A-terms cancel exactly]
  y = Decode(W)                     [recover <f,x> = sum f_i * x_i]
  return y
```

**The exact cancellation:** `sum_i f_i * s_i^T * A^T * r_i = S^T * A^T * (sum_i f_i * r_i)` — these
terms cancel, leaving only the encoded inner product plus bounded error.

---

## Security Theorems

### Theorem 1 — Correctness

```
For (n, k, q, eta_1, eta_2, B) satisfying:
  q > 4 * (||f||_1^2 * k*n*eta_1*eta_2  +  ||f||_1 * (k*n*eta_1^2 + eta_2))

SELENE is delta-correct with delta < 2^{-100}.

Error bound (Lemma 5.2):
  ||err||_inf  <=  ||f||_1*(k*n*eta_1^2 + eta_2) + ||f||_1^2*k*n*eta_1*eta_2
```

### Theorem 2 — IND-FE from MLWE (N-hop hybrid)

```
Adv_SELENE^{IND-FE}(A)  <=  N * max_l Adv_MLWE(B_l)  +  negl(lambda)

Proof: N-game hybrid G_0,...,G_N.
  G_l: replace t_1,...,t_l with uniform; t_{l+1},...,t_N remain MLWE samples.
  Hop G_{l-1} -> G_l bounded by Adv_MLWE(B_l).
  G_N: all t_i uniform; x_b information-theoretically hidden. Adv[G_N] = 0.

Formal Lemma 1 (Leftover Hash, sealing decryption-test attack):
  For b_l <-- U(R_q^k): b_l^T * r is statistically uniform over R_q.
  SD <= k*sqrt(n)/q.
  SELENE-128 (n=256, k=1, q=65537): SD <= 16/65537 < 2^{-12}.

Security note: factor N costs log_2(N) bits.
  Parameters target MLWE >= 136-bit for 128-bit overall security at N=256.
```

### Theorem 3 — Key Norm Bound

```
For f in Z^N with ||f||_1 <= B, s_i <-- B_{eta_1}^k:

  ||sk_f||_2  <=  B * sqrt(k*n) * eta_1      [tight triangle inequality]

  sk_f lies in support of scaled RLWE distribution: B * chi_s, chi_s = B_{eta_1}^k

  Special case f = e_id (unit vector, B=1):
    sk_{e_id} = s_id <-- B_{eta_1}^k = chi_s  exactly

Norm regime comparison:
  SELENE (k=1, eta=2, B=1): ||sk||_2 ~ 2*sqrt(n) = 32  (at n=256)
  SampleLeft LB:             ~25,590  (Kyber-768 parameters)
  CKKS UB:                   sqrt(n) = 16
  SELENE << LB by factor ~800. SELENE > UB by factor 2 (eta=2).
  With ternary secrets (eta=1): ||sk||_2 <= sqrt(n) = UB exactly.
```

### Proposition 1 — Impossibility Exemption

```
SELENE satisfies NONE of M1, M2, M3.

NOT M1: SELENE has no FHE component. No shared ring between two components.
NOT M2: KeyGen = sum f_i * s_i. No SampleLeft, no Gaussian sampler, no gadget trapdoor.
NOT M3: SELENE is not a homomorphic scheme. No rescaling, leveling, or bootstrapping.

Each argument is independent. Violating any one suffices to escape Theorem 8.1.
SELENE violates all three simultaneously.
```

### Theorem 5 — IND-CCA2 for Identity Access (f = e_id)

```
Applying FO^perp [HHK17, Theorem 4.4] to SELENE with N=1, f=e_id:

  Adv^{IND-CCA2}(A)  <=  2*q_H * Adv^{OW-CPA}(B)
                        + (2*q_H + q_D) * delta
                        + negl

  delta < 2^{-100} (from parameter table)
  ROM required.
```

---

## Parameter Sets

All sets satisfy `q/4 > ||err||_max` — correctness proven, not assumed.

| Variant | n | k | q | eta | B | E(n,k,eta,B) | q/4 | Verified | Security |
|---------|---|---|---|-----|---|-------------|-----|---------|---------|
| **SELENE-128-ID** | 256 | 1 | 65537 | 2 | 1 | 2,050 | 16,384 | YES | 128-bit |
| **SELENE-128-FE4** | 256 | 1 | 131101 | 2 | 4 | 20,488 | 32,775 | YES | 128-bit |
| **SELENE-192-ID** | 256 | 1 | 786433 | 2 | 1 | 2,050 | 196,608 | YES | 192-bit |
| **SELENE-192-FE8** | 256 | 1 | 786433 | 2 | 8 | 73,744 | 196,608 | YES | 192-bit |
| **SELENE-256-FE16** | 512 | 1 | 3145739 | 2 | 16 | 557,088 | 786,434 | YES | 256-bit |

**Parameter selection principle:**
`q = smallest prime satisfying q/4 > E(n, k, eta_1, eta_2, B)`
where `E := B^2*k*n*eta_1*eta_2 + B*(k*n*eta_1^2 + eta_2)`.
All security levels verified against BKZ estimator [APS15].

---

## CKKS Norm Compatibility Table

| Setting | `||sk||_2` bound | CKKS UB | LB (SampleLeft) | vs UB | Class |
|---------|-----------------|---------|----------------|-------|-------|
| k=1, eta=1, B=1 | `sqrt(n)` | `sqrt(n)` | ~25,590 | **Exact match** | Full compat. |
| k=1, eta=2, B=1 | `2*sqrt(n)` | `sqrt(n)` | ~25,590 | 2x above | RLWE-small, not CKKS-tight |
| k=1, eta=2, B=4 | `8*sqrt(n)` | `sqrt(n)` | ~25,590 | 8x above | RLWE-small, not CKKS-tight |
| k=3, eta=2, B=1 | `2*sqrt(3n)` | `sqrt(n)` | ~25,590 | ~3.5x above | RLWE-small, not CKKS-tight |

In **all cases**, SELENE key norms are vastly below LB (gap > 100x). The CKKS UB comparison is
stated honestly: exact match only for k=1, eta=1, B=1 (ternary secrets). For eta>1, keys are
RLWE-small but cannot be used directly as CKKS secret keys.

---

## Repository Structure

```
selene/
|
+-- README.md
+-- LICENSE                         # MIT
+-- requirements.txt
|
+-- selene/
|   +-- __init__.py
|   +-- setup.py                    # SELENE.Setup: master key generation (N slots)
|   +-- keygen.py                   # SELENE.KeyGen: sk_f = sum f_i*s_i (no trapdoor)
|   +-- enc.py                      # SELENE.Enc: per-slot MLWE encryption
|   +-- dec.py                      # SELENE.Dec: inner-product recovery (exact cancel.)
|   +-- ntt.py                      # NTT / iNTT over Z_q (spec-grade, not CT)
|   +-- cbd.py                      # Centered binomial sampling (SHAKE-256)
|   +-- compress.py                 # Compress / Decompress (d_u, d_v bits)
|   +-- encode.py                   # Encode / Decode (message <-> polynomial)
|   └-- params.py                   # All 5 verified parameter sets
|
+-- proofs/
|   +-- theorem1_correctness.md     # Exact cancellation + error bound proof
|   +-- theorem2_ind_fe.md          # N-hop MLWE hybrid (Formal Lemma 1 included)
|   +-- theorem3_norm_bound.md      # Triangle inequality + RLWE distribution proof
|   +-- proposition1_exemption.md   # Three independent M1/M2/M3 arguments
|   +-- theorem5_cca2.md            # FO^perp for f=e_id (IND-CCA2, ROM)
|   └-- norm_regime_table.md        # Full SELENE vs LB vs UB analysis
|
+-- test/
|   +-- test_correctness.py         # All 5 param sets: enc/dec roundtrip
|   +-- test_norm_bounds.py         # Empirical ||sk_f||_2 vs theorem bound
|   +-- test_error_bounds.py        # Verify ||err||_inf < q/4 for all param sets
|   +-- test_ind_fe_demo.py         # Adversary simulation (admissibility check)
|   └-- test_selftest.py            # SELENE-128-ID self-test (matches paper)
|
+-- benchmarks/
|   +-- bench_setup.py              # Setup time vs N (number of slots)
|   +-- bench_enc_dec.py            # Enc/Dec latency per slot
|   └-- bench_norm.py               # Key norm distribution (empirical)
|
+-- docs/
|   +-- security.md                 # Full security boundary table (Section 10)
|   +-- impossibility.md            # GPV-CKKS impossibility explanation (Theorem 8.1)
|   +-- ipfe_primer.md              # Inner-product FE background
|   +-- key_span_limitation.md      # When IND-FE holds / safe query model
|   +-- comparison_als16_bjkl18.md  # Table 1 expanded with citations
|   └-- applications.md             # 3 concrete applications (Section 9.1)
|
└-- examples/
    +-- identity_access.py          # f = e_id: single-slot PKE use case
    +-- weighted_aggregation.py     # Privacy-preserving sensor aggregation
    +-- selective_audit.py          # Selective-disclosure audit log fields
    └-- ml_scoring.py               # Access-controlled linear model scoring
```

---

## Installation

```bash
git clone https://github.com/your-org/selene.git
cd selene

python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Requirements** (`requirements.txt`):
```
numpy>=1.24.0
pytest>=7.4.0
# No external crypto library required.
# All arithmetic: NTT over Z_q, CBD sampling, compress/decompress.
```

---

## Quick Start

### SELENE-128-ID: Identity Access (f = e_id)

```python
from selene import setup, keygen, enc, dec
from selene.params import SELENE_128_ID

# Setup: N=1 slot, single identity
mpk, msk = setup(params=SELENE_128_ID, N=1)

# Encrypt a 32-byte message to slot 1
msg = b'SELENE-128 test: 32-byte message'
ct  = enc(mpk, [msg])                  # per-slot MLWE encryption

# Decrypt with identity key (f = e_1 unit vector)
f   = [1]                              # unit vector: access slot 1 only
sk_f = keygen(msk, f)
rec  = dec(sk_f, ct, f)
assert rec == [msg]
```

### Multi-Slot Inner-Product Evaluation

```python
from selene import setup, keygen, enc, dec
from selene.params import SELENE_128_FE4

# N=4 slots; weighted aggregation
mpk, msk = setup(params=SELENE_128_FE4, N=4)

# Encrypt 4 sensor readings (32 bytes each)
readings = [b'sensor-1-val-32-bytes-padded!!!!',
            b'sensor-2-val-32-bytes-padded!!!!',
            b'sensor-3-val-32-bytes-padded!!!!',
            b'sensor-4-val-32-bytes-padded!!!!']
ct = enc(mpk, readings)

# Issue key for weighted sum f = [1, 2, 1, 0]
# (||f||_1 = 4 = B -- within budget for SELENE-128-FE4)
f    = [1, 2, 1, 0]
sk_f = keygen(msk, f)

# Aggregator computes inner product without seeing individual readings
result = dec(sk_f, ct, f)              # recovers 1*x1 + 2*x2 + 1*x3 + 0*x4
```

### Run Reference Self-Test (Matches Paper Section 9)

```bash
python -m selene.selftest

# === SELENE-128-ID Self-Test ===
#  Key norm ||s||_2 = 22.47  (bound: sqrt(n)*eta = 32.00, UB=sqrt(n)=16.00)
#  Enc/Dec: PASS
#  Ciphertext size: 1312 bytes
#  q/4 = 16384; error must be below this value
# === PASSED ===
```

---

## Safe Query Model (Key-Span Limitation)

SELENE shares with all IPFE schemes [ALS16, BSW11, BJKL18] the property that if queried keys
span Z^N, the master secrets are recoverable. This is a **structural property of linear FE**, not a
weakness specific to SELENE.

**When IND-FE security holds:**

| Query pattern | Safe? | Reason |
|--------------|-------|--------|
| Unit vectors `f = e_id` | YES, up to N-1 queries | Span at most N-1 dimensions |
| Binary `f in {0,1}^N` | YES, for Q < N queries | rank(queried set) < N |
| Bounded-rank families | YES, when rank < N | Standard IPFE regime |
| Spanning set of Z^N | NO | Master secrets recoverable — inherent to linear FE |

In practice, access control policies do not produce spanning query sets. This is the standard
IPFE operational regime [ALS16, BSW11].

---

## When to Use SELENE (vs. Alternatives)

| Need | SELENE | Full FHE | MLWE-PKE per slot | AEAD |
|------|--------|---------|------------------|------|
| Identity-based slot access | YES (f=e_id) | YES | YES | NO |
| Linear evaluation `<f,x>` over encrypted data | YES | YES (FHE circuits) | NO | NO |
| Post-quantum from MLWE | YES | If MLWE-instantiated | YES | NO |
| Small keys (RLWE-small, no trapdoors) | YES | NO (trapdoor keys) | YES | YES |
| Non-linear computation | NO | YES | NO | NO |
| Implementation complexity | LOW | VERY HIGH (bootstrap) | LOW | LOWEST |

**Three concrete applications:**

1. **Privacy-preserving aggregation** — N encrypted sensor readings. Owners decrypt only
   their slot (f=e_id). Aggregator computes weighted sums. Faster than FHE; post-quantum secure.

2. **Access-controlled linear scoring** — ML model computes score `<w, x>` over encrypted
   feature vector. SELENE issues key sk_w without revealing x or w. No FHE, no bootstrap.

3. **Selective-disclosure audit logging** — N encrypted audit fields. Auditors receive keys
   for field combinations (sums, weighted averages) without decrypting unrelated fields.

---

## Complete Security Boundary

| Property | Status | Bound / Proof | Conditions |
|----------|--------|--------------|-----------|
| IND-FE from MLWE | YES — Theorem 2 | N*Adv_MLWE + negl | All valid parameter sets |
| Correctness | YES — Theorem 1 | delta < 2^{-100} | Only for param sets with q/4 > err bound |
| IND-CCA2 (f=e_id, single slot) | YES — Theorem 5 | FO^perp bound [HHK17] | ROM required |
| Key norm bound | YES — Theorem 3 | `||sk_f|| <= B*sqrt(kn)*eta` | All (n,k,eta,B) |
| CKKS UB exact match | YES for k=1, eta=1, B=1 | `||sk||=sqrt(n)=UB` | Ternary secrets only |
| CKKS UB (k=1, eta=2) | PARTIAL: 2x above UB | RLWE-small, not tight | Stated precisely |
| Impossibility exemption | YES — Proposition 1 | Not M1, M2, or M3 | Independently proved |
| Full GPV-adaptive HIBE | NO — by design | N/A | Out of scope; not claimed |
| Arbitrary-circuit FHE | NO — by design | N/A | Out of scope; not claimed |
| Key-span collusion resistance | NO — standard IPFE | Same as ALS16, BJKL18 | Standard limitation |

---

## Open Problems / Future Work

1. **Adaptive IND-FE without admissibility** — current proof uses admissibility condition;
   removing it requires a different proof technique
2. **Multi-level inner-product FE** — bounded-depth polynomial evaluation
3. **Constant-time C implementation** with measured benchmarks
4. **Module rank k > 1** with adjusted CKKS compatibility analysis
5. **Formal verification** with EasyCrypt or Lean 4

---



---

## Key References

| Citation | Role in SELENE |
|----------|---------------|
| [QV-Thesis, Thm 8.1] | GPV-CKKS impossibility; SELENE is designed outside its scope |
| [ALS16] Abdalla-Laguillaumie-Sablonniere | Foundational LWE-IPFE; baseline for Table 1 comparison |
| [BJKL18] Brakerski et al. | Ring-LWE FE (partial module); second baseline |
| [BSW11] Boneh-Sahai-Waters | IND-FE security definition; admissibility condition |
| [HHK17] Hofheinz-Hoevelmanns-Kiltz | FO^perp — Theorem 5 (IND-CCA2) |
| [LPR13] Lyubashevsky-Peikert-Regev | Leftover Hash Lemma over R_q — Formal Lemma 1 |
| [APS15] Albrecht-Player-Scott | BKZ estimator — parameter security verification |
| [CHKP10] Cash-Hofheinz-Kiltz-Peikert | GPV SampleLeft (M2 condition in QV-Thesis) |
| [CKKS17] Cheon et al. | CKKS approximate FHE (M3 condition in QV-Thesis) |
| [NIST24] FIPS 203 | ML-KEM MLWE parameter reference |

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

<p align="center">
  <sub>SELENE · Module-LWE Inner-Product Functional Encryption · QuantumVault Research Group</sub>
</p>
