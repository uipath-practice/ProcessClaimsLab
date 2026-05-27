# Normalisation rules

Agents read documents that come in many shapes — currencies, locales, date formats, address conventions. The case entity is the system of record and must hold canonical, comparable values. The case-write layer applies the rules below between extraction and persistence.

These rules apply to the values that land **on the case entity**. The original document text remains in `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_AssessmentReportJSON` unchanged for human review.

---

## 1. Dates

| Surface | Format | Direction |
|---|---|---|
| Document text | Source-dependent (`DD/MM/YYYY`, `MM/DD/YYYY`, `D MMM YYYY`, ...) | Read |
| Agent JSON | `DD/MM/YYYY` | Read / write |
| Case entity field (e.g. `incidentDate`) | ISO 8601 `YYYY-MM-DD` | Write |
| Case entity timestamp | ISO 8601 with timezone `YYYY-MM-DDTHH:mm:ssZ` | Write |

### Parsing rules

1. **Prefer the document's locale.** If the source document carries a clear locale signal (insurer header country, currency symbol, address country), use that country's conventional date format.
2. **Default to DD/MM/YYYY when ambiguous.** This is the synthetic-document generator's current default.
3. **Reject impossible dates.** A claim with `dateOfSubmission < incidentDate` is impossible and the eligibility agent must flag it (rule E-4 in the sample HK example).
4. **Two-digit years are not accepted.** All dates carry a 4-digit year.

### Edge case — MM/DD/YYYY vs DD/MM/YYYY ambiguity

When a date could parse as either (e.g. `03/04/2026`), Agent 01 flags the ambiguity in its E-4 / E-5 check rather than guessing. The reviewer sees the ambiguity in the Action App and resolves it via the inquiry mechanism or by continuing manually.

---

## 2. Currency and amounts

| Surface | Format | Direction |
|---|---|---|
| Document text | Symbol + formatted number (`HK$230,000.00`, `₹125,000.00`, `$3,200.50`) | Read |
| Agent JSON | Preserve formatted string in document-derived fields; emit bare number + currency code in agent-calculated fields | Read / write |
| Case entity numeric field (e.g. `totalClaimAmount`) | Bare decimal | Write |
| Case entity currency field (e.g. `currency`) | ISO 4217 code (`USD`, `INR`, `HKD`, `EUR`, `GBP`, `SGD`, `CAD`) | Write |

### Symbol → ISO 4217 mapping

| Symbol(s) | ISO code | Notes |
|---|---|---|
| `$` | Inferred from policy-currency context (`USD` for US insurer; could also be `CAD`, `AUD`, `SGD`, `NZD`) | Don't guess — always cross-reference the insurer / property country. |
| `US$`, `USD` | `USD` | |
| `HK$`, `HKD` | `HKD` | |
| `S$`, `SGD` | `SGD` | |
| `CA$`, `CAD` | `CAD` | |
| `A$`, `AUD` | `AUD` | |
| `NZ$`, `NZD` | `NZD` | |
| `€`, `EUR` | `EUR` | |
| `£`, `GBP` | `GBP` | |
| `₹`, `INR`, `Rs`, `Rs.` | `INR` | |
| `¥`, `JPY` | `JPY` | If ambiguous with CNY, use the policy country. |
| `¥`, `CNY`, `RMB`, `元` | `CNY` | |

When the policy carries a clear ISO code (`coverage_summary.currency`), that wins over symbol inference for ambiguous cases.

### Parsing rules

1. **Strip the symbol and grouping separators** (`HK$230,000.00` → `230000.00`).
2. **Preserve decimal places.** Two decimals is standard; JPY and a few others use zero — follow the document.
3. **No FX conversion anywhere.** Cross-currency comparisons are forbidden in agent rules.
4. **Mixed currencies inside one claim is an error.** If the assessor report's `cost_summary` currency differs from the policy currency, Agent 02 fires AV-5 (internal consistency) with severity `critical`.

### Worked example

| Source string | Symbol | Currency context (policy) | Numeric | Stored on case |
|---|---|---|---|---|
| `HK$230,000.00` | `HK$` | `HKD` | `230000.00` | `230000.00`, currency=`HKD` |
| `$3,200.50` | `$` | `USD` (Apex Mutual Hartford) | `3200.50` | `3200.50`, currency=`USD` |
| `₹125,000.00` | `₹` | `INR` | `125000.00` | `125000.00`, currency=`INR` |
| `Rs 75,000` | `Rs` | `INR` | `75000.00` | `75000.00`, currency=`INR` |

---

## 3. Addresses

| Surface | Format | Direction |
|---|---|---|
| Document text | As-typed | Read |
| Agent JSON | As-extracted (preserve original casing and punctuation) | Read / write |
| Case entity field | As-extracted (display form) | Write |
| Rule E-3 comparison form | Lowercased, whitespace-collapsed, abbreviations expanded, punctuation stripped | Internal only |

### Normalisation for E-3 (address match)

Applied to claim, policy, and assessment-report addresses before comparison:

1. **Lowercase** the entire string.
2. **Collapse whitespace** to single spaces.
3. **Expand common abbreviations**:
   - `st` / `st.` → `street`
   - `rd` / `rd.` → `road`
   - `ave` / `av` / `av.` → `avenue`
   - `apt` / `apt.` / `#` → `unit`
   - `bldg` → `building`
   - `flr` / `fl.` → `floor`
4. **Strip punctuation** (`,`, `.`, `-`, `/`, `(`, `)`).
5. **Country-specific normalisations** are out of scope for v1 (HK floor designations like `23/F` are kept literal).

The normalised form is used only for the E-3 comparison. The case-entity address fields and the JSON outputs keep the original display form.

---

## 4. Booleans

| Source representation | Normalised |
|---|---|
| `"Yes"`, `"yes"`, `"YES"`, `"Y"`, `"Y."`, `"✓"` | `true` |
| `"No"`, `"no"`, `"NO"`, `"N"`, `"N."`, `"-"`, blank | `false` |
| `"True"`, `"true"`, `"TRUE"` | `true` |
| `"False"`, `"false"`, `"FALSE"` | `false` |
| `"1"` | `true` |
| `"0"` | `false` |
| `null`, `""` | `null` (not false) |

Agents emit canonical `true` / `false` / `null` in JSON outputs. The case entity stores them as native booleans (with the null allowance where the field is nullable).

---

## 5. Names

Names are NOT normalised on the case entity — they are stored as-extracted. Rule E-2 (identity match) does the comparison with case-insensitive matching and nickname tolerance (e.g. `Robert` / `Bob`, middle-name variants), but the stored name is the source name.

---

## 6. Phone numbers

Phone numbers are stored as-extracted. No normalisation. The agents never use phone numbers as keys or for matching, so format variance is harmless.

---

## 7. Email addresses

Emails are stored as-extracted. The Settlement robot uses the email exactly as it appears on `out_ClaimDataJSON.ClaimClaimant[0].EmailAddress`. Validation (regex match for a valid local-part@domain) is applied at intake and surfaced as a data-quality flag if it fails.

---

## 8. Where these rules live in code

| Rule | Owner |
|---|---|
| Date and timestamp normalisation | Case-write layer of the intake robot and each agent's output-handler |
| Currency symbol → ISO mapping | Agent 01 (sets `coverage_summary.currency`) |
| Address E-3 normalisation | Agent 01 (internal — output addresses are display form) |
| Boolean normalisation | Intake robot (case-write); agents may rely on this canonicalisation |
| Symbol stripping for arithmetic | Each agent that does arithmetic (Agent 04 mainly) |

---

*End of normalisation.md.*
