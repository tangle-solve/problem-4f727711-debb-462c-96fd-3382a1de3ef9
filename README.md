<!-- solve-problem:4f727711-debb-462c-96fd-3382a1de3ef9 -->
# Adaptive privacy across failed encrypted batches

A [Tangle Solve problem](https://solve.tangle.tools/problems/4f727711-debb-462c-96fd-3382a1de3ef9).

## What is the problem?

**Setting.** In an encrypted-mempool protocol (as in threshold-encryption MEV-prevention systems), users encrypt transactions to a committee, which reveals/decrypts them only once a batch is finalized and ordered. Formalize as follows:

- Each transaction *t* has content *m* and ciphertext *c = Enc_pk(m; r)* under a threshold public key *pk* held by a rotating committee.
- A batch *B* may **fail**: the committee changes before decryption, the batch is dropped, only a threshold subset decrypts (selective decryption failure), or the batch is reordered/retried. Users then **re-encrypt and resubmit**, typically *m* unchanged with fresh randomness *r′*.
- The adversary is **adaptive and stateful**: it observes multiple ciphertexts of the same *m* across retries, may corrupt committee members over time (learning decryption shares and, eventually, *r* and *m*), and may choose *which* batches fail (selective failure).

**Problem.** Characterize — and bound — the leakage of *m* (and of metadata: submission time, retry count, failure pattern) accumulated across the *composition* of failed batches. Concretely:

1. Define a composable security notion (UC-style or game-based) capturing "privacy across adaptive retries, committee rotation, and selective failure."
2. Prove either (a) leakage is simulatable from public metadata alone, or (b) exhibit an attack: an adaptive adversary using retry patterns, ciphertext reuse, or later committee compromise that distinguishes/extracts *m* beyond single-session security.
3. Derive the leakage bound as a function of number of retries, committee churn, and fraction of compromised members.

The census record is informal; the most reasonable precise reading is a **stateful, multi-session leakage-resilience question for threshold encryption with re-encryption on failure**. Ambiguity: whether ciphertext reuse (same *r*) is allowed — treat both variants.

## Why does it matters?

Encrypted mempools (deployed designs from Fuel, Shutter, Arbitrum Timeboost-adjacent research, and the USENIX 2025 "epochless" line) promise MEV resistance, but the promise silently degrades if pending transactions leak information each time a batch fails and is retried — exactly when users are most exposed (their intent is known to be still-live). A composable bound would let protocol designers certify retry handling; an attack would show real-world profit opportunities for adversaries who *induce* failures to extract intent. It also connects to long-standing selective-failure/openability problems in threshold cryptography, giving that literature a concrete new application.

## What's already known?

- **Single-session security is classical**: threshold encryption with re-randomizable ciphertexts is IND-CPA/CCA secure under standard assumptions; re-encryption with fresh randomness is standard in mixnet literature. This does *not* directly answer multi-retry adaptive leakage.
- **Bormet et al. (2025), BEAT-MEV**: epochless batched threshold encryption for MEV prevention — removes fixed epochs so committees/keys can change continuously, but the paper's analysis is essentially per-batch; retry composition and post-hoc compromise are out of scope (verify — treat the exact claimed scope as needing confirmation from the paper).
- **Selective-failure attacks**: known in the threshold-cryptography literature (e.g., proofs of partial knowledge, "faulty share" attacks); typically mitigated by verifiable encryption / DLEQ proofs. Whether such mitigations suffice under adaptive committee corruption across retries is open.
- **Forward security / key-evolving signatures** handle "later compromise" for authenticity, but the analogous forward privacy for *transaction contents* across re-encryption is not standard.
- **Composability frameworks** (UC, constructive cryptography, and leakage-resilient cryptography à la Naor–Segev / Dziembowski–Tentes) provide the tools; no one has instantiated them for failed-batch retries.

No published attack or bound for the exact composition is known to this census — this is genuinely open (verify against the BEAT-MEV paper's future-work section).

## What would count as solving it?

**Variant A (positive / bound).** A proof in a composable framework: define ideal functionality ℱ_retry-enc capturing encrypted-mempool retries; construct a protocol (threshold encryption + re-encryption + forward-secure key evolution) realizing it; prove adversary advantage ≤ negligible(λ) + f(retries, churn, compromised fraction), with f explicit (e.g., leakage ≤ metadata + ℓ bits with stated ℓ). A referee accepts: formal ideal functionality, full simulation proof or concrete leakage bound.

**Variant B (negative / attack).** A valid adversary strategy: given oracle access to re-encryption of the same message across adaptively-failed batches, plus adaptive corruption of a bounded fraction of committee members, distinguish two messages with non-negligible advantage — or fully recover *m* — while the same adversary loses in the single-batch game. Must specify the protocol class attacked (e.g., "any scheme with deterministic re-encryption," or "BEAT-MEV's epochless construction").

**Variant C (construction).** A retry-handling mechanism (fresh keys per retry? encrypted memos? delay-based release?) achieving A's bound with practical overhead — with proofs and benchmarks.

Matching A/B results (attack + matching defense) is the strongest outcome.

## How to start

1. **Read Bormet et al. (2025) first**, especially its security model and any discussion of failures/retries. Extract exactly what environment abstraction they use — your job is to extend it with state.
2. **Formalize the simplest non-trivial game**: two retries, one committee change, static adversary corrupting one old member. Ask: does the old share plus two ciphertexts of *m* leak more than either alone? For El Gamal–style re-randomization the naive answer is "no" (DDH), but check the *timing/metadata* channel: retry counts depend on *m* (e.g., failing batches that would reveal the transaction is valuable).
3. **Import selective-failure tools**: verifiable secret sharing / DLEQ proofs prevent malformed-share leakage; check whether adaptive corruption after the fact defeats them.
4. **Main obstacle**: the leakage is likely dominated not by the ciphertext math but by the *adaptive failure pattern itself* — the adversary controls failures and learns from *whether* retries occur. Modeling this metadata channel honestly is where the problem lives.

## Key references

- Bormet et al., "BEAT-MEV: Epochless Approach to Batched Threshold Encryption for MEV Prevention," USENIX Security 2025. Proposes epochless batched threshold encryption for MEV-resistant ordering; the baseline construction whose retry-composition privacy is in question. https://www.usenix.org/conference/usenixsecurity25/presentation/bormet
- Canetti, "Universally Composable Security" (FOCS 2001 / JACS 2019 version). The composability framework you should instantiate for the retry functionality.
- Naor & Segev, "Public-Key Cryptosystems Resistant to Key Leakage" (CRYPTO 2009) and Dziembowski & Tentes, "Leakage-Resilient Cryptography from the Inner-Product Extractor" (ASIACRYPT 2010). Standard bounded-leakage machinery for quantifying what a partially compromised committee learns. (verify applicability)
- Breidenbach et al., "Threshold Encryption for Fair Transaction Ordering in FairBlock" / related fair-ordering literature — context on encrypted mempool deployments and their threat models. (verify exact citation)

## Warnings

- The census record is programmatic, not from a paper: "Bormet et al. (2025)" and the problem statement itself should be sanity-checked against the actual USENIX paper — do not assume the open problem is stated there verbatim.
- Do not conflate *ciphertext* leakage with *metadata* leakage (retry counts, timing, which batches failed). A pure-crypto "no leakage" proof is likely easy and uninteresting; the real problem is the adaptive metadata channel.
- Watch the ciphertext-reuse case: if any implementation re-encrypts deterministically or reuses *r*, trivial correlation attacks exist — state which case you address.
- "Later compromise" requires forward-secure secret sharing or key evolution; ordinary threshold shares decrypted-but-unrevealed may already be logged.
- Committee rotation is where naive proofs break: the simulator must handle old-committee shares surviving into new epochs.

---
*`BCGT-000321` · flagged for review*

## Acceptance guidance

**Variant A (positive / bound).** A proof in a composable framework: define ideal functionality ℱ_retry-enc capturing encrypted-mempool retries; construct a protocol (threshold encryption + re-encryption + forward-secure key evolution) realizing it; prove adversary advantage ≤ negligible(λ) + f(retries, churn, compromised fraction), with f explicit (e.g., leakage ≤ metadata + ℓ bits with stated ℓ). A referee accepts: formal ideal functionality, full simulation proof or concrete leakage bound.

**Variant B (negative / attack).** A valid adversary strategy: given oracle access to re-encryption of the same message across adaptively-failed batches, plus adaptive corruption of a bounded fraction of committee members, distinguish two messages with non-negligible advantage — or fully recover *m* — while the same adversary loses in the single-batch game. Must specify the protocol class attacked (e.g., "any scheme with deterministic re-encryption," or "BEAT-MEV's epochless construction").

**Variant C (construction).** A retry-handling mechanism (fresh keys per retry? encrypted memos? delay-based release?) achieving A's bound with practical overhead — with proofs and benchmarks.

Matching A/B results (attack + matching defense) is the strongest outcome.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/4f727711-debb-462c-96fd-3382a1de3ef9?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000321.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000321)
