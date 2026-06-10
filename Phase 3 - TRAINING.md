Below is the deep technical drill-down into **Phase 3 — Custom extraction model build** for the Interfor POD / delivery-slip scenario, incorporating all corrections from the re-assessment and grounded in the latest Microsoft documentation for v4.0 (2024-11-30 GA).

***

# Phase 3 — Custom neural extraction model build (deep dive)

***

# 1. Understanding the labelled dataset structure

Before opening Studio, the team must understand what the labelling process physically produces. Microsoft documents three generated file types alongside the training documents: citeturn7search157

| File                 | Purpose                                                                                                                       | Created when                                                                                |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `fields.json`        | Master field dictionary for the entire training dataset. Contains every field name, sub-field, and type definition.           | When the **first field** is added to the project. One file per project.                     |
| `{file}.ocr.json`    | Layout API response for each training document. Contains text spans, word positions, bounding polygons, and table structures. | When Studio runs **Layout/OCR** on each uploaded document. One per document.                |
| `{file}.labels.json` | Label mapping for each document. Contains the spans of text and associated polygons that the labeller assigned to each field. | When a **field is labelled** in a document. One per document, updated on each label change. |

Microsoft states: *"A fields.json file is created when the first field is added. There's one fields.json file for the entire training dataset, the field list contains the field name and associated sub fields and types. The Studio runs each of the documents through the Layout API. The layout response for each of the sample files in the dataset is added as {file}.ocr.json. A {file}.labels.json file is created or updated when a field is labeled in a document."* citeturn7search157

### Why this matters for developers and BAs

* **`fields.json`** is the single source of truth for the extraction schema. Any schema change (adding, renaming, retyping a field) modifies this file.
* **`{file}.ocr.json`** is the Layout API output. If OCR quality is poor for a document, the downstream labelling and training quality is also poor. Inspect these files when extraction accuracy is low.
* **`{file}.labels.json`** contains the ground truth. Labelling consistency across documents directly determines model accuracy.

### Physical storage structure

In the Azure Blob Storage container linked to the Studio project, the training folder looks like:

```text
/training/extraction/
  ├── fields.json
  ├── doc_001.pdf
  ├── doc_001.pdf.ocr.json
  ├── doc_001.pdf.labels.json
  ├── doc_002.jpg
  ├── doc_002.jpg.ocr.json
  ├── doc_002.jpg.labels.json
  ├── doc_003.pdf
  ├── doc_003.pdf.ocr.json
  ├── doc_003.pdf.labels.json
  └── ... (repeat for each training document)
```

***

# 2. Building a balanced training dataset

## 2.1 Microsoft's guidance on dataset balance

Microsoft states: *"A balanced dataset represents all the typical variations you would expect to see for the document. Creating a balanced dataset results in a model with the highest possible accuracy."* citeturn7search157

For neural models specifically: *"When your dataset has a manageable set of variations, about 15 or fewer, create a single dataset with a few samples of each of the different variations to train a single model. If the number of template variations is larger than 15, you train multiple models and compose them together."* citeturn7search157

## 2.2 Variation dimensions for the Interfor POD scenario

Based on all four attached documents, the following variation dimensions must be represented:

### Dimension 1 — Document format

| Variation                   | Example from attachments                          | Min samples |
| --------------------------- | ------------------------------------------------- | ----------- |
| Clean system-generated PDF  | `63461366_response.pdf` citeturn1search1          | 3–5         |
| Scanned PDF                 | `POD Scenario 3.PDF` citeturn1search2             | 3–5         |
| Mobile-photo capture (JPEG) | `63506743_response.jpeg`, `63574059response.jpeg` | 3–5         |

Microsoft states: *"If you expect to analyze both digital and scanned documents, add a few examples of each type to the training dataset."* citeturn7search157

### Dimension 2 — Branch/location variation

The documents show different Interfor branches with different header addresses:

| Branch            | Header address                                             | Source                                   |
| ----------------- | ---------------------------------------------------------- | ---------------------------------------- |
| Preston / PS      | Interfor U.S. Inc., 700 Westpark Drive, Peachtree City, GA | Image 1                                  |
| Port Angeles / PA | …& Marketing Ltd., 1600-4720 Kingsway, Burnaby BC V5H 4N2  | Image 2                                  |
| DeQuincy / DQ     | (from PDF)                                                 | `63461366_response.pdf` citeturn1search1 |

Include samples from **at least 3–5 different branches** to ensure the model generalises across header address variations.

### Dimension 3 — Product table variation

| Variation                       | Example                                               | Min samples |
| ------------------------------- | ----------------------------------------------------- | ----------- |
| Single product group, few rows  | Image 1: SY24#2 with 2 rows                           | 2–3         |
| Single product group, many rows | `63461366_response.pdf`                               | 2–3         |
| Multiple product groups         | Image 2: HF242+ and HF24STUD in two separate sections | 3–5         |
| Cross-page table                | If production documents have tables spanning pages    | 2–3         |

Microsoft states: *"For documents containing tables with a variable number of rows, ensure that the training dataset also represents documents with different numbers of rows."* citeturn7search157

### Dimension 4 — Signature variation

| Variation                      | Example                                                                 | Min samples |
| ------------------------------ | ----------------------------------------------------------------------- | ----------- |
| All three signatures present   | Image 2: shipper "Dylan King", carrier "K Franklin", customer with date | 3–5         |
| Only shipper + customer signed | Image 1: shipper + customer, carrier blank                              | 2–3         |
| No signatures (unsigned slip)  | If production includes unsigned slips                                   | 2–3         |

Microsoft requires: *"To train a custom neural model with signature detection, you need to use at least five samples with signature labeled along with variations to get the most accurate results."* citeturn7search168

### Dimension 5 — Optional/conditional fields

| Variation                    | Example                                           |
| ---------------------------- | ------------------------------------------------- |
| Notes present                | Image 2: "JIT CONTRACT ORDER - PRIORITY SHIPMENT" |
| Notes absent                 | Image 1, Documents 1 and 2                        |
| Route present                | Image 1: "5/19 prod."                             |
| Route absent                 | Image 2, Documents 1 and 2                        |
| Shipment Date present        | Image 1: "May 29, 2026"                           |
| Shipment Date absent         | Image 2                                           |
| Restrictions present (tarp)  | Image 1: "Tarp required CALL 24 HRS IN ADVANCE"   |
| Restrictions present (hours) | Image 2: "Mon-Fri 7am-3:30pm"                     |

Microsoft states: *"If your dataset contains documents with optional fields, validate that the training dataset has a few documents with the options represented."* citeturn7search157

### Dimension 6 — Exception/handwriting variation

| Variation               | Example                                                    |
| ----------------------- | ---------------------------------------------------------- |
| Exception note present  | `POD Scenario 3.PDF`: "Wrong Size Lumber" citeturn1search2 |
| Handwritten annotations | Image 2: "DSU" marks on quantity column                    |
| No exceptions           | Image 1, Document 1                                        |

### Dimension 7 — Load weight variation

| Variation                        | Example            |
| -------------------------------- | ------------------ |
| Actual Load Weight = 0.000 LB    | Image 1            |
| Estimated Load Weight with value | Image 2: 61,392 LB |

## 2.3 Recommended minimum training set

| Variation category          | Target count             | Rationale                                                    |
| --------------------------- | ------------------------ | ------------------------------------------------------------ |
| Clean PDFs                  | 5                        | Minimum per format type                                      |
| Scanned PDFs                | 5                        | Minimum per format type                                      |
| Mobile photos (JPEG)        | 5                        | Minimum per format type; lower quality requires more samples |
| Different branches          | 3–5 branches represented | Header address generalisation                                |
| Single product group        | 5                        | Table row variation                                          |
| Multiple product groups     | 5                        | Critical for Image 2-style documents                         |
| All signatures present      | 5                        | Signature detection minimum                                  |
| Missing signatures          | 3                        | Optional field representation                                |
| Exception/handwritten notes | 3–5                      | Exception detection training                                 |
| Notes field present         | 3                        | Optional field representation                                |
| **Total minimum**           | **20–30 documents**      | Covers all critical variation dimensions                     |

Microsoft states: *"Use a larger data set (10-15 images) if your form images are of lower quality."* citeturn7search175

**Training data limits:** The neural model training data cannot exceed 1 GB or 50,000 pages. citeturn7search176

***

# 3. Create the Studio project

## Step 1 — Open Studio

Navigate to Document Intelligence Studio → **Custom extraction model**.

## Step 2 — Create a project

1. Select **Create a project**.
2. Enter:
   * **Project name:** `Interfor-POD-DeliverySlip-Extraction`
   * **Description:** `Custom neural extraction model for Interfor delivery slip/POD documents. Extracts order, shipment, product lines, totals, POD signatures, exception notes. v4.0 2024-11-30.`
3. Select **Continue**.

Microsoft states: *"Start by navigating to the Document Intelligence Studio… select the Custom extraction model tile and select the Create a project button."* citeturn7search175

## Step 3 — Select or create the Document Intelligence resource

1. Select the Document Intelligence resource (`di-pod-prod`).
2. **Critical:** Custom neural models are only available in specific Azure regions. Microsoft states: *"Custom neural models are only available in a few regions. If you plan on training a neural model, select or create a resource in one of these supported regions."* citeturn7search175

Supported regions include: Australia East, Brazil South, Canada Central, Central India, Central US, East Asia, East US, East US2, France Central, Japan East, South Central US, Southeast Asia, UK South, West Europe, West US2, US Gov Arizona, US Gov Virginia. citeturn7search168

3. Select **Continue**.

## Step 4 — Connect storage

1. Select the storage account: `stpodidpprod`.
2. Select the container.
3. Enter the **Folder path**: `training/extraction`
4. Select **Continue**.

Microsoft states: *"The Folder path should be empty if your training documents are in the root of the container. If your documents are in a subfolder, enter the relative path from the container root in the Folder path field."* citeturn7search175

## Step 5 — Review and create

Review the project configuration and select **Create Project**.

***

# 4. Upload and run Layout/OCR

## Step 1 — Upload training documents

Upload all training documents to the storage folder. The Studio lists them in the left panel.

## Step 2 — Run Layout for all documents

Microsoft states: *"The first time you upload documents you get the option to run layout or auto label for all documents."* citeturn7search166

1. Select **Run layout** for all documents.
2. Wait for all documents to complete. Each document generates a `{file}.ocr.json` file in blob storage.
3. If using the Free (F0) tier, rate limits may prevent all documents from being processed at once. Wait and retry, or upgrade to S0. citeturn7search166

## Step 3 — Verify OCR quality

For each document:

1. Open the document in the labelling workspace.
2. Verify that key text is highlighted and selectable:
   * Order Number value
   * Shipment Number value
   * Product table cells
   * Total values
   * Carrier name
   * Restriction text
3. Check for OCR failures on:
   * Handwritten text (signatures, exception notes, "DSU" marks)
   * Low-contrast areas
   * Clipped/cropped edges (especially mobile photos like Image 2)
4. If OCR quality is poor for a document, consider replacing it with a better-quality sample or accepting that the field may require region labelling.

***

# 5. Define fields

## 5.1 Field types available for custom neural models

Microsoft documents these capabilities for v4.0 neural models: citeturn7search168

| Capability                    | Custom neural      | Custom template |
| ----------------------------- | ------------------ | --------------- |
| Form fields (key-value pairs) | ✅ Supported        | ✅ Supported     |
| Selection marks               | ✅ Supported        | ✅ Supported     |
| Tabular fields (tables)       | ✅ Supported        | ✅ Supported     |
| Signature                     | ✅ Supported (v4.0) | ✅ Supported     |
| Region labelling              | ✅ Supported        | ✅ Supported     |
| Overlapping fields            | ✅ Supported (v4.0) | ❌ Not supported |

There are four field types available in the Studio labelling interface: citeturn7search166

1. **Field** — a text field (String, Date, Number, etc.)
2. **Selection mark** — a true/false value based on a checkmark
3. **Signature** — an image from within a bounding box (draw region)
4. **Table** — a dynamic or fixed table with columns and rows

## 5.2 Field naming best practices

Microsoft states: *"For custom neural models, use semantically relevant names for fields. For example, if the value being extracted is Effective Date, name it effective\_date or EffectiveDate not a generic name like date1."* citeturn7search157

Apply this to the POD scenario:

| ❌ Bad name | ✅ Good name                 | Reason                           |
| ---------- | --------------------------- | -------------------------------- |
| `field1`   | `OrderNumber`               | Semantic relevance               |
| `date1`    | `OrderDate`                 | Distinguishes from ShipmentDate  |
| `address1` | `SoldToAddress`             | Distinguishes from ShipToAddress |
| `sig1`     | `CustomerDeliverySignature` | Identifies which signature       |

## 5.3 Complete field definition for the Interfor POD project

### Scalar fields (Field type)

Create each of these using the **+ Add a field** button → **Field**:

```text
OrderNumber          (String)
OrderDate            (Date)
CustomerNumber       (String)
Salesperson          (String)
SoldToName           (String)
SoldToAddress        (String)
ShipToName           (String)
ShipToAddress        (String)
ShipToPhone          (String)
CarrierName          (String)
CarrierPhone         (String)
Route                (String)       ← optional, present on some slips
ShipmentNumber       (String)
ShipmentDate         (Date)         ← optional, present on some slips
BranchLocation       (String)
Unit                 (String)
UnitNumber           (String)
Restrictions         (String)
Notes                (String)       ← optional, e.g. "JIT CONTRACT ORDER"
TotalUnits           (Number)
TotalPieces          (Number)
TotalQuantity        (String)       ← includes UOM, e.g. "23.578 MBF"
LoadWeight           (String)       ← e.g. "61,392 LB"
LoadWeightType       (String)       ← "Actual" or "Estimated"
PrintDateTime        (String)
ShipperSignatureDate (String)
CarrierSignatureDate (String)
CustomerSignatureDate(String)
ReceivedByName       (String)       ← handwritten names
HandwrittenNoteText  (String)       ← exception text, e.g. "Wrong Size Lumber"
HandwrittenAnnotations(String)      ← e.g. "DSU" marks
ExceptionReferenceNumber(String)    ← e.g. "# 0013856"
```

### Signature fields (Signature type)

Create three separate Signature-type fields:

```text
ShipperSignature
CarrierSignature
CustomerDeliverySignature
```

**Critical labelling rule:** Microsoft states: *"To label a signature, use field type as Signature and draw the regions for signature. Signature field only supports one draw region per field."* citeturn7search168

Each of these three fields must be:

* Created with **field type = Signature**
* Labelled by **drawing a region** (bounding box) around the signature area
* Limited to **one draw region per field**

**Training requirement for signatures:** Microsoft states: *"To train a custom neural model with signature detection, you need to use at least five samples with signature labeled along with variations to get the most accurate results."* citeturn7search168

### Table field (Table type)

Create one dynamic table named `ProductLines`:

```text
ProductLines (dynamic table)
  Columns:
    ProductDescription    (String)
    ProductCode           (String)
    Length                (String)
    Units                 (Number)
    Pieces                (Number)
    PcsPerUnit            (Number)
    Quantity              (Number)
    UOM                   (String)
```

After creating the table field, select it in the right panel to customise columns. You can rename, insert, and delete columns using the column menu. citeturn7search166

***

# 6. Label the training documents

This is the most time-intensive and accuracy-critical phase.

## 6.1 Core labelling workflow

Microsoft states: *"Start labeling your dataset and creating your first field by selecting the plus (➕) button… Enter a name for the field. Assign a value to the field by choosing a word or words in the document. Select the field in either the dropdown or the field list on the right navigation bar."* citeturn7search175

For each training document:

1. Open the document in the labelling workspace.
2. For each field in the field list:
   * Locate the value in the document.
   * Select the word(s) that represent the value.
   * Assign the selection to the field.
3. Repeat for all fields and all documents.

## 6.2 Labelling techniques — when to use each

### Technique 1 — Word/text selection (default)

Use for most printed text fields:

* Click on individual words to select them.
* The selected words are assigned to the field.

**Example:** Select `2448058` and assign it to `OrderNumber`.

### Technique 2 — Shift-select (for multi-word spans)

Microsoft states: *"When labeling a large span of text, rather than mark each word in the span, hold down the shift key as you're selecting the words to speed up labeling and ensure you don't miss any words in the span of text."* citeturn7search158

**Use for:** Multi-word values like addresses, carrier names, product descriptions.

**Example:** Hold Shift and select from `The Building Center Inc.` through `Pineville, NC 28134 US` to assign to `SoldToAddress`.

### Technique 3 — Region labelling (for bounding box selection)

Microsoft states: *"A second option for labeling larger spans of text is to use region labeling. When region labeling is used, the OCR results are populated in the value at training time."* citeturn7search158

**Use for:**

* Handwritten text that OCR may not select cleanly
* Signature regions (required for Signature fields)
* Exception notes
* Areas where text selection is difficult due to scan quality

**Important difference for neural models:** Microsoft states: *"Region labels in custom neural models use the results from the Layout API for specified region. This feature is different from template models where, if no value is present, text is generated at training time."* citeturn7search168

This means for neural models, if no OCR text exists in the drawn region, no synthetic text is injected. The model uses whatever the Layout API recognised.

### Technique 4 — Auto-label tables

Microsoft states: *"Tables can be challenging to label, when they have many rows or dense text. If the layout table extracts the result you need, you should just use that result and skip the labeling process. In instances where the layout table isn't exactly what you need, you can start with generating the table field from the values layout extracts. Start by selecting the table icon on the page and select on the auto label button. You can then edit the values as needed. Auto label currently only supports single page tables."* citeturn7search158

**Use for:** The product line table in Interfor delivery slips.

**Process:**

1. Select the table icon on the page where the product table appears.
2. Click **Auto label**.
3. Review the auto-generated column/row mappings.
4. Edit any incorrect mappings.
5. Add or remove columns to match the target schema.

**Limitation:** Auto-label only supports single-page tables. For tables spanning multiple pages, label manually.

### Technique 5 — Search

Microsoft states: *"The Studio now includes a search box for instances when you know you need to find specific words to label, but just don't know where to locate them in the document. Simply search for the word or phrase and navigate to the specific section in the document to label the occurrence."* citeturn7search158

**Use for:** Finding specific values like `Tarp required` or `JIT CONTRACT ORDER` in long documents.

### Technique 6 — Overlapping fields

Microsoft states: *"Overlapping fields are supported for fields and table cells. If you expect your analyze results to contain overlapping fields, you should add at least one sample to the training dataset with the specific field overlaps labeled. To label an overlapping field, use the region labeling feature to select the regions for each field. Both complete and partial overlaps are supported."* citeturn7search158

**Limits for overlapping fields:** citeturn7search168

* Any token or word can only be labelled as **two fields maximum**.
* Overlapping fields in a table **cannot span table rows**.
* At least **one sample** with the specific overlap must exist in the training dataset.
* Use **region labelling** (not field selection) to designate overlapping spans.

**Potential overlap in POD scenario:** Product description text and product code may spatially overlap (e.g., "Southern Yellow Pine 2×4 No.2 S4S Eased Edge Kiln Dried and Heat Treated **SY24#2**" — where `ProductDescription` and `ProductCode` share the same line).

## 6.3 Field-by-field labelling guide for Interfor POD

### Order section

| Field            | Where to find it         | Labelling technique    | Notes                            |
| ---------------- | ------------------------ | ---------------------- | -------------------------------- |
| `OrderNumber`    | "Order Number:" value    | Word select            | e.g., `2448058`                  |
| `OrderDate`      | "Order Date:" value      | Word select            | e.g., `May 15, 2026`             |
| `CustomerNumber` | "Customer" value         | Word select            | e.g., `300905`                   |
| `Salesperson`    | "Salesperson:" value     | Word select            | e.g., `Will Shy`                 |
| `SoldToName`     | "Sold To:" first line    | Shift-select or region | e.g., `The Building Center Inc.` |
| `SoldToAddress`  | "Sold To:" address lines | Shift-select or region | Multi-line address               |

**Labelling rule:** Microsoft states: *"Don't include the surrounding text. For example when labeling a checkbox, name the field to indicate the check box selection."* citeturn7search157 Apply this to all fields — label only the **value**, not the label text itself. Do not include "Order Number:" in the `OrderNumber` label; only include `2448058`.

### Shipping section

| Field            | Where to find it         | Labelling technique    | Notes                                              |
| ---------------- | ------------------------ | ---------------------- | -------------------------------------------------- |
| `ShipToName`     | "Ship To:" first line    | Shift-select or region | e.g., `SHANE PRIDGEN`                              |
| `ShipToAddress`  | "Ship To:" address lines | Shift-select or region | Multi-line                                         |
| `ShipToPhone`    | "Phone:" value           | Word select            | e.g., `910-546-4700`                               |
| `CarrierName`    | "Carrier:" value         | Shift-select           | e.g., `CHRISTOPHER WOODS dba FIRE TRAIL TRANSPORT` |
| `CarrierPhone`   | Carrier phone if present | Word select            | May not be present on all slips                    |
| `Route`          | "Route:" value           | Word select            | e.g., `5/19 prod.` — optional field                |
| `ShipmentNumber` | "Shipment" value         | Word select            | e.g., `5285729`                                    |
| `ShipmentDate`   | "Shipment Date:" value   | Word select            | Optional — present on some slips                   |
| `BranchLocation` | "Branch/Location" value  | Shift-select           | e.g., `Preston / PS`                               |
| `Unit`           | "Unit:" value            | Word select            | e.g., `Truck`                                      |
| `UnitNumber`     | "Unit #:" value          | Word select            | e.g., `03`                                         |
| `Restrictions`   | "Restrictions:" value    | Shift-select or region | e.g., `Tarp required CALL 24 HRS IN ADVANCE`       |

### Notes section

| Field   | Where to find it        | Labelling technique    | Notes                                                     |
| ------- | ----------------------- | ---------------------- | --------------------------------------------------------- |
| `Notes` | "NOTES" section content | Shift-select or region | e.g., `JIT CONTRACT ORDER - PRIORITY SHIPMENT` — optional |

### Product table

| Step | Action                                                                                                                                                           |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Locate the product information section.                                                                                                                          |
| 2    | Select the table icon on the page.                                                                                                                               |
| 3    | Click **Auto label** to generate initial table structure.                                                                                                        |
| 4    | Review and correct column mappings to match: `ProductDescription`, `ProductCode`, `Length`, `Units`, `Pieces`, `PcsPerUnit`, `Quantity`, `UOM`.                  |
| 5    | If the document has **multiple product groups** (like Image 2 with HF242+ and HF24STUD), label all rows from both groups into the **same** `ProductLines` table. |
| 6    | For cross-page tables, label each row across pages in a **single table**.                                                                                        |

Microsoft states: *"Tabular fields support cross page tables by default. To label a table that spans multiple pages, label each row of the table across the different pages in a single table."* citeturn7search168

Microsoft also states: *"Tabular fields are also useful when extracting repeating information within a document that isn't recognized as a table."* citeturn7search168

### Totals section

| Field            | Where to find it           | Labelling technique | Notes                           |
| ---------------- | -------------------------- | ------------------- | ------------------------------- |
| `TotalUnits`     | "Total" row, Units column  | Word select         | e.g., `18`                      |
| `TotalPieces`    | "Total" row, Pieces column | Word select         | e.g., `3,536`                   |
| `TotalQuantity`  | "Total" row, Quantity+UOM  | Shift-select        | e.g., `23.578 MBF`              |
| `LoadWeight`     | Load Weight value          | Shift-select        | e.g., `61,392 LB` or `0.000 LB` |
| `LoadWeightType` | Label text prefix          | Word select         | `Actual` or `Estimated`         |

### Signatures

| Field                       | Labelling technique                                   | Critical rule                              |
| --------------------------- | ----------------------------------------------------- | ------------------------------------------ |
| `ShipperSignature`          | **Draw region** around shipper signature area         | Signature type, one draw region per field  |
| `CarrierSignature`          | **Draw region** around carrier signature area         | Signature type, one draw region per field  |
| `CustomerDeliverySignature` | **Draw region** around customer signature area        | Signature type, one draw region per field  |
| `ShipperSignatureDate`      | Word select or region on date near shipper signature  | String type, separate from Signature field |
| `CarrierSignatureDate`      | Word select or region on date near carrier signature  | String type                                |
| `CustomerSignatureDate`     | Word select or region on date near customer signature | String type, e.g., `5/26/26`               |
| `ReceivedByName`            | Region label on handwritten name                      | String type, e.g., "Dylan King"            |

### Exception fields

| Field                      | Labelling technique                          | Notes                                                                |
| -------------------------- | -------------------------------------------- | -------------------------------------------------------------------- |
| `HandwrittenNoteText`      | Region label over handwritten exception text | e.g., "Wrong Size Lumber" from `POD Scenario 3.PDF` citeturn1search2 |
| `HandwrittenAnnotations`   | Region label over annotation marks           | e.g., "DSU" from Image 2                                             |
| `ExceptionReferenceNumber` | Word select or region                        | e.g., `# 0013856` from `POD Scenario 3.PDF` citeturn1search2         |

## 6.4 Labelling consistency rules

Microsoft provides these critical labelling guidelines: citeturn7search157

| Rule                     | Description                                                                                                                                       | POD scenario example                                                                                                                        |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Consecutive sequence** | *"Value tokens/words of one field must be either in a consecutive sequence in natural reading order, without interleaving with other fields."*    | When labelling `SoldToAddress`, select the full address block consecutively. Do not skip words.                                             |
| **Representative data**  | *"Values in training cases should be diverse and representative. For example, if a field is named date, values for this field should be a date."* | `OrderDate` should have different dates across training samples. Do not use the same date in all five samples.                              |
| **No surrounding text**  | *"Labeling values is required. Don't include the surrounding text."*                                                                              | For `OrderNumber`, label only `2448058`, not `Order Number: 2448058`.                                                                       |
| **Consistent context**   | *"If a value appears in multiple contexts within the document, consistently pick the same context across documents to label the value."*          | If `BranchLocation` appears in both the header and the shipment section, always label the one in the shipment section across all documents. |

## 6.5 Handling optional/missing fields

For documents where a field does not exist (e.g., `Notes` is absent on Image 1, `Route` is absent on Image 2):

* **Do not label the field** in that document. Simply skip it.
* Microsoft states: *"If your dataset contains documents with optional fields, validate that the training dataset has a few documents with the options represented."* citeturn7search157

The model learns that the field is optional and may or may not appear.

***

# 7. Train the model

## 7.1 Initiate training

1. Select the **Train** button in the upper-right corner of Studio.
2. Enter a **unique model ID**:

```text
interfor-pod-extraction-v001
```

3. Optionally enter a **description**:

```text
Custom neural extraction model for Interfor delivery slip/POD. v4.0 2024-11-30.
Extracts order, shipment, product lines, totals, signatures, exception notes.
Training dataset: 25 documents. Branches: Preston, Port Angeles, DeQuincy.
```

4. Select **Build mode: Neural**.

Microsoft states: *"The Build operation supports template and neural custom models… set the buildMode to neural."* citeturn7search168

## 7.2 Neural model build REST API reference

The underlying REST API call is:

```http
POST https://{endpoint}/documentintelligence/documentModels:build?api-version=2024-11-30

{
  "modelId": "interfor-pod-extraction-v001",
  "description": "Custom neural extraction model for Interfor POD",
  "buildMode": "neural",
  "azureBlobSource": {
    "containerUrl": "https://stpodidpprod.blob.core.windows.net/training",
    "prefix": "extraction/"
  },
  "maxTrainingHours": 10
}
```

citeturn7search168

## 7.3 `maxTrainingHours` — billing and duration control

Microsoft states: *"With version v4.0 2024-11-30 (GA), you can receive 10 hours of free model training, and train a model for as long as 10 hours. You can choose to spend all of 10 free hours on a single model build with a large set of data, or utilize it across multiple builds by adjusting the maximum duration value for the build operation by specifying maxTrainingHours."* citeturn7search168

| Training parameter               | Value                                 | Notes                                                 |
| -------------------------------- | ------------------------------------- | ----------------------------------------------------- |
| Free training hours/month        | 10 hours                              | Shared across all neural model builds in the resource |
| `maxTrainingHours` default       | Not specified (uses available budget) | Set explicitly to control cost                        |
| Paid training                    | After free hours are exhausted        | Billing per hour; see pricing page                    |
| Minimum billing per training job | 30 minutes                            | Even if training completes faster                     |

**Recommendation for this POD scenario:**

* For the initial training run with 20–30 documents, set `maxTrainingHours` to `1` or `2`. This is sufficient for a small dataset and preserves free hours for retraining iterations.
* Monitor actual training time:

```http
GET /documentModels/interfor-pod-extraction-v001

{
  "modelId": "interfor-pod-extraction-v001",
  "trainingHours": 0.23,
  ...
}
```

Microsoft states: *"Each build takes a different amount of time depending on the type and size of the training dataset. Billing is calculated for the actual time spent training the neural model with a minimum of 30 minutes per training job."* citeturn7search168

## 7.4 Training time expectations

| Build mode | Typical duration | Notes                                      |
| ---------- | ---------------- | ------------------------------------------ |
| Template   | 1–5 minutes      | Fast but requires consistent visual layout |
| Neural     | 20–60+ minutes   | Longer; depends on dataset size            |

Microsoft states: *"Template models train in a few minutes. Neural models can take up to 30 minutes to train."* citeturn7search175 With v4.0 and `maxTrainingHours`, neural training can run much longer for larger datasets.

## 7.5 Monitor training status

Navigate to the **Models** menu in Studio to view the training operation status.

Microsoft states: *"Navigate to the Models menu to view the status of the train operation."* citeturn7search175

***

# 8. Test the model

## 8.1 Test in Studio

1. Select the trained model from the Models list.
2. Select the **Test** button.
3. Select **+ Add** to upload a test file **not used in training**.
4. Select **Analyze**.

Microsoft states: *"Select the model and select on the Test button. Select the + Add button to select a file to test the model… choose the Analyze button to test the model."* citeturn7search175

## 8.2 Evaluate results

Review:

| Evaluation area         | What to check                         | Where in Studio                             |
| ----------------------- | ------------------------------------- | ------------------------------------------- |
| Extracted field values  | Are they correct?                     | Right navigation bar — field list           |
| Field confidence scores | Are they above threshold?             | Right navigation bar — confidence per field |
| Table structure         | Are rows and columns correct?         | Right navigation bar — table view           |
| Table confidence        | Table, row, and cell confidence       | JSON response                               |
| Signature detection     | Are signatures detected?              | Field list — signature fields               |
| Missing fields          | Are optional fields correctly absent? | Field list                                  |
| Handwritten text        | Are exception notes captured?         | Field list — HandwrittenNoteText            |
| JSON response           | Is the structure correct?             | Right navigation bar — JSON tab             |
| Sample code             | Is the SDK code usable?               | Right navigation bar — code tab             |

Microsoft states: *"The model results are displayed in the main window and the fields extracted are listed in the right navigation bar… Validate your model by evaluating the results for each field. The right navigation bar also has the sample code to invoke your model and the JSON results from the API."* citeturn7search175

## 8.3 Table confidence evaluation (v4.0 feature)

Microsoft states: *"Fixed or dynamic tables add confidence support for the following elements: Table confidence, a measure of how accurately the entire table is recognized. Row confidence, a measure of recognition of an individual row. Cell confidence, a measure of recognition of an individual cell. The recommended approach is to review the accuracy in a top-down manner starting with the table first, followed by the row and then the cell."* citeturn7search168

For the POD product table, evaluate:

1. **Table confidence** — Is the entire product table detected?
2. **Row confidence** — Are individual product lines correctly separated?
3. **Cell confidence** — Are individual values (Length, Units, Pieces, etc.) in the right cells?

## 8.4 Test matrix

Run these test categories:

| Test ID | Document type                           | Expected outcome                                   | Key fields to validate                                        |
| ------- | --------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------- |
| T01     | Clean PDF, single product group         | All fields extracted with high confidence          | OrderNumber, ShipmentNumber, ProductLines, TotalQuantity      |
| T02     | Mobile photo, single product group      | All fields extracted; confidence may be lower      | Same + ShipperSignature, CustomerDeliverySignature            |
| T03     | Scanned PDF, multiple product groups    | Both product groups in ProductLines table          | ProductLines row count, multiple ProductCode values           |
| T04     | Document with Notes field               | Notes extracted                                    | Notes = "JIT CONTRACT ORDER…"                                 |
| T05     | Document with Route field               | Route extracted                                    | Route value                                                   |
| T06     | Document with Shipment Date             | ShipmentDate extracted                             | ShipmentDate value                                            |
| T07     | Document with exception note            | HandwrittenNoteText extracted                      | HandwrittenNoteText = "Wrong Size Lumber"                     |
| T08     | Document with DSU annotations           | HandwrittenAnnotations extracted                   | HandwrittenAnnotations value                                  |
| T09     | Document with all signatures            | All three signature fields detected                | ShipperSignature, CarrierSignature, CustomerDeliverySignature |
| T10     | Document with missing carrier signature | CarrierSignature absent, others present            | Verify absent field not falsely detected                      |
| T11     | Document with zero load weight          | LoadWeight = "0.000 LB", LoadWeightType = "Actual" | Verify exact value extraction                                 |
| T12     | Low-quality mobile photo                | Evaluate degradation                               | Compare confidence scores against T01/T02                     |

## 8.5 Iterative improvement

If test results show weak fields:

1. Identify which fields have low confidence or incorrect values.
2. Add the problematic test document back to the training dataset.
3. Label the fields correctly in that document.
4. Retrain with a **new model ID**: `interfor-pod-extraction-v002`.

Microsoft states: *"Congratulations you learned to train a custom model in the Document Intelligence Studio! Your model is ready for use with the REST API or the SDK to analyze documents."* citeturn7search175

***

# 9. Advanced considerations

## 9.1 Region labelling behaviour for neural models

Microsoft states: *"Region labels in custom neural models use the results from the Layout API for specified region. This feature is different from template models where, if no value is present, text is generated at training time."* citeturn7search168

**Implication for POD scenario:** If you draw a region around a handwritten exception note and the Layout API did not recognise the handwritten text, the neural model receives an empty value for that region. This means handwritten text extraction accuracy depends on the Layout API's OCR quality for handwriting.

**Mitigation:** Include diverse handwriting samples in training. If OCR consistently fails on certain handwriting styles, consider post-processing with a separate handwriting recognition step or flagging for human review based on the Signature/region confidence score.

## 9.2 Contiguous value labelling rule

Microsoft states: *"Value tokens/words of one field must be either: In a consecutive sequence in natural reading order, without interleaving with other fields; In a region that don't cover any other fields."* citeturn7search168

**Implication for POD scenario:** When labelling `SoldToAddress`, all address words must be consecutive. Do not interleave with `CustomerNumber` or other fields. If the visual layout places the customer number within the address block, use region labelling to isolate each field.

## 9.3 Model versioning convention

Use explicit versioned model IDs:

```text
interfor-pod-extraction-v001    ← initial training
interfor-pod-extraction-v002    ← after first improvement cycle
interfor-pod-extraction-v003    ← after second improvement cycle
```

Never overwrite a model ID. Keep previous versions available for regression comparison and rollback.

## 9.4 Copy model across regions

If the model was trained in a supported region but needs to be used in a different region, Microsoft supports model copying:

*"You can copy a model trained in one of the select regions listed to any other region and use it accordingly. Use the REST API or Document Intelligence Studio to copy a model to another region."* citeturn7search168

***

# 10. Deliverables from Phase 3

| Deliverable                           | Owner                     | Description                                                |
| ------------------------------------- | ------------------------- | ---------------------------------------------------------- |
| Trained model ID                      | Developer / model builder | `interfor-pod-extraction-v001`                             |
| `fields.json`                         | Model builder             | Locked field schema                                        |
| Labelled training set in blob storage | Model builder / BA        | All training documents with `.labels.json` and `.ocr.json` |
| Test result set                       | QA / model builder        | Field-level accuracy and confidence for each test document |
| Table confidence report               | Model builder             | Table, row, and cell confidence scores from v4.0           |
| Signature detection report            | Model builder             | Accuracy of three signature fields across test documents   |
| JSON response samples                 | Developer                 | Actual API responses from Studio test                      |
| Sample SDK code                       | Developer                 | From Studio right-panel code tab                           |
| Known limitation register             | Model builder / BA        | Documents/fields that do not extract reliably              |
| Training time and cost log            | Model builder             | `trainingHours` from model metadata, free hours remaining  |
| Model version register                | Solution specialist       | Model ID, creation date, dataset version, test results     |

***

# Accuracy checks — Phase 3 specific

| Claim                                                                                   | Source                                                                                                                                               | Status                                       |
| --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| Labelled dataset consists of `fields.json`, `{file}.ocr.json`, and `{file}.labels.json` | [Best practices for labeling](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-labels?view=doc-intel-4.0.0)    | ✅ Confirmed citeturn7search157               |
| Balanced dataset = all typical variations represented                                   | Same source                                                                                                                                          | ✅ Confirmed citeturn7search157               |
| Neural models: ≤15 variations → single model; >15 → compose                             | Same source                                                                                                                                          | ✅ Confirmed citeturn7search157               |
| Semantically relevant field names improve accuracy                                      | Same source                                                                                                                                          | ✅ Confirmed citeturn7search157               |
| Do not include surrounding text when labelling values                                   | Same source                                                                                                                                          | ✅ Confirmed citeturn7search157               |
| Signature field type = Signature, draw region, one region per field                     | [Custom neural model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)            | ✅ Confirmed citeturn7search168               |
| ≥5 signature samples with variations required                                           | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| v4.0 supports table, row, and cell confidence                                           | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| Overlapping fields supported; use region labelling; max 2 fields per token              | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| Neural region labels use Layout API results; no synthetic text injected                 | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| `buildMode: neural` in REST API build operation                                         | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| `maxTrainingHours` controls training duration and cost                                  | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| 10 free training hours/month; paid after that; 30-min minimum per job                   | Same source                                                                                                                                          | ✅ Confirmed citeturn7search168               |
| Neural training data max 1 GB, 50,000 pages                                             | [Service limits](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/service-limits?view=doc-intel-4.0.0)                      | ✅ Confirmed citeturn7search176               |
| Auto-label tables supports single-page tables only                                      | [Labeling tips](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-label-tips?view=doc-intel-4.0.0)              | ✅ Confirmed citeturn7search158               |
| Shift-select for multi-word spans                                                       | Same source                                                                                                                                          | ✅ Confirmed citeturn7search158               |
| Search box available in Studio for finding specific words                               | Same source                                                                                                                                          | ✅ Confirmed citeturn7search158               |
| Studio project creation: name → resource → storage → folder path → create               | [Build and train](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/how-to-guides/build-a-custom-model?view=doc-intel-4.0.0) | ✅ Confirmed citeturn7search175               |
| Neural models only available in specific Azure regions                                  | Same source + neural model doc                                                                                                                       | ✅ Confirmed citeturn7search175turn7search168 |
| 5 documents minimum to start; 10–15 for lower-quality forms                             | Same source                                                                                                                                          | ✅ Confirmed citeturn7search175               |
| Models can be copied to other regions                                                   | [Custom neural model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)            | ✅ Confirmed citeturn7search168               |

***

This concludes the Phase 3 deep dive. The next logical step would be to execute this labelling and training plan with the actual production documents, evaluate results against the test matrix, and then proceed to Phase 4 (classification, if needed) and Phase 6 (Logic Apps / Power Platform orchestration hand-off).

Continuing the Phase 3 deep dive with the remaining advanced topics: **table confidence evaluation**, **accuracy and confidence score interpretation**, **composed models**, **model copy operations**, and **production troubleshooting**.

***

# 11. Table confidence evaluation — deep dive

Tables are the most operationally critical extraction target in the Interfor POD scenario. The product line table contains Length, Units, Pieces, Pcs/Unit, Quantity, and UOM — the values used for receipt reconciliation, AP matching, and claims processing. Image 2 shows two separate product groups (HF242+ and HF24STUD) with sub-totals per group and a grand total. citeturn1search2 Getting table extraction right is non-negotiable.

## 11.1 Three levels of table confidence (v4.0)

Microsoft states: *"Fixed or dynamic tables add confidence support for the following elements: Table confidence, a measure of how accurately the entire table is recognized. Row confidence, a measure of recognition of an individual row. Cell confidence, a measure of recognition of an individual cell. The recommended approach is to review the accuracy in a top-down manner starting with the table first, followed by the row and then the cell."* citeturn8search165

### Evaluation hierarchy

```text
Level 1: Table confidence
  → Was the table itself detected correctly?
  → Are the boundaries right?
  → Is the column count correct?

Level 2: Row confidence
  → Is each product line correctly separated?
  → Are sub-total and total rows correctly identified?
  → Are cross-page rows handled?

Level 3: Cell confidence
  → Is each value (Length, Units, Pieces, Pcs/Unit, Quantity, UOM) in the right cell?
  → Is the numeric value transcribed correctly?
```

## 11.2 Common questions and answers from Microsoft documentation

### Can cells have high confidence while the row has low confidence?

**Yes.** Microsoft states: *"A correctly predicted cell that belongs to a row with other possible misses would have high cell confidence, but the row's confidence should be low. Similarly, a correct row in a table with challenges with other rows would have high row confidence whereas the table's overall confidence would be low."* citeturn8search165

**POD scenario implication:** If a product line has 5 out of 6 cells correct but one cell (e.g., `PcsPerUnit`) is wrong, the row confidence should be lower than the correctly extracted cells. Use row confidence as the trigger for line-level review.

### How do merged cells affect confidence?

Microsoft states: *"Regardless of the type of table, the expectation for merged cells is that they should have lower confidence values. Furthermore, the cell that is missing (because it was merged with an adjacent cell) should have NULL value with lower confidence as well."* citeturn8search165

**POD scenario implication:** The Sub-Total and Total rows in the Interfor delivery slip often have merged cells (e.g., the Sub-Total row skips `Length` and `Pcs/Unit` columns). Expect lower confidence on these rows. Handle them as summary/validation rows rather than product lines.

### What about optional/empty cells?

Microsoft states: *"If your training dataset is representative of the optionality of cells, it helps the model know how often a value tends to appear in the training set, and thus what to expect during inference. This feature is used when computing the confidence of either a prediction or of making no prediction at all (NULL). You should expect an empty field with high confidence for missing values that are mostly empty in the training set too."* citeturn8search165

**POD scenario implication:** Some product lines may not have a `PcsPerUnit` value, or the `UOM` may be implied rather than explicitly printed. If the training dataset reflects this, the model should produce high-confidence NULL for those cells.

### Cross-page table rows

Microsoft states: *"Expect the cell confidence to be high and row confidence to be potentially lower than rows that aren't split. The proportion of split rows in the training data set can affect the confidence score."* citeturn8search165

**POD scenario implication:** If any production delivery slips have product tables that span page boundaries, include those in training. The model will learn to handle them, but expect lower row confidence on split rows.

## 11.3 Recommended table confidence evaluation template

For each test document, capture this matrix:

```text
Document: [filename]
Table: ProductLines

┌──────────┬───────────────┬──────────────┬─────────────┐
│ Level    │ Confidence    │ Expected     │ Pass/Fail   │
├──────────┼───────────────┼──────────────┼─────────────┤
│ Table    │ 0.xx          │ ≥ 0.85       │             │
├──────────┼───────────────┼──────────────┼─────────────┤
│ Row 1    │ 0.xx          │ ≥ 0.80       │             │
│ Row 2    │ 0.xx          │ ≥ 0.80       │             │
│ Row 3    │ 0.xx          │ ≥ 0.80       │             │
│ SubTotal │ 0.xx          │ ≥ 0.70 *     │             │
│ Total    │ 0.xx          │ ≥ 0.70 *     │             │
├──────────┼───────────────┼──────────────┼─────────────┤
│ Cells    │ Per-cell      │ ≥ 0.80       │             │
└──────────┴───────────────┴──────────────┴─────────────┘

* Sub-total and total rows may have merged cells → expect lower confidence
```

## 11.4 Table confidence decision rules for the orchestration

```text
If table confidence < 0.70
    → Requires Review: entire product table unreliable

If any row confidence < 0.60
    → Requires Review: specific product line unreliable

If any cell confidence < 0.50 on critical columns (Quantity, Pieces)
    → Requires Review: specific cell unreliable

If sub-total/total row confidence is high AND quantity reconciliation passes
    → Accept table even if individual cells are marginal
```

## 11.5 Best practice for table evaluation

Microsoft recommends a top-down approach: *"The recommended approach is to review the accuracy in a top-down manner starting with the table first, followed by the row and then the cell."* citeturn8search165

For the POD scenario, this means:

```text
Step 1: Check table-level confidence → is the table detected at all?
Step 2: Check row count → does it match the expected number of product lines?
Step 3: Check row confidence → are individual lines reliable?
Step 4: Check cell confidence → are critical numeric values (Quantity, Pieces) reliable?
Step 5: Cross-validate → does sum(line Quantities) = TotalQuantity?
```

***

# 12. Accuracy vs confidence — interpretation guide

This distinction is critical for solution specialists and BAs. Microsoft explicitly distinguishes between the two: citeturn8search165

## 12.1 Definitions

| Metric                 | What it measures                                                               | When it is produced        | Who uses it                                                             |
| ---------------------- | ------------------------------------------------------------------------------ | -------------------------- | ----------------------------------------------------------------------- |
| **Estimated accuracy** | Per-field aggregate performance estimated from labelled evaluation data        | At training time           | Model builder, solution specialist — for model selection and comparison |
| **Confidence**         | Per-prediction certainty for a specific extracted field in a specific document | At analysis/inference time | Developer, orchestration logic — for accept/review/reject routing       |

Microsoft states: *"Accuracy scores for custom models: Custom neural and generative models don't provide accuracy scores during training."* citeturn8search165

**Critical implication:** For custom **neural** models, you do **not** get a per-field estimated accuracy score after training. You must evaluate accuracy yourself by running the trained model against a holdout test set and comparing extracted values to ground truth.

Custom **template** models do provide estimated accuracy scores.

## 12.2 The accuracy × confidence matrix

Microsoft provides this interpretation framework: citeturn8search165

| Accuracy | Confidence | Interpretation                                                                   | Action                                                             |
| -------- | ---------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **High** | **High**   | Model is performing well. Balanced training dataset.                             | Accept automatically.                                              |
| **High** | **Low**    | Analysed document looks different from training data.                            | Add more labelled documents. Consider adding a new model variant.  |
| **Low**  | **High**   | Unlikely. If it occurs, add more labelled data or split distinct document types. | Add training data. Evaluate composed models.                       |
| **Low**  | **Low**    | Insufficient training data or visually distinct documents mixed.                 | Add more labelled data. Split distinct types into separate models. |

## 12.3 Word confidence scores (v4.0)

Microsoft states: *"Field level confidence includes word confidence scores with 2024-11-30 (GA) API version for custom models."* citeturn8search165

This means the v4.0 API returns not just field-level confidence but also the confidence of individual words within the field. This is valuable for the POD scenario because:

* A field like `ShipToAddress` may have high overall field confidence but low word confidence on a specific word (e.g., a misspelled street name from poor scan quality).
* Validation logic can use word-level confidence to identify exactly which part of a multi-word field is unreliable.

## 12.4 Composite confidence evaluation

Microsoft recommends evaluating multiple confidence layers together: *"While evaluating confidence scores, you should also look at the underlying extraction confidence to generate a comprehensive confidence for the extracted result. Evaluate the OCR results for text extraction or selection marks depending on the field type to generate a composite confidence score for the field."* citeturn8search165

For the POD scenario, this means:

```text
Composite confidence for OrderNumber =
    min(
        field confidence(OrderNumber),
        word confidence(OrderNumber words),
        OCR confidence(underlying text)
    )

If composite confidence < threshold → Review
```

## 12.5 Recommended confidence thresholds for the Interfor POD

These are starting points. Calibrate against actual production results.

| Field category                        | Recommended threshold | Rationale                                                                  |
| ------------------------------------- | --------------------- | -------------------------------------------------------------------------- |
| OrderNumber                           | ≥ 0.85                | ERP matching — high accuracy required                                      |
| ShipmentNumber                        | ≥ 0.85                | TMS matching — high accuracy required                                      |
| CustomerNumber                        | ≥ 0.80                | Customer identification                                                    |
| BranchLocation                        | ≥ 0.75                | Less critical; limited set of known values                                 |
| ProductLines cells (Quantity, Pieces) | ≥ 0.80                | Financial reconciliation                                                   |
| ProductLines cells (Length, UOM)      | ≥ 0.70                | Operational but less sensitive                                             |
| TotalQuantity                         | ≥ 0.85                | Cross-validation anchor                                                    |
| LoadWeight                            | ≥ 0.75                | Operational                                                                |
| CustomerDeliverySignature             | ≥ 0.70                | Presence detection is binary; confidence threshold can be lower            |
| HandwrittenNoteText                   | ≥ 0.50                | Handwriting OCR is inherently lower confidence; trigger review on presence |

## 12.6 Improving confidence scores

Microsoft provides these tips: citeturn8search165

```text
If readResults confidence is low → improve input document quality
If pageResults confidence is low → ensure documents are the same type
Consider human review workflows
Use forms with different values in each field
Use a larger training set → model learns fields with greater accuracy
```

For the POD scenario specifically:

* Include diverse branch/customer/product combinations in training.
* Include different signature handwriting styles.
* Include both clean and low-quality scans.
* Separate visually distinct variants if a single model cannot handle them.

***

# 13. Composed models — evaluation path

## 13.1 What changed in v4.0

Microsoft states: *"The v4.0 2024-11-30 (GA) model compose operation adds an explicitly trained classifier instead of an implicit classifier for analysis."* and *"The 2024-11-30 (GA) implementation of the model compose operation replaces the implicit classification from the earlier versions with an explicit classification step and adds conditional routing."* citeturn8search176

This is a fundamental change from v3.x. In v3.x, composed models used an **implicit** classifier (best-match routing). In v4.0, composed models require an **explicitly trained classifier** and support **confidence-based routing**.

## 13.2 Benefits of v4.0 composed models

Microsoft lists these benefits: citeturn8search176

| Benefit                             | Description                                                                                  |
| ----------------------------------- | -------------------------------------------------------------------------------------------- |
| **Incremental improvement**         | Consistently improve classifier quality by adding more samples.                              |
| **Complete control over routing**   | Provide a confidence threshold for document type routing.                                    |
| **Ignore specific document types**  | Don't force extraction on unrecognised types.                                                |
| **Multiple instances of same type** | When paired with `split` mode, process multiple documents of the same type in a single file. |
| **Add-on feature support**          | Query fields, barcodes, etc. can be specified per analysis model.                            |
| **Up to 500 custom models**         | The composed model can contain up to 500 assigned models.                                    |

## 13.3 When to evaluate composed models for the POD scenario

Evaluate composed models if:

```text
Scenario 1:
  A single neural model for all POD variants shows inconsistent accuracy.
  For example:
    - Clean PDF accuracy: 95%
    - Mobile photo accuracy: 72%
    - Exception POD accuracy: 68%

  Action:
    Train separate models:
      interfor-pod-extraction-clean-v001
      interfor-pod-extraction-photo-v001
      interfor-pod-extraction-exception-v001

    Train a classifier:
      interfor-pod-quality-classifier-v001
      Classes: CleanPDF, MobilePhoto, ExceptionPOD

    Compose into a single model:
      interfor-pod-extraction-composed-v001

Scenario 2:
  Different Interfor branches use slightly different form layouts.
  For example:
    - Preston/PS uses one header layout
    - Port Angeles/PA uses a different header layout

  Action:
    If a single neural model handles both → no composition needed
    If accuracy degrades across branches → train per-branch models and compose

Scenario 3:
  The intake includes both Interfor delivery slips and other document types
  (BOL, invoice, carrier doc) that each need different extraction models.

  Action:
    This is the classic composed model use case.
    Train a classifier and compose all extraction models.
```

## 13.4 Compose model REST API

The v4.0 compose operation requires: citeturn8search184

```http
POST {endpoint}/documentintelligence/documentModels:compose?api-version=2024-11-30
```

Request body:

```json
{
  "modelId": "interfor-pod-extraction-composed-v001",
  "description": "Composed model for Interfor POD variants",
  "classifierId": "interfor-pod-quality-classifier-v001",
  "split": "auto",
  "docTypes": {
    "CleanPDF": {
      "modelId": "interfor-pod-extraction-clean-v001"
    },
    "MobilePhoto": {
      "modelId": "interfor-pod-extraction-photo-v001"
    },
    "ExceptionPOD": {
      "modelId": "interfor-pod-extraction-exception-v001"
    }
  }
}
```

The compose operation returns `202 Accepted` with `Operation-Location` and `Retry-After` headers. citeturn8search184

**Key parameters:**

* `classifierId` (required): The explicitly trained classifier that routes documents.
* `docTypes` (required): Dictionary mapping classifier document types to extraction model IDs.
* `split` (optional): Controls file splitting behaviour (`none`, `perPage`, `auto`).
* `modelId` (required): The composed model's unique ID.

## 13.5 Billing for composed models

Microsoft states: *"Composed models are billed the same as individual custom models. The pricing is based on the number of pages analyzed by the downstream analysis model. Billing is based on the extraction price for the pages routed to an extraction model. With the addition of the explicit classification charges are incurred for the classification of all pages in the input file."* citeturn8search176

This means:

* Classification charges apply to **all pages** in the input.
* Extraction charges apply only to pages **routed to an extraction model**.
* Pages not matched to any extraction model incur classification cost only.

## 13.6 Composed model decision tree for the POD scenario

```text
Start with a single neural model.
    ↓
Test against all document variants.
    ↓
Is accuracy consistent across all variants?
    Yes → Use the single model. No composition needed.
    No  → Identify which variants degrade accuracy.
            ↓
        Can the issue be fixed by adding more training samples?
            Yes → Add samples, retrain, re-test.
            No  → Train separate models per variant.
                    ↓
                Train a classifier for variants.
                Compose models.
                Test composed model.
                Compare against single model.
                    ↓
                Composed model better?
                    Yes → Deploy composed model.
                    No  → Revisit training data quality.
```

***

# 14. Model copy operations — cross-region, DR, and environment promotion

## 14.1 Why copy matters

Production model management requires:

* **Environment promotion:** dev → test → staging → prod
* **Cross-region deployment:** Train in one region, deploy in another
* **Disaster recovery:** Backup model to a secondary region
* **Compliance:** Deploy to a specific region for data residency

## 14.2 Two-step copy process

### Step 1 — Authorise the copy on the **target** resource

```http
POST {target-endpoint}/documentintelligence/documentModels:authorizeCopy?api-version=2024-11-30

{
  "modelId": "interfor-pod-extraction-v001",
  "description": "Production copy of Interfor POD extraction model"
}
```

Response (200 OK): citeturn8search187

```json
{
  "targetResourceId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.CognitiveServices/accounts/{targetResource}",
  "targetResourceRegion": "canadacentral",
  "targetModelId": "interfor-pod-extraction-v001",
  "targetModelLocation": "https://{target}.cognitiveservices.azure.com/documentintelligence/documentModels/interfor-pod-extraction-v001",
  "accessToken": "<token>",
  "expirationDateTime": "2026-06-11T09:12:54.552Z"
}
```

### Step 2 — Execute the copy on the **source** resource

```http
POST {source-endpoint}/documentintelligence/documentModels/interfor-pod-extraction-v001:copyTo?api-version=2024-11-30
```

Body: the entire authorisation response from Step 1. citeturn8search171

Response: `202 Accepted` with `Operation-Location` and `Retry-After`.

## 14.3 Environment promotion strategy

```text
DEV environment:
  Resource: di-pod-dev (e.g., East US 2)
  Models trained and tested here.

TEST environment:
  Resource: di-pod-test (e.g., East US 2 or Canada Central)
  Copy model from DEV → TEST.
  Run integration tests.

PROD environment:
  Resource: di-pod-prod (e.g., Canada Central)
  Copy model from TEST → PROD after sign-off.
  Logic App / Power Automate points to PROD resource.

DR environment:
  Resource: di-pod-dr (e.g., West US 2)
  Copy model from PROD → DR.
  Keep DR resource warm for failover.
```

## 14.4 Copy vs rebuild

| Approach                  | When to use                                                   | Speed               | Risk                                             |
| ------------------------- | ------------------------------------------------------------- | ------------------- | ------------------------------------------------ |
| **Copy**                  | Source resource is active and accessible. Model is identical. | Fast (minutes)      | Zero — identical copy.                           |
| **Rebuild from metadata** | Source resource is decommissioned. Need to modify labels.     | Slow (full retrain) | Training may produce slightly different results. |

***

# 15. Production troubleshooting guide

## 15.1 Training issues

| Symptom                             | Likely cause                                          | Resolution                                                        |
| ----------------------------------- | ----------------------------------------------------- | ----------------------------------------------------------------- |
| Training fails immediately          | Insufficient documents (< 5)                          | Add more labelled documents.                                      |
| Training fails with storage error   | CORS not configured or RBAC insufficient              | Reconfigure CORS; verify Storage Blob Data Contributor role.      |
| Training takes excessively long     | Very large dataset or `maxTrainingHours` set too high | Reduce dataset size or set `maxTrainingHours` explicitly.         |
| Training produces low-quality model | Imbalanced or non-representative dataset              | Follow balanced dataset guidance; add variation samples.          |
| Training exhausts free hours        | Multiple neural training runs                         | Monitor `trainingHours` in model metadata; budget for paid hours. |

## 15.2 Labelling issues

| Symptom                                  | Likely cause                                                             | Resolution                                                                                                                                     |
| ---------------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Field not extracted at inference         | Field not labelled in enough training documents                          | Label the field in at least 5 documents.                                                                                                       |
| Table columns misaligned                 | Auto-label produced incorrect column mapping                             | Manually correct column assignments; re-train.                                                                                                 |
| Signature always returns null            | Field type is String instead of Signature                                | Change field type to Signature; use draw region.                                                                                               |
| Overlapping fields fail                  | More than 2 fields labelled on the same token                            | Reduce overlap to max 2 fields per token. citeturn8search180                                                                                   |
| Region label returns empty               | Layout API did not recognise text in the drawn region                    | Neural models use Layout results for regions — no synthetic text. citeturn8search180 Improve scan quality or add more region-labelled samples. |
| Inconsistent extraction across documents | Labelling inconsistency (different contexts labelled for the same field) | Audit all `.labels.json` files for consistency.                                                                                                |

## 15.3 Inference/analysis issues

| Symptom                                                     | Likely cause                                                     | Resolution                                                                                   |
| ----------------------------------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| API returns 429 Too Many Requests                           | Rate limit exceeded                                              | Implement retry with `Retry-After` header. Increase rate limit in Azure portal if needed.    |
| API returns 413 or large payload error                      | Document exceeds 500 MB (S0) or 4 MB (F0)                        | Compress or split the document. Use S0 tier.                                                 |
| Low confidence on previously high-confidence fields         | Document variant not seen in training                            | Add variant to training set and retrain.                                                     |
| Table not detected                                          | Document layout differs significantly from training samples      | Add documents with that layout to training set.                                              |
| Handwritten text not extracted                              | OCR did not recognise the handwriting                            | Include more handwriting samples in training. Consider separate handwriting processing step. |
| Signature detection returns "unsigned" on a signed document | Signature is too faint, too small, or not in the labelled region | Ensure signature area is consistently positioned; improve scan quality.                      |

## 15.4 Confidence calibration troubleshooting

Microsoft states: *"If the confidence score for the readResults object is low, improve the quality of your input documents. If the confidence score for the pageResults object is low, ensure that the documents you're analyzing are of the same type."* citeturn8search165

For the POD scenario:

```text
If field confidence is consistently low across all documents:
    → Training data quality issue. Retrain with better samples.

If field confidence is low on specific documents:
    → Document quality issue. Flag for review; do not retrain on bad data.

If field confidence is high but extracted value is wrong:
    → Model is confidently wrong. This is the most dangerous case.
    → Add that document to training with correct labels.
    → Retrain and specifically test for regression on that pattern.
```

***

# 16. Model lifecycle management

## 16.1 Model expiration

Neural models have a defined lifecycle. Microsoft documents that custom neural models trained with v3.0 and v3.1 expire. For v4.0, verify the current lifecycle policy. Plan for periodic retraining regardless of expiration to incorporate new document variants and improve accuracy.

## 16.2 Model version registry

Maintain a register for all models:

| Model ID                                | Type       | Build mode | Created    | Dataset version | Training hours | Test accuracy   | Status        | Notes                |
| --------------------------------------- | ---------- | ---------- | ---------- | --------------- | -------------- | --------------- | ------------- | -------------------- |
| `interfor-pod-extraction-v001`          | Extraction | Neural     | 2026-06-10 | DS-001          | 0.5            | See test matrix | Active — DEV  | Initial training     |
| `interfor-pod-extraction-v002`          | Extraction | Neural     | 2026-06-18 | DS-002          | 0.7            | See test matrix | Active — TEST | Added photo variants |
| `interfor-pod-classifier-v001`          | Classifier | —          | 2026-06-20 | CL-001          | —              | See test matrix | Active — DEV  | 2 classes            |
| `interfor-pod-extraction-composed-v001` | Composed   | —          | 2026-06-25 | —               | —              | See test matrix | Candidate     | Evaluating           |

## 16.3 Retirement checklist

Before retiring a model version:

```text
[ ] New model version is trained and tested.
[ ] New model passes all holdout test cases.
[ ] New model passes regression tests against previous test set.
[ ] New model is copied to all target environments.
[ ] Orchestration (Logic App / Power Automate) parameters are updated.
[ ] Old model ID is documented as retired.
[ ] Old model is kept for 30 days minimum for rollback.
[ ] Old model is deleted after retention period.
```

***

# 17. Complete Phase 3 deliverable checklist

| #  | Deliverable                                                | Owner                    | Status                                                                    |
| -- | ---------------------------------------------------------- | ------------------------ | ------------------------------------------------------------------------- |
| 1  | Trained extraction model ID                                | Model builder            | `interfor-pod-extraction-v001`                                            |
| 2  | `fields.json` (locked schema)                              | Model builder            | Stored in blob                                                            |
| 3  | Labelled training set (all `.labels.json` and `.ocr.json`) | Model builder / BA       | Stored in blob                                                            |
| 4  | Labelling guide document                                   | BA + model builder       | Conventions for each field                                                |
| 5  | Test result set (all test documents with pass/fail)        | QA / model builder       | Covers all 12 test categories                                             |
| 6  | Table confidence report                                    | Model builder            | Table, row, cell scores per document                                      |
| 7  | Signature detection report                                 | Model builder            | Accuracy across signature variations                                      |
| 8  | Confidence threshold matrix                                | BA + solution specialist | Per-field thresholds                                                      |
| 9  | Accuracy evaluation report                                 | Solution specialist      | Since neural models don't provide accuracy at training, manual evaluation |
| 10 | JSON response samples                                      | Developer                | Actual API responses from Studio                                          |
| 11 | Sample SDK code                                            | Developer                | From Studio right-panel code tab                                          |
| 12 | Known limitation register                                  | Model builder / BA       | Fields/documents that don't extract reliably                              |
| 13 | Training time and cost log                                 | Model builder            | `trainingHours`, free hours remaining                                     |
| 14 | Model version register                                     | Solution specialist      | Model ID, dataset version, test results                                   |
| 15 | Composed model evaluation (if needed)                      | Solution specialist      | Decision: single vs composed                                              |
| 16 | Model copy plan                                            | Developer / platform     | DEV → TEST → PROD → DR                                                    |
| 17 | Troubleshooting runbook                                    | Developer / operations   | Common issues and resolutions                                             |

***

# Accuracy checks — continuation

| Claim                                                                             | Source                                                                                                                                                                               | Status                                       |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------- |
| Table confidence available at table, row, and cell level in v4.0                  | [Accuracy and confidence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/accuracy-confidence?view=doc-intel-4.0.0)                                | ✅ Confirmed citeturn8search165               |
| Top-down approach: table → row → cell                                             | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| Merged cells have lower confidence; missing cells have NULL with lower confidence | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| Cross-page rows: cell confidence high, row confidence potentially lower           | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| Optional/empty cells: high confidence NULL if training reflects optionality       | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| Custom neural models do NOT provide accuracy scores during training               | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| Field-level confidence includes word confidence in v4.0                           | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| High accuracy + low confidence = document differs from training data              | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search165               |
| v4.0 composed models require explicit classifier, not implicit                    | [Composed models](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/composed-models?view=doc-intel-4.0.0)                                              | ✅ Confirmed citeturn8search176               |
| Composed models support up to 500 assigned models                                 | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search176               |
| Composed model billing: classification on all pages, extraction on routed pages   | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search176               |
| Compose REST API requires `classifierId`, `docTypes`, `modelId`                   | [Compose Model REST API](https://learn.microsoft.com/en-us/rest/api/aiservices/document-models/compose-model?view=rest-aiservices-v4.0 "2024-11-30")                                 | ✅ Confirmed citeturn8search184               |
| Copy model: two-step process (authorise on target, copy from source)              | <https://learn.microsoft.com/en-us/rest/api/aiservices/document-models/authorize-model-copy> + <https://learn.microsoft.com/en-us/rest/api/aiservices/document-models/copy-model-to> | ✅ Confirmed citeturn8search187turn8search171 |
| Region labels in neural models use Layout API results; no synthetic text          | [Custom neural model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/train/custom-neural?view=doc-intel-4.0.0)                                            | ✅ Confirmed citeturn8search180               |
| Overlapping fields: max 2 fields per token; cannot span table rows                | Same source                                                                                                                                                                          | ✅ Confirmed citeturn8search180               |

***

This completes the full Phase 3 deep dive. We have now covered:

| Section | Topic                                                                                                   |
| ------- | ------------------------------------------------------------------------------------------------------- |
| 1–3     | Dataset structure, training data balance, project creation                                              |
| 4–5     | OCR/Layout, field definition with all field types                                                       |
| 6       | Labelling techniques (word select, shift-select, region, auto-label tables, search, overlapping)        |
| 7       | Training (build mode, `maxTrainingHours`, billing, monitoring)                                          |
| 8       | Testing (Studio test, evaluation matrix, iterative improvement)                                         |
| 9       | Advanced considerations (region labels, contiguous values, versioning, model copy)                      |
| 10      | Deliverables                                                                                            |
| 11      | **Table confidence deep dive** (three levels, merged cells, cross-page, optional cells, decision rules) |
| 12      | **Accuracy vs confidence** (definitions, matrix, word confidence, composite evaluation, thresholds)     |
| 13      | **Composed models** (v4.0 changes, REST API, billing, decision tree)                                    |
| 14      | **Model copy operations** (authorise + copy, environment promotion, DR)                                 |
| 15      | **Troubleshooting** (training, labelling, inference, confidence calibration)                            |
| 16      | **Model lifecycle** (expiration, version registry, retirement)                                          |
| 17      | **Complete deliverable checklist**                                                                      |


