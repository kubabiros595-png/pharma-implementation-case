# 04 — Compliance: GDPR, EU AI Act and pharmacovigilance

> **Not legal advice.** This is a portfolio case on a fictional company. It shows how an implementation team would prepare the compliance questions for the client's Data Protection Officer, Legal and Drug Safety, who make the actual decisions. Every legal statement links to the official source it is based on; links were opened and checked on 7 November 2026.

Sources used (all official):

| Short name | Official source |
|---|---|
| GDPR | [Regulation (EU) 2016/679](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng) |
| AI Act | [Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) |
| AI Omnibus | [Regulation (EU) 2026/1744](https://eur-lex.europa.eu/eli/reg/2026/1744/oj/eng), amending the AI Act |
| MDR | [Regulation (EU) 2017/745](https://eur-lex.europa.eu/eli/reg/2017/745/oj/eng) on medical devices |
| Medicines Directive | [Directive 2001/83/EC](https://eur-lex.europa.eu/eli/dir/2001/83/oj/eng) |
| PV Implementing Regulation | [Implementing Regulation (EU) No 520/2012](https://eur-lex.europa.eu/eli/reg_impl/2012/520/oj/eng) |
| GVP Module VI | [EMA, GVP Module VI Rev 2 (EMA/873138/2011 Rev 2)](https://www.ema.europa.eu/en/documents/regulatory-procedural-guideline/guideline-good-pharmacovigilance-practices-gvp-module-vi-collection-management-submission-reports_en.pdf) |
| EMA AI reflection paper | [EMA/CHMP/CVMP/83833/2023](https://www.ema.europa.eu/en/documents/scientific-guideline/reflection-paper-use-artificial-intelligence-ai-medicinal-product-lifecycle_en.pdf) |
| EU-US Data Privacy Framework | [Commission Implementing Decision (EU) 2023/1795](https://eur-lex.europa.eu/eli/dec_impl/2023/1795/oj/eng) |

---

## 1. GDPR

### 1.1 Roles

- **Nordvale Pharma** decides why and how enquiries are processed, so it is the **controller**.
- **The AI provider** processes masked enquiry text on Nordvale's instructions, so it acts as a **processor**. Processing by a processor must be governed by a contract that sets out the subject matter, duration, nature and purpose, data types and the controller's rights ([GDPR Art. 28(3)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)), and the controller may only use processors that give sufficient guarantees ([Art. 28(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)).
- **The workflow host** (n8n) is a second processor for the same reasons.

### 1.2 What personal data is involved

Enquiries can contain **health data** — personal data about a person's physical or mental health ([Art. 4(15)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)) — which is a special category whose processing is prohibited unless an Article 9(2) condition applies ([Art. 9(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). Contact details (name, email, phone) are ordinary personal data.

### 1.3 Legal basis (for the DPO to confirm)

| Processing | Article 6 basis (proposal) | Article 9 condition (proposal) |
|---|---|---|
| Recording and forwarding possible adverse events to Drug Safety | Legal obligation of the company ([Art. 6(1)(c)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)) — the marketing authorisation holder must record suspected adverse reactions it receives ([Directive 2001/83/EC, Art. 107](https://eur-lex.europa.eu/eli/dir/2001/83/oj/eng)) | Public interest in public health, including the safety of medicinal products, based on law with safeguards ([Art. 9(2)(i)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)) |
| Answering ordinary medical information enquiries | Legitimate interests ([Art. 6(1)(f)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)), subject to a balancing test | To be decided by the DPO per country; options are listed in [Art. 9(2)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng) |

### 1.4 Data minimisation and privacy by design

GDPR requires personal data to be limited to what is necessary ([Art. 5(1)(c)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)) and requires technical measures such as pseudonymisation and data minimisation to be built into the processing ([Art. 25(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). How the solution does this:

| Measure | Where |
|---|---|
| Names, emails, phones, ID numbers, dates of birth replaced with placeholders **before** the AI step | `Mask personal data` node |
| Second scan blocks the AI call if anything is left | `Check masking` node |
| Contact details kept in one table only (`PIC Intake`), never sent to the AI | data flow, [02_solution_design.md §5](02_solution_design.md) |
| Queues and audit log store masked text only | `Save to queue`, `Write audit log` |
| The AI receives no response texts and no identifiers linking back to the intake record | `Build AI request` |

Test result: 10/10 planted personal data items masked; 0 of 47 contact fields reached the AI text ([results](../results/README.md)).

### 1.5 Automated decisions

A person has the right not to be subject to a decision based **solely** on automated processing that has legal or similarly significant effects ([Art. 22(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). In this design no decision about the requester is solely automated: the workflow only routes and prepares drafts; a named person approves every reply, and Drug Safety assesses every flagged case.

### 1.6 Data protection impact assessment (outline)

A DPIA is required for processing **on a large scale of special categories of data** ([Art. 35(3)(b)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). At the fictional volume (about 1,200 enquiries a month, many with health data), the DPO would most likely require one. Article 35(7) lists the minimum content ([Art. 35(7)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)); the outline below follows it:

| Art. 35(7) element | Content prepared by the project |
|---|---|
| (a) Description of processing and purposes | [01_discovery.md](01_discovery.md), [02_solution_design.md](02_solution_design.md) — process, data flow, roles |
| (b) Necessity and proportionality | AI only classifies and picks an approved response ID; masked text only; contact details stay in one store; human decides |
| (c) Risks to individuals | [03_risk_assessment.md](03_risk_assessment.md) — R1 missed AE, R3 data leak, R6 over-reliance |
| (d) Measures to address the risks | Masking + gate, processor contracts, access control on review form (n8n login), audit log, retention schedule below, training |

### 1.7 Retention (proposal)

Personal data may be kept in identifiable form no longer than necessary for its purpose ([Art. 5(1)(e)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). Proposed schedule for the DPO and Drug Safety to confirm:

| Data | Proposed retention (fictional) |
|---|---|
| Enquiries routed to Drug Safety | Follow the company's pharmacovigilance record retention, set under PV legislation ([Implementing Regulation (EU) No 520/2012](https://eur-lex.europa.eu/eli/reg_impl/2012/520/oj/eng)) — owned by Drug Safety, not by this workflow |
| Ordinary enquiries and replies (intake table, queue) | Fixed period agreed with the DPO (e.g. 3 years after closure), then deleted |
| Audit log | Same period as the record it describes |
| Data held by the AI provider | Zero or minimal retention under the processor contract; no use for model training |
| Test data | Fictional; no retention constraint |

A record of processing activities must also be kept ([Art. 30(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)).

### 1.8 Transfers outside the EU

If the AI provider processes data outside the EEA, the transfer needs a Chapter V basis ([Art. 44](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)): an adequacy decision ([Art. 45(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)), such as the decision for organisations certified under the EU-US Data Privacy Framework ([Decision (EU) 2023/1795](https://eur-lex.europa.eu/eli/dec_impl/2023/1795/oj/eng)), or standard data protection clauses ([Art. 46(2)(c)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). The procurement checklist in [06](06_training_and_handover.md) asks for the provider's transfer mechanism and data location options.

### 1.9 Transparency

When data is collected from the requester, the controller must provide its identity, the purposes and legal basis, recipients and any transfers at the time of collection ([Art. 13(1)](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)). The enquiry form would link to a privacy notice that also says an AI tool helps sort enquiries and that a person reviews every reply.

---

## 2. EU AI Act

### 2.1 Role

Nordvale uses the AI system under its own authority in a professional context, so it is a **deployer** ([AI Act Art. 3(4)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)).

### 2.2 Risk classification — reasoning

| Question | Assessment | Source |
|---|---|---|
| Is it a prohibited practice? | No. It does not manipulate, score people socially or use biometrics. | [Art. 5](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) |
| Is it high-risk as a product or safety component under Annex I (e.g. a medical device)? | No. A medical device is something the manufacturer intends for medical purposes such as diagnosis, prevention, monitoring or treatment of disease ([MDR Art. 2(1)](https://eur-lex.europa.eu/eli/reg/2017/745/oj/eng)). The workflow sorts correspondence and copies approved product texts; it makes no diagnostic or treatment decision about a patient. | [AI Act Art. 6(1)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) |
| Is it in an Annex III high-risk area? | No. The closest entry is "emergency healthcare patient triage systems" under access to essential services (Annex III point 5(d)). Enquiry routing is not patient triage, is not emergency care, and does not decide access to a service. | [Art. 6(2) and Annex III](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) |
| Even if a regulator saw it as close to Annex III? | The derogation for systems that perform a narrow procedural or preparatory task without materially influencing a decision would likely apply; the provider would have to document that assessment. | [Art. 6(3)–(4)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) |
| Transparency duty for systems that interact directly with people? | Not triggered: requesters never interact with the AI; they receive a reply written from approved text and sent by a person. Voluntary disclosure is still recommended (privacy notice). | [Art. 50(1)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng) |

**Conclusion (for Legal to confirm):** a limited-purpose, non-high-risk AI system. The high-risk obligations do not apply, but the project applies their core ideas voluntarily (human oversight, logging, testing), because the business risk — a missed adverse event — is high.

### 2.3 Obligations that do apply

- **AI literacy.** Providers and deployers must take measures to support the AI literacy of staff who operate AI systems; the July 2026 amendment rewrote the original duty in this way ([AI Act Art. 4](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng), as replaced by [Regulation (EU) 2026/1744, Art. 1(5)](https://eur-lex.europa.eu/eli/reg/2026/1744/oj/eng)). Covered by the training plan in [06](06_training_and_handover.md).
- **Timeline context.** The AI Act's general provisions (Chapters I–II) have applied since 2 February 2025 ([Art. 113](https://eur-lex.europa.eu/eli/reg/2024/1689/oj/eng)). The AI Omnibus moves the application of high-risk obligations to 2 December 2027 for Annex III systems and 2 August 2028 for Annex I systems ([Regulation (EU) 2026/1744, recital 40](https://eur-lex.europa.eu/eli/reg/2026/1744/oj/eng)). This matters if the classification above is ever revisited.

### 2.4 Voluntary controls mirroring high-risk requirements

| Idea | How the solution does it |
|---|---|
| Human oversight | Every draft needs approval by a named, logged-in reviewer; AE, complaint and off-label enquiries never get a draft |
| Logging | Every routing and review decision in `PIC Audit Log` with actor, time, decision, source ID + version, workflow/rule/prompt versions |
| Accuracy and robustness testing | `PIC Eval` on 26 cases; must pass before any change goes live |
| Fail-safe | AI error or invalid output → human queue |

This matches EMA's view that AI used in the medicinal product lifecycle should follow a risk-based approach with human oversight, and that the marketing authorisation holder stays responsible for validating and monitoring AI used in pharmacovigilance ([EMA AI reflection paper](https://www.ema.europa.eu/en/documents/scientific-guideline/reflection-paper-use-artificial-intelligence-ai-medicinal-product-lifecycle_en.pdf)).

---

## 3. Pharmacovigilance requirements (general level)

| Requirement | What it means for this workflow | Source |
|---|---|---|
| Reports received through **medical enquiry or product information services** are treated as **spontaneous reports** | Every MI enquiry must be screened for possible adverse reactions — this is why AE detection is the workflow's first job | [GVP VI.B.1.1.1](https://www.ema.europa.eu/en/documents/regulatory-procedural-guideline/guideline-good-pharmacovigilance-practices-gvp-module-vi-collection-management-submission-reports_en.pdf) |
| A valid case needs four minimum elements: identifiable reporter, identifiable patient, suspected product, suspected reaction | Drug Safety, not the workflow, decides validity; the workflow forwards anything that **might** become a case | [GVP VI.B.2](https://www.ema.europa.eu/en/documents/regulatory-procedural-guideline/guideline-good-pharmacovigilance-practices-gvp-module-vi-collection-management-submission-reports_en.pdf) |
| **Day zero** is when the minimum information reaches any company personnel, including contractors | The clock starts at receipt in MI, not when Drug Safety notices it — so routing must be immediate; every enquiry is time-stamped at intake | [GVP VI.B.7](https://www.ema.europa.eu/en/documents/regulatory-procedural-guideline/guideline-good-pharmacovigilance-practices-gvp-module-vi-collection-management-submission-reports_en.pdf) |
| Serious cases are submitted within 15 days, non-serious within 90 days of receipt | A case sitting for days in an ordinary queue eats into this window — the core business problem in [01](01_discovery.md) | [GVP VI.C.3](https://www.ema.europa.eu/en/documents/regulatory-procedural-guideline/guideline-good-pharmacovigilance-practices-gvp-module-vi-collection-management-submission-reports_en.pdf), [Directive 2001/83/EC Art. 107(3)](https://eur-lex.europa.eu/eli/dir/2001/83/oj/eng) |
| Special situations (pregnancy exposure, overdose, medication error, misuse, lack of efficacy) have their own handling rules | The workflow routes them to Drug Safety even without symptoms; Drug Safety applies the rules on whether and how they are reported | [GVP VI.B.6](https://www.ema.europa.eu/en/documents/regulatory-procedural-guideline/guideline-good-pharmacovigilance-practices-gvp-module-vi-collection-management-submission-reports_en.pdf) |

---

## 4. Open decisions for the client

1. DPO: confirm Article 6 and 9 bases per country; decide whether a DPIA is required (likely yes).
2. Procurement: processor agreement with the AI provider (no training on data, retention, sub-processors, data location, transfer mechanism).
3. Drug Safety: approve the AE keyword rules and the routing of special situations; set retention for safety records.
4. Legal: confirm the AI Act classification and record the reasoning in section 2.2.
