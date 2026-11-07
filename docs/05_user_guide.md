# 05 — User guide for the Medical Information team

> Fictional case (Nordvale Pharma). For MI coordinators, MI specialists, the senior MI reviewer and Drug Safety. Not medical advice.

## 1. What changed, in one minute

- Enquiries from the web form are **sorted automatically** into queues within seconds.
- Anything that **might** be an adverse event goes straight to **Drug Safety**. You will not see a draft for it.
- For ordinary product questions you get a **draft built only from an approved standard response**, with its ID and version. You decide whether it is sent.
- **Nothing is sent to a requester automatically.** You still send replies from the MI mailbox.
- Personal data is hidden from the AI. In queues you see the masked text (`[NAME]`, `[PHONE]` …). Contact details are in the intake record, which only MI staff can open.

## 2. The queues

| Queue | Who works it | What is in it | Draft? |
|---|---|---|---|
| `drug_safety` | Drug Safety officer | Any enquiry where the keyword rules **or** the AI saw a possible adverse event or special situation (pregnancy, overdose, medication error, child exposure, lack of effect) | No |
| `quality` | Quality officer | Product or packaging complaints with no adverse event | No |
| `senior_mi` | Senior MI reviewer | Questions about use outside the approved label (other age, dose, route, indication) | No |
| `draft_review` | MI specialist | Product questions with a matching approved response | Yes |
| `mi_human` | MI coordinator / specialist | Everything the system could not handle safely: no matching response, "other" topics, AI error, masking check failed | No |

Each row in `PIC Queue` shows a **reason**. Common reasons:

| Reason | Meaning | What to do |
|---|---|---|
| `possible adverse event: rules (…) + AI (…)` | Both detectors flagged it | Drug Safety assesses as usual |
| `possible adverse event: AI (…)` only | Only the AI flagged it | Drug Safety assesses. If it is not a safety case, Drug Safety moves it on (usually to `senior_mi` or `draft_review` by hand) and notes why |
| `no approved response matches` | Product question outside the library | Answer it as before; if it recurs, ask the senior MI reviewer to write a new standard response |
| `category other` | Not a medical question | Forward to the right team |
| `ai_unavailable` / `ai_invalid_output` | The AI step failed | Triage by hand; the AE keyword rules were still applied |
| `pii_check_failed` | Masking left something that looked like personal data, so the AI was not called | Triage by hand; report the example to IT so masking can be improved |
| `citation_check_failed` / `draft blocked by checks` | The AI pointed to a response that is not approved for this product, or the draft failed a check | Answer by hand; report to IT |

## 3. Reviewing a draft (MI specialist)

1. Open the **draft review form** (you must be logged in to n8n — your login is recorded in the audit log).
2. For each draft, read in this order:
   - the **masked enquiry** — does the draft actually answer it?
   - the **source** (`SR-…` and version) — is it the right product and still current?
   - the **checks** line — all should say `pass`;
   - the **draft text**.
3. Choose a decision:

| Decision | When | What happens |
|---|---|---|
| **approve** | The draft answers the question fully and correctly | Status becomes `approved` — ready for **you** to send from the MI mailbox |
| **edit** | Small wording change needed (greeting, a sentence order) | Edit in the text box. **Keep the `[Source: …]` line exactly as it is.** The edit is checked again; if it fails, it is blocked and the draft stays pending |
| **reject** | Wrong response, missing information, or you notice anything safety-related | **A reason is required.** Then handle the enquiry by hand |
| **keep pending** | You need more time | Nothing changes |

### Stop and send to Drug Safety if…

…while reviewing, you notice **any** sign of a side effect, a pregnancy, an overdose, a wrong medicine, a child taking the product, or "it stopped working" — even if the system did not flag it. **Reject** the draft with the reason "possible AE — forwarded to Drug Safety" and forward the enquiry immediately. A missed flag is the most important thing for the team to learn from: tell the IT owner so the rules can be updated.

### Edits are blocked if they contain

- recommendations or advice: "you should stop / take / increase …", "I recommend", "we recommend";
- promises: "completely safe", "100% safe", "no side effects", "guaranteed", "cure";
- diagnosis wording, or "no need to see a doctor";
- personal data (emails, long numbers);
- a missing or changed `[Source: …]` line.

## 4. Rules that never change

1. Never remove or ignore a Drug Safety routing. Only Drug Safety can decide that something is not a safety case.
2. Never paste a requester's personal data into a draft or a note.
3. Never answer an off-label question from the draft queue. Off-label goes to the senior MI reviewer only.
4. Never send a reply that you would not sign yourself. "The system suggested it" is not a reason.
5. If something looks wrong in the system, stop and tell the IT owner. Do not work around it.

## 5. Drug Safety officer

- Your queue contains everything **either** detector flagged. Expect some false alarms; that is intended.
- Day zero for reporting starts when MI received the enquiry, not when you open it. The intake time is in `PIC Intake.received_at` and in the queue row.
- To follow up with the reporter, open the intake record by `enquiry_id` (contact details live only there).
- You own the AE keyword rule list. Changes go through the change process in [06](06_training_and_handover.md) and the test must pass before they go live.

## 6. Senior MI reviewer

- Work the `senior_mi` queue (off-label) and any `mi_human` items that need a medical answer.
- You own the response library (`PIC Response Library`). Each entry has an ID, version, product, title, scope, text, status, approver and approval date.
- To update a response: add a **new version** with status `approved` and set the old version to `retired`. Never edit the text of an approved version in place — drafts and the audit log refer to the exact version.

## 7. FAQ

**Can the AI write an answer if there is no standard response?** No. It can only point to an approved response or say there is none.

**Does the AI see the patient's name?** No. Names, emails, phone numbers, ID numbers and dates of birth are masked first, and the AI call is blocked if anything is left.

**What if the AI is down?** Enquiries go to the `mi_human` queue. The keyword rules still route possible adverse events to Drug Safety.

**Who can see the audit log?** MI leads, Drug Safety, Quality, the DPO and auditors.
