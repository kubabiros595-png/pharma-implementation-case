# Test report — PIC Eval

> Fictional data only. Every number below comes from the n8n data tables `PIC Test Runs` and `PIC Test Results`, written by the `PIC Eval` workflow (exported here as [test_runs.csv](test_runs.csv) and [test_results_per_case.csv](test_results_per_case.csv)).

## What was tested

- **26 fictional enquiries** ([../data/test_enquiries.csv](../data/test_enquiries.csv)): 10 answerable product questions, 1 product question with no approved answer, 2 non-medical, 4 off-label, 7 possible adverse events (5 of them hidden inside another question or described as a "special situation"), 2 product complaints.
- **10 personal data items planted in the enquiry text** (names, emails, phone numbers, a date of birth, a patient record number), plus **47 contact-detail fields** (name, email, phone) that must never reach the AI.
- **5 review decisions** through the review workflow: approve, valid edit, edit with a forbidden phrase (must be blocked), reject without a reason (must be blocked), reject with a reason.
- **Audit log coverage:** one "routed" row per enquiry and one "review" row per decision.

Every run goes through the real workflow: real masking code, real keyword rules and real calls to Claude.

## Results

| Metric | Target | Run 1 (`run-124`, v1.0) | Run 2 (`run-152`, v1.1) | Run 3 (`run-180`, v1.1, test set v2) |
|---|---|---|---|---|
| Possible AEs routed to Drug Safety | 100% | **7 / 7** | **7 / 7** | **7 / 7** |
| …caught by keyword rules alone | — | 7 / 7 | 7 / 7 | 7 / 7 |
| …caught by Claude alone | — | 7 / 7 | 7 / 7 | 7 / 7 |
| False alarms to Drug Safety | as few as possible | 0 | 1 (OL-04) | 0 |
| Classification accuracy (5 categories) | ≥ 85% | 25 / 26 (96%) | 26 / 26 (100%) | 26 / 26 (100%) |
| Routed to the expected queue | — | 25 / 26 | 25 / 26 | 26 / 26 |
| Planted personal data masked before AI | 100% | 10 / 10 | 10 / 10 | 10 / 10 |
| Contact fields that reached the AI text | 0 | 0 of 47 | 0 of 47 | 0 of 47 |
| Drafts built (expected 10) | 10 | 10 | 10 | 10 |
| Drafts citing an existing, approved ID + version | 100% | 10 / 10 | 10 / 10 | 10 / 10 |
| Drafts citing the expected response | — | 10 / 10 | 10 / 10 | 10 / 10 |
| Drafts for AE, complaint or off-label enquiries | 0 | 0 | 0 | 0 |
| Review decisions handled as expected | 5 / 5 | 5 / 5 | 5 / 5 | 5 / 5 |
| Audit log rows found / expected | 100% | 31 / 31 | 31 / 31 | 31 / 31 |
| Overall pass (all targets) | — | pass | pass | pass |

**No adverse event was missed in any run.**

**Test set v2 (run 3).** Before run 3, case AE-02 was reworded: it mentioned a real active substance, which breaks the rule that the case uses no real medicines. "…taken together with [real substance]" became "…taken together with a painkiller from the pharmacy". The 25 other cases are unchanged. Runs 1–2 used test set v1; run 3 used v2 (n8n table `PIC Test Cases`; v1 kept as `PIC Test Cases v1 (superseded)`).

## Every deviation, explained

**Run 1 — PC-01 (product complaint) went to the human queue instead of Quality.**
Claude classified it correctly as a product complaint, but replied with two JSON blocks (the second one prefixed with "Correction: returning the required JSON only"). The v1.0 parser expected exactly one block, rejected the reply and routed the enquiry to the MI human queue with reason `ai_invalid_output`. This is the fail-safe working as designed: nothing was lost, a person would have picked it up.
*Fix (v1.1):* the parser now extracts every JSON block. It accepts them only if they agree on the category; if any block flags an adverse event, the flag wins; if blocks cite different response IDs, no draft is built. Unit-tested on the exact failing reply before re-running.

**Run 3 — no deviations.** All 26 cases matched the expected category and queue.

**Run 2 — OL-04 (off-label dose question) went to Drug Safety instead of the senior MI reviewer.**
The text: *"Can I take twice the normal dose of Brenquilo if my cough is really bad at night?"* Claude classified it correctly as off-label, but also set `ae_suspected = true` ("possible lack of effect… possible intended overdose"). The keyword rules did not fire. Because either signal is enough, the enquiry went to Drug Safety. This is a false alarm, but in the safe direction: the prompt tells the model to flag when unsure, and Drug Safety would close it and pass it to the senior MI reviewer. In runs 1 and 3 the same enquiry was not flagged, which shows the model is not fully deterministic on borderline wording.
*Decision:* not tuned away. Making the model less sensitive to pass this one case would trade a harmless false alarm for a higher risk of a missed AE.

## Limitations of this test

- 26 cases is a small, hand-written set. The keyword rules and the test cases were written by the same person, so the 7/7 for the rules is an optimistic estimate for real-world wording. In a real project the rules would be tuned and validated on historical, de-identified enquiries by Drug Safety.
- All enquiries are in English.
- Three runs only; model output varies between runs (see OL-04). A real validation would repeat the run several times and track the spread.
- The workflow was built and tested on n8n Cloud with fictional data. It was not deployed.
