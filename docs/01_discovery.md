# 01 — Discovery: Medical Information enquiry handling at Nordvale Pharma

> **Fictional case.** Nordvale Pharma, its products, people and every number in this document are invented for a portfolio exercise. Nothing here describes a real company, medicine or patient.

## 1. Context

Nordvale Pharma (fictional) runs a Medical Information (MI) team that answers unsolicited questions about its marketed products. Questions come from three groups:

| Requester | Typical questions |
|---|---|
| Healthcare professionals (physicians) | dosing in special populations, interactions, storage, use outside the approved label |
| Pharmacists | storage and stability, substitution, product availability, packaging issues |
| Patients and caregivers | how to take the product, what to do after a missed dose, side effects they noticed |

Three things can be hidden inside any enquiry, whatever its wording:

- **a possible adverse event (AE)** — something bad that happened to a person after using the product, which the company must pass to its drug safety (pharmacovigilance) function;
- **a product quality complaint** — a defect in the product or its packaging, which goes to Quality;
- **an off-label question** — use outside the approved indication, dose, population or route, which only a qualified MI specialist may answer, and only reactively.

## 2. Current process (as-is), step by step

1. **Intake.** Enquiries arrive through a web contact form, a shared mailbox and phone calls (phone calls are typed up by the coordinator as a free-text note).
2. **Logging.** The MI coordinator copies each enquiry, including the requester's name, email, phone and any patient details, into a shared tracker spreadsheet.
3. **Triage by reading.** The coordinator reads the text and assigns a category and priority by judgement. There is no checklist for spotting adverse events.
4. **Assignment.** The enquiry is assigned to an MI specialist by product.
5. **Research.** The specialist searches a shared drive for a matching standard response document (SRD). Several versions of the same SRD sit side by side; the approved one is not marked.
6. **Drafting.** The specialist writes a reply, often by copying and editing an older reply.
7. **Review.** Replies to physicians are peer-reviewed; replies to patients and pharmacists usually are not.
8. **Sending.** The specialist sends the reply from the shared mailbox.
9. **Safety and quality forwarding.** If someone *notices* a possible AE or a complaint, they forward the enquiry by email to Drug Safety or Quality. Nothing forces this check, so it depends on attention.
10. **Closure.** The tracker row is marked closed. There is no record of which SRD version was used or who approved the wording.

## 3. Roles

| Role | Responsibility today | Role in the new process |
|---|---|---|
| MI coordinator | intake, logging, triage, assignment | monitors queues, handles "needs human" items |
| MI specialist | research, drafting, sending | reviews AI-prepared drafts: approve / reject / edit |
| Senior MI reviewer (medical) | peer review of physician replies | answers all off-label questions; owns the standard response library |
| Drug Safety officer (pharmacovigilance) | receives forwarded AEs | works the safety queue; owns AE detection rules |
| Quality officer | receives forwarded complaints | works the complaint queue |
| Data Protection Officer (DPO) | — (not involved today) | signs off data minimisation and the impact assessment |
| Head of Medical Information | process owner | business owner, go/no-go decision |
| IT / automation owner | shared drive and mailbox | runs and monitors the workflow |

## 4. Problems found

| # | Problem | Why it matters |
|---|---|---|
| P1 | Possible AEs sometimes sit in the ordinary queue for days | Safety reporting timelines run from the moment the company becomes aware of a case, not from when someone notices it (see [04_compliance.md](04_compliance.md)) |
| P2 | AEs are often hidden in harmless-looking wording ("my mother got a rash after the tablets, can she keep taking them?") | Reading-based triage misses them under time pressure |
| P3 | Personal data is copied into the tracker, emails and drafts | More copies mean more exposure and harder deletion |
| P4 | Specialists copy old replies; the approved SRD version is unclear | Risk of outdated or unapproved medical content |
| P5 | Off-label questions are sometimes answered by whoever picks them up | Promotional-compliance and patient-safety risk |
| P6 | No audit trail of who approved which wording from which source | Cannot show an inspector or auditor what was sent and why |
| P7 | Slow first response | Requesters wait; patients call again or stop asking |

## 5. Baseline numbers (FICTIONAL)

> **All figures below are invented for this case** to give the project a measurable starting point. They are not taken from any real company or study.

| Metric (fictional) | Baseline |
|---|---|
| Enquiries per month | 1,200 |
| Mix: product information / AE / complaint / off-label / other | 58% / 9% / 6% / 12% / 15% |
| Median time to first response | 3.2 working days |
| Possible AEs that reach Drug Safety within 1 working day | 78% |
| AEs found in the ordinary queue for more than 24 h (internal sample of 110 AE cases) | 4 (3.6%) |
| Replies in a sample of 50 that used an outdated SRD version | 11 (22%) |
| Average handling time per enquiry | 25 minutes |
| Places where requester personal data is stored per enquiry | 4 (form, mailbox, tracker, draft) |
| MI team size | 6 FTE |

## 6. Requirements

### Functional

| ID | Requirement |
|---|---|
| FR-01 | Accept an enquiry from a web form or webhook (text, requester type, product name, channel). |
| FR-02 | Mask names, email addresses, phone numbers and identifying numbers **before** any text is sent to an AI model. |
| FR-03 | Detect possible AEs with **two independent checks**: keyword rules and an AI classifier. If **either** flags an AE, route the enquiry to the Drug Safety queue immediately, with no reply draft. |
| FR-04 | Classify every enquiry into exactly one of: product information, adverse event, product complaint, off-label question, other — returned as JSON with a short rationale. |
| FR-05 | For ordinary product-information questions, draft a reply **only** from approved standard responses (each with an ID and version) and cite the source ID. |
| FR-06 | If no approved standard response matches, produce no draft and route to a human. |
| FR-07 | Route every off-label question to a senior MI reviewer, never to a draft. |
| FR-08 | Route product complaints to the Quality queue. |
| FR-09 | Run code checks on every draft: cited ID exists in the library, no personal data in the AI-bound text, no forbidden phrases (e.g. dosing advice, "safe for you", recommendations to stop or change treatment). |
| FR-10 | Present every draft to a human reviewer who must approve, reject or edit it. |
| FR-11 | Write an audit log entry for every routing decision and every human decision: who, when, what decision, which source ID and version. |

### Non-functional and constraints

| ID | Requirement |
|---|---|
| NFR-01 | Nothing is sent outside the system automatically: no emails, no messages to requesters. Approved drafts wait for a human to send. |
| NFR-02 | The AI never gives medical advice and never writes free-form medical content. |
| NFR-03 | A failure of the AI step must fail safe: the enquiry goes to a human queue, never to "no action". |
| NFR-04 | All test data is synthetic; no real patient, product or company data. |
| NFR-05 | Routing to the safety queue must not depend on the AI being available (keyword rules run first and alone are enough to route). |
| NFR-06 | The workflow and its rule set are version-controlled and exportable without credentials. |

## 7. Success criteria

### For the build-and-test phase (measured in stage 4)

| Criterion | Target |
|---|---|
| Possible AEs routed to the safety queue (recall on the test set) | **100%** — every miss documented |
| Classification accuracy across the five categories | ≥ 85% |
| Seeded personal data items masked before the AI step | 100% |
| Drafts that cite an existing standard-response ID and version | 100% |
| Drafts produced for AE, complaint or off-label enquiries | 0 |
| Human decisions with a complete audit log entry | 100% |

### For a live pilot (fictional targets, not measured in this portfolio case)

| KPI | Baseline (fictional) | Target (fictional) |
|---|---|---|
| Possible AEs reaching Drug Safety within 1 working day | 78% | 100% |
| Median time to first response | 3.2 days | ≤ 1 day |
| Replies citing an outdated SRD version | 22% | 0% |
| Places storing requester personal data per enquiry | 4 | 1 (intake record only) |

## 8. Out of scope

- Sending replies to requesters (stays a human action in the existing mailbox).
- Submitting individual case safety reports to authorities (stays with Drug Safety in its own system).
- Phone-call transcription.
- Writing new standard responses (the library is owned by the senior MI reviewer).

## 9. Assumptions and open questions

- An approved standard-response library exists and is owned by a named person; this case uses a small fictional library.
- Drug Safety, not this workflow, decides whether a flagged enquiry is a valid reportable case. The workflow only makes sure nothing is missed.
- The AI provider is used under a data processing agreement; see [04_compliance.md](04_compliance.md).
- Open: should every flagged AE also trigger a short holding reply to the requester? (Decision for Drug Safety; not automated here.)
