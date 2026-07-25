# AI Operations Analyst Take-Home: De-identification QA

## Overview

Your task, should you accept, is to review a small batch of synthetic clinical notes and the
de-identified versions a model produced from them. Unlike a system that simply blanks
out sensitive text, this model returns a **synthetically de-identified** note: each
sensitive value is replaced with a realistic *fake* (surrogate) value, so the output
still reads like a normal note. This is what we return in production.

There are **no `[REDACTED]` markers**. To find problems you must compare each output
against its original and decide, value by value, whether the de-identification was done
correctly. A real value that leaked, or a low-quality fake, now blends in with the good
surrogates — so careful reading and judgment matter more than scanning for tags.

The files are synthetic and created for this exercise. They are not real patient records.

Expected time: 75-90 minutes. Please do not spend more than 2 hours.

## Files

- `data/original_notes/`: original synthetic notes
- `data/model_outputs/`: the model's de-identified version of each note
- `error_log_template.csv`: suggested format for your findings

Each original note has a matching model output with the same file name. Note that some
documents may describe the **same patient**, so it is worth checking for consistency
across files, not just within a single file.

## De-identification Policy for This Exercise

Treat each model output as a proposed production-ready, de-identified version of the
original note.

The model should replace the following with realistic surrogate values:

- Patient names and other person names that identify the patient
- Full dates
- Phone numbers, fax numbers, email addresses, URLs, and IP addresses
- Medical record numbers, SSNs, member IDs, account numbers, and similar identifiers
- Street addresses, cities, counties, ZIP codes, and other geography below state level
- Vehicle identifiers, driver license numbers, and device serial numbers

Each surrogate value must meet these quality requirements:

1. **Fully replaced.** No part of the real value may remain. A real surname left behind a
   fake first name, or a real ZIP left behind a renamed city, is still a leak.
2. **Realistic and correctly formatted.** A fake SSN must look like an SSN, a fake phone
   like a phone, a fake VIN like a VIN, and so on. Malformed or obvious-placeholder
   values are defects.
3. **Internally consistent.** The same real entity should map to the same surrogate
   everywhere it appears — within a document *and across documents*. Linked values should
   agree with each other (a fake email should match the fake name; a URL or visit token
   should not carry the real one), and date relationships (intervals, same-day events)
   should be preserved.
4. **Non-revealing.** A surrogate, or any retained token, must not encode or hint at the
   real value (for example, an email or URL built from the real name, initials, or date).

The model should **preserve, unchanged**, content that is clinically or contextually
useful and not a direct identifier — including diagnoses, symptoms, medications,
procedures, device model names (as opposed to serial numbers), blood type, vital signs,
vehicle make/model, and non-identifying demographic fields. Silently changing this content
to a different but plausible value is an error, even though the note still reads fine.

## Your Task

1. Compare each original note against its matching model output.
2. Fill out `error_log_template.csv` with the issues you find. For each issue cite both
   the original value and the output value.
3. Write a short QA memo, ideally under one page, summarizing:
   - The highest-risk issues you found
   - Recurring model failure patterns
   - 2-3 practical recommendations for improving the model, QA process, or guidelines

You do not need to find every possible issue. We care most about prioritization, clear
reasoning, and whether your findings would help an engineering or product teammate take
action. Not every value is a defect — many surrogates are handled correctly.

## Deliverables

Please submit:

- Your completed findings file, using `error_log_template.csv` or a similar spreadsheet
- Your short QA memo
- (Optional bonus) A brief sketch of one automated validation check you would add to catch
  a whole class of these issues at scale — for example, format validation for surrogate
  IDs, date-interval consistency, or detecting when a surrogate still contains the real
  value. A short description or rough pseudocode is plenty; do not spend much time here.

## FAQ

### Do I need to code?

No. You may use Excel, Google Sheets, SQL, Python, or any workflow you prefer. The only
optional coding is the bonus validation-check sketch, and a description is fine.

### Do I need HIPAA expertise?

No. Use the policy above. If something is ambiguous, explain your reasoning.

### Are these real patient notes?

No. All candidate-facing notes are synthetic.

### Should I report every issue?

No. Prioritize the issues that matter most. A concise, high-signal issue log is better
than a very long list of low-value observations.

### What if I see the same issue multiple times?

You can log representative examples and mention the broader pattern in your memo.

### What are reviewers looking for?

We are looking for careful review, practical severity judgment, clear communication, and
recommendations that would help a product or engineering team improve the system.
