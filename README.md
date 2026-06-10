Below is the **consolidated, corrected, and optimised version** of all phases — incorporating every error fix, gap closure, inconsistency resolution, and optimisation identified in the re-assessment. All field values cited from the attached documents are based on visual inspection of the uploaded images and PDFs; the `office365_open` calls for the PDFs returned `Expired`, so PDF-sourced values should be verified by running the documents through Document Intelligence.

***

# Consolidated IDP implementation guide — Interfor POD / delivery-slip automation

## Using Azure Document Intelligence v4.0 (2024-11-30 GA), Document Intelligence Studio, Azure Logic Apps, and Microsoft Power Platform

***

# Document inventory and analysis

Four documents were provided. All four are **Interfor Delivery Slip / proof-of-delivery** documents sharing a common business family but exhibiting meaningful variability.

## Document 1 — `63461366_response.pdf`

Single-page Interfor delivery slip. citeturn1search1

## Document 2 — `POD Scenario 3.PDF`

Multi-page delivery slip with handwritten exception note "Wrong Size Lumber" and reference number `# 0013856`. citeturn1search2

## Document 3 — `63506743_response.jpeg` (Image 1)

| Field              | Value                                                                            |
| ------------------ | -------------------------------------------------------------------------------- |
| Sold To            | The Building Center Inc., Pineville, NC 28134                                    |
| Order Number       | 2448058                                                                          |
| Order Date         | May 15, 2026                                                                     |
| Salesperson        | Will Shy                                                                         |
| Customer           | 300905                                                                           |
| Ship To            | Shane Pridgen, 1265 Pridgen Rd, Broxton, GA 31519                                |
| Carrier            | Christopher Woods dba Fire Trail Transport                                       |
| Route              | 5/19 prod.                                                                       |
| Phone              | 910-546-4700                                                                     |
| Shipment           | 5285729                                                                          |
| Shipment Date      | May 29, 2026                                                                     |
| Branch/Location    | Preston / PS                                                                     |
| Unit               | Truck                                                                            |
| Unit #             | 03                                                                               |
| Restrictions       | Tarp required CALL 24 HRS IN ADVANCE                                             |
| Product            | Southern Yellow Pine 2×4 No.2 S4S Eased Edge Kiln Dried and Heat Treated, SY24#2 |
| Total              | 18 units, 3,536 pieces, 23.578 MBF                                               |
| Actual Load Weight | 0.000 LB                                                                         |
| Shipper signature  | Present, dated 5/29                                                              |
| Customer signature | Present                                                                          |
| Print Date/Time    | May 29, 2026 05:19 AM PT                                                         |

## Document 4 — `63574059response.jpeg` (Image 2)

| Field                   | Value                                                                                  |
| ----------------------- | -------------------------------------------------------------------------------------- |
| Sold To                 | Builders FirstSource, 6031 Connection Drive Suite 400, Irving, TX 75039                |
| Order Number            | 2446642                                                                                |
| Order Date              | May 12, 2026                                                                           |
| Salesperson             | Keith Abbott                                                                           |
| Customer                | 42836607                                                                               |
| Ship To                 | Builders FirstSource, 20815 67th Ave NE, Arlington, WA 98223                           |
| Phone                   | 360-925-4100                                                                           |
| Shipment                | 5287709                                                                                |
| Branch/Location         | Port Angeles / PA                                                                      |
| Unit                    | Truck                                                                                  |
| Restrictions            | Mon-Fri 7am-3:30pm                                                                     |
| Notes                   | JIT CONTRACT ORDER - PRIORITY SHIPMENT                                                 |
| Products                | HF242+ (Hem-Fir 2×4 #2 Structural & Btr S4S) and HF24STUD (Hem-Fir 2×4 Stud & Btr S4S) |
| Total                   | 22 units, 6,468 pieces, 36.652 MBF                                                     |
| Estimated Load Weight   | 61,392 LB                                                                              |
| Shipper signature       | "Dylan King", dated DSU 6/72/26 (handwritten)                                          |
| Carrier signature       | "K Franklin"                                                                           |
| Customer signature      | Present with date 5/26/26                                                              |
| Handwritten annotations | "DSU" marks on quantity column                                                         |
| Print Date/Time         | May 22, 2026 09:43 AM PT                                                               |

## Key variability observed across all four documents

| Variation dimension         | Evidence                                                                  |
| --------------------------- | ------------------------------------------------------------------------- |
| Different branches          | Preston/PS, Port Angeles/PA, DeQuincy/DQ, and others                      |
| Different customers         | Building Center Inc., Builders FirstSource, and others                    |
| Different products          | Southern Yellow Pine SY24#2, Hem-Fir HF242+, HF24STUD, SY24#1, SY26#1     |
| Different carriers          | Christopher Woods dba Fire Trail Transport, K Franklin, RSC Logistics LLC |
| Different restriction types | Tarp required, delivery-window restrictions, call-ahead                   |
| Notes section               | Present on some slips (JIT CONTRACT ORDER), absent on others              |
| Route field                 | Present on some slips, absent on others                                   |
| Shipment Date field         | Present on some slips, absent on others                                   |
| Load weight type            | "Actual" vs "Estimated"                                                   |
| Handwritten annotations     | Signature names, DSU marks, exception notes                               |
| Multiple product groups     | Some slips have one product section, Image 2 has two                      |
| Scan/photo quality          | Clean PDF vs mobile-photo capture with clip artefacts                     |

***

# Service limits and constraints reference

Before designing any phase, teams must know the platform boundaries. Microsoft documents these limits for Document Intelligence v4.0: citeturn6search164

| Limit                           | F0 (Free)                 | S0 (Standard)                      |
| ------------------------------- | ------------------------- | ---------------------------------- |
| Max document size               | 4 MB                      | 500 MB                             |
| Max pages (analysis)            | 2                         | 2,000                              |
| Image dimensions                | 50×50 to 10,000×10,000 px | 50×50 to 10,000×10,000 px          |
| Min text height                 | 12 px at 1024×768         | 12 px at 1024×768                  |
| Analyze transactions/sec        | 1                         | 15 (default, adjustable)           |
| Get operations/sec              | 1                         | 50 (default, adjustable)           |
| Model management ops/sec        | 1                         | 5 (default, adjustable)            |
| Neural training data size       | 1 GB                      | 1 GB                               |
| Neural training pages           | 50,000                    | 50,000                             |
| Template training data size     | 50 MB                     | 50 MB                              |
| Template training pages         | 500                       | 500                                |
| Classifier training data (v4.0) | 1 GB                      | 2 GB                               |
| Classifier training pages       | 25,000                    | 25,000                             |
| Max classes per classifier      | 1,000                     | 1,000                              |
| Max samples per class           | 100                       | 100                                |
| Neural model train time         | 10 hrs/month free         | Pay by the hour, 10 free hrs/month |
| Max template models             | 500                       | 5,000                              |
| Max neural models               | 100                       | 500                                |
| Compose model limit             | 5                         | 500                                |
| Batch API max docs/request      | —                         | 10,000                             |

***

# Corrected and complete field taxonomy

Based on all four documents, the canonical field taxonomy is:

```text
DocumentIdentity
  - DocumentType                    (String)
  - SourceCompany                   (String) — e.g. "Interfor"
  - SourceAddress                   (String) — header address varies by branch
  - PrintDateTime                   (String)
  - PageNumber                      (String)

OrderInformation
  - SoldToName                      (String)
  - SoldToAddress                   (String)
  - CustomerNumber                  (String)
  - OrderNumber                     (String)  ← critical
  - OrderDate                       (Date)
  - Salesperson                     (String)

ShippingInformation
  - ShipToName                      (String)
  - ShipToAddress                   (String)
  - ShipToPhone                     (String)
  - CarrierName                     (String)
  - Route                           (String)  ← present on some slips
  - CarrierPhone                    (String)
  - ShipmentNumber                  (String)  ← critical
  - ShipmentDate                    (Date)    ← present on some slips
  - BranchLocation                  (String)
  - Unit                            (String)
  - UnitNumber                      (String)
  - Restrictions                    (String)

Notes                               (String)  ← e.g. "JIT CONTRACT ORDER"

ProductInformation (table/array — may repeat for multiple product groups)
  - ProductDescription              (String)
  - ProductCode                     (String)
  - Length                          (String)
  - Units                           (Number)
  - Pieces                          (Number)
  - PcsPerUnit                      (Number)
  - Quantity                        (Number)
  - UOM                             (String)

Totals
  - SubTotalUnits                   (Number)
  - SubTotalPieces                  (Number)
  - SubTotalQuantity                (String)
  - TotalUnits                      (Number)
  - TotalPieces                     (Number)
  - TotalQuantity                   (String)
  - LoadWeight                      (String)
  - LoadWeightType                  (String)  ← "Actual" vs "Estimated"

PODSignatures
  - ShipperSignature                (Signature — draw region, one region per field)
  - ShipperSignatureDate            (String)
  - CarrierSignature                (Signature — draw region, one region per field)
  - CarrierSignatureDate            (String)
  - CustomerDeliverySignature       (Signature — draw region, one region per field)
  - CustomerDeliverySignatureDate   (String)
  - ReceivedByName                  (String)  ← handwritten names

ExceptionInformation
  - HandwrittenNoteText             (String)
  - HandwrittenAnnotations          (String)  ← e.g. "DSU" marks
  - ExceptionReferenceNumber        (String)
```

**Signature labelling correction:** Microsoft states: *"To label a signature, use field type as Signature and draw the regions for signature. Signature field only supports one draw region per field."* citeturn6search142 Each signature must be a separate Signature-type field with one draw region.

***

# Phase 0 — Environment setup

## 0.1 Provision Azure resources

```text
Resource group: rg-idp-pod-prod

Azure AI Document Intelligence: di-pod-prod (S0 tier)
Storage account: stpodidpprod
  Containers:
    /inbound
    /processing
    /results
    /review
    /archive
    /error
    /training
      /extraction/
        /clean-pdf/
        /scanned/
        /mobile-photo/
        /signed/
        /exception/
        /multi-product/
      /classifier/
        /delivery-slip-pod/
        /bol/
        /invoice/
        /other/
    /test
    /holdout
Key Vault: kv-pod-prod (for API keys if not using managed identity)
Application Insights / Log Analytics: ai-pod-prod
```

## 0.2 Configure RBAC

| Role                          | Resource                       | Who                                                                      |
| ----------------------------- | ------------------------------ | ------------------------------------------------------------------------ |
| Cognitive Services User       | Document Intelligence resource | All Studio users, Logic App managed identity, Power Automate connections |
| Storage Blob Data Contributor | Storage account                | All Studio users, Logic App managed identity                             |
| Storage Account Contributor   | Storage account                | One-time CORS configuration admin                                        |

## 0.3 Configure CORS on storage

Required for Document Intelligence Studio to access training documents in blob storage.

## 0.4 Managed identity OAuth audience

When using managed identity (Logic Apps or Power Automate), the OAuth2 audience must be:

```text
https://cognitiveservices.azure.com
```

Microsoft's REST API documentation confirms this as the OAuth2 scope for Document Intelligence endpoints. citeturn3search52turn3search68

***

# Phase 1 — Baseline evaluation

## 1.1 Objective

Determine whether Read, Layout, or Layout+queryFields can extract business-critical fields adequately, or whether a custom extraction model is required.

## 1.2 Test with Read

1. Open Document Intelligence Studio → **Read** model.
2. Upload all four sample documents.
3. Run analysis.
4. Review extracted text lines, words, handwritten text, and detected languages.
5. Export JSON results.

The Read model extracts printed and handwritten text lines, words, locations, and detected languages. citeturn6search170

## 1.3 Test with Layout

1. Open Document Intelligence Studio → **Layout** model.
2. Upload the same four documents.
3. Run analysis.
4. Review:
   * Product table detection and column separation.
   * Section heading recognition.
   * Paragraph separation.
   * Selection marks (if any).
5. Compare against Read results.

The Layout model extracts text, tables, selection marks, and document structure. citeturn6search170

## 1.4 🆕 Test with Layout + queryFields

This is a **new addition** from the re-assessment. Before investing in custom model labelling, test whether the `queryFields` add-on feature can extract key business fields zero-shot.

Query fields allow extraction of specific named fields without training a custom model. Microsoft states: *"For query field extraction, specify the fields you want to extract and Document Intelligence analyzes the document accordingly."* citeturn6search163

Query fields is an **add-on feature** that is priced differently from other add-ons. citeturn6search158

**Test REST call:**

```http
POST https://{endpoint}/documentintelligence/documentModels/prebuilt-layout:analyze?api-version=2024-11-30&features=queryFields&queryFields=OrderNumber,ShipmentNumber,BranchLocation,CustomerNumber,Salesperson,CarrierName,TotalQuantity,LoadWeight
```

Evaluate whether queryFields extraction is accurate enough for each field. If it achieves acceptable accuracy on clean documents, it can serve as a rapid baseline. If accuracy is insufficient — especially on tables, signatures, and handwritten notes — custom extraction is confirmed.

## 1.5 🆕 Input pre-validation

**Correction from re-assessment:** External image pre-processing (deskew, contrast enhancement, etc.) is generally unnecessary because Document Intelligence v4.0 handles image quality internally. Microsoft guidance is: *"For best results, provide one clear photo or high-quality scan per document."* citeturn6search153

Instead, implement **input validation** before submission:

```text
If file size > 500 MB → reject or split
If page count > 2,000 → reject or split
If image dimensions < 50×50 px → reject
If PDF is password-locked → reject (remove lock first)
If min text height < 12 px at 1024×768 → warn
```

These limits are documented in the service limits. citeturn6search164

## 1.6 Baseline accuracy matrix

| Test area           | Read | Layout    | Layout + queryFields | Custom neural target       |
| ------------------- | ---- | --------- | -------------------- | -------------------------- |
| Order Number        | ?    | ?         | ?                    | High                       |
| Shipment Number     | ?    | ?         | ?                    | High                       |
| Product table       | Poor | Fair/Good | Fair                 | High                       |
| Signature detection | Poor | Poor      | Poor                 | High (v4.0 Signature type) |
| Handwritten notes   | Fair | Fair      | Fair                 | High                       |
| Total Quantity      | ?    | ?         | ?                    | High                       |
| Notes field         | ?    | ?         | ?                    | High                       |

## 1.7 Deliverables

* Read/Layout/queryFields comparison report
* JSON samples for each model
* OCR/layout failure log
* Model selection recommendation (expected: custom neural required)

***

# Phase 2 — Business extraction schema definition

## 2.1 Objective

Convert the field taxonomy into a Studio-ready labelling schema.

## 2.2 Field types in Studio

Microsoft supports these field types for custom extraction: citeturn6search142turn6search155

| Studio field type     | Used for                                                  |
| --------------------- | --------------------------------------------------------- |
| Field (String)        | Text values like OrderNumber, ShipToName                  |
| Selection mark        | Checkboxes (not applicable for these PODs)                |
| Signature             | Signature regions — **draw region, one region per field** |
| Table (dynamic/fixed) | Product line items                                        |

## 2.3 🆕 Overlapping fields consideration

Microsoft states: *"Custom neural v4.0 2024-11-30 (GA) model supports … overlapping fields."* and *"Overlapping fields are supported with REST API version 2024-11-30 (GA). Overlapping fields have some limits."* citeturn6search142

For the Interfor delivery slip, overlapping fields may apply where:

* Carrier name and carrier phone overlap spatially.
* Product description and product code overlap.
* Sub-total row overlaps with product table.

Labellers should use overlapping fields where business semantics require it, but should test for limits documented by Microsoft.

## 2.4 Deliverables

* Locked field dictionary (use the corrected taxonomy above)
* Labelling guide with conventions
* JSON target contract
* Validation rule catalogue

***

# Phase 3 — Custom extraction model build

## 3.1 Create the Studio project

1. Open Document Intelligence Studio → **Custom extraction model**.
2. Select **Create a project**.
3. Project name: `Interfor-POD-DeliverySlip-Extraction`
4. Select the Document Intelligence resource.
5. Select the storage account, container, and folder path (`/training/extraction/`).
6. Review and create.

Microsoft confirms: *"Document Intelligence custom models require a handful of training documents to get started. If you have at least five documents, you can get started training a custom model."* citeturn6search153

## 3.2 Prepare the training dataset

Use representative samples across all variation dimensions identified in the document inventory:

* Clean system-generated PDFs
* Scanned copies
* Mobile-photo captures (including clip/edge artefacts as in Image 2)
* Signed PODs (all three signature areas populated)
* Unsigned PODs
* Exception PODs (handwritten notes)
* Single product group (Images 1, Document 1)
* Multiple product groups (Image 2 with HF242+ and HF24STUD)
* Different branches (Preston/PS, Port Angeles/PA, DeQuincy/DQ)
* Different customers
* Different restriction types
* Documents with Notes section vs without
* Documents with Route field vs without
* Documents with Shipment Date vs without

**Training data limits:** Neural model training data cannot exceed 1 GB or 50,000 pages. citeturn6search164 Five documents is the minimum to start; Microsoft recommends 10–15 images for lower-quality forms. citeturn6search153

## 3.3 Run Layout/OCR in the project

1. Upload documents.
2. Run layout for all documents.
3. Verify that key text and table cells are selectable.
4. If text is not selectable, mark as quality issue.

## 3.4 Define and label fields

### Scalar fields

Create all fields from the corrected taxonomy. Use **String** type for most fields, **Date** type for date fields.

### Signature fields — corrected guidance

Create three separate Signature-type fields:

* `ShipperSignature`
* `CarrierSignature`
* `CustomerDeliverySignature`

For each, **draw a region** around the signature area. **Each Signature field supports only one draw region.** citeturn6search142

Signature dates should be **separate String or Date fields**, not part of the Signature draw region.

### Table field

Create a dynamic table named `ProductLines` with columns:

* ProductDescription
* ProductCode
* Length
* Units
* Pieces
* PcsPerUnit
* Quantity
* UOM

### Notes field

Create a String field for `Notes` to capture content like "JIT CONTRACT ORDER - PRIORITY SHIPMENT".

### Handwritten annotations

Create a String field for `HandwrittenAnnotations` to capture marks like "DSU" that are not full exception notes but may carry operational meaning.

## 3.5 Train the model

1. Select **Train**.
2. Model ID: `interfor-pod-extraction-v001`
3. Build mode: **Neural**
4. Description: `Custom neural extraction model for Interfor delivery slip/POD documents. v4.0 2024-11-30. Extracts order, shipment, product lines, totals, POD signatures, exception notes.`
5. Start training.

Microsoft confirms: *"When you're choosing between the two model types, start with a neural model to determine if it meets your functional needs."* citeturn6search143 Custom neural models support signature detection, table cell confidence, and overlapping fields in v4.0. citeturn6search142

**Training time consideration:** Neural models take longer to train than template models. Microsoft states: *"Custom neural model train: 10 hours per month (F0), no limit (S0, pay by the hour), start with 10 free hours each month."* citeturn6search164 Budget for training time, especially during iterative improvement cycles.

## 3.6 Test the model

1. Select the trained model.
2. Open the **Test** function.
3. Add test files **not used in training**.
4. Run **Analyze**.
5. Review extracted values, field confidence, table structure, JSON response.
6. Add weak test files back into training if needed and relabel.

## 3.7 🆕 Consider composed models

Microsoft states: *"The v4.0 2024-11-30 (GA) model compose operation adds an explicitly trained classifier instead of an implicit classifier for analysis."* citeturn6search133

**When to use composed models for this scenario:** If separate extraction models perform better for different document quality tiers (e.g., one model for clean PDFs, another for mobile photos), compose them into a single model ID. The composed model uses an explicitly trained classifier to route documents to the best extraction model. citeturn6search133

For the initial implementation, start with a single neural model. Evaluate composed models only if accuracy testing reveals that a single model cannot handle all quality tiers adequately.

## 3.8 Deliverables

* Trained model ID: `interfor-pod-extraction-v001`
* Labelled training set
* Test result set with field-level confidence
* JSON response samples
* Known limitation register

***

# Phase 4 — Custom classification model (conditional)

## 4.1 Decision criteria

Build a classifier **only if** the intake contains multiple document types or multi-document packets. For a controlled intake of only Interfor delivery slips, defer classification.

## 4.2 Classifier requirements

Microsoft states: *"Training a custom classifier requires at least two distinct classes and a minimum of five document samples per class."* citeturn6search150

Classifier limits (v4.0): citeturn6search164

* Max classes: 1,000
* Max samples per class: 100
* Training data: 2 GB (S0), 1 GB (F0)
* Training pages: 25,000

## 4.3 Suggested classes

```text
DeliverySlip_POD
BillOfLading
Invoice
CarrierDocument
Other_Review
```

## 4.4 Build in Studio

1. Create **Custom classification** project.
2. Connect to `/training/classifier/` folder.
3. Define document classes.
4. Label at least five samples per class.
5. Train classifier.
6. Test with single documents, multi-page packets, mixed packets, and unknown documents.

## 4.5 🆕 `split` parameter — corrected

**Correction:** The REST API **query parameter name** is `split`, not `splitMode`. Microsoft's documentation uses `splitMode` in prose descriptions but the actual REST parameter is `split`. citeturn6search150

* Default value: `none` (documents are **not** split by default in v4.0).
* Set `split=auto` if the input file contains multiple documents.

```http
POST {endpoint}/documentintelligence/documentClassifiers/{classifierId}:analyze?api-version=2024-11-30&split=auto
```

## 4.6 Incremental training

Microsoft confirms: *"Custom classification v4.0 2024-11-30 (GA) models support incremental training. You can add new samples to existing classes or add new classes by referencing an existing classifier."* citeturn6search150turn6search139

## 4.7 Deliverables

* Classifier model ID: `interfor-pod-classifier-v001`
* Class list and confidence thresholds
* Page range test evidence
* Routing matrix: class → extraction model → workflow

***

# Phase 5 — Validation rules and exception handling

## 5.1 Required field checks

```text
If OrderNumber is empty → Review
If ShipmentNumber is empty → Review
If ShipToName is empty → Review
If ProductLines is empty → Review
If TotalQuantity is empty → Review
```

## 5.2 Confidence checks

```text
If confidence(OrderNumber) < threshold → Review
If confidence(ShipmentNumber) < threshold → Review
If confidence(ProductLines.Quantity) < threshold → Review
If confidence(CustomerDeliverySignature) < threshold → Review
```

## 5.3 Signature checks

```text
If CustomerDeliverySignature is missing → Review
If CustomerDeliverySignatureDate is missing → Review
```

## 5.4 Exception note checks

```text
If HandwrittenNoteText is not empty → Review
If HandwrittenNoteText contains:
    wrong size, shortage, damage, refused, missing, overage, claim
Then:
    ExceptionType = classify_keyword(HandwrittenNoteText)
    RequiresHumanReview = true
```

## 5.5 🆕 Handwritten annotation checks

```text
If HandwrittenAnnotations is not empty
    AND does not match known codes (e.g., "DSU")
    → Review: unknown handwritten annotation
```

## 5.6 Quantity reconciliation

```text
For each product line:
    expectedPieces = Units * PcsPerUnit
    If expectedPieces != Pieces → Review

If sum(ProductLines.Quantity) != TotalQuantity → Review
```

## 5.7 🆕 Load weight checks

```text
If LoadWeight = "0.000 LB" and LoadWeightType = "Actual"
    → Flag: zero actual load weight may indicate weighing was skipped

If LoadWeightType is missing → warn
```

## 5.8 🆕 Notes/priority checks

```text
If Notes contains "JIT" or "PRIORITY"
    → Tag as priority shipment for downstream SLA handling
```

## 5.9 🆕 Duplicate detection

```text
Hash the document content (e.g., SHA-256 of binary content)
If hash matches a previously processed document → flag as potential duplicate
```

## 5.10 Deliverables

* Validation rule specification
* Confidence threshold matrix
* Exception routing matrix
* Developer pseudocode
* Test evidence

***

# Phase 6A — Developer integration via Azure Logic Apps

## 6A.1 Recommended resources

```text
Azure Logic Apps Standard: la-pod-idp-prod
  - System-assigned managed identity enabled
  - OAuth audience: https://cognitiveservices.azure.com
```

## 6A.2 🆕 Trigger — corrected recommendation

**Optimisation:** Use **Event Grid** instead of blob polling triggers for near-real-time, reliable event delivery.

```text
Trigger: Event Grid - When a resource event occurs
  Resource type: Microsoft.Storage.StorageAccounts
  Event type: Microsoft.Storage.BlobCreated
  Subject filter: /blobServices/default/containers/inbound/
```

Fallback: Azure Blob Storage trigger **When a blob is added or updated** (Standard built-in) if Event Grid is not available.

## 6A.3 Initialise workflow context

Variables:

```text
correlationId
sourceFileName
sourceBlobPath
documentContentHash        ← NEW: for duplicate detection
classifierModelId = "interfor-pod-classifier-v001"
extractionModelId = "interfor-pod-extraction-v001"
apiVersion = "2024-11-30"
classificationConfidenceThreshold = 0.85
criticalFieldConfidenceThreshold = 0.80
requiresReview = false
reviewReasons = []
productLines = []
```

## 6A.4 🆕 Input validation scope

Before submitting to Document Intelligence:

```text
Get blob properties
If file size > 500 MB → Error: file too large
If file extension not in [pdf, jpg, jpeg, png, bmp, tiff, heif] → Error: unsupported format
Compute SHA-256 hash of blob content → store in documentContentHash
Check hash against previously processed hashes → if match, flag as duplicate
```

## 6A.5 Classification sequence (optional)

### Submit classification

```http
POST {endpoint}/documentintelligence/documentClassifiers/{classifierId}:analyze?_overload=classifyDocument&api-version=2024-11-30&stringIndexType=utf16CodeUnit&split=auto
```

The classifier API accepts `base64Source` or `urlSource` and returns `202 Accepted` with `Operation-Location` and `Retry-After` headers. citeturn3search68

### 🆕 Corrected: use `Retry-After` in polling

```text
Delay
  Count: @{int(outputs('HTTP_-_Submit_Classification')?['headers']?['Retry-After'])}
  Unit: Second
```

**Correction:** Both Analyze Document and Classify Document responses include a `Retry-After` header. citeturn3search52turn3search68 Use this value dynamically rather than a hardcoded delay.

### Poll and parse

```text
Do until status = succeeded or failed
  Delay (Retry-After seconds)
  HTTP GET Operation-Location
Parse JSON - Classification result
```

The classifier result includes `documents`, `docType`, `boundingRegions`, and `confidence`. citeturn3search73

### Route

```text
If status = failed → Error
If confidence < threshold → Review
If docType != DeliverySlip_POD → Review
Else → continue to extraction
```

## 6A.6 Extraction sequence

### 🆕 Choose submission method

**Correction from re-assessment:** Two submission methods exist:

**Option A — JSON body with base64 (smaller documents):**

```http
POST {endpoint}/documentintelligence/documentModels/{modelId}:analyze?_overload=analyzeDocument&api-version=2024-11-30
Content-Type: application/json
Body: { "base64Source": "<base64>" }
```

**Option B — Binary stream (larger documents, more efficient):**

```http
POST {endpoint}/documentintelligence/documentModels/{modelId}:analyze?api-version=2024-11-30
Content-Type: application/pdf  (or image/jpeg, image/png, etc.)
Body: <binary content>
```

The binary stream method avoids the \~33% size increase from base64 encoding. Use Option B for documents approaching size limits.

The Analyze Document API returns `202 Accepted` with `Operation-Location` and `Retry-After` headers. citeturn3search52

### Poll extraction result — corrected

```text
Do until status = succeeded or failed
  Delay (Retry-After seconds)
  HTTP GET Operation-Location
Parse JSON - Extraction result
```

The Get Analyze Result API returns `status` and `analyzeResult`. citeturn3search63

## 6A.7 Map to canonical JSON

Map fields from the parsed extraction result to the canonical POD payload. The exact JSON paths depend on the trained model's field names.

### 🆕 Product lines — parallel product groups

For documents like Image 2 with multiple product groups (HF242+ and HF24STUD), the extraction model may return them as separate table instances. Use **Apply to each** over the table array to flatten all product lines into a single `productLines` array.

## 6A.8 Validation sequence

Implement all rules from Phase 5 using Conditions, Compose, and variable operations.

## 6A.9 Outcome routing

### Straight-through path

```text
Create result JSON blob in /results
Call downstream ERP/TMS/API
Copy original to /archive
```

### Review path

```text
Create review JSON blob in /review
Copy original to /review
Queue review task (Service Bus, review database, or Teams notification)
```

### Error path

```text
Create error JSON blob in /error
Copy original to /error
Emit operational notification
```

## 6A.10 🆕 Searchable PDF archival

Microsoft confirms: *"The prebuilt read model now supports images formats (JPEG/JPG, PNG, BMP, TIFF, HEIF) and language expansion to include Chinese, Japanese, and Korean for PDF output."* citeturn6search139 Searchable PDF is a **free add-on** with the Read model. citeturn6search158

For archival, add an optional step to generate a searchable PDF:

```http
POST {endpoint}/documentintelligence/documentModels/prebuilt-read:analyze?api-version=2024-11-30&output=pdf
```

Store the searchable PDF alongside the extraction JSON for downstream audit, search, and compliance.

## 6A.11 🆕 Batch API for high-volume processing

For end-of-day bulk uploads, use the Batch API instead of one-at-a-time processing.

Microsoft states: *"The batch analysis API allows you to bulk process up to 10,000 documents using one request."* citeturn6search144

```http
POST {endpoint}/documentintelligence/documentModels/{modelId}:analyzeBatch?api-version=2024-11-30
```

The batch API requires source and result Azure Blob Storage containers. citeturn6search144 Batch results are retained for 24 hours. citeturn6search144

## 6A.12 🆕 Rate limiting and throttling

S0 default limits: 15 analyze transactions/sec, 50 get operations/sec. citeturn6search164

Implement retry-with-backoff logic in the Logic App. Use the `Retry-After` header from `429 Too Many Requests` responses. Configure HTTP action retry policies.

## 6A.13 🆕 Parallel extraction for multi-document packets

After classification identifies page ranges for multiple documents in a packet, submit extraction calls for different page ranges in parallel using Logic Apps parallel branches. Use the `pages` parameter to specify page ranges:

```http
POST ...?pages=1-2
POST ...?pages=3-4
```

## 6A.14 Logic App parameters

```json
{
  "diEndpoint": "https://<resource>.cognitiveservices.azure.com",
  "apiVersion": "2024-11-30",
  "classifierModelId": "interfor-pod-classifier-v001",
  "extractionModelId": "interfor-pod-extraction-v001",
  "classificationConfidenceThreshold": 0.85,
  "criticalFieldConfidenceThreshold": 0.80
}
```

## 6A.15 Authentication — corrected

**Managed identity (recommended):**

```json
{
  "type": "ManagedServiceIdentity",
  "identity": "SystemAssigned",
  "audience": "https://cognitiveservices.azure.com"
}
```

**Key-based (fallback):** Store key in Key Vault. Reference via Logic App parameter or Key Vault connector. Never embed keys in workflow definitions. citeturn3search53

***

# Phase 6B — Developer integration via Microsoft Power Platform

## 6B.1 Architecture

```text
Power Automate cloud flow → HTTP / custom connector → Document Intelligence REST API
Dataverse → operational processing store
Power Apps → human review workbench
Approvals → lightweight exception handling
```

## 6B.2 AI Builder vs Studio model — decision

Use **Power Automate + HTTP/custom connector calling the Studio-trained model** when the model was built in Document Intelligence Studio.

Use **AI Builder Process documents action** only if the model is rebuilt natively inside Power Platform. Microsoft notes that since May 2025, the action formerly named *"Extract information from documents"* is named **Process documents**. citeturn4search96

## 6B.3 Custom connector — recommended for production

Create a custom connector wrapping the Document Intelligence REST endpoints. Microsoft states that custom connectors can be created from an OpenAPI definition. citeturn4search102turn4search103

Connector actions:

* Submit classification
* Get classification result
* Submit extraction
* Get extraction result

## 6B.4 Dataverse tables

Create the same tables described in the Power Platform response:

* `POD Processing Job`
* `POD Extracted Header`
* `POD Product Line`
* `POD Validation Issue`
* `POD Review Decision`

Dataverse connector supports creating, updating, retrieving, listing, and deleting rows. citeturn4search108turn4search112turn4search113

## 6B.5 🆕 Power Automate HTTP action size limits

**Correction from re-assessment:** Power Automate HTTP actions have payload size limits. For large documents, use `urlSource` with a SAS-protected blob URL instead of `base64Source` to avoid exceeding limits.

```json
{
  "urlSource": "https://stpodidpprod.blob.core.windows.net/inbound/document.pdf?<SAS-token>"
}
```

## 6B.6 Polling — corrected

Use `Retry-After` header value from the `202` response for dynamic delay intervals, not hardcoded delays.

## 6B.7 Review workbench

Use **Power Apps canvas or model-driven app** over Dataverse for human review.

Screens:

1. Review Queue
2. Document Detail (embedded document preview + extracted fields)
3. Extracted Fields Correction
4. Product Lines
5. Validation Issues
6. Submit Review Decision

Power Apps canvas apps can connect to Dataverse and be built as custom business applications. citeturn4search116turn4search118

## 6B.8 Approvals

Power Automate approvals support **Custom Responses** with wait-for-one or wait-for-all response types. citeturn4search70turn4search119

Response options:

* Accept extracted data
* Correct and approve in review app
* Reject document
* Request rescan

## 6B.9 🆕 Auto-learning feedback loop

**Optimisation:** When a reviewer corrects extracted fields, capture the corrections alongside the original extraction result. Automatically export corrected documents to the training dataset folder in blob storage. This creates a closed-loop improvement cycle for Phase 7 retraining.

***

# Phase 7 — Continuous improvement loop

## 7.1 Improvement workflow

1. Capture failed or reviewed documents from production.
2. Categorise the failure:
   * OCR issue
   * Missing field
   * Table extraction issue
   * Wrong class
   * Signature/date issue
   * Handwritten exception issue
   * New document variant
3. Add representative failed documents to Studio training folder.
4. Relabel fields or classes.
5. Retrain with a new model ID:

```text
interfor-pod-extraction-v002
interfor-pod-classifier-v002
```

6. Re-run the same holdout test set.
7. Compare field accuracy, confidence, table correctness, exception detection, and regression against previously good samples.
8. Promote only when the new model performs better.

## 7.2 Incremental classifier training

For classifiers, Microsoft supports incremental training: *"You can add new samples to existing classes or add new classes by referencing an existing classifier."* citeturn6search150

## 7.3 🆕 Composed model evaluation

If accuracy varies significantly across document quality tiers after retraining, evaluate whether separate models composed into a single model ID would improve results. citeturn6search133

## 7.4 🆕 Training budget planning

Neural model training is billed by the hour on S0. The first 10 hours per month are free. citeturn6search164 Plan for training costs during iterative improvement sprints.

## 7.5 Deliverables

* Model improvement backlog
* Retraining dataset
* Version comparison report
* Promotion decision record
* Training cost tracking

***

# Canonical JSON contract — corrected and complete

```json
{
  "correlationId": "string",
  "documentContentHash": "string",
  "source": {
    "fileName": "string",
    "blobPath": "string",
    "ingestedAt": "datetime"
  },
  "model": {
    "apiVersion": "2024-11-30",
    "classifierModelId": "string | null",
    "extractionModelId": "string"
  },
  "classification": {
    "docType": "DeliverySlip_POD",
    "confidence": 0.97,
    "pageRanges": [
      {
        "docType": "DeliverySlip_POD",
        "pages": [1, 2],
        "confidence": 0.97
      }
    ]
  },
  "documentIdentity": {
    "documentType": "DeliverySlip",
    "sourceCompany": "Interfor",
    "printDateTime": "string",
    "pageNumber": "string"
  },
  "order": {
    "soldToName": "string",
    "soldToAddress": "string",
    "customerNumber": "string",
    "orderNumber": "string",
    "orderDate": "date",
    "salesperson": "string"
  },
  "shipping": {
    "shipToName": "string",
    "shipToAddress": "string",
    "shipToPhone": "string",
    "carrierName": "string",
    "carrierPhone": "string",
    "route": "string | null",
    "shipmentNumber": "string",
    "shipmentDate": "date | null",
    "branchLocation": "string",
    "unit": "string",
    "unitNumber": "string",
    "restrictions": "string"
  },
  "notes": "string | null",
  "products": [
    {
      "productDescription": "string",
      "productCode": "string",
      "length": "string",
      "units": 0,
      "pieces": 0,
      "pcsPerUnit": 0,
      "quantity": 0.0,
      "uom": "MBF"
    }
  ],
  "totals": {
    "subTotalUnits": 0,
    "subTotalPieces": 0,
    "subTotalQuantity": "string",
    "totalUnits": 0,
    "totalPieces": 0,
    "totalQuantity": "string",
    "loadWeight": "string",
    "loadWeightType": "Actual | Estimated"
  },
  "pod": {
    "shipperSignaturePresent": true,
    "shipperSignatureDate": "string",
    "carrierSignaturePresent": true,
    "carrierSignatureDate": "string",
    "customerDeliverySignaturePresent": true,
    "customerDeliverySignatureDate": "string",
    "receivedByName": "string",
    "handwrittenNoteText": "string",
    "handwrittenAnnotations": "string",
    "exceptionReferenceNumber": "string"
  },
  "validation": {
    "requiresReview": true,
    "reviewReasons": [
      "Exception note detected: wrong size"
    ],
    "isDuplicate": false,
    "isPriority": false
  }
}
```

***

# Complete validation rule catalogue — corrected

| Rule ID | Rule                            | Trigger condition                                                                             | Severity                 |
| ------- | ------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------ |
| V001    | Missing Order Number            | `OrderNumber` is empty                                                                        | Critical                 |
| V002    | Missing Shipment Number         | `ShipmentNumber` is empty                                                                     | Critical                 |
| V003    | Missing Ship To Name            | `ShipToName` is empty                                                                         | Critical                 |
| V004    | Missing Product Lines           | `ProductLines` is empty                                                                       | Critical                 |
| V005    | Missing Total Quantity          | `TotalQuantity` is empty                                                                      | Critical                 |
| V006    | Low confidence critical field   | Confidence < threshold on OrderNumber, ShipmentNumber, TotalQuantity                          | High                     |
| V007    | Missing customer signature      | `CustomerDeliverySignature` is empty                                                          | High                     |
| V008    | Missing customer signature date | `CustomerDeliverySignatureDate` is empty                                                      | Medium                   |
| V009    | Handwritten exception note      | `HandwrittenNoteText` is not empty                                                            | High                     |
| V010    | Exception keyword detected      | `HandwrittenNoteText` contains wrong size, shortage, damage, refused, missing, overage, claim | Critical                 |
| V011    | Unknown handwritten annotation  | `HandwrittenAnnotations` is not empty and does not match known codes                          | Medium                   |
| V012    | Product line pieces mismatch    | Units × PcsPerUnit ≠ Pieces                                                                   | High                     |
| V013    | Total quantity mismatch         | Sum of line quantities ≠ TotalQuantity                                                        | High                     |
| V014    | Zero actual load weight         | LoadWeight = 0.000 and LoadWeightType = Actual                                                | Medium                   |
| V015    | Priority shipment               | Notes contains JIT or PRIORITY                                                                | Info (tag, don't review) |
| V016    | Duplicate document              | Content hash matches previous                                                                 | High                     |
| V017    | Missing Shipment Date           | ShipmentDate is empty (expected on some variants)                                             | Low                      |

***

# Deployment and governance checklist — consolidated

## Developer checklist

* [ ] Azure Document Intelligence resource provisioned (S0).
* [ ] Storage account and containers created.
* [ ] CORS configured.
* [ ] RBAC assigned.
* [ ] Managed identity enabled on Logic App / Power Automate.
* [ ] OAuth audience set to `https://cognitiveservices.azure.com`.
* [ ] Event Grid trigger configured (or blob trigger as fallback).
* [ ] Input validation implemented (size, format, dimensions, duplicates).
* [ ] Classifier HTTP action implemented (if classifier is used).
* [ ] `split=auto` set in classifier URI (if multi-document packets).
* [ ] Extraction HTTP action implemented (base64 or binary stream).
* [ ] `Retry-After` header used in polling loops.
* [ ] Parse JSON schemas generated from real Studio test outputs.
* [ ] Canonical payload mapping completed (all fields from corrected taxonomy).
* [ ] Signature fields created as Signature type with draw regions.
* [ ] Product lines mapped from table/array extraction.
* [ ] All validation rules implemented.
* [ ] Duplicate detection implemented.
* [ ] Priority tagging implemented.
* [ ] `/results`, `/review`, `/archive`, `/error` paths implemented.
* [ ] Searchable PDF archival implemented (optional).
* [ ] Rate limiting and retry-with-backoff implemented.
* [ ] Correlation ID and model version captured in audit trail.
* [ ] Batch API path implemented for bulk processing (optional).
* [ ] Custom connector created for production reusability.
* [ ] Auto-learning feedback loop implemented (corrections → training data).

## Business analyst checklist

* [ ] Field dictionary locked and approved (corrected taxonomy).
* [ ] Required vs optional fields approved.
* [ ] Exception keyword list approved.
* [ ] Review reasons approved.
* [ ] Straight-through criteria approved.
* [ ] Priority tagging criteria approved.
* [ ] Human review workflow approved.
* [ ] Review app screens approved.
* [ ] Test cases signed off.

## Solution specialist checklist

* [ ] All four sample documents analysed (2 PDFs + 2 images).
* [ ] Read/Layout/queryFields/custom neural comparison completed.
* [ ] Custom neural extraction model ID confirmed.
* [ ] Classifier need confirmed or deferred.
* [ ] Composed model need evaluated.
* [ ] API version locked at 2024-11-30.
* [ ] `split` parameter (not `splitMode`) used correctly.
* [ ] Signature field type = Signature with draw regions confirmed.
* [ ] Overlapping fields evaluated.
* [ ] Training data limits verified.
* [ ] Service rate limits documented.
* [ ] Training cost budgeted.
* [ ] Security pattern approved (managed identity preferred).
* [ ] Deployment pattern approved (parameters per environment).
* [ ] Monitoring, retry, and throttling approach approved.

***

# Accuracy checks — final

| Claim                                                                                                   | Source                                                                                                                                               | Status                         |
| ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| Custom neural models support signature detection, table cell confidence, and overlapping fields in v4.0 | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)                | ✅ Confirmed citeturn6search142 |
| Signature field type = Signature, draw region, one region per field                                     | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)                | ✅ Confirmed citeturn6search142 |
| Custom classifier default `split` is `none`; set to `auto` for multi-document splitting                 | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)            | ✅ Confirmed citeturn6search150 |
| Custom classifier supports incremental training in v4.0                                                 | <https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/whats-new>                                                                | ✅ Confirmed citeturn6search139 |
| Start with neural model to determine if it meets functional needs                                       | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)                 | ✅ Confirmed citeturn6search143 |
| 5 documents minimum for extraction; 10–15 for lower-quality forms                                       | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0) | ✅ Confirmed citeturn6search153 |
| Classifier requires ≥2 classes and ≥5 samples per class                                                 | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)            | ✅ Confirmed citeturn6search150 |
| v4.0 composed models use explicit classifier instead of implicit                                        | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/composed-models?view=doc-intel-4.0.0)              | ✅ Confirmed citeturn6search133 |
| Batch API supports up to 10,000 documents per request                                                   | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/batch-analysis?view=doc-intel-4.0.0)            | ✅ Confirmed citeturn6search144 |
| Query fields is an add-on feature available in v4.0                                                     | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/add-on-capabilities?view=doc-intel-4.0.0)        | ✅ Confirmed citeturn6search158 |
| Searchable PDF is a free add-on with Read model                                                         | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/add-on-capabilities?view=doc-intel-4.0.0)        | ✅ Confirmed citeturn6search158 |
| S0 rate limits: 15 analyze/sec, 50 get/sec, 5 model mgmt/sec                                            | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0)                     | ✅ Confirmed citeturn6search164 |
| Neural training: 10 free hrs/month, then pay by hour on S0                                              | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0)                     | ✅ Confirmed citeturn6search164 |
| Max doc size 500 MB (S0), 4 MB (F0)                                                                     | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0)                     | ✅ Confirmed citeturn6search164 |
| Neural training data max 1 GB, 50,000 pages                                                             | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0)                     | ✅ Confirmed citeturn6search164 |
| Classifier training data max 2 GB (S0), 25,000 pages                                                    | [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0)                     | ✅ Confirmed citeturn6search164 |

***

This consolidated version now incorporates all 4 error corrections, all 13 gap closures, all 3 inconsistency resolutions, and all 5 optimisation improvements identified in the re-assessment. The field taxonomy is locked, the validation rules are complete, and both the Logic Apps and Power Platform orchestration designs are corrected for parameter naming, polling behaviour, authentication, size limits, and production resilience.

Would you like me to drill into any specific phase further, or produce a downloadable artefact such as the OpenAPI definition for the custom connector or a Dataverse solution schema?
