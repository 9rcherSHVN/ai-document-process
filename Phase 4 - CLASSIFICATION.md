Below is the deep technical drill-down into **Phase 4 — Custom classification model build** for the Interfor POD / delivery-slip scenario. This follows the same level of depth as the Phase 3 drill-down.

***

# Phase 4 — Custom classification model build (deep dive)

***

# 1. When to build a classifier — decision framework

From the re-assessment, the decision is **conditional**: build a classifier only if the production intake contains mixed document types or multi-document packets.

## 1.1 Scenarios that justify classification for this POD use case

| Scenario                                                                                    | Classifier needed? | Rationale                                                                  |
| ------------------------------------------------------------------------------------------- | ------------------ | -------------------------------------------------------------------------- |
| All inbound files are Interfor delivery slips from a controlled channel                     | ❌ No               | Single document family; route directly to extraction                       |
| Inbound files include delivery slips mixed with BOLs, invoices, or carrier docs             | ✅ Yes              | Must identify document type before routing to the correct extraction model |
| Inbound files are multi-page scanned packets containing delivery slips plus other documents | ✅ Yes              | Must split the packet and classify each document section                   |
| Multiple instances of the same delivery slip in one file (e.g., batch of scanned PODs)      | ✅ Yes              | Must identify page boundaries between documents                            |
| Intake includes unknown/unsupported documents that should be rejected or routed to review   | ✅ Yes              | Must detect and isolate unrecognised document types                        |

Microsoft states: *"Custom classification models perform classification of an input file one page at a time to identify the documents within and can also identify multiple documents or multiple instances of a single document within an input file."* citeturn9search178

Supported scenarios explicitly listed by Microsoft: citeturn9search178

* A single file containing one document type
* A single file containing multiple document types
* A single file containing multiple instances of the same document

## 1.2 Classifier vs composed model decision

Microsoft provides a comparison table for custom classification models versus composed models: citeturn9search178

| Capability                                  | Custom classifier                                  | Composed model                                    |
| ------------------------------------------- | -------------------------------------------------- | ------------------------------------------------- |
| Analyse single unknown document             | Multiple calls: classify first → then extract      | Single call: composed model selects best match    |
| Analyse file with multiple document types   | Multiple calls: classify → extract per document    | Single call but processes only the first instance |
| Ignore unrecognised document types          | ✅ Supported — confidence thresholds allow skipping | ❌ Cannot ignore; always attempts extraction       |
| Split multi-document files into page ranges | ✅ Supported with `split=auto`                      | ❌ Not supported                                   |

**For this POD scenario:** If the production intake includes multi-document packets, the classifier approach is superior because it can identify page ranges per document and handle unknown types. If all documents are the same type and arrive as single files, a composed model or even a standalone extraction model is sufficient.

***

# 2. Understanding the classification model fundamentals

## 2.1 How classification works

Microsoft states: *"The model classifies each page of the input document, unless specified, to one of the classes in the labeled dataset. You can specify the page numbers to analyze in the input document as well."* citeturn9search178

This means:

* Classification is **page-level**, not document-level.
* The classifier assigns each page to a class.
* The response groups contiguous pages of the same class into document instances with page ranges.
* You can restrict classification to specific pages using the `pages` query parameter.

## 2.2 Training requirements

| Requirement                       | Value  | Source             |
| --------------------------------- | ------ | ------------------ |
| Minimum classes                   | 2      | citeturn9search178 |
| Minimum samples per class         | 5      | citeturn9search178 |
| Maximum classes                   | 1,000  | citeturn9search178 |
| Maximum samples per class         | 100    | citeturn9search178 |
| Maximum training data size (v4.0) | 2 GB   | citeturn9search186 |
| Maximum training pages (v4.0)     | 25,000 | citeturn9search178 |

## 2.3 Supported file formats for classifier training

Microsoft documents that custom classification models support: citeturn9search178turn9search186

| Format                                       | Supported                                   |
| -------------------------------------------- | ------------------------------------------- |
| PDF                                          | ✅                                           |
| JPEG/JPG, PNG, BMP, TIFF, HEIF               | ✅                                           |
| Word (DOCX), Excel (XLSX), PowerPoint (PPTX) | ✅ (not supported in Studio — REST API only) |

**Important:** Office file types can be used for classifier training, but **only through the REST API**, not through Document Intelligence Studio. citeturn9search178

For the Interfor POD scenario, the training data is primarily PDFs and JPEG images, so Studio is fully supported.

***

# 3. Class design for the Interfor POD scenario

## 3.1 Recommended class taxonomy

### Tier 1 — Minimum viable classifier (2 classes)

```text
DeliverySlip_POD
Other_Review
```

Use this for the initial implementation if the primary goal is to separate delivery slips from everything else.

### Tier 2 — Expanded classifier (5–7 classes)

```text
Interfor_DeliverySlip
Interfor_SignedPOD
Interfor_ExceptionPOD
BillOfLading
Invoice
CarrierDocument
Other_Review
```

Use this when the intake includes multiple document families and the team wants fine-grained routing.

### Tier 3 — Full production classifier

```text
Interfor_DeliverySlip_Clean
Interfor_DeliverySlip_Scanned
Interfor_DeliverySlip_Photo
Interfor_SignedPOD
Interfor_ExceptionPOD
BillOfLading
Invoice
CarrierLoadConfirmation
CustomerPickupSlip
ClaimDocument
ShortageReport
DamageReport
CoverSheet
BlankPage
Other_Review
```

Use this only when the production volume and document variety justify the investment.

## 3.2 "Other" class best practice

Microsoft states: *"The classifier attempts to assign each document to one of the classes, if you expect the model to see document types not in the classes that are part of the training dataset, you should plan to set a threshold on the classification score or add a few representative samples of the document types to an 'other' class. Adding an 'other' class ensures that unneeded documents don't affect your classifier quality."* citeturn9search178

**For this POD scenario:** Always include an `Other_Review` class. Populate it with:

* Random unrelated documents (invoices, letters, emails, spreadsheets)
* Blank or partially blank pages
* Cover sheets or fax headers
* Any documents that should not be processed as PODs

This prevents the classifier from forcibly assigning a non-POD document to the `DeliverySlip_POD` class with artificially inflated confidence.

## 3.3 Sample sourcing per class

For the attached documents:

| Class              | Source examples                                                                                   | Min samples |
| ------------------ | ------------------------------------------------------------------------------------------------- | ----------- |
| `DeliverySlip_POD` | `63461366_response.pdf` citeturn1search1, `POD Scenario 3.PDF` citeturn1search2, Image 1, Image 2 | 5–15        |
| `Other_Review`     | Random non-POD documents                                                                          | 5–10        |

For Tier 2/3 classes, source samples from production intake.

***

# 4. Prepare the training dataset

## 4.1 Folder structure — recommended approach

Microsoft states: *"You can organize the training dataset by folders where the folder name is the label or class for documents."* and *"If the documents are organized in folders, the Studio prompts you to use the folder names as labels."* citeturn9search186

**Recommended folder structure:**

```text
/training/classifier/
  ├── DeliverySlip_POD/
  │   ├── interfor_pod_clean_001.pdf
  │   ├── interfor_pod_clean_002.pdf
  │   ├── interfor_pod_scanned_001.pdf
  │   ├── interfor_pod_photo_001.jpeg
  │   ├── interfor_pod_photo_002.jpeg
  │   ├── interfor_pod_signed_001.pdf
  │   ├── interfor_pod_exception_001.pdf
  │   ├── interfor_pod_multiproduct_001.jpeg
  │   └── ... (5–15 samples)
  │
  ├── BillOfLading/
  │   ├── bol_001.pdf
  │   ├── bol_002.pdf
  │   └── ... (5+ samples)
  │
  ├── Invoice/
  │   ├── inv_001.pdf
  │   ├── inv_002.pdf
  │   └── ... (5+ samples)
  │
  └── Other_Review/
      ├── misc_letter_001.pdf
      ├── blank_page_001.pdf
      ├── fax_cover_001.pdf
      └── ... (5+ samples)
```

## 4.2 Alternative: flat file list approach

If the training data is not organized by folders, use the `azureBlobFileListSource` approach with JSON Lines files.

Microsoft states: *"Alternatively, if you have a flat list of files or only plan to use a few select files within each folder to train the model, you can use the azureBlobFileListSource property to train the model. This step requires a file list in JSON Lines format."* citeturn9search178

Example `DeliverySlip_POD.jsonl`:

```jsonl
{"file":"classifier/interfor_pod_clean_001.pdf"}
{"file":"classifier/interfor_pod_clean_002.pdf"}
{"file":"classifier/interfor_pod_scanned_001.pdf"}
{"file":"classifier/interfor_pod_photo_001.jpeg"}
{"file":"classifier/interfor_pod_photo_002.jpeg"}
```

## 4.3 Training data balance and quality tips

Microsoft states: citeturn9search186

* *"If possible, use text-based PDF documents instead of image-based documents. Scanned PDFs are handled as images."*
* *"If your form images are of lower quality, use a larger data set (10-15 images, for example)."*

For the POD scenario:

* Include both clean PDFs and mobile photos in each class.
* Include samples from different branches (Preston/PS, Port Angeles/PA, DeQuincy/DQ) within the `DeliverySlip_POD` class.
* Include samples with different products, carriers, and restriction types.
* For the `Other_Review` class, include visually diverse non-POD documents.

## 4.4 Layout results requirement

**Critical:** Microsoft states: *"The classification model requires results from the layout model for each training document."* and *"When using the API or SDK to train a classifier, you need to add the layout results to the folders containing the individual documents. The layout results should be in the format of the API response when calling layout directly."* citeturn9search186

In Studio, this is handled automatically: *"If you don't provide the layout results, the Studio attempts to run the layout model for each document before training the classifier."* citeturn9search186

**Throttling warning:** Microsoft warns: *"This process is throttled and can result in a 429 response. In the Studio, before training with the classification model, run the layout model on each document and upload it to the same location as the original document."* citeturn9search186

**Best practice:** Run Layout on all training documents **before** initiating classifier training to avoid 429 throttling during the training process.

***

# 5. Create the classification project in Studio

## Step 1 — Open Studio

Navigate to Document Intelligence Studio → **Custom classification model**.

## Step 2 — Create a project

Microsoft states: *"Select the Custom classification model tile, on the custom models section of the page and select the Create a project button."* citeturn9search186

1. Select **Create a project**.
2. Enter:
   * **Project name:** `Interfor-POD-Classifier`
   * **Description:** `Custom classifier for Interfor delivery slips, BOLs, invoices, and other document types. v4.0 2024-11-30.`
3. Select **Continue**.

## Step 3 — Select or create the Document Intelligence resource

1. Select the Document Intelligence resource (`di-pod-prod`).
2. Select **Continue**.

## Step 4 — Connect storage

1. Select the storage account: `stpodidpprod`.
2. Select the container.
3. Enter the **Folder path**: `training/classifier`
4. Select **Continue**.

Microsoft states: *"The Folder path should be empty if your training documents are in the root of the container. If your documents are in a subfolder, enter the relative path from the container root in the Folder path field."* citeturn9search186

## Step 5 — Review and create

Review the project configuration and select **Create Project**.

***

# 6. Label the training data

## 6.1 Folder-based auto-labelling

If the training data is organised by folders (recommended), Studio prompts you to use folder names as labels.

Microsoft states: *"If the documents are organized in folders, the Studio prompts you to use the folder names as labels. This step simplifies your labeling down to a single select."* citeturn9search186

For the recommended folder structure:

* `DeliverySlip_POD/` → class `DeliverySlip_POD`
* `BillOfLading/` → class `BillOfLading`
* `Invoice/` → class `Invoice`
* `Other_Review/` → class `Other_Review`

## 6.2 Manual labelling

If documents are in a flat list or folders do not match class names:

1. Open the labelling window.
2. Select a document from the file list.
3. Select the **add label** selection mark to assign a class.
4. Use **Ctrl-select** to multi-select documents and assign them to the same class in bulk.

Microsoft states: *"To assign a label to a document, select the add label selection mark to assign a label. Use control-select to multi-select documents to assign to a label."* citeturn9search186

## 6.3 Key difference from extraction labelling

Classification labelling is **document-level**, not field-level. You do not select individual words or draw regions. You simply assign the entire document to a class. This makes classifier labelling significantly faster than extraction labelling.

## 6.4 Generated files after labelling

After labelling, the storage account contains: citeturn9search186

* `.ocr.json` files corresponding to each training document (from Layout API)
* `{class-name}.jsonl` files for each labelled class

***

# 7. Run Layout on all documents before training

Microsoft explicitly warns about throttling: *"In the Studio, before training with the classification model, run the layout model on each document and upload it to the same location as the original document. Once the layout results are added, you can train the classifier model with your documents."* citeturn9search186

## Step-by-step

1. In the Studio labelling window, ensure all documents have been processed by Layout.
2. If any documents show a "Layout not run" status, run Layout on those documents first.
3. Wait for all Layout results to complete before proceeding to training.
4. Verify that `.ocr.json` files exist alongside each training document in blob storage.

***

# 8. Train the classifier

## 8.1 Train in Studio

1. Select the **Train** button in the upper-right corner.
2. Enter a unique **classifier ID**:

```text
interfor-pod-classifier-v001
```

3. Optionally enter a **description**:

```text
Custom classifier for Interfor POD/delivery slip document routing.
Classes: DeliverySlip_POD, BillOfLading, Invoice, Other_Review.
v4.0 2024-11-30.
```

4. Select **Train**.

Microsoft states: *"On the train model dialog, provide a unique classifier ID and, optionally, a description. The classifier ID accepts a string data type. Select Train to initiate the training process."* citeturn9search186

## 8.2 Training speed

Microsoft states: *"Classifier models train in a few minutes. Navigate to the Models menu to view the status of the train operation."* citeturn9search186

Classifier training is significantly faster than neural extraction model training. Expect 1–5 minutes for a typical dataset.

## 8.3 Train via REST API

The Build Classifier REST API: citeturn9search198

```http
POST {endpoint}/documentintelligence/documentClassifiers:build?api-version=2024-11-30
```

Request body using folder-based training: citeturn9search178

```json
{
  "classifierId": "interfor-pod-classifier-v001",
  "description": "Custom classifier for Interfor POD document routing",
  "docTypes": {
    "DeliverySlip_POD": {
      "azureBlobSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "prefix": "classifier/DeliverySlip_POD/"
      }
    },
    "BillOfLading": {
      "azureBlobSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "prefix": "classifier/BillOfLading/"
      }
    },
    "Invoice": {
      "azureBlobSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "prefix": "classifier/Invoice/"
      }
    },
    "Other_Review": {
      "azureBlobSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "prefix": "classifier/Other_Review/"
      }
    }
  }
}
```

Response: `202 Accepted` with `Operation-Location` and `Retry-After` headers. citeturn9search198

### Alternative: file list-based training

```json
{
  "classifierId": "interfor-pod-classifier-v001",
  "description": "Custom classifier for Interfor POD document routing",
  "docTypes": {
    "DeliverySlip_POD": {
      "azureBlobFileListSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "fileList": "classifier/DeliverySlip_POD.jsonl"
      }
    },
    "Other_Review": {
      "azureBlobFileListSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "fileList": "classifier/Other_Review.jsonl"
      }
    }
  }
}
```

Microsoft states: *"For each class, add a new file with a list of files to be submitted for training."* citeturn9search178

***

# 9. Document splitting — the `split` parameter

## 9.1 Three split modes

Microsoft states: *"The analyze operation now includes a splitMode property that gives you granular control over the splitting behavior."* citeturn9search178

| `split` value            | Behaviour                                              | Use when                             |
| ------------------------ | ------------------------------------------------------ | ------------------------------------ |
| `none` (default in v4.0) | Entire input file treated as a single document         | Single-document files                |
| `perPage`                | Each page classified as an individual document         | Batch of unrelated single-page scans |
| `auto`                   | Service identifies document boundaries and page ranges | Multi-document packets               |

**Critical v4.0 change:** Microsoft states: *"The v4.0 2024-11-30 (GA) API, custom classification model doesn't split documents by default during the analyzing process. You need to explicitly set the splitMode property to auto to preserve the behavior from previous releases. The default for splitMode is none."* citeturn9search178

## 9.2 Which mode to use for the POD scenario

| POD intake pattern                                      | Recommended `split` |
| ------------------------------------------------------- | ------------------- |
| Single delivery slip per file                           | `none`              |
| Multi-page delivery slip packet (all pages are one POD) | `none`              |
| Scanned batch: multiple PODs in one PDF                 | `auto`              |
| Mixed packet: delivery slip + BOL + invoice in one PDF  | `auto`              |
| Batch of single-page scans stapled into one PDF         | `perPage`           |

## 9.3 REST API `split` parameter

**Correction from re-assessment:** The REST API **query parameter name** is `split`, not `splitMode`. Microsoft's prose documentation uses `splitMode` in descriptions, but the actual REST query parameter is `split`. citeturn9search187

```http
POST {endpoint}/documentintelligence/documentClassifiers/{classifierId}:analyze?_overload=classifyDocument&api-version=2024-11-30&split=auto
```

The classify document API accepts these query parameters: `classifierId`, `api-version`, `stringIndexType`, `split`, and `pages`. citeturn9search187

***

# 10. Test the classifier

## 10.1 Test in Studio

Microsoft states: *"Once the model training is complete, you can test your model by selecting the model on the models list page. Select the model and choose the Test button. Add a new file by browsing for a file or dropping a file into the document selector. With a file selected, choose the Analyze button to test the model."* citeturn9search186

## 10.2 Studio test results

Microsoft states: *"The model results are displayed with the list of identified documents, a confidence score for each document identified and the page range for each of the documents identified."* citeturn9search186

## 10.3 Test matrix for the POD scenario

| Test ID | Input                                    | Expected class                                 | Expected pages           | Key validation                                    |
| ------- | ---------------------------------------- | ---------------------------------------------- | ------------------------ | ------------------------------------------------- |
| TC01    | Single-page clean delivery slip PDF      | `DeliverySlip_POD`                             | 1                        | High confidence (≥ 0.90)                          |
| TC02    | Single-page mobile photo JPEG            | `DeliverySlip_POD`                             | 1                        | Confidence may be slightly lower                  |
| TC03    | Multi-page delivery slip (2 pages)       | `DeliverySlip_POD`                             | 1–2                      | Both pages assigned to same class                 |
| TC04    | Multi-page packet: POD (p1–2) + BOL (p3) | `DeliverySlip_POD` (p1–2), `BillOfLading` (p3) | Split correctly          | Requires `split=auto`                             |
| TC05    | Batch: 3 delivery slips in one PDF       | `DeliverySlip_POD` × 3                         | Page ranges per document | Requires `split=auto`                             |
| TC06    | Invoice document                         | `Invoice`                                      | All pages                | Verify not misclassified as POD                   |
| TC07    | Unknown/unsupported document             | `Other_Review` or low confidence               | All pages                | Confidence below threshold                        |
| TC08    | Blank page                               | `Other_Review` or low confidence               | 1                        | Should not classify as POD                        |
| TC09    | Delivery slip with exception note        | `DeliverySlip_POD`                             | All pages                | Exception PODs still classified as delivery slips |
| TC10    | Mixed: POD + Invoice + unknown           | Three documents identified                     | Correct page ranges      | Full packet splitting test                        |

## 10.4 Evaluate confidence thresholds

Microsoft states: *"To set the threshold for your application, use the confidence score from the response."* citeturn9search178

### Recommended starting thresholds

| Routing decision        | Confidence condition                            | Action                                                  |
| ----------------------- | ----------------------------------------------- | ------------------------------------------------------- |
| High confidence match   | ≥ 0.85                                          | Route to extraction model automatically                 |
| Medium confidence match | 0.60–0.85                                       | Route to extraction but flag for post-extraction review |
| Low confidence match    | < 0.60                                          | Route to human review — do not extract                  |
| Unrecognised type       | Class = `Other_Review` regardless of confidence | Route to human review                                   |

These thresholds should be calibrated against actual test results.

***

# 11. Classify Document REST API — full reference

## 11.1 Submit classification

```http
POST {endpoint}/documentintelligence/documentClassifiers/{classifierId}:analyze?_overload=classifyDocument&api-version=2024-11-30&stringIndexType=utf16CodeUnit&split=auto
```

Request body: citeturn9search187

```json
{
  "base64Source": "<base64-encoded-document>"
}
```

Or:

```json
{
  "urlSource": "https://stpodidpprod.blob.core.windows.net/inbound/document.pdf?<SAS>"
}
```

Response: `202 Accepted` with headers: citeturn9search187

* `Operation-Location`: URL to poll for results
* `Retry-After`: Recommended delay before polling

## 11.2 Get classification result

```http
GET {endpoint}/documentintelligence/documentClassifiers/{classifierId}/analyzeResults/{resultId}?api-version=2024-11-30
```

citeturn9search195

### Sample response structure

```json
{
  "status": "succeeded",
  "createdDateTime": "2026-06-10T13:00:46Z",
  "lastUpdatedDateTime": "2026-06-10T13:00:49Z",
  "analyzeResult": {
    "apiVersion": "2024-11-30",
    "modelId": "interfor-pod-classifier-v001",
    "stringIndexType": "textElements",
    "contentFormat": "text",
    "content": "",
    "pages": [
      { "pageNumber": 1, "width": 8.5, "height": 11, "unit": "inch", "spans": [] },
      { "pageNumber": 2, "width": 8.5, "height": 11, "unit": "inch", "spans": [] },
      { "pageNumber": 3, "width": 8.5, "height": 11, "unit": "inch", "spans": [] }
    ],
    "documents": [
      {
        "docType": "DeliverySlip_POD",
        "boundingRegions": [
          { "pageNumber": 1, "polygon": [0, 0, 8.5, 0, 8.5, 11, 0, 11] },
          { "pageNumber": 2, "polygon": [0, 0, 8.5, 0, 8.5, 11, 0, 11] }
        ],
        "confidence": 0.97,
        "spans": []
      },
      {
        "docType": "BillOfLading",
        "boundingRegions": [
          { "pageNumber": 3, "polygon": [0, 0, 8.5, 0, 8.5, 11, 0, 11] }
        ],
        "confidence": 0.92,
        "spans": []
      }
    ]
  }
}
```

citeturn9search195

### Key response elements

| Element                                    | Description                      | Use in orchestration                         |
| ------------------------------------------ | -------------------------------- | -------------------------------------------- |
| `documents[].docType`                      | Classified document type         | Route to correct extraction model            |
| `documents[].confidence`                   | Classification confidence        | Accept/review/reject threshold               |
| `documents[].boundingRegions[].pageNumber` | Pages assigned to this document  | Pass to extraction API via `pages` parameter |
| `status`                                   | `succeeded`, `failed`, `running` | Polling loop exit condition                  |

***

# 12. Incremental training — deep dive

## 12.1 When to use incremental training

Microsoft states: *"Classifiers should be periodically updated whenever the following changes occur: You add new templates for an existing class. You add new document types for recognition. Classifier confidence is low."* citeturn9search177

Specific triggers for the POD scenario:

* A new Interfor branch starts sending delivery slips with a different header layout.
* A new carrier document type enters the intake stream.
* The classifier produces low confidence on a category of documents.
* Business adds a new document class (e.g., `ClaimDocument`).

## 12.2 How incremental training works

Microsoft states: *"Incremental training doesn't introduce any new API endpoints. The documentClassifiers:build request payload is modified to support incremental training. Incremental training results in a new classifier model being created with the existing classifier left untouched. The new classifier has all the document samples and types of the old classifier along with the newly provided samples."* citeturn9search177

**Key properties:**

* The original classifier is **not modified**.
* A **new classifier** is created that includes all original samples plus new ones.
* The `baseClassifierId` property references the existing classifier.

## 12.3 Incremental training REST API

```http
POST {endpoint}/documentintelligence/documentClassifiers:build?api-version=2024-11-30
```

```json
{
  "classifierId": "interfor-pod-classifier-v002",
  "description": "Incremental update: added new branch samples and ClaimDocument class",
  "baseClassifierId": "interfor-pod-classifier-v001",
  "docTypes": {
    "DeliverySlip_POD": {
      "azureBlobSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "prefix": "classifier-incremental/DeliverySlip_POD_new_branch/"
      }
    },
    "ClaimDocument": {
      "azureBlobSource": {
        "containerUrl": "https://stpodidpprod.blob.core.windows.net/training?<SAS>",
        "prefix": "classifier-incremental/ClaimDocument/"
      }
    }
  }
}
```

citeturn9search177

**Behaviour:**

* Existing `docType` (e.g., `DeliverySlip_POD`): new samples are **added to** the existing samples from the base classifier.
* New `docType` (e.g., `ClaimDocument`): the new class is **only present** in the new classifier.

## 12.4 Important limitations

Microsoft states: citeturn9search177

* *"Incremental training only works when the base classifier and the incrementally trained classifier are both trained on the same API version."*
* *"Incremental training is only supported with API version v4.0 2024-11-30 (GA) or later."*
* *"Incremental training requires that you provide the original model ID as the baseClassifierId."*
* *"Training dataset size limits for the incremental classifier are the same as for other classifier model."*

## 12.5 GET response behaviour for incremental classifiers

Microsoft states: *"The GET response from an incrementally trained classifier differs from the standard classifier GET response. The incrementally trained classifier doesn't return all the document types supported. It returns the document types added or updated in the incremental training step and the extended base classifier. To get a complete list of document types, the base classifier must be listed."* citeturn9search177

**Implication:** When managing classifier versions, you must track the full lineage (base → v002 → v003) to understand the complete class list.

## 12.6 Deleting base classifiers

Microsoft states: *"Deleting a base classifier doesn't impact the use of an incrementally trained classifier."* citeturn9search177

This means v002 continues to work even if v001 is deleted. However, I recommend **not deleting** base classifiers during the active improvement cycle for auditability.

***

# 13. Model overwriting

Microsoft states: *"The v4.0 2024-11-30 (GA) custom classification model supports overwriting a model in-place. You can now update the custom classification in-place. Directly overwriting the model would lose you the ability to compare model quality before deciding to replace the existing model. Model overwriting is allowed when the allowOverwrite property is explicitly specified in the request body. It's impossible to recover the overwritten, original model once this action is performed."* citeturn9search178

```json
{
  "classifierId": "interfor-pod-classifier-v001",
  "allowOverwrite": true,
  "docTypes": { ... }
}
```

**My recommendation:** Do **not** use `allowOverwrite` in production. Always create new classifier IDs for new versions. This preserves the ability to compare versions and roll back.

***

# 14. Copy classifier across regions

## 14.1 Supported regions for classifier copy

Microsoft states: *"The custom classification v4.0 2024-11-30 (GA) model supports copying a model to and from any of the following regions: East US, West US2, West Europe."* citeturn9search178

**Important limitation:** Classifier copy is currently limited to these three regions. If your production resource is in a different region (e.g., Canada Central), you cannot directly copy the classifier to/from that region. You would need to train in a supported region and use the copied model, or train directly in the production region.

**Also note:** Microsoft states in the incremental training documentation: *"Copy operation for classifiers is currently unavailable."* citeturn9search177

**Resolution of conflicting statements:** The custom classifier concept page says copy is supported for v4.0 in specific regions. The incremental training page says copy is unavailable. This may indicate that copy was added after the incremental training page was last updated, or that copy for incrementally trained classifiers specifically has limitations. **Verify this behaviour in your target environment before relying on classifier copy in production.**

## 14.2 Copy REST API

### Step 1 — Authorise copy on target resource

```http
POST {target-endpoint}/documentintelligence/documentClassifiers:authorizeCopy?api-version=2024-11-30
```

```json
{
  "classifierId": "interfor-pod-classifier-v001",
  "description": "Production copy of POD classifier"
}
```

### Step 2 — Execute copy on source resource

```http
POST {source-endpoint}/documentintelligence/documentClassifiers/{classifierId}:copyTo?api-version=2024-11-30
```

Body: the authorisation response from Step 1.

citeturn9search178

***

# 15. Classifier routing matrix — production design

After classification, the orchestration (Logic Apps or Power Automate) uses the classifier output to route each identified document.

## 15.1 Routing table

| Classified docType | Confidence        | Action                            | Extraction model               | Downstream                |
| ------------------ | ----------------- | --------------------------------- | ------------------------------ | ------------------------- |
| `DeliverySlip_POD` | ≥ 0.85            | Extract                           | `interfor-pod-extraction-v001` | ERP/TMS/AP                |
| `DeliverySlip_POD` | 0.60–0.85         | Extract + flag for review         | `interfor-pod-extraction-v001` | ERP/TMS/AP + review queue |
| `DeliverySlip_POD` | < 0.60            | Route to human review             | None                           | Review queue              |
| `BillOfLading`     | ≥ 0.85            | Extract (if BOL model exists)     | `bol-extraction-v001`          | TMS                       |
| `Invoice`          | ≥ 0.85            | Extract (if invoice model exists) | `prebuilt-invoice` or custom   | AP                        |
| `Other_Review`     | Any               | Route to human review             | None                           | Review queue              |
| Any class          | Status = `failed` | Route to error                    | None                           | Error queue               |

## 15.2 Page-range extraction

When the classifier identifies a multi-document packet, pass the page ranges to the extraction API using the `pages` parameter:

```http
POST {endpoint}/documentintelligence/documentModels/interfor-pod-extraction-v001:analyze?api-version=2024-11-30&pages=1-2
```

```http
POST {endpoint}/documentintelligence/documentModels/bol-extraction-v001:analyze?api-version=2024-11-30&pages=3
```

This avoids extracting irrelevant pages and improves accuracy and performance.

## 15.3 Parallel extraction for multi-document packets

When multiple documents are identified in a single packet, submit extraction calls for each document's page range **in parallel** using Logic Apps parallel branches or Power Automate parallel branches. This reduces total processing time.

***

# 16. Classifier vs extraction — combined flow

```text
Input document arrives
  │
  ├─ Single-document file (no classifier needed)
  │   └─ Call extraction model directly
  │
  └─ Multi-document file or unknown type (classifier needed)
      │
      ├─ Call classifier with split=auto
      │
      ├─ For each identified document in response:
      │   │
      │   ├─ docType = DeliverySlip_POD, confidence ≥ threshold
      │   │   └─ Call extraction model with pages=<page range>
      │   │
      │   ├─ docType = other known type, confidence ≥ threshold
      │   │   └─ Call appropriate extraction model with pages=<page range>
      │   │
      │   ├─ docType = Other_Review or confidence < threshold
      │   │   └─ Route to human review
      │   │
      │   └─ No documents identified
      │       └─ Route to error/review
      │
      └─ Aggregate all extraction results into canonical payload
```

***

# 17. Troubleshooting guide — classifier-specific

| Symptom                                                          | Likely cause                                                         | Resolution                                                                                                                 |
| ---------------------------------------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Training fails with 429                                          | Layout not pre-run on training documents                             | Run Layout on all documents before training. citeturn9search186                                                            |
| All pages classified as one class                                | `split=none` (default in v4.0)                                       | Set `split=auto` for multi-document files. citeturn9search178                                                              |
| Low confidence on known document types                           | Insufficient or unrepresentative training samples                    | Add more diverse samples for the affected class.                                                                           |
| Unknown documents classified with high confidence                | No `Other_Review` class in training                                  | Add an `Other_Review` class with diverse non-target samples. citeturn9search178                                            |
| Incrementally trained classifier missing classes in GET response | Expected behaviour — only new/updated classes returned               | Query the base classifier for the complete class list. citeturn9search177                                                  |
| Classifier copy fails                                            | Target region not in supported list (East US, West US2, West Europe) | Train directly in the target region or use a supported region. citeturn9search178                                          |
| Office files not accepted in Studio                              | Office file training only supported via REST API                     | Use REST API for DOCX/XLSX/PPTX training. citeturn9search178                                                               |
| Classifier assigns exception POD to wrong class                  | Exception PODs visually similar to normal PODs                       | Keep exception PODs in the same `DeliverySlip_POD` class; use extraction model to detect exceptions via handwritten notes. |

***

# 18. Complete Phase 4 deliverables

| #  | Deliverable                               | Owner                           | Description                                                 |
| -- | ----------------------------------------- | ------------------------------- | ----------------------------------------------------------- |
| 1  | Trained classifier model ID               | Model builder                   | `interfor-pod-classifier-v001`                              |
| 2  | Class taxonomy                            | BA + solution specialist        | Locked list of document classes with descriptions           |
| 3  | Training dataset in blob storage          | Model builder                   | Folder-organised samples per class                          |
| 4  | Layout results for all training documents | Model builder                   | `.ocr.json` files alongside training documents              |
| 5  | Test result set                           | QA / model builder              | Confidence and page ranges per test document                |
| 6  | Confidence threshold matrix               | BA + solution specialist        | Per-class accept/review/reject thresholds                   |
| 7  | Routing matrix                            | Solution specialist             | Class → extraction model → downstream system                |
| 8  | `split` parameter decision                | Developer / solution specialist | `none`, `perPage`, or `auto` per intake pattern             |
| 9  | Incremental training plan                 | Model builder                   | When and how to add classes or samples                      |
| 10 | Classifier version register               | Solution specialist             | Model ID, base classifier, classes, creation date           |
| 11 | Copy/promotion plan                       | Developer / platform            | Region availability, DEV → TEST → PROD strategy             |
| 12 | Troubleshooting runbook                   | Developer / operations          | Common issues and resolutions                               |
| 13 | "Other" class sample library              | BA                              | Diverse non-target documents for ongoing classifier quality |

***

# Accuracy checks — Phase 4 specific

| Claim                                                                                                           | Source                                                                                                                                                                          | Status                                       |
| --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| Classifier requires ≥ 2 classes, ≥ 5 samples per class                                                          | [Custom classification model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)                           | ✅ Confirmed citeturn9search178               |
| Max 1,000 classes, max 100 samples per class                                                                    | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search178               |
| v4.0 training data max 2 GB, 25,000 pages                                                                       | Same source + [Build classifier how-to](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-classifier?view=doc-intel-4.0.0) | ✅ Confirmed citeturn9search178turn9search186 |
| v4.0 default `split` is `none`; set `split=auto` for multi-document files                                       | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search178               |
| Three split modes: `none`, `perPage`, `auto`                                                                    | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search178               |
| Incremental training supported in v4.0; uses `baseClassifierId`                                                 | [Incremental classifiers](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/incremental-classifier?view=doc-intel-4.0.0)                        | ✅ Confirmed citeturn9search177               |
| Incremental training creates new model; original untouched                                                      | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search177               |
| Incremental GET response only returns new/updated classes                                                       | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search177               |
| Deleting base classifier does not impact incremental classifier                                                 | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search177               |
| Incremental training requires same API version                                                                  | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search177               |
| Model overwrite supported with `allowOverwrite: true`                                                           | [Custom classification model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-classifier?view=doc-intel-4.0.0)                           | ✅ Confirmed citeturn9search178               |
| Classifier copy supported in v4.0 for East US, West US2, West Europe                                            | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search178               |
| Office file types supported for classifier training via REST API only                                           | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search178               |
| "Other" class recommended for unrecognised document handling                                                    | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search178               |
| Folder-based auto-labelling in Studio                                                                           | [Build classifier how-to](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-classifier?view=doc-intel-4.0.0)               | ✅ Confirmed citeturn9search186               |
| Layout must be run on all training docs before classifier training                                              | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search186               |
| 429 throttling warning if Layout not pre-run                                                                    | Same source                                                                                                                                                                     | ✅ Confirmed citeturn9search186               |
| Classify Document REST API: POST with `classifierId`, `split`, `pages`                                          | [Classify Document REST API](https://learn.microsoft.com/en-us/rest/api/aiservices/document-classifiers/classify-document?view=rest-aiservices-v4.0 "2024-11-30")               | ✅ Confirmed citeturn9search187               |
| Get Classify Result returns `documents[]` with `docType`, `boundingRegions`, `confidence`                       | [Get Classify Result REST API](https://learn.microsoft.com/en-us/rest/api/aiservices/document-classifiers/get-classify-result?view=rest-aiservices-v4.0 "2024-11-30")           | ✅ Confirmed citeturn9search195               |
| Build Classifier REST API requires `classifierId`, `docTypes`; supports `baseClassifierId` and `allowOverwrite` | [Build Classifier REST API](https://learn.microsoft.com/en-us/rest/api/aiservices/document-classifiers/build-classifier?view=rest-aiservices-v4.0 "2024-11-30")                 | ✅ Confirmed citeturn9search198               |

***

This completes the Phase 4 deep dive. The classifier is now fully designed with class taxonomy, training data structure, Studio build process, REST API reference, `split` parameter guidance, incremental training strategy, copy/promotion plan, routing matrix, and troubleshooting runbook.

Next: Phase 5 (validation rules deep dive), Phase 7 (continuous improvement loop), or produce a downloadable artefact such as the OpenAPI definition for the custom connector.
