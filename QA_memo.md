# De-identification QA Memo

**Reviewer:** Katrina Van
**Scope:** 8 note/output pairs (registration, referral, social history, telehealth intake, EHR message thread, ED note, device follow-up, billing summary)

## Highest-risk issues

1. **Clinical content silently corrupted (doc_02, doc_03).** The diagnosis in doc_02 changed from "hypertensive heart disease with heart failure" to "type 2 diabetes mellitus," and the blood type in doc_03 changed from O positive to AB negative. Neither field is an identifier — both are explicitly protected clinical content under the policy — so these aren't privacy failures, they're data corruption. If either value were acted on by a clinician downstream, this is a direct patient-safety risk, a different tier of severity than something like a leaked ZIP code, which is a privacy/compliance issue but doesn't affect care decisions.

2. **Raw identifiers passed through completely unchanged.** Across nearly every document, at least one required field was left as the real value with no substitution at all: an email address (doc_01, doc_05), an IP address (doc_04), a portal URL (doc_04, doc_05), a driver's license number (doc_06), a device serial number (doc_07), and an account number (doc_08). These are direct exposures of real PII in a document meant to be safe to share externally, and this was the single most common failure type in the review.

3. **Same real patient assigned two different fake identities across documents (doc_01 vs. doc_08).** Confirmed via matching real MRN and SSN in both original files, the same patient was de-identified as "Carmen L. Delgado" in one document and "Brenda Caldwell" in another. This breaks the policy's cross-document consistency requirement and would cause confusion for anyone trying to link records belonging to the same patient — a data-quality risk rather than a direct privacy exposure.

## Recurring model failure patterns

- **Partial name replacement.** In four separate instances (doc_01, doc_05 patient and staff names), only one half of a person's name was replaced — most often the surname — while the document still read naturally, making this class of leak easy to miss on a casual read.
- **Incomplete geography sets.** In three documents (doc_01, doc_06, doc_08), city and/or street address were correctly replaced while county and/or ZIP were left as the real value in the same sentence, suggesting the model treats geography fields independently rather than as a linked set that must change together.
- **Preserve-list content altered.** In four documents (doc_02, doc_03, doc_06, doc_07), clinically or contextually necessary content — diagnosis, blood type, vehicle make/model, device model — was changed instead of preserved. doc_03 was the most extreme case, with six of nine fields altered even though none were identifiers.
- **Derived fields skipped after the primary field is fixed.** In doc_04 and doc_05, the patient's name was correctly replaced, but linked technical fields nearby (URL tokens, email, phone) were left as the real value — and in both cases the leftover token still encoded the real patient's initials, a non-revealing violation stacked on top of the leak.

## Recommendations

1. **Treat linked fields as single units, not independent tokens.** Names should be replaced as one full unit (first + last together), not as separate first-name and last-name substitutions — this would prevent the partial-name leaks seen in doc_01 and doc_05. The same logic applies to geography: address, city, county, and ZIP should be replaced as one linked set, so a fix can't land "3 out of 4 complete" the way it did in doc_01, doc_06, and doc_08.

2. **Hard-lock preserve-list fields with an automated verification check.** Diagnosis, blood type, medications, vitals, device model, and vehicle make/model should never be substituted at all. Rather than relying on a person to catch it after the fact, add an automated check that compares each of these fields in the output against the original and rejects the output if they don't match exactly. This is the highest-priority fix given the patient-safety risk demonstrated in doc_02 and doc_03.

3. **Extend identifier detection to derived/secondary fields, not just primary ones.** Emails, URLs, phone numbers, and other technical fields were repeatedly left unchanged even when the primary name field nearby was correctly replaced (doc_04, doc_05). Detection should scan for identifiers throughout the whole document — including fields linked to an already-replaced name — not just the first or most obvious mention.

## Bonus: one automated validation check sketch

**Leak-substring detector.** For every field the model claims to have de-identified, check whether any contiguous substring of length >=4 from the *real* value still appears in the *output* value (case-insensitive, ignoring punctuation/whitespace).

```
for each (original_value, output_value) in identified_fields:
    real_tokens = extract_substrings(original_value, min_len=4)
    for token in real_tokens:
        if token.lower() in normalize(output_value).lower():
            flag(field, original_value, output_value, reason="possible leaked substring")
```

This single check would have caught the unchanged email/IP/URL-token/DL/serial/account-number leaks, the truncated VIN, and the retained-surname cases -- the majority of findings in this review -- without needing separate logic per field type.