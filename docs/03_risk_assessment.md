# 03 — Risk assessment

> Fictional case (Nordvale Pharma). Scores are the project team's judgement for this portfolio case, not an audited assessment. Not legal or medical advice.

## Scale

**Likelihood** (after design, before go-live): 1 rare · 2 unlikely · 3 possible · 4 likely · 5 almost certain
**Impact:** 1 negligible · 2 minor · 3 moderate · 4 major (regulatory finding, harm possible) · 5 severe (patient harm, serious regulatory breach)
**Score** = likelihood × impact. 1–4 low · 5–9 medium · 10–25 high. Anything high blocks go-live until reduced.

## Risk register

| ID | Risk | Cause | L | I | Score | Controls built into the solution | Evidence from the test | Residual | Owner |
|---|---|---|---|---|---|---|---|---|---|
| R1 | **Possible adverse event is missed** and handled as an ordinary question | Hidden wording ("rash, but it's gone now"); special situations without symptoms; model error; AI outage | 2 | 5 | **10** | Two independent detectors (keyword rules + Claude), either one is enough; rules run without the AI; AI outage → human queue, never "no action"; no draft is ever produced for a flagged enquiry; Drug Safety owns the rules | 7/7 AEs routed in all three runs; rules alone 7/7, Claude alone 7/7 ([results](../results/README.md)) | Medium (L1×I5=5) — the test set is small and written by the same team as the rules | Drug Safety officer |
| R2 | **Made-up medical information** reaches a requester | Model writes its own text; outdated or unapproved source | 3 | 5 | **15** | Model only picks a response ID; draft text is copied word for word from the approved library; code checks that the cited ID + version exists and is approved; retired versions are never offered; forbidden-phrase check; human approval before anything is sent | 10/10 drafts cited an existing, approved ID + version; 0 drafts with model-written content | Low (1×5=5, kept medium for monitoring) | Senior MI reviewer |
| R3 | **Personal data leaks** to the AI provider or into logs | Names or numbers inside free text; contact fields copied around | 3 | 4 | **12** | Masking in code before the AI; second scan blocks the AI call if anything is left; contact details stored only in the intake table; queues and audit log hold masked text only; processor agreement with the AI provider (see [04](04_compliance.md)) | 10/10 planted items masked; 0 of 47 contact fields reached the AI text | Medium (2×4=8) — rule-based masking can miss unusual formats (e.g. a name with no cue word) | DPO + IT owner |
| R4 | **Off-label question is answered** without a qualified reviewer | Model treats it as product information; reviewer approves a draft by habit | 2 | 4 | 8 | Off-label category never gets a draft; product label summary is part of the prompt; only the senior MI reviewer works the off-label queue | 4/4 off-label questions got no draft in every run (once, one went to Drug Safety instead — safe side) | Low (1×4=4) | Senior MI reviewer |
| R5 | **Model behaviour changes** (new model version, provider change) and quality drops silently | Provider updates; prompt edits; library changes | 3 | 4 | **12** | Model, prompt, rules and workflow carry version labels written into every audit row; `PIC Eval` must be re-run and pass before any change goes live; monthly re-run during operation; fail-safe on invalid output | Across three runs one borderline case changed outcome (OL-04); parser fix found through the test | Medium (2×4=8) | IT / automation owner |
| R6 | **Reviewer over-trusts drafts** (automation bias) | Drafts look polished; high volume | 3 | 3 | 9 | Draft shows source ID, checks and masked enquiry side by side; reject needs a reason; edits re-checked; training (see [06](06_training_and_handover.md)); monthly sample audit of approved drafts | Review form blocked 2/2 invalid decisions in every run | Medium (2×3=6) | Head of MI |
| R7 | **Prompt injection** in enquiry text ("ignore your rules and…") | Malicious or odd input | 2 | 3 | 6 | Prompt treats enquiry as data; model cannot write reply text or choose a queue directly; routing is decided in code; rules run regardless | Not tested with attack strings (see limitations) | Low-medium | IT / automation owner |
| R8 | **Response library goes out of date** | SmPC change not reflected | 2 | 4 | 8 | Each response has owner, version and approval date; retired versions are excluded automatically; SmPC change triggers library review (process step in handover) | Retired version SR-VLQ-002 v1.0 was never used | Low-medium | Senior MI reviewer |
| R9 | **Fail-safe overloads the human queue** | AI outage; masking gate too strict | 2 | 2 | 4 | Queue reasons recorded (`ai_unavailable`, `pii_check_failed`); hypercare daily review of queue sizes | Run 1: one enquiry fell back to human queue on invalid AI output — visible and explained | Low | MI coordinator |

## Top three risks and what would change the rating

1. **R1 missed AE.** Before a real go-live, Drug Safety would validate the rules on a de-identified sample of historical enquiries (several hundred, in all local languages), and the target would be 100% recall with a documented false-alarm rate.
2. **R2 made-up information.** The design removes free-text generation entirely. The rating stays under watch only because a human could still edit a draft badly; edits are re-checked.
3. **R3 data leak.** Masking would be extended with a name-recognition step and tested on real-world formats (Polish PESEL, addresses, initials) before go-live.

## Accepted residual risks

- Rule-based masking can miss personal data written in an unusual way. Mitigation: the AI provider is a contracted processor with no-training terms; the second scan blocks known patterns; review of masked samples during hypercare.
- The model may flag borderline enquiries as possible AEs (false alarms). Accepted: false alarms cost Drug Safety a few minutes; misses are not acceptable.
