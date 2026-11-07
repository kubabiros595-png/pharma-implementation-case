# 06 — Training, go-live and handover

> Fictional case (Nordvale Pharma). Durations, dates and targets are planning assumptions for a portfolio exercise.

## 1. Training plan

| Audience | Format | Length | Content | Proof of competence |
|---|---|---|---|---|
| MI specialists (5) | Live session + hands-on in the test environment | 2 × 60 min | Queues and reasons; reviewing a draft; the edit rules; when to reject and forward to Drug Safety; what never to do ([05](05_user_guide.md)) | Review 10 test drafts, including 2 hidden-AE traps and 1 edit that must be blocked; 10/10 correct |
| MI coordinator (1) | Live session | 60 min | Monitoring queues; `mi_human` reasons; reporting issues to IT | Walk through 5 queue rows and name the correct action |
| Senior MI reviewer | 1:1 session | 60 min | Off-label queue; owning the response library; versioning and retiring responses | Add a new response version and retire the old one in the test environment |
| Drug Safety officers | 1:1 session | 60 min | Safety queue; false alarms vs misses; day zero from intake time; owning the keyword rules | Classify 7 flagged test cases; propose one rule change through the change process |
| Head of MI, DPO | Briefing | 30 min | What the AI does and does not do; audit log; risks and residual risks ([03](03_risk_assessment.md)) | Sign-off on go-live checklist |
| All users | Short e-learning | 15 min | AI literacy basics: what a language model is, why it can be wrong, why a person decides (supports the AI literacy duty in [04 §2.3](04_compliance.md)) | Quiz, 80% pass mark |

Materials: [05_user_guide.md](05_user_guide.md), a one-page "stop and forward to Drug Safety" card, recorded walkthrough of the review form, the test environment with fictional enquiries.

## 2. Go-live checklist

All items must be checked and signed before go-live.

**Compliance and contracts**
- [ ] DPO confirmed legal bases and DPIA outcome ([04 §1](04_compliance.md))
- [ ] Processor agreement with the AI provider signed: no training on data, retention, sub-processors, data location, transfer mechanism
- [ ] Privacy notice on the enquiry form updated (AI-assisted sorting, human review)
- [ ] Legal confirmed the AI Act classification ([04 §2.2](04_compliance.md))

**Content and rules**
- [ ] Response library reviewed and approved by the senior MI reviewer; every entry has an approver and date
- [ ] AE keyword rules approved by Drug Safety, validated on a de-identified historical sample
- [ ] Product label summary in the prompt matches the current SmPCs

**Technical**
- [ ] `PIC Eval` passed on the production configuration: 100% AE recall, ≥ 85% classification, 100% masking, 100% valid citations, 0 drafts for restricted categories, 100% audit coverage
- [ ] Review form requires login; only MI staff can open it
- [ ] Intake table access limited to MI staff and Drug Safety
- [ ] Workflow exported and version-tagged; no credentials in exports
- [ ] Fail-safe tested: AI switched off → enquiries land in `mi_human`, AEs still reach Drug Safety
- [ ] Error notifications go to the IT owner

**People**
- [ ] All users trained and competence checks passed
- [ ] Hypercare rota agreed (see below)
- [ ] Rollback plan agreed: switch the form back to the shared mailbox; queues stay readable

## 3. Hypercare (first 4 weeks after go-live)

| Week | Activity |
|---|---|
| 1 | **Daily** 15-minute check: queue sizes, every `mi_human` reason, every Drug Safety item flagged by only one detector. Drug Safety reviews a sample of **non-flagged** enquiries daily to look for misses |
| 2 | Same as week 1; first tuning proposals (rules, library gaps) collected but **not** changed without the change process |
| 3–4 | Check every two days; weekly report to the Head of MI |
| End of week 4 | Hypercare review: metrics below, open issues, decision to hand over |

**Issue priorities during hypercare**

| Priority | Example | Response |
|---|---|---|
| P1 | A possible AE was not routed to Drug Safety | Same day: Drug Safety handles the case; rule fix proposed; root cause written up |
| P2 | Personal data seen in the AI text or a queue | Same day: DPO informed; masking fix |
| P3 | Wrong draft source, blocked edits that should pass, many `mi_human` items | Within 3 working days |

## 4. Success metrics

| Metric | Baseline (fictional) | Target | Measured from |
|---|---|---|---|
| Possible AEs reaching Drug Safety within 1 working day | 78% | 100% | Queue timestamps vs intake time |
| Missed AEs found in Drug Safety's sample review | 4 per 110 AE cases | 0 | Daily sample review |
| Median time to first response | 3.2 days | ≤ 1 day | Intake time → reply sent |
| Drafts approved without edits | — | ≥ 70% | Audit log |
| Replies citing an outdated response version | 22% | 0% | Audit log source versions |
| Places storing requester personal data per enquiry | 4 | 1 | Data flow review |
| `mi_human` items caused by AI errors | — | < 2% | Queue reasons |

## 5. Handover to the support team

### 5.1 Who owns what

| Area | Responsible | Accountable | Consulted |
|---|---|---|---|
| Workflow running, errors, exports | IT / automation owner | Head of MI | — |
| Response library | Senior MI reviewer | Head of MI | Regulatory |
| AE keyword rules | Drug Safety officer | Head of Drug Safety | IT owner |
| Prompt and model version | IT owner | Head of MI | Drug Safety, DPO |
| Personal data, retention, provider contract | DPO | Head of MI | IT owner |

### 5.2 Change process (any change to rules, prompt, model, library or workflow)

1. Request with reason and owner approval.
2. Change made in the test copy, version label increased.
3. Run `PIC Eval`. It must pass. Add a new test case for every real-world miss or false alarm that triggered the change.
4. Record the result row from `PIC Test Runs` in the change ticket.
5. Deploy, and note the new version labels (they appear in every new audit row).

Model provider updates count as changes: re-run `PIC Eval` before switching models, and at least once a month during operation.

### 5.3 Runbook (short)

| Symptom | Check | Action |
|---|---|---|
| Many `ai_unavailable` reasons | AI provider status, credentials | Wait or switch provider credential; enquiries are safe in `mi_human` |
| Many `ai_invalid_output` reasons | Recent prompt or model change | Roll back to the previous version; run `PIC Eval` |
| `pii_check_failed` on ordinary enquiries | Masked text in the queue row | Add a masking pattern; re-test |
| Draft cites a retired response | Library statuses | Should be impossible — treat as P1 and stop drafts until fixed |
| Review form shows nothing | `PIC Queue` rows with `pending_review` | Normal if there are no drafts |

### 5.4 Handover sign-off

The support team accepts handover when: hypercare metrics meet target for two consecutive weeks, no open P1/P2 issues, the runbook has been walked through once with the IT owner, and the change process has been used successfully at least once.
