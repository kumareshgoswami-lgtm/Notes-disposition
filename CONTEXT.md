# AR Auto-Audit: Notes vs Disposition (Status / RCA / Resolution Action)

Daily automated quality audit for Anka's offshore AR team. It checks each worked claim's
**Claim Status, Denial/Rejection Category, RCA and Resolution Action** against the
client rule workbook and against what the agent actually wrote in the **notes**.

Owner: Mick Harper (Anka Health). Built 26-Oct-2026 on the 23-Oct-2026 work dump.

---

## 1. Inputs

One Excel workbook (`Note vs Disposition-Status-RCA-Resolution.xlsx`, kept in OneDrive and **never committed**):

| Sheet | Role |
|---|---|
| 1st sheet (`AR_Work_Dump`) | One day of worked claims (≈1,900 rows, 40 cols). **Replace daily.** |
| `1. Overview` | Which form the agent sees; which fields are mandatory per form |
| `2. Claim Status - RCA` | Allowed RCA list per Claim Status / Denial Category / Rejection Category |
| `3. Resolution Actions` | The 18 AR actions, 3 Coding actions, which queue completes each, parked vs closed |
| `Final Status Action Matrix` | Y/N grid: which Resolution Action is allowed for which Claim Status |
| `action definitions` | Definition, use-when / do-not-use-when, and "notes must capture" per action |

Key dump columns: `Claim Status`, `Category` (denial category), `RCA`, `Rejection Category`,
`Last Resolution Action`, `From/To/Current Queue`, `Status Source`, `Representative Name`,
`Escalation Reason`, `Notes (PHI Safe` (sic: the header is truncated in the source), `Agent`, `Emp ID`, `Balance Amount`.

The rule tabs are **parsed at runtime**, so editing the grid or RCA lists changes the audit without code changes.

---

## 2. Pipeline

```
Daily dump (OneDrive, contains PHI)
   │
   ├─► ar_auto_audit.py   ── rules engine, runs fully local ──► AUDIT RESULT.xlsx (contains PHI; stays local)
   │
   └─► phi_scrub.py       ── de-identify ──► SCRUBBED.xlsx  (only file allowed to reach any AI model)
                                         ├─► QUARANTINE.xlsx (rows still risky; never sent)
                                         └─► REID KEY.xlsx   (token → real value; local, restricted)
                                                │
                              (planned) AI note reviewer: local Ollama first (no BAA in place)
```

### Run
```bash
python ar_auto_audit.py "<workbook.xlsx>" ["<output.xlsx>"]
python phi_scrub.py     "<workbook.xlsx>" ["<out_dir>"]
```
Requires Python 3.10+, `pandas`, `openpyxl`.

---

## 3. Audit scoring convention

- Every claim is audited (100% population, not a sample).
- **Fatal** = the error changes the financial or claim outcome (wrong action or status, a recovery route lost, duplicate-denial risk, a mandatory RCA missing on a rejection).
- **Non-fatal** = gaps in fields or note hygiene.
- Fatal is **binary per claim**.
  - **Critical accuracy** = (claims − fatal claims) / claims
  - **Overall accuracy** = (claims − claims with any defect) / claims
- Agent zones by critical accuracy: Green ≥98, Watch 95–98, Amber 90–95, Red <90.
- N-codes come from keyword inference over the note text. **A QA person confirms them before agent feedback.**

### Defect codes

| Code | Severity | Check |
|---|---|---|
| F01 | Non-fatal | Claim Status blank (non-Coding form) |
| F02 | Non-fatal | Status Source blank |
| F03 | Non-fatal | Denied without Denial Category |
| F04 | Non-fatal | Rejected without Rejection Category |
| F05 | **Fatal** | Rejected without RCA (mandatory on Rejection) |
| F06 | Non-fatal | Status from Payer Call/IVR but Representative Name blank |
| F07 | Non-fatal | Escalated without Escalation Reason |
| F08 | **Fatal** | Coding-queue claim closed with a non-coding action |
| F09 | **Fatal** | Coding-only action used outside Coding queue |
| G01 | **Fatal** | Action not allowed for status per Final Status Action Matrix |
| G02 | **Fatal** | Appeal Required/Submitted on a claim that is not Denied |
| R01 | Non-fatal | Denial Category not in approved list |
| R02 | Non-fatal | Denial Category filled but status not Denied |
| R03 | Non-fatal | RCA not in the list for its status/category |
| R04 | Non-fatal | Denied with category but no RCA (needed for trending) |
| N01 | **Fatal** | Note's concluding action contradicts the disposition |
| N02 | Non-fatal | Note's status (denied / not on file / paid) contradicts Claim Status |
| N03 | **Fatal** | "Zero balance" or "OJT out of scope" closure while the note shows open balance or work done |
| N04 | **Fatal** | Claim Resubmitted (freq 1) on a denied claim with an ICN; should be Corrected Claim |
| N05 | **Fatal** | Adjustment while note cites an open recovery route and no adjustment |
| Q01 | Non-fatal | Note < 40 characters |
| Q02 | Non-fatal | Payer call without call reference # |
| Q03 | Non-fatal | Note documents a call but Status Source isn't Payer Call |
| Q04 | Non-fatal | Parked action without TAT / next follow-up in note |

How N01 works: notes carry claim history, so only the **concluding clause** is judged. That's the text after the last "hence / therefore / thus / so …ed", or else the last 40% of the note. Action families that are near-equivalent (resubmit, corrected claim and rebill; appeal and MR) are not flagged against each other.

Alias used: `Escalate to Client` (data) = `Escalated to Onshore Team` (rules).

### Output workbook tabs
Summary + code legend · Agent Scorecard (accuracy, $ at risk, top defect, zone) · Defect Pareto ·
Grid Violations (status × action) · Client View · Findings (Line Level).

---

## 4. Baseline result: 23-Oct-2026 dump

- 1,892 claims, 61 agents, 12 clients.
- **Critical accuracy 91.5%**, overall accuracy 62.6%. 161 fatal claims, $164.7K balance on fatal claims.
- Fatal mix: N01 73, F05 50, G01 21, N03 16, N04 11, N05 1.
- Agent zones: Green 25, Watch 11, Amber 9, Red 16. Lowest: 37.5%, 55.0% and 62.5% critical accuracy.
- The top 5 defects make up 65% of all defects: Q04 190, F02 147, F01 134, Q03 117, R02 98.
- A manual spot-check of 16 fatal flags confirmed them. Formal calibration is still pending.

### Problems with the setup itself (not agent errors)
- Action-name mismatch: "Escalate to Client" vs "Escalated to Onshore Team".
- Categories used but not in the approved list: `Contractual Adjustment`, `Other/Uncategorized`. `Submission` has no RCAs.
- The system allows submission with Status Source / Claim Status blank, despite the "mandatory" rule (~280 defects/day).
- 13 Unique IDs were worked twice in one day.
- The "PHI Safe" notes column still contains names, claim/ICN/member/auth numbers, phones and addresses.

---

## 5. PHI handling (no BAA in place)

- Assume **no BAA** with any AI or cloud vendor. Raw dumps, audit results, quarantine files and the REID KEY **never** go to a cloud service or to git (`.gitignore` blocks `*.xlsx`, `*.csv`, `scrub_out/`).
- `phi_scrub.py` (Safe Harbor style):
  - Names, phones, emails, URLs, SSNs, addresses/ZIPs and long IDs become salted-hash tokens (stable across days).
  - Dates become day offsets from Worked Date (`[D-34]`).
  - Kept: CPT/HCPCS (including `CPT-` prefixed), ICD-10, modifiers, CARC/RARC, $ amounts, payer names.
  - A final residual scan quarantines rows that still look risky. On 23-Oct: 1,883 safe, 9 quarantined.
- Pattern scrubbing is not proof of de-identification. **Compliance must approve** the method and a reviewed ~100-row sample before scrubbed data reaches any external service.
- Set a private `SALT` in `phi_scrub.py`, and keep it out of git for production.

---

## 6. Roadmap

1. **Calibrate (week 1):** QA manually audits about 50 fatal flags plus clean samples. Tune regexes to at least 90% precision, then lock which codes are fatal.
2. **Run it daily:** automated 100% check. A QA person confirms fatal flags, plus a random 5% of clean claims.
3. **AI note reviewer:** on the scrubbed file, a local **Ollama** model (8–14B) returns structured JSON `{status, action, rca, confidence}` constrained to workbook lists. Run it only on flagged claims plus a 10–15% sample (≈300/day). Compare against the QA answer key before trusting it. Switch to a cloud model only if a BAA exists.
4. **Trends:** 7/30-day agent accuracy, feeding the BQ/PIP governance process. Defect repeat rates by client and payer.
5. **More checks:**
   - Timely-filing risk (age vs payer TFL, especially before adjustments).
   - Adjustment without approval reference.
   - Bill-to-patient on CO group code.
   - Repeat touches with no progress.
   - Next-FUP date vs the action's required TAT.

---

## 7. Files

| File | Purpose |
|---|---|
| `ar_auto_audit.py` | Rules engine and scorecard writer |
| `phi_scrub.py` | De-identification, quarantine and re-identification key |
| `CONTEXT.md` | This document |
| `.gitignore` | Blocks all data and outputs from git |
