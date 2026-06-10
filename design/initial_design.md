## Executive recommendation

Based on the attached [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) and [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb), I would recommend a **hybrid intelligent document processing strategy**: use **Document Intelligence Layout as the baseline OCR/layout parser**, then add a **custom extraction model** for delivery-slip/POD business fields, and add a **custom classification model only if the production intake contains multiple document types or multiple POD variants**. The reason is that these documents are not just plain OCR documents: they contain structured header sections, shipment/order fields, line-item tables, signatures, dates, handwritten exceptions, and variable scan/photo quality. [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) contains an INTERFOR Delivery Slip with order, shipping, product, carrier, signature and print-date sections, including extracted fields such as Order Number `2447651`, Shipment `5284456`, Branch/Location `DeQuincy / DQ`, Unit `Truck`, and Total `23.292 MBF`.  [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) shows the same overall business document family but with multiple pages, handwritten delivery/exception notes, and a page where the document includes the handwritten note “Wrong Size Lumber” plus a reference number `# 0013856`. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf) [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

***

## 1) Recommended IDP strategy and why

### Recommended target architecture

For this use case, I would design the process as:

1. **Document ingestion and image normalisation**
   * Accept PDF scans, mobile photos, and scanned images.
   * Apply pre-processing where needed: deskew, rotation correction, contrast enhancement, crop/edge detection, and page-quality validation.
   * The attached examples show both clean generated/scanned PDFs and lower-quality scanned/photo-like documents, so image quality handling is not optional. [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) includes visibly degraded scan quality and handwritten notes on page 1, while page 2 contains a structured delivery slip plus handwritten exception text. [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

2. **Document classification / routing**
   * If the intake contains only one document type — for example, only INTERFOR Delivery Slip / POD documents — classification can initially be rule-based or lightweight.
   * If the intake contains mixed documents — delivery slips, bills of lading, invoices, proof-of-delivery photos, exception forms, claims, carrier paperwork — use a **custom classification model** to identify the document type and route each page/document to the correct extractor.
   * Microsoft states that custom classification models identify document types before invoking extraction models and can be paired with custom extraction models. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0), [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)

3. **Field extraction**
   * Use a **custom extraction model**, preferably **custom neural** first, to extract business-specific fields:
     * Order Number
     * Order Date
     * Salesperson
     * Customer number
     * Sold To
     * Ship To
     * Carrier
     * Phone
     * Shipment
     * Branch/Location
     * Unit / Unit #
     * Restrictions
     * Product Description
     * Product code, such as `SY24#1`, `SY26#1`
     * Line-item table: Length, Units, Pieces, Pcs/Unit, Quantity, UOM
     * Sub-total and Total
     * Actual or Estimated Load Weight
     * Signature presence
     * Signature dates
     * Handwritten exception notes
     * Print Date/Time
     * Page number
   * These fields are visible across the attached delivery slips, but the content and quality vary between pages. [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) includes a clean single-page delivery slip with structured sections and signature/date areas.  [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) includes pages with OCR errors, handwritten notes, and exception text such as “Wrong Size Lumber”, which makes generic OCR alone insufficient for reliable automation. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf) [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

4. **Post-processing and business validation**
   * Apply deterministic validation rules after extraction:
     * Required fields present: Order Number, Shipment, Ship To, Total Quantity, UOM.
     * Numeric consistency: Sub-total equals Total where expected.
     * Product table consistency: Units × Pcs/Unit should match Pieces where the document follows that rule.
     * Signature/date validation: confirm customer delivery signature/date presence.
     * Exception detection: flag handwritten notes such as “Wrong Size Lumber”.
   * This is important because the output should not simply be “OCR text”; the workflow should produce trusted structured data suitable for ERP, transportation, claims, receiving, or AP/AR processes.

5. **Human-in-the-loop review**
   * Route low-confidence fields, missing signatures, handwritten exceptions, and mismatched totals to review.
   * Use field-level confidence, validation failures, and exception keywords as review triggers.
   * In [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb), the handwritten content and scan noise are exactly the type of evidence that should trigger review rather than automatic straight-through processing. [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

### Why this strategy fits the attached documents

The attached documents are **semi-structured operational logistics documents**, not simple paragraphs of text. They have recurring sections and tables, but they also contain variability in customer, carrier, shipment, product rows, signatures, handwritten notes, scan quality, and page-level artefacts. [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) has a clear sectioned delivery-slip format with order, shipping and product information.  [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) demonstrates the harder production reality: multiple pages, OCR distortions, handwritten notes, and exception annotations. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf) [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

Therefore, the effective IDP strategy is not “READ only” and not “generic OCR plus regex only”. The better pattern is:

> **Layout/readability first → classify if mixed intake → custom extraction → validation → exception workflow.**

***

## 2) How to select the right models: READ vs LAYOUT vs custom

### READ model — use when the requirement is text-only OCR

The **Read** model is appropriate when the goal is to extract printed or handwritten text lines, words, locations, and detected languages. Microsoft’s model catalogue describes the Read model as extracting written or printed text lines, words, locations, and detected languages. [\[ai.azure.com\]](https://ai.azure.com/catalog/models/Azure-AI-Document-Intelligence)

For these delivery slips, READ is useful for:

* OCR baseline extraction.
* Capturing free text.
* Testing raw recognisability of scanned/photo documents.
* Detecting handwritten text in review zones.

But I would **not** select READ as the primary production model for this scenario, because the business requirement is very likely structured extraction: fields, tables, signatures, dates, line items, and exception notes. The attached files contain structured tables and labelled zones such as Order Information, Shipping Information, Product Information, signature sections and delivery notes. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf), [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

### LAYOUT model — use as the baseline for structure and tables

The **Layout** model is a better initial model than READ for these documents because it extracts structural information such as tables, selection marks, paragraphs, titles, headings, and subheadings in addition to text extraction. Microsoft’s model catalogue describes Layout as extracting tables, selection marks, paragraphs, titles, headings and subheadings on top of text extraction. [\[ai.azure.com\]](https://ai.azure.com/catalog/models/Azure-AI-Document-Intelligence)

For this use case, Layout is valuable for:

* Detecting the product line-item table.
* Preserving row/column relationships: Length, Units, Pieces, Pcs/Unit, Quantity, UOM.
* Understanding document sections.
* Providing spatial anchors for post-processing rules.
* Supporting downstream chunking/search if these PODs are indexed for retrieval.

However, Layout alone may still not deliver business-ready extraction because it does not know that “Shipment”, “Branch/Location”, “Unit #”, “Restrictions”, “Customer Delivery - Received By”, or handwritten “Wrong Size Lumber” are business-critical fields. That mapping is where custom extraction becomes valuable.

### Prebuilt models — likely not the best fit here

Document Intelligence includes multiple prebuilt models, including invoices, receipts, contracts and others. Microsoft’s model overview lists Read and Layout as document analysis models and also lists prebuilt models such as Invoice, Receipt and Contract. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/model-overview?view=doc-intel-4.0.0)

For these attached files, I would **not** rely on Invoice or Receipt prebuilt models as the primary model because the documents are delivery slips/PODs for lumber shipments, not standard invoices or retail receipts. The product table and delivery-signature logic are domain-specific. [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) and [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) are both titled “DELIVERY SLIP” and contain shipping/product/signature sections rather than invoice tax/payment fields. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf), [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

### Custom extraction model — recommended

A **custom extraction model** is the right choice when the target output is a fixed set of business fields from a known document family. Microsoft states that to create a custom extraction model, labelled documents are used to train the model, and only five examples of the same form or document type are needed to get started. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

For this use case, I would train a custom extraction model because:

* The delivery slips have a recurring business schema.
* The required fields are domain-specific.
* The tables need structured extraction.
* The signature/date areas matter operationally.
* Handwritten notes and exception comments need special handling.
* Post-processing rules can use extracted fields to automate workflow decisions.

### Custom neural vs custom template

I would start with **custom neural**, not custom template, unless you confirm that all delivery slips use a highly consistent visual template.

Microsoft documentation states that custom neural models combine layout and language features and are suitable for extracting labelled fields from structured and semi-structured documents.  Microsoft also states that custom template models rely on a consistent visual template, and variations in visual structure affect accuracy. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0) [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0), [\[docs.azure.cn\]](https://docs.azure.cn/en-us/ai-services/document-intelligence/train/custom-model)

For the attached files, custom neural is more appropriate because:

* The visual format is similar but not identical across examples.
* The product rows vary.
* Header contact information varies.
* Some pages contain handwritten notes and scan artefacts.
* One file contains multiple pages with different quality and annotation patterns. [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

I would only choose **custom template** if the production document set is consistently generated from the exact same system/template with minimal layout variation and the main challenge is extracting predictable zones.

***

## 3) Should a custom classification model and custom extraction model be built?

### Decision: build a custom extraction model — yes

My decision is **yes**, build a custom extraction model.

Reasoning:

* The documents are clearly from a repeatable business process: delivery slips / proof-of-delivery handling.
* The target fields are specific to shipping and lumber delivery workflows.
* The fields are visible but spread across sections and tables.
* The attached samples include signatures and handwritten notes, which are important for downstream workflow decisions.
* Microsoft supports custom extraction models trained from labelled examples for forms and documents specific to a business. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

The extraction model should include at least these field groups:

| Field group       | Recommended extraction fields                                             | Why it matters                      |
| ----------------- | ------------------------------------------------------------------------- | ----------------------------------- |
| Document identity | Document type, vendor/source, page count, print date/time                 | Routing, audit, duplicate detection |
| Order             | Order Number, Order Date, Salesperson, Customer                           | ERP/OMS matching                    |
| Shipping          | Sold To, Ship To, Carrier, Phone, Shipment, Branch/Location, Unit, Unit # | TMS and delivery reconciliation     |
| Restrictions      | Tarp required, delivery window, no-Friday delivery, call prior            | Operational compliance              |
| Product table     | Description, code, Length, Units, Pieces, Pcs/Unit, Quantity, UOM         | Receipt and quantity validation     |
| Totals            | Sub-total, Total, Actual/Estimated Load Weight                            | Reconciliation and exception checks |
| POD evidence      | Shipper signature, Carrier signature, Customer delivery signature, dates  | Proof of delivery                   |
| Exceptions        | Handwritten notes, “Wrong Size Lumber”, reference numbers                 | Claims and issue workflow           |

### Decision: build a custom classification model — conditional yes

My decision is **conditional**:

* **Do build a custom classification model** if the production intake includes more than one document class, such as:
  * Interfor delivery slips
  * Bills of lading
  * Carrier load confirmations
  * POD photos
  * Invoices
  * Claims/shortage/damage documents
  * Customer pickup documents
  * Multi-page packets containing several document types

* **Do not build it immediately** if all inbound documents are already known to be Interfor delivery slips/PODs and arrive through a controlled channel.

Microsoft states that custom classification models are used where the document type must be identified before invoking the extraction model, and that a classification model can be paired with a custom extraction model.  Microsoft also states that training a custom classifier requires at least two distinct classes and a minimum of five document samples per class. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0) [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)

For your attached documents specifically, I see a **single dominant business family**: INTERFOR Delivery Slip / POD. [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) is one delivery-slip document, while [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) contains delivery-slip pages with POD/exception annotations.  So classification is not mandatory if the intake is limited to this family. But classification becomes highly valuable if these documents are part of a broader AP/logistics/POD automation pipeline. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf), [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

***

## Practical model-selection decision tree

Use this decision process:

```text
Is the input only used for searchable text?
    Yes → Use READ.
    No  → Continue.

Do you need tables, section structure, paragraphs, headers, or layout?
    Yes → Use LAYOUT as baseline.
    No  → READ may be enough.

Do you need named business fields such as Order Number, Shipment, Ship To, Total, signature date?
    Yes → Build a custom extraction model.
    No  → LAYOUT plus rules may be sufficient.

Are multiple document types mixed in the same intake stream or packet?
    Yes → Build a custom classification model and route to extractors.
    No  → Classification can be deferred.

Are all documents visually identical templates?
    Yes → Consider custom template.
    No / uncertain / variable scans → Prefer custom neural.
```

***

## Recommended implementation approach for this scenario

### Phase 1 — Baseline evaluation

Run the attached documents through:

* READ
* LAYOUT
* A quick labelled custom neural extraction proof of concept

Compare:

* OCR quality on printed fields
* Table row/column preservation
* Signature/date detection
* Handwritten note capture
* Accuracy on order/shipment/product fields
* Confidence scores and failure patterns

### Phase 2 — Define business schema

Create a canonical JSON output such as:

```json
{
  "documentType": "DeliverySlipPOD",
  "sourceCompany": "Interfor",
  "order": {
    "orderNumber": "",
    "orderDate": "",
    "salesperson": "",
    "customerNumber": ""
  },
  "shipping": {
    "soldTo": "",
    "shipTo": "",
    "carrier": "",
    "phone": "",
    "shipment": "",
    "branchLocation": "",
    "unit": "",
    "unitNumber": "",
    "restrictions": ""
  },
  "productLines": [
    {
      "description": "",
      "productCode": "",
      "length": "",
      "units": "",
      "pieces": "",
      "pcsPerUnit": "",
      "quantity": "",
      "uom": ""
    }
  ],
  "totals": {
    "subTotalUnits": "",
    "subTotalPieces": "",
    "subTotalQuantity": "",
    "totalUnits": "",
    "totalPieces": "",
    "totalQuantity": "",
    "loadWeight": ""
  },
  "pod": {
    "shipperSignaturePresent": null,
    "carrierSignaturePresent": null,
    "customerSignaturePresent": null,
    "signatureDates": [],
    "handwrittenNotes": []
  },
  "exceptions": {
    "hasException": null,
    "exceptionText": "",
    "requiresReview": null
  }
}
```

### Phase 3 — Train custom extraction

Use labelled samples by document variant:

* Clean generated PDFs
* Scanned PDFs
* Mobile photos
* Signed PODs
* Unsigned PODs
* Exception PODs
* Multi-page packets
* Different branches/locations
* Different product table lengths

Although Microsoft says five examples are enough to get started for a custom extraction model, I would not stop at five for production. Five is a starting threshold, not a production-quality dataset size.  For production, I would build a representative labelled dataset covering all meaningful variations seen in the attached documents. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)

### Phase 4 — Add classifier if needed

If your intake has document mixing, train classes such as:

* `Interfor_DeliverySlip`
* `Signed_POD`
* `Exception_POD`
* `BillOfLading`
* `Invoice`
* `Carrier_Document`
* `Other_Review`

The classifier is justified if page-level routing is needed because Microsoft custom classifiers can identify document types and return page ranges for identified classes. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)

### Phase 5 — Production validation rules

I would implement validation rules such as:

* If `Customer Delivery - Received By` signature is absent → review.
* If handwritten notes exist → review.
* If exception keywords exist, such as wrong size, shortage, damage, refused, missing, overage → review.
* If `Total Quantity` does not equal sum of product line quantities → review.
* If `Order Number` or `Shipment` missing → review.
* If product code missing → review.
* If confidence below threshold on critical fields → review.

***

## Accuracy checks on information

### What is directly supported by the attached files

* The documents are INTERFOR Delivery Slip documents with structured sections for order information, shipping information, product information, signatures, and print date/time. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf), [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)
* [63461366\_response.pdf](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf?EntityRepresentationId=5c1f912d-50c6-4d23-b269-564bd26e541a) includes fields such as Order Number `2447651`, Order Date `May 14, 2026`, Shipment `5284456`, Branch/Location `DeQuincy / DQ`, Unit `Truck`, Unit # `148`, and Total `23.292 MBF`. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf)
* [POD Scenario 3.PDF](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF?EntityRepresentationId=53798816-3687-4e8f-b07c-28365be2dadb) includes two delivery-slip pages, with page 1 showing Shipment `5161988` and page 2 showing Order Number `2351016`, Shipment `5159191`, Carrier `RSC LOGISTICS, LLC`, and a handwritten exception note “Wrong Size Lumber”. [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)
* The attached documents include both printed structured content and handwritten annotations/signatures, which supports the recommendation for extraction plus validation/review rather than OCR-only processing. [\[63461366_response \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/63461366_response.pdf), [\[POD Scenario 3 \| PDF\]](https://interfor-my.sharepoint.com/personal/dong_shen_interfor_com/Documents/Microsoft%20Copilot%20Chat%20Files/POD%20Scenario%203.PDF)

### What is directly supported by Microsoft documentation

* Read is appropriate for extracting written or printed text lines, words, locations, and detected languages. [\[ai.azure.com\]](https://ai.azure.com/catalog/models/Azure-AI-Document-Intelligence)
* Layout adds structural extraction such as tables, selection marks, paragraphs, titles, headings and subheadings. [\[ai.azure.com\]](https://ai.azure.com/catalog/models/Azure-AI-Document-Intelligence)
* Custom extraction models are trained from labelled documents and can start with five examples of the same form or document type. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* Custom classification models identify document types before invoking extraction models and can be paired with custom extraction models. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0)
* Custom classifier training requires at least two distinct classes and a minimum of five document samples per class. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)
* Custom neural models combine layout and language features and are suitable for structured and semi-structured documents. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)
* Custom template models depend on a consistent visual template, and layout variation can affect accuracy. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-model?view=doc-intel-4.0.0), [\[docs.azure.cn\]](https://docs.azure.cn/en-us/ai-services/document-intelligence/train/custom-model)

### Assumptions I am making

* I am assuming the business goal is to automate POD/delivery-slip processing into an enterprise workflow, not just archive searchable PDFs.
* I am assuming the downstream system needs structured values for order, shipment, product lines, totals, signatures and exceptions.
* I am assuming the two PDFs and the two image examples are representative of production variability, including signed copies, scanned copies, mobile photos and handwritten exception notes.
* I am not assuming the exact production volume, SLA, confidence thresholds, downstream ERP/TMS target, or all document classes because those are not specified in the attached files.

***

## Final decision

My recommendation is:

> **Use Layout as the default baseline analysis model, build a custom neural extraction model for Interfor delivery slips/PODs, and add a custom classification model only if the intake includes multiple document types or multi-document packets.**

This gives the best balance of technical accuracy, maintainability and workflow value. READ alone is too shallow for this use case; Layout is necessary but not sufficient for business-grade automation; custom extraction is justified by the repeated delivery-slip schema and business-specific fields; custom classification is valuable when routing ambiguity exists, but can be deferred if all documents are already known to be the same type.
