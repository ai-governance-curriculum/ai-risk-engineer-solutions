# Exercise 04 — ML-BOM, SPDX 3.0 AI Profile, and SLSA Provenance for One Deployed Model — Reference Solution

## 1. Solution overview

This exercise authors three supply-chain artifacts for **one** deployed model — CarePilot's RAG inference container — using three current community standards:

- A **CycloneDX ML-BOM** (OWASP CycloneDX bill-of-materials spec with the Machine-Learning extension) at JSON schema 1.6, naming the model, dataset, and model-card components and the relationships between them.
- An **SPDX 3.0 AI Profile** document covering the same model as a cross-ecosystem complement to the CycloneDX ML-BOM, at SPDX specification 3.0 (which adds the AI and Dataset profiles alongside the Core profile).
- A **SLSA v1.0 build provenance** attestation for the inference container, following the SLSA Provenance v1 predicate schema.

The deliverable is a directory of artifacts under `artifacts/carepilot/supply-chain/` plus a short rationale document that explains **why three artifacts**, **what each one is authoritative for**, and **how they compose** at the ingestion gate (mod-110 exercise 05 / exercise 06). The three artifacts are not redundant — CycloneDX ML-BOM is the inventory with the most-mature ML extensions today; SPDX 3.0 is the cross-ecosystem standard that many regulated procurement processes already consume; SLSA Provenance is the attestation of **how** the inference container was built. A consumer who ingests only one of the three gets a partial picture.

The reference solution:

- Writes complete JSON for the CycloneDX ML-BOM, SPDX 3.0 AI-profile document, and SLSA Provenance in-line below, so the student has a copy-adaptable starting point rather than a prose description.
- Uses placeholder identifiers for hashes, URIs, and signing-key references (`"sha256:TODO-run-time-hash"`, `pkg:<purl>` shapes) and explains how they are populated at build time.
- Names the exact **schema versions** used: CycloneDX 1.6, SPDX 3.0 (which supersedes the AI-BOM draft previously shipped as SPDX 3.0 draft under a different name), SLSA 1.0 Provenance predicate.
- States what each artifact is **not** — CycloneDX ML-BOM is not a SLSA attestation; SPDX 3.0 is not a signing scheme; SLSA Provenance is not a BOM. Mixing them up is the module's primary common mistake.
- Keeps claims tied to the upstream project documentation (`cyclonedx.org`, `spdx.dev`, `slsa.dev`) and flags with `<!-- verify-citation: ... -->` any specific field name or version string that must be confirmed against the current spec at authoring time.

Design decisions worth calling out:

- **Three artifacts, not one.** Procurement, security, and governance stakeholders want different cuts. Rather than force one artifact to do all three jobs, the solution authors three narrowly-scoped artifacts and explains the composition.
- **Hashes are placeholders.** Real content-addressable digests only exist after the build runs. The solution uses `sha256:TODO-run-time-hash`-shaped placeholders and names the CI step that populates them. A submission that bakes in a fake digest would be worse than a placeholder.
- **Signing is deferred to exercise 05.** The attestations are *produced* here; they are *signed and verified* in exercise 05 with Sigstore. The solution explicitly hands off to that exercise rather than inventing a signing block.
- **One deployed model.** Scope is the CarePilot RAG inference container; the base-model weights (consumed from the hosted vendor API per exercise 02) are *components* in the ML-BOM, not the top-level subject.

## 2. Implementation — worked answer (authored artifacts)

### 2.1 Repository layout

```
artifacts/carepilot/supply-chain/
├── ml-bom.cdx.json                 # CycloneDX 1.6 ML-BOM
├── spdx-3-ai.json                  # SPDX 3.0 document with the AI profile
├── slsa-provenance.v1.intoto.json  # SLSA v1.0 Provenance in-toto statement
├── sbom-vs-mlbom-note.md           # scope note: when to use SBOM vs ML-BOM vs SPDX AI profile
└── rationale.md                    # the "why three artifacts" document
```

### 2.2 CycloneDX ML-BOM (`ml-bom.cdx.json`)

CycloneDX 1.6 ships native support for machine-learning-model components via the `type: "machine-learning-model"` component type and the `modelCard` property on components. The following is a complete-shape reference the student adapts. Field names follow CycloneDX 1.6 JSON schema.

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "serialNumber": "urn:uuid:TODO-generate-at-build",
  "version": 1,
  "metadata": {
    "timestamp": "TODO-set-at-build",
    "tools": {
      "components": [
        {
          "type": "application",
          "name": "cyclonedx-cli",
          "version": "TODO-pin-version"
        }
      ]
    },
    "authors": [
      {"name": "ai-risk-engineering", "email": "ai-risk@example.invalid"}
    ],
    "component": {
      "bom-ref": "pkg:oci/carepilot-rag@sha256:TODO-run-time-hash",
      "type": "container",
      "name": "carepilot-rag",
      "version": "2025.q1.0",
      "purl": "pkg:oci/carepilot-rag@sha256:TODO-run-time-hash",
      "description": "CarePilot RAG inference container — hospital triage-support decision-support assistant (not placed on US market as a device; see mod-101 context)."
    }
  },
  "components": [
    {
      "bom-ref": "model:carepilot-llm-adapter",
      "type": "machine-learning-model",
      "name": "carepilot-llm-adapter",
      "version": "2025.q1.0",
      "description": "The thin adapter fine-tune applied by the regional health system over the hosted-vendor base LLM. Hosted-vendor base model itself is referenced as a dependency (see pkg:thirdparty/HostedLLM-Vendor-Z below).",
      "modelCard": {
        "modelParameters": {
          "approach": {
            "type": "supervised"
          },
          "task": "clinical triage acuity classification with free-text rationale (retrieval-augmented)",
          "architectureFamily": "TODO-fill-at-authoring",
          "modelArchitecture": "TODO-fill-at-authoring",
          "datasets": [
            {"ref": "dataset:carepilot-triage-labels"}
          ],
          "inputs": [
            {"format": "text", "description": "clinician-entered chief complaint + structured intake fields"}
          ],
          "outputs": [
            {"format": "json", "description": "ranked acuity + rationale"}
          ]
        },
        "quantitativeAnalysis": {
          "performanceMetrics": [
            {
              "type": "triage-concordance-24h",
              "value": "TODO-populate-from-evaluation-run",
              "slice": "all"
            },
            {
              "type": "false-low-acuity-rate",
              "value": "TODO-populate-from-evaluation-run",
              "slice": "all"
            }
          ],
          "graphics": {
            "description": "ROC and calibration plots are stored out-of-band in the evaluation report; not embedded in the BOM to keep the BOM small."
          }
        },
        "considerations": {
          "users": ["ED clinicians", "triage nurses"],
          "useCases": ["ED triage decision-support in a HITL flow"],
          "technicalLimitations": [
            "Paediatric < 2y out of scope",
            "Obstetric emergencies have a protocol override",
            "Mass-casualty events out of scope"
          ],
          "performanceTradeoffs": [
            "Lower temperature preferred for determinism; reduces diversity of rationale phrasing"
          ],
          "ethicalConsiderations": [
            {
              "name": "subgroup-performance",
              "mitigationStrategies": "mod-106 subgroup analysis + the post-market monitoring plan"
            }
          ],
          "fairnessAssessments": [
            {
              "groupAtRisk": "underserved demographic subgroup identified in mod-106 exercise 02",
              "mitigationStrategy": "subgroup-stratified acceptance criteria in the PCCP (mod-110 ex03)"
            }
          ]
        }
      }
    },
    {
      "bom-ref": "pkg:thirdparty/HostedLLM-Vendor-Z@variant-X",
      "type": "machine-learning-model",
      "name": "HostedLLM-Vendor-Z base model (variant-X)",
      "version": "variant-X",
      "description": "Hosted base LLM consumed at inference time per the third-party due diligence in mod-110 ex02. Model weights are not redistributed; this component records the dependency only.",
      "externalReferences": [
        {"type": "documentation", "url": "https://vendor.example.invalid/model-card/variant-X"}
      ]
    },
    {
      "bom-ref": "dataset:carepilot-triage-labels",
      "type": "data",
      "name": "carepilot-triage-labels",
      "version": "2025.q1",
      "description": "Hospital-ED labelled triage cases; labelled per the reference-dataset SOP (mod-110 ex03 GMLP 5).",
      "externalReferences": [
        {"type": "documentation", "url": "artifacts/carepilot/data/reference-dataset-card.md"}
      ]
    },
    {
      "bom-ref": "corpus:carepilot-retrieval-corpus",
      "type": "data",
      "name": "carepilot-retrieval-corpus",
      "version": "2025.q1",
      "description": "Retrieval corpus: ED triage protocols revision X + clinical-handbook ED sections revision Y."
    }
  ],
  "dependencies": [
    {
      "ref": "pkg:oci/carepilot-rag@sha256:TODO-run-time-hash",
      "dependsOn": [
        "model:carepilot-llm-adapter",
        "corpus:carepilot-retrieval-corpus"
      ]
    },
    {
      "ref": "model:carepilot-llm-adapter",
      "dependsOn": [
        "pkg:thirdparty/HostedLLM-Vendor-Z@variant-X",
        "dataset:carepilot-triage-labels"
      ]
    }
  ]
}
```

Where the spec version or a specific property name has changed between revisions, the student is expected to re-check. <!-- verify-citation: confirm all field names (`modelCard`, `modelParameters.approach`, `quantitativeAnalysis.performanceMetrics[].type`, etc.) against the current CycloneDX 1.6 JSON schema at `cyclonedx.org/docs/1.6/json/`. -->

### 2.3 SPDX 3.0 document with the AI profile (`spdx-3-ai.json`)

SPDX 3.0 is organised around **profiles**: Core, Software, Dataset, AI, Build, Licensing, Lite, Security, among others. A document opts in to profiles by declaring `profileConformance`. The AI profile adds the `AIPackage` element and fields that describe AI-system components.

```json
{
  "@context": "https://spdx.org/rdf/3.0.0/spdx-context.jsonld",
  "@graph": [
    {
      "type": "SpdxDocument",
      "spdxId": "https://example.invalid/spdx/carepilot/2025-q1",
      "name": "CarePilot RAG inference — SPDX 3.0 AI-profile document",
      "specVersion": "3.0.0",
      "profileConformance": ["core", "software", "dataset", "ai"],
      "creationInfo": {
        "specVersion": "3.0.0",
        "created": "TODO-set-at-build",
        "createdBy": ["https://example.invalid/agent/ai-risk-engineering"],
        "createdUsing": ["https://example.invalid/tool/TODO-pin-spdx-tool-version"]
      },
      "rootElement": ["https://example.invalid/spdx/carepilot/2025-q1/package/carepilot-rag"]
    },
    {
      "type": "software.Package",
      "spdxId": "https://example.invalid/spdx/carepilot/2025-q1/package/carepilot-rag",
      "name": "carepilot-rag",
      "packageVersion": "2025.q1.0",
      "primaryPurpose": "container",
      "downloadLocation": "oci://example.invalid/carepilot-rag@sha256:TODO-run-time-hash"
    },
    {
      "type": "ai.AIPackage",
      "spdxId": "https://example.invalid/spdx/carepilot/2025-q1/aipackage/carepilot-llm-adapter",
      "name": "carepilot-llm-adapter",
      "packageVersion": "2025.q1.0",
      "primaryPurpose": "aiModel",
      "ai.autonomyType": "humanInTheLoop",
      "ai.useCases": [
        "ED triage decision-support for trained clinicians, with clinician sign-off before patient-facing action"
      ],
      "ai.limitations": [
        "Paediatric < 2y out of scope",
        "Obstetric emergencies have a protocol override",
        "Mass-casualty events out of scope"
      ],
      "ai.safetyRiskAssessment": "medium",
      "ai.modelExplainability": "retrieval-grounded rationale emitted alongside the acuity recommendation",
      "ai.standardCompliance": ["NIST-AI-RMF-1.0", "FDA-GMLP-2021"],
      "ai.hyperparameters": {"temperature": "TODO-pin-at-authoring"},
      "ai.modelDataPreprocessing": "retrieval over the scoped corpus + PII redaction per mod-107 ex05",
      "ai.informationAboutTraining": "Local fine-tune over hosted-vendor base LLM on quarterly ED-labelled cases; details in mod-110 ex03 PCCP",
      "ai.informationAboutApplication": "Deployed inside hospital network; SSO-authenticated; HITL"
    },
    {
      "type": "dataset.DatasetPackage",
      "spdxId": "https://example.invalid/spdx/carepilot/2025-q1/dataset/carepilot-triage-labels",
      "name": "carepilot-triage-labels",
      "packageVersion": "2025.q1",
      "primaryPurpose": "dataset",
      "dataset.datasetType": "structuredText",
      "dataset.intendedUse": "training + evaluation for CarePilot-adapter fine-tune",
      "dataset.knownBias": "hospital-catchment demographic distribution; see `reference-dataset-card.md` for documented coverage gaps",
      "dataset.sensitiveDataPresent": "yes — PHI; access gated by hospital policy",
      "dataset.confidentialityLevel": "amber"
    },
    {
      "type": "Relationship",
      "from": "https://example.invalid/spdx/carepilot/2025-q1/package/carepilot-rag",
      "relationshipType": "dependsOn",
      "to": ["https://example.invalid/spdx/carepilot/2025-q1/aipackage/carepilot-llm-adapter"]
    },
    {
      "type": "Relationship",
      "from": "https://example.invalid/spdx/carepilot/2025-q1/aipackage/carepilot-llm-adapter",
      "relationshipType": "trainedOn",
      "to": ["https://example.invalid/spdx/carepilot/2025-q1/dataset/carepilot-triage-labels"]
    }
  ]
}
```

<!-- verify-citation: confirm the current SPDX 3.0 JSON-LD shape, element names (`ai.AIPackage`, `ai.autonomyType`, `ai.useCases`, `ai.limitations`, `ai.safetyRiskAssessment`, `ai.modelExplainability`, `ai.standardCompliance`, `dataset.DatasetPackage`), and the `profileConformance` value set, against the current SPDX 3.0 specification at `spdx.github.io/spdx-spec/v3.0.0/`. -->

### 2.4 SLSA v1.0 build provenance (`slsa-provenance.v1.intoto.json`)

SLSA Provenance v1 is an in-toto Statement with `predicateType = https://slsa.dev/provenance/v1` wrapping the `buildDefinition` (the build inputs) and `runDetails` (metadata about the execution).

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "name": "carepilot-rag",
      "digest": {
        "sha256": "TODO-run-time-hash-of-the-container-image"
      }
    }
  ],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://example.invalid/build-types/carepilot-rag/v1",
      "externalParameters": {
        "sourceRepo": "https://example.invalid/repo/carepilot",
        "ref": "refs/tags/2025.q1.0",
        "commit": "TODO-populated-by-ci",
        "buildScript": ".github/workflows/dx-build.yml"
      },
      "internalParameters": {
        "builderImage": "ghcr.io/example-invalid/builder@sha256:TODO-pin",
        "trainingRecipe": "training/dx/recipe.yaml@TODO-commit"
      },
      "resolvedDependencies": [
        {
          "uri": "git+https://example.invalid/repo/carepilot",
          "digest": {"sha1": "TODO-populated-by-ci"}
        },
        {
          "uri": "pkg:thirdparty/HostedLLM-Vendor-Z@variant-X",
          "annotations": {"note": "runtime-only dependency; weights not redistributed"}
        },
        {
          "uri": "artifact:carepilot/data/reference-dataset-card.md",
          "digest": {"sha256": "TODO-populated-by-ci"}
        }
      ]
    },
    "runDetails": {
      "builder": {
        "id": "https://example.invalid/builders/github-hosted/v1"
      },
      "metadata": {
        "invocationId": "TODO-populated-by-ci",
        "startedOn": "TODO-populated-by-ci",
        "finishedOn": "TODO-populated-by-ci"
      },
      "byproducts": [
        {
          "name": "ml-bom.cdx.json",
          "digest": {"sha256": "TODO-populated-by-ci"}
        },
        {
          "name": "spdx-3-ai.json",
          "digest": {"sha256": "TODO-populated-by-ci"}
        }
      ]
    }
  }
}
```

The attestation is wrapped in a DSSE envelope and signed in **exercise 05** (Sigstore cosign). <!-- verify-citation: confirm the SLSA Provenance v1 predicate field layout (`buildDefinition.{buildType, externalParameters, internalParameters, resolvedDependencies}`, `runDetails.{builder, metadata, byproducts}`) against the current slsa.dev/spec/v1.0/provenance page. -->

### 2.5 Scope note — when each artifact is authoritative (`sbom-vs-mlbom-note.md`)

```markdown
# When to use each artifact — scope boundaries

| Artifact | Authoritative for | Not authoritative for |
|---|---|---|
| CycloneDX 1.6 ML-BOM | Enumeration of ML components (models, datasets, corpora) and their relationships; machine-learning model-card fields | Build provenance; license-ecosystem interop with SPDX-consuming procurement tools |
| SPDX 3.0 AI profile | Cross-ecosystem license + supply-chain exchange, AIPackage + DatasetPackage semantics, procurement-tool ingestion | Deep ML-model-card fields (CycloneDX's `modelCard` is richer today) |
| SLSA Provenance v1 | How the artifact was built — builder identity, source commit, resolved dependencies, build parameters, byproducts | What is inside the artifact (that is the BOM's job) |

**Composition at the gate.** Mod-110 exercise 05 and exercise 06 ingest all three: SLSA provenance establishes **where** the artifact came from, the ML-BOM / SPDX document names **what** is in it, and the Sigstore signature plus the scanners (ModelScan, picklescan) verify integrity.
```

### 2.6 Rationale document (`rationale.md`)

```markdown
# Why three artifacts

- **CycloneDX ML-BOM (1.6).** Richest ML-model-card shape available today in a BOM standard; strong tooling (Dependency-Track consumes it).
- **SPDX 3.0 AI profile.** Cross-ecosystem; many regulated procurement processes already ingest SPDX; AI profile specifically adds AIPackage + DatasetPackage so the SBOM world understands "this is a model, this is training data."
- **SLSA Provenance v1.** A BOM says what you have; SLSA says how you got it. Procurement needs both.

**Signing.** All three attestations are signed in mod-110 ex05. The ingestion gate in mod-110 ex05 fails-closed if any of the three is missing or signature-invalid; the content-scanning gate in mod-110 ex06 fails-closed if the model binary itself fails a safety scan.

**Base-model weights.** The hosted vendor's base-model weights are a runtime dependency (not redistributed), captured in CycloneDX as `pkg:thirdparty/...`, in SPDX as a referenced AIPackage with an external URL, and in SLSA as a `resolvedDependencies` entry with the annotation note. Mod-110 ex02 is the due-diligence packet for the vendor itself.

**What is still gap-y in the current tooling.** CycloneDX ML extension and SPDX 3.0 AI profile are the current community state; both are evolving, and the richest fidelity is in CycloneDX today. Where a procurement tool has not yet caught up to SPDX 3.0, the ML-BOM is the operational artifact.
```

## 3. Validation steps

1. **Three artifact files produced.** `ml-bom.cdx.json`, `spdx-3-ai.json`, `slsa-provenance.v1.intoto.json` all present in `artifacts/carepilot/supply-chain/`.
2. **CycloneDX schema validity.** `cyclonedx validate --input-file artifacts/carepilot/supply-chain/ml-bom.cdx.json --input-version v1_6` passes. Placeholder hashes (`sha256:TODO-run-time-hash`) are accepted by the schema (strings); real hashes are populated by CI before signing.
3. **SPDX schema validity.** `pyspdx` or the SPDX reference tooling validates the document against the SPDX 3.0 JSON-LD schema; the document declares `profileConformance: [core, software, dataset, ai]`.
4. **SLSA statement is a well-formed in-toto Statement.** The document has `_type: https://in-toto.io/Statement/v1`, `subject[].digest`, `predicateType: https://slsa.dev/provenance/v1`, and both `buildDefinition` and `runDetails` are present.
5. **Cross-artifact references align.** The ML-BOM's `pkg:oci/carepilot-rag@sha256:...` matches the SLSA statement's `subject[0].digest.sha256` and the SPDX package `downloadLocation` oci reference. A reviewer greps for the digest placeholder and confirms consistency (same placeholder; real digests are populated at CI time).
6. **No invented facts.** Spec-level field names outside the Notes are cited with `<!-- verify-citation: ... -->` where unverified; no fabricated vendor model cards or dataset statistics appear.
7. **Signing is handed off, not simulated.** `rationale.md` explicitly defers signing to exercise 05; there is no fabricated DSSE envelope with a fake key.
8. **Scope note is present.** `sbom-vs-mlbom-note.md` names what each artifact is authoritative for and what it is not.

## 4. Rubric — review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Three artifact files present | ML-BOM, SPDX 3.0 AI, SLSA Provenance all present and non-empty | Any missing or empty |
| R2 | CycloneDX 1.6 ML-BOM schema valid | `bomFormat: CycloneDX`, `specVersion: 1.6`, components include at least one `type: machine-learning-model` with a populated `modelCard` | Wrong spec version; no ML-typed component |
| R3 | SPDX 3.0 document shape | `specVersion: 3.0.0`, `profileConformance` declares the AI and Dataset profiles, at least one `ai.AIPackage` and one `dataset.DatasetPackage` present | Uses SPDX 2.x or omits the AI profile |
| R4 | SLSA v1 Provenance shape | In-toto Statement with `predicateType: https://slsa.dev/provenance/v1`, `buildDefinition` and `runDetails` populated | Uses SLSA 0.1 / 0.2 or omits `buildDefinition` / `runDetails` |
| R5 | Cross-artifact alignment | Same subject digest placeholder used across all three; same component names referenced in BOM and SPDX | Digests mismatch; components named inconsistently |
| R6 | Placeholders honest | Where real values are not known (hashes, commit SHAs, timestamps), the solution uses labelled `TODO-*` placeholders and names the CI step that populates them | Fabricated hashes or commits |
| R7 | Scope note present | `sbom-vs-mlbom-note.md` names what each artifact is authoritative for | No scope note; artifacts treated as redundant |
| R8 | Signing handed off | Rationale defers signing to exercise 05 | Fake signatures or ad-hoc signing block invented |
| R9 | Linkage to other exercises | References mod-110 ex02 (vendor), ex03 (PCCP), ex05 (signing), ex06 (ingestion scan), mod-107 ex05 (PII redaction), mod-106 (bias) | Floats free of the module |
| R10 | No invented facts | Spec field names carry `<!-- verify-citation: ... -->` where not from a referenced primary source; no fabricated model-card numbers | Fabricated performance numbers, dataset counts, vendor details |

Scoring bands: *Pass* — R1–R8 green, R9/R10 honoured; *Pass with revisions* — R5 misalignment (fix is mechanical); *Fail* — any of R1, R2, R3, R4.

## 5. Common mistakes

- **Treating an SBOM as an ML-BOM.** A classic SPDX 2.x or CycloneDX 1.x without the ML extension names the Python libraries, not the model. An ML-BOM needs ML-typed components (CycloneDX `type: machine-learning-model`) with a `modelCard`.
- **Using SLSA to replace the BOM.** SLSA says how the artifact was built; it does not enumerate the model, dataset, or model-card. The two are complementary, not alternatives.
- **Shipping SPDX 2.x when the exercise asks for SPDX 3.0.** The AI profile is 3.0-only. SPDX 2.x can carry some AI fields through conventions, but the exercise asks for the 3.0 profile explicitly.
- **Fabricated hashes.** Invent a `sha256:a1b2c3…` and the attestation looks authoritative but is wrong. Placeholder `TODO-run-time-hash` strings are better; CI populates them.
- **No subject in SLSA.** `subject[]` is the attested artifact; without it, the provenance is unanchored.
- **Signing block made up locally.** Keys, DSSE envelopes, Rekor entries are all **exercise 05**. Inventing them here muddies both exercises.
- **Mixing up model-card and system-card fidelity.** CycloneDX's `modelCard` corresponds to the per-model card; higher-level system descriptions belong in the SPDX AIPackage's `ai.informationAboutApplication` or in a separate system-card document.
- **Dropping the hosted-vendor base model.** The vendor base model is a runtime dependency; omitting it leaves the ML-BOM materially incomplete even though the weights are not redistributed.
- **Treating ML-BOM authoring as a one-time task.** Every build produces an attestation set; the BOM lives next to the artifact.
- **Ignoring data.** The ML-BOM should name the training / fine-tune dataset and the retrieval corpus; the SPDX 3.0 document should carry a `DatasetPackage`. A BOM that only lists the model component is thin.

## 6. References

- **OWASP CycloneDX.** CycloneDX specification, version 1.6. `https://cyclonedx.org/docs/1.6/json/` — primary source for the ML-BOM shape used in §2.2. <!-- verify-citation: confirm the current 1.6 field names at `cyclonedx.org/docs/1.6/json/` at authoring time. -->
- **Linux Foundation SPDX.** SPDX 3.0 specification. `https://spdx.github.io/spdx-spec/v3.0.0/` — primary source for the AI and Dataset profiles used in §2.3. <!-- verify-citation: confirm the 3.0 JSON-LD shape, profile names, and AIPackage / DatasetPackage field names. -->
- **OpenSSF / SLSA.** SLSA v1.0 Provenance specification. `https://slsa.dev/spec/v1.0/provenance` — primary source for the attestation shape in §2.4. <!-- verify-citation: confirm the v1.0 predicate field names. -->
- **in-toto.** in-toto Statement specification. `https://in-toto.io/Statement/v1` — the envelope used for the SLSA attestation.
- **NIST SP 800-218.** Secure Software Development Framework (SSDF). Referenced for the build-system expectations that SLSA formalises.
- **Federal Food, Drug, and Cosmetic Act §524B.** Cyber-device SBOM obligation referenced from mod-110 ex03 GMLP 2; relevant to CarePilot-Dx's hypothetical submission. <!-- verify-citation: confirm the statutory text when quoted. -->
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — catalogue of attacks the signing + scanning gate in ex05 and ex06 is designed to interdict.
- **MITRE ATLAS.** `https://atlas.mitre.org/` — ML-supply-chain tactics / techniques; informs which dependencies the attestations must cover.
- **Module running system.** CarePilot, `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md`; vendor dependency via mod-110 ex02; PCCP context via mod-110 ex03.
- **Exercise prompt.** `lessons/mod-110-model-risk-management-and-supply-chain/exercises/exercise-04-ml-bom-spdx-slsa-provenance-authoring.md`.
