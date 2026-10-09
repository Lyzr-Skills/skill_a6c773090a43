---
name: bucket-execution-signatures
description: Fact bucket for WHEN and HOW a document became binding - every signature block (signed or empty) with signer, party, method, date, time and time zone as printed; e-signature certificate and email events with timestamps; the document's effective date and how it is defined; offer-expiry and execution-condition clauses. The model transcribes; code converts time zones and computes the last signature date. Feeds the Order Effective Date check, late / early renewal, signing after start, SSA execution and marketplace acceptance date.
metadata:
  bucket_id: execution_signatures
  schema_version: 2.0.0
  output_model: execution_signatures_models.py (ExecutionSignaturesOutput)
  brd_refs: Sec 3.1, 4.1.1, 6.1.1, 6.5, 7.2 step 3
  tracker_rows: 10, 25 (acceptance email date), 28, 41, 42, 60
---

# Bucket: Execution & Signatures

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (Execution & Signatures) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `execution_signatures_models.py`
(`ExecutionSignaturesOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`execution_signatures.values.<field>`, `execution_signatures.records.<collection>[<key>].<leaf>`,
`execution_signatures.evidence.clauses[<clause_id>]`, `execution_signatures.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`execution_signatures_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - Execution & Signatures

**Core rule: transcribe timestamps exactly, compute nothing.** Code converts every timestamp to
US Eastern Time, takes the latest signature, and compares it with Salesforce. A wrong conversion by
the model would silently break the zero-tolerance date rule (BRD 6.1.1), so the model never converts.

## 1. Applicability
**Applies** to any document with signature blocks, an e-signature certificate, an effective-date
definition, or timestamped execution events: OS, SSA / customer paper, SOW, change order,
amendment, partner agreement, PO (if signed or dated as issued), marketplace private offer, and
Deal Desk / marketplace acceptance emails.

**Does not apply** to documents with none of these, e.g. an SLA exhibit or DPA incorporated by reference.

## 2. Ownership boundaries
| Fact | Owner bucket | Note |
|---|---|---|
| Party legal names and addresses | Parties & Geography | `party_name` here is only the name printed on the signature block |
| Draft / version labels and watermarks | Document Identity & References | combine with `signature_status` to decide "is this the executed contract" |
| Subscription term start and end dates | Term & Termination | needed for late / early renewal and signing-after-start checks |
| Order Schedule number | Document Identity & References | the envelope ID is here |

## 3. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `signature_status` | enum `fully_signed`, `partially_signed`, `unsigned`, `unknown` | From the signature blocks in this file. `unknown` if blocks are illegible or cut off. Never decide superseded / expired here. |
| `effective_date` | Date | Only when the effective date is printed as a date. If the document defines it by a rule ("effective as of the date of the last signature below"), status `not_found` and set `effective_date_basis`. Several printed effective dates -> `candidates` + `conflict` note. |
| `effective_date_basis` | enum `explicit_date`, `latest_signature`, `earliest_signature`, `subscription_start`, `other` | From the definition wording; capture the wording as an `effective_date_definition` clause. Silent document -> `not_found`. |
| `certificate_present` | Boolean | A DocuSign Certificate of Completion, Adobe Sign audit report or similar page is in this file. Its absence is `found` / false when the document is complete and has no such page (quote the last signature block). |
| `envelope_id` | Text | E-signature envelope / agreement ID exactly as printed, from the header stamp or the certificate. |

## 4. records (calculation rules)

### signatures: one item per signature block, signed or not
| Leaf | Type | Notes |
|---|---|---|
| `party_role` | enum `anaplan`, `customer`, `partner`, `reseller`, `witness`, `other` | |
| `party_name` | Text | Organisation printed on the block. |
| `signer_name`, `signer_title` | Text | As printed or typed. |
| `signature_present` | Boolean | true if signed (image, `/s/`, DocuSign stamp, wet ink); false for an empty block. |
| `signature_method` | enum `docusign`, `adobe_sign`, `other_esignature`, `wet_ink`, `typed_name`, `unknown` | `docusign` only when DocuSign is named (stamp, envelope header, certificate). |
| `date` | Date | raw = the **whole** date-time string as printed on the block (e.g. `10-Apr-26 \| 5:35:23 PM EDT`); parsed = ISO date of the **printed local date**, not converted. `10-Apr-26` -> `2026-04-10`. Ambiguous day / month -> null + `normalization` note. |
| `time` | Text | raw = time as printed; parsed = 24-hour `HH:MM:SS` (seconds `00` if not printed). Null if no time is printed. |
| `timezone` | Text | raw = zone as printed (`EDT`, `CEST`, `UTC-05:00`, `(GMT+01:00) Paris`); parsed = UTC offset `+HH:MM` / `-HH:MM` only when printed as an offset or the abbreviation is one of: UTC/GMT `+00:00`, EST `-05:00`, EDT `-04:00`, CST `-06:00`, CDT `-05:00`, MST `-07:00`, MDT `-06:00`, PST `-08:00`, PDT `-07:00`, BST `+01:00`, CET `+01:00`, CEST `+02:00`, JST `+09:00`, SGT `+08:00`, AEST `+10:00`, AEDT `+11:00`. Anything else (e.g. `IST`, which is ambiguous) -> null. |

Key: `<party_role>:<n>` (`anaplan:1`, `customer:1`, `customer:2`).
A handwritten date -> transcribe it and add a `handwritten_date` signal. A block date that differs
from the certificate date for the same signer -> keep both (block in `signatures`, certificate in
`execution_events`) and add a signal; never pick one.

### execution_events: one item per timestamped event
| Leaf | Type | Notes |
|---|---|---|
| `event_type` | enum `envelope_sent`, `delivered`, `viewed`, `signed`, `completed`, `declined`, `voided`, `corrected`, `email_sent`, `marketplace_offer_accepted`, `other` | |
| `actor_name`, `actor_email` | Text | Person or system named on the event. |
| `actor_party_role` | enum (signature roles + `system`) | |
| `timestamp` | Text | raw = as printed; parsed = local `YYYY-MM-DDTHH:MM:SS` with **no conversion and no offset**. |
| `timezone` | Text | Same rules as signatures. Certificates often print one zone for all events: copy it onto every event. |
| `source_kind` | enum `esignature_certificate`, `email_header`, `document_text`, `other` | |

Key: `E1…` in printed order. For Deal Desk / marketplace emails, record the email's sent time as
`email_sent` (from the header) and any stated acceptance as `marketplace_offer_accepted`.

## 5. evidence (contextual agent)
**Clause categories** (★ = full-verbatim):
`effective_date_definition` ★, `counterparts_esignature`, `execution_conditions` ★ (effective only
on PO receipt, payment, board or government approval: conditions precedent), `offer_expiry` ★
("must be executed on or before…", "pricing valid until…"), `acceptance_by_conduct` ★ (PO issuance,
marketplace acceptance or use constitutes acceptance), `other_execution`.

**Signal types**:
| Type | Record when |
|---|---|
| `unsigned_block` | A signature block is empty |
| `certificate_completed` | Certificate status shows Completed |
| `certificate_missing` | Blocks look e-signed (stamp, envelope ID) but no certificate is in the file |
| `date_differs_by_timezone` | The same signing shows different calendar dates in different places / zones |
| `signature_date_conflict` | Block date and certificate date for the same signer disagree beyond a time-zone difference |
| `timezone_missing` | A signature date or time has no time zone |
| `handwritten_date` | A date is handwritten |
| `backdating_language` | "effective as of" a date earlier than the signatures |
| `offer_expiry_present` | The document sets a deadline for signature or pricing validity |
| `condition_precedent` | Effectiveness depends on an event other than signature |
| `modified_after_signature` | Certificate shows corrected / voided / re-sent events, or text was changed after a signature |
| `other` | Anything else about execution a reviewer should see |

Signal descriptions state what is printed, e.g. "Customer block dated 11-Apr-26 CEST; certificate
shows 4/10/2026 PDT", not "customer signed late".

**Hints** (each exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `fully_executed` | All required parties have signed this document | #28 SSA executed, draft detection |
| `execution_conditional` | The contract's effectiveness depends on a condition beyond signature (ASC 606 step 1, contract existence) | context review |

## 6. Code-side derivations (never done by the model)
1. Each signature -> UTC using `date.parsed` + `time.parsed` + `timezone.parsed`; then to US Eastern
   Time. Missing time or zone -> date-only comparison, flagged "Accountant to Confirm".
2. Source priority for each signer: certificate `signed` event, then the block timestamp. If both exist and disagree beyond the time-zone difference -> exception.
3. `last_signature_et` = latest signed timestamp in ET across required parties; `first_signature_et` = earliest.
4. `resolved_effective_date`: `explicit_date` -> `effective_date`; `latest_signature` -> date of `last_signature_et`; `earliest_signature` -> `first_signature_et`; `subscription_start` -> Term & Termination start.
5. `executed` = every Anaplan and customer block has `signature_present` = true.

## 7. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| #10 Order Effective Date (Rule 6.1.1) | calc | \|SF `Contract_Date__c` - date(`last_signature_et`)\| >= 1 day -> "Date Mismatch Exception"; output last signature (ET), Anaplan and customer sign dates | `records.signatures`, `records.execution_events[event_type=signed]` |
| #10 Effective date | calc | `resolved_effective_date` vs SF `Contract_Date__c` | `values.effective_date`, `values.effective_date_basis` |
| #41 Late Renewal | calc | Type = Renewal AND date(`last_signature_et`) > Term start | derived + Term & Termination |
| #42 Early Renewal | calc | Type = Renewal AND month(`last_signature_et`) < month(Term start) | derived + Term & Termination |
| #60 Signing > Start | calc | date(`last_signature_et`) > Term start; New / Add-on -> recommend revenue start = last signature (BRD 6.5) | derived + Term & Termination |
| BRD 6.5 Revenue start | calc | New / Add-on: later of last signature and OED; Renewal / Supersede: scheduled OED | derived |
| BRD 7.2 s3 SOW dates | calc | SOW signatures vs SOW order effective date | same paths on the SOW |
| #28 SSA executed | fact | SSA has Anaplan and customer blocks signed | `records.signatures`, hint `fully_executed` |
| Draft detection | context | Unsigned or partially signed + Identity `draft_document` -> not the executed contract; look for another file (tracker #10 note: 83 / 113 agreement PDFs predate signature) | `values.signature_status`, hint `fully_executed` |
| #25 Marketplace acceptance | fact | Acceptance email found Y / N + sent date (ET) | `records.execution_events[email_sent, marketplace_offer_accepted]` |
| Offer expiry | context | Signature after the stated deadline (after ET conversion) -> flag pricing validity for review | `evidence.clauses[offer_expiry]`, derived `last_signature_et` |
| Conditions precedent | context | Contract may not exist under ASC 606 until the condition is met | `evidence.clauses[execution_conditions]`, hint `execution_conditional` |
| Update check | code | Same `envelope_id` with a later `completed` event or `corrected` events -> re-review | `values.envelope_id`, `records.execution_events` |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", Money as above.
```

## C2. `execution_signatures` output

```jsonc
{
  "bucket": "execution_signatures",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "signature_status": Value<"fully_signed" | "partially_signed" | "unsigned" | "unknown">,
    "effective_date": Value<Date>,
    "effective_date_basis": Value<"explicit_date" | "latest_signature" | "earliest_signature" | "subscription_start" | "other">,
    "certificate_present": Value<Boolean>,
    "envelope_id": Value<Text>
  },

  "records": {
    "signatures": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "party_role": Leaf<"anaplan" | "customer" | "partner" | "reseller" | "witness" | "other">,
      "party_name": Leaf<Text>,
      "signer_name": Leaf<Text>,
      "signer_title": Leaf<Text>,
      "signature_present": Leaf<Boolean>,
      "signature_method": Leaf<"docusign" | "adobe_sign" | "other_esignature" | "wet_ink" | "typed_name" | "unknown">,
      "date": Leaf<Date>,
      "time": Leaf<Time>,
      "timezone": Leaf<UtcOffset>
    }>,
    "execution_events": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "event_type": Leaf<"envelope_sent" | "delivered" | "viewed" | "signed" | "completed" | "declined" | "voided" | "corrected" | "email_sent" | "marketplace_offer_accepted" | "other">,
      "actor_name": Leaf<Text>,
      "actor_party_role": Leaf<"anaplan" | "customer" | "partner" | "reseller" | "witness" | "other" | "system">,
      "actor_email": Leaf<Text>,
      "timestamp": Leaf<Timestamp>,
      "timezone": Leaf<UtcOffset>,
      "source_kind": Leaf<"esignature_certificate" | "email_header" | "document_text" | "other">
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `effective_date_definition` ★, `counterparts_esignature`, `execution_conditions` ★, `offer_expiry` ★, `acceptance_by_conduct` ★, `other_execution`
  (★ = full-verbatim, never truncated)
- `signal_type`: `unsigned_block`, `certificate_completed`, `certificate_missing`, `date_differs_by_timezone`, `signature_date_conflict`, `timezone_missing`, `handwritten_date`, `backdating_language`, `offer_expiry_present`, `condition_precedent`, `modified_after_signature`, `other`
- `hint_name` (each exactly once): `fully_executed`, `execution_conditional`
