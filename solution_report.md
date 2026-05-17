## Finance — Suspicious Transaction Detection

## Task 2: Define the Business Problem

# What problem is being solved?
Financial institutions process millions of transactions daily across payment networks, wire transfers, and card systems. Identifying suspicious or fraudulent transactions in real time is a critical compliance obligation and a core risk management function. Current rule-based systems — which flag transactions based on predefined thresholds and patterns — are brittle, generate massive false positive volumes, and are entirely blind to novel fraud schemes that don't match known rules.
# The problem: rule-based fraud detection cannot adapt to evolving fraud patterns, generates excessive false positives that overwhelm investigation teams, and fails to detect coordinated or subtle fraud rings.

# Who are the users or stakeholders?

**Primary users**: Fraud Analysts, AML (Anti-Money Laundering) Investigators, Compliance Officers
**Secondary stakeholders**: Risk Management leadership, Regulators (RBI, FinCEN, FCA), Customers whose accounts are affected
**Technical stakeholders**: Data Engineering, InfoSec, Model Risk Management

# What is the current manual or traditional process?
Rule-based transaction monitoring systems flag transactions that exceed predefined thresholds (e.g., transactions above ₹10 lakh, transactions to high-risk jurisdictions) or match known suspicious patterns. Flagged transactions enter a queue reviewed by human analysts who investigate transaction history, customer profiles, and counterparty relationships to determine whether a Suspicious Activity Report (SAR) must be filed.

# What are the limitations of the current process?

**Static rules cannot detect novel fraud** — new fraud schemes exploit gaps between rule updates, which are infrequent and manual
**False positive rate exceeds 85%** — analysts spend most of their time reviewing legitimate transactions flagged by over-broad rules
**No behavioral baseline** — rules cannot distinguish a transaction that is unusual for a specific customer from one that is universally unusual
**Cannot process velocity at scale** — real-time card and UPI transaction streams cannot be fully reviewed by humans
**Coordinated fraud rings invisible** — no graph-level analysis to detect distributed fraud across multiple accounts


## Task 3: Identify the AI Task Type

# Selected AI Task Type: Anomaly Detection + Classification (Hybrid)

Why this AI task type is suitable
Financial fraud has two distinct subproblems:
**Anomaly Detection** is required as the primary task because fraud patterns are not fully known in advance — new fraud schemes emerge continuously, and labeling every fraud type is impossible. The model must identify transactions that deviate significantly from a customer's established behavioral baseline, even if that deviation pattern has never been seen before.
**Classification** is applied as a secondary task on flagged transactions to categorize them by fraud type (card fraud, account takeover, money laundering, synthetic identity) — enabling efficient routing to the right investigation team and informing SAR filing decisions.
The hybrid approach covers both known fraud (via classification) and unknown fraud (via anomaly detection), which no purely supervised or purely unsupervised approach achieves alone.

## Task 4: Data Requirement Plan

# Type of data needed

**Primary: Structured** — Transaction records in tabular format (transaction ledger, card payment logs, UPI transaction records)
**Secondary: Structured** — Customer profile data, device metadata, geolocation logs

# Structured or unstructured data

Entirely **structured** tabular data.

# Input features

Feature Category Specific FeaturesTransaction attributesAmount, currency, transaction type (debit/credit/wire/UPI), channel (ATM/online/branch)Temporal featuresTimestamp, day of week, hour of day, time since last transactionMerchant dataMerchant Category Code (MCC), merchant name, merchant countryGeolocationTransaction location vs. customer home location; distance from previous transactionCustomer profileAccount age, average monthly transaction volume, typical transaction size distributionDevice metadataDevice fingerprint, IP address, new device flagNetwork featuresCounterparty account age, counterparty transaction historyVelocity featuresTransactions per hour, amount per 24 hours, unique merchants per week

# Target variable or labels

**Anomaly detection**: Reconstruction error (continuous) — no label required
**Classification**: Binary Fraudulent (1) / Legitimate (0) or multi-class fraud type

# Data collection method

Core banking transaction database (primary source)
Payment gateway logs (card and UPI transactions)
Customer KYC records and profile databases
Case management system (historical confirmed fraud labels from analyst investigations)
Device and IP intelligence feeds (third-party enrichment)

# Data quality risks

RiskDescriptionMitigationSevere class imbalanceFraud is typically <0.5% of transactionsSMOTE, anomaly detection (unsupervised), asymmetric lossLabel delayFraud confirmed only weeks after investigationSemi-supervised learning; timestamp-aware training splitsData driftFraud patterns evolve continuouslyMonthly retraining; concept drift detection (PSI, KS test)Adversarial manipulationFraudsters study model behavior to evade detectionEnsemble models; model architecture confidentialityMissing dataCross-border transactions may have incomplete merchant or geolocation dataImputation strategies; robust feature engineering

## Task 5: Model Recommendation

# Recommended Model: Autoencoder + Feed-Forward Neural Network Ensemble

Why this model is appropriate
An **Autoencoder** is the ideal unsupervised model for anomaly detection on transaction data:

It learns a compressed representation of normal transaction patterns during training
At inference, transactions it cannot reconstruct accurately (high reconstruction error) are anomalous
This approach requires no fraud labels — it learns from the vast majority of legitimate transactions
It naturally adapts to each customer's behavioral baseline when trained on per-customer sequences

A **Feed-Forward Neural Network (FFNN)** is then applied as a supervised classifier on the labeled fraud cases identified from historical investigations:

The FFNN learns discriminative features that distinguish confirmed fraud types
Provides interpretable class probability scores for SAR documentation
Can be updated frequently as new confirmed fraud labels become available

Together, the Autoencoder catches novel/unknown fraud; the FFNN classifies and categorizes known fraud types.

# Model architecture
Input
  └── Normalized transaction feature vector (40+ engineered features)

Stage 1: Anomaly Detection (Autoencoder)
  ├── Encoder
  │   ├── Dense(128, ReLU) → BatchNorm → Dropout(0.3)
  │   ├── Dense(64, ReLU)  → BatchNorm
  │   └── Dense(32, ReLU)           ← Bottleneck (latent representation)
  └── Decoder
      ├── Dense(64, ReLU)
      ├── Dense(128, ReLU)
      └── Dense(40, Linear)         ← Reconstructed transaction
  
  Anomaly Score = MSE(input, reconstruction)
  Threshold: Transactions with score > μ + 2σ → flagged

Stage 2: Classification (FFNN) — applied to flagged transactions
  ├── Dense(128, ReLU) → BatchNorm → Dropout(0.4)
  ├── Dense(64, ReLU)  → Dropout(0.3)
  ├── Dense(32, ReLU)
  └── Dense(N_classes, Softmax)
      → Output: Fraud type probability vector

Explainability Layer
  └── SHAP values computed for every flagged transaction
      → Feature attribution report attached to each SAR draft

Real-time Serving
  └── Kafka stream → Feature engineering → Model inference (<500ms) → Alert queue

# Training strategy

**Autoencoder**: Trained on 12 months of confirmed legitimate transactions; loss = MSE; Adam optimizer
**FFNN**: Trained on labeled fraud cases with class-weighted cross-entropy; oversampling for minority fraud types
**Threshold calibration**: Anomaly threshold set on a held-out validation set to achieve target recall ≥ 90%
**Retraining cadence**: Monthly on rolling 90-day window; triggered immediately on detected distribution shift


## Task 6: Evaluation Plan

# Technical metrics
MetricTargetRationaleRecall (Fraud class)≥ 90%Primary objective: catch fraud before financial lossPrecision (Fraud class)≥ 80%Reduce false positives overwhelming analyst teamsFalse Positive Rate< 5% of all flagged transactionsAnalyst workload sustainabilityAUC-PR (Precision-Recall)≥ 0.85Preferred metric for severely imbalanced datasetsInference latency (p99)< 500msRegulatory and UX requirement for real-time card decisions
Business metrics
KPITargetFinancial fraud losses≥ 30% reduction in fraud-related lossesAnalyst investigation efficiency≥ 50% reduction in false positive review timeSAR filing compliance100% SAR filed within regulatory deadlineCustomer account freezes due to false positives≥ 40% reduction
KPI baseline from reference data
The business_kpi_sample.csv shows that in early 2025, error rates peaked at 11.16% (May) and manual processing hours averaged 500+ hours/month with resolution times of 32–45 hours. The AI solution should target a trajectory similar to the improvement observed in H2 2025 (error rate declining to 4–6%, resolution time to 20–25 hours), achieved systematically rather than through operational fluctuation.

# Possible failure cases

Novel coordinated fraud rings operating across multiple accounts simultaneously evade single-account anomaly models
High-value legitimate transactions from VIP customers with unusual spending patterns repeatedly flagged
Model performance degradation during seasonal spending peaks (Diwali, year-end) due to distribution shift
Adversarial fraudsters who study model patterns by probing with low-value test transactions before executing large fraud

Human review and validation process

High-risk flags (anomaly score in top 1%) reviewed by senior fraud analysts within 4 hours
Medium-risk flags reviewed within 24 hours; auto-closed if no evidence found within 48 hours
SHAP explainability reports auto-generated for every flagged transaction to support analyst investigation and SAR documentation
Monthly model performance reviews tracking Precision, Recall, and false positive rate against baseline
Quarterly retraining with new confirmed fraud labels; full model validation before production deployment


## Task 7: Responsible AI Considerations

1. **False positives blocking legitimate users**
Incorrectly flagging a genuine transaction can freeze a customer's account, blocking access to funds during urgent needs. This harms customers and damages institutional trust.
Mitigation: Conservative account freeze policy — AI flags trigger investigation, not automatic freeze; automated reinstatement within 2 hours for transactions not escalated; customer appeal mechanism with < 24-hour resolution SLA.
2. **Algorithmic bias**
If training data reflects historical enforcement patterns biased toward certain geographic areas, transaction types, or customer segments, the model may disproportionately flag legitimate transactions from those groups.
Mitigation: Quarterly fairness audits comparing false positive rates across customer demographic segments, transaction types, and geographies; bias-aware training techniques (reweighting, adversarial debiasing).
3. **Privacy and data security**
Transaction data reveals granular details of customers' financial lives. Model training pipelines and inference systems process highly sensitive personal financial information.
Mitigation: Differential privacy during model training; on-premise deployment with no third-party data sharing; data minimization — only features necessary for the model are extracted; full encryption at rest and in transit; strict audit logging of all data access.
4. **Adversarial attacks**
Sophisticated fraudsters who reverse-engineer the model's decision boundary can craft transactions specifically designed to evade detection — a phenomenon known as adversarial evasion.
Mitigation: Ensemble model architecture (harder to reverse-engineer than a single model); model architecture is confidential; adversarial training with synthetic adversarial examples; regular red-team exercises by internal security teams.
5. **Regulatory explainability requirements**
Adverse actions on customer accounts (freezes, SAR filings) require documented, explainable rationale under regulations such as GDPR (Right to Explanation), RBI guidelines, and FinCEN requirements.
Mitigation: SHAP-based feature attribution for every flagged transaction; full decision audit trail stored for 7 years; model cards documenting training data, intended use, and performance metrics; legal review of all model output used in adverse action decisions.

## Task 8: Final Solution Summary

CategoryDetailsProblemRule-based fraud detection produces >85% false positives, misses novel fraud patterns, and cannot scale with real-time transaction velocityProposed AI SolutionTwo-stage hybrid system: Autoencoder for unsupervised behavioral anomaly detection + FFNN classifier for fraud type categorization, with SHAP explainabilityRequired Data12 months of transaction records (labeled + unlabeled); customer profile data; device/IP metadata; historical confirmed fraud labels from case management systemModel RecommendationAutoencoder (anomaly detection) + FFNN ensemble (classification) with SHAP explanations and real-time Kafka serving pipelineExpected Business Impact≥30% reduction in fraud losses; ≥50% reduction in analyst false-positive workload; <500ms real-time inference; 100% SAR regulatory complianceKey RisksFalse positives blocking customers (mitigated by human-in-the-loop); adversarial evasion (mitigated by ensemble + red-teaming); demographic bias (mitigated by quarterly fairness audits)