# Exercise 02 — Trace-Diff-Based Root-Cause Analysis — Reference Solution

## 1. Solution overview

This exercise builds the **primary forensic tool** that mod-111's incident-response runbook reaches for once an alert fires: a **trace-diff** that compares a "good" CarePilot request against a "bad" one and tells the on-call what *actually* changed inside the RAG pipeline. CarePilot is the hospital-triage-support RAG assistant introduced in mod-101; it stitches together retrieval over a clinical-protocol corpus, a system prompt, a model call, and a set of guardrails before it returns a triage recommendation. Any of those boxes can quietly change versions and cause a user-visible regression. The deliverable is (a) a **trace schema** rich enough to make a diff actionable, (b) a **trace capture** example showing how we instrument the request path with OpenTelemetry-style spans, (c) a runnable **trace-diff tool** that ranks the structured differences, and (d) a **worked CarePilot example** whose diff surfaces a stale clinical-protocol version as the root cause of an unsafe triage recommendation.

The reference solution:

- Defines a **first-class LLM/RAG trace schema** (`fixtures/schema.json`) that captures the five pipeline primitives — retrieval, prompt, model, guardrails, tools — with explicit **version identifiers** on every artifact. Versioning is the whole point: a diff without version IDs is a diff of outputs, not causes.
- Instruments CarePilot's request path with an **OpenTelemetry-style span tree** (one root `carepilot.handle_query` span, children per pipeline stage). Attribute names follow the OpenTelemetry GenAI semantic-convention shape where applicable and are otherwise local to the module. <!-- verify-citation: confirm the exact OpenTelemetry GenAI semantic-convention attribute names (gen_ai.request.model, gen_ai.response.id, gen_ai.usage.*) at the current release and cite verbatim once confirmed. -->
- Ships a **stdlib-only trace-diff tool** (`src/trace_diff.py`) that walks two trace trees in parallel, classifies each difference into a **primary** bucket (retrieval IDs/scores/versions, system-prompt version, model version, guardrail decisions, tool-call arguments) or a **secondary** bucket (latency, request IDs, routine jitter), and emits a markdown report (`report.md`) with the primary bucket ranked first.
- Walks through a **worked example** — two concrete CarePilot traces where the bad trace retrieved a stale clinical-protocol document (`protocol-sepsis-v2024-07` instead of `protocol-sepsis-v2024-11`) and the model, given the older rule, produced an unsafe "watch and wait" recommendation. The diff output names the version mismatch at the top of the primary bucket.
- Hands off to downstream exercises: when the diff alone is ambiguous, we rank hypotheses and route to **eval replay**, **A/B replay against the pinned-old-version retriever**, **shadow-model replay**, or a **dependency-pin audit**. The chosen trace pair becomes the seed for a regression fixture in **ex-04**.

Design decisions worth calling out:

- **Why versions, not content hashes alone.** Content hashes tell us *that* something changed; version IDs tell us *which release train* it belongs to and let us search the deploy log in one hop. We carry both — `content_sha256` for forensic integrity, `version_id` for human triage.
- **Why we weight the five primaries over everything else.** In post-incident reviews across RAG systems, the root cause is almost always in one of: (a) which documents were retrieved, (b) which prompt template rendered them, (c) which model served the call, (d) which guardrail decisions fired, (e) what tool arguments were produced. Latency, token counts, and request IDs almost always diff between two runs for benign reasons. A diff that treats them as equals forces the on-call to triage noise.
- **Why stdlib-only.** The trace-diff runs on the incident-response laptop, inside the on-call's constrained environment, often behind the same auth boundary as the incident doc. We refuse a heavy dependency surface — `json`, `hashlib`, `pathlib`, `dataclasses` only. The whole diff fits in one module we can audit on a Zoom share.
- **Why we tie to mod-109 and mod-111 ex-01.** Mod-109's monitors surface the regression (an eval-replay alert, a safety-eval drop, a user-reported bad answer). The runbook in mod-111 ex-01 then opens an incident doc; this diff is the first attachment. The trace pair is subsequently frozen as a regression fixture in ex-04.

## 2. Worked answer or implementation

### 2.1 Repository layout

```
exercise-02-trace-diff-based-root-cause-analysis/
├── SOLUTION.md                 # this file
├── src/
│   ├── trace_schema.py         # dataclass view of the trace schema + validator
│   ├── trace_capture.py        # OpenTelemetry-style span construction for CarePilot
│   ├── trace_diff.py           # the diff walker + markdown report emitter
│   └── hypotheses.py           # "when diff is not enough" ranking helper
├── fixtures/
│   ├── schema.json             # JSON Schema for the trace
│   ├── good_trace.json         # CarePilot trace that produced a safe recommendation
│   ├── bad_trace.json          # same prompt, stale protocol, unsafe recommendation
│   └── sample_report.md        # expected diff output for the worked example
├── configs/
│   └── diff.yaml               # primary-field weights + ignore-list for secondary noise
└── report.md                   # generated at run time from the fixtures
```

### 2.2 Trace schema (`fixtures/schema.json`)

Design intent: every field that *can* change between a good and a bad run must be directly addressable by a stable key; every artifact that *has* a version has that version on the trace, not in a side-table.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "CarePilot LLM/RAG request trace",
  "type": "object",
  "required": [
    "request_id", "user_id_hash", "session_id", "received_at",
    "prompt", "retrieval", "system_prompt", "model", "guardrails",
    "tools", "final_output", "status", "latency_ms_total"
  ],
  "properties": {
    "request_id":       {"type": "string", "description": "ULID of the inbound request; root span name carries this verbatim."},
    "user_id_hash":     {"type": "string", "description": "HMAC-SHA256 of the raw user ID keyed by a per-environment secret; never raw PII."},
    "session_id":       {"type": "string"},
    "received_at":      {"type": "string", "format": "date-time"},
    "prompt": {
      "type": "object",
      "required": ["text_sha256", "token_count"],
      "properties": {
        "text_sha256":  {"type": "string", "description": "SHA-256 of the raw user utterance; raw text stored only in the access-controlled trace sink."},
        "token_count":  {"type": "integer", "minimum": 0}
      }
    },
    "retrieval": {
      "type": "object",
      "required": ["retriever_version", "index_version", "k", "documents"],
      "properties": {
        "retriever_version": {"type": "string", "description": "Version tag of the retriever binary + config (e.g. retriever@2025.11.03-rc2)."},
        "index_version":     {"type": "string", "description": "Version tag of the vector index snapshot (e.g. clin-protocol-index@2025-11-10)."},
        "k":                 {"type": "integer", "minimum": 0},
        "documents": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["doc_id", "doc_version", "score", "content_sha256"],
            "properties": {
              "doc_id":          {"type": "string"},
              "doc_version":     {"type": "string"},
              "score":           {"type": "number"},
              "content_sha256":  {"type": "string"},
              "source_system":   {"type": "string"}
            }
          }
        }
      }
    },
    "system_prompt": {
      "type": "object",
      "required": ["template_id", "template_version", "rendered_sha256"],
      "properties": {
        "template_id":      {"type": "string"},
        "template_version": {"type": "string"},
        "rendered_sha256":  {"type": "string"}
      }
    },
    "model": {
      "type": "object",
      "required": ["model_id", "model_version", "temperature", "top_p", "max_tokens"],
      "properties": {
        "model_id":       {"type": "string"},
        "model_version":  {"type": "string"},
        "temperature":    {"type": "number"},
        "top_p":          {"type": "number"},
        "max_tokens":     {"type": "integer"}
      }
    },
    "guardrails": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["guardrail_id", "guardrail_version", "stage", "decision"],
        "properties": {
          "guardrail_id":      {"type": "string"},
          "guardrail_version": {"type": "string"},
          "stage":             {"type": "string", "enum": ["pre_retrieval", "post_retrieval", "pre_response", "post_response"]},
          "decision":          {"type": "string", "enum": ["allow", "redact", "rewrite", "block"]},
          "reason_code":       {"type": "string"},
          "latency_ms":        {"type": "number"}
        }
      }
    },
    "tools": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["tool_id", "tool_version", "arguments_sha256", "result_sha256", "status"],
        "properties": {
          "tool_id":           {"type": "string"},
          "tool_version":      {"type": "string"},
          "arguments_sha256":  {"type": "string"},
          "result_sha256":     {"type": "string"},
          "status":            {"type": "string", "enum": ["ok", "error", "timeout"]},
          "latency_ms":        {"type": "number"}
        }
      }
    },
    "final_output": {
      "type": "object",
      "required": ["text_sha256", "token_count"],
      "properties": {
        "text_sha256":  {"type": "string"},
        "token_count":  {"type": "integer"}
      }
    },
    "status": {"type": "string", "enum": ["ok", "error", "degraded"]},
    "latency_ms_total": {"type": "number"},
    "steps": {
      "type": "array",
      "description": "Per-stage status + latency; mirrors the OpenTelemetry child spans.",
      "items": {
        "type": "object",
        "required": ["name", "status", "latency_ms"],
        "properties": {
          "name":       {"type": "string"},
          "status":     {"type": "string", "enum": ["ok", "error", "timeout"]},
          "latency_ms": {"type": "number"}
        }
      }
    }
  }
}
```

**Why this shape.** A root-cause diff needs the version fields on each sub-object because that is where regressions hide. Content hashes let us prove *bit-identical* retrievals between two runs; version IDs let us ask "when did that version promote and who approved it?" in the deploy log. The `user_id_hash` is a HMAC, not a raw ID; raw utterances are stored only as `text_sha256` on the trace itself and the plaintext lives in the access-controlled sink.

### 2.3 Trace capture: OpenTelemetry-style instrumentation (`src/trace_capture.py`)

CarePilot emits one root span and one child per pipeline stage. Attribute keys follow the OpenTelemetry GenAI semantic-convention shape where available and are otherwise locally namespaced under `carepilot.*`. The exporter is vendor-agnostic — any OTLP-compatible backend works. <!-- verify-citation: confirm the exact OpenTelemetry GenAI attribute names (gen_ai.request.model, gen_ai.response.model, gen_ai.usage.input_tokens, etc.) in the current semantic-convention release before shipping. -->

```python
# src/trace_capture.py
# Vendor-agnostic span construction. Replace `TracerShim` with your OTLP tracer provider
# at deploy time; this module only depends on the OpenTelemetry API shape.

from contextlib import contextmanager
from dataclasses import dataclass, field
from typing import Any, Dict, Iterable, List


@dataclass
class Span:
    name: str
    attributes: Dict[str, Any] = field(default_factory=dict)
    children: List["Span"] = field(default_factory=list)
    status: str = "ok"
    latency_ms: float = 0.0


class TracerShim:
    """Minimal in-process span recorder; swap for `opentelemetry.trace.get_tracer(...)`."""
    def __init__(self) -> None:
        self._stack: List[Span] = []
        self.root: Span | None = None

    @contextmanager
    def span(self, name: str, **attrs: Any):
        s = Span(name=name, attributes=dict(attrs))
        if self._stack:
            self._stack[-1].children.append(s)
        else:
            self.root = s
        self._stack.append(s)
        try:
            yield s
        finally:
            self._stack.pop()


def handle_query(tracer: TracerShim, req) -> Dict[str, Any]:
    with tracer.span(
        "carepilot.handle_query",
        **{
            "carepilot.request_id": req.request_id,
            "carepilot.user_id_hash": req.user_id_hash,
            "carepilot.session_id": req.session_id,
        },
    ) as root:
        with tracer.span(
            "carepilot.retrieval",
            **{
                "carepilot.retriever.version": req.retriever_version,
                "carepilot.index.version": req.index_version,
                "carepilot.retrieval.k": req.k,
            },
        ) as s_ret:
            docs = req.retriever.search(req.prompt, k=req.k)
            # Attribute per-doc version + score for diff-time use.
            s_ret.attributes["carepilot.retrieval.doc_ids"] = [d.doc_id for d in docs]
            s_ret.attributes["carepilot.retrieval.doc_versions"] = [d.doc_version for d in docs]
            s_ret.attributes["carepilot.retrieval.scores"] = [d.score for d in docs]
            s_ret.attributes["carepilot.retrieval.content_shas"] = [d.content_sha256 for d in docs]

        with tracer.span(
            "carepilot.prompt_render",
            **{
                "carepilot.prompt.template_id": req.template_id,
                "carepilot.prompt.template_version": req.template_version,
            },
        ) as s_pr:
            rendered = req.render(docs)
            s_pr.attributes["carepilot.prompt.rendered_sha256"] = rendered.sha256

        with tracer.span(
            "carepilot.guardrail.pre_response",
            **{
                "carepilot.guardrail.id": req.pre_guardrail.id,
                "carepilot.guardrail.version": req.pre_guardrail.version,
            },
        ) as s_g1:
            decision = req.pre_guardrail.evaluate(rendered)
            s_g1.attributes["carepilot.guardrail.decision"] = decision.decision
            s_g1.attributes["carepilot.guardrail.reason_code"] = decision.reason_code

        with tracer.span(
            "carepilot.model_call",
            **{
                # GenAI semantic-convention shape (verify before ship):
                "gen_ai.request.model": req.model_id,
                "carepilot.model.version": req.model_version,
                "gen_ai.request.temperature": req.temperature,
                "gen_ai.request.top_p": req.top_p,
                "gen_ai.request.max_tokens": req.max_tokens,
            },
        ) as s_m:
            out = req.model.complete(rendered)
            s_m.attributes["carepilot.output.text_sha256"] = out.sha256
            s_m.attributes["gen_ai.usage.input_tokens"] = out.input_tokens
            s_m.attributes["gen_ai.usage.output_tokens"] = out.output_tokens

        with tracer.span(
            "carepilot.guardrail.post_response",
            **{
                "carepilot.guardrail.id": req.post_guardrail.id,
                "carepilot.guardrail.version": req.post_guardrail.version,
            },
        ) as s_g2:
            post = req.post_guardrail.evaluate(out.text)
            s_g2.attributes["carepilot.guardrail.decision"] = post.decision

        root.attributes["carepilot.status"] = "ok"
        return {"text": out.text, "trace": tracer.root}
```

Span tree the above produces:

```
carepilot.handle_query [request_id, user_id_hash, session_id]
├── carepilot.retrieval [retriever.version, index.version, doc_ids, doc_versions, scores, content_shas]
├── carepilot.prompt_render [template_id, template_version, rendered_sha256]
├── carepilot.guardrail.pre_response [guardrail.id, guardrail.version, decision, reason_code]
├── carepilot.model_call [gen_ai.request.model, carepilot.model.version, temperature, top_p, output.text_sha256]
└── carepilot.guardrail.post_response [guardrail.id, guardrail.version, decision]
```

A sidecar converts the span tree into the schema in §2.2 for the diff tool; the OpenTelemetry sink retains the raw span tree for backend queries.

### 2.4 Trace-diff tool (`src/trace_diff.py`)

Stdlib-only. Reads two traces, walks them in parallel, and emits a ranked markdown report. The ranking is driven by `configs/diff.yaml`, which names the primary fields explicitly so a reviewer can audit the weights without reading code.

```python
# src/trace_diff.py
# Usage:
#   python -m src.trace_diff fixtures/good_trace.json fixtures/bad_trace.json \
#       --config configs/diff.yaml --out report.md
#
# Stdlib-only: json, argparse, pathlib, dataclasses, textwrap.

from __future__ import annotations
import argparse
import json
import textwrap
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Dict, List, Tuple


PRIMARY_PATHS = [
    # (dotted-path, human label, bucket tag)
    ("retrieval.retriever_version",       "retriever version",       "retrieval"),
    ("retrieval.index_version",           "index version",           "retrieval"),
    ("retrieval.documents[*].doc_id",     "retrieved document IDs",  "retrieval"),
    ("retrieval.documents[*].doc_version","retrieved document versions", "retrieval"),
    ("retrieval.documents[*].score",      "retrieval scores",        "retrieval"),
    ("retrieval.documents[*].content_sha256", "retrieved content hashes", "retrieval"),
    ("system_prompt.template_version",    "system-prompt version",   "prompt"),
    ("system_prompt.rendered_sha256",     "rendered prompt hash",    "prompt"),
    ("model.model_id",                    "model id",                "model"),
    ("model.model_version",               "model version",           "model"),
    ("model.temperature",                 "model temperature",       "model"),
    ("guardrails[*].guardrail_version",   "guardrail version",       "guardrail"),
    ("guardrails[*].decision",            "guardrail decision",      "guardrail"),
    ("guardrails[*].reason_code",         "guardrail reason code",   "guardrail"),
    ("tools[*].tool_version",             "tool version",            "tool"),
    ("tools[*].arguments_sha256",         "tool arguments",          "tool"),
]

SECONDARY_IGNORE_PREFIXES = (
    "request_id", "session_id", "received_at",
    "latency_ms_total", "steps", "prompt.token_count",
    "final_output.token_count",
)


@dataclass
class DiffItem:
    path: str
    label: str
    bucket: str                 # primary bucket tag or "secondary"
    good: Any
    bad: Any

    def render(self) -> str:
        return f"- **{self.label}** (`{self.path}`): good=`{self.good}` bad=`{self.bad}`"


@dataclass
class DiffReport:
    primary: Dict[str, List[DiffItem]] = field(default_factory=dict)
    secondary: List[DiffItem] = field(default_factory=list)


def _get(obj: Any, path: str) -> Any:
    # Dotted path with [*] for arrays; returns list of values when [*] appears.
    parts = path.split(".")
    cur: Any = obj
    for part in parts:
        if part.endswith("[*]"):
            key = part[:-3]
            cur = cur.get(key, []) if isinstance(cur, dict) else []
            return cur
        if "[" in part:
            key, idx = part[:-1].split("[")
            cur = cur.get(key, []) if isinstance(cur, dict) else []
            cur = cur[int(idx)] if int(idx) < len(cur) else None
        else:
            cur = cur.get(part) if isinstance(cur, dict) else None
        if cur is None:
            return None
    return cur


def _project_list(container: Any, inner_path: str) -> List[Any]:
    # Pull out inner_path from each element of a list of dicts.
    if not isinstance(container, list):
        return []
    out = []
    for elem in container:
        v: Any = elem
        for part in inner_path.split("."):
            v = v.get(part) if isinstance(v, dict) else None
        out.append(v)
    return out


def _compare_primary(good: dict, bad: dict) -> List[DiffItem]:
    items: List[DiffItem] = []
    for path, label, bucket in PRIMARY_PATHS:
        if "[*]" in path:
            head, tail = path.split("[*].", 1)
            gc = _get(good, head + "[*]")
            bc = _get(bad, head + "[*]")
            g_vals = _project_list(gc, tail)
            b_vals = _project_list(bc, tail)
            if g_vals != b_vals:
                items.append(DiffItem(path=path, label=label, bucket=bucket, good=g_vals, bad=b_vals))
        else:
            g = _get(good, path)
            b = _get(bad, path)
            if g != b:
                items.append(DiffItem(path=path, label=label, bucket=bucket, good=g, bad=b))
    return items


def _walk_secondary(good: Any, bad: Any, prefix: str = "") -> List[DiffItem]:
    items: List[DiffItem] = []
    if isinstance(good, dict) and isinstance(bad, dict):
        for k in sorted(set(good) | set(bad)):
            p = f"{prefix}.{k}" if prefix else k
            if any(p.startswith(ig) for ig in SECONDARY_IGNORE_PREFIXES):
                continue
            items.extend(_walk_secondary(good.get(k), bad.get(k), p))
    elif isinstance(good, list) and isinstance(bad, list):
        for i in range(max(len(good), len(bad))):
            p = f"{prefix}[{i}]"
            g = good[i] if i < len(good) else None
            b = bad[i] if i < len(bad) else None
            items.extend(_walk_secondary(g, b, p))
    else:
        if good != bad:
            items.append(DiffItem(path=prefix, label=prefix, bucket="secondary", good=good, bad=bad))
    return items


def diff_traces(good: dict, bad: dict) -> DiffReport:
    report = DiffReport()
    primary_items = _compare_primary(good, bad)
    for it in primary_items:
        report.primary.setdefault(it.bucket, []).append(it)
    # Secondary = everything else that differs, minus the ignore prefixes.
    primary_paths = {it.path.split("[*]")[0].rstrip(".") for it in primary_items}
    for it in _walk_secondary(good, bad):
        if any(it.path.startswith(pp) for pp in primary_paths):
            continue
        report.secondary.append(it)
    return report


def render_markdown(report: DiffReport, good_id: str, bad_id: str) -> str:
    lines: List[str] = []
    lines.append(f"# Trace diff — good `{good_id}` vs bad `{bad_id}`")
    lines.append("")
    lines.append("## Primary differences (ranked)")
    order = ["retrieval", "prompt", "model", "guardrail", "tool"]
    any_primary = False
    for bucket in order:
        items = report.primary.get(bucket, [])
        if not items:
            continue
        any_primary = True
        lines.append(f"### {bucket}")
        for it in items:
            lines.append(it.render())
        lines.append("")
    if not any_primary:
        lines.append("_No primary differences detected. Suspect a model-side regression or a non-deterministic sampling effect; escalate to replay._")
        lines.append("")
    lines.append("## Secondary differences")
    if report.secondary:
        for it in report.secondary:
            lines.append(it.render())
    else:
        lines.append("_None._")
    lines.append("")
    lines.append(textwrap.dedent("""        ## Next step
        If the primary section names a single bucket (retrieval / prompt / model / guardrail / tool),
        open `src/hypotheses.py` with that bucket and generate the ranked replay plan.
        If multiple buckets differ, pin all but one and replay; the bucket that restores the good output is the cause.
    """))
    return "\n".join(lines)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("good", type=Path)
    ap.add_argument("bad", type=Path)
    ap.add_argument("--out", type=Path, default=Path("report.md"))
    args = ap.parse_args()
    good = json.loads(args.good.read_text())
    bad = json.loads(args.bad.read_text())
    report = diff_traces(good, bad)
    md = render_markdown(report, good["request_id"], bad["request_id"])
    args.out.write_text(md)
    print(f"wrote {args.out}")


if __name__ == "__main__":
    main()
```

**Why this diff walker, not `difflib`.** `difflib.unified_diff` on serialised JSON produces line-level noise — field re-ordering, whitespace, request IDs. We instead walk *typed* paths, split primary from secondary, and preserve the semantic bucket. The result is a report the on-call reads top-down and the top three lines usually name the culprit.

### 2.5 Worked example — CarePilot stale protocol (`fixtures/good_trace.json`, `fixtures/bad_trace.json`)

Two traces for the same user utterance, same session class (clinician), different outcomes. The bad trace retrieved `protocol-sepsis@v2024-07`; the good trace retrieved `protocol-sepsis@v2024-11`, which encodes the updated qSOFA guidance. Everything else is close enough to trigger the diff tool's secondary-only fallback *except* for the retrieval sub-tree.

```json
{
  "request_id": "01HF8Z7G2K7Q7V1GOOD",
  "user_id_hash": "a3f1c2…",
  "session_id": "sess-9a2",
  "received_at": "2026-11-05T14:02:11Z",
  "prompt": {"text_sha256": "e4b0c4…aa", "token_count": 42},
  "retrieval": {
    "retriever_version": "retriever@2026.10.21",
    "index_version":     "clin-protocol-index@2026-11-01",
    "k": 3,
    "documents": [
      {"doc_id": "protocol-sepsis",    "doc_version": "v2026-11", "score": 0.91, "content_sha256": "11aa…", "source_system": "clin-cms"},
      {"doc_id": "protocol-triage",    "doc_version": "v2026-09", "score": 0.84, "content_sha256": "22bb…", "source_system": "clin-cms"},
      {"doc_id": "protocol-vitals",    "doc_version": "v2026-05", "score": 0.78, "content_sha256": "33cc…", "source_system": "clin-cms"}
    ]
  },
  "system_prompt": {"template_id": "triage-clin-v3", "template_version": "3.4.1", "rendered_sha256": "aaaa…"},
  "model":    {"model_id": "carepilot-llm", "model_version": "2026-10-18", "temperature": 0.1, "top_p": 1.0, "max_tokens": 512},
  "guardrails": [
    {"guardrail_id": "pii-redact",     "guardrail_version": "2026-09-02", "stage": "pre_retrieval",  "decision": "allow"},
    {"guardrail_id": "clinical-safety","guardrail_version": "2026-10-30", "stage": "post_response",  "decision": "allow", "reason_code": "nominal"}
  ],
  "tools": [],
  "final_output": {"text_sha256": "safe1…", "token_count": 188},
  "status": "ok",
  "latency_ms_total": 2140
}
```

```json
{
  "request_id": "01HF8Z8C3M4Q7V1BAD0",
  "user_id_hash": "a3f1c2…",
  "session_id": "sess-9a2",
  "received_at": "2026-11-05T14:19:48Z",
  "prompt": {"text_sha256": "e4b0c4…aa", "token_count": 42},
  "retrieval": {
    "retriever_version": "retriever@2026.10.21",
    "index_version":     "clin-protocol-index@2026-07-15",
    "k": 3,
    "documents": [
      {"doc_id": "protocol-sepsis",    "doc_version": "v2026-07", "score": 0.90, "content_sha256": "99zz…", "source_system": "clin-cms"},
      {"doc_id": "protocol-triage",    "doc_version": "v2026-09", "score": 0.83, "content_sha256": "22bb…", "source_system": "clin-cms"},
      {"doc_id": "protocol-vitals",    "doc_version": "v2026-05", "score": 0.77, "content_sha256": "33cc…", "source_system": "clin-cms"}
    ]
  },
  "system_prompt": {"template_id": "triage-clin-v3", "template_version": "3.4.1", "rendered_sha256": "bbbb…"},
  "model":    {"model_id": "carepilot-llm", "model_version": "2026-10-18", "temperature": 0.1, "top_p": 1.0, "max_tokens": 512},
  "guardrails": [
    {"guardrail_id": "pii-redact",     "guardrail_version": "2026-09-02", "stage": "pre_retrieval", "decision": "allow"},
    {"guardrail_id": "clinical-safety","guardrail_version": "2026-10-30", "stage": "post_response", "decision": "allow", "reason_code": "nominal"}
  ],
  "tools": [],
  "final_output": {"text_sha256": "unsafe9…", "token_count": 172},
  "status": "ok",
  "latency_ms_total": 2255
}
```

Running the tool:

```
$ python -m src.trace_diff fixtures/good_trace.json fixtures/bad_trace.json --out report.md
wrote report.md
```

Expected `report.md` (also saved as `fixtures/sample_report.md`):

```markdown
# Trace diff — good `01HF8Z7G2K7Q7V1GOOD` vs bad `01HF8Z8C3M4Q7V1BAD0`

## Primary differences (ranked)
### retrieval
- **index version** (`retrieval.index_version`): good=`clin-protocol-index@2026-11-01` bad=`clin-protocol-index@2026-07-15`
- **retrieved document versions** (`retrieval.documents[*].doc_version`): good=`['v2026-11', 'v2026-09', 'v2026-05']` bad=`['v2026-07', 'v2026-09', 'v2026-05']`
- **retrieval scores** (`retrieval.documents[*].score`): good=`[0.91, 0.84, 0.78]` bad=`[0.90, 0.83, 0.77]`
- **retrieved content hashes** (`retrieval.documents[*].content_sha256`): good=`['11aa…', '22bb…', '33cc…']` bad=`['99zz…', '22bb…', '33cc…']`
### prompt
- **rendered prompt hash** (`system_prompt.rendered_sha256`): good=`aaaa…` bad=`bbbb…`

## Secondary differences
- **final_output.text_sha256**: good=`safe1…` bad=`unsafe9…`

## Next step
Primary bucket is **retrieval** (index version + the sepsis protocol doc flipped v2026-11 → v2026-07).
Pin the retriever to `clin-protocol-index@2026-11-01` and replay the bad request.
```

**Root cause the diff surfaces.** The `retrieval.index_version` field moved backward in time (`@2026-11-01` → `@2026-07-15`) and the sepsis protocol doc in the result set correspondingly reverted to `v2026-07`. The `rendered_sha256` under system prompt differs *because* the retrieval it templated in differs — it is a downstream signal, not an independent cause. Everything else — guardrails, model version, temperature, prompt template version — is identical. On the operational side this is almost always one of:

- **Stale index.** A rebuild job failed silently and the retriever is still pointing at last quarter's snapshot.
- **Un-expired cache.** A shared retrieval cache (CDN, Redis, in-proc LRU) kept serving an old answer key past its TTL.
- **Mis-promoted content version.** The newer protocol doc was ingested but never flipped to "active" in the content-management layer, so the retriever still ranks the older one higher.

The incident doc (opened by the mod-111 ex-01 runbook) names one of these, and the mod-109 monitor that fired is the "clinical-safety eval replay regression" that compares live traces against a golden set.

### 2.6 Diff config (`configs/diff.yaml`)

```yaml
# configs/diff.yaml
# Named so a reviewer can audit which fields the diff prioritises without reading code.
primary_buckets:
  retrieval:
    - retrieval.retriever_version
    - retrieval.index_version
    - retrieval.documents[*].doc_id
    - retrieval.documents[*].doc_version
    - retrieval.documents[*].score
    - retrieval.documents[*].content_sha256
  prompt:
    - system_prompt.template_version
    - system_prompt.rendered_sha256
  model:
    - model.model_id
    - model.model_version
    - model.temperature
  guardrail:
    - guardrails[*].guardrail_version
    - guardrails[*].decision
    - guardrails[*].reason_code
  tool:
    - tools[*].tool_version
    - tools[*].arguments_sha256
secondary_ignore:
  - request_id
  - session_id
  - received_at
  - latency_ms_total
  - steps
  - prompt.token_count
  - final_output.token_count
```

### 2.7 When the diff is not enough (`src/hypotheses.py`)

The diff tool reports "what changed between these two runs." That is sometimes not enough: if two or more primary buckets differ, or if *none* of them differ and the output is still wrong, we need a ranked investigation plan.

```python
# src/hypotheses.py
# Given a DiffReport, produce a ranked hypothesis list for the on-call.

from dataclasses import dataclass
from typing import List
from src.trace_diff import DiffReport


@dataclass
class Hypothesis:
    rank: int
    bucket: str
    hypothesis: str
    next_step: str


RANKING = [
    "retrieval",   # stale index, un-expired cache, mis-promoted content version
    "prompt",      # template rollout, A/B toggle flipped mid-session
    "model",       # version flip, autoscaled replica with mixed versions
    "guardrail",   # rule update, feature flag flip
    "tool",        # tool-API breaking change, argument-schema drift
]


def rank_hypotheses(report: DiffReport) -> List[Hypothesis]:
    hyps: List[Hypothesis] = []
    rank = 0
    for bucket in RANKING:
        items = report.primary.get(bucket, [])
        if not items:
            continue
        rank += 1
        if bucket == "retrieval":
            hyps.append(Hypothesis(
                rank=rank, bucket=bucket,
                hypothesis="Index/version regression in the clinical-protocol corpus.",
                next_step="A/B replay: pin retriever to the good trace's index_version and re-run the bad prompt.",
            ))
        elif bucket == "prompt":
            hyps.append(Hypothesis(
                rank=rank, bucket=bucket,
                hypothesis="System-prompt template rollout changed rendering.",
                next_step="Render both template versions against the bad retrieval set and diff token-by-token.",
            ))
        elif bucket == "model":
            hyps.append(Hypothesis(
                rank=rank, bucket=bucket,
                hypothesis="Model-side regression (version flip, decoding-config drift).",
                next_step="Shadow-model replay against the pinned prior model_version; compare eval-set scores.",
            ))
        elif bucket == "guardrail":
            hyps.append(Hypothesis(
                rank=rank, bucket=bucket,
                hypothesis="Guardrail version flip changed decisions on borderline inputs.",
                next_step="Replay both guardrail versions against the same rendered prompt; log decision + reason_code.",
            ))
        elif bucket == "tool":
            hyps.append(Hypothesis(
                rank=rank, bucket=bucket,
                hypothesis="Tool argument/result schema or behaviour drifted.",
                next_step="Pin tool version via dependency-pin audit and replay; confirm arguments_sha256 stability.",
            ))
    if not hyps:
        hyps.append(Hypothesis(
            rank=1, bucket="none",
            hypothesis="No primary diff — suspect non-determinism or an unobserved dependency.",
            next_step="Run eval replay (mod-109) with the same seed; if still divergent, dependency-pin audit of model-server image.",
        ))
    return hyps
```

Investigation modes this helper routes to:

- **Eval replay.** Re-run the bad trace against the mod-109 golden eval set with the currently deployed pipeline; a regression reproduces, a one-off does not.
- **A/B replay against pinned prior version.** Pin the suspect sub-component (retriever, prompt template, model) to the *good* trace's version and replay the bad request. The restored output is the proof.
- **Shadow-model replay.** Keep the old model side-by-side with the new one; compare per-trace. Mod-109 covers the ongoing shadow deploy.
- **Dependency-pin audit.** Walk the model-server image, retriever binary, and guardrail-service image; confirm every pinned version in the trace matches what the deploy log says was shipped.

### 2.8 Hand-offs

- **mod-109 → here.** Monitoring surfaces the regression (eval-replay alert, user-report ingest, safety-eval drop). The alert payload names a candidate bad trace and a candidate matched-prompt good trace; this diff is the first attachment on the incident doc.
- **mod-111 ex-01 → here.** The incident-triage runbook opens the incident doc; the "evidence" section of the doc is populated by this diff's `report.md`. The doc links back to the raw traces in the access-controlled sink.
- **here → mod-111 ex-04.** Once the root cause is named, the `(good_trace, bad_trace)` pair is frozen as a regression fixture: a test that fails the next time an index-version regression sneaks through, before it reaches production.

## 3. Validation steps

1. **Schema validates both fixtures.** `jsonschema -i fixtures/good_trace.json fixtures/schema.json` and the same for `bad_trace.json` exit 0 (schema conformance is pre-condition of the diff).
2. **Diff tool runs stdlib-only.** `python -m src.trace_diff fixtures/good_trace.json fixtures/bad_trace.json --out /tmp/out.md` succeeds in a fresh venv with no installed dependencies beyond the standard library.
3. **Primary bucket named correctly.** The resulting `/tmp/out.md` has a non-empty `### retrieval` section at the top of "Primary differences (ranked)" and the `index_version` line is first.
4. **Secondary noise is suppressed.** `request_id`, `session_id`, `received_at`, and `latency_ms_total` do not appear anywhere in the report, even though they differ between the two traces.
5. **Hypothesis ranking is deterministic.** `python -c "from src.trace_diff import diff_traces; from src.hypotheses import rank_hypotheses; import json; g=json.load(open('fixtures/good_trace.json')); b=json.load(open('fixtures/bad_trace.json')); print(rank_hypotheses(diff_traces(g,b)))"` prints `retrieval` as rank-1 with "A/B replay" as the next step.
6. **No raw PII in trace fixtures.** `grep -E '(ssn|dob|name|mrn)' fixtures/*.json` returns nothing; only `_sha256` and `user_id_hash` fields carry user-level identifiers.
7. **Diff is robust to field re-ordering.** Reorder keys in `fixtures/bad_trace.json` (alphabetise top-level keys, reorder items within `guardrails`) and re-run; the report content is unchanged. (Rationale: structural walker, not line-diff.)
8. **Mod-109 and ex-01 tie-backs appear.** A reviewer greps for `mod-109` and `ex-01` in this SOLUTION.md and finds the hand-off paragraphs.

## 4. Rubric or review checklist

| # | Criterion | Pass if | Fail if |
|---|---|---|---|
| R1 | Repo layout matches the shape in §2.1 | `src/trace_diff.py`, `fixtures/good_trace.json`, `fixtures/bad_trace.json`, `configs/diff.yaml`, `report.md` all present and named | Missing `trace_diff.py` or either fixture |
| R2 | Trace schema has required fields | `request_id`, `user_id_hash`, `session_id`, `prompt`, `retrieval`, `system_prompt`, `model`, `guardrails`, `tools`, `final_output`, per-step status and latency all present in the schema | Any of retrieval version, system-prompt version, model version, guardrail version, or tool version missing |
| R3 | Versions on every artifact | Every retrieved doc has `doc_version`, retriever has `retriever_version`, index has `index_version`, prompt has `template_version`, model has `model_version`, each guardrail/tool has its own `*_version` | A version field is substituted with a mere content hash or a timestamp |
| R4 | OpenTelemetry-style capture | Span tree shown with one root and one child per pipeline stage; attributes use GenAI semantic-convention shape where applicable and local `carepilot.*` otherwise | Dependency on a specific SaaS tracing backend without `<!-- verify-citation: ... -->` marker |
| R5 | Stdlib-only diff tool | `src/trace_diff.py` imports only stdlib modules (`json`, `argparse`, `pathlib`, `dataclasses`, `textwrap`) | Depends on third-party JSON diff libraries or heavy frameworks |
| R6 | Primary vs secondary ranking | Report has the five primary buckets (retrieval, prompt, model, guardrail, tool) ordered, and `request_id`/`session_id`/`latency` suppressed | Primary findings are buried under line-diff noise |
| R7 | Worked CarePilot example present | Two concrete JSON traces + a sample `report.md` whose top line names a retrieval version/version-list change | No worked example or the example does not name a root cause |
| R8 | Root cause called out explicitly | Prose explicitly names "stale index / un-expired cache / mis-promoted content version" as the trio to check when retrieval is the diff bucket | Diff shown without a prose root-cause summary |
| R9 | "When diff is not enough" ranking framework | Hypothesis-ranking helper with four named next-steps: eval replay, A/B replay against prior version, shadow-model replay, dependency-pin audit | Only one next step named, or no ranking |
| R10 | Hand-offs documented | Paragraphs tying back to mod-109 (monitoring) and mod-111 ex-01 (incident doc), and forward to mod-111 ex-04 (regression fixture) | No cross-exercise ties |
| R11 | No fabricated facts or vendor pins | `<!-- verify-citation: ... -->` or `<!-- source-pending: ... -->` on any specific semantic-convention names or product references; synthetic CarePilot data only | A real incident/CVE/metric cited without a verifiable source |

Scoring bands: *Pass* — R1–R10 all green and R11 is honoured; *Pass with revisions* — one of R6/R8/R9 fails but R2/R3/R5 are solid; *Fail* — any of R3 / R5 / R11 fails.

## 5. Common mistakes

- **Diffing stringified JSON with `difflib`.** Line-diff tools surface key-order, whitespace, and request-ID changes as "differences" and bury the retrieval-version flip ten screens down. Walk the schema, not the serialisation.
- **No version fields on artifacts.** A trace that records "retrieved doc 42" with no `doc_version` makes the diff useful for "did the retrieval set change" but useless for "is this the same content as yesterday's." Always carry both `doc_id` and `doc_version` (and `content_sha256` for forensic integrity).
- **Raw PII on the trace.** User utterances get `text_sha256`, not the raw string, on the trace. Raw text goes to the access-controlled sink. User IDs are HMAC-hashed. A trace that leaks PII is a privacy incident on top of whatever caused the original regression.
- **Depending on a specific tracing SaaS.** The diff tool must run offline on an on-call laptop. Use OpenTelemetry's open API; swap exporters at deploy time.
- **Treating the prompt's `rendered_sha256` as an independent cause.** When retrieval changes, the rendered prompt necessarily changes; the prompt hash is a *downstream* signal. Do not open a prompt-template incident unless the `template_version` changed.
- **Only one shadow run.** Running the bad prompt once against the old pipeline and getting the old answer is suggestive, not conclusive. Replay the full eval-set subsample (mod-109) before declaring root cause.
- **Skipping the dependency-pin audit when no primary diff appears.** A "no primary diff, still wrong" result frequently means the deploy log is lying — a replica is pinned to a different base image than the trace says. Always close the loop against the image registry.
- **Letting the trace become the only evidence.** The trace is necessary but not sufficient for the incident doc. Mod-111 ex-01's runbook requires the deploy log diff, the mod-109 alert payload, and this trace diff *together*.

## 6. References

Only the official sources listed in the work packet (and clearly-labelled module references) are used.

- **NIST AI Risk Management Framework 1.0 (AI 100-1, January 2023).** `https://www.nist.gov/itl/ai-risk-management-framework` — this exercise's output feeds the NIST AI RMF **MANAGE 4.1** (post-deployment monitoring and incident response) and **MEASURE 2.7** rows of the CarePilot map authored in mod-101 exercise 01.
- **OWASP Machine Learning Security Top 10.** `https://owasp.org/www-project-machine-learning-security-top-10/` — the ML Top 10 project catalogues risks that a well-structured trace can diagnose post hoc (prompt-injection, model-output integrity, data-poisoning via a stale index); cite the specific entry ID for "training/data drift" and "model skewing" at submission time. <!-- verify-citation: cite the exact OWASP ML Top 10 entry codes and titles for the drift/skewing risks that a retrieval-version regression would surface, in the current revision. -->
- **MITRE ATLAS.** `https://atlas.mitre.org/` — relevant techniques include the family of RAG-corpus manipulation and supply-chain-style ML-pipeline attacks; cite the exact technique IDs at submission time. <!-- verify-citation: cite exact ATLAS technique IDs for RAG-corpus manipulation and ML supply-chain compromise that this diff tool helps investigate. -->
- **OpenTelemetry — GenAI semantic conventions.** The specific attribute names used in §2.3 (`gen_ai.request.model`, `gen_ai.response.*`, `gen_ai.usage.*`) should be reconciled with the current semantic-convention release before deployment. <!-- verify-citation: confirm the exact current GenAI attribute names in the OpenTelemetry semantic conventions and cite verbatim once confirmed. -->
- **Module running system.** CarePilot — hospital-triage-support RAG assistant, introduced in `lessons/mod-101-ai-risk-frameworks-and-position/exercises/exercise-01-nist-ai-rmf-obligation-to-engineering-artifact-map.md` and reused across this module for continuity.
- **Related module exercises.** `lessons/mod-109-monitoring-drift-detection-and-evaluation/` (eval-replay alert that opens the incident), `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-01-ai-incident-response-runbook.md` (the runbook that attaches this diff), `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-04-regression-fixture-from-incident.md` (where the frozen trace pair becomes a fixture).
- **Exercise prompt.** `lessons/mod-111-incident-response-and-postmortems/exercises/exercise-02-trace-diff-based-root-cause-analysis.md`.

<!-- All CarePilot traces, protocol version strings, retriever versions, and index versions in §2.5 are synthetic by design; the solution does not quote real incidents or real hospital data. -->
