# Remittance Advice export via E-Documents — Implementation Plan

## Context

Business Central prints remittance advice via two BaseApp reports: **399 "Remittance Advice - Journal"** (unposted payment journal lines) and **400 "Remittance Advice - Entries"** (posted payment Vendor Ledger Entries), both in `src/Layers/W1/BaseApp/Purchases/Reports/`. This feature adds an electronic counterpart: a new **"Create E-Documents"** request-page checkbox on both reports creates E-Document records of a new type **"Remittance Advice"** that flow through the standard E-Documents engine (service → format interface → workflow → service integration) and export as **UBL 2.1 `RemittanceAdvice`** XML via the PEPPOL BIS 3.0 format implementation.

### Decisions

- Trigger = report request-page checkbox via `reportextension` in E-Documents Core (no E-Doc code in BaseApp; assumes UKSendRemittanceAdvice actions have moved to BaseApp — that app is not touched, and no new report-running page actions are added).
- Both reports get the capability; journal-sourced E-Documents are **re-pointed to the posted payment VLE** at posting (journal lines are deleted at posting).
- New flag on **Gen. Journal Line only** — soft-flag pattern (like "Exported to Payment File", not "Check Printed"): non-editable, confirm-on-rerun, **Void action** clears it. Report 400 flow detects duplicates by E-Document lookup on Record ID (no VLE field).
- **Full service flow** (not just blob creation): E-Doc. Service Supported Types, workflow, Service Integration — same as posted sales invoices.
- Extensibility via the **existing interface pattern**: enum 6121 "E-Document Type" + enum 6101 "E-Document Format" implementing interface `"E-Document"`; future formats handle the new type in their own implementation codeunit.
- All code in **E-Documents Core** (`src/Apps/W1/EDocument/App`); no new dependencies (Core already depends on PEPPOL).

### Verified architecture facts

- Outbound entry point: `EDocExport.CreateEDocument(RecRef, DocumentSendingProfile, EDocumentType, AllowReExport)` (codeunit 6102, `Processing/EDocExport.Codeunit.al:65`); `AllowReExport = true` already handles "E-Document exists for Record ID → re-export" (precedent: purchase-order release, `EDocumentSubscribers.Codeunit.al:198`).
- Vendor profile resolution: `EDocumentProcessing.GetDocSendingProfileForCustVend('', VendorNo)` reads `Vendor."Document Sending Profile"` with default-profile fallback (`EDocumentProcessing.Codeunit.al:217-240, 522-539`).
- Re-point precedent exists: E-Document field 30 `"Journal Line System ID"` (Guid) + subscriber `OnAfterPostGLAcc` (`EDocumentSubscribers.Codeunit.al:441`). Vendor-payment equivalent event: `OnAfterVendLedgEntryInsert(var VendorLedgerEntry, GenJournalLine, DtldLedgEntryInserted, PreviewMode)` — `GenJnlPostLine.Codeunit.al:8800`, raised at line 1474.
- Report-extension precedent: `src/Apps/IS/ISCore/app/src/ReportExtensions/ISPurchaseInvoice.ReportExt.al` (dataset `modify(...)` triggers + `requestpage { layout { addlast(Options) } }`).
- Enum 6121 values end at 21 → value **22** free. `"E-Document Service Status"` has `Canceled` (value 4).
- Core-owned "PEPPOL-namespace but not BIS" export precedents: xmlport 6100 `Format/FinResultsPEPPOLBIS30.XmlPort.al`, DOM-based codeunit 6130 `EDocShipmentExportToXml.Codeunit.al`.

## Design overview

- **Granularity:** one E-Document per payment = per (Vendor No., Document No.) group of journal lines (399) / per payment VLE (400).
- **Anchor:** `E-Document."Document Record ID"` → first vendor journal line of the group (table 81) or payment VLE (table 25); `"Journal Line System ID"` (field 30) stores the anchor line's SystemId for re-pointing.
- **Shared data assembly:** one Core codeunit computes header + applied-document lines (paid docs, amounts, discounts, currency) from either source into a temp buffer table, so every format consumes identical data.
- **Format:** new case in codeunit 6165 "EDoc PEPPOL BIS 3.0" delegating to a new **DOM-based** Core codeunit producing plain UBL 2.1 (no PEPPOL BIS profile exists for remittance advice → belongs in Core, not the PEPPOL app; same reasoning as Fin. Results / Shipment; CustomizationID/ProfileID omitted, with an integration event to inject them).

## Object inventory

New files under `src/Apps/W1/EDocument/App/src/RemittanceAdvice/` unless marked *modify*. IDs from Core's free ranges (verify at implementation time; per-type ID space).

| Object | ID | Name | Purpose |
|---|---|---|---|
| enum value *(modify enum 6121)* | 22 | `"Remittance Advice"` | `Document/EDocumentType.Enum.al` |
| table | 6117 | `"E-Doc. Remit. Advice Buffer"` | Temp buffer: header row (Line No. 0) + one row per applied doc |
| codeunit | 6110 | `"E-Doc. Remittance Advice Mgt."` | Data assembly from journal group or payment VLE; Check validation; E-Doc lookup |
| codeunit | 6111 | `"E-Doc. Remit. Advice Export"` | Orchestration: collection state for report runs, confirms, profile resolution, flagging, `EDocExport.CreateEDocument` |
| codeunit | 6112 | `"E-Doc. Remit. Advice To XML"` | UBL 2.1 RemittanceAdvice DOM generation from buffer |
| reportextension | 6100 | `"E-Doc. Remit. Advice Journal"` | extends report 399: checkbox + collect + OnPostReport create |
| reportextension | 6101 | `"E-Doc. Remit. Advice Entries"` | extends report 400: same for VLEs |
| tableextension | 6101 | `"E-Doc. Gen. Journal Line"` | field 6100 `"Remit. Advice E-Doc. Created"` (Boolean, Editable=false) + helpers |
| pageextension | 6100 | `"E-Doc. Payment Journal"` | page 256: show flag; Void action; open-E-Document action |
| codeunit *(modify)* | 6102 | `"E-Doc. Export"` | `PopulateEDocument`: cases for tables 81/25 guarded by Document Type = Remittance Advice |
| codeunit *(modify)* | 6108 | `"E-Document Processing"` | profile-resolution cases for tables 81/25; `GetLines` case; `GetTypeFromSourceDocument` case for table 25 |
| codeunit *(modify)* | 6103 | `"E-Document Subscribers"` | `OnAfterVendLedgEntryInsert` re-point; Gen. Journal Line delete-safety subscriber |
| codeunit *(modify)* | 6165 | `"EDoc PEPPOL BIS 3.0"` | `Check` + `Create` cases for the new type |
| codeunit *(modify)* | — | `"E-Doc. PEPPOL Validation"` | `CheckRemittanceAdvice` (journal + VLE variants) |
| permissionsets *(modify)* | — | `EDocCoreObjects`/`Read`/`Edit` | add new objects |
| test codeunit | next free | `"E-Doc. Remit. Advice Test"` | `src/Apps/W1/EDocument/Test/src/Processing/` |

## Detailed design

### 1. Data assembly (table 6117 + codeunit 6110)

Buffer table (`TableType = Temporary`): line rows with `"Line No."` (PK), `"Applied Doc. Type"`, `"Our Document No."`, `"External Document No."`, `"Document Date"`, `"Posting Date"`, `"Currency Code"` (payment currency), `"Original Amount"`, `"Remaining Amount"`, `"Paid Amount"`, `"Pmt. Discount Amount"`, `"Vendor Ledger Entry No."`; header row (`Line No. = 0`) with `"Payment Document No."`, `"Vendor No."`, `"Payment Date"`, `"Currency Code"`, `"Total Paid Amount"`, `"Total Discount"`, payment-means data (bank payment type, recipient bank account when resolvable). Amounts stored positive (vendor perspective) — report columns are negated at assembly.

Codeunit 6110 API:

- `BuildFromJournalPayment(AnchorGenJnlLine, var TempBuffer)` — iterate group lines (same Template/Batch/Document No./Account No., Account Type = Vendor); resolve applied VLEs via `"Applies-to ID"` or `"Applies-to Doc. No./Type"` + credit-memo applications via Detailed Vendor Ledg. Entry; port report 399's currency conversion (`ExchangeAmtFCYToFCY` to payment currency, payment-currency rounding), payment-discount qualification (`Posting Date <= Pmt. Discount Date` and |remaining| ≥ |remaining disc. possible|), and paid-amount allocation math (report 399 lines 284-386, 537-560).
- `BuildFromPostedPayment(PaymentVendLedgEntry, var TempBuffer)` — applied entries via `"Closed by Entry No."` both directions + `FindApplnEntriesDtldtLedgEntry` walk (report 400 lines 403-441); per-line amounts from Detailed Vendor Ledg. Entry sums (Entry Type = Application, Document Type = Payment, Document No. = payment doc no., Unapplied = false) and discounts from `"Pmt. Disc. Rcd.(LCY)"` (report 400 lines 231-276).
- `CheckJournalPayment` / `CheckPostedPayment` — TestFields (Account Type = Vendor, Document Type = Payment, vendor exists), error when zero applied documents (UBL requires ≥1 `RemittanceAdviceLine`), company info present.
- `FindEDocument(var EDocument; RecRef): Boolean` — lookup by `"Document Record ID"`.

Format-interface RecordRef mapping: `SourceDocumentHeader` = anchor record (set by engine from Record ID); `SourceDocumentLines` = same single record via a new `"Remittance Advice"` case in `EDocumentProcessing.GetLines` (open Record ID's table, SetRange PK). Format implementations rebuild the buffer from the header — same approach as shipment/purchase-order.

### 2. Report extensions (6100 / 6101)

Skeleton (per `ISPurchaseInvoice.ReportExt.al` precedent):

- requestpage: `addlast(...)` group "Electronic Document" with `field(CreateEDocuments; ...)` — **verify anchor first**: reports 399/400 have *empty* request-page layouts; if `addlast(Content)` doesn't compile, fallback = add an empty neutral `area(content) { group(Options) }` scaffold to the two BaseApp reports (no E-Doc logic in BaseApp, acceptable).
- Report 399: `modify("Gen. Journal Line")` (the nested vendor-filtered dataitem) `trigger OnAfterAfterGetRecord()` → `EDocRemitAdviceExport.CollectJournalLine(...)` when checkbox on. `trigger OnPostReport()` → group collected lines by (Account No., Document No.), anchor = lowest Line No., one E-Document per group.
- Report 400: `modify("Vendor Ledger Entry")` collecting `"Entry No."`; OnPostReport creates one per entry.
- Checkbox default false → zero behavior change. Preview-vs-print indistinguishable — accepted (explicit opt-in); duplication is impossible anyway: one E-Document per Record ID, re-export requires confirm.
- Confirms (GuiAllowed-guarded, default No): journal flow — any group line flagged → "already created … create again?" → Yes = proceed with `AllowReExport := true`, No = skip group; entries flow — existing E-Document for VLE Record ID → same confirm. Non-GUI runs skip silently.

### 3. Export engine wiring

Per payment, codeunit 6111:

1. `EDocumentProcessing.GetDocSendingProfileForDocRef(AnchorRecRef)` — extended with cases for table 81 (vendor = "Account No." when "Account Type" = Vendor) and table 25 (vendor = "Vendor No."); skip silently if profile isn't `"Extended E-Document Service Flow"` (same as posted-doc subscribers).
2. `EDocExport.CreateEDocument(AnchorRecRef, Profile, Enum::"E-Document Type"::"Remittance Advice", AllowReExport)` — creates record, exports per service via format interface, logs, starts workflow. Sending works unchanged.
3. On success (journal flow): set `"Remit. Advice E-Doc. Created" := true` on all group lines (plain `Modify()`).

Core modifications:

- `EDocExport.PopulateEDocument` (case ~line 319): `Database::"Gen. Journal Line", Database::"Vendor Ledger Entry":` guarded by `EDocument."Document Type" = "Remittance Advice"` (protects existing General-Journal-type flows) → new `PopulateRemittanceAdviceEDocument`: Document No. (payment), Bill-to/Pay-to = vendor, dates, Currency Code, `"Source Type" := Vendor`, amounts = total paid (Abs), and for table 81 `"Journal Line System ID" := SystemId`.
- `EDocumentProcessing.GetTypeFromSourceDocument`: add `Database::"Vendor Ledger Entry" → "Remittance Advice"`. **Do not** change the table-81 mapping (owned by existing "General Journal" flow).
- Supported types: no engine change (`EDocServiceSupportedType.Get` fall-through works). **No auto-seeding** in `OnAfterValidateDocumentFormat` — users add "Remittance Advice" on the E-Doc. Service Supported Types page (enum-generic, no page change).

### 4. Posting re-point + lifecycle (EDocumentSubscribers)

- Subscriber on `Gen. Jnl.-Post Line`::`OnAfterVendLedgEntryInsert`: exit on `PreviewMode`/null SystemId; find E-Document where `"Journal Line System ID" = GenJournalLine.SystemId` and Document Type = Remittance Advice; `Validate("Document Record ID", VendorLedgerEntry.RecordId)`, update Document No./Posting Date, `Modify()`. Runs inside posting transaction → rollback-safe.
- Delete-safety subscriber on table 81 `OnAfterDeleteEvent` (guard `IsTemporary`): if an un-re-pointed Remittance Advice E-Document references the line (`"Journal Line System ID" = SystemId` and Record ID still table 81 — posting deletes lines *after* VLE insert, so re-pointed docs are excluded), set its service statuses `Canceled` + document status accordingly. Keep the record and Record ID (audit); orphans from trigger-less bulk deletes are harmless (Void/E-Document page can cancel).
- Multi-line payments (no "Summarize per Vendor"): e-doc re-points to the anchor line's VLE only — accepted (risk R4).

### 5. PEPPOL format implementation (codeunits 6165 + 6112)

- `EDocPEPPOLBIS30.Check` (~line 35): cases for tables 81/25 → `EDocPEPPOLValidation.CheckRemittanceAdvice` (company Name/Country/VAT-or-GLN endpoint, vendor Name/Country; structural checks delegated to codeunit 6110).
- `EDocPEPPOLBIS30.Create` (case ~line 87): `"Remittance Advice": GenerateRemittanceAdviceXMLFile(SourceDocumentHeader, DocOutStream)` — build buffer via 6110 (branch on table 81/25), run codeunit 6112, stream XmlDocument to OutStream (CopyStream pattern of `GenerateShipmentXMLFile`).
- Codeunit 6112 — root `RemittanceAdvice`, ns `urn:oasis:names:specification:ubl:schema:xsd:RemittanceAdvice-2` + cac/cbc (as xmlport 6100). Element order per UBL 2.1 schema.

BC → UBL mapping:

| UBL element | Source |
|---|---|
| `cbc:UBLVersionID` | `2.1` |
| `cbc:ID` | Payment Document No. |
| `cbc:IssueDate` | Payment Date (`Format(...,0,9)`) |
| `cbc:DocumentCurrencyCode` | Currency Code or GLSetup LCY (reports' `CurrencyCode()` semantics) |
| `cbc:TotalPaymentAmount` @currencyID | Total Paid Amount |
| `cbc:PaymentOrderReference` | Payment Document No. |
| `cbc:LineCountNumeric` | line count |
| `cac:AccountingCustomerParty` (payer = **our company**) | Company Info: EndpointID (VAT Reg. No. / GLN schemeID 0088, reuse xmlport 6100 supplier-party logic — roles swapped vs invoices), PartyName, PostalAddress, PartyTaxScheme (VAT), PartyLegalEntity |
| `cac:AccountingSupplierParty` (payee = **vendor**) | Vendor: EndpointID, Name, PostalAddress, PartyLegalEntity |
| `cac:PaymentMeans` (0..1) | PaymentMeansCode 31 (credit transfer) / 20 (check) from Bank Payment Type; PaymentID = payment Document No.; PayeeFinancialAccount from Recipient Bank Account when resolvable (omit for VLE flow if not) |
| `cac:RemittanceAdviceLine` (1..*) | per buffer line: `cbc:ID` (seq.), `cbc:DebitLineAmount` (paid, invoices) / `cbc:CreditLineAmount` (credit memos), `cbc:BalanceAmount` (remaining), `cbc:InvoicingPartyReference` (vendor's External Doc. No.), `cac:BillingReference/cac:InvoiceDocumentReference` (or `CreditNoteDocumentReference`) with `cbc:ID` + `cbc:IssueDate`; discount folded into amounts + line `cbc:Note` 'Payment discount: X' (no dedicated UBL element) |

- CustomizationID/ProfileID omitted; integration event `OnBeforeAddHeaderElements` for injection. Amounts `Format(x, 0, 9)`, 2 decimals, currencyID on every amount.

### 6. Flag + Void (tableextension 6101, pageextension 6100)

- Field 6100 `"Remit. Advice E-Doc. Created"`: Editable=false, DataClassification=CustomerContent, tooltip. Helpers `SetRemitAdviceEDocCreated`, `HasRemitAdviceEDoc`. No TestField blocking anywhere (soft flag).
- Page 256 extension: field `addafter("Exported to Payment File")`; actions:
  - **Void Remittance Advice E-Doc.**: selection filter → confirm → per flagged group: clear flag on all group lines; find E-Document; if service status Sent/Approved/Pending Response → error directing to E-Document page Cancel action (`EDocument.Page.al:287`, `ISentDocumentActions` — do not duplicate); else set service statuses `Canceled` + document status via `EDocumentProcessing.ModifyServiceStatus`/`ModifyEDocumentStatus`. Keep the E-Document record; re-run of report re-exports it via `AllowReExport` (same as page `Recreate` action).
  - **Open Remittance Advice E-Document**: navigate to the linked E-Document.
- Entries flow: no flag; "void" = Cancel/Recreate on E-Document page (document only).

### 7. Service setup (docs/tests, no code)

E-Document Service (format PEPPOL BIS 3.0 + integration) → add "Remittance Advice" in E-Doc. Service Supported Types → standard workflow (E-Document Created → Send to service) → Document Sending Profile with "Extended E-Document Service Flow" on the vendor (or default). Batch services work unchanged (PEPPOL `CreateBatch` is a no-op today — same limitation as other types).

## Implementation order (parallelizable)

- **Stream A (foundation):** enum value 22 → table 6117 → codeunit 6110 (data assembly, hardest logic).
- **Stream B (engine):** EDocumentProcessing + EDocExport modifications → codeunit 6111 → report extensions → subscribers.
- **Stream C (format):** EDocPEPPOLBIS30 + validation + codeunit 6112 (needs table 6117 shape from A).
- **Stream D (UX):** tableextension + pageextension + permission sets.

A first; B/C/D can run in parallel after A's buffer/API shape is fixed. Tests last.

## Test plan

Test app `src/Apps/W1/EDocument/Test` (use `LibraryEDocument`, `EDocFormatMock` + `EDocImplState` patterns, `EDocE2ETest` style). New codeunit `EDocRemitAdviceTest`:

1. Journal happy path: applied payment line → report 399 with checkbox (RequestPageXML) → E-Document created (type/direction/vendor/amounts/Journal Line System ID), status Exported, flag set; XML XPath assertions (root/ns, ID, TotalPaymentAmount, line DebitLineAmount + InvoiceDocumentReference).
2. Checkbox off → nothing.
3. Multi-line group / multi-vendor batch → one e-doc per (vendor, doc no.); flags on all group lines.
4. Applies-to ID incl. credit memo → credit line + discount math parity with report values.
5. Entries happy path (posted payment → report 400).
6. Duplicate/confirm: rerun → ConfirmHandler No = skip; Yes = same record re-exported.
7. Re-point at posting: Record ID becomes payment VLE; No./date updated.
8. Preview posting → no re-point.
9. Void: flag cleared, statuses Canceled; Void after Sent → error; re-run after void re-exports.
10. Journal line deleted unposted → statuses Canceled.
11. Format extensibility: mock format service + supported type → mock `Create` invoked with table 81/25 header RecRef.
12. Type not in supported types → no e-document, no error.
13. Check failure (missing company VAT/GLN) → Export Error status.

## Verification

- Build both apps (E-Documents Core + Test) with the repo's AL compiler workflow; zero new warnings.
- Run `EDocRemitAdviceTest` + existing `EDocE2ETest` suite (regression: existing types unaffected, esp. General Journal/G/L Entry flows sharing field 30 and table-81 sources).
- Manual smoke (BC container): set up service/workflow/profile, run report 399 from Payment Journal with checkbox → verify E-Document page shows the doc, XML blob content, flag on lines; post → Record ID re-pointed; Void → statuses Canceled and flag cleared; report 400 on a posted payment → e-doc created; rerun → confirm prompt.

## Risks / open questions

- **R1** Request-page anchor on empty base layout — verify `addlast(Content)` compiles in a reportextension; fallback: neutral empty `group(Options)` scaffold added to the two BaseApp reports.
- **R2** No PEPPOL BIS profile for remittance advice — access-point receivers may reject; CustomizationID/ProfileID empty + injection event.
- **R3** Report-399 allocation math fidelity (partial payments, cross-currency, credit memos) — mitigate with parity tests (report = source of truth).
- **R4** Multi-VLE payments: e-doc re-points to anchor line's VLE only.
- **R5** Trigger-less bulk journal deletion can orphan an e-doc (dangling Record ID) — harmless, cancelable.
- **R6** `GetTypeFromSourceDocument` now maps table 25 → Remittance Advice — audit callers (none pass VLEs today).
- **Open**: PaymentMeans for VLE flow (limited bank info — may omit); Embed-PDF-in-export flag ignored for this type in v1.
