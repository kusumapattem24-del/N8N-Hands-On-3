# Customer Signup Cleanser - Lead Data Quality & Deduplication

**Assessment:** Build & Validate a Production-Ready Lead Data Quality Workflow  
**Document date:** 22 October 2026  
**Status:** Implementation blueprint; execution evidence and exported n8n JSON must be attached after verification.

## 1. Objective and scope
Build an n8n workflow that checks required fields, cleans names/emails/phones, removes duplicates based on normalized email, preserves the original email, and produces CRM-ready lead objects. Invalid records must be routed separately without stopping valid records.

## 2. Architecture

```text
Mock_Leads
    |
Validate_Required_Fields (IF: fullName and email nonblank)
    | TRUE                          | FALSE
    v                               v
Clean_Standardize             Rejected_Missing_Fields
    |
Remove_Duplicates (cleanedEmail, current input)
    |
Map_Final_Output
    |
CRM_Ready_Leads
```

**Important:** The Remove Duplicates node drops repeated items; if a separate duplicate audit output is required, create an additional pre-deduplication audit branch. Do not claim a duplicate rejection log exists unless implemented.

## 3. Node-by-node configuration

| Node | Type | Configuration / purpose |
|---|---|---|
| Mock_Leads | Manual Trigger + Code or Edit Fields | Supply sample records with id, fullName, email, phone, signUpDate. |
| Validate_Required_Fields | IF | Require trimmed non-empty fullName and email; TRUE continues, FALSE goes to rejection. |
| Rejected_Missing_Fields | Edit Fields | Preserve source id and reason `MISSING_FULL_NAME_OR_EMAIL`. |
| Clean_Standardize | Edit Fields | Create cleanedName, cleanedEmail, cleanedPhone and originalEmail; keep input fields needed downstream. |
| Remove_Duplicates | Remove Duplicates | Operation: remove items repeated within current input; compare field `cleanedEmail`. |
| Map_Final_Output | Edit Fields | Build id, name, email, phone, status, originalEmail using expressions. |
| CRM_Ready_Leads | No Op / output | Inspect and capture final output. |

## 4. Validation and cleaning expressions

**IF node conditions** (use AND):

```javascript
{{ String($json.fullName ?? '').trim().length > 0 }}
{{ String($json.email ?? '').trim().length > 0 }}
```

Set both comparisons to Boolean `true`. Required-field presence is distinct from email-format validation; an additional email-format IF is recommended for production.

**Edit Fields expressions** (use exact field names in your input):

```javascript
cleanedName: {{ String($json.fullName ?? '').trim().replace(/\s+/g, ' ').toLowerCase().replace(/\b\w/g, c => c.toUpperCase()) }}
cleanedEmail: {{ String($json.email ?? '').trim().toLowerCase() }}
cleanedPhone: {{ String($json.phone ?? '').replace(/\D/g, '') }}
originalEmail: {{ $json.email ?? '' }}
```

**Note:** Expression support varies by n8n version and expression editor. If the editor rejects inline callbacks, use the Code node or supported expression methods and test each expression before continuing. Phone cleaning removes formatting only; it does not validate country codes or length.

**Final mapping** (Map_Final_Output):

```javascript
id: {{ $json.id }}
name: {{ $json.cleanedName }}
email: {{ $json.cleanedEmail }}
phone: {{ $json.cleanedPhone }}
status: {{ 'READY_FOR_CRM' }}
originalEmail: {{ $json.originalEmail }}
```

Preserving `originalEmail` in Clean_Standardize avoids fragile item-index cross-node references. If cross-node referencing is specifically required, use `$('Mock_Leads').item.json.email` only when item linking is preserved and validated; otherwise use the carried-forward field.

## 5. Illustrative sample input and output

**Input** (synthetic test data):

```json
[
  {"id":"L001","fullName":"  SARAH   CONNOR ","email":" Sarah@Example.COM ","phone":"(555) 010-2000"},
  {"id":"L002","fullName":"Sarah Connor","email":"sarah@example.com","phone":"555-010-2000"},
  {"id":"L003","fullName":" Anna  Lee ","email":"ANNA@EXAMPLE.COM","phone":"555 010 3000"},
  {"id":"L004","fullName":"Anna Lee","email":"anna@example.com","phone":"5550103000"},
  {"id":"L005","fullName":"","email":"missing@example.com","phone":"5550104000"},
  {"id":"L006","fullName":"Maya Patel","email":"maya@example.com","phone":"5550105000"}
]
```

**Expected**: L002 and L004 removed as duplicates after normalization; L005 rejected for missing name; L001, L003 and L006 remain. This is a planned expected result, **not a recorded execution result**.

```json
{
  "id":"L001",
  "name":"Sarah Connor",
  "email":"sarah@example.com",
  "phone":"5550102000",
  "status":"READY_FOR_CRM",
  "originalEmail":" Sarah@Example.COM "
}
```

## 6. Test plan (mark actual results after running)

| ID | Scenario | Expected result | Execution |
|---|---|---|---|
| TC-01 | Complete valid lead | Normalized CRM-ready lead | Not yet evidenced |
| TC-02 | Blank fullName | Rejected_Missing_Fields | Not yet evidenced |
| TC-03 | Blank email | Rejected_Missing_Fields | Not yet evidenced |
| TC-04 | Duplicate Sarah Connor, different email casing | One retained | Not yet evidenced |
| TC-05 | Duplicate Anna Lee, surrounding spaces | One retained | Not yet evidenced |
| TC-06 | Phone with punctuation | Digits only | Not yet evidenced |
| TC-07 | Missing optional phone | Empty string, no crash | Not yet evidenced |
| TC-08 | Original email preservation | Raw source email retained | Not yet evidenced |
| TC-09 | Multiple valid + invalid records | Valid path continues independently | Not yet evidenced |

## 7. Naming, annotations and troubleshooting

- Name nodes exactly as documented; use sticky notes for validation, normalization, deduplication and output mapping.
- Annotate owner, input schema, output schema, deduplication scope and exception behavior.
- If duplicates remain, confirm normalization occurs **before** Remove Duplicates and compare field is `cleanedEmail`.
- If the final object has missing fields, check `Keep Other Input Fields` and field naming/casing in Edit Fields.
- If cross-node expressions fail, inspect item linking and prefer carrying required fields forward.
- For production, add stricter email/phone validation, secure CRM credentials, PII retention controls and observability.

## 8. Evidence and submission

Capture screenshots of: complete workflow canvas; IF true and false branches; cleaned values; deduplication results for Sarah Connor and Anna Lee; final CRM-ready JSON; and successful execution. Export the latest n8n workflow JSON from the n8n editor (this README is **not** the workflow export).

Suggested repository structure:

```text
customer-signup-cleanser/
  README.md
  workflow/Customer_Signup_Cleanser.json
  docs/Customer_Signup_Cleanser_Assessment.pdf
  evidence/ (screenshots)
```

Record a Loom/YouTube demonstration showing each branch and the final output. Use a **new public GitHub repository** if your LMS rejects a previously submitted repository URL. Remove personal data and secrets before publishing.

## 9. Assessment checklist

- [ ] Required-field IF and rejection route tested
- [ ] Name, email and phone normalization tested
- [ ] cleanedEmail deduplication verified
- [ ] Cross-node / preserved original email verified
- [ ] CRM-ready output verified
- [ ] Annotations and naming reviewed
- [ ] n8n workflow JSON exported
- [ ] Screenshots and video uploaded
- [ ] New public GitHub repository verified
- [ ] LMS submission completed

## 10. Assumptions and limitations

- Sample leads are synthetic; actual workflow execution has not been independently inspected for this document.
- Deduplication scope is the current execution input, not historical CRM records.
- First occurrence is retained; upstream input ordering determines which duplicate survives.
- `READY_FOR_CRM` is an illustrative status label; align with the CRM contract if different.
- No actual GitHub URL, Loom/YouTube URL or execution screenshot has been supplied for this assessment.
