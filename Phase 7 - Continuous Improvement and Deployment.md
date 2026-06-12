Below is the deep technical drill-down into **Phase 7 — Continuous improvement loop** for the Interfor POD / delivery-slip scenario. This phase transforms the IDP solution from a one-time deployment into a self-improving, production-grade system.

***

# Phase 7 — Continuous improvement loop (deep dive)

***

# 1. Why continuous improvement is non-negotiable

Document Intelligence custom models are **not static**. Their accuracy degrades over time because:

1. **Document variability evolves.** New Interfor branches open. New customers appear. New product codes launch. Carrier names change. Form layouts receive minor updates.
2. **Scan and photo quality varies.** Mobile devices change. Scanning hardware is replaced. Lighting conditions differ seasonally.
3. **Business processes change.** New exception types emerge. New fields become required. New restriction codes appear. Notes sections expand.
4. **Models expire.** Microsoft states: *"The model is configured to expire two years after its creation for all requests utilizing a GA API to build it."* citeturn11search45

Additionally, Microsoft states: *"Classifiers should be periodically updated whenever the following changes occur: You add new templates for an existing class. You add new document types for recognition. Classifier confidence is low."* citeturn11search30

The continuous improvement loop ensures the system adapts to these changes proactively rather than reactively.

***

# 2. The closed-loop improvement architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                    PRODUCTION PROCESSING                        │
│                                                                 │
│  Inbound → Classify → Extract → Validate → Route → Post        │
│                                    │                            │
│                              ┌─────┴─────┐                     │
│                              │ Validation │                     │
│                              │  Results   │                     │
│                              └─────┬─────┘                     │
│                                    │                            │
│                    ┌───────────────┼───────────────┐            │
│                    │               │               │            │
│                    ▼               ▼               ▼            │
│              Straight-        Human           Error             │
│              through          Review          Queue             │
│                                │                                │
│                         ┌──────┴──────┐                         │
│                         │ Corrections │                         │
│                         │  Captured   │                         │
│                         └──────┬──────┘                         │
└────────────────────────────────┼────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    IMPROVEMENT ENGINE                            │
│                                                                 │
│  Step 1: Collect production feedback data                       │
│  Step 2: Categorise failure patterns                            │
│  Step 3: Curate retraining dataset                              │
│  Step 4: Retrain model in Studio                                │
│  Step 5: Evaluate new model against holdout test set            │
│  Step 6: Compare new model vs current production model          │
│  Step 7: Promote or reject new model                            │
│  Step 8: Update orchestration to use new model ID               │
│  Step 9: Monitor post-deployment performance                    │
│  Step 10: Repeat                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

***

# 3. Step 1 — Collect production feedback data

## 3.1 Four data sources for improvement

| Source                                      | What it provides                                         | How to capture                          |
| ------------------------------------------- | -------------------------------------------------------- | --------------------------------------- |
| **Validation issues**                       | Rules that triggered, confidence scores, failure reasons | Dataverse `POD Validation Issue` table  |
| **Human review corrections**                | Reviewer-corrected field values                          | Dataverse `POD Review Decision` table   |
| **Straight-through post-processing errors** | Downstream ERP/TMS rejections after auto-posting         | ERP/TMS error callback or exception log |
| **User-reported issues**                    | Operations staff flagging incorrect extractions          | Manual reporting or feedback form       |

## 3.2 Correction capture schema (from Phase 5)

Each reviewer correction should be stored as:

```json
{
  "jobId": "string",
  "extractionModelId": "interfor-pod-extraction-v001",
  "correctedFields": {
    "OrderNumber": {
      "originalValue": "244805B",
      "correctedValue": "2448058",
      "correctionType": "OCR_error",
      "originalConfidence": 0.72
    },
    "ShipmentNumber": {
      "originalValue": null,
      "correctedValue": "5285729",
      "correctionType": "Missing_field",
      "originalConfidence": null
    }
  },
  "correctedProductLines": [
    {
      "lineNumber": 2,
      "correctedColumn": "Quantity",
      "originalValue": 1.366,
      "correctedValue": 1.386,
      "correctionType": "OCR_error"
    }
  ],
  "reviewerAction": "ApproveWithCorrections",
  "reviewedAt": "datetime",
  "reviewedBy": "reviewer ID"
}
```

## 3.3 Automated feedback pipeline

Build a scheduled Power Automate flow or Azure Function that:

1. Queries all `POD Review Decision` rows from the last improvement cycle period.
2. Queries all `POD Validation Issue` rows with severity ≥ High.
3. Queries all downstream posting failures.
4. Aggregates into a **feedback dataset**.
5. Exports source documents and corrections to a staging blob container.

```text
/improvement/
  /cycle-2026-07/
    /feedback-dataset/
      /extraction-failures/
      /low-confidence/
      /reviewer-corrections/
      /downstream-rejections/
      /user-reported/
    /analysis/
      feedback-summary.json
```

***

# 4. Step 2 — Categorise failure patterns

## 4.1 Failure taxonomy

| Category                        | Description                                                 | Example from POD scenario                   | Improvement action                                            |
| ------------------------------- | ----------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------- |
| **OCR error**                   | Text correctly located but misread                          | `244805B` instead of `2448058`              | Add more samples of that text style; verify scan quality      |
| **Missing field**               | Field present in document but not extracted                 | Shipment Number blank despite being printed | Add more labelled samples; check labelling consistency        |
| **Wrong field value**           | Incorrect value assigned to field                           | Order Date assigned to Shipment Date        | Audit labelling; check for spatial ambiguity                  |
| **Table structure error**       | Product lines merged, split, or misaligned                  | Two product groups collapsed into one       | Add multi-product-group samples; re-label tables              |
| **Signature detection failure** | Signature present but detected as "unsigned"                | Faint or partial signature not detected     | Add more signature samples with variations citeturn11search27 |
| **Handwriting not captured**    | Handwritten exception note not extracted                    | "Wrong Size Lumber" missed                  | Add more region-labelled handwriting samples                  |
| **New document variant**        | Document from new branch/layout not seen in training        | New Interfor branch with different header   | Add samples from new branch; retrain                          |
| **Classification error**        | Document assigned to wrong class                            | Invoice classified as delivery slip         | Add samples to classifier training; retrain classifier        |
| **Confidence calibration**      | Model confidently wrong (high confidence, wrong value)      | Most dangerous failure type                 | Add document to training with correct labels; retrain         |
| **Downstream rejection**        | Extracted values syntactically correct but business-invalid | Order number format changed in ERP          | Update validation rules; may not require retraining           |

## 4.2 Failure analysis report template

Generate this report for each improvement cycle:

```text
Improvement Cycle: 2026-07
Period: 2026-06-15 to 2026-07-14
Model: interfor-pod-extraction-v001
Classifier: interfor-pod-classifier-v001

Total documents processed:          1,247
Straight-through rate:               78.3% (976)
Review rate:                         19.6% (244)
Error rate:                           2.2% (27)

Top validation rules triggered:
  V006 Low confidence OrderNumber:    87 occurrences
  V013 Handwritten exception note:    54 occurrences
  V017 Product line pieces mismatch:  38 occurrences
  V009 Missing customer signature:    31 occurrences
  V001 Missing Order Number:          12 occurrences

Top correction types:
  OCR error:                          43 corrections
  Missing field:                      28 corrections
  Table structure error:              19 corrections
  Signature detection failure:         8 corrections

New variants identified:
  New branch: Savannah / SV (3 documents)
  New product code: HF28STUD (2 documents)
  New carrier: Pacific Freight LLC (4 documents)

Recommended actions:
  1. Add 5+ samples from Savannah branch
  2. Add 5+ samples with HF28STUD product code
  3. Add 3+ low-quality mobile photo samples
  4. Retrain extraction model as v002
  5. Add Savannah branch samples to classifier
```

***

# 5. Step 3 — Curate the retraining dataset

## 5.1 Dataset composition strategy

Microsoft states: *"A balanced dataset represents all the typical variations you would expect to see for the document. Creating a balanced dataset results in a model with the highest possible accuracy."* citeturn11search34

The retraining dataset should be composed of:

```text
Original training dataset (from v001)
  + Production failure documents (labelled with correct values)
  + New variant documents (labelled)
  - Remove any mislabelled documents from original set (if discovered)
```

## 5.2 Adding corrections to the training dataset

For each reviewer correction:

1. Copy the original source document from `/review/` to `/training/extraction/`.
2. Run Layout on the document to generate a fresh `.ocr.json` file.
3. Create or update the `.labels.json` file with the **corrected** values.
4. Verify the labels are consistent with the field taxonomy.

**Critical requirement for retraining with a new API version:** Microsoft states: \*"To retrain a model with a more recent API version, ensure that the layout results for the documents in your training dataset correspond to the API version of the build model request. For instance, if you plan to build the model with the v3.1:2023-07-31 API version, the corresponding *.ocr.json files in your training dataset should also be generated with the v3.1:2023-07-31 API version."* citeturn11search45

This means: **if the new model will use the same API version (2024-11-30), the existing `.ocr.json` files are fine. If upgrading to a newer API version, regenerate all `.ocr.json` files.**

## 5.3 Dataset size management

| Limit                         | Value  | Source             |
| ----------------------------- | ------ | ------------------ |
| Neural training data max size | 1 GB   | citeturn11search34 |
| Neural training max pages     | 50,000 | citeturn11search34 |

If the cumulative dataset exceeds these limits, curate by:

* Keeping the most representative samples.
* Removing redundant samples that are very similar.
* Prioritising documents that caused production failures.
* Keeping at least 5 samples per variation dimension.

## 5.4 Dataset versioning

```text
/training/extraction/
  /ds-v001/   ← original training set
  /ds-v002/   ← v001 + cycle 1 corrections + new variants
  /ds-v003/   ← v002 + cycle 2 corrections + new variants
```

Never modify the original dataset in place. Always create a new versioned copy.

***

# 6. Step 4 — Retrain the model in Studio

## 6.1 Extraction model retraining

1. Open Document Intelligence Studio → **Custom extraction model**.
2. Open the project `Interfor-POD-DeliverySlip-Extraction`.
3. Point the project to the new dataset folder (`/training/extraction/ds-v002/`).
4. Run Layout on any new documents that do not have `.ocr.json` files.
5. Label all new documents according to the labelling guide.
6. Verify labelling consistency across old and new documents.
7. Select **Train**.
8. Enter a **new model ID**: `interfor-pod-extraction-v002`.
9. Select **Build mode: Neural**.
10. Set `maxTrainingHours` appropriately.
11. Start training.

Microsoft states: *"When you're choosing between the two model types, start with a neural model to determine if it meets your functional needs."* citeturn11search24

**Training cost:** Microsoft states: *"With version v4.0 2024-11-30 (GA), you can receive 10 hours of free model training, and train a model for as long as 10 hours."* citeturn11search27 Budget for paid training hours during improvement cycles.

## 6.2 Classifier retraining (if needed)

### Option A — Incremental training

Use incremental training when adding samples to existing classes or adding new classes.

Microsoft states: *"Incremental training doesn't introduce any new API endpoints. The documentClassifiers:build request payload is modified to support incremental training. Incremental training results in a new classifier model being created with the existing classifier left untouched."* citeturn11search30

```http
POST {endpoint}/documentintelligence/documentClassifiers:build?api-version=2024-11-30

{
  "classifierId": "interfor-pod-classifier-v002",
  "baseClassifierId": "interfor-pod-classifier-v001",
  "docTypes": {
    "DeliverySlip_POD": {
      "azureBlobSource": {
        "containerUrl": "...",
        "prefix": "classifier-incremental/DeliverySlip_POD_new_branch/"
      }
    }
  }
}
```

### Option B — Full retrain

Use full retrain when:

* Many classes changed simultaneously.
* The base classifier is suspect.
* Upgrading API version.

```text
New classifier ID: interfor-pod-classifier-v002
Full dataset: /training/classifier/ds-v002/
```

***

# 7. Step 5 — Evaluate the new model

## 7.1 Holdout test set

**Never test with training data.** Maintain a separate holdout test set that is **never** used for training.

```text
/test/
  /holdout/          ← permanent test set, never used for training
    test_clean_001.pdf
    test_photo_001.jpeg
    test_exception_001.pdf
    test_multiproduct_001.jpeg
    ...
  /regression/       ← documents from previous production failures
    regression_001.pdf
    regression_002.jpeg
    ...
  /new-variants/     ← new document types not in training
    newbranch_001.pdf
    ...
```

## 7.2 Evaluation methodology

For each test document, run analysis with **both** the current production model and the new candidate model:

```text
Current: interfor-pod-extraction-v001
New:     interfor-pod-extraction-v002
```

Capture:

| Document       | Field          | v001 value | v001 confidence | v002 value | v002 confidence | Ground truth | v001 correct? | v002 correct? |
| -------------- | -------------- | ---------- | --------------- | ---------- | --------------- | ------------ | ------------- | ------------- |
| test\_001.pdf  | OrderNumber    | 2448058    | 0.95            | 2448058    | 0.97            | 2448058      | ✅             | ✅             |
| test\_002.jpeg | ShipmentNumber | 528572B    | 0.72            | 5285729    | 0.91            | 5285729      | ❌             | ✅             |
| ...            | ...            | ...        | ...             | ...        | ...             | ...          | ...           | ...           |

## 7.3 Evaluation metrics

Calculate these metrics for both models:

### Field-level metrics

```text
For each field:
  Accuracy = count(correct) / count(total)
  Average confidence = mean(confidence scores)
  False positive rate = count(confidently wrong) / count(total)
```

### Table-level metrics

```text
Table detection rate = count(table detected) / count(total documents with tables)
Row accuracy = count(correct rows) / count(total rows)
Cell accuracy = count(correct cells) / count(total cells)
```

### Document-level metrics

```text
Full document accuracy = count(all fields correct) / count(total documents)
Straight-through eligible rate = count(documents passing all validation rules) / count(total)
```

## 7.4 Regression testing

**Critical:** The new model must not perform **worse** on any document that the current model handles correctly.

```text
Regression check:
  For each test document:
    If v001 was correct AND v002 is incorrect:
      Flag as REGRESSION
      Document must be investigated

  If any regression exists:
    Do NOT promote v002 without resolution
```

## 7.5 Comparison report template

```text
Model Comparison Report
========================
Current model: interfor-pod-extraction-v001
Candidate model: interfor-pod-extraction-v002
Test set: holdout + regression (42 documents)

Field-level accuracy:
  OrderNumber:       v001: 93%  → v002: 97%  (+4%)  ✅ Improved
  ShipmentNumber:    v001: 90%  → v002: 95%  (+5%)  ✅ Improved
  ProductLines:      v001: 85%  → v002: 91%  (+6%)  ✅ Improved
  CustomerSignature: v001: 88%  → v002: 92%  (+4%)  ✅ Improved
  HandwrittenNote:   v001: 60%  → v002: 75%  (+15%) ✅ Improved
  BranchLocation:    v001: 95%  → v002: 94%  (-1%)  ⚠️ Marginal

Document-level accuracy:
  Full accuracy:     v001: 72%  → v002: 84%  (+12%) ✅ Improved

Regressions:
  0 regressions detected  ✅

Recommendation: PROMOTE v002
```

***

# 8. Step 6 — Promotion decision

## 8.1 Promotion criteria

| Criterion                                             | Threshold                         | Required?   |
| ----------------------------------------------------- | --------------------------------- | ----------- |
| Overall field accuracy improvement                    | > 0% (any improvement)            | Yes         |
| Zero regressions on holdout set                       | 0 regressions                     | Yes         |
| Critical field accuracy (OrderNumber, ShipmentNumber) | ≥ 90%                             | Yes         |
| No field accuracy decrease > 2%                       | < 2% decrease on any field        | Yes         |
| Straight-through eligible rate improvement            | > 0% improvement                  | Recommended |
| Reviewer approval                                     | Signed off by solution specialist | Yes         |

## 8.2 Promotion approval record

```json
{
  "promotionId": "PROMO-2026-07-001",
  "currentModelId": "interfor-pod-extraction-v001",
  "candidateModelId": "interfor-pod-extraction-v002",
  "testSetVersion": "holdout-v002 + regression-v001",
  "testDocumentCount": 42,
  "overallAccuracyChange": "+12%",
  "regressionCount": 0,
  "criticalFieldAccuracy": {
    "OrderNumber": 0.97,
    "ShipmentNumber": 0.95
  },
  "decision": "PROMOTE",
  "approvedBy": "Solution Specialist Name",
  "approvedAt": "2026-07-15T14:30:00Z",
  "deploymentTarget": "PROD",
  "rollbackPlan": "Revert to interfor-pod-extraction-v001"
}
```

***

# 9. Step 7 — Deploy the new model

## 9.1 Copy model to production environment

Use the two-step copy process:

### Step 1 — Authorise copy on target (PROD)

```http
POST {prod-endpoint}/documentintelligence/documentModels:authorizeCopy?api-version=2024-11-30

{
  "modelId": "interfor-pod-extraction-v002",
  "description": "Promoted from DEV after cycle 2026-07 evaluation"
}
```

### Step 2 — Execute copy from source (DEV/TEST)

```http
POST {dev-endpoint}/documentintelligence/documentModels/interfor-pod-extraction-v002:copyTo?api-version=2024-11-30
```

Body: the authorisation response from Step 1.

citeturn11search45

## 9.2 Update orchestration parameters

### Logic Apps

Update the Logic App parameter:

```json
{
  "extractionModelId": "interfor-pod-extraction-v002"
}
```

### Power Automate

Update the flow variable or environment variable:

```text
extractionModelId = "interfor-pod-extraction-v002"
```

### Blue-green deployment option

For zero-downtime deployment:

1. Deploy the new model alongside the old model.
2. Route a percentage of traffic to the new model (e.g., 10%).
3. Monitor for regressions.
4. Gradually increase traffic to 100%.
5. Retire the old model after the monitoring period.

Implement this in Logic Apps or Power Automate using a condition:

```text
If random() < 0.10:
  Use interfor-pod-extraction-v002
Else:
  Use interfor-pod-extraction-v001
```

***

# 10. Step 8 — Post-deployment monitoring

## 10.1 Key metrics to monitor after promotion

| Metric                            | Baseline (v001) | Target (v002) | Alert if         |
| --------------------------------- | --------------- | ------------- | ---------------- |
| Straight-through rate             | 78.3%           | ≥ 80%         | Drops below 75%  |
| Average OrderNumber confidence    | 0.91            | ≥ 0.93        | Drops below 0.88 |
| Average ShipmentNumber confidence | 0.88            | ≥ 0.92        | Drops below 0.85 |
| Review rate                       | 19.6%           | ≤ 18%         | Exceeds 22%      |
| Error rate                        | 2.2%            | ≤ 2%          | Exceeds 3%       |
| V014 exception keyword rate       | 4.3%            | Stable        | Spikes above 6%  |
| Reviewer correction rate          | 35% of reviewed | ≤ 30%         | Exceeds 40%      |

## 10.2 Monitoring period

```text
First 24 hours: active monitoring, check every hour
Days 2–7: daily monitoring
Weeks 2–4: weekly monitoring
After week 4: normal cadence (weekly or bi-weekly)
```

## 10.3 Rollback criteria

```text
If straight-through rate drops more than 5% below baseline → rollback
If any critical field confidence drops more than 10% below baseline → rollback
If reviewer correction rate increases more than 15% above baseline → rollback
If error rate doubles → rollback
```

## 10.4 Rollback procedure

1. Update orchestration parameter to previous model ID:

```text
extractionModelId = "interfor-pod-extraction-v001"
```

2. Log the rollback with reason.
3. Investigate the cause of degradation.
4. Add failing documents to the next improvement cycle.

***

# 11. Model lifecycle and expiration management

## 11.1 Expiration rules

Microsoft states: *"The model is configured to expire two years after its creation for all requests utilizing a GA API to build it."* citeturn11search45

Microsoft also states: *"Models trained with a preview API shouldn't be used in production and should be retrained once the corresponding GA API version is available. Compatibility between preview API versions and GA API versions isn't always maintained."* citeturn11search45

### Expiration timeline for this scenario

```text
interfor-pod-extraction-v001
  Created: 2026-06-10
  API version: 2024-11-30 (GA)
  Expires: 2028-06-10

interfor-pod-extraction-v002
  Created: 2026-07-15
  API version: 2024-11-30 (GA)
  Expires: 2028-07-15
```

## 11.2 Viewing expiration date

Microsoft states: *"The GET model API returns the model details including the expirationDateTime property."* citeturn11search45

```http
GET {endpoint}/documentModels/interfor-pod-extraction-v001?api-version=2024-11-30
```

Response:

```json
{
  "modelId": "interfor-pod-extraction-v001",
  "createdDateTime": "2026-06-10T09:00:00Z",
  "expirationDateTime": "2028-06-10T09:00:00Z",
  "apiVersion": "2024-11-30"
}
```

## 11.3 Proactive expiration management

```text
Set calendar reminder: 6 months before expiration
  → Begin planning retraining cycle

Set calendar reminder: 3 months before expiration
  → Retrain model with latest dataset and current GA API version
  → Evaluate and promote new model

Set calendar reminder: 1 month before expiration
  → Verify new model is in production
  → Verify old model is no longer referenced
```

## 11.4 API version retirement

Microsoft states: *"Notification of retirement of any particular GA API version will be communicated at least 3 years before expiration."* citeturn11search45

Monitor the <https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/whats-new> page for API version retirement notices. citeturn11search29

### API retirement timeline (current)

Microsoft states: citeturn11search29

* *"Document Intelligence REST API v2.1 reaches end of support on September 15, 2027."*
* *"Document Intelligence REST API 2022-08-31 v3.0 reaches end of support on March 30, 2029."*
* v4.0 (2024-11-30) is the current GA version.

## 11.5 Retraining for API version migration

Microsoft states: *"To retrain a model with a more recent API version, ensure that the layout results for the documents in your training dataset correspond to the API version of the build model request."* citeturn11search45

This means when migrating from v4.0 to a future API version:

1. Regenerate all `.ocr.json` files using the new API version's Layout API.
2. Verify labelling compatibility.
3. Retrain the model with the new API version.
4. Evaluate against holdout test set.
5. Promote only after successful evaluation.

***

# 12. Improvement cycle cadence

## 12.1 Recommended cadence

| Phase                   | Frequency                                | Trigger                                                             |
| ----------------------- | ---------------------------------------- | ------------------------------------------------------------------- |
| **Reactive cycle**      | Ad hoc                                   | Sudden accuracy drop, new document variant, business process change |
| **Scheduled cycle**     | Monthly (first 6 months), then quarterly | Calendar-based review of production metrics                         |
| **Expiration cycle**    | Every 18 months                          | Model approaching 2-year expiration                                 |
| **API migration cycle** | When new GA API version released         | Microsoft announcement                                              |

## 12.2 Cycle timeline

```text
Week 1: Collect feedback data
Week 2: Analyse failure patterns, curate retraining dataset
Week 3: Retrain models, evaluate against holdout set
Week 4: Promote or reject, deploy, monitor
```

## 12.3 Improvement velocity targets

| Metric                | Cycle 1 target | Cycle 2 target | Cycle 3 target | Steady state |
| --------------------- | -------------- | -------------- | -------------- | ------------ |
| Straight-through rate | 78% → 84%      | 84% → 88%      | 88% → 91%      | ≥ 90%        |
| Review rate           | 20% → 14%      | 14% → 10%      | 10% → 8%       | ≤ 10%        |
| Error rate            | 2.2% → 1.5%    | 1.5% → 1%      | 1% → 0.5%      | ≤ 1%         |

***

# 13. Batch API for bulk reprocessing

When a new model is promoted, you may want to reprocess historical documents to capture previously missed fields or correct previously wrong values.

Microsoft states: *"The batch analysis API allows you to bulk process up to 10,000 documents using one request."* citeturn11search39

## 13.1 Reprocessing workflow

```text
1. Identify documents processed with v001 that had validation issues.
2. Copy source documents to a reprocessing container.
3. Call Batch API with v002 model.
4. Compare new results with original results.
5. Update Dataverse records where v002 produces better results.
6. Log all changes for audit.
```

## 13.2 Batch API call

```http
POST {endpoint}/documentintelligence/documentModels/interfor-pod-extraction-v002:analyzeBatch?api-version=2024-11-30

{
  "azureBlobSource": {
    "containerUrl": "https://stpodidpprod.blob.core.windows.net/reprocessing?<SAS>"
  },
  "resultContainerUrl": "https://stpodidpprod.blob.core.windows.net/reprocessing-results?<SAS>",
  "resultPrefix": "v002-reprocess/",
  "overwriteExisting": true
}
```

Microsoft states: *"Batch operation results are retained for 24 hours after completion."* citeturn11search39

## 13.3 Batch API limits

| Limit                                | Value    | Source             |
| ------------------------------------ | -------- | ------------------ |
| Max documents per batch              | 10,000   | citeturn11search39 |
| Results retention                    | 24 hours | citeturn11search39 |
| Requires source Azure Blob container | Yes      | citeturn11search39 |
| Requires result Azure Blob container | Yes      | citeturn11search39 |

***

# 14. Classifier improvement — specific considerations

## 14.1 When to retrain the classifier

Microsoft states: *"Classifiers should be periodically updated whenever the following changes occur: You add new templates for an existing class. You add new document types for recognition. Classifier confidence is low."* citeturn11search30

## 14.2 Incremental vs full retrain decision

| Situation                           | Approach             | Rationale                             |
| ----------------------------------- | -------------------- | ------------------------------------- |
| Adding samples to existing classes  | Incremental training | Faster; preserves existing classifier |
| Adding 1–2 new classes              | Incremental training | New classes added alongside existing  |
| Significant changes to many classes | Full retrain         | Clean slate with full control         |
| Upgrading API version               | Full retrain         | Layout results must match API version |
| Base classifier is suspect          | Full retrain         | Avoid propagating errors              |

## 14.3 Incremental training limitations

Microsoft states: citeturn11search30

* *"Incremental training only works when the base classifier and the incrementally trained classifier are both trained on the same API version."*
* *"The GET response from an incrementally trained classifier differs from the standard classifier GET response. The incrementally trained classifier doesn't return all the document types supported. It returns the document types added or updated in the incremental training step and the extended base classifier."*
* *"Deleting a base classifier doesn't impact the use of an incrementally trained classifier."*

## 14.4 Classifier version chain management

```text
interfor-pod-classifier-v001 (base)
  ↓ incremental: added new branch samples
interfor-pod-classifier-v002 (base: v001)
  ↓ incremental: added ClaimDocument class
interfor-pod-classifier-v003 (base: v002)
  ↓ full retrain (API version upgrade or major restructure)
interfor-pod-classifier-v004 (new base)
```

Track the complete lineage in the model version register. When using GET on an incrementally trained classifier, query the base classifier to get the complete class list.

***

# 15. Composed model improvement

If the production system uses composed models (Phase 3, Section 13):

## 15.1 Recompose after extraction model update

When any constituent extraction model is retrained:

1. Retrain or verify the classifier used in the composed model.
2. Recompose with updated model IDs:

```http
POST {endpoint}/documentintelligence/documentModels:compose?api-version=2024-11-30

{
  "modelId": "interfor-pod-extraction-composed-v002",
  "classifierId": "interfor-pod-quality-classifier-v001",
  "docTypes": {
    "CleanPDF": { "modelId": "interfor-pod-extraction-clean-v002" },
    "MobilePhoto": { "modelId": "interfor-pod-extraction-photo-v002" },
    "ExceptionPOD": { "modelId": "interfor-pod-extraction-exception-v001" }
  }
}
```

3. Test the composed model against the holdout set.
4. Promote only if overall accuracy improves.

***

# 16. Validation rule improvement

Validation rules (Phase 5) should also evolve alongside models.

## 16.1 Rule improvement triggers

| Trigger                                          | Action                                                               |
| ------------------------------------------------ | -------------------------------------------------------------------- |
| A validation rule triggers on > 30% of documents | Investigate: is the rule too strict or is the model underperforming? |
| A validation rule never triggers                 | Evaluate: is the rule still relevant? Is the threshold too lenient?  |
| Reviewers consistently override a specific rule  | Adjust the rule threshold or remove the rule                         |
| New exception keywords appear in production      | Add keywords to the V014 exception keyword list                      |
| New handwritten annotation codes appear          | Add codes to the V015 known annotation list                          |
| Downstream ERP/TMS validation changes            | Update V028/V029 cross-document validation rules                     |

## 16.2 Confidence threshold recalibration

After each model promotion, recalibrate confidence thresholds:

```text
1. Run new model against holdout set.
2. For each field, plot confidence vs accuracy.
3. Identify the confidence threshold where accuracy reaches the target (e.g., 95%).
4. Update the threshold in orchestration parameters.
```

Microsoft states: *"For scenarios where accuracy is critical, confidence can be used to determine whether to automatically accept the prediction or flag it for human review."* citeturn11search28

***

# 17. Operational dashboard

## 17.1 Recommended dashboard views

### View 1 — Processing overview (daily/weekly)

```text
Total documents processed
Straight-through rate
Review rate
Error rate
Average processing time
```

### View 2 — Model quality trend (weekly/monthly)

```text
Average confidence per critical field (trend line)
Validation rule trigger rates (trend line)
Reviewer correction rate (trend line)
Straight-through rate (trend line)
```

### View 3 — Improvement cycle status

```text
Current production model ID and version
Model expiration date
Days since last improvement cycle
Next scheduled improvement cycle
Training hours used this month (vs 10 free hours)
```

### View 4 — Exception analysis

```text
Exception type distribution (pie chart)
Exception keyword frequency (bar chart)
New exception types (flagged for attention)
```

***

# 18. Complete Phase 7 deliverables

| #  | Deliverable                         | Owner                   | Description                                                         |
| -- | ----------------------------------- | ----------------------- | ------------------------------------------------------------------- |
| 1  | Feedback collection pipeline        | Developer               | Automated export of corrections, validation issues, and errors      |
| 2  | Failure analysis report template    | Solution specialist     | Standardised format for each improvement cycle                      |
| 3  | Retraining dataset curation process | Model builder / BA      | Steps for adding corrections and new variants to training data      |
| 4  | Dataset versioning convention       | Model builder           | `ds-v001`, `ds-v002`, etc.                                          |
| 5  | Model retraining runbook            | Model builder           | Step-by-step Studio retraining process                              |
| 6  | Holdout test set                    | QA                      | Permanent test set never used for training                          |
| 7  | Evaluation methodology              | Solution specialist     | Side-by-side comparison of old vs new model                         |
| 8  | Regression testing process          | QA                      | Verify no regressions on previously correct documents               |
| 9  | Comparison report template          | Solution specialist     | Field-level and document-level accuracy comparison                  |
| 10 | Promotion decision record template  | Solution specialist     | Formal approval with criteria and evidence                          |
| 11 | Model copy and deployment runbook   | Developer / platform    | DEV → TEST → PROD copy steps                                        |
| 12 | Blue-green deployment guide         | Developer               | Gradual traffic migration procedure                                 |
| 13 | Post-deployment monitoring plan     | Operations              | Metrics, thresholds, and alert configuration                        |
| 14 | Rollback procedure                  | Developer / operations  | Steps to revert to previous model                                   |
| 15 | Model lifecycle calendar            | Solution specialist     | Expiration dates, scheduled improvement cycles, API migration dates |
| 16 | Batch reprocessing runbook          | Developer               | Steps to reprocess historical documents with new model              |
| 17 | Classifier improvement plan         | Model builder           | Incremental vs full retrain decision criteria                       |
| 18 | Validation rule review process      | BA                      | Periodic review of rule relevance and thresholds                    |
| 19 | Confidence recalibration process    | Solution specialist     | Post-promotion threshold adjustment                                 |
| 20 | Operational dashboard specification | Operations / developer  | Views, metrics, and alerting                                        |
| 21 | Training cost tracking              | Model builder / finance | Hours used, free hours remaining, paid hours budget                 |

***

# Accuracy checks — Phase 7 specific

| Claim                                                                          | Source                                                                                                                                                   | Status                         |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| GA models expire 2 years after creation                                        | [Custom model lifecycle](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-lifecycle?view=doc-intel-4.0.0)          | ✅ Confirmed citeturn11search45 |
| Preview models expire 2 years after creation; should not be used in production | Same source                                                                                                                                              | ✅ Confirmed citeturn11search45 |
| GET model API returns `expirationDateTime` property                            | Same source                                                                                                                                              | ✅ Confirmed citeturn11search45 |
| Retraining with new API version requires regenerating `.ocr.json` files        | Same source                                                                                                                                              | ✅ Confirmed citeturn11search45 |
| GA API version retirement communicated ≥ 3 years before expiration             | Same source                                                                                                                                              | ✅ Confirmed citeturn11search45 |
| v2.1 end of support: September 15, 2027                                        | [What's new](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/whats-new?view=doc-intel-4.0.0)                                   | ✅ Confirmed citeturn11search29 |
| v3.0 end of support: March 30, 2029                                            | Same source                                                                                                                                              | ✅ Confirmed citeturn11search29 |
| Classifiers should be updated when new templates, new types, or low confidence | [Incremental classifiers](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/incremental-classifier?view=doc-intel-4.0.0) | ✅ Confirmed citeturn11search30 |
| Incremental training creates new classifier; original untouched                | Same source                                                                                                                                              | ✅ Confirmed citeturn11search30 |
| Incremental training requires same API version                                 | Same source                                                                                                                                              | ✅ Confirmed citeturn11search30 |
| Balanced dataset = highest accuracy                                            | [Build and train](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0)     | ✅ Confirmed citeturn11search34 |
| Neural training max 1 GB, 50,000 pages                                         | Same source                                                                                                                                              | ✅ Confirmed citeturn11search34 |
| 5 documents minimum; 10–15 for lower quality                                   | Same source                                                                                                                                              | ✅ Confirmed citeturn11search34 |
| Custom neural model: start with neural to determine functional fit             | [Custom document models](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)              | ✅ Confirmed citeturn11search24 |
| v4.0 neural supports signature detection, table confidence, overlapping fields | [Custom neural model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)                | ✅ Confirmed citeturn11search27 |
| 10 free training hours/month; paid after that                                  | Same source                                                                                                                                              | ✅ Confirmed citeturn11search27 |
| Batch API: up to 10,000 documents per request; results retained 24 hours       | [Batch analysis](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/batch-analysis?view=doc-intel-4.0.0)                 | ✅ Confirmed citeturn11search39 |
| Confidence can be used to accept or flag for human review                      | [Accuracy and confidence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)    | ✅ Confirmed citeturn11search28 |

***

This completes the Phase 7 deep dive. The continuous improvement loop is now fully specified with feedback collection, failure categorisation, dataset curation, retraining procedures, evaluation methodology, regression testing, promotion criteria, deployment strategies (including blue-green), post-deployment monitoring, rollback procedures, model lifecycle management, batch reprocessing, classifier-specific improvement, validation rule evolution, and operational dashboarding.

All seven phases of the Interfor POD / delivery-slip IDP implementation are now documented at full technical depth. Would you like me to produce a downloadable artefact — such as the custom connector OpenAPI definition, a Dataverse solution schema, or a consolidated executive summary of the entire implementation guide?
