# GovAI — AI Governance-as-Code

**Executable, auditable AI behavioral policy enforcement for regulated industries.**

GovAI is an open source framework for defining AI behavioral policies as versioned, testable Python code — enforced continuously in production and producing regulatory-grade audit evidence.

> Terraform for enterprise AI governance.

---

## The Problem

Every enterprise deploying AI cannot prove their AI behaves the way they say it does.

Legal, compliance, regulators, and CISOs ask:
- Can you prove your banking chatbot never gave specific investment advice?
- Can you prove your credit AI always explained rejection reasons?
- Can you prove your AI always disclosed it was AI when asked?

Current answer: system prompt + manual testing + hope. That is not sufficient for regulatory scrutiny.

## The Solution

GovAI provides a two-layer hybrid enforcement system:

| Layer | What it does | Who builds it |
|-------|-------------|---------------|
| **Layer A — Rule-Based** | Deterministic Python policy modules. Fast (< 10ms), fully auditable, 100% confidence. Catches clear violations. | Policy library contributors |
| **Layer B — LLM Evaluator** | LLM-powered evaluators with calibrated confidence scores. Catches nuanced grey-zone violations rules miss. | AI evaluation contributors |

The **Arbitration Engine** reconciles both layers into a single `ComplianceDecision` with a cryptographically hashed audit trail.

## Regulatory Coverage

| Jurisdiction | Framework | Status |
|-------------|-----------|--------|
| 🇮🇳 India | RBI FREE-AI Framework (Aug 2025) | ✅ v0.1 — 6 policies, 1 LLM evaluator |
| 🇮🇳 India | SEBI AI Guidelines | 🔜 Planned v0.2 |
| 🇮🇳 India | IRDAI AI Guidelines | 🔜 Planned v0.2 |
| 🇮🇳 India | DPDP Act 2023 | 🔜 Planned v0.3 |
| 🇪🇺 EU | EU AI Act Articles 52 + Annex III | 🔜 Planned v0.4 |

## Quick Start

```bash
pip install govai
```

```python
import asyncio
import govai.policies.rbi  # registers all RBI policies

from govai import EnforcementEngine, AIInteraction, Jurisdiction

async def main():
    engine = EnforcementEngine(jurisdiction=Jurisdiction.IN)

    interaction = AIInteraction(
        application_id="hdfc-chatbot-prod",
        model_id="gpt-4o",
        user_query="Should I invest in HDFC Bank stock?",
        ai_response=(
            "HDFC Bank has shown strong fundamentals over the past year. "
            "Many analysts consider it a core holding in Indian banking portfolios."
        ),
    )

    decision = await engine.evaluate(interaction)
    print(decision.status)           # ComplianceStatus.REVIEW
    print(decision.arbitration_reason)
    # → "Rules passed. LLM evaluator flagged a GREY_ZONE violation with
    #    confidence 0.88 (threshold 0.85). Analyst consensus framing may
    #    constitute implicit investment advice under RBI FREE-AI Rec-15."

asyncio.run(main())
```

## Architecture

```
govai/
├── core/
│   ├── models.py          # AIInteraction, ComplianceDecision, etc.
│   ├── base.py            # RuleBasedPolicy, LLMEvaluator abstract classes
│   ├── engine.py          # EnforcementEngine — the entry point
│   ├── arbitration.py     # ArbitrationEngine — reconciles both layers
│   ├── audit.py           # Append-only audit trail (JSONL)
│   └── registry.py        # Auto-discovery registry
├── policies/
│   └── rbi/
│       └── free_ai/
│           └── policies.py  # RBI FREE-AI Rec-15 through Rec-20
├── evaluators/
│   └── rbi/
│       └── investment_advice.py  # Rec-15 LLM evaluator
└── tests/
    └── rbi/
        └── test_rbi_policies.py
```

## The Arbitration Logic

| Rule Result | LLM Result | Decision | Action |
|-------------|------------|----------|--------|
| PASS | PASS | **CLEAN** | Log. No action. |
| FAIL (Critical) | Any | **VIOLATION** | Flag. Alert compliance team. |
| PASS | FAIL (confidence ≥ 0.85) | **REVIEW** | Flag for human review. |
| PASS | FAIL (confidence < 0.85) | **NEAR_MISS** | Log for weekly review. |

## Contributing

GovAI is built by a community of contributors. See [CONTRIBUTING.md](CONTRIBUTING.md).

**Layer A contributors** build rule-based policy modules:
- One class, one regulatory requirement, deterministic Python
- Must include unit tests with clear violation and clean cases
- Citation accuracy verified by domain lead before merge

**Layer B contributors** build LLM evaluators:
- Stage 1: Build 200-example benchmark dataset (60 clear / 60 clean / 80 grey-zone)
- Stage 2: Design prompt architecture citing specific regulatory text
- Stage 3: Measure precision, recall, ECE — must meet thresholds before merge
- Stage 4: Integrate into runtime with benchmark metrics in docstring

**Merge thresholds for LLM evaluators:**

| Metric | Minimum |
|--------|---------|
| Precision — clear violations | ≥ 0.90 |
| Precision — grey zone | ≥ 0.75 |
| Recall — clear violations | ≥ 0.85 |
| False positive rate | ≤ 0.05 |
| Expected Calibration Error | ≤ 0.10 |

## Running Tests

```bash
pip install govai[dev]
pytest govai/tests/ -v --cov=govai
```

## License

Apache License 2.0. See [LICENSE](LICENSE).

## Regulatory Disclaimer

GovAI is a software tool designed to assist with AI governance implementation. It does not constitute legal advice. Organizations must consult qualified legal and compliance professionals when implementing regulatory compliance programs. GovAI contributors and maintainers make no warranty regarding the regulatory accuracy of any policy module.
