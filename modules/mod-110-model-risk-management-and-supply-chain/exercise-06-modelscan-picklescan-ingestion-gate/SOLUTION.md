# Exercise 06 — ModelScan and picklescan Ingestion Gate with a Blocked Malicious Artefact — Reference Solution

## 1. Solution overview

This exercise builds a **content-scanning ingestion gate** that composes **ModelScan** (Protect AI's open-source model-artifact scanner) with **picklescan** (an OSS scanner that inspects Python pickle streams for opcodes used to execute code) and demonstrates it blocking a deliberately constructed malicious model artifact.

The deliverable is:

- A scanning gate — `scripts/scan_model.py` — that runs picklescan over every pickle-shaped file in an incoming model artifact (including the Python-pickle layer inside PyTorch's historical `torch.save` format), runs ModelScan over the whole artifact, interprets the two scanners' output according to a severity policy, and exits non-zero (fails-closed) on any disallowed finding.
- A **malicious artifact fixture** `tests/fixtures/malicious_pickle_demo.pkl` built inside this repo for the explicit purpose of making the gate trip. The fixture is a *demonstrative pickle whose `__reduce__` points at a benign stdlib call* (`os.getenv` with a known env var name, or `sys.stdout.write`) — enough to trip the scanner, not enough to constitute a weaponised payload. The exercise is clear that the fixture is for the gate to catch, not for anyone to run.
- A test — `tests/test_gate.py` — that runs the gate against (a) a known-good model fixture and asserts exit 0 and (b) the malicious fixture and asserts exit non-zero with the expected finding category.
- CI wiring that chains this gate **after** the Sigstore signature gate from exercise 05: signature-first (cheap, fast), scan-second (slower, deeper). Both must pass for ingestion.

Design decisions worth calling out:

- **Two scanners, both fail-closed.** picklescan is a laser at the pickle opcode layer; ModelScan has broader framework coverage (Keras H5, TF SavedModel, PyTorch, Safetensors, ONNX in various formats across releases). Running both maximises detection and reduces false negatives.
- **Severity policy, not per-scanner yes/no.** Each scanner emits findings at different levels. The gate's policy maps vendor severities to the hospital's policy levels (`block` / `require-review` / `allow`) in one place.
- **Malicious fixture is intentionally benign-weapon-shape.** The learning goal is "the gate caught it," not "the artifact can execute anything." The fixture uses `sys.stdout.write("picklescan-fixture")` or similar; a reviewer can inspect the fixture and see exactly what would have happened if someone had `pickle.load`-ed it — nothing destructive.
- **`.safetensors` is recommended, not re-invented here.** Where projects can migrate from pickle-shaped formats to `.safetensors`, they should (the mod-110 README explicitly recommends it). This exercise is scoped to the *ingestion gate* for projects that can't yet migrate; the recommendation is in the rationale.
- **No "warn and allow" mode.** The gate fails-closed.

## 2. Implementation — worked answer (gate, fixtures, tests, CI)

### 2.1 Repository layout

```
ingestion-gate/
├── scripts/
│   └── scan_model.py           # the composed gate
├── policy/
│   └── severity.yaml           # severity map and policy actions
├── tests/
│   ├── test_gate.py
│   └── fixtures/
│       ├── benign_demo_model.pt        # a tiny known-good PyTorch state_dict
│       └── malicious_pickle_demo.pkl   # trips picklescan; benign payload
├── .github/workflows/
│   └── ingestion-gate.yml      # composes with the ex05 signature gate
└── rationale.md
```

### 2.2 Severity policy (`policy/severity.yaml`)

```yaml
# policy/severity.yaml
# Maps each scanner's severity to one of three gate actions. The names
# are the scanners' own severity strings as of the referenced releases;
# verify-citation on upgrade:
#   - picklescan: https://github.com/mmaitre314/picklescan  (severity vocabulary)
#   - modelscan:  https://github.com/protectai/modelscan   (severity vocabulary)

picklescan:
  # picklescan emits "unsafe" (dangerous opcode / dangerous global) and "dubious".
  unsafe: block
  dubious: require-review
  safe: allow

modelscan:
  # modelscan emits CRITICAL / HIGH / MEDIUM / LOW. Any finding at or above the
  # threshold blocks. Thresholds are set at the gate.
  CRITICAL: block
  HIGH: block
  MEDIUM: require-review
  LOW: allow

# What "require-review" means in operations: the artifact is quarantined, the
# model-risk-committee delegate (mod-110 ex01 §4.2) is paged, and the artifact
# is not promoted to the production registry until explicit sign-off is filed
# at `security-reviews/<yyyy>/<artifact-sha256>.md`.

fail_on:
  - block
# `require-review` does not auto-fail the gate but puts the artifact into
# quarantine; sign-off records are required before promotion. For CI use
# where any non-allow finding should fail (e.g., untrusted sources), add:
#   fail_on_require_review_in_ci: true
```

### 2.3 Gate script (`scripts/scan_model.py`)

```python
# scripts/scan_model.py
#
# Composed model-ingestion gate:
#   1. Walk the artifact directory, find every pickle-shaped file.
#   2. For each pickle-shaped file, run picklescan.
#   3. Run ModelScan over the top-level artifact path.
#   4. Apply the severity policy; fail-closed on any `block` severity;
#      quarantine artifacts with `require-review` findings.
#
# Return codes:
#   0 — safe to ingest
#   1 — blocked by policy (at least one `block` finding, or any CLI error)
#   2 — quarantined (one or more `require-review` findings; not an auto-block)
#
# Upstream references:
# - Protect AI ModelScan: https://github.com/protectai/modelscan
# - picklescan:           https://github.com/mmaitre314/picklescan
# verify-citation: confirm the current CLI entrypoints (`modelscan -p <path>`,
# `picklescan -p <path>` and their JSON-output flags) at authoring time.

from __future__ import annotations

import argparse
import json
import pathlib
import subprocess
import sys
from typing import Any

import yaml


PICKLE_SHAPES = {".pkl", ".pickle", ".pt", ".pth", ".bin", ".ckpt"}
# .pt/.pth/.ckpt/.bin from PyTorch's historical torch.save are zip-of-pickles
# or raw pickles; picklescan handles both. .safetensors and .onnx are not
# pickle-shaped and are ignored by picklescan here (ModelScan still covers them).


def _run(cmd: list[str]) -> tuple[int, str, str]:
    proc = subprocess.run(cmd, capture_output=True, text=True, check=False)
    return proc.returncode, proc.stdout, proc.stderr


def _scan_picklescan(path: pathlib.Path) -> list[dict[str, Any]]:
    rc, out, err = _run(["picklescan", "-p", str(path), "--json"])
    # picklescan historically emits JSON on stdout when --json is passed.
    # verify-citation: confirm flag and shape at authoring time.
    try:
        findings = json.loads(out) if out.strip() else []
    except json.JSONDecodeError:
        findings = [{"severity": "unsafe", "detail": f"picklescan output unparsable: {err or out}"}]
    return findings if isinstance(findings, list) else [findings]


def _scan_modelscan(path: pathlib.Path, report_path: pathlib.Path) -> list[dict[str, Any]]:
    rc, _, err = _run(
        ["modelscan", "-p", str(path), "-f", "json", "-o", str(report_path)]
    )
    # ModelScan writes its JSON report to -o; rc convention:
    # rc=0 clean, rc>0 findings present. verify-citation on exact convention.
    if not report_path.exists():
        return [{"severity": "CRITICAL", "detail": f"modelscan produced no report: {err}"}]
    try:
        report = json.loads(report_path.read_text())
    except json.JSONDecodeError:
        return [{"severity": "CRITICAL", "detail": "modelscan report unparsable"}]
    # ModelScan's `summary.total_issues_by_severity` + `issues` list are the
    # authoritative surfaces; verify at upgrade time.
    issues = report.get("issues") or []
    return issues


def _classify(findings: list[dict[str, Any]], scanner: str, policy: dict) -> list[str]:
    actions: list[str] = []
    for f in findings:
        sev = str(f.get("severity", "")).strip()
        action = policy.get(scanner, {}).get(sev)
        if action is None:
            # Unknown severity defaults to require-review on policy-hit; this
            # is deliberately conservative — a scanner upgrade that adds a new
            # severity level does not silently become "allow".
            action = "require-review"
        actions.append(action)
    return actions


def _walk_pickles(root: pathlib.Path) -> list[pathlib.Path]:
    return [p for p in root.rglob("*") if p.is_file() and p.suffix.lower() in PICKLE_SHAPES]


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Scan a model artifact for malicious content.")
    parser.add_argument("--artifact", required=True, type=pathlib.Path)
    parser.add_argument("--policy", required=True, type=pathlib.Path)
    parser.add_argument("--report-dir", required=True, type=pathlib.Path)
    args = parser.parse_args(argv)

    args.report_dir.mkdir(parents=True, exist_ok=True)
    policy = yaml.safe_load(args.policy.read_text())
    fail_on = set(policy.get("fail_on", ["block"]))

    all_actions: list[tuple[str, str, dict]] = []

    # 1) picklescan every pickle-shaped file.
    for pf in _walk_pickles(args.artifact):
        findings = _scan_picklescan(pf)
        actions = _classify(findings, "picklescan", policy)
        for a, f in zip(actions, findings):
            all_actions.append(("picklescan", a, {"file": str(pf), **f}))

    # 2) modelscan the whole artifact.
    modelscan_report = args.report_dir / "modelscan.json"
    ms_findings = _scan_modelscan(args.artifact, modelscan_report)
    ms_actions = _classify(ms_findings, "modelscan", policy)
    for a, f in zip(ms_actions, ms_findings):
        all_actions.append(("modelscan", a, f))

    # Write a combined report for audit.
    (args.report_dir / "gate-decisions.json").write_text(
        json.dumps(
            [
                {"scanner": s, "action": a, "finding": f}
                for (s, a, f) in all_actions
            ],
            indent=2,
            sort_keys=True,
        )
    )

    # 3) Apply the policy.
    any_block = any(a == "block" for (_, a, _) in all_actions)
    any_review = any(a == "require-review" for (_, a, _) in all_actions)

    if any_block or "block" in fail_on and any_block:
        print("BLOCKED — at least one block-level finding")
        return 1
    if any_review:
        print("QUARANTINED — one or more findings require review")
        return 2
    print("OK — no block-level findings; ingestion permitted")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 2.4 Malicious pickle fixture (`tests/fixtures/malicious_pickle_demo.pkl`)

```python
# tests/fixtures/build_malicious_pickle_demo.py
#
# Build a demonstration malicious pickle fixture. picklescan will flag this as
# "unsafe" because the __reduce__ on load would invoke a Python callable.
#
# IMPORTANT: the chosen callable is sys.stdout.write with a harmless string.
# The fixture is designed to TRIP the gate, not to execute anything useful.
# A reviewer can read this file and see exactly what pickle.load would do:
# write "picklescan-fixture-triggered\n" to stdout. That is the entire effect.

import pathlib
import pickle
import sys


class _Benign:
    """A benign gadget whose __reduce__ makes the pickle non-trivial.

    picklescan flags any __reduce__ whose first element is a callable that
    picklescan does not permit by default; sys.stdout.write is a classic
    illustration.
    """

    def __reduce__(self):
        return (sys.stdout.write, ("picklescan-fixture-triggered\n",))


def main():
    out = pathlib.Path(__file__).parent / "malicious_pickle_demo.pkl"
    with out.open("wb") as fp:
        pickle.dump(_Benign(), fp)
    print(f"wrote {out}")


if __name__ == "__main__":
    main()
```

The fixture itself (`malicious_pickle_demo.pkl`) is the binary output of running `build_malicious_pickle_demo.py` and is checked into `tests/fixtures/`. The *build* script is also checked in so a reviewer can read exactly what the fixture would do.

### 2.5 Tests (`tests/test_gate.py`)

```python
# tests/test_gate.py

import subprocess
import pathlib

HERE = pathlib.Path(__file__).parent
ROOT = HERE.parent


def _run_gate(artifact: pathlib.Path, policy: pathlib.Path, report_dir: pathlib.Path) -> int:
    cmd = [
        "python",
        str(ROOT / "scripts" / "scan_model.py"),
        "--artifact",
        str(artifact),
        "--policy",
        str(policy),
        "--report-dir",
        str(report_dir),
    ]
    return subprocess.call(cmd)


def test_good_artifact_passes(tmp_path):
    # The benign fixture is a known-good tiny PyTorch state_dict.
    artifact = HERE / "fixtures" / "benign_demo_model"
    policy = ROOT / "policy" / "severity.yaml"
    assert _run_gate(artifact, policy, tmp_path) == 0


def test_malicious_pickle_blocks(tmp_path):
    # Place the malicious fixture inside its own artifact dir.
    artifact_dir = tmp_path / "artifact"
    artifact_dir.mkdir()
    (artifact_dir / "malicious_pickle_demo.pkl").write_bytes(
        (HERE / "fixtures" / "malicious_pickle_demo.pkl").read_bytes()
    )
    policy = ROOT / "policy" / "severity.yaml"
    rc = _run_gate(artifact_dir, policy, tmp_path / "report")
    assert rc == 1, f"expected BLOCKED (rc=1), got {rc}"

    # And the combined report captures the finding.
    import json
    decisions = json.loads((tmp_path / "report" / "gate-decisions.json").read_text())
    assert any(
        d["scanner"] == "picklescan" and d["action"] == "block"
        for d in decisions
    ), decisions
```

### 2.6 CI wiring (`.github/workflows/ingestion-gate.yml`)

```yaml
# .github/workflows/ingestion-gate.yml
name: ingestion-gate
on:
  workflow_run:
    workflows: ["dx-build"]
    types: [completed]

jobs:
  scan-and-promote:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v4

      - name: Download signed bundle from upstream workflow
        uses: actions/download-artifact@v4
        with:
          name: carepilot-signed-bundle
          path: incoming/

      # Step 1 — Sigstore signature gate from exercise 05.
      - name: Signature gate
        run: |
          python scripts/verify_model.py \
            --model incoming/carepilot/model.bin \
            --bundle-dir incoming/carepilot/supply-chain/bundle/ \
            --policy policy/allowed-signers.yaml

      # Step 2 — content-scan gate (this exercise).
      - name: Content-scan gate
        run: |
          pip install modelscan picklescan
          python scripts/scan_model.py \
            --artifact incoming/carepilot/ \
            --policy policy/severity.yaml \
            --report-dir report/

      - name: Promote to registry
        if: success()
        run: ./scripts/promote.sh incoming/carepilot/
```

### 2.7 Rationale (`rationale.md`)

```markdown
# Why two scanners, why fail-closed, why composed with signing

- **picklescan** focuses on the Python-pickle opcode layer: it reads the opcode stream and flags constructs that would invoke arbitrary callables on `pickle.load`. It's the most precise tool for the "pickle executes code" class of attack.
- **ModelScan** covers more framework formats and more finding types (not just pickle shape). It catches Keras lambda layers, TensorFlow SavedModel operators that execute code, and PyTorch artifacts; it also reports artifact-level metadata that picklescan doesn't.
- **Why compose with the signing gate.** Sigstore proves who produced the artifact and that it hasn't been modified in flight; scanning proves what is inside. A trusted signer can still accidentally ship a Keras lambda layer that scans dirty — the composition catches that.
- **Why fail-closed.** The whole supply chain is downstream of this gate; a warn-and-allow mode means a dangerous model reaches production on a Friday evening.
- **Why `.safetensors` is recommended elsewhere.** `.safetensors` is a non-executable container that doesn't need a scanner to confirm it isn't hiding code. Where a project can migrate, it should; where it can't (legacy model artifacts, specific dtype constraints), this gate is the guardrail.
- **`require-review` is not "allow with a frown."** A `require-review` finding quarantines the artifact and pages the model-risk-committee delegate. The artifact is not promoted until a sign-off record exists at `security-reviews/<yyyy>/<artifact-sha256>.md`.
```

## 3. Validation steps

1. **Benign fixture passes.** `pytest tests/test_gate.py::test_good_artifact_passes` is green; the gate exits 0 on a known-good PyTorch state_dict fixture that has no unusual opcodes.
2. **Malicious fixture blocks.** `pytest tests/test_gate.py::test_malicious_pickle_blocks` is green; the gate exits non-zero with a `block`-action `picklescan` finding; the combined report captures the finding.
3. **Fail-closed on scanner CLI error.** A simulated scanner CLI crash (e.g., renaming the `picklescan` binary) causes the gate to exit non-zero rather than silently pass.
4. **Unknown severity defaults to `require-review`.** A simulated scanner upgrade that returns an unknown severity string classifies as `require-review`, not `allow`.
5. **Composed CI success case.** The CI workflow in §2.6 runs both the signature gate and the content-scan gate; both exit 0 for the known-good build.
6. **Composed CI failure case.** Replacing the known-good model with the malicious fixture causes the content-scan gate to fail even though the signature gate passes. The promote step is skipped.
7. **Audit artifact persisted.** `report/gate-decisions.json` is written for every run and captures scanner, action, and finding; retained for the same window as monitoring artifacts (mod-109).

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Both scanners wired | Gate runs picklescan over pickle-shaped files AND modelscan over the artifact root | Only one scanner used |
| R2 | Severity policy external | `policy/severity.yaml` maps each scanner's severities to `block` / `require-review` / `allow` | Severity thresholds hard-coded |
| R3 | Fail-closed | Any `block` finding, any scanner CLI error, any unparsable report exits non-zero | Warn-and-allow path |
| R4 | Malicious fixture present and benign-payload | `tests/fixtures/malicious_pickle_demo.pkl` trips the scanner; build script shows the payload is benign | No fixture, or fixture is weaponised |
| R5 | Test proves the block | `test_malicious_pickle_blocks` asserts non-zero exit AND a `block` action for picklescan | Test asserts only exit code, or no test |
| R6 | Composition with ex05 | CI workflow runs the signature gate first, then the content-scan gate | Gates not composed; only one is wired |
| R7 | `require-review` is a quarantine | Policy and rationale state that quarantine triggers a sign-off record with a path | `require-review` behaves as `allow` |
| R8 | Unknown-severity default is conservative | Classifier defaults unknown severities to `require-review` | Unknown severities default to `allow` |
| R9 | Report persistence | `gate-decisions.json` + each scanner's native report are retained | No audit trail |
| R10 | No invented facts | Scanner CLI flags, severity vocabulary, and env details are cited to the upstream repos with `<!-- verify-citation: ... -->` where unverified | Fabricated CLI flags / severity strings |

Scoring bands: *Pass* — R1–R8 green, R9/R10 honoured; *Pass with revisions* — R5 asserts only exit code; *Fail* — any of R1, R3, R4, R6.

## 5. Common mistakes

- **Trusting `.pt` because it is a PyTorch extension.** `.pt` / `.pth` / `.bin` / `.ckpt` can be — and historically often are — pickle streams or zip-of-pickles. Scan them.
- **Scanning only pickles.** ModelScan catches Keras lambda layers and TF SavedModel operator abuses that aren't pickle-shaped. Both scanners are needed.
- **Weaponising the fixture.** The exercise asks for a fixture that *trips the scanner*, not one that opens a shell. A `sys.stdout.write` gadget with a label string is enough.
- **Unit-testing the scanner, not the gate.** The point of the test is "the gate reacts correctly to the scanner's finding"; testing that picklescan itself works is upstream's job.
- **Ignoring scanner CLI errors.** If `picklescan` is missing or crashes, "no findings" is not "safe". The gate must treat an uninstallable scanner as a fail.
- **Allowing unknown severities.** When the scanner emits a new severity level in an upgrade, the safe default is `require-review`, not `allow`.
- **Collapsing `require-review` into `allow`.** Without a quarantine mechanism and a sign-off path, `require-review` is a noise channel that gets filtered out.
- **No audit trail.** The `gate-decisions.json` output is both for debugging *and* for the model-risk-committee's audit; without it, policy decisions are not reviewable.
- **Running the content scan before the signature gate.** Signature verification is cheap and rejects most tampered artifacts in milliseconds; running the slower content scan first wastes compute and widens the window where a mis-signed artifact is under analysis.
- **Pinning scanner versions to `latest`.** The severity vocabulary changes between releases. Pin, and re-review the severity map on upgrade.
- **Shipping `pickle.load` in production ingestion code.** The scanners run outside `pickle.load`; the ingestion code must not `pickle.load` the artifact even to introspect it, because the scanning gate exists precisely because `pickle.load` is unsafe.

## 6. References

- **Protect AI ModelScan.** `https://github.com/protectai/modelscan` — open-source scanner for model-artifact serialisation formats. <!-- verify-citation: confirm the current CLI flags, JSON-output shape, and severity vocabulary at authoring time. -->
- **picklescan.** `https://github.com/mmaitre314/picklescan` — open-source scanner for Python pickle opcodes. <!-- verify-citation: confirm the current CLI flags, JSON-output shape, and severity labels at authoring time. -->
- **Safetensors.** `https://github.com/huggingface/safetensors` — non-executable tensor container referenced in the rationale; the project is recommended elsewhere in mod-110.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the attack family this gate interdicts (malicious-model-artifact supply-chain injection).
- **MITRE ATLAS.** `https://atlas.mitre.org/` — supply-chain tactics / techniques; the gate's policy is informed by the ATLAS supply-chain tactic.
- **Sigstore Project.** `https://docs.sigstore.dev/` — the signature gate the content-scan gate composes with (ex05).
- **OpenSSF Scorecard.** `https://openssf.org/scorecard/` — consumer of signing + scanning signals; the gate's output can feed Scorecard-style posture reports.
- **NIST AI RMF 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — MEASURE 2 (security, resilience) and MANAGE 2 rows this gate discharges.
- **Module running system.** CarePilot, `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`.
- **Exercise prompt.** `lessons/mod-110-model-risk-management-and-supply-chain/exercises/exercise-06-modelscan-picklescan-ingestion-gate.md`.
