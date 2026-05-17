# part-4-ai-solution-design

# 💰 Finance — Suspicious Transaction Detection

AI-powered fraud detection using Autoencoder + FFNN ensemble with real-time inference and SHAP explainability.

---

## Problem
Rule-based systems miss novel fraud, generate **>85% false positives**, and can't scale with real-time transaction volumes.

## Solution
Two-stage AI pipeline:
- **Stage 1 — Autoencoder:** Detects anomalous transactions by learning normal behaviour (unsupervised)
- **Stage 2 — FFNN + SHAP:** Classifies fraud type and generates explainability reports for analysts

## Tech Stack
| Component | Detail |
|-----------|--------|
| Model | Autoencoder + Feed-Forward Neural Network |
| Data | Structured transaction records (40+ features) |
| Serving | Kafka stream → inference < 500ms |
| Explainability | SHAP feature attribution per flagged transaction |

## Key Metrics
| KPI | Target |
|-----|--------|
| Fraud Recall | ≥ 90% |
| False Positive Rate | < 5% |
| Inference Latency | < 500ms |
| Fraud Loss Reduction | ≥ 30% |

## Files
```
finance/
├── README.md
├── solution_report.md          # Full task breakdown (Tasks 1–8)
├── finance_solution_architecture.png
└── business_kpi_sample.csv
```

## Responsible AI
- SHAP explanations required for all adverse account actions
- Quarterly fairness audits across customer segments
- Human analyst reviews all High-risk flags before action