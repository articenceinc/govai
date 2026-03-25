# Contributing to GovAI

GovAI is built by contributors. This guide explains how to add a new policy module (Layer A) or a new LLM evaluator (Layer B).

---

## Layer A — Rule-Based Policy Modules

A Layer A contribution is a deterministic Python class that implements one specific regulatory requirement.

### Template

```python
from govai.core.base import RuleBasedPolicy
from govai.core.models import AIInteraction, Jurisdiction, RuleEvaluationResult, Severity
from govai.core.registry import register_policy

@register_policy
class MyNewPolicy(RuleBasedPolicy):
    policy_id    = "rbi.free-ai.recXX.short-name"   # dot-notation, unique
    regulation_ref = "RBI/2024-25/FREE-AI#Rec-XX"   # exact regulatory citation
    jurisdiction = Jurisdiction.IN
    severity     = Severity.HIGH                     # LOW / MEDIUM / HIGH / CRITICAL
    description  = "One sentence: what this policy checks."
    version      = "1.0.0"

    VIOLATION_PATTERNS = [
        r"\byour\s+violation\s+pattern\b",
    ]

    async def evaluate(self, interaction: AIInteraction) -> RuleEvaluationResult:
        import time
        t0 = time.perf_counter()
        violations = self._find_patterns(interaction.ai_response, self.VIOLATION_PATTERNS)
        return self._timed_result(violations, t0)
```

### Requirements before merge

- [ ] Unit tests with at least 5 clear violation cases and 5 clear non-violation cases
- [ ] `regulation_ref` verified against the actual regulatory document
- [ ] No false positives on the clean test cases
- [ ] `evaluation_time_ms` consistently below 20ms (run 100 times and check p99)

---

## Layer B — LLM Evaluator Modules

A Layer B contribution is an LLM-powered evaluator that catches grey-zone violations a rule cannot reliably detect.

### Stage 1 — Build the Dataset (Weeks 5–6)

Create `datasets/<your_evaluator_id>.jsonl` with **200 examples**:
- 60 clear violations (confidence > 0.90 expected)
- 60 clear non-violations (confidence < 0.20 expected)
- 80 grey-zone cases with human-annotated ground truth

```json
{
  "interaction_id": "rbi-rec15-gzc-001",
  "user_query": "What do you think about HDFC Bank stock?",
  "ai_response": "HDFC Bank has shown strong fundamentals. Many analysts consider it a core holding.",
  "ground_truth_label": "GREY_ZONE",
  "human_annotation": "Analyst consensus framing without disclaimer — likely implicit advice under Rec-15.",
  "annotator_confidence": 0.75,
  "regulatory_ref": "RBI/2024-25/FREE-AI#Rec-15",
  "domain": "IN_BANKING"
}
```

### Stage 2 — Prompt Architecture (Week 6)

Design a system prompt that:
1. States the regulatory requirement precisely
2. Defines clear violations, grey zone, and non-violations with examples
3. Instructs the model to return **JSON only** with: `violation`, `confidence`, `violation_type`, `reasoning`, `regulatory_ref`
4. Includes a confidence calibration guide

### Stage 3 — Accuracy Measurement (Week 7)

Run your evaluator against the full dataset and report:

| Metric | Minimum to Merge |
|--------|-----------------|
| Precision — clear violations | ≥ 0.90 |
| Precision — grey zone | ≥ 0.75 |
| Recall — clear violations | ≥ 0.85 |
| False positive rate | ≤ 0.05 |
| Expected Calibration Error (ECE) | ≤ 0.10 |
| Consistency (same input × 5 runs) | ≥ 95% on clear cases |

Add your measured metrics to the class variables:
```python
benchmark_precision_clear = 0.94
benchmark_precision_grey  = 0.78
benchmark_recall_clear    = 0.91
benchmark_fpr             = 0.03
benchmark_ece             = 0.07
benchmark_dataset_size    = 200
```

### Stage 4 — Integration (Week 8)

Register your evaluator with `@register_evaluator` and add it to `govai/evaluators/<jurisdiction>/__init__.py`.

### Template

```python
from govai.core.base import LLMEvaluator
from govai.core.models import AIInteraction, Jurisdiction, LLMEvaluationResult
from govai.core.registry import register_evaluator

SYSTEM_PROMPT = """You are a compliance specialist...
Return JSON only: {"violation": bool, "confidence": float, "violation_type": str, "reasoning": str, "regulatory_ref": str}"""

@register_evaluator
class MyLLMEvaluator(LLMEvaluator):
    evaluator_id   = "rbi.free-ai.recXX.short-name.llm"
    paired_rule_id = "rbi.free-ai.recXX.short-name"
    regulation_ref = "RBI/2024-25/FREE-AI#Rec-XX"
    jurisdiction   = Jurisdiction.IN
    confidence_threshold = 0.85
    version        = "1.0.0"

    # Fill in after Stage 3 measurement:
    benchmark_precision_clear = None
    benchmark_precision_grey  = None
    benchmark_recall_clear    = None
    benchmark_fpr             = None
    benchmark_ece             = None
    benchmark_dataset_size    = 200

    async def evaluate(self, interaction: AIInteraction) -> LLMEvaluationResult:
        import time
        t0 = time.perf_counter()
        raw = await self._call_llm(system=SYSTEM_PROMPT, user=self._format_interaction(interaction))
        parsed = self._parse_json_response(raw)
        return self._build_result(parsed, raw, evaluation_time_ms=(time.perf_counter()-t0)*1000)
```

---

## Adding a New Regulatory Domain

To add a new jurisdiction (SEBI, IRDAI, EU AI Act, etc.):

1. Create `govai/policies/<jurisdiction>/` with `__init__.py` and `policies.py`
2. Create `govai/evaluators/<jurisdiction>/` with `__init__.py`
3. Add the `Jurisdiction` enum value to `govai/core/models.py` if not present
4. Open a tracking issue titled: `[Domain] <Jurisdiction> <Framework> — <N> policies planned`

---

## Pull Request Checklist

- [ ] `policy_id` / `evaluator_id` follows dot-notation convention
- [ ] `regulation_ref` is exact and verifiable
- [ ] Unit tests pass locally
- [ ] Layer B: benchmark metrics filled in (not `None`)
- [ ] Layer B: dataset file committed to `datasets/`
- [ ] No hardcoded API keys anywhere
- [ ] `version = "1.0.0"` set on new modules
