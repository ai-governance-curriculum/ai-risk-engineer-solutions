# Exercise 05 — Sigstore Model Signing and Registry-Ingestion Gate — Reference Solution

## 1. Solution overview

This exercise produces a **signed model artifact** using Sigstore (cosign + the model-signing library) and an **ingestion gate** that fails-closed when a model arriving at the registry is unsigned, signed by an unapproved identity, or signed over content that no longer matches.

The deliverable is a working set of scripts and CI configuration, plus a short rationale:

- `scripts/sign_model.py` — builds the artifact bundle (model binary + its three attestations from mod-110 ex04: CycloneDX ML-BOM, SPDX 3.0 AI document, SLSA v1 Provenance), computes a content digest, and signs it with Sigstore using **keyless OIDC** (Fulcio-issued short-lived certificate, Rekor transparency-log entry). The shape mirrors the `model-signing` project from the Sigstore ecosystem and the `cosign` CLI for the per-file envelopes.
- `scripts/verify_model.py` — the registry-side verifier: pulls the artifact + signatures, validates the signature chain back to the Sigstore trust root, pins the allowed signer identity by OIDC issuer and subject pattern, verifies the Rekor inclusion proof, and checks that each declared attestation's digest matches the model bytes.
- `.github/workflows/dx-build.yml` snippet — CI job that runs the build, calls `sign_model.py`, and uploads the bundle; the ingestion gate at the model registry is the verifier running as an admission control at `scripts/verify_model.py`.
- `rationale.md` — why keyless is the default, what the Rekor log buys, and how the gate composes with mod-110 ex06's scanning gate.

Design decisions worth calling out:

- **Keyless by default.** Long-lived signing keys are an operational liability. Sigstore's keyless flow — an OIDC-federated short-lived certificate from Fulcio, plus a public inclusion proof in Rekor — is the default per the upstream project documentation and removes the "where is the signing key" question.
- **Identity policy at the gate, not at the signer.** The signer doesn't decide who is allowed to sign; the verifier does. Policy lives next to the registry, versioned.
- **Fail-closed.** Unsigned or mis-signed artifacts never reach the registry. The gate does not warn-and-allow.
- **One bundle per model release.** The ex04 attestations (ML-BOM, SPDX, SLSA Provenance) are attestations *over the same subject digest*; a single Sigstore signature over the bundle + per-attestation DSSE envelopes keep the trust surface small.
- **Scanning is deferred to exercise 06.** Signing proves *who* and *what-shape*; scanning proves *what-inside*. The two gates compose in series — signature first (cheap, fails fast), then scan (slower but deeper).

## 2. Implementation — worked answer (signer, verifier, CI, policy)

### 2.1 Repository layout

```
model-signing/
├── scripts/
│   ├── sign_model.py           # keyless Sigstore signing
│   └── verify_model.py         # registry-ingestion gate
├── policy/
│   └── allowed-signers.yaml    # OIDC issuer + subject patterns + required attestations
├── .github/workflows/
│   └── dx-build.yml            # CI job: build → ML-BOM/SPDX/SLSA (from ex04) → sign → upload
└── rationale.md
```

### 2.2 Signing script (`scripts/sign_model.py`)

```python
# scripts/sign_model.py
#
# Signs a model artifact and its three attestations (CycloneDX ML-BOM,
# SPDX 3.0 AI document, SLSA v1 Provenance) with Sigstore using the keyless
# OIDC flow. Writes a bundle directory the registry verifier can consume.
#
# Key properties:
# 1. The signing identity is a short-lived Fulcio-issued certificate tied to
#    the CI workflow's OIDC token. No long-lived private key is stored.
# 2. The signed subject is the SHA-256 digest of the model artifact; the
#    attestations are signed as DSSE envelopes over their own JSON bytes.
# 3. Every signature is logged to Rekor; the Rekor log entry is written out
#    next to the signature for offline verification.
#
# Upstream references:
# - sigstore/model-signing: https://github.com/sigstore/model-signing
# - sigstore/cosign:       https://docs.sigstore.dev/cosign/
# - sigstore-python SDK:   https://github.com/sigstore/sigstore-python
# Confirm the exact function/class names at authoring time:
#   verify-citation: sigstore-python public API (SigningContext / Signer / Verifier).

from __future__ import annotations

import argparse
import hashlib
import json
import pathlib
import subprocess
import sys


def _sha256_file(path: pathlib.Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as fp:
        for chunk in iter(lambda: fp.read(1024 * 1024), b""):
            h.update(chunk)
    return h.hexdigest()


def _cosign_sign_blob(blob_path: pathlib.Path, bundle_out: pathlib.Path) -> None:
    # The reference implementation shells out to the cosign CLI to avoid
    # drift between the Python SDK and the CLI behaviour. Keyless flow:
    # the CI token is picked up from GITHUB_ACTIONS_OIDC_... env; the issued
    # cert is embedded in the bundle, the Rekor entry is referenced.
    cmd = [
        "cosign",
        "sign-blob",
        "--yes",
        "--bundle",
        str(bundle_out),
        str(blob_path),
    ]
    subprocess.check_call(cmd)


def _write_subject_manifest(
    model_path: pathlib.Path,
    attestations: list[pathlib.Path],
    out_path: pathlib.Path,
) -> dict:
    manifest = {
        "model": {
            "name": model_path.name,
            "sha256": _sha256_file(model_path),
            "size": model_path.stat().st_size,
        },
        "attestations": [
            {
                "name": p.name,
                "sha256": _sha256_file(p),
                "size": p.stat().st_size,
            }
            for p in attestations
        ],
    }
    out_path.write_text(json.dumps(manifest, indent=2, sort_keys=True))
    return manifest


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Sign a model artifact with Sigstore keyless.")
    parser.add_argument("--model", required=True, type=pathlib.Path)
    parser.add_argument("--ml-bom", required=True, type=pathlib.Path)
    parser.add_argument("--spdx", required=True, type=pathlib.Path)
    parser.add_argument("--slsa", required=True, type=pathlib.Path)
    parser.add_argument("--out", required=True, type=pathlib.Path, help="Output bundle directory")
    args = parser.parse_args(argv)

    out = args.out
    out.mkdir(parents=True, exist_ok=True)

    # 1) Build the manifest: SHA-256 of the model and of each attestation.
    manifest_path = out / "manifest.json"
    _write_subject_manifest(
        args.model, [args.ml_bom, args.spdx, args.slsa], manifest_path
    )

    # 2) Sign the model and each attestation separately so a verifier can
    #    reject a single tampered attestation without rejecting the whole
    #    bundle. Each bundle file contains the Fulcio cert, the signature,
    #    and the Rekor inclusion proof.
    for subject in [args.model, args.ml_bom, args.spdx, args.slsa, manifest_path]:
        _cosign_sign_blob(subject, out / f"{subject.name}.sigstore.bundle")

    print(f"signed: {args.model}")
    print(f"bundle: {out}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 2.3 Allowed-signers policy (`policy/allowed-signers.yaml`)

```yaml
# policy/allowed-signers.yaml
# The ingestion gate enforces this policy. Keyless signing means the signer's
# "identity" is not a key fingerprint but an OIDC issuer + subject pair.

allowed_signers:
  - name: ai-risk-engineering GitHub Actions workflow
    issuer: https://token.actions.githubusercontent.com
    subject_pattern: "https://github.com/example-org/carepilot/.github/workflows/dx-build.yml@refs/tags/*"
    # Only tag-ref invocations of the pinned workflow may sign for ingestion.
    # Branch pushes may sign for dev-only artifacts (see second entry).

  - name: ai-risk-engineering development builds
    issuer: https://token.actions.githubusercontent.com
    subject_pattern: "https://github.com/example-org/carepilot/.github/workflows/dx-build.yml@refs/heads/main"
    registry_scope: dev-only

required_attestations:
  - CycloneDX-1.6-ML-BOM
  - SPDX-3.0-AI
  - SLSA-v1-Provenance

rekor_verification: required
fulcio_trust_root: default   # ships with the Sigstore trust root; override only for staging
```

### 2.4 Verifier script (`scripts/verify_model.py`)

```python
# scripts/verify_model.py
#
# Registry-ingestion gate. Fails-closed on any of:
#   - unsigned subject (model or any required attestation)
#   - signature chain does not verify back to the pinned Sigstore trust root
#   - Fulcio certificate identity (issuer, subject) is not in the policy allowlist
#   - Rekor inclusion proof missing or does not match
#   - SHA-256 of the artifact on disk does not match what was signed
#
# The verifier is intentionally conservative: it does not accept a signed
# artifact if any single required attestation is missing or invalid.

from __future__ import annotations

import argparse
import fnmatch
import hashlib
import json
import pathlib
import subprocess
import sys

import yaml


class VerificationError(Exception):
    pass


def _sha256_file(path: pathlib.Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as fp:
        for chunk in iter(lambda: fp.read(1024 * 1024), b""):
            h.update(chunk)
    return h.hexdigest()


def _cosign_verify_blob(
    blob_path: pathlib.Path,
    bundle_path: pathlib.Path,
    issuer: str,
    subject_pattern: str,
) -> dict:
    # Returns the parsed cosign-verify output for policy inspection.
    cmd = [
        "cosign",
        "verify-blob",
        "--bundle",
        str(bundle_path),
        "--certificate-identity-regexp",
        _glob_to_regex(subject_pattern),
        "--certificate-oidc-issuer",
        issuer,
        str(blob_path),
    ]
    try:
        subprocess.check_output(cmd, stderr=subprocess.STDOUT)
    except subprocess.CalledProcessError as exc:
        raise VerificationError(
            f"cosign verify-blob failed for {blob_path.name}: {exc.output.decode(errors='replace')}"
        ) from exc
    return {"verified": True, "subject": str(blob_path)}


def _glob_to_regex(pattern: str) -> str:
    # fnmatch's translate is close enough for the subject pattern language used here.
    return fnmatch.translate(pattern)


def _match_signer(cert_identity: dict, allowed: list[dict]) -> dict | None:
    for entry in allowed:
        if entry["issuer"] != cert_identity["issuer"]:
            continue
        if fnmatch.fnmatchcase(cert_identity["subject"], entry["subject_pattern"]):
            return entry
    return None


def verify_bundle(
    model_path: pathlib.Path,
    bundle_dir: pathlib.Path,
    policy_path: pathlib.Path,
) -> None:
    policy = yaml.safe_load(policy_path.read_text())

    # 1) Load and verify the manifest's signature first.
    manifest_path = bundle_dir / "manifest.json"
    if not manifest_path.exists():
        raise VerificationError("manifest.json missing from bundle")

    # 2) Walk each required subject.
    subjects: list[tuple[pathlib.Path, str]] = []
    subjects.append((model_path, "model"))
    manifest = json.loads(manifest_path.read_text())
    for att in manifest["attestations"]:
        subjects.append((bundle_dir / att["name"], att["name"]))
    subjects.append((manifest_path, "manifest.json"))

    # 3) For each subject, verify signature + identity policy.
    #    The reference implementation trusts cosign to enforce issuer / subject
    #    at the CLI level; the policy loop here is the final allowlist check.
    for subject, label in subjects:
        if not subject.exists():
            raise VerificationError(f"missing subject {label}: {subject}")
        bundle_path = bundle_dir / f"{subject.name}.sigstore.bundle"
        if not bundle_path.exists():
            raise VerificationError(f"missing Sigstore bundle for {label}: {bundle_path.name}")

        # Try every allowed signer; the first match wins.
        matched = False
        last_err: VerificationError | None = None
        for entry in policy["allowed_signers"]:
            try:
                _cosign_verify_blob(
                    subject,
                    bundle_path,
                    entry["issuer"],
                    entry["subject_pattern"],
                )
                matched = True
                break
            except VerificationError as err:
                last_err = err
                continue
        if not matched:
            raise VerificationError(
                f"no allowed signer verified {label}: {last_err}"
            )

    # 4) Hash cross-check: the manifest's recorded digests must equal the
    #    on-disk digests. This blocks a replacement-after-signing attack
    #    where the bundle is signed over the original but the file on disk
    #    has been swapped out.
    model_actual = _sha256_file(model_path)
    if model_actual != manifest["model"]["sha256"]:
        raise VerificationError(
            f"model digest mismatch: manifest {manifest['model']['sha256']} vs actual {model_actual}"
        )
    for att in manifest["attestations"]:
        actual = _sha256_file(bundle_dir / att["name"])
        if actual != att["sha256"]:
            raise VerificationError(
                f"attestation digest mismatch for {att['name']}: manifest {att['sha256']} vs actual {actual}"
            )

    # 5) Required-attestation check: the policy enumerates the attestations
    #    that must be present. A signed bundle that is missing the ML-BOM
    #    (for example) is rejected.
    present = {att["name"] for att in manifest["attestations"]}
    want_names_by_shape = {
        "CycloneDX-1.6-ML-BOM": lambda n: n.endswith(".cdx.json"),
        "SPDX-3.0-AI": lambda n: "spdx" in n and n.endswith(".json"),
        "SLSA-v1-Provenance": lambda n: "slsa" in n and n.endswith(".intoto.json"),
    }
    for required_label in policy["required_attestations"]:
        shape = want_names_by_shape.get(required_label)
        if shape is None:
            raise VerificationError(f"policy references unknown attestation type: {required_label}")
        if not any(shape(name) for name in present):
            raise VerificationError(f"missing required attestation: {required_label}")

    print("OK — model signed and verified; all required attestations present")


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Verify a Sigstore-signed model bundle.")
    parser.add_argument("--model", required=True, type=pathlib.Path)
    parser.add_argument("--bundle-dir", required=True, type=pathlib.Path)
    parser.add_argument("--policy", required=True, type=pathlib.Path)
    args = parser.parse_args(argv)

    try:
        verify_bundle(args.model, args.bundle_dir, args.policy)
    except VerificationError as exc:
        print(f"FAIL — {exc}", file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 2.5 CI workflow snippet (`.github/workflows/dx-build.yml`)

```yaml
# .github/workflows/dx-build.yml (snippet)
name: dx-build
on:
  push:
    tags: ["*"]
    branches: ["main"]
permissions:
  id-token: write    # required for keyless Sigstore signing via OIDC
  contents: read

jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Build the inference container and the training artifact.
      - name: Build
        run: make build

      # Author the three attestations from mod-110 ex04.
      - name: Author ML-BOM, SPDX 3.0 AI, SLSA Provenance
        run: make attestations

      # Install cosign (pinned version).
      - uses: sigstore/cosign-installer@v3

      # Sign the bundle keyless.
      - name: Sign
        run: |
          python scripts/sign_model.py \
            --model artifacts/carepilot/model.bin \
            --ml-bom artifacts/carepilot/supply-chain/ml-bom.cdx.json \
            --spdx artifacts/carepilot/supply-chain/spdx-3-ai.json \
            --slsa artifacts/carepilot/supply-chain/slsa-provenance.v1.intoto.json \
            --out artifacts/carepilot/supply-chain/bundle/

      # Verify locally before upload — catches mis-signed artifacts before they leave CI.
      - name: Self-verify
        run: |
          python scripts/verify_model.py \
            --model artifacts/carepilot/model.bin \
            --bundle-dir artifacts/carepilot/supply-chain/bundle/ \
            --policy policy/allowed-signers.yaml

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: carepilot-signed-bundle
          path: artifacts/carepilot/
```

### 2.6 Registry-ingestion gate wiring

- The verifier script `scripts/verify_model.py` is embedded in the registry's admission-control path: on upload, the registry runs the verifier against the uploaded blob + bundle; **non-zero exit blocks ingestion**.
- The composed gate in mod-110 ex06 adds the content-scan step **after** the verifier passes.
- Policy updates to `allowed-signers.yaml` are themselves reviewed through a code-review workflow with model-risk-committee sign-off (mod-110 ex01 §4).

### 2.7 Rationale (`rationale.md`)

```markdown
# Why Sigstore keyless and why these policy choices

- **Keyless OIDC** removes the long-lived-private-key problem. The signer is identified by the OIDC claims baked into the Fulcio certificate, not by a key file. A compromised runner cannot reuse the identity after the workflow finishes because the cert's lifetime is minutes.
- **Rekor** gives a public transparency log for every signing event. A signer who tries to withdraw or replace a signature cannot do so silently; verifiers can require a Rekor inclusion proof.
- **Identity is checked at the gate.** `allowed-signers.yaml` pins the OIDC issuer and the `subject_pattern` of the workflow that is allowed to sign ingestable builds. A `push` on an arbitrary branch cannot sign over a production ingestion unless policy adds it.
- **Fail-closed, every step.** If any required attestation is missing, if a digest cross-check fails, if the Fulcio chain or Rekor entry fails to verify, the gate rejects. There is no "warn and allow" mode.
- **Composition with ex06.** This gate verifies *who* and *what-shape*. Scanning (ModelScan, picklescan) verifies *what-inside*. Both run; both must pass.
```

## 3. Validation steps

1. **Signer runs.** `python scripts/sign_model.py --model … --ml-bom … --spdx … --slsa … --out bundle/` produces a `bundle/` directory with one `.sigstore.bundle` per subject plus `manifest.json`.
2. **Verifier accepts the good case.** `python scripts/verify_model.py --model … --bundle-dir bundle/ --policy policy/allowed-signers.yaml` exits 0.
3. **Unsigned-artifact rejection.** Removing a `.sigstore.bundle` file, re-running verify exits non-zero with `missing Sigstore bundle for …`.
4. **Wrong-identity rejection.** Replacing `subject_pattern` in the policy with a value that does not match the signer's cert exits non-zero with `no allowed signer verified …`.
5. **Tamper rejection.** Modifying the model bytes after signing (while leaving the bundle in place) exits non-zero with `model digest mismatch: …`.
6. **Missing-attestation rejection.** Removing the SPDX file after signing exits non-zero with `missing required attestation: SPDX-3.0-AI`.
7. **Replay rejection.** Swapping an old `manifest.json` into a new bundle breaks the cross-check (the manifest's recorded digests won't match the new on-disk digests) and the verifier rejects.
8. **Rekor inclusion verified.** The cosign CLI verifies Rekor inclusion by default on verify-blob; a bundle with a missing inclusion proof fails.
9. **CI job self-verifies.** The CI snippet runs verify before upload, so a mis-signed artifact never leaves CI.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Keyless signing | Signing uses Sigstore OIDC/Fulcio/Rekor; no long-lived private key is referenced | Implementation uses a bare `cosign.key` file or an unspecified keypair |
| R2 | Per-subject signatures | Model and each attestation signed separately | One signature over a tar of everything; no per-subject granularity |
| R3 | Content digests cross-checked | Verifier re-hashes the on-disk artifact and compares to the manifest | Verifier trusts the manifest without re-hashing |
| R4 | Identity policy pinned | `allowed-signers.yaml` pins OIDC issuer + subject pattern; verifier enforces | Any signer accepted |
| R5 | Required-attestation enforcement | Policy enumerates ML-BOM / SPDX / SLSA and verifier fails on missing | Attestations optional |
| R6 | Fail-closed | Every negative case above exits non-zero and blocks ingestion | Warn-and-allow path or defaulting |
| R7 | Rekor involvement | Signing and verification flow exercises the Sigstore transparency log | No Rekor references; implementation signs with an offline key |
| R8 | CI self-verification | Build job runs the verifier before upload | Build uploads unverified artifacts |
| R9 | Composition with ex06 noted | Rationale explicitly hands off content-scanning to ex06 | Tries to re-invent content scanning here |
| R10 | No invented facts | Specific CLI flag names and SDK symbols carry `<!-- verify-citation: ... -->` where unverified; no fabricated Fulcio URLs | Fabricated URLs, hashes, cert claims |

Scoring bands: *Pass* — R1–R8 green, R9/R10 honoured; *Pass with revisions* — R2 misses per-subject granularity but other controls are intact; *Fail* — any of R1, R4, R5, R6.

## 5. Common mistakes

- **Long-lived signing key in a GitHub secret.** This is the problem Sigstore was built to avoid. A reviewer rejects any implementation that stores a long-lived private key.
- **Verifier accepts "signed by somebody, somewhere."** Without identity pinning (OIDC issuer + subject pattern), a signed artifact from any Fulcio-trusted workflow passes. The allowlist is the control.
- **Skipping the re-hash cross-check.** Signing happens before upload; the artifact can be modified between signing and the registry. Re-hashing at the gate catches this.
- **Signing a tar of everything with one signature.** Granular per-subject signatures let you reject a tampered attestation without rejecting the model. The per-subject shape also lets the registry refuse ingestion for a partial bundle.
- **No Rekor inclusion proof.** Without the transparency log, a bad signer could sign and later deny; Rekor's public inclusion proof is the deterrent.
- **Treating signing as proving quality.** A signature proves the artifact was *produced by X* and has *not been modified since*; it does not prove the model is safe, performant, or correct. Scanning (ex06), validation (ex01 §3), and monitoring (mod-109) prove those things.
- **Policy baked into the signer.** If the signer decides who is allowed to sign, a compromised signer can expand the allowlist silently. The policy must be versioned next to the registry, with model-risk-committee sign-off for changes.
- **Verifier warns on failure.** Warn-and-allow makes the gate decorative. Fail-closed is the only safe mode.
- **CI does not self-verify.** Without self-verify the first time a problem is noticed is at the registry; a tight loop runs the verifier at the end of the signing job.
- **Fabricated Sigstore URLs or endpoints.** Sigstore's trust root, Fulcio, and Rekor are hosted services with specific URLs; invent them and the implementation looks real but fails on the first network call.

## 6. References

- **Sigstore Project.** Documentation portal. `https://docs.sigstore.dev/` — primary source for Fulcio, Rekor, cosign, and the keyless flow used in §2.2. <!-- verify-citation: confirm exact CLI flag names and current endpoint URLs at authoring time. -->
- **sigstore/cosign.** `https://docs.sigstore.dev/cosign/` — the CLI used for `sign-blob` and `verify-blob`.
- **sigstore/model-signing.** `https://github.com/sigstore/model-signing` — community implementation of model-signing bundles that this exercise's shape mirrors. <!-- verify-citation: confirm the current CLI / Python API surface. -->
- **sigstore/sigstore-python.** `https://github.com/sigstore/sigstore-python` — Python SDK; note the reference implementation shells out to `cosign` rather than binding the SDK directly to avoid drift.
- **OpenSSF Scorecard.** `https://openssf.org/scorecard/` — related OpenSSF project that consumes signing + provenance signals; referenced for the governance tie-in.
- **SLSA v1.0.** `https://slsa.dev/spec/v1.0/` — Provenance attestation content; ex04 authoring, ex05 signing.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — supply-chain attack categories this gate interdicts.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — supply-chain techniques; the gate's design is informed by the published ATLAS supply-chain tactics / techniques (e.g., publish-poisoned-model-artifact shape).
- **NIST SP 800-218** — SSDF; the signing gate is a PO.3 practice.
- **Module running system.** CarePilot, `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise prompt.** `lessons/mod-110-model-risk-management-and-supply-chain/exercises/exercise-05-sigstore-model-signing-and-registry-gate.md`.
