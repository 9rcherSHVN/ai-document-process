Absolutely — this is the **critical implementation step**, and it’s where teams either build a robust document-processing platform or end up with a fragile OCR demo.

I’ll explain this in two layers:

1. **What Azure AI Document Intelligence actually provides and how developers should customise it around a business-owned schema**
2. **How developers should implement factual confidence scoring that is grounded in what was actually extracted, not invented by an LLM**

I’ll keep the source-backed product behaviour separate from my engineering recommendations so you can see exactly what is documented versus what I’m recommending as the solution design.

***

# 1) First principle: the business schema must drive the model design

The business document owner’s schema is the **contract** that defines:

* which document type this is,
* which fields matter,
* which fields are required,
* which fields repeat in tables,
* and which fields are only review-time metadata.

In Azure AI Document Intelligence, **custom extraction models** are trained by labelling the values you want extracted from your business documents, and **custom classification models** can be paired with extraction models to identify the document type before extraction. Microsoft explicitly documents both patterns. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

That means the schema should not be treated as a nice-to-have document for downstream integration only.  
The schema should be used to drive **all** of the following:

* training data selection,
* field labelling,
* model selection,
* runtime mapping,
* confidence routing,
* review UX,
* and validation rules.

***

# 2) What developers should build from the owner-provided schema

## 2.1 Convert the business schema into three technical artefacts

When the document owner provides the schema, developers should convert it into:

### A. **Document type definition**

Example:

* DeliverySlip
* Invoice
* POD
* PackingList

This matters because Document Intelligence supports pairing a **custom classification model** with a **custom extraction model** so the correct extraction path is selected for the document type. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

### B. **Field definition catalogue**

Each field should be formally defined with:

* field name
* business meaning
* data type
* cardinality
* required/optional
* line-level vs header-level
* expected source region if known

Example:

```json
{
  "fieldName": "shipment_number",
  "type": "string",
  "required": true,
  "scope": "header",
  "pattern": "^[0-9]{6,12}$"
}
```

### C. **Validation rule catalogue**

This is separate from extraction:

* required field checks
* regex / format checks
* numeric ranges
* date validity
* reconciliation rules
* signature requirements

This rule engine is **your application logic**, not an Azure AI Document Intelligence training feature.

***

# 3) How to choose the right Azure AI Document Intelligence model strategy

Microsoft documents several model paths:

* **Layout model** for extracting text, text locations, tables, selection marks, and structure. Key-value pairs can also be extracted with the layout model using the optional feature flag. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)
* **Prebuilt models** for common document types such as invoices, receipts, contracts, and IDs. The prebuilt invoice model extracts key invoice information and line items. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0), [\[Overview o...processing\]](https://learn.microsoft.com/en-US/microsoft-365/documentprocessing/prebuilt-overview)
* **Custom extraction models** for fields specific to your documents. Microsoft states these return structured JSON output and support labelled field extraction. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* **Custom classification models** to detect which document class a file belongs to before invoking extraction. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

## 3.1 Practical selection rule

### Use **prebuilt** when:

* the document type is already well supported by Microsoft, such as standard invoices. The product documentation says the invoice model already extracts key invoice fields and line items. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0), [\[Overview o...processing\]](https://learn.microsoft.com/en-US/microsoft-365/documentprocessing/prebuilt-overview)

### Use **custom extraction** when:

* the owner’s schema includes fields not covered by prebuilt models,
* the form is internal or semi-standard,
* or the document semantics are business-specific.

### Use **custom classification + custom extraction** when:

* you have multiple document types coming into the same pipeline,
* each type has a different schema,
* and document routing must happen before extraction. Microsoft explicitly documents this pairing. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

### Recommendation

For your logistics-style use cases, developers should usually implement:

```text
Custom classification → Custom extraction → Canonical schema mapping
```

This is my engineering recommendation based on your earlier architecture, not a Microsoft product requirement.

***

# 4) How developers should model the business schema into a trainable extraction model

## 4.1 Build a schema-to-label matrix

Developers should create a mapping sheet or JSON definition that links each business field to the training label.

Example:

```json
{
  "documentType": "DeliverySlip",
  "fields": [
    { "schemaField": "order_number", "modelLabel": "OrderNumber", "type": "string" },
    { "schemaField": "shipment_number", "modelLabel": "ShipmentNumber", "type": "string" },
    { "schemaField": "carrier_name", "modelLabel": "CarrierName", "type": "string" },
    { "schemaField": "receiver_signature", "modelLabel": "ReceiverSignature", "type": "signature" }
  ]
}
```

This is not something Microsoft provides out of the box — it is the implementation discipline developers should add.

***

## 4.2 Build the labelled training set

Microsoft documents that for custom extraction models you can get started with **as few as five documents**, though it also recommends larger sets, especially when image quality is lower or the forms vary more. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0)

Microsoft also documents that:

* custom extraction supports PDF and image formats such as JPEG/JPG, PNG, BMP, TIFF, and HEIF, and
* scanned PDFs are handled as images. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0)

### Developer implementation guidance

For each document type:

1. collect representative samples,
2. make sure the required fields are actually present,
3. vary values across examples,
4. include lower-quality scans if the real-world inbound stream includes them,
5. label **exactly** the business-owned fields.

Microsoft’s guidance specifically recommends:

* using examples with all fields completed,
* using forms with different values in each field,
* and using larger sets when images are lower quality. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0)

***

## 4.3 Choose custom neural vs template

Microsoft documents both custom template and custom neural model types, and recommends starting with a **neural model** to determine whether it meets the functional need. The custom neural model supports extracting key data fields from structured and semi-structured documents, and in the 2024-11-30 GA version it supports signature detection plus table, row, and cell confidence. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)

### Recommendation

For business documents whose layout varies between branches, suppliers, or carriers, developers should generally start with **custom neural**, not template. That recommendation is consistent with Microsoft’s “start with neural” guidance. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

***

# 5) How the runtime extraction pipeline should work technically

## 5.1 Native Document Intelligence response is not the final business contract

Microsoft documents that the Analyse Document response contains extracted objects such as:

* pages,
* paragraphs,
* styles,
* tables,
* languages,
* and semantic elements depending on the model. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

That response is **model output**, not your final business object.

So developers should implement a runtime mapping layer like this:

```text
Stored document
   ↓
Azure AI Document Intelligence
   ↓
Native analysis JSON
   ↓
Mapper / normaliser
   ↓
Business-owned canonical extraction contract
```

***

## 5.2 Canonical field envelope

Each extracted field in the canonical schema should preserve:

* normalised value
* raw OCR content
* confidence
* source location

Example:

```json
{
  "value": "2447651",
  "raw_text": "2447651",
  "confidence": 0.98,
  "bounding_regions": [
    { "page": 1, "polygon": [ ... ] }
  ]
}
```

Why this matters:

* Microsoft documents that the service returns structured data including relationships from the original file, bounding boxes, confidence scores, and more. [\[Use Foundr...oft Fabric\]](https://learn.microsoft.com/en-US/fabric/data-science/how-to-use-ai-services-with-synapseml)
* Microsoft also documents that the analyse response includes content elements positioned by spans and grouped by pages, allowing you to ground extracted values back to the source. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

This is what makes your human review queue fact-based instead of just “trust the model”.

***

# 6) How to implement targeted extraction from the business schema

## 6.1 Targeted extraction means: extract only what the owner cares about

Developers should not let the downstream system depend directly on every field the model happens to detect.

Instead:

* train and label the business-schema fields,
* map only those fields into the canonical contract,
* keep other extracted content as supporting evidence only.

This gives you:

* deterministic integration,
* less noisy output,
* tighter review UX,
* and more stable versioning.

***

## 6.2 Recommended implementation pattern

### Step 1 — classify the document

Use a custom classification model when multiple document types can enter the pipeline. Microsoft explicitly documents custom classification for this use case. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

### Step 2 — invoke the matching extraction model

Use the schema-driven extraction model for that type. Microsoft documents that custom extraction models return structured JSON output tailored to labelled fields. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

### Step 3 — map into your canonical schema

Transform model labels into business-owned field names.

### Step 4 — attach grounding metadata

Preserve field confidence and source region.

### Step 5 — run your validation and routing rules

This is where business confidence and review decisions happen.

***

# 7) How factual confidence scoring should work

This is where many implementations go wrong.

## 7.1 There are two different confidence concepts

### A. **Extraction confidence from Azure AI Document Intelligence**

Microsoft documents that Document Intelligence returns confidence estimates for predicted words, key-value pairs, selection marks, regions, and signatures, and that field confidence values are between 0 and 1. Microsoft also notes that not all fields return confidence scores. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

This tells you:

> “How statistically certain is the extraction model that this extracted value is correct?”

It does **not** tell you:

* whether the value is valid in your business,
* whether the arithmetic balances,
* whether the document should be auto-posted,
* or whether downstream integration should trust it.

### B. **Business validation confidence / routing score**

This is your application logic. It is not a Microsoft Document Intelligence output.

This score should answer:

> “Given what was extracted and validated, how safe is it to auto-process this document?”

***

## 7.2 Recommended factual scoring model

The most robust way to score “based upon what was actually extracted” is to compute the score only from:

1. **fields that were actually extracted**, and
2. **validation rules evaluated against those extracted values**

That means no speculative AI reasoning and no invented certainty.

### Recommended formula

```text
Document Readiness Score =
    Weighted Extraction Confidence
  + Validation Pass Score
  + Completeness Score
```

### A. Weighted Extraction Confidence

Use the actual field confidences returned by the service for business-critical fields.

Example weights:

* document reference number = 20%
* shipment / invoice number = 20%
* line items total = 20%
* customer / ship-to = 15%
* signature = 15%
* date fields = 10%

Only use the fields that matter for your workflow.

### B. Validation Pass Score

Evaluate rule outcomes such as:

* required fields present,
* regex match,
* date parse success,
* quantity > 0,
* total = sum(line items),
* signature present if required.

### C. Completeness Score

Calculate what percentage of required fields were populated.

***

## 7.3 Why this is factual

Because every score component is grounded in:

* extracted field values,
* model-supplied confidence where available,
* and deterministic business rules.

Microsoft’s documentation explicitly supports using confidence values to determine whether to automatically accept a prediction or flag it for human review. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

That is the foundation for human-review routing.

***

# 8) Developer implementation pattern for confidence scoring

## 8.1 Store field confidence at field level

Example:

```json
"shipment_number": {
  "value": "5284456",
  "raw_text": "5284456",
  "confidence": 0.97,
  "bounding_regions": [...]
}
```

## 8.2 Run deterministic validators

Example rule output:

```json
{
  "rule_id": "REQ-001",
  "field": "shipment_number",
  "status": "PASS",
  "severity": "error",
  "message": "Shipment number present"
}
```

## 8.3 Compute document-level score

Example:

```json
{
  "validation": {
    "status": "NeedsReview",
    "confidence_score": 0.84,
    "errors": ["Receiver signature missing"],
    "warnings": ["Carrier phone missing"]
  }
}
```

That final `confidence_score` is **your computed routing score**, not the native Document Intelligence score.

***

# 9) Suggested developer implementation workflow

## Phase 1 — Schema engineering

Developers should:

1. take the owner schema,
2. classify fields into:
   * header fields,
   * line-item fields,
   * signature / special fields,
3. mark required vs optional,
4. define regex / type constraints,
5. version the schema.

## Phase 2 — Model engineering

Developers should:

1. collect training documents per type,
2. label fields matching the schema,
3. build custom classification if there are multiple doc types,
4. build custom extraction per type,
5. test on holdout documents.

Microsoft documents:

* custom classification,
* custom extraction,
* supported formats,
* training data input requirements,
* and the recommendation to start with neural models. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)

## Phase 3 — Runtime engineering

Developers should:

1. persist the document,
2. classify it,
3. extract it,
4. map the native response to canonical schema,
5. run validation rules,
6. compute document readiness score,
7. auto-process or route to human review.

## Phase 4 — Review and feedback

Because the service returns confidence and position information and your schema preserves source grounding, reviewers can correct values against the original document. Microsoft documents confidence scoring and source-related output structures that support this style of post-processing and review. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0), [\[Use Foundr...oft Fabric\]](https://learn.microsoft.com/en-US/fabric/data-science/how-to-use-ai-services-with-synapseml)

***

# 10) The most important implementation lesson

If developers skip the business schema as the **first-class model input**, the result is usually:

* noisy extraction,
* brittle downstream mapping,
* inconsistent confidence handling,
* and review workflows that users do not trust.

If developers use the schema correctly, they get:

* targeted extraction,
* factual confidence scoring,
* stable integration contracts,
* and explainable human review.

***

# 11) Accuracy check

## Directly supported by Microsoft documentation

* Custom extraction models are trained on labelled fields and return structured JSON output. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* Custom classification models can be paired with extraction models. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* You can start custom extraction with as few as five documents, though larger/better training sets improve quality. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0)
* Microsoft recommends starting with neural models to determine whether they meet functional needs. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* Custom neural models support signature detection and table/row/cell confidence in the current GA version. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)
* The analyse response includes structured elements such as pages, paragraphs, tables, styles, and semantic elements. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)
* Confidence values are provided for supported extracted predictions and can be used for human-review decisions; not all fields return confidence scores. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

## My engineering recommendations

The following are my design recommendations rather than Microsoft product claims:

* schema-to-label matrix,
* canonical field envelope,
* weighted readiness score formula,
* separation of extraction confidence vs business validation confidence,
* rule taxonomy and review routing strategy.

***

# 12) Best next step

The most valuable next move is to make this concrete with one of these:

1. **A field-by-field developer template**  
   Business schema → training labels → canonical schema → validation rules

2. **A sample implementation pack for one document type**  
   For example: Delivery Slip or POD, including:
   * schema definition,
   * labelled field plan,
   * scoring rules,
   * Python mapping skeleton

If you want, I can do that next and make it implementation-ready.
