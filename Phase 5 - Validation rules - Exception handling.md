Below is the deep technical drill-down into **Phase 5 — Validation rules and exception handling** for the Interfor POD / delivery-slip scenario. This phase sits between Document Intelligence extraction output and downstream systems (ERP, TMS, AP, receiving). It is the layer that turns raw extracted data into **trusted, business-grade structured data** suitable for automation.

***

# Phase 5 — Validation rules and exception handling (deep dive)

***

# 1. Understanding the Document Intelligence response structure

Before writing any validation rule, developers must understand exactly what the custom extraction model returns. Every rule references specific JSON paths in the response.

## 1.1 Top-level response structure

Microsoft states: *"The Analyze operation is an async API. Submitting a document returns an Operation-Location header that contains the URL to poll for completion. When an analysis request completes successfully, the response contains the elements described in the model data extraction."* citeturn10search1

The response contains these major sections:

| Section              | Description                                                                                      | Used for validation                            |
| -------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| `pages`              | Words, lines, spans, word-level confidence per page                                              | OCR quality validation, word confidence checks |
| `tables`             | Layout-detected tables (row/column structure)                                                    | Cross-reference with custom table fields       |
| `documents`          | Custom model extracted fields, tables, signatures                                                | All business field validation                  |
| `documents[].fields` | Key-value extracted fields with `content`, `confidence`, `valueType`, `boundingRegions`, `spans` | Field-level validation                         |
| `styles`             | Font style information including `isHandwritten`                                                 | Handwriting detection                          |

## 1.2 Field-level response structure

Each extracted field in the `documents[].fields` dictionary has this structure:

```json
{
  "OrderNumber": {
    "type": "string",
    "valueString": "2448058",
    "content": "2448058",
    "boundingRegions": [
      {
        "pageNumber": 1,
        "polygon": [x1, y1, x2, y2, x3, y3, x4, y4]
      }
    ],
    "confidence": 0.95,
    "spans": [
      { "offset": 234, "length": 7 }
    ]
  }
}
```

Microsoft states: *"Field confidence indicates an estimated probability between 0 and 1 that the prediction is correct. For example, a confidence value of 0.95 (95%) indicates that the prediction is likely correct 19 out of 20 times."* citeturn10search4

## 1.3 Signature field response structure

For Signature-type fields (v4.0), the value is `"signed"` or `"unsigned"`:

Microsoft states: *"The signature field value type uses 'signed' or 'unsigned' as the normalized representation."* citeturn10search1

```json
{
  "CustomerDeliverySignature": {
    "type": "signature",
    "valueSignature": "signed",
    "content": "{signature}",
    "confidence": 0.88,
    "boundingRegions": [...]
  }
}
```

## 1.4 Table field response structure

For custom table fields (e.g., `ProductLines`), the response contains an array of row objects, each with cell-level fields:

```json
{
  "ProductLines": {
    "type": "array",
    "valueArray": [
      {
        "type": "object",
        "valueObject": {
          "Length": { "type": "string", "valueString": "10'", "confidence": 0.92 },
          "Units": { "type": "number", "valueNumber": 16, "confidence": 0.94 },
          "Pieces": { "type": "number", "valueNumber": 3328, "confidence": 0.91 },
          "PcsPerUnit": { "type": "number", "valueNumber": 208, "confidence": 0.90 },
          "Quantity": { "type": "number", "valueNumber": 22.192, "confidence": 0.93 },
          "UOM": { "type": "string", "valueString": "MBF", "confidence": 0.96 }
        },
        "confidence": 0.89
      }
    ]
  }
}
```

Microsoft states: *"Confidence scores for tables, table rows, and table cells are available starting with the 2024-11-30 (GA) API version for custom models."* citeturn10search4

## 1.5 Word-level confidence (v4.0)

Microsoft states: *"Field level confidence includes word confidence scores with 2024-11-30 (GA) API version for custom models."* and *"The pages array contains an array of words and each word has an associated span and confidence score."* citeturn10search4

This means each word in the `pages[].words[]` array has its own confidence. Validation logic can cross-reference field spans with word-level confidence to identify which specific word within a multi-word field is unreliable.

***

# 2. Validation architecture

## 2.1 Three validation layers

The validation system should be designed as three distinct layers that execute sequentially:

```text
Layer 1: Structural validation
  → Is the response well-formed?
  → Did the extraction succeed?
  → Are required sections present?

Layer 2: Business field validation
  → Are required fields present and non-empty?
  → Are confidence scores above thresholds?
  → Are signatures detected?
  → Are exception indicators present?
  → Do quantities reconcile?

Layer 3: Cross-document / contextual validation
  → Is this a duplicate document?
  → Does the order number exist in the ERP/TMS?
  → Is the shipment already received?
  → Does the branch/location match expected routing?
```

## 2.2 Validation output model

Every validation check should produce a standardised result:

```json
{
  "ruleId": "V001",
  "ruleName": "Missing Order Number",
  "severity": "Critical",
  "category": "RequiredField",
  "passed": false,
  "fieldName": "OrderNumber",
  "extractedValue": null,
  "expectedCondition": "Non-empty string",
  "confidence": null,
  "message": "Order Number is missing from the extraction result",
  "requiresReview": true
}
```

***

# 3. Layer 1 — Structural validation rules

These rules verify that the Document Intelligence response is technically usable before any business logic is applied.

## Rule S001 — Extraction API status check

```text
Condition:
  analyzeResult.status != "succeeded"

Action:
  Route to error queue.
  Do NOT proceed with any further validation.
  Log API status and error details.

Severity: Critical
```

## Rule S002 — Document count check

```text
Condition:
  analyzeResult.documents is empty or null

Action:
  Route to error queue.
  Indicates the custom model did not recognise the document type.

Severity: Critical
```

## Rule S003 — Document type confidence check

Microsoft states: *"The document type confidence is an indicator of closely the analyzed document resembles documents in the training dataset. When the document type confidence is low, it's indicative of template or structural variations in the analyzed document."* citeturn10search4

```text
Condition:
  analyzeResult.documents[0].confidence < 0.60

Action:
  Route to human review.
  Reason: Document type confidence too low — document may not be a delivery slip.

Severity: High
```

## Rule S004 — Page count check

```text
Condition:
  analyzeResult.pages count < 1

Action:
  Route to error queue.
  Indicates the document could not be processed.

Severity: Critical
```

## Rule S005 — OCR quality check

Microsoft states: *"If the confidence score for the readResults object is low, improve the quality of your input documents."* citeturn10search4

```text
Condition:
  For each page in analyzeResult.pages:
    Calculate average word confidence across all words on the page.
    If average word confidence < 0.50:
      Flag page as poor OCR quality.

Action:
  If any page flagged: add review reason "Poor OCR quality on page {n}".
  Do not reject — proceed with extraction but flag for review.

Severity: Medium
```

This rule is relevant because the attached samples include both clean PDFs and mobile-photo captures. Image 2 (`63574059response.jpeg`) shows a document captured with a clip artefact and variable lighting, which could produce lower OCR confidence on some words.

***

# 4. Layer 2 — Business field validation rules

## 4.1 Required field validation

### Rule V001 — Missing Order Number

```text
JSON path:
  analyzeResult.documents[0].fields.OrderNumber.content

Condition:
  Value is null, empty, or whitespace-only

Action:
  requiresReview = true
  Add validation issue:
    ruleId: V001
    severity: Critical
    category: RequiredField
    message: "Order Number is missing"
```

### Rule V002 — Missing Shipment Number

```text
JSON path:
  analyzeResult.documents[0].fields.ShipmentNumber.content

Condition:
  Value is null, empty, or whitespace-only

Action:
  requiresReview = true
  Add validation issue:
    ruleId: V002
    severity: Critical
    category: RequiredField
    message: "Shipment Number is missing"
```

### Rule V003 — Missing Ship To Name

```text
JSON path:
  analyzeResult.documents[0].fields.ShipToName.content

Condition:
  Value is null, empty, or whitespace-only

Action:
  requiresReview = true
  severity: Critical
```

### Rule V004 — Missing Product Lines

```text
JSON path:
  analyzeResult.documents[0].fields.ProductLines.valueArray

Condition:
  Array is null, empty, or has zero elements

Action:
  requiresReview = true
  severity: Critical
  message: "No product lines extracted"
```

### Rule V005 — Missing Total Quantity

```text
JSON path:
  analyzeResult.documents[0].fields.TotalQuantity.content

Condition:
  Value is null, empty, or whitespace-only

Action:
  requiresReview = true
  severity: Critical
```

## 4.2 Confidence-based validation

Microsoft states: *"For scenarios where accuracy is critical, confidence can be used to determine whether to automatically accept the prediction or flag it for human review."* citeturn10search4

### Rule V006 — Critical field low confidence

```text
Fields to check:
  OrderNumber
  ShipmentNumber
  CustomerNumber
  TotalQuantity

Threshold: 0.85

Condition:
  For each field:
    If field.confidence < 0.85:
      requiresReview = true
      Add validation issue:
        ruleId: V006
        severity: High
        category: LowConfidence
        fieldName: {field name}
        confidence: {actual confidence}
        message: "Low confidence on {field name}: {confidence}"
```

### Rule V007 — Non-critical field low confidence

```text
Fields to check:
  BranchLocation
  CarrierName
  Restrictions
  Notes
  LoadWeight

Threshold: 0.70

Condition:
  For each field:
    If field.confidence < 0.70:
      Add validation issue:
        ruleId: V007
        severity: Medium
        category: LowConfidence
        fieldName: {field name}
        confidence: {actual confidence}
      Do NOT automatically require review — log for monitoring only.
```

### Rule V008 — Composite confidence check

Microsoft states: *"While evaluating confidence scores, you should also look at the underlying extraction confidence to generate a comprehensive confidence for the extracted result. Evaluate the OCR results for text extraction or selection marks depending on the field type to generate a composite confidence score for the field."* citeturn10search4

```text
For critical fields (OrderNumber, ShipmentNumber, TotalQuantity):

  Step 1: Get field confidence from documents[0].fields.{field}.confidence
  Step 2: Get field span from documents[0].fields.{field}.spans
  Step 3: Find matching words in pages[].words[] by span offset/length
  Step 4: Get word-level confidence for each matching word
  Step 5: Calculate composite confidence:
    compositeConfidence = min(fieldConfidence, min(wordConfidences))

  If compositeConfidence < 0.80:
    requiresReview = true
    Add validation issue:
      ruleId: V008
      severity: High
      category: CompositeConfidence
      message: "Composite confidence for {field}: {compositeConfidence}"
```

## 4.3 Signature validation

### Rule V009 — Missing customer delivery signature

```text
JSON path:
  analyzeResult.documents[0].fields.CustomerDeliverySignature.valueSignature

Condition:
  Value is null OR value == "unsigned"

Action:
  requiresReview = true
  Add validation issue:
    ruleId: V009
    severity: High
    category: PODEvidence
    message: "Customer delivery signature is missing or unsigned"
```

### Rule V010 — Missing customer signature date

```text
JSON path:
  analyzeResult.documents[0].fields.CustomerSignatureDate.content

Condition:
  Value is null, empty, or whitespace-only

Action:
  requiresReview = true
  severity: Medium
  message: "Customer signature date is missing"
```

### Rule V011 — Low confidence signature

```text
JSON path:
  analyzeResult.documents[0].fields.CustomerDeliverySignature.confidence

Condition:
  confidence < 0.65

Action:
  requiresReview = true
  severity: Medium
  message: "Customer signature confidence is low: {confidence}"
```

All three signature rules are relevant because the attached delivery slips include shipper, carrier, and customer signature areas with varying levels of completion. Image 1 has shipper + customer signatures but no carrier signature. Image 2 has all three. citeturn1search1turn1search2

### Rule V012 — Missing carrier or shipper signature (informational)

```text
Condition:
  ShipperSignature is "unsigned" or null
  OR CarrierSignature is "unsigned" or null

Action:
  Add validation issue (informational — not blocking):
    ruleId: V012
    severity: Low
    category: PODEvidence
    message: "Shipper/Carrier signature missing — may be acceptable depending on process"
```

## 4.4 Exception and handwriting detection

### Rule V013 — Handwritten exception note detected

```text
JSON path:
  analyzeResult.documents[0].fields.HandwrittenNoteText.content

Condition:
  Value is not null AND not empty

Action:
  requiresReview = true
  Add validation issue:
    ruleId: V013
    severity: High
    category: DeliveryException
    message: "Handwritten exception note detected: '{extracted text}'"
```

This is the most critical exception rule. `POD Scenario 3.PDF` contains "Wrong Size Lumber" as handwritten exception text. citeturn1search2

### Rule V014 — Exception keyword matching

```text
Exception keyword list:
  wrong size
  shortage
  damage
  damaged
  refused
  missing
  overage
  claim
  reject
  broken
  wet
  mold
  mould
  incorrect
  not ordered

Condition:
  toLowerCase(HandwrittenNoteText) contains any keyword from the list

Action:
  requiresReview = true
  Add validation issue:
    ruleId: V014
    severity: Critical
    category: DeliveryException
    exceptionType: classify_keyword(matched keyword)
    message: "Delivery exception keyword detected: '{matched keyword}' in note: '{full text}'"
```

### Exception type classification logic

```text
function classify_keyword(keyword):
  if keyword in ["wrong size", "incorrect", "not ordered"]:
    return "WrongProduct"
  if keyword in ["shortage", "missing"]:
    return "Shortage"
  if keyword in ["damage", "damaged", "broken", "wet", "mold", "mould"]:
    return "Damage"
  if keyword in ["refused", "reject"]:
    return "Refused"
  if keyword in ["overage"]:
    return "Overage"
  if keyword in ["claim"]:
    return "Claim"
  return "Unknown"
```

### Rule V015 — Unknown handwritten annotations

```text
JSON path:
  analyzeResult.documents[0].fields.HandwrittenAnnotations.content

Condition:
  Value is not null AND not empty
  AND value does not match known annotation codes

Known codes:
  DSU  (delivery status update)
  CPU  (customer pickup)
  COD  (cash on delivery)
  TBD  (to be determined)

Action:
  If known code: tag with annotation type, do NOT require review
  If unknown code: requiresReview = true
    severity: Medium
    message: "Unknown handwritten annotation: '{value}'"
```

Image 2 (`63574059response.jpeg`) contains "DSU" marks, which is an example of a known annotation code.

### Rule V016 — Handwriting style detection

Microsoft states: *"A style element describes the font style to apply to text content… Currently, the only detected font style is whether the text is handwritten."* citeturn10search1

```text
JSON path:
  analyzeResult.styles[]

Condition:
  Any style where isHandwritten == true
  AND the handwritten span overlaps with a business field that is NOT
    a signature, HandwrittenNoteText, or HandwrittenAnnotations field

Action:
  Add validation issue:
    ruleId: V016
    severity: Medium
    category: HandwritingDetection
    message: "Handwritten content detected in field '{field name}' — may indicate manual override or correction"
```

This rule catches cases where someone has crossed out a printed value and written a correction by hand.

## 4.5 Quantity reconciliation

### Rule V017 — Product line pieces mismatch

```text
For each row in ProductLines.valueArray:
  expectedPieces = row.Units.valueNumber * row.PcsPerUnit.valueNumber

  If row.Units and row.PcsPerUnit are both present and non-null:
    If abs(expectedPieces - row.Pieces.valueNumber) > tolerance:
      requiresReview = true
      Add validation issue:
        ruleId: V017
        severity: High
        category: QuantityMismatch
        message: "Product line {n}: Units({Units}) × PcsPerUnit({PcsPerUnit}) = {expected} ≠ Pieces({actual})"
```

Set tolerance to 0 for exact integer math, or a small tolerance if floating-point rounding may apply.

### Rule V018 — Total quantity mismatch

```text
sumQuantity = sum of all ProductLines[].Quantity.valueNumber

Parse TotalQuantity to extract numeric value:
  e.g., "23.578 MBF" → 23.578

If abs(sumQuantity - parsedTotalQuantity) > tolerance:
  requiresReview = true
  Add validation issue:
    ruleId: V018
    severity: High
    category: QuantityMismatch
    message: "Sum of line quantities ({sumQuantity}) ≠ Total Quantity ({parsedTotalQuantity})"
```

This rule is directly applicable: Image 1 shows a total of `23.578 MBF` across two product lines (`22.192 + 1.386 = 23.578`). Image 2 shows sub-totals of `13.328 MBF` and `23.324 MBF` across two product groups, with a grand total of `36.652 MBF` (`13.328 + 23.324 = 36.652`). citeturn1search1turn1search2

### Rule V019 — Total units/pieces mismatch

```text
sumUnits = sum of all ProductLines[].Units.valueNumber
sumPieces = sum of all ProductLines[].Pieces.valueNumber

If sumUnits != TotalUnits.valueNumber:
  Add validation issue:
    ruleId: V019
    severity: High
    message: "Sum of line units ({sumUnits}) ≠ Total Units ({TotalUnits})"

If sumPieces != TotalPieces.valueNumber:
  Add validation issue:
    ruleId: V019
    severity: High
    message: "Sum of line pieces ({sumPieces}) ≠ Total Pieces ({TotalPieces})"
```

## 4.6 Table confidence validation

Microsoft states: *"The recommended approach is to review the accuracy in a top-down manner starting with the table first, followed by the row and then the cell."* citeturn10search4

### Rule V020 — Table-level confidence

```text
JSON path:
  analyzeResult.documents[0].fields.ProductLines.confidence

Condition:
  confidence < 0.70

Action:
  requiresReview = true
  severity: High
  message: "Product table overall confidence is low: {confidence}"
```

### Rule V021 — Row-level confidence

```text
For each row in ProductLines.valueArray:
  If row.confidence < 0.60:
    requiresReview = true
    Add validation issue:
      ruleId: V021
      severity: High
      message: "Product line {n} row confidence is low: {confidence}"
```

### Rule V022 — Cell-level confidence on critical columns

```text
For each row in ProductLines.valueArray:
  For each critical column in [Quantity, Pieces]:
    If column.confidence < 0.50:
      requiresReview = true
      Add validation issue:
        ruleId: V022
        severity: High
        message: "Product line {n}, column {column}: cell confidence is low: {confidence}"
```

## 4.7 Load weight validation

### Rule V023 — Zero actual load weight

```text
Condition:
  LoadWeightType.content == "Actual"
  AND (LoadWeight parsed value == 0 OR LoadWeight.content == "0.000 LB")

Action:
  Add validation issue:
    ruleId: V023
    severity: Medium
    category: DataAnomaly
    message: "Actual Load Weight is zero — may indicate weighing was skipped"
```

Image 1 shows `Actual Load Weight: 0.000 LB`, while Image 2 shows `Estimated Load Weight: 61,392 LB`. This rule catches the anomaly in Image 1.

### Rule V024 — Missing load weight type

```text
Condition:
  LoadWeightType is null or empty

Action:
  Add validation issue:
    ruleId: V024
    severity: Low
    message: "Load weight type (Actual/Estimated) is missing"
```

## 4.8 Notes and priority detection

### Rule V025 — Priority shipment detection

```text
JSON path:
  analyzeResult.documents[0].fields.Notes.content

Condition:
  toLowerCase(Notes) contains "jit" OR "priority"

Action:
  Set isPriority = true
  Add validation issue (informational):
    ruleId: V025
    severity: Info
    category: Priority
    message: "Priority shipment detected: '{Notes content}'"
  Tag the canonical payload with isPriority = true for downstream SLA handling
```

Image 2 contains `JIT CONTRACT ORDER - PRIORITY SHIPMENT` in the Notes section.

## 4.9 Duplicate detection

### Rule V026 — Content hash duplicate

```text
Pre-condition:
  Before calling Document Intelligence, compute SHA-256 hash of the document binary content.

Condition:
  Hash matches a previously processed document hash in the processing store (Dataverse or database).

Action:
  requiresReview = true
  isDuplicate = true
  Add validation issue:
    ruleId: V026
    severity: High
    category: Duplicate
    message: "Document content hash matches previously processed document {previous job ID}"
```

### Rule V027 — Business key duplicate

```text
Condition:
  OrderNumber + ShipmentNumber combination already exists in the processing store
  AND previous processing status is "Posted" or "Review Approved"

Action:
  requiresReview = true
  isDuplicate = true
  Add validation issue:
    ruleId: V027
    severity: High
    category: Duplicate
    message: "Order {OrderNumber} / Shipment {ShipmentNumber} already processed as job {previous job ID}"
```

***

# 5. Layer 3 — Cross-document / contextual validation

These rules require access to external systems.

### Rule V028 — Order exists in ERP

```text
Condition:
  Call ERP API or query Dataverse to verify OrderNumber exists.
  If not found:
    Add validation issue:
      ruleId: V028
      severity: High
      category: ERPValidation
      message: "Order Number {OrderNumber} not found in ERP"
```

### Rule V029 — Shipment already received

```text
Condition:
  Call TMS/receiving API to check ShipmentNumber status.
  If status = "Already Received":
    Add validation issue:
      ruleId: V029
      severity: High
      category: TMSValidation
      message: "Shipment {ShipmentNumber} already marked as received in TMS"
```

### Rule V030 — Branch/location mismatch

```text
Condition:
  BranchLocation extracted value does not match expected routing for the carrier or customer.

Action:
  Add validation issue:
    ruleId: V030
    severity: Medium
    category: RoutingValidation
    message: "Branch/Location '{BranchLocation}' does not match expected routing"
```

***

# 6. Complete validation rule catalogue

| Rule ID  | Name                              | Category             | Severity | Requires review? | Trigger                        |
| -------- | --------------------------------- | -------------------- | -------- | ---------------- | ------------------------------ |
| **S001** | Extraction API failed             | Structural           | Critical | Error queue      | status ≠ succeeded             |
| **S002** | No documents detected             | Structural           | Critical | Error queue      | documents empty                |
| **S003** | Low document type confidence      | Structural           | High     | Yes              | docType confidence < 0.60      |
| **S004** | No pages                          | Structural           | Critical | Error queue      | pages empty                    |
| **S005** | Poor OCR quality                  | Structural           | Medium   | Yes              | Average word confidence < 0.50 |
| **V001** | Missing Order Number              | RequiredField        | Critical | Yes              | Empty/null                     |
| **V002** | Missing Shipment Number           | RequiredField        | Critical | Yes              | Empty/null                     |
| **V003** | Missing Ship To Name              | RequiredField        | Critical | Yes              | Empty/null                     |
| **V004** | Missing Product Lines             | RequiredField        | Critical | Yes              | Empty array                    |
| **V005** | Missing Total Quantity            | RequiredField        | Critical | Yes              | Empty/null                     |
| **V006** | Critical field low confidence     | LowConfidence        | High     | Yes              | Confidence < 0.85              |
| **V007** | Non-critical field low confidence | LowConfidence        | Medium   | No (log only)    | Confidence < 0.70              |
| **V008** | Composite confidence failure      | CompositeConfidence  | High     | Yes              | Composite < 0.80               |
| **V009** | Missing customer signature        | PODEvidence          | High     | Yes              | unsigned or null               |
| **V010** | Missing customer signature date   | PODEvidence          | Medium   | Yes              | Empty/null                     |
| **V011** | Low signature confidence          | PODEvidence          | Medium   | Yes              | Confidence < 0.65              |
| **V012** | Missing shipper/carrier signature | PODEvidence          | Low      | No (log only)    | unsigned or null               |
| **V013** | Handwritten exception note        | DeliveryException    | High     | Yes              | Non-empty text                 |
| **V014** | Exception keyword match           | DeliveryException    | Critical | Yes              | Keyword detected               |
| **V015** | Unknown handwritten annotation    | HandwritingDetection | Medium   | Yes              | Unknown code                   |
| **V016** | Handwriting in business field     | HandwritingDetection | Medium   | Yes              | isHandwritten overlaps field   |
| **V017** | Product line pieces mismatch      | QuantityMismatch     | High     | Yes              | Units × PcsPerUnit ≠ Pieces    |
| **V018** | Total quantity mismatch           | QuantityMismatch     | High     | Yes              | Sum ≠ Total                    |
| **V019** | Total units/pieces mismatch       | QuantityMismatch     | High     | Yes              | Sum ≠ Total                    |
| **V020** | Low table confidence              | TableConfidence      | High     | Yes              | Table confidence < 0.70        |
| **V021** | Low row confidence                | TableConfidence      | High     | Yes              | Row confidence < 0.60          |
| **V022** | Low cell confidence (critical)    | TableConfidence      | High     | Yes              | Cell confidence < 0.50         |
| **V023** | Zero actual load weight           | DataAnomaly          | Medium   | Yes              | 0.000 LB + Actual              |
| **V024** | Missing load weight type          | DataAnomaly          | Low      | No (log only)    | Empty/null                     |
| **V025** | Priority shipment                 | Priority             | Info     | No (tag only)    | JIT/PRIORITY keyword           |
| **V026** | Content hash duplicate            | Duplicate            | High     | Yes              | Hash match                     |
| **V027** | Business key duplicate            | Duplicate            | High     | Yes              | Order+Shipment match           |
| **V028** | Order not in ERP                  | ERPValidation        | High     | Yes              | ERP lookup fails               |
| **V029** | Shipment already received         | TMSValidation        | High     | Yes              | TMS status = received          |
| **V030** | Branch routing mismatch           | RoutingValidation    | Medium   | Yes              | Unexpected branch              |

***

# 7. Exception handling workflow design

## 7.1 Review routing matrix

After all validation rules execute, the accumulated `reviewReasons` array determines routing:

```text
If any Critical severity issue exists:
  → Route to Senior Review queue
  → Do NOT auto-post to downstream

If any High severity issue exists AND no Critical:
  → Route to Standard Review queue
  → Do NOT auto-post to downstream

If only Medium severity issues exist:
  → Route to extraction with review flag
  → Auto-post to downstream with "Pending Review" status

If only Low/Info severity issues exist:
  → Auto-post to downstream
  → Log issues for monitoring

If no issues:
  → Auto-post to downstream (straight-through processing)
```

## 7.2 Review queue priority

```text
Priority 1 (highest):
  V014 — Exception keyword match (claims, damage, shortage)
  V026/V027 — Duplicate detection
  V028 — Order not in ERP

Priority 2:
  V001–V005 — Missing required fields
  V009 — Missing customer signature
  V017–V019 — Quantity mismatches

Priority 3:
  V006/V008 — Low confidence on critical fields
  V013 — Handwritten exception note (without keyword match)
  V020–V022 — Low table confidence

Priority 4:
  V015/V016 — Handwriting detection
  V023 — Zero load weight
  S005 — Poor OCR quality
```

## 7.3 Human review interface requirements

The review app (Power Apps or custom web app) should present:

### Left panel — Source document

```text
Embedded document viewer (PDF/image)
Zoom and pan
Page navigation
Highlight bounding regions of flagged fields
```

### Right panel — Extracted data

```text
Section 1: Validation summary
  - Count of issues by severity
  - List of all validation issues with rule ID, message, confidence

Section 2: Extracted header fields
  - Each field with extracted value, confidence, and edit capability
  - Flagged fields highlighted in red/yellow

Section 3: Product lines table
  - Each row with cell values, row confidence, cell confidence
  - Quantity reconciliation status per row
  - Editable cells for corrections

Section 4: Signatures
  - Signature detection status (signed/unsigned)
  - Confidence score
  - Bounding region reference

Section 5: Exception notes
  - Extracted handwritten text
  - Exception type classification
  - Reviewer can confirm, edit, or dismiss

Section 6: Reviewer actions
  - Approve as extracted
  - Approve with corrections
  - Reject document
  - Request rescan
  - Add reviewer comment
```

## 7.4 Correction capture for model improvement

Microsoft confirms that Document Intelligence does not currently have a built-in API for retraining based on user feedback. Microsoft Q\&A states: *"Azure Document Intelligence (ADI) does not currently have a built-in API for retraining the model based on user feedback. However, there are some workarounds that you can use to achieve a feedback mechanism… You can use the feedback data to identify areas where the model is making errors, and then update the model to improve its accuracy."* citeturn10search13

For each reviewer correction:

```json
{
  "jobId": "string",
  "originalExtractionModelId": "interfor-pod-extraction-v001",
  "correctedFields": {
    "OrderNumber": {
      "originalValue": "244805B",
      "correctedValue": "2448058",
      "correctionType": "OCR_error"
    }
  },
  "correctedProductLines": [],
  "reviewerAction": "ApproveWithCorrections",
  "reviewerComment": "OCR misread last digit as B instead of 8",
  "reviewedAt": "datetime",
  "reviewedBy": "reviewer ID"
}
```

Corrections should be:

1. Stored in Dataverse or a corrections database.
2. Periodically exported to the training dataset.
3. Used to create corrected `.labels.json` files for retraining in Studio.
4. Fed into Phase 7 (continuous improvement loop).

***

# 8. Implementation in Logic Apps

## 8.1 Validation scope structure

```text
Scope: Validation
  │
  ├── Initialize variables
  │     requiresReview = false
  │     reviewReasons = []
  │     validationIssues = []
  │     isPriority = false
  │     isDuplicate = false
  │
  ├── Structural validation
  │     S001: Check extraction status
  │     S002: Check documents array
  │     S003: Check document type confidence
  │     S004: Check pages array
  │     S005: Calculate OCR quality
  │
  ├── Required field validation
  │     V001–V005: Check each required field
  │
  ├── Confidence validation
  │     V006–V008: Check field and composite confidence
  │
  ├── Signature validation
  │     V009–V012: Check signatures
  │
  ├── Exception validation
  │     V013–V016: Check handwritten content
  │
  ├── Quantity validation
  │     V017–V019: Check product line math
  │
  ├── Table confidence validation
  │     V020–V022: Check table/row/cell confidence
  │
  ├── Load weight and priority
  │     V023–V025: Check anomalies and priority
  │
  ├── Duplicate detection
  │     V026–V027: Check hash and business key
  │
  └── Aggregate results
        Compose final validation summary
        Set requiresReview based on highest severity
        Set routing decision
```

## 8.2 Example Logic Apps expression for V001

```text
Condition: Missing Order Number

Expression:
  @empty(
    body('Parse_JSON_-_Extraction_Result')?['analyzeResult']?['documents']?[0]?['fields']?['OrderNumber']?['content']
  )

If true:
  Set variable: requiresReview = true
  Append to array: reviewReasons
    Value: "V001: Missing Order Number"
  Append to array: validationIssues
    Value: {
      "ruleId": "V001",
      "severity": "Critical",
      "category": "RequiredField",
      "fieldName": "OrderNumber",
      "message": "Order Number is missing"
    }
```

## 8.3 Example Logic Apps expression for V014

```text
Condition: Exception keyword in handwritten note

Expression:
  @contains(
    toLower(
      body('Parse_JSON_-_Extraction_Result')?['analyzeResult']?['documents']?[0]?['fields']?['HandwrittenNoteText']?['content']
    ),
    'wrong size'
  )

If true:
  Set variable: requiresReview = true
  Append to array: reviewReasons
    Value: "V014: Exception keyword 'wrong size' detected"
  Append to array: validationIssues
    Value: {
      "ruleId": "V014",
      "severity": "Critical",
      "category": "DeliveryException",
      "exceptionType": "WrongProduct",
      "message": "Delivery exception: 'wrong size' in note: 'Wrong Size Lumber'"
    }
```

## 8.4 Example Logic Apps expression for V018

```text
Quantity reconciliation requires Apply to each:

Apply to each ProductLines row:
  Compose lineQuantity = row.Quantity.valueNumber
  Append lineQuantity to sumArray

After loop:
  Compose sumQuantity = sum of sumArray values

Parse TotalQuantity to extract numeric value

Condition:
  abs(sumQuantity - parsedTotalQuantity) > 0.01

If true:
  requiresReview = true
  Append validation issue V018
```

***

# 9. Implementation in Power Automate

The same logic applies. Key differences:

* Use **Compose** and **Append to array variable** actions for building the validation issue list.
* Use **Condition** actions for each rule.
* Use **Apply to each** for product line iteration.
* Store validation issues as Dataverse rows in the `POD Validation Issue` table.
* Use **Approvals** or **Power Apps** for the review interface.

The Dataverse connector supports creating rows for each validation issue. citeturn4search108turn4search112turn4search113

***

# 10. Monitoring and analytics

## 10.1 Recommended metrics

Track these metrics over time to measure model and process quality:

| Metric                           | Source                                                | Purpose                               |
| -------------------------------- | ----------------------------------------------------- | ------------------------------------- |
| Straight-through processing rate | % of documents with no validation issues              | Overall automation effectiveness      |
| Review rate by rule              | Count of documents flagged per rule ID                | Identify most common failure patterns |
| Average field confidence         | Mean confidence per field across all documents        | Model quality trend                   |
| Exception rate                   | % of documents with V013/V014 issues                  | Business exception volume             |
| Duplicate rate                   | % of documents flagged as V026/V027                   | Upstream process quality              |
| Reviewer correction rate         | % of reviewed documents with field corrections        | Model improvement opportunity         |
| Time to review                   | Average time between review assignment and completion | Operational efficiency                |
| Post-review accuracy             | % of reviewer corrections that change a field value   | Human review value                    |

## 10.2 Alert thresholds

```text
If straight-through rate drops below 70% → Alert model team
If any single rule triggers on > 30% of documents → Alert model team
If average OrderNumber confidence drops below 0.85 → Alert model team
If duplicate rate exceeds 10% → Alert operations team
If review backlog exceeds 50 documents → Alert review team
```

***

# 11. Complete Phase 5 deliverables

| #  | Deliverable                             | Owner                            | Description                                               |
| -- | --------------------------------------- | -------------------------------- | --------------------------------------------------------- |
| 1  | Validation rule catalogue               | BA + solution specialist         | All 30 rules with conditions, thresholds, and actions     |
| 2  | Exception keyword list                  | BA                               | Maintained list of delivery exception keywords            |
| 3  | Known annotation code list              | BA + operations                  | DSU, CPU, COD, etc.                                       |
| 4  | Confidence threshold matrix             | BA + solution specialist         | Per-field thresholds for critical and non-critical fields |
| 5  | Review routing matrix                   | BA + solution specialist         | Severity → queue → priority mapping                       |
| 6  | Review app requirements                 | BA + UX                          | Screens, fields, correction capture, reviewer actions     |
| 7  | Validation issue data model             | Developer                        | JSON schema and Dataverse table design                    |
| 8  | Correction capture schema               | Developer                        | Reviewer corrections for model improvement                |
| 9  | Logic Apps / Power Automate expressions | Developer                        | Implementation-ready expressions for each rule            |
| 10 | Monitoring dashboard requirements       | Operations + solution specialist | Metrics, thresholds, alerts                               |
| 11 | Exception type classification logic     | BA + developer                   | Keyword → exception type mapping                          |
| 12 | Cross-document validation API contracts | Developer                        | ERP, TMS, duplicate check interfaces                      |
| 13 | Test evidence                           | QA                               | Pass/fail results for each rule against test documents    |

***

# Accuracy checks — Phase 5 specific

| Claim                                                                             | Source                                                                                                                                                        | Status                         |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| Confidence is 0–1 probability that prediction is correct                          | [Accuracy and confidence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)         | ✅ Confirmed citeturn10search4  |
| Confidence can be used to accept or flag for human review                         | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Field-level confidence includes word confidence in v4.0                           | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Table, row, and cell confidence available in v4.0                                 | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Top-down table evaluation: table → row → cell                                     | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Composite confidence: evaluate OCR + field + word confidence together             | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| readResults low confidence → improve input quality                                | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| pageResults low confidence → ensure same document type                            | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Document type confidence indicates resemblance to training data                   | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Custom neural models do NOT provide accuracy scores during training               | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search4  |
| Signature field returns "signed" or "unsigned"                                    | [Analyze document response](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0) | ✅ Confirmed citeturn10search1  |
| `isHandwritten` style detection available                                         | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search1  |
| Analyze response contains pages, tables, documents, styles, paragraphs            | Same source                                                                                                                                                   | ✅ Confirmed citeturn10search1  |
| No built-in API for retraining from user feedback                                 | [Microsoft Q\&A](https://learn.microsoft.com/en-us/answers/questions/1703062/azure-document-intelligence-feedback-mechanism-wit)                              | ✅ Confirmed citeturn10search13 |
| Merged cells have lower confidence; missing cells have NULL with lower confidence | [Accuracy and confidence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)         | ✅ Confirmed citeturn10search4  |

***

This completes the Phase 5 deep dive. The validation layer is now fully specified with 30 rules across three layers (structural, business field, and cross-document), complete with JSON paths, Logic Apps expressions, exception classification logic, review routing, correction capture, and monitoring metrics.

Would you like me to proceed to Phase 7 (continuous improvement loop deep dive), produce the OpenAPI definition for the custom connector, or create a downloadable artefact?
