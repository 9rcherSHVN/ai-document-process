Below is my technical re-assessment of the C# implementation example, using a deeper production-architecture lens rather than treating it as just a code scaffold.

## 1. Executive assessment

The previous implementation is **directionally correct** as a conceptual C# scaffold. It correctly follows the core v3 architecture idea: **deterministic orchestrator + specialist skills + canonical schema + validation + confidence routing**. That matches your architecture notes, where the end-to-end pipeline includes ingestion, pre-processing, OCR/layout, classification, schema-based extraction, validation/business rules, human review, and downstream output/integration. [Detailed technical architecture - AI Document Intelligence.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B42A57CFD-44CA-42F8-8600-55B4FD2DC0AD%7D\&file=Detailed%20technical%20architecture%20-%20AI%20Document%20Intelligence.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=87a67de5-ba5b-46ff-865e-e771e6298402) and [Recommended technical design solution - AI Document Intelligence.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D\&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=6b920bbf-36e1-40d4-99ac-aade513e1655) explicitly describe that pipeline and the use of an orchestration layer coordinating specialised document-processing skills. [\[Detailed t...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B42A57CFD-44CA-42F8-8600-55B4FD2DC0AD%7D&file=Detailed%20technical%20architecture%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Recommende...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

However, the implementation is still closer to a **developer demo skeleton** than a production-grade enterprise document processing platform. The largest gaps are not in the basic C# syntax; they are in **runtime orchestration, model/version governance, schema/rule externalisation, idempotency, evidence handling, observability, security, and continuous improvement**. Your own platform notes emphasise that the business schema should drive model design, runtime mapping, confidence routing, review UX, and validation rules, and that factual confidence scoring must be grounded in extracted evidence rather than invented by an LLM. [Document Type - JSON schema - AI Document Processing Platform.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D\&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=0ac573f6-93fb-480c-80a0-25064fd9cd7d) states this schema-driven principle directly. [\[Document T...g Platform \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

So my verdict:

> **Good architecture pattern. Incomplete production implementation. Needs to move from “skills as classes” to “governed, observable, versioned, event-driven document processing platform.”**

***

## 2. Key system components in the implementation

Here are the key system components I would identify from the implementation example, grouped as business-facing and engineering-facing components.

### A. Business-facing platform components

| Component                             | Purpose                                                                                                          | Assessment                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Document intake / ingestion layer** | Accepts documents from Blob, SharePoint, API uploads, email attachments, or other sources.                       | Correctly implied, but underbuilt. It should become a first-class component, not just an Azure Function trigger.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Document classification component** | Decides whether the document is a delivery slip, invoice, POD, bill of lading, packing list, etc.                | Correct architectural component. Your architecture explicitly calls for custom classification before extraction when multiple document types enter the same pipeline. [Document Type - JSON schema - AI Document Processing Platform.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D\&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=0ac573f6-93fb-480c-80a0-25064fd9cd7d) and [Designing a multi-agent document AI system with specialized skills.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D\&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=c1d8080f-8c08-4142-80b7-555c63371ce4) both support this classification-before-extraction pattern. [\[Document T...g Platform \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Designing...zed skills \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx&action=default&mobileredirect=true&DefaultItemOpen=1) |
| **Schema-based extraction component** | Extracts only business-relevant fields according to the document type’s schema.                                  | Correct, but the implementation hard-codes too much and does not yet use a proper schema registry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| **Canonical document contract**       | Converts native Azure Document Intelligence output into a stable business schema.                                | This is one of the strongest parts of the design. [Document Type - JSON schema - for AI Document Procssing Pipeline Design.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D\&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=d3951ea7-2264-45c9-b688-ccb0a9d754a0) explicitly distinguishes the application-owned canonical contract from the native Document Intelligence response. [\[Document T...ine Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Validation rules engine**           | Applies structural, field-level, and cross-field business rules.                                                 | Conceptually correct, but rules should not live only in C# code.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Human review queue**                | Routes low-confidence or exception cases to reviewers with extracted value, evidence, and correction capability. | Mentioned architecturally but not implemented in enough detail. [Recommended technical design solution - AI Document Intelligence.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D\&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=6b920bbf-36e1-40d4-99ac-aade513e1655) states that low-confidence cases should be sent to a human review queue with extracted text, cropped evidence region, correction, and feedback capture. [\[Recommende...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Output and integration layer**      | Pushes approved canonical data into ERP, TMS, WMS, SQL, Dataverse, SharePoint, or API workflows.                 | Correctly listed but implemented only as TODO. This must become a reliable integration subsystem.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

### B. Engineering-facing platform components

| Component                                | Purpose                                                                                              | Assessment                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Orchestrator**                         | Controls pipeline sequence and routing.                                                              | Good conceptual separation. Your notes describe the orchestrator plus specialist skills pattern rather than a single general-purpose agent. [Designing a multi-agent document AI system with specialized skills.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D\&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=c1d8080f-8c08-4142-80b7-555c63371ce4) explicitly lists Classification, Extraction, Schema Mapping, Validation, Confidence Scoring, and Review Decision skills. [\[Designing...zed skills \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx&action=default&mobileredirect=true&DefaultItemOpen=1) |
| **Document Intelligence client wrapper** | Encapsulates Azure Document Intelligence SDK calls.                                                  | Needed, but the previous code used SDK calls too directly inside business skills. I would isolate SDK specifics behind adapters.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **Model registry**                       | Maps document type, version, business unit, and environment to model IDs.                            | Missing. This is a major production gap.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Schema registry**                      | Stores canonical schemas, required fields, mappings, and field metadata by document type/version.    | Missing. The previous `FieldMap` dictionary is not enough.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Rules registry / rules engine**        | Stores validation rules separately from code.                                                        | Missing. Hard-coded rules will become a maintenance bottleneck.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Evidence store**                       | Stores bounding polygons, page references, spans, extracted text, and reviewer evidence.             | Weakly represented. Prior code used `BoundingRegion.ToString()`, which is not sufficient.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Observability store**                  | Tracks pipeline run, stage logs, errors, confidence scores, reviewer corrections, and model version. | Missing. Your OIC technical design work strongly values pipeline observability, run IDs, stage logs, status, and reprocessability; the same discipline should be applied here. [OIC Custom Integration Layer - Technical Design.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B781ABF99-40A9-43E6-9E48-77A9DE9966D6%7D\&file=OIC%20Custom%20Integration%20Layer%20-%20Technical%20Design.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=e85d496c-df3a-4f17-a072-ffd3e2254089) describes pipeline run and stage log patterns for integration observability. [\[OIC Custom...cal Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B781ABF99-40A9-43E6-9E48-77A9DE9966D6%7D&file=OIC%20Custom%20Integration%20Layer%20-%20Technical%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)                                                    |
| **Continuous improvement loop**          | Captures reviewer corrections and model performance feedback for retraining.                         | Architecturally implied, not implemented.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Security and identity layer**          | Handles managed identity, Key Vault, private networking, RBAC, audit, and data retention.            | Underdeveloped.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

***

## 3. Technical reassessment of the previous C# implementation

### 3.1 What was good

#### Good: correct separation of responsibilities

The implementation separated classification, extraction, mapping, validation, scoring, and review decision into independent skills. That is aligned with your v3 architecture, which explicitly states that Azure Document Intelligence is not the whole platform; the platform must add orchestration, schema mapping, validation, confidence scoring, and integration. [Designing a multi-agent document AI system with specialized skills.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D\&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=c1d8080f-8c08-4142-80b7-555c63371ce4) makes the same distinction by stating that Document Intelligence provides extraction/layout/classification capabilities, while schema mapping, validation, scoring, and review decisions must be built around it. [\[Designing...zed skills \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

#### Good: canonical schema concept

The `CanonicalDocument` model was the right idea. Your architecture correctly treats the canonical schema as an **application-owned extraction contract**, not as the raw Azure response. [Document Type - JSON schema - for AI Document Procssing Pipeline Design.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D\&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=d3951ea7-2264-45c9-b688-ccb0a9d754a0) says the canonical schema sits after OCR/extraction and before storage, validation, review, and downstream integration. [\[Document T...ine Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

#### Good: deterministic validation and routing

The previous implementation avoided using an LLM to “decide everything”. That is consistent with your broader design principle of using deterministic code for rules and using AI only where it adds value. [technical solution notes.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B4862C24B-6B01-4043-9CEE-98D20AD7595F%7D\&file=technical%20solution%20notes.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=727eb2dd-a723-40dc-baa0-7056a96d5414) describes the principle of deterministic-before-generative and using AI only where code cannot reliably reason. [\[technical...tion notes \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B4862C24B-6B01-4043-9CEE-98D20AD7595F%7D&file=technical%20solution%20notes.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

***

## 4. Main technical issues and corrections

### Issue 1 — The Azure Function HTTP upload implementation was too simplified

The previous example used a simplified HTTP trigger and assumed easy multipart parsing. In production, I would avoid making the main processing function parse large multipart documents directly. A better design is:

1. Upload document to Blob Storage.
2. Emit a message/event with `documentId`, `blobUri`, `sourceSystem`, `correlationId`.
3. Let the orchestrator process by reference.
4. Persist every stage result.

This is more reliable for large files, retries, and auditability. It also aligns better with the Azure SDK, because the official .NET API includes `AnalyzeDocumentAsync` overloads that can analyse a document from `BinaryData`, `AnalyzeDocumentOptions`, or a `Uri`. DocumentIntelligenceClient.AnalyzeDocumentAsync Method lists overloads including `AnalyzeDocumentAsync(WaitUntil, String, BinaryData, CancellationToken)` and `AnalyzeDocumentAsync(WaitUntil, String, Uri, CancellationToken)`. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/dotnet/api/azure.ai.documentintelligence.documentintelligenceclient.analyzedocumentasync?view=azure-dotnet)

**What I would change:**

```csharp
public sealed record DocumentSubmittedEvent(
    string DocumentId,
    Uri BlobUri,
    string SourceSystem,
    string FileName,
    string ContentType,
    string CorrelationId);
```

Then process from Blob URI or from a short-lived managed-access reference rather than passing large streams through every class.

***

### Issue 2 — SDK usage must be pinned to an exact version

The code was written as a reasonable conceptual scaffold, but in production the exact SDK signatures must be pinned and tested against the chosen package version. Azure Document Intelligence client library for .NET states that the `Azure.AI.DocumentIntelligence` client library is version `1.0.0` and that this version defaults to the `2024-11-30` service version. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.documentintelligence-readme?view=azure-dotnet)

The previous code mixed conceptual classes such as `ClassifyDocumentContent` and direct stream conversion. That may need adjustment depending on the exact SDK version and overloads used.

**What I would change:**

* Create a `DocumentIntelligenceAdapter` project.
* Keep all Azure SDK-specific code there.
* Unit-test adapter methods with fixed SDK version.
* Prevent the rest of the platform from depending on SDK object models.

```csharp
public interface IDocumentAnalysisAdapter
{
    Task<ClassificationResultDto> ClassifyAsync(
        Uri documentUri,
        string classifierId,
        CancellationToken ct);

    Task<ExtractionResultDto> ExtractAsync(
        Uri documentUri,
        string modelId,
        CancellationToken ct);
}
```

This avoids Azure SDK types leaking into the business domain.

***

### Issue 3 — Classification split mode was missing

This is a significant design correction. If you process document packets containing multiple document types, classification is not just “one file = one class”. The official custom classification documentation says that in v4.0 `2024-11-30`, custom classification does **not** split documents by default during analysis; `splitMode` must be explicitly set to `auto` to preserve previous behaviour. Document Intelligence custom classification model states that the default for `splitMode` is `none` and that multi-document inputs need splitting enabled by setting `splitMode` to `auto`. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)

The previous implementation assumed a top document result and did not handle multi-document packets or page ranges.

**What I would change:**

* Support two modes:
  * **Single-document mode**: one file represents one document.
  * **Packet mode**: one file may contain several document classes or instances.
* Preserve page ranges from classification.
* Run extraction per classified segment/page range, not blindly against the whole file.

Conceptually:

```csharp
public sealed record ClassifiedSegment(
    string SegmentId,
    DocumentType DocumentType,
    double Confidence,
    int StartPage,
    int EndPage,
    string ClassifierModelId,
    string ClassifierModelVersion);
```

This is critical for real enterprise scanning batches.

***

### Issue 4 — The schema mapping layer was too hard-coded

The previous `FieldMap` dictionary is acceptable for a demo, but not for your final architecture. It creates several problems:

* Developers must redeploy code for field mapping changes.
* Business analysts cannot review mappings easily.
* Model version changes become risky.
* Required fields and semantic field types are not centrally governed.
* Completeness scoring becomes unreliable.

Your architecture says the business document schema should drive training, labelling, model selection, runtime mapping, confidence routing, review UX, and validation rules. [Document Type - JSON schema - AI Document Processing Platform.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D\&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=0ac573f6-93fb-480c-80a0-25064fd9cd7d) states that schema should drive all of these lifecycle steps. [\[Document T...g Platform \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

**What I would change:**

Introduce a versioned schema registry:

```json
{
  "documentType": "DeliverySlip",
  "schemaVersion": "1.0.0",
  "modelBindings": {
    "classifierModelId": "logistics-classifier-v3",
    "extractionModelId": "delivery-slip-extractor-v5"
  },
  "fields": [
    {
      "canonicalName": "shipmentNumber",
      "sourceLabels": ["shipment_number", "ShipmentNo", "DeliveryShipment"],
      "type": "string",
      "required": true,
      "critical": true,
      "minConfidence": 0.90
    },
    {
      "canonicalName": "shipToName",
      "sourceLabels": ["ship_to", "ShippingAddress", "ShipTo"],
      "type": "string",
      "required": true,
      "critical": true,
      "minConfidence": 0.85
    }
  ]
}
```

Then the mapper becomes metadata-driven:

```csharp
public interface ISchemaRegistry
{
    Task<DocumentSchema> GetSchemaAsync(
        DocumentType documentType,
        string schemaVersion,
        CancellationToken ct);
}
```

This is one of the biggest improvements I would make.

***

### Issue 5 — Completeness scoring was effectively broken

The previous `CompletenessScore` used:

```csharp
var requiredFields = doc.Fields.Where(f => f.IsRequired).ToList();
```

But `IsRequired` was never populated from a schema. Therefore, most documents would likely have no required fields defined, causing completeness to default to `1.0`. That produces artificially high confidence.

**What I would change:**

Completeness must come from the schema registry, not from extracted fields themselves.

```csharp
double CalculateCompleteness(
    CanonicalDocument doc,
    DocumentSchema schema)
{
    var required = schema.Fields.Where(f => f.Required).ToList();

    if (required.Count == 0)
        return 1.0;

    int present = required.Count(field =>
        doc.TryGetValue(field.CanonicalName, out var value) &&
        !string.IsNullOrWhiteSpace(value?.ToString()));

    return (double)present / required.Count;
}
```

This is a correctness issue, not just an optimisation.

***

### Issue 6 — Confidence scoring was too naive

The previous scoring used a simple average of field confidence plus validation and completeness. That is a useful starting point, but production scoring needs more nuance.

The Microsoft confidence guidance says a confidence score indicates probability by measuring the degree of statistical certainty that an extracted result is detected correctly. Interpret and improve accuracy and confidence scores states that confidence scores indicate probability/statistical certainty for extracted results. [\[github.com\]](https://github.com/MicrosoftDocs/azure-ai-docs/blob/main/articles/ai-services/document-intelligence/concept/accuracy-confidence.md)

But enterprise routing should not treat all fields equally. A missing `carrierName` may be a review issue; a missing `shipmentNumber` may be a hard stop.

**What I would change:**

Use weighted, severity-aware scoring:

```csharp
public sealed record FieldScoringPolicy(
    string FieldName,
    bool Required,
    bool Critical,
    double MinimumConfidence,
    double Weight);
```

Then:

* Critical field below threshold → human review or rejection.
* Non-critical field below threshold → lower score but not necessarily reject.
* Validation failure severity affects routing.
* Classification confidence and extraction confidence are scored separately.
* Table/line-item confidence is scored separately from header fields.

A better score model:

```text
CompositeScore =
  35% CriticalFieldConfidence
+ 20% NonCriticalFieldConfidence
+ 20% ValidationPassScore
+ 15% CompletenessScore
+ 10% ClassificationConfidence
```

And **any critical hard-fail rule should override the composite score**.

***

### Issue 7 — Bounding region handling was insufficient

The previous code stored:

```csharp
BoundingRegion = kvp.Value.BoundingRegions?.FirstOrDefault()?.ToString()
```

That is not production-grade. The review UI needs page number, polygon coordinates, spans, source content, model field name, canonical field name, and possibly cropped image evidence.

Document Intelligence APIs analyze document response states that the response contains pages, paragraphs, tables, styles, languages, documents, and top-level content with spans; it also describes layout and semantic response elements. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/analyze-document-response?view=doc-intel-4.0.0)

**What I would change:**

```csharp
public sealed record FieldEvidence
{
    public int PageNumber { get; init; }
    public IReadOnlyList<float> Polygon { get; init; } = [];
    public int SpanOffset { get; init; }
    public int SpanLength { get; init; }
    public string? SourceText { get; init; }
    public string? CroppedImageUri { get; init; }
}
```

Human review should show:

* extracted value,
* confidence,
* page number,
* highlighted region,
* OCR text around the field,
* validation error,
* correction input.

Without that, the human review queue becomes a data-entry form rather than an evidence-based review tool.

***

### Issue 8 — Pre-processing was listed but not implemented

Your architecture includes pre-processing: MIME/file signature detection, multi-page splitting, deskewing, denoising, contrast enhancement, orientation correction, duplicate detection, and barcode/QR detection. [Recommended technical design solution - AI Document Intelligence.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D\&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=6b920bbf-36e1-40d4-99ac-aade513e1655) explicitly lists those pre-processing responsibilities. [\[Recommende...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

The previous C# implementation skipped this. That is acceptable for a scaffold but not for production.

**What I would change:**

Add a dedicated pre-processing stage:

```csharp
public interface IDocumentPreprocessor
{
    Task<PreprocessingResult> PrepareAsync(
        RawDocumentReference input,
        CancellationToken ct);
}
```

The output should include:

* normalised file,
* true file type,
* page count,
* image quality score,
* detected rotation/orientation,
* duplicate hash,
* split document candidates,
* warnings.

This is important because model accuracy is often lost before OCR even starts.

***

### Issue 9 — Orchestration should be durable and stateful

The previous `DocumentOrchestrator` was a simple in-memory sequence. That is fine for explanation, but fragile for production. If classification succeeds and extraction fails, you need to resume, retry, or route to exception handling without losing state.

Your own integration architecture work in [OIC Custom Integration Layer - Technical Design.docx](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B781ABF99-40A9-43E6-9E48-77A9DE9966D6%7D\&file=OIC%20Custom%20Integration%20Layer%20-%20Technical%20Design.docx\&action=default\&mobileredirect=true\&DefaultItemOpen=1\&EntityRepresentationId=e85d496c-df3a-4f17-a072-ffd3e2254089) emphasises run IDs, stage logging, status tracking, and reprocess design for integration pipelines.  The same principle applies here. [\[OIC Custom...cal Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B781ABF99-40A9-43E6-9E48-77A9DE9966D6%7D&file=OIC%20Custom%20Integration%20Layer%20-%20Technical%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

**What I would change:**

Use a durable orchestration model:

```text
DocumentSubmitted
  → PreprocessRequested
  → ClassificationRequested
  → ExtractionRequested
  → MappingRequested
  → ValidationRequested
  → ReviewDecisionRequested
  → IntegrationPublishRequested
```

Each stage writes:

```text
DocumentProcessingRun
DocumentProcessingStageLog
DocumentProcessingError
DocumentFieldEvidence
DocumentReviewCorrection
```

This gives you auditability, replay, operational support, and reprocessing.

***

### Issue 10 — Output integration was too vague

The previous implementation ended with TODO comments for Dataverse/ERP/API integration. That is a major gap. The output stage should use a reliable integration pattern:

* canonical payload persisted first,
* outbox message created,
* integration worker publishes to target systems,
* retries are controlled,
* target response is logged,
* downstream IDs are captured,
* duplicate submission is prevented.

**What I would change:**

```csharp
public interface ICanonicalDocumentPublisher
{
    Task PublishAsync(
        CanonicalDocument document,
        IntegrationTarget target,
        CancellationToken ct);
}
```

And use an outbox table:

```text
OUTBOX_MESSAGE
- MessageId
- DocumentId
- TargetSystem
- PayloadJson
- Status
- AttemptCount
- LastError
- CreatedAtUtc
- ProcessedAtUtc
```

This is especially important if the target is ERP, TMS, WMS, Dataverse, or a transactional API.

***

## 5. If I built it differently, my revised target architecture would be this

I would restructure the platform into **nine production modules**.

```text
1. Ingestion Gateway
   - API upload
   - Blob drop
   - SharePoint connector
   - Email attachment intake
   - Source metadata capture

2. Document Store
   - Raw document archive
   - Normalised document copy
   - Page images / evidence crops
   - Retention and legal hold policy

3. Pre-processing Service
   - File validation
   - Page splitting
   - Rotation/orientation
   - Duplicate detection
   - Image quality scoring

4. Document Intelligence Adapter
   - Classification call
   - Layout call
   - Extraction call
   - SDK-version isolation
   - Retry and throttling handling

5. Model + Schema Registry
   - Document type definitions
   - Model ID bindings
   - Schema version
   - Required fields
   - Source-to-canonical mappings
   - Field-level thresholds

6. Canonical Mapping Engine
   - Native DI response → canonical contract
   - Header fields
   - Line items
   - Evidence preservation
   - Schema version stamping

7. Validation + Scoring Engine
   - Structural rules
   - Field rules
   - Cross-field rules
   - Critical-field gating
   - Composite confidence
   - Deterministic routing

8. Human Review Workbench
   - Queue
   - Evidence highlighting
   - Correction capture
   - Reviewer reason codes
   - Feedback dataset generation

9. Integration + Observability Layer
   - Dataverse / SQL / ERP / TMS / WMS publishers
   - Outbox pattern
   - Pipeline run logs
   - Application Insights / OpenTelemetry
   - Model performance dashboards
```

That design is more platform-like and less demo-like.

***

## 6. Specific optimisations I would recommend

### Optimisation 1 — Replace reflection-based mapping with compiled mapping plans

The previous mapper uses reflection. Reflection is acceptable in a sample, but I would not make it the core path for high-volume processing.

Better:

```csharp
public sealed class MappingPlan
{
    public string DocumentType { get; init; } = "";
    public string SchemaVersion { get; init; } = "";
    public IReadOnlyList<FieldMappingRule> FieldMappings { get; init; } = [];
}
```

Compile mapping rules at startup or cache them by `documentType + schemaVersion`.

***

### Optimisation 2 — Add idempotency and duplicate detection

Every document should have:

```text
contentHash
sourceSystem
sourceDocumentId
schemaVersion
classifierModelId
extractionModelId
processingRunId
```

Before processing:

```csharp
if (await _documentRegistry.AlreadyProcessedAsync(contentHash, sourceSystem))
{
    return await _documentRegistry.GetExistingResultAsync(contentHash, sourceSystem);
}
```

This prevents duplicate ERP submissions and duplicate human review workload.

***

### Optimisation 3 — Add model version governance

The previous example uses static model IDs:

```json
"ClassificationModelId": "logistics-doc-classifier-v2"
```

That is not enough. Use model registry records:

```json
{
  "documentType": "DeliverySlip",
  "environment": "prod",
  "classifierModelId": "classifier-logistics-2026-06",
  "extractorModelId": "extractor-delivery-slip-2026-06",
  "schemaVersion": "1.2.0",
  "status": "Active",
  "effectiveFromUtc": "2026-06-01T00:00:00Z"
}
```

This enables rollback, A/B testing, and audit.

***

### Optimisation 4 — Use critical-field gating before composite score

A composite score can hide fatal errors. Example:

* 20 non-critical fields are high confidence.
* `shipmentNumber` is missing.
* Composite score still looks high.

So routing should be:

```text
Step 1: hard reject / review if critical fields fail
Step 2: apply validation hard-fail rules
Step 3: compute composite score
Step 4: route by threshold
```

This is safer than score-only routing.

***

### Optimisation 5 — Separate operational exception from business exception

The previous code has only general exception handling. I would distinguish:

| Exception type              | Example                                   | Handling                           |
| --------------------------- | ----------------------------------------- | ---------------------------------- |
| Technical failure           | SDK timeout, Blob unavailable, throttling | Retry / dead-letter                |
| Document quality failure    | unreadable scan, unsupported format       | Human review / reject              |
| Business validation failure | missing shipment number                   | Human review                       |
| Integration failure         | ERP API rejects payload                   | Integration retry / support queue  |
| Governance failure          | unknown schema version                    | Stop pipeline, configuration alert |

This improves supportability.

***

### Optimisation 6 — Externalise validation rules

Instead of hard-coded C# validation, define rules in JSON/YAML/DMN-style metadata.

Example:

```yaml
ruleId: DELSLIP_001
documentType: DeliverySlip
severity: error
category: Field
field: shipmentNumber
condition: required
message: Shipment number is required for delivery slips.
```

Then the C# engine evaluates rules. This lets business analysts and solution specialists review the rule catalogue without reading code.

***

### Optimisation 7 — Add a proper continuous improvement loop

Your architecture discussions already include continuous improvement as a later phase. The implementation should capture:

* original value,
* corrected value,
* reviewer ID,
* reason code,
* confidence before correction,
* model ID/version,
* schema version,
* source document sample,
* whether correction is eligible for retraining.

This becomes the data foundation for retraining and regression testing.

***

## 7. Revised “minimum production-grade” component list

If I were setting an implementation baseline, I would say the platform is not production-ready until these components exist:

1. **Document ingestion gateway**
2. **Raw and processed document storage**
3. **Pre-processing service**
4. **Document Intelligence adapter**
5. **Classifier model registry**
6. **Extraction model registry**
7. **Canonical schema registry**
8. **Mapping engine**
9. **Validation rules engine**
10. **Confidence scoring engine**
11. **Human review queue/workbench**
12. **Correction feedback store**
13. **Outbox-based integration publisher**
14. **Pipeline run/stage observability store**
15. **Error/retry/dead-letter mechanism**
16. **Security/identity/secrets layer**
17. **Model evaluation and regression test harness**
18. **Deployment/version governance**

The previous C# example covered only parts of 4, 8, 9, 10, and 11.

***

## 8. The most important correction to the implementation

If I had to choose only one correction, it would be this:

> **Move schema, mappings, required fields, thresholds, model bindings, and validation rules out of C# code and into versioned registries.**

That single change improves:

* business analyst reviewability,
* PM change control,
* developer maintainability,
* model version governance,
* testing,
* confidence scoring accuracy,
* human review UX,
* downstream integration stability.

The second most important correction is:

> **Make the orchestrator durable and observable.**

Without persistent run/stage state, the platform will be difficult to support once documents fail halfway through the pipeline.

***

## 9. Final technical verdict

The previous C# implementation is a **strong teaching example** for explaining the v3 architecture to a mixed audience. It is not yet the implementation I would hand to a production team as-is.

### Keep

* Orchestrator + specialist skills pattern.
* Canonical document contract.
* Classification before extraction.
* Schema-based extraction.
* Deterministic validation.
* Confidence-based routing.
* Human-in-the-loop concept.

### Fix

* SDK isolation and exact version alignment.
* Multi-document classifier splitting.
* Blob-first ingestion.
* Schema registry.
* Model registry.
* Rules registry.
* Evidence model.
* Durable orchestration.
* Idempotency.
* Outbox integration.
* Observability.
* Security/managed identity.
* Continuous improvement loop.

### Reframe

Do not describe this as merely a **C# implementation of Document Intelligence**. It is better described as:

> **A governed enterprise document processing platform that uses Azure AI Document Intelligence as the extraction engine, but owns orchestration, schema, validation, review, integration, and continuous improvement.**

That framing is technically more accurate and will help business analysts, project managers, solution specialists, and developers understand where Microsoft’s service ends and where your enterprise platform begins.
