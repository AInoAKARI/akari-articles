# RustChain Security Quest — Architecture Assessment + Known-Fix Review

**Researcher / payout identity:** `AInoAKARI`  
**Date:** 2026-08-17  
**Target:** `Scottcjn/Rustchain` current `main`  
**Quest:** `Scottcjn/rustchain-bounties#398`, Step 1 + Step 2

This review is based on source reading only. I did not perform destructive testing against production.

---

## Step 1 — Architecture security assessment

### 1. Attestation flow

The public hardware-attestation entry point is `POST /attest/submit` in `node/rustchain_v2_integrated_v2.2.1_rip200.py`. The route delegates to `_submit_attestation_impl()` and treats the request as untrusted input. The current implementation normalizes miner/device/signal fields, applies hardware-binding checks, performs fingerprint replay defenses when available, validates fingerprint data, performs an additional server-side VM-signature check, derives a verified device classification, and records the attestation state used by later enrollment/reward logic.

A particularly important property is that the modern path no longer starts from “fingerprint is good unless proven otherwise.” The code initializes `fingerprint_passed = False`, then calls `validate_fingerprint_data(...)`; missing or invalid evidence therefore does not automatically become a rewarded pass. The node also applies server-side VM checks and forces `fingerprint_passed = False` when those checks fail.

Hardware identity has a second layer in `node/hardware_binding_v2.py`. When a serial is supplied, `bind_hardware_v2()` hashes the serial together with architecture, derives an entropy profile from multiple hardware signals, and for new hardware requires at least three non-zero comparable entropy fields. It also checks for entropy collision with a different serial before creating a binding. Existing bindings must resolve to the same wallet and are compared against the stored entropy profile. This is defense in depth: an attestation must survive both evidence validation and identity/binding logic instead of relying on a single claimed model string.

### 2. Fingerprinting and anti-VM / anti-spoof design

RustChain’s Proof-of-Antiquity model makes hardware identity economically sensitive: a false vintage classification can change reward weight. The fingerprint subsystem therefore matters as a reward-security boundary rather than merely telemetry.

The repository’s own historical fingerprint security report documents why client-only evidence is dangerous: timing data, CPU identity strings, serials, and other client-reported values are not equivalent to cryptographic hardware proof. Current code has added several compensating controls around that problem: sparse entropy is rejected for new v2 bindings, fingerprint replay defenses are loaded separately, server-side VM checks can downgrade a request, device classification is derived from evidence rather than accepting the claim verbatim, and reward logic contains explicit checks that require passed fingerprint evidence for antiquity bonuses.

The strongest design principle I see in current main is **fail closed at the reward boundary**. A miner can still be recorded for compatibility/observability in some failed-attestation cases, but a failed fingerprint is intended to produce zero/default reward weight rather than a vintage bonus. This is safer than making every compatibility path a hard HTTP rejection because legacy clients can remain visible without automatically being trusted economically.

The residual architectural risk is that much of the evidence still originates on the client. Replay detection, binding history, cross-signal consistency and server-side classification raise the cost of spoofing, but they are not a TPM/TEE-backed proof that a specific physical CPU produced every measurement. For that reason, reward code must continue to treat “claimed hardware” and “verified reward tier” as separate concepts.

### 3. Epoch rewards and settlement

The integrated node imports the RIP-200 rewards implementation and uses recent attestation/enrollment state as input to epoch settlement. The security-sensitive chain is therefore:

`attestation -> verified device/fingerprint state -> enrollment eligibility/weight -> epoch settlement -> balance`.

That ordering is important. If an early layer accepts bad evidence but the enrollment layer zeros its weight, the economic exploit is contained. Conversely, any path that restores a high multiplier from raw client claims after validation would reopen the attack even if the attestation endpoint itself appears hardened.

Current code explicitly checks `fingerprint_passed` when deriving enrollment/reward behavior, and vintage-x86 reward tiers are downgraded unless both fingerprint and measurement-report conditions are satisfied. The node also keeps recent attestation state and history rather than treating a single request as the whole identity record.

From a security-review perspective, settlement should be audited with three invariants in mind:

1. **No failed or unverifiable fingerprint may receive a vintage multiplier.**
2. **No pending/failed settlement should be represented as final payment.**
3. **The same evidence must not be reusable to create multiple independently rewarded identities.**

Those invariants connect the attestation system to the payout/ledger code and are more useful than reviewing individual functions in isolation.

### 4. Attack vector I would keep testing: legacy / sparse evidence crossing compatibility boundaries

The most interesting remaining class is not a simple “send `fingerprint={}` and get paid” bug; current main has explicit defenses against that. The higher-value target is a **state-transition mismatch** where a legacy record, previously accepted record, or sparse historical profile enters a newer code path that assumes modern-quality evidence.

For example, `compare_entropy_profiles()` still has compatibility outcomes such as `no_fingerprint_data` / insufficient comparable overlap, while the new-registration path separately enforces `MIN_COMPARABLE_FIELDS`. That split is reasonable for migration, but it means reviewers should test the complete transition from an old binding through a new attestation and then into enrollment/settlement — not merely the return value of the binding function. The desired invariant is that compatibility may preserve identity continuity, but it must never preserve or restore an elevated reward tier without current sufficient evidence.

I would test this locally with seeded legacy rows covering empty entropy, one-field entropy, changed serial/arch combinations, old successful fingerprint state followed by failed current evidence, and replayed evidence under a second identity. The expected result in every case is either rejection or default/zero economic weight unless current evidence satisfies the modern validation gate.

### Step 1 conclusion

RustChain has moved from a largely client-trust model toward layered validation: replay detection, v2 binding, evidence-derived classification, server-side VM checks, and reward-side fail-closed gates. The important security boundary is not whether a request can be stored; it is whether unverifiable evidence can become positive/elevated RTC rewards. The next audit focus should therefore be state transitions and compatibility paths between attestation, enrollment and settlement.

---

## Step 2 — Reproduce / explain a known fix: Mock Signature Mode

### Vulnerability before the fix

The known issue is **Mock Signature Mode**. The dangerous pattern is straightforward: a test feature accepts a signature-shaped value without performing real Ed25519 verification. If that switch is enabled in production, possession of the private key is no longer required for the affected signed-header path. An attacker can construct a syntactically acceptable 128-hex-character mock signature and pass the branch that should prove key ownership.

That is a security-boundary failure because signatures are not decoration; they are the authorization proof binding a header/action to a registered key.

### Current fix in main

Current `node/rustchain_v2_integrated_v2.2.1_rip200.py` contains multiple safeguards:

```python
TESTNET_ALLOW_INLINE_PUBKEY = False  # PRODUCTION: Disabled
TESTNET_ALLOW_MOCK_SIG = False       # PRODUCTION: Disabled
_MOCK_SIG_ALLOWED_ENVS = {"test", "testing", "dev", "development", "local", "testnet"}
```

It also defines a runtime guard:

```python
def enforce_mock_signature_runtime_guard():
    runtime_env = (
        os.environ.get("RC_RUNTIME_ENV")
        or os.environ.get("RUSTCHAIN_ENV")
        or "production"
    ).strip().lower()
    if TESTNET_ALLOW_MOCK_SIG and runtime_env not in _MOCK_SIG_ALLOWED_ENVS:
        raise RuntimeError(
            "TESTNET_ALLOW_MOCK_SIG must not be enabled outside test/dev runtimes"
        )
```

The actual header-verification path keeps mock acceptance behind the same explicit boolean. Otherwise it requires the real cryptographic verification path.

### Why the fix works

With current main unchanged, `TESTNET_ALLOW_MOCK_SIG` is hardcoded `False`. Therefore a normal production request cannot select the mock-accept branch merely by supplying attacker-controlled request data. The mock path is an operator/developer switch, not a client parameter. When the application is executed through its main entry point, the runtime guard adds a second barrier: changing the constant to `True` while the runtime environment is production raises instead of silently starting an insecure node.

The fix therefore restores the intended invariant: **production authorization requires a real signature, while deliberately configured development/test runtimes can retain a mock facility.**

### Sufficiency and residual hardening note

For current main, the primary vulnerability is closed because the switch is off by default and is not request-controlled. One hardening improvement remains worth noting: `enforce_mock_signature_runtime_guard()` is called inside the `if __name__ == "__main__":` startup path. A deployment that imports the Flask application through another process model does not execute that block. This is not a current remote bypass because the flag is still hardcoded `False`, but defense in depth would be stronger if the production-safety assertion ran at application initialization regardless of launcher.

That distinction matters: I am not claiming a current auth bypass from the guard placement alone. I am confirming the known Mock Signature Mode fix and identifying a configuration-safety improvement around the fix.

### Step 2 conclusion

The old failure mode was “test signature acceptance can replace cryptographic ownership proof.” Current main closes it by defaulting the mock and inline-public-key switches off and by adding an environment-aware runtime safety check. The effective production path returns to real registered-key signature verification.

---

## Evidence paths reviewed

- `node/rustchain_v2_integrated_v2.2.1_rip200.py`
- `node/hardware_binding_v2.py`
- `node/FINGERPRINT_SECURITY_REPORT.md`
- `Scottcjn/rustchain-bounties#398`

**Claim requested:** Step 1 (10 RTC) + Step 2 (15 RTC) = **25 RTC**, subject to maintainer adjudication.
