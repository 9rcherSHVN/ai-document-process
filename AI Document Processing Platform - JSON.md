Absolutely — let’s do a proper technical deep dive and tighten the architecture thinking.

The short version is:

> The **JSON schema / extraction contract** is **not** the same thing as the native Azure AI Document Intelligence response.  
> It is a **canonical, application-owned contract** that sits **after** OCR / extraction and **before** storage, validation, review, and downstream integration. Azure AI Document Intelligence returns structured analysis objects such as pages, paragraphs, tables, styles, languages, and model-specific semantic results; the service is designed to return structured JSON output for applications and workflows. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/?view=doc-intel-4.0.0)

That distinction is the key to making the design work cleanly in **Azure**, **Python**, **Power Platform**, and **ERP/TMS/WMS**.

***

# 1) What I meant by “tailored for Power Platform, Azure AI Document Intelligence, Python pipelines, and enterprise integration”

## 1.1 Canonical contract, not vendor-native payload

When I said the schema is “tailored”, I meant the contract is designed to be the **stable internal representation** of a document regardless of whether the extraction came from:

* a prebuilt invoice model,
* a layout model plus post-processing,
* a custom extraction model,
* or even a future non-Microsoft OCR provider.

Azure AI Document Intelligence returns model outputs such as pages, words, lines, paragraphs, tables, styles, languages, and semantic elements, and its responses vary depending on the model type. The docs explicitly describe the response as a model-specific analysis result containing different objects, not a single business-ready ERP contract. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

So the canonical JSON schema exists to solve these problems:

1. **Insulate your downstream systems from model changes**  
   If you change from `prebuilt-layout` to a custom neural model, your ERP payload does not need to change. That is an architectural choice, not something the service does for you.

2. **Normalise multiple document types into one enterprise model**  
   Delivery slips, invoices, PODs, and packing lists can all land in one governed contract.

3. **Attach operational metadata that the extraction service does not own**  
   For example:
   * `document_id`
   * `blob_url`
   * `processing_stage`
   * `review_status`
   * `validation.errors`
   * `routing_decision`

4. **Add explainability and deterministic outputs**  
   This matches the structured-output discipline already reflected in your own design notes, where you emphasised “strict JSON schema response” and “Structured JSON / no free-text in core reasoning tools”. [\[System Design - NOTE \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B823653A4-4F2A-4E23-9C5B-6AB8EB5D8978%7D&file=System%20Design%20-%20NOTE.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Building a...ntic layer \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BAE5EEF43-24DA-4C36-B494-63BCD15021B0%7D&file=Building%20a%20real%20semantic%20layer.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

***

# 2) How the schema fits into the Azure pipeline

Here is the practical pipeline with the schema in the middle.

```text
Input (file / base64 / binary stream)
    ↓
Ingestion + normalisation
    ↓
Blob persistence
    ↓
Azure AI Document Intelligence analysis
    ↓
Raw DI response (vendor-native JSON)
    ↓
Mapping layer (Python / Azure Function)
    ↓
Canonical extraction contract (your JSON schema)
    ↓
Validation rules + confidence aggregation
    ↓
Dataverse / SQL / API payload / Human Review Queue
```

The technical point is this:

* **Raw Document Intelligence response** = model-centric extraction objects. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)
* **Canonical contract** = business-centric document object that your application controls.

***

# 3) What the native Azure AI Document Intelligence response looks like conceptually

The official documentation says the Analyse operation is asynchronous and returns an `Operation-Location` header to poll for completion. When the analysis completes, the response contains extracted content, grouped by page and reading order, plus layout and semantic objects depending on the model used. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

## 3.1 Response building blocks

From the official docs, the response may contain:

* `pages`
* `paragraphs`
* `tables`
* `styles`
* `languages`
* `documents`
* top-level `content` with spans into that content string. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

The layout model specifically extracts text, text locations, tables, selection marks, and document structure. The layout model can also extract key-value pairs when `features=keyValuePairs` is enabled. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)

Prebuilt models such as the invoice model extract domain-specific fields and line items. The documentation explicitly says the prebuilt invoice model extracts key fields and line items from invoices. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)

Custom extraction models are trained to return structured JSON output tailored to the labelled fields in your own documents. Custom classification models can be paired with extraction models so that document type is identified before extraction is invoked. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

***

# 4) Why the canonical schema is necessary even though Document Intelligence already returns JSON

Because the service JSON is **analysis JSON**, not an **enterprise document contract**.

## 4.1 Example of the mismatch

A raw response might give you:

* a recognised table,
* a paragraph,
* a key-value pair,
* a field confidence,
* a bounding region,
* a span into top-level text.

That is excellent for extraction, but downstream systems usually need:

* `document_type`
* `order_number`
* `shipment_number`
* `carrier_name`
* `line_items[]`
* `validation.status`
* `review_required`
* `blob_uri`
* `processing_stage`

The canonical schema therefore acts as a **translation layer**:

* from native OCR semantics
* into governed business semantics

This is consistent with the structured-output and mapping discipline shown in your internal notes, where you explicitly call out strict schema-based responses, weighted scoring, deterministic outputs, semantic rules, and explainability evidence. [\[System Design - NOTE \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B823653A4-4F2A-4E23-9C5B-6AB8EB5D8978%7D&file=System%20Design%20-%20NOTE.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Building a...ntic layer \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BAE5EEF43-24DA-4C36-B494-63BCD15021B0%7D&file=Building%20a%20real%20semantic%20layer.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

***

# 5) How the mapping works in Azure / Python in practice

## 5.1 Mapping pattern by model type

### A) Layout model path

Use the layout model when:

* document types vary,
* you need raw structure first,
* or you are building your own extraction logic.

The layout model returns text, locations, tables, selection marks, and structure. Key-value extraction can also be enabled. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)

**Mapping logic** then:

* scan tables for product rows,
* scan key-value pairs for `Order Number`, `Ship To`, etc.,
* map them into your field envelope.

### B) Prebuilt model path

Use prebuilt invoice when the document is a standard invoice. The prebuilt invoice model extracts key fields and line items directly. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)

**Mapping logic** then:

* copy recognised invoice fields into the canonical contract,
* normalise date formats, amounts, and currencies,
* add operational metadata and validation.

### C) Custom model path

Use custom extraction when internal logistics forms have recurring but specialised fields. The official docs say custom extraction models return structured JSON output, and classifier models can be paired with extraction models to identify document type first. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

**Mapping logic** then:

* map model field names to canonical business field names,
* preserve confidence and bounding regions,
* version field mappings explicitly.

***

## 5.2 Canonical field envelope

This is the important pattern:

```json
{
  "value": "2447651",
  "raw_text": "2447651",
  "confidence": 0.98,
  "bounding_box": {
    "page": 1,
    "x": 0.41,
    "y": 0.18,
    "width": 0.12,
    "height": 0.02
  }
}
```

This envelope is useful because it carries all of the following in one place:

* **business value** (`value`)
* **raw OCR evidence** (`raw_text`)
* **model confidence** (`confidence`)
* **human-review grounding** (`bounding_box`)

That design aligns directly with the way the service exposes content positions, spans, and structural elements in the analysis response. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

***

## 5.3 Python mapping layer example

Below is the pattern I mean technically. This code is an implementation recommendation, not copied from the docs.

```python
def map_field(di_field, fallback_text=None):
    if di_field is None:
        return {
            "value": None,
            "raw_text": fallback_text,
            "confidence": 0.0,
            "bounding_box": None
        }

    return {
        "value": di_field.get("value"),
        "raw_text": di_field.get("content"),
        "confidence": di_field.get("confidence"),
        "bounding_box": normalise_regions(di_field.get("boundingRegions"))
    }
```

Then your mapper transforms native output into a canonical contract:

```python
canonical = {
    "document_metadata": {
        "document_id": document_id,
        "document_type": detected_type,
        "source_file_name": file_name
    },
    "header": {
        "document_reference": {
            "order_number": map_field(fields.get("OrderNumber")),
            "shipment_number": map_field(fields.get("ShipmentNumber"))
        }
    },
    "validation": {
        "status": "Pending",
        "errors": [],
        "warnings": [],
        "confidence_score": None
    }
}
```

The `fields` object above may come from:

* prebuilt invoice fields,
* custom model labelled fields,
* or your own layout/key-value extraction logic.

***

# 6) How the schema works in Dataverse / Power Platform

This is where the schema becomes especially useful.

## 6.1 Dataverse does not want raw OCR complexity everywhere

You generally do **not** want to expose the full native response with all pages, spans, words, and structural objects to every business consumer.

Instead, you normally store:

### A) Raw document artefacts

* Blob URL
* native Document Intelligence response JSON (optional but recommended)

### B) Canonical document record

* `DocumentId`
* `DocumentType`
* `Status`
* `ConfidenceScore`
* `BlobUrl`
* `RawJson`

### C) Child rows

* `LineItems`
* `ReviewTasks`
* `ValidationIssues`

That is what I meant by “tailored for Power Platform”: Dataverse works best when business entities are explicit and review workflows can bind directly to stable business fields rather than raw OCR internals.

***

## 6.2 Human Review Queue

Because your updated architecture requires physical persistence, the reviewer can see:

* the original Blob document,
* the canonical extracted values,
* per-field confidence,
* and the evidence region.

That last point is why the field envelope matters. It gives the reviewer a grounded correction experience instead of “JSON only”.

This also lines up with the explainability discipline in your own design notes, where you emphasised evidence and deterministic outputs. [\[Building a...ntic layer \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BAE5EEF43-24DA-4C36-B494-63BCD15021B0%7D&file=Building%20a%20real%20semantic%20layer.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

***

# 7) How validation rules work with the schema

This is the second major concept: **validation is not the same as extraction**.

The schema has two distinct confidence domains:

1. **Extraction confidence**  
   Returned by Azure AI Document Intelligence for recognised results where supported. Microsoft documents that analysis results return estimated confidence for predicted words, key-value pairs, selection marks, regions, and signatures, and that some fields may not have confidence values. Field confidence is between 0 and 1. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

2. **Business validation confidence / routing score**  
   This is **your own application logic**, not a Document Intelligence feature.

That distinction matters a lot.

***

## 7.1 What Document Intelligence confidence means

The docs say a confidence score is the probability measuring statistical certainty that an extracted result is detected correctly. Microsoft also states that these scores can be used to decide whether to automatically accept the prediction or flag it for human review. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

So if `shipment_number.confidence = 0.97`, that means:

* the extraction model is highly confident in the detected value.

It does **not** mean:

* the shipment number is valid in your ERP,
* the shipment belongs to the listed customer,
* totals reconcile,
* or the document should be auto-posted.

Those checks belong to your validation engine.

***

## 7.2 Validation rule categories

I recommend three rule layers.

### A) Structural validation

Checks whether the extracted contract is minimally complete.

Examples:

* document type detected
* at least one header reference present
* at least one line item for invoice / delivery documents
* blob persisted
* file type supported

### B) Field validation

Checks field-level correctness.

Examples:

* `order_number` format matches regex
* `document_date` is a valid date
* `quantity > 0`
* `currency` is known
* `uom` is allowed

### C) Cross-field / business validation

Checks logical consistency.

Examples:

* `total_amount == sum(line_items.total_price)`
* `ship_to.name` present when `document_type = DeliverySlip`
* receiver signature required for POD
* duplicate shipment number blocked
* customer/carrier combinations allowed

This is strongly aligned with the internal patterns in your own files, where you explicitly describe weighted scoring, aggregator logic, validation rules, and business-rule enforcement as externalised and deterministic. [\[System Design - NOTE \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B823653A4-4F2A-4E23-9C5B-6AB8EB5D8978%7D&file=System%20Design%20-%20NOTE.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Building a...ntic layer \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BAE5EEF43-24DA-4C36-B494-63BCD15021B0%7D&file=Building%20a%20real%20semantic%20layer.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[FIT to LIM...ng version \| Word\]](https://interfor.sharepoint.com/sites/Project-Hub/_layouts/15/Doc.aspx?sourcedoc=%7B6C0C83DB-A728-4C3E-BC49-2A2D73199097%7D&file=FIT%20to%20LIMS%20Requirements%20V2%20-%20afte%20meeting%208June%20long%20version.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

***

# 8) How validation produces the overall confidence score

## 8.1 Important distinction: two scores

I recommend storing:

* `field.confidence` → native extraction confidence from Document Intelligence
* `validation.confidence_score` → your document-level routing score

That avoids conflating OCR certainty with business readiness.

***

## 8.2 Suggested scoring model

This part is **my design recommendation**, not an Azure-native feature.

A clean approach is:

```text
overall_score =
  extraction_component * 0.60
+ validation_component * 0.25
+ completeness_component * 0.15
```

### Extraction component

Weighted average of important field confidences, for example:

* order number
* invoice number
* shipment number
* total amount
* receiver signature
* line item totals

### Validation component

Rule pass/fail weighting, for example:

* all mandatory fields present
* format checks passed
* arithmetic checks passed
* duplicate checks passed

### Completeness component

Fraction of required fields populated.

***

## 8.3 Example

Suppose:

* `order_number.confidence = 0.98`
* `shipment_number.confidence = 0.96`
* `carrier_name.confidence = 0.91`
* `product_code.confidence = 0.88`
* `receiver_signed.present = true`
* mandatory fields all present
* arithmetic checks pass
* one warning: carrier phone missing

Then you might compute:

```text
extraction_component = 0.93
validation_component = 0.95
completeness_component = 1.00

overall_score = 0.93*0.60 + 0.95*0.25 + 1.00*0.15
              = 0.9455
```

Then route:

* `>= 0.90` → auto-process
* `0.75–0.89` → soft review
* `< 0.75` → manual review

Again, that thresholding is an application design recommendation. Microsoft’s documentation supports using confidence to determine whether to auto-accept or flag for human review, but the exact weighting and thresholds are yours to define. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

***

# 9) Why the schema makes confidence scoring easier

Without a canonical schema, the validation engine would have to understand multiple raw response shapes:

* layout response
* invoice response
* custom model response
* future versions of the service

With the canonical contract, validation logic becomes model-agnostic:

```python
def validate_required_fields(doc):
    errors = []
    if not doc["header"]["document_reference"]["order_number"]["value"]:
        errors.append("Missing order number")
    if doc["document_metadata"]["document_type"] == "DeliverySlip":
        if not doc["parties"]["ship_to"]["name"]["value"]:
            errors.append("Missing ship-to name")
    return errors
```

That is why the contract is the **control point** for:

* business rules,
* confidence aggregation,
* human review routing,
* and downstream ERP posting.

***

# 10) How this maps specifically to Azure AI Document Intelligence implementations

## 10.1 Prebuilt invoice

* Use prebuilt invoice model
* take service fields and line items
* map into canonical contract
* validate totals and required references
* store field confidences
* compute routing score

The official docs say the prebuilt invoice model extracts key fields and line items. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)

## 10.2 Layout + key-value path

* use `prebuilt-layout`
* optionally enable key-value extraction
* use table detection and reading-order text
* map values with evidence regions
* run stronger post-processing rules

The official docs state the layout model extracts text, locations, tables, selection marks, and structure, and can extract key-value pairs with the optional feature flag. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)

## 10.3 Custom extraction + classification

* classify document first
* invoke matching custom extraction model
* map model-labelled fields to canonical schema
* preserve service confidence
* apply business validation

The official docs explicitly state that classifier models can be paired with custom extraction models and that custom extraction models return structured JSON output tailored to labelled fields. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

***

# 11) Where the schema sits relative to training and model quality

Microsoft distinguishes:

* **field-level confidence at analysis time**, and
* **estimated accuracy for certain custom models during training**.  
  The docs also note that custom neural and generative models do **not** provide training accuracy scores, whereas other custom model build outputs may include estimated accuracy. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

So in practice:

* **training accuracy** helps you judge model quality during build time
* **field confidence** helps you judge extraction certainty at runtime
* **validation score** helps you judge business readiness in your application

Those are three separate concepts, and it is a common design mistake to collapse them into one number.

***

# 12) My recommended final model for your implementation

For your Azure + M365 design, I would make the contract own these three layers:

## Layer A — Extraction evidence

Per field:

* `value`
* `raw_text`
* `confidence`
* `bounding_box`

## Layer B — Validation evidence

Per rule:

* `rule_id`
* `status`
* `severity`
* `message`
* `affected_fields[]`

## Layer C — Routing decision

At document level:

* `overall_confidence_score`
* `review_required`
* `auto_post_eligible`
* `review_reason`

That gives you:

* explainability,
* auditability,
* deterministic routing,
* and a clean Power Apps human-review experience.

***

# 13) Accuracy check on what I’ve said

## Directly supported by Microsoft documentation

* Azure AI Document Intelligence is asynchronous and returns an analysis response with pages, paragraphs, tables, styles, languages, and model-specific elements. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)
* The layout model extracts text, structure, tables, selection marks, and can extract key-value pairs when enabled. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)
* Prebuilt invoice extracts key fields and line items. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/use-sdk-rest-api?view=doc-intel-4.0.0)
* Custom extraction returns structured JSON output, and classifier models can be paired with extraction models. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* Confidence scores are returned for predicted words, key-value pairs, selection marks, regions, and signatures where supported, and they can be used to decide auto-accept versus human review. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)
* Not all fields return confidence scores. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)

## My architecture / implementation recommendations

The following are my engineering recommendations rather than explicit Microsoft product behaviour:

* treating the canonical JSON schema as a stable enterprise contract,
* separating extraction confidence from business validation confidence,
* weighted document-level routing score,
* specific threshold ranges,
* storing validation evidence separately from extraction evidence,
* Dataverse table structure and review model.

## Internal alignment with your existing design style

Your internal files already reflect the same architectural direction:

* strict JSON schema responses,
* weighted scoring aggregator,
* deterministic outputs,
* explainability evidence,
* explicit validation rules and rule engines. [\[System Design - NOTE \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B823653A4-4F2A-4E23-9C5B-6AB8EB5D8978%7D&file=System%20Design%20-%20NOTE.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Building a...ntic layer \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BAE5EEF43-24DA-4C36-B494-63BCD15021B0%7D&file=Building%20a%20real%20semantic%20layer.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[FIT to LIM...ng version \| Word\]](https://interfor.sharepoint.com/sites/Project-Hub/_layouts/15/Doc.aspx?sourcedoc=%7B6C0C83DB-A728-4C3E-BC49-2A2D73199097%7D&file=FIT%20to%20LIMS%20Requirements%20V2%20-%20afte%20meeting%208June%20long%20version.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

***

If you want, I can do the next deep dive in one of two directions:

1. **Show the exact canonical schema mapped field-by-field to a sample Azure AI Document Intelligence response**
2. **Design the validation engine in detail — including rule taxonomy, score formulas, and Dataverse tables for review/audit**

If you want the most immediately useful next step, I’d suggest **option 1 first**, because it makes the entire pipeline concrete.
