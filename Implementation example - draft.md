Based on your v3 Final Architecture documents, here is a comprehensive **C# implementation example** of the AI Document Processing Platform. I've structured this for your mixed audience — business analysts will understand the domain models and flow, project managers will see the pipeline stages, solution specialists will appreciate the architectural patterns, and developers can use this as a working scaffold.

***

## 🏗️ Architecture Recap (from v3 Final)

Your architecture defines an **Orchestrator + Specialist Skills** pattern with 8 pipeline stages: [\[Detailed t...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B42A57CFD-44CA-42F8-8600-55B4FD2DC0AD%7D&file=Detailed%20technical%20architecture%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Recommende...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

| Stage                   | Skill                         | Azure Feature                      |
| ----------------------- | ----------------------------- | ---------------------------------- |
| 1. Ingestion            | File Detection                | Azure Blob / SharePoint            |
| 2. Pre-processing       | Image Enhancement             | Custom code                        |
| 3. OCR + Layout         | Layout Analysis               | Document Intelligence Layout Model |
| 4. Classification       | Document Type Detection       | Custom Classification Model        |
| 5. Extraction           | Schema-based Field Extraction | Custom Extraction / Prebuilt Model |
| 6. Schema Mapping       | Canonical Contract Transform  | **You build this**                 |
| 7. Validation & Scoring | Rules Engine + Confidence     | **You build this**                 |
| 8. Output & Integration | ERP/Dataverse/API Push        | **You build this**                 |

***

## 📁 Solution Structure

```
DocumentAI.Platform/
├── DocumentAI.Platform.sln
├── src/
│   ├── DocumentAI.Core/                  # Domain models, interfaces, enums
│   │   ├── Models/
│   │   │   ├── CanonicalDocument.cs
│   │   │   ├── ExtractionField.cs
│   │   │   ├── ValidationResult.cs
│   │   │   ├── ConfidenceScore.cs
│   │   │   └── ProcessingContext.cs
│   │   ├── Enums/
│   │   │   ├── DocumentType.cs
│   │   │   └── ReviewDecision.cs
│   │   └── Interfaces/
│   │       ├── IClassificationSkill.cs
│   │       ├── IExtractionSkill.cs
│   │       ├── IValidationSkill.cs
│   │       └── IOrchestrator.cs
│   ├── DocumentAI.Skills/               # Specialist skill implementations
│   │   ├── ClassificationSkill.cs
│   │   ├── ExtractionSkill.cs
│   │   ├── SchemaMappingSkill.cs
│   │   ├── ValidationSkill.cs
│   │   ├── ConfidenceScoringSkill.cs
│   │   └── ReviewDecisionSkill.cs
│   ├── DocumentAI.Orchestrator/         # Pipeline orchestration
│   │   └── DocumentOrchestrator.cs
│   └── DocumentAI.Api/                  # Azure Function / Web API host
│       ├── Functions/
│       │   └── ProcessDocumentFunction.cs
│       └── Program.cs
└── tests/
    └── DocumentAI.Tests/
```

***

## 1️⃣ Domain Models (Core Layer)

These models implement the **canonical, application-owned contract** that sits between OCR/extraction and storage/validation/integration — a key v3 principle. [\[Document T...ine Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Enums/DocumentType.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Enums;

/// <summary>
/// Business document classes supported by the classification model.
/// Maps 1:1 to your custom classification model's trained classes.
/// </summary>
public enum DocumentType
{
    Unknown = 0,
    DeliverySlip,
    Invoice,
    ProofOfDelivery,
    BillOfLading,
    PackingList,
    PurchaseOrder,
    Receipt,
    QualityCertificate
}

// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Enums/ReviewDecision.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Enums;

public enum ReviewDecision
{
    AutoApproved,       // All fields above threshold — straight-through
    SentToReview,       // One or more fields below threshold
    Rejected,           // Critical validation failure
    ManualOverride      // Reviewer corrected and approved
}
```

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Models/ExtractionField.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Models;

/// <summary>
/// Canonical field envelope — wraps every extracted value with
/// provenance, confidence, and validation metadata.
/// This is the "canonical field envelope" from the v3 design.
/// </summary>
public sealed record ExtractionField
{
    public string FieldName { get; init; } = string.Empty;
    public string? Value { get; init; }
    public double Confidence { get; init; }          // 0.0–1.0 from Document Intelligence
    public bool IsRequired { get; init; }
    public string? BoundingRegion { get; init; }      // Page coordinates for review UI
    public int PageNumber { get; init; }
    public string? RawModelOutput { get; init; }      // Original DI field key
    public bool PassedValidation { get; init; } = true;
    public string? ValidationMessage { get; init; }
}
```

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Models/CanonicalDocument.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Models;

using DocumentAI.Core.Enums;

/// <summary>
/// The canonical, application-owned document contract.
/// This is NOT the native Azure DI response — it is the standardised
/// business schema that sits AFTER extraction and BEFORE storage,
/// validation, review, and downstream integration.
/// </summary>
public sealed class CanonicalDocument
{
    // ── Identity ──────────────────────────────────────────
    public string DocumentId { get; set; } = Guid.NewGuid().ToString();
    public string SourceFileName { get; set; } = string.Empty;
    public string SourceBlobUri { get; set; } = string.Empty;
    public DateTime ProcessedAtUtc { get; set; } = DateTime.UtcNow;

    // ── Classification ────────────────────────────────────
    public DocumentType DocumentType { get; set; } = DocumentType.Unknown;
    public double ClassificationConfidence { get; set; }

    // ── Business Fields (schema-driven) ───────────────────
    public string? IssuerName { get; set; }
    public string? OrderNumber { get; set; }
    public DateTime? OrderDate { get; set; }
    public string? ShipmentNumber { get; set; }
    public string? SoldToName { get; set; }
    public string? ShipToName { get; set; }
    public string? CarrierName { get; set; }
    public string? BranchLocation { get; set; }
    public bool SignaturesPresent { get; set; }

    // ── Line Items ────────────────────────────────────────
    public List<LineItem> LineItems { get; set; } = [];

    // ── All extracted fields with metadata ────────────────
    public List<ExtractionField> Fields { get; set; } = [];

    // ── Scoring & Routing ─────────────────────────────────
    public ConfidenceScore? OverallScore { get; set; }
    public ReviewDecision ReviewDecision { get; set; }
    public List<ValidationResult> ValidationResults { get; set; } = [];
}

public sealed record LineItem
{
    public string? ProductDescription { get; init; }
    public string? ProductCode { get; init; }
    public decimal? Quantity { get; init; }
    public string? UnitOfMeasure { get; init; }
    public decimal? UnitPrice { get; init; }
    public decimal? LineTotal { get; init; }
    public double Confidence { get; init; }
}
```

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Models/ConfidenceScore.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Models;

/// <summary>
/// Factual confidence scoring — grounded in what was extracted,
/// NOT invented by an LLM. Three-component scoring model from v3.
/// </summary>
public sealed record ConfidenceScore
{
    public double ExtractionScore { get; init; }      // Weighted avg of field confidences
    public double ValidationScore { get; init; }      // % of rules passed
    public double CompletenessScore { get; init; }    // % of required fields present

    /// <summary>
    /// Composite score: 50% extraction + 30% validation + 20% completeness.
    /// Thresholds: ≥ 0.85 = auto-approve, < 0.85 = human review.
    /// </summary>
    public double CompositeScore =>
        (ExtractionScore * 0.50) +
        (ValidationScore * 0.30) +
        (CompletenessScore * 0.20);
}

// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Models/ValidationResult.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Models;

public sealed record ValidationResult
{
    public string RuleName { get; init; } = string.Empty;
    public string Category { get; init; } = string.Empty;  // Structural | Field | CrossField
    public bool Passed { get; init; }
    public string Message { get; init; } = string.Empty;
    public string? FieldName { get; init; }
}
```

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Models/ProcessingContext.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Models;

/// <summary>
/// Carries state through the pipeline — the orchestrator passes
/// this context to each specialist skill.
/// </summary>
public sealed class ProcessingContext
{
    public string CorrelationId { get; set; } = Guid.NewGuid().ToString();
    public Stream DocumentStream { get; set; } = Stream.Null;
    public string FileName { get; set; } = string.Empty;
    public string ContentType { get; set; } = string.Empty;
    public CanonicalDocument Result { get; set; } = new();
    public List<string> ProcessingLog { get; set; } = [];
    public bool HasCriticalFailure { get; set; }

    public void Log(string message)
        => ProcessingLog.Add($"[{DateTime.UtcNow:HH:mm:ss.fff}] {message}");
}
```

***

## 2️⃣ Interfaces (Skill Contracts)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Core/Interfaces/
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Core.Interfaces;

using DocumentAI.Core.Models;

public interface IClassificationSkill
{
    Task<ProcessingContext> ClassifyAsync(ProcessingContext context);
}

public interface IExtractionSkill
{
    Task<ProcessingContext> ExtractAsync(ProcessingContext context);
}

public interface ISchemaMappingSkill
{
    ProcessingContext MapToCanonical(ProcessingContext context);
}

public interface IValidationSkill
{
    ProcessingContext Validate(ProcessingContext context);
}

public interface IConfidenceScoringSkill
{
    ProcessingContext Score(ProcessingContext context);
}

public interface IReviewDecisionSkill
{
    ProcessingContext Decide(ProcessingContext context);
}

public interface IDocumentOrchestrator
{
    Task<CanonicalDocument> ProcessAsync(Stream document, string fileName, string contentType);
}
```

***

## 3️⃣ Specialist Skills (Implementation Layer)

### Skill 1 — Classification [\[Designing...zed skills \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/ClassificationSkill.cs
// ═══════════════════════════════════════════════════════════
using Azure;
using Azure.AI.DocumentIntelligence;
using DocumentAI.Core.Enums;
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace DocumentAI.Skills;

public sealed class ClassificationSkill : IClassificationSkill
{
    private readonly DocumentIntelligenceClient _client;
    private readonly DocumentAIOptions _options;
    private readonly ILogger<ClassificationSkill> _logger;

    public ClassificationSkill(
        DocumentIntelligenceClient client,
        IOptions<DocumentAIOptions> options,
        ILogger<ClassificationSkill> logger)
    {
        _client  = client;
        _options = options.Value;
        _logger  = logger;
    }

    public async Task<ProcessingContext> ClassifyAsync(ProcessingContext context)
    {
        context.Log("Classification skill started.");

        // ── Call the custom classification model ──────────
        // The model ID matches what you trained in Document Intelligence Studio.
        var content = new ClassifyDocumentContent(
            BinaryData.FromStream(context.DocumentStream));

        Operation<AnalyzeResult> operation = await _client.ClassifyDocumentAsync(
            WaitUntil.Completed,
            _options.ClassificationModelId,   // e.g. "logistics-doc-classifier-v2"
            content);

        AnalyzeResult result = operation.Value;

        if (result.Documents is null || result.Documents.Count == 0)
        {
            context.Log("⚠ Classification returned no documents. Marking as Unknown.");
            context.Result.DocumentType = DocumentType.Unknown;
            context.Result.ClassificationConfidence = 0.0;
            return context;
        }

        // ── Map the model's docType string to our enum ───
        var topDoc = result.Documents
            .OrderByDescending(d => d.Confidence)
            .First();

        context.Result.DocumentType = MapDocType(topDoc.DocumentType);
        context.Result.ClassificationConfidence = topDoc.Confidence ?? 0.0;

        context.Log($"Classified as {context.Result.DocumentType} " +
                     $"(confidence: {context.Result.ClassificationConfidence:P1}).");

        // Reset stream position for the next skill
        context.DocumentStream.Position = 0;
        return context;
    }

    /// <summary>
    /// Maps the string label from Document Intelligence Studio
    /// to the application's canonical DocumentType enum.
    /// </summary>
    private static DocumentType MapDocType(string? modelLabel) => modelLabel?.ToLowerInvariant() switch
    {
        "delivery_slip"       => DocumentType.DeliverySlip,
        "invoice"             => DocumentType.Invoice,
        "proof_of_delivery"   => DocumentType.ProofOfDelivery,
        "bill_of_lading"      => DocumentType.BillOfLading,
        "packing_list"        => DocumentType.PackingList,
        "purchase_order"      => DocumentType.PurchaseOrder,
        "receipt"             => DocumentType.Receipt,
        "quality_certificate" => DocumentType.QualityCertificate,
        _                     => DocumentType.Unknown
    };
}
```

### Skill 2 — Extraction [\[Document T...g Platform \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/ExtractionSkill.cs
// ═══════════════════════════════════════════════════════════
using Azure;
using Azure.AI.DocumentIntelligence;
using DocumentAI.Core.Enums;
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace DocumentAI.Skills;

public sealed class ExtractionSkill : IExtractionSkill
{
    private readonly DocumentIntelligenceClient _client;
    private readonly DocumentAIOptions _options;
    private readonly ILogger<ExtractionSkill> _logger;

    public ExtractionSkill(
        DocumentIntelligenceClient client,
        IOptions<DocumentAIOptions> options,
        ILogger<ExtractionSkill> logger)
    {
        _client  = client;
        _options = options.Value;
        _logger  = logger;
    }

    public async Task<ProcessingContext> ExtractAsync(ProcessingContext context)
    {
        context.Log("Extraction skill started.");

        // ── Select the right model based on classification ──
        string modelId = ResolveModelId(context.Result.DocumentType);
        context.Log($"Using model: {modelId}");

        var content = new AnalyzeDocumentContent(
            BinaryData.FromStream(context.DocumentStream));

        Operation<AnalyzeResult> operation = await _client.AnalyzeDocumentAsync(
            WaitUntil.Completed,
            modelId,
            content);

        AnalyzeResult result = operation.Value;

        // ── Convert DI fields into ExtractionField envelopes ──
        if (result.Documents is { Count: > 0 })
        {
            var doc = result.Documents[0];
            foreach (var kvp in doc.Fields)
            {
                context.Result.Fields.Add(new ExtractionField
                {
                    FieldName      = kvp.Key,
                    Value          = kvp.Value.Content,
                    Confidence     = kvp.Value.Confidence ?? 0.0,
                    RawModelOutput = kvp.Key,
                    BoundingRegion = kvp.Value.BoundingRegions?.FirstOrDefault()?.ToString(),
                    PageNumber     = kvp.Value.BoundingRegions?.FirstOrDefault()?.PageNumber ?? 1
                });
            }
        }

        // ── Also extract tables for line items ──
        if (result.Tables is { Count: > 0 })
        {
            foreach (var table in result.Tables)
            {
                context.Log($"Table found: {table.RowCount} rows × {table.ColumnCount} cols on page {table.BoundingRegions?.FirstOrDefault()?.PageNumber}");
                // Table-to-LineItem mapping is handled by SchemaMappingSkill
            }
        }

        context.Log($"Extracted {context.Result.Fields.Count} fields.");
        context.DocumentStream.Position = 0;
        return context;
    }

    /// <summary>
    /// Model selection rule from the v3 architecture:
    ///   • Standard invoices → Prebuilt Invoice model
    ///   • Semi-structured / business-specific → Custom Neural Extraction
    ///   • Unknown layout → Layout Model with key-value extraction
    /// </summary>
    private string ResolveModelId(DocumentType docType) => docType switch
    {
        DocumentType.Invoice => "prebuilt-invoice",                     // Microsoft prebuilt
        DocumentType.Receipt => "prebuilt-receipt",                     // Microsoft prebuilt
        DocumentType.DeliverySlip       => _options.CustomExtractionModelId,  // Your custom model
        DocumentType.ProofOfDelivery    => _options.CustomExtractionModelId,
        DocumentType.BillOfLading       => _options.CustomExtractionModelId,
        DocumentType.PackingList        => _options.CustomExtractionModelId,
        DocumentType.PurchaseOrder      => _options.CustomExtractionModelId,
        DocumentType.QualityCertificate => _options.CustomExtractionModelId,
        _                               => "prebuilt-layout"           // Fallback
    };
}
```

### Skill 3 — Schema Mapping (you build this) [\[Document T...ine Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/SchemaMappingSkill.cs
// ═══════════════════════════════════════════════════════════
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;
using Microsoft.Extensions.Logging;

namespace DocumentAI.Skills;

/// <summary>
/// Transforms raw Document Intelligence field keys into the
/// canonical, application-owned schema. This is where the
/// "mismatch" between DI output and your business contract
/// gets resolved.
/// </summary>
public sealed class SchemaMappingSkill : ISchemaMappingSkill
{
    private readonly ILogger<SchemaMappingSkill> _logger;

    // ── Field mapping dictionary ──────────────────────────
    // Left  = Document Intelligence field key (from training labels)
    // Right = Canonical schema property name
    private static readonly Dictionary<string, string> FieldMap = new(StringComparer.OrdinalIgnoreCase)
    {
        ["VendorName"]           = nameof(CanonicalDocument.IssuerName),
        ["InvoiceId"]            = nameof(CanonicalDocument.OrderNumber),
        ["order_number"]         = nameof(CanonicalDocument.OrderNumber),
        ["InvoiceDate"]          = nameof(CanonicalDocument.OrderDate),
        ["order_date"]           = nameof(CanonicalDocument.OrderDate),
        ["shipment_number"]      = nameof(CanonicalDocument.ShipmentNumber),
        ["CustomerName"]         = nameof(CanonicalDocument.SoldToName),
        ["sold_to"]              = nameof(CanonicalDocument.SoldToName),
        ["ShippingAddress"]      = nameof(CanonicalDocument.ShipToName),
        ["ship_to"]              = nameof(CanonicalDocument.ShipToName),
        ["carrier"]              = nameof(CanonicalDocument.CarrierName),
        ["branch"]               = nameof(CanonicalDocument.BranchLocation),
    };

    public SchemaMappingSkill(ILogger<SchemaMappingSkill> logger) => _logger = logger;

    public ProcessingContext MapToCanonical(ProcessingContext context)
    {
        context.Log("Schema mapping skill started.");
        var doc = context.Result;

        foreach (var field in doc.Fields)
        {
            if (!FieldMap.TryGetValue(field.FieldName, out var canonicalName))
            {
                _logger.LogDebug("Unmapped field: {Field}", field.FieldName);
                continue;
            }

            // Use reflection or a switch to set the canonical property
            SetCanonicalProperty(doc, canonicalName, field.Value);
        }

        // ── Map line items from table extraction ──────────
        var lineItemFields = doc.Fields
            .Where(f => f.FieldName.StartsWith("Items.", StringComparison.OrdinalIgnoreCase))
            .GroupBy(f => ExtractLineIndex(f.FieldName));

        foreach (var group in lineItemFields)
        {
            doc.LineItems.Add(new LineItem
            {
                ProductDescription = group.FirstOrDefault(f => f.FieldName.EndsWith("Description"))?.Value,
                ProductCode        = group.FirstOrDefault(f => f.FieldName.EndsWith("ProductCode"))?.Value,
                Quantity           = ParseDecimal(group.FirstOrDefault(f => f.FieldName.EndsWith("Quantity"))?.Value),
                UnitOfMeasure      = group.FirstOrDefault(f => f.FieldName.EndsWith("Unit"))?.Value,
                UnitPrice          = ParseDecimal(group.FirstOrDefault(f => f.FieldName.EndsWith("UnitPrice"))?.Value),
                LineTotal          = ParseDecimal(group.FirstOrDefault(f => f.FieldName.EndsWith("Amount"))?.Value),
                Confidence         = group.Average(f => f.Confidence)
            });
        }

        context.Log($"Mapped {doc.LineItems.Count} line items to canonical schema.");
        return context;
    }

    private static void SetCanonicalProperty(CanonicalDocument doc, string propName, string? value)
    {
        var prop = typeof(CanonicalDocument).GetProperty(propName);
        if (prop is null || value is null) return;

        if (prop.PropertyType == typeof(DateTime?) && DateTime.TryParse(value, out var dt))
            prop.SetValue(doc, dt);
        else if (prop.PropertyType == typeof(string))
            prop.SetValue(doc, value);
    }

    private static int ExtractLineIndex(string fieldName)
    {
        // "Items.0.Description" → 0
        var parts = fieldName.Split('.');
        return parts.Length > 1 && int.TryParse(parts[1], out var idx) ? idx : 0;
    }

    private static decimal? ParseDecimal(string? value)
        => decimal.TryParse(value, out var d) ? d : null;
}
```

### Skill 4 — Validation (you build this) [\[Recommende...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/ValidationSkill.cs
// ═══════════════════════════════════════════════════════════
using DocumentAI.Core.Enums;
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;
using Microsoft.Extensions.Logging;

namespace DocumentAI.Skills;

/// <summary>
/// Applies three categories of validation rules from the v3 design:
///   A) Structural validation  — document-level checks
///   B) Field validation       — per-field format/range checks
///   C) Cross-field / business — inter-field logic checks
/// </summary>
public sealed class ValidationSkill : IValidationSkill
{
    private readonly ILogger<ValidationSkill> _logger;

    public ValidationSkill(ILogger<ValidationSkill> logger) => _logger = logger;

    public ProcessingContext Validate(ProcessingContext context)
    {
        context.Log("Validation skill started.");
        var doc = context.Result;
        var results = new List<ValidationResult>();

        // ═══ A) STRUCTURAL VALIDATION ═════════════════════
        results.Add(ValidateStructural(
            "DOC_TYPE_KNOWN",
            doc.DocumentType != DocumentType.Unknown,
            "Document type must be classified before validation."));

        results.Add(ValidateStructural(
            "FIELDS_PRESENT",
            doc.Fields.Count > 0,
            "At least one field must be extracted."));

        // ═══ B) FIELD VALIDATION ══════════════════════════
        results.Add(ValidateField(
            "ORDER_NUM_FORMAT",
            "OrderNumber",
            !string.IsNullOrWhiteSpace(doc.OrderNumber) &&
            doc.OrderNumber.Length >= 4,
            "Order number must be at least 4 characters."));

        results.Add(ValidateField(
            "SHIPMENT_REQUIRED_FOR_DELIVERY",
            "ShipmentNumber",
            doc.DocumentType != DocumentType.DeliverySlip ||
            !string.IsNullOrWhiteSpace(doc.ShipmentNumber),
            "Shipment number is required for delivery slips."));

        results.Add(ValidateField(
            "SHIP_TO_NOT_BLANK",
            "ShipToName",
            !string.IsNullOrWhiteSpace(doc.ShipToName),
            "Ship-to name should not be blank."));

        // Validate each line item quantity is numeric and positive
        for (int i = 0; i < doc.LineItems.Count; i++)
        {
            results.Add(ValidateField(
                $"LINE_{i}_QTY_NUMERIC",
                $"LineItems[{i}].Quantity",
                doc.LineItems[i].Quantity is > 0,
                $"Line item {i} quantity must be a positive number."));
        }

        // ═══ C) CROSS-FIELD / BUSINESS VALIDATION ════════
        results.Add(ValidateCrossField(
            "DELIVERY_HAS_PRODUCT",
            doc.DocumentType != DocumentType.DeliverySlip ||
            doc.LineItems.Count > 0,
            "If document type = delivery slip, product section must exist."));

        results.Add(ValidateCrossField(
            "SIGNATURE_WITH_RECEIPT",
            !doc.SignaturesPresent ||
            doc.DocumentType is DocumentType.ProofOfDelivery or DocumentType.DeliverySlip,
            "Signature present but document type does not typically require signatures."));

        doc.ValidationResults = results;
        int passed = results.Count(r => r.Passed);
        context.Log($"Validation complete: {passed}/{results.Count} rules passed.");

        return context;
    }

    // ── Helper methods for rule construction ──────────────
    private static ValidationResult ValidateStructural(string name, bool passed, string msg) => new()
    {
        RuleName = name, Category = "Structural", Passed = passed,
        Message = passed ? "OK" : msg
    };

    private static ValidationResult ValidateField(string name, string field, bool passed, string msg) => new()
    {
        RuleName = name, Category = "Field", Passed = passed,
        FieldName = field, Message = passed ? "OK" : msg
    };

    private static ValidationResult ValidateCrossField(string name, bool passed, string msg) => new()
    {
        RuleName = name, Category = "CrossField", Passed = passed,
        Message = passed ? "OK" : msg
    };
}
```

### Skill 5 — Confidence Scoring [\[Document T...g Platform \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Document T...ine Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/ConfidenceScoringSkill.cs
// ═══════════════════════════════════════════════════════════
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;

namespace DocumentAI.Skills;

/// <summary>
/// Factual confidence scoring — grounded in what was extracted.
/// Three-component model:
///   • Extraction  (50%) — weighted average of DI field confidences
///   • Validation  (30%) — percentage of business rules passed
///   • Completeness(20%) — percentage of required fields present
/// </summary>
public sealed class ConfidenceScoringSkill : IConfidenceScoringSkill
{
    public ProcessingContext Score(ProcessingContext context)
    {
        context.Log("Confidence scoring skill started.");
        var doc = context.Result;

        // ── Extraction score ──────────────────────────────
        double extractionScore = doc.Fields.Count > 0
            ? doc.Fields.Average(f => f.Confidence)
            : 0.0;

        // ── Validation score ──────────────────────────────
        double validationScore = doc.ValidationResults.Count > 0
            ? (double)doc.ValidationResults.Count(r => r.Passed) / doc.ValidationResults.Count
            : 0.0;

        // ── Completeness score ────────────────────────────
        var requiredFields = doc.Fields.Where(f => f.IsRequired).ToList();
        double completenessScore = requiredFields.Count > 0
            ? (double)requiredFields.Count(f => !string.IsNullOrWhiteSpace(f.Value)) / requiredFields.Count
            : 1.0; // If no required fields defined, assume complete

        doc.OverallScore = new ConfidenceScore
        {
            ExtractionScore   = Math.Round(extractionScore, 4),
            ValidationScore   = Math.Round(validationScore, 4),
            CompletenessScore = Math.Round(completenessScore, 4)
        };

        context.Log($"Scores → Extraction: {doc.OverallScore.ExtractionScore:P1}, " +
                     $"Validation: {doc.OverallScore.ValidationScore:P1}, " +
                     $"Completeness: {doc.OverallScore.CompletenessScore:P1}, " +
                     $"Composite: {doc.OverallScore.CompositeScore:P1}");

        return context;
    }
}
```

### Skill 6 — Review Decision

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/ReviewDecisionSkill.cs
// ═══════════════════════════════════════════════════════════
using DocumentAI.Core.Enums;
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;
using Microsoft.Extensions.Options;

namespace DocumentAI.Skills;

/// <summary>
/// Threshold-based routing:
///   ≥ 0.85 composite  → AutoApproved  (straight-through processing)
///   ≥ 0.50 composite  → SentToReview  (human review queue)
///   &lt; 0.50 composite  → Rejected     (critical failure)
/// </summary>
public sealed class ReviewDecisionSkill : IReviewDecisionSkill
{
    private readonly DocumentAIOptions _options;

    public ReviewDecisionSkill(IOptions<DocumentAIOptions> options)
        => _options = options.Value;

    public ProcessingContext Decide(ProcessingContext context)
    {
        context.Log("Review decision skill started.");
        var score = context.Result.OverallScore?.CompositeScore ?? 0.0;

        context.Result.ReviewDecision = score switch
        {
            >= 0.85 => ReviewDecision.AutoApproved,
            >= 0.50 => ReviewDecision.SentToReview,
            _       => ReviewDecision.Rejected
        };

        context.Log($"Decision: {context.Result.ReviewDecision} (composite: {score:P1}).");

        // Flag critical failure for the orchestrator
        if (context.Result.ReviewDecision == ReviewDecision.Rejected)
            context.HasCriticalFailure = true;

        return context;
    }
}
```

***

## 4️⃣ Orchestrator (Pipeline Controller) [\[Designing...zed skills \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Orchestrator/DocumentOrchestrator.cs
// ═══════════════════════════════════════════════════════════
using DocumentAI.Core.Interfaces;
using DocumentAI.Core.Models;
using Microsoft.Extensions.Logging;

namespace DocumentAI.Orchestrator;

/// <summary>
/// The Orchestrator Agent — controls flow, selects models, routes decisions.
/// This is NOT an LLM agent; it is deterministic pipeline orchestration
/// that coordinates specialist skills in sequence.
/// 
/// Pipeline: Classify → Extract → Map → Validate → Score → Decide
/// </summary>
public sealed class DocumentOrchestrator : IDocumentOrchestrator
{
    private readonly IClassificationSkill    _classifier;
    private readonly IExtractionSkill        _extractor;
    private readonly ISchemaMappingSkill     _mapper;
    private readonly IValidationSkill        _validator;
    private readonly IConfidenceScoringSkill _scorer;
    private readonly IReviewDecisionSkill    _reviewer;
    private readonly ILogger<DocumentOrchestrator> _logger;

    public DocumentOrchestrator(
        IClassificationSkill    classifier,
        IExtractionSkill        extractor,
        ISchemaMappingSkill     mapper,
        IValidationSkill        validator,
        IConfidenceScoringSkill scorer,
        IReviewDecisionSkill    reviewer,
        ILogger<DocumentOrchestrator> logger)
    {
        _classifier = classifier;
        _extractor  = extractor;
        _mapper     = mapper;
        _validator  = validator;
        _scorer     = scorer;
        _reviewer   = reviewer;
        _logger     = logger;
    }

    public async Task<CanonicalDocument> ProcessAsync(
        Stream document, string fileName, string contentType)
    {
        var context = new ProcessingContext
        {
            DocumentStream = document,
            FileName       = fileName,
            ContentType    = contentType
        };

        context.Result.SourceFileName = fileName;
        context.Log($"Pipeline started for: {fileName}");

        try
        {
            // ── Stage 1-2: Ingestion & pre-processing ────
            // (handled upstream — Blob trigger / API gateway)
            context.Log("Document received from ingestion layer.");

            // ── Stage 3-4: OCR + Classification ──────────
            context = await _classifier.ClassifyAsync(context);
            if (context.HasCriticalFailure) return context.Result;

            // ── Stage 5: Schema-based Extraction ─────────
            context = await _extractor.ExtractAsync(context);
            if (context.HasCriticalFailure) return context.Result;

            // ── Stage 6: Canonical Schema Mapping ────────
            context = _mapper.MapToCanonical(context);

            // ── Stage 7a: Validation Rules ───────────────
            context = _validator.Validate(context);

            // ── Stage 7b: Confidence Scoring ─────────────
            context = _scorer.Score(context);

            // ── Stage 7c: Review Decision ────────────────
            context = _reviewer.Decide(context);

            context.Log("Pipeline completed successfully.");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Pipeline failed for {FileName}", fileName);
            context.Log($"❌ PIPELINE ERROR: {ex.Message}");
            context.HasCriticalFailure = true;
        }

        return context.Result;
    }
}
```

***

## 5️⃣ Azure Function Host (API Entry Point)

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Api/Functions/ProcessDocumentFunction.cs
// ═══════════════════════════════════════════════════════════
using System.Net;
using DocumentAI.Core.Enums;
using DocumentAI.Core.Interfaces;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;

namespace DocumentAI.Api.Functions;

public sealed class ProcessDocumentFunction
{
    private readonly IDocumentOrchestrator _orchestrator;
    private readonly ILogger<ProcessDocumentFunction> _logger;

    public ProcessDocumentFunction(
        IDocumentOrchestrator orchestrator,
        ILogger<ProcessDocumentFunction> logger)
    {
        _orchestrator = orchestrator;
        _logger       = logger;
    }

    /// <summary>
    /// HTTP trigger: POST /api/process-document
    /// Accepts multipart/form-data with a single file.
    /// </summary>
    [Function("ProcessDocument")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "process-document")]
        HttpRequestData req)
    {
        _logger.LogInformation("Document processing request received.");

        // Parse the uploaded file
        var formData = await req.ReadFormDataAsync();
        var file = req.Body; // Simplified — use multipart parser in production

        var result = await _orchestrator.ProcessAsync(
            file,
            formData?["fileName"] ?? "unknown.pdf",
            req.Headers.GetValues("Content-Type").FirstOrDefault() ?? "application/pdf");

        // Return structured result
        var response = req.CreateResponse(
            result.ReviewDecision == ReviewDecision.Rejected
                ? HttpStatusCode.UnprocessableEntity
                : HttpStatusCode.OK);

        await response.WriteAsJsonAsync(result);
        return response;
    }

    /// <summary>
    /// Blob trigger: auto-process documents dropped into the
    /// "incoming-documents" container.
    /// </summary>
    [Function("ProcessDocumentBlob")]
    public async Task RunBlob(
        [BlobTrigger("incoming-documents/{name}", Connection = "AzureStorageConnection")]
        Stream blobStream,
        string name)
    {
        _logger.LogInformation("Blob trigger fired for: {BlobName}", name);
        var result = await _orchestrator.ProcessAsync(blobStream, name, "application/pdf");

        _logger.LogInformation(
            "Result: {DocType} | Score: {Score:P1} | Decision: {Decision}",
            result.DocumentType,
            result.OverallScore?.CompositeScore,
            result.ReviewDecision);

        // TODO: Push to Dataverse / ERP / review queue based on ReviewDecision
    }
}
```

***

## 6️⃣ DI Registration & Configuration

```csharp
// ═══════════════════════════════════════════════════════════
// DocumentAI.Api/Program.cs
// ═══════════════════════════════════════════════════════════
using Azure;
using Azure.AI.DocumentIntelligence;
using DocumentAI.Core.Interfaces;
using DocumentAI.Orchestrator;
using DocumentAI.Skills;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices((hostContext, services) =>
    {
        var config = hostContext.Configuration;

        // ── Configuration ─────────────────────────────────
        services.Configure<DocumentAIOptions>(config.GetSection("DocumentAI"));

        // ── Azure Document Intelligence client ────────────
        services.AddSingleton(_ => new DocumentIntelligenceClient(
            new Uri(config["DocumentAI:Endpoint"]!),
            new AzureKeyCredential(config["DocumentAI:ApiKey"]!)));

        // ── Register specialist skills ────────────────────
        services.AddScoped<IClassificationSkill, ClassificationSkill>();
        services.AddScoped<IExtractionSkill, ExtractionSkill>();
        services.AddScoped<ISchemaMappingSkill, SchemaMappingSkill>();
        services.AddScoped<IValidationSkill, ValidationSkill>();
        services.AddScoped<IConfidenceScoringSkill, ConfidenceScoringSkill>();
        services.AddScoped<IReviewDecisionSkill, ReviewDecisionSkill>();

        // ── Register orchestrator ─────────────────────────
        services.AddScoped<IDocumentOrchestrator, DocumentOrchestrator>();
    })
    .Build();

await host.RunAsync();

// ═══════════════════════════════════════════════════════════
// DocumentAI.Skills/DocumentAIOptions.cs
// ═══════════════════════════════════════════════════════════
namespace DocumentAI.Skills;

public sealed class DocumentAIOptions
{
    public string Endpoint { get; set; } = string.Empty;
    public string ApiKey { get; set; } = string.Empty;
    public string ClassificationModelId { get; set; } = "logistics-doc-classifier-v2";
    public string CustomExtractionModelId { get; set; } = "logistics-extractor-v2";
    public double AutoApproveThreshold { get; set; } = 0.85;
    public double RejectThreshold { get; set; } = 0.50;
}
```

```json
// appsettings.json
{
  "DocumentAI": {
    "Endpoint": "https://<your-resource>.cognitiveservices.azure.com/",
    "ApiKey": "→ Use Azure Key Vault in production ←",
    "ClassificationModelId": "logistics-doc-classifier-v2",
    "CustomExtractionModelId": "logistics-extractor-v2",
    "AutoApproveThreshold": 0.85,
    "RejectThreshold": 0.50
  }
}
```

***

## 7️⃣ Sample Output (What the Pipeline Returns)

```json
{
  "documentId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "sourceFileName": "delivery-slip-2447651.pdf",
  "processedAtUtc": "2026-06-13T00:52:00Z",
  "documentType": "DeliverySlip",
  "classificationConfidence": 0.98,
  "orderNumber": "2447651",
  "shipmentNumber": "5284456",
  "soldToName": "Taiga Building Products",
  "shipToName": "Edmonton Warehouse #3",
  "carrierName": "TransX Ltd",
  "signaturesPresent": true,
  "lineItems": [
    {
      "productDescription": "2x4x8 SPF #2&Btr KD",
      "productCode": "LBR-2048-SPF",
      "quantity": 480,
      "unitOfMeasure": "PCS",
      "unitPrice": 3.45,
      "lineTotal": 1656.00,
      "confidence": 0.96
    }
  ],
  "overallScore": {
    "extractionScore": 0.9540,
    "validationScore": 0.8750,
    "completenessScore": 1.0000,
    "compositeScore": 0.9395
  },
  "reviewDecision": "AutoApproved",
  "validationResults": [
    { "ruleName": "DOC_TYPE_KNOWN",    "category": "Structural", "passed": true },
    { "ruleName": "ORDER_NUM_FORMAT",  "category": "Field",      "passed": true },
    { "ruleName": "SHIPMENT_REQUIRED", "category": "Field",      "passed": true },
    { "ruleName": "LINE_0_QTY_NUMERIC","category": "Field",      "passed": true },
    { "ruleName": "DELIVERY_HAS_PRODUCT","category": "CrossField","passed": true }
  ]
}
```

***

## 📊 Quick Reference for Your Audience

| Role                    | What to Focus On                                                                                                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Business Analyst**    | `CanonicalDocument` model — these are your business fields. `ValidationSkill` — these are your business rules.                              |
| **Project Manager**     | The 8-stage pipeline in `DocumentOrchestrator` — each stage is a trackable deliverable.                                                     |
| **Solution Specialist** | The Orchestrator + Specialist Skills pattern, model selection logic in `ExtractionSkill`, and the three-component confidence scoring model. |
| **Developer**           | Full DI registration in `Program.cs`, the `ExtractionField` envelope pattern, and the Blob/HTTP triggers for deployment.                    |

***

This implementation is directly grounded in your v3 Final Architecture's core principles: **schema-driven extraction**, **deterministic orchestration** (not LLM-based), **factual confidence scoring**, and **threshold-based human review routing**. [\[Detailed t...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7B42A57CFD-44CA-42F8-8600-55B4FD2DC0AD%7D&file=Detailed%20technical%20architecture%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Designing...zed skills \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BE0F10E3C-0554-459E-BCBE-30DBDB2459FD%7D&file=Designing%20a%20multi-agent%20document%20AI%20system%20with%20specialized%20skills.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Document T...g Platform \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFE04F539-01EF-48F8-B605-E3C87F6D88C3%7D&file=Document%20Type%20-%20JSON%20schema%20-%20AI%20Document%20Processing%20Platform.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Document T...ine Design \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BDDAB5EC2-DD1C-4093-A1A3-E2814A879C82%7D&file=Document%20Type%20-%20JSON%20schema%20-%20for%20AI%20Document%20Procssing%20Pipeline%20Design.docx&action=default&mobileredirect=true&DefaultItemOpen=1), [\[Recommende...telligence \| Word\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/_layouts/15/Doc.aspx?sourcedoc=%7BFC2926E8-3E73-4B7A-A81A-3A32851BAFAC%7D&file=Recommended%20technical%20design%20solution%20-%20AI%20Document%20Intelligence.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

Want me to drill deeper into any specific skill — for example, expanding the **Dataverse integration** for the output stage, or adding **unit tests** for the validation rules?
