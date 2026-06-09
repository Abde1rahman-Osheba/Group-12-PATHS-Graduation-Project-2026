# PATHS — Deep Architecture Blueprint
**Personalized AI Talent Hiring System**

> Source of truth: `GRADPROJ.pdf` (86 pages, Jan 2026).
> This blueprint converts the documented vision, ERD, agent map, system analysis, functional/non-functional requirements, and implementation plan into a buildable backend + AI architecture.
> Scope: backend, AI, data, integrations, security, audit. The frontend is intentionally ignored — assume a new frontend is built later against the API surface defined in §17.

---

## 0. Architectural North Star

PATHS is **not** a single ML pipeline. It is a **multi-agent, evidence-driven, human-in-the-loop hiring operating system** with three architectural laws:

1. **Evidence over inference.** Every score, decision, or recommendation must be traceable to a stored evidence record (CV claim, GitHub artifact, assessment answer, interview transcript). No "black-box" outputs.
2. **Anonymize before evaluate, de-anonymize only on outreach approval.** Bias-sensitive attributes are stripped before any scoring or interview agent reads the candidate.
3. **AI proposes, humans dispose.** Every consequential transition (shortlist publish, outreach send, candidate reject, hire) requires a recorded HITL approval.

Everything below is engineered to enforce these three laws.

---

## 1. Core Platform Architecture

### 1.1 Responsibility
Multi-tenant identity, organizations, roles/permissions, and the canonical actor graph that every other module depends on.

### 1.2 Why it exists
Every job, candidate, application, and audit log is owned by an organization and acted on by a typed user. Without a strong RBAC + tenant boundary, fairness, audit, and compliance (Egypt PDPL, GDPR, EEOC) all collapse.

### 1.3 Modules
- `auth` — login, session, password, MFA, SSO hooks.
- `tenancy` — organizations, plans, workspaces.
- `iam` — users, roles, permissions, role-assignments.
- `members` — recruiter / hiring_manager / interviewer / admin / super-admin.
- `candidate-account` (separate trust boundary) — candidate-facing self-service.
- `audit-core` — base audit log writer used by every module.

### 1.4 Database tables (high level)
| Table | Purpose |
|---|---|
| `organizations` | tenant root (name, plan, region, locale, status) |
| `org_settings` | weights, thresholds, sourcing mode defaults, retention policy |
| `users` | id, email, password_hash, status, mfa_secret, last_login |
| `user_profiles` | name, avatar, locale, timezone |
| `roles` | recruiter, hiring_manager, interviewer, admin, super_admin, candidate |
| `permissions` | granular: `job.create`, `shortlist.publish`, `outreach.send`, `decision.finalize`, `candidate.deanonymize`, `audit.read` … |
| `role_permissions` | M:N |
| `org_memberships` | user × organization × role(s) × status |
| `invitations` | pending org invites with expiry |
| `candidates_account` | candidate-side login (separate from `users` for isolation) |
| `consents` | per-candidate consent records (purpose, scope, expiry) |

### 1.5 APIs (representative)
- `POST /auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/mfa/*`
- `POST /orgs`, `GET /orgs/:id`, `PATCH /orgs/:id/settings`
- `POST /orgs/:id/invite`, `POST /invitations/:token/accept`
- `GET /orgs/:id/members`, `PATCH /orgs/:id/members/:uid/roles`
- `GET /me`, `PATCH /me`

### 1.6 Services / business logic
- Tenant resolver middleware (`X-Org-Id` or subdomain → org context).
- RBAC enforcer (decorator/guard) consulting `role_permissions` and per-resource ownership.
- Audit writer that every mutation must call.
- Consent gate used by sourcing/outreach/scoring modules.

### 1.7 AI / agent logic
None directly. This module is the **policy plane** that every agent runs inside.

### 1.8 Inputs
Login credentials, invite tokens, role-change requests, org settings, candidate consents.

### 1.9 Outputs
Authenticated session/JWT carrying `{user_id, org_id, role[], permissions[]}`; audit events for every grant/revoke.

### 1.10 Depends on
Nothing upstream. Everything else depends on this.

### 1.11 Security / privacy / fairness / audit
- Argon2id password hashing, mandatory TLS, JWT short TTL + refresh rotation.
- Hard tenant isolation: every query filters by `org_id`.
- Distinct trust boundary for candidates (separate token audience).
- All role/permission/consent changes are append-only audit events.
- Default-deny RBAC; permissions are explicit.

### 1.12 Build first
Tenancy, users, roles, permissions, org memberships, audit writer, RBAC guard. **Without this, nothing else is safe to build.**

### 1.13 Build later
SSO/SAML, SCIM provisioning, multi-workspace per org, custom permission sets per organization.

---

## 2. Candidate Data Architecture

### 2.1 Responsibility
Ingest fragmented candidate signals (CV, LinkedIn, GitHub, portfolio, ATS export), resolve identities, deduplicate, and produce one **Master Candidate Profile** with provenance and an **Evidence Store**.

### 2.2 Why it exists
The documentation's #1 problem statement is data fragmentation: 4–6 disconnected tools per organization. A unified, evidence-backed profile is the precondition for fair scoring and explainable decisions (FR §3.2.2.1.3, NFR data-integrity).

### 2.3 Modules
- `ingestion` — file uploads, URL imports, ATS connectors, CSV import.
- `parsing` — CV/PDF parser, LinkedIn parser, GitHub fetcher, portfolio scraper.
- `normalization` — title canonicalization, skill canonicalization, experience banding, date harmonization.
- `identity-resolution` — deterministic + probabilistic candidate matching.
- `dedupe` — merge/split workflows with HITL conflict resolution.
- `master-profile` — composer that builds the canonical view.
- `evidence-store` — append-only evidence items with source + timestamp + confidence.
- `provenance` — per-field source map (which source produced which field, when).

### 2.4 Database tables
| Table | Purpose |
|---|---|
| `candidates` | canonical id, status, candidate_type (active/passive), experience_years, anonymized_alias |
| `candidate_identities` | external ids per source (linkedin_url, github_login, ats_id, email_hash, phone_hash) |
| `candidate_sources` | each raw payload pulled (source, url, fetched_at, raw_blob_uri) |
| `candidate_profile_fields` | normalized field × value × source × confidence × verified_at |
| `candidate_skills` | candidate × skill × proficiency × evidence_id × last_verified |
| `evidence_items` | id, candidate_id, type (cv_claim/github_repo/portfolio_artifact/assessment/interview), source_uri, extracted_text, confidence, ts |
| `dedupe_candidates` | proposed merges (a, b, score, status, reviewer_id) |
| `merge_history` | who merged what, when, why (audit) |
| `consents` | reused from §1 |

### 2.5 APIs
- `POST /candidates/import/cv` (multipart)
- `POST /candidates/import/linkedin` `{url}`
- `POST /candidates/import/github` `{login}`
- `POST /candidates/import/portfolio` `{url}`
- `POST /candidates/import/ats` (CSV/JSON batch)
- `GET /candidates/:id` (master profile)
- `GET /candidates/:id/evidence`
- `GET /candidates/:id/sources`
- `POST /candidates/dedupe/suggest`
- `POST /candidates/:a/merge/:b` (HITL)
- `POST /candidates/:id/split` (HITL)

### 2.6 Services / business logic
- Async parsing pipeline: upload → queue → parse → normalize → evidence write → identity resolve → upsert master.
- Identity resolver: deterministic keys (email, phone, linkedin_url, github_login) → probabilistic (name + last employer + city) → ML similarity over embeddings.
- Conflict policy: `latest-wins` for volatile fields (title), `union` for skills, `highest-confidence-wins` for contradictory facts; conflicts above a threshold queued for HITL.
- Provenance writer on every field write.

### 2.7 AI / agent logic
- **Candidate Profile Agent** (composer + parser orchestrator).
- **Identity Resolution Agent** (matching + conflict triage).
- LLM use is bounded to extraction and normalization, never to inventing facts. All extracted claims become `evidence_items` with `confidence`.

### 2.8 Inputs
Raw files, URLs, ATS exports, scrape results from the Sourcing Agent (§4).

### 2.9 Outputs
Master Candidate Profile, Evidence Store entries, dedupe proposals, identity graph nodes.

### 2.10 Depends on
§1 (tenancy/RBAC), §10 (vector store for similarity), §14 (audit/consent).

### 2.11 Security / privacy / fairness / audit
- Raw blobs stored encrypted with KMS; access logged.
- PII (email/phone) hashed for matching, plaintext only for outreach after approval.
- Right-to-erasure path: candidate id can be hard-deleted with cascade record.
- Every merge/split is auditable and reversible.

### 2.12 Build first
CV upload → parse → normalize → master profile → evidence items, then identity resolution and dedupe.

### 2.13 Build later
Live LinkedIn scraping at scale, ATS deep integrations, portfolio JS-rendering scrapers, real-time sync from external sources.

---

## 3. Job and Requirement Architecture

### 3.1 Responsibility
Capture structured job definitions (title, level, location, mode, skills, criteria, rubric, pipeline stages) so every downstream agent operates on the same source of truth.

### 3.2 Why it exists
ERD §3.4 makes Job the second-most central entity after Candidate. Loose job definitions are the root cause of inconsistent evaluation (literature review §2.2.2).

### 3.3 Modules
- `jobs` — CRUD, lifecycle (draft → published → closed).
- `job-jd-parser` — turn pasted/uploaded JD into structured fields.
- `job-skills` — required vs preferred, weights, must-have flags.
- `job-rubric` — scoring dimensions, weights, thresholds, Top-K rule.
- `pipeline-stages` — configurable per job (sourced → screened → assessed → hr → tech → decision → hired).
- `job-criteria` — non-skill criteria (location, comp, work mode, language).

### 3.4 Database tables
| Table | Purpose |
|---|---|
| `jobs` | id, org_id, title, level, mode (inbound/outbound), status, salary_min/max, location, work_mode, headcount, opened_at, closed_at, owner_user_id |
| `job_descriptions` | job_id, raw_text, structured_json, version |
| `skills` | canonical skills dictionary (name, type, parent, embedding) |
| `job_skills` | job × skill × required(bool) × weight × min_proficiency |
| `job_criteria` | job × key × value × is_hard_gate |
| `job_rubric` | job × dimension × weight × threshold |
| `job_pipeline_stages` | job × stage × order × hitl_required |
| `job_collaborators` | users on a job (recruiter, hiring_manager, interviewers) |
| `jobs_history` | versioned snapshots |

### 3.5 APIs
- `POST /jobs`, `GET /jobs`, `GET /jobs/:id`, `PATCH /jobs/:id`, `POST /jobs/:id/publish`, `POST /jobs/:id/close`
- `POST /jobs/:id/jd/parse` (extract structured)
- `POST /jobs/:id/skills` / `PATCH` / `DELETE`
- `POST /jobs/:id/rubric`
- `POST /jobs/:id/pipeline`
- `POST /jobs/:id/collaborators`

### 3.6 Services / business logic
- JD parser (LLM + skill canonicalizer) emits proposed `job_skills` + criteria; recruiter must confirm.
- Rubric validator: weights sum to 1.0 ± tolerance; thresholds within bounds.
- Pipeline templates per role family (technical / non-technical / volume).
- Versioning: every published change snapshots and rescoring is offered (HITL).

### 3.7 AI / agent logic
- **JD Parsing Agent** (subroutine of Candidate Profile / Sourcing setup): extracts skills, level, criteria from free-text JD. Output is *proposal*, not auto-applied.

### 3.8 Inputs
Recruiter form, pasted JD, JD upload, role template.

### 3.9 Outputs
Structured Job, Job Skills, Rubric, Pipeline definition.

### 3.10 Depends on
§1, skills dictionary in §10.

### 3.11 Security / privacy / fairness / audit
- Hard gates flagged separately from soft criteria so the bias-guardrail can audit them.
- Any criterion that could leak protected proxies (e.g., location-only, school-only) flagged with a fairness warning.
- Job version history is append-only.

### 3.12 Build first
Jobs CRUD, skills dictionary, job_skills, rubric, pipeline stages.

### 3.13 Build later
JD parser, role templates marketplace, multilingual JD support.

---

## 4. Sourcing Architecture

### 4.1 Responsibility
Fill jobs with candidates via two modes: **Inbound** (org provides applicants/imports) and **Outbound** (PATHS searches and discovers passive candidates).

### 4.2 Why it exists
Doc §3.1 explicitly defines the two-mode contract. Outbound is what differentiates PATHS from a plain ATS.

### 4.3 Modules
- `inbound-applications` — candidate self-apply, recruiter manual add, ATS import to a job.
- `outbound-search` — natural-language and structured queries against external sources.
- `talent-pools` — saved cohorts of candidates per org.
- `saved-searches` — reusable parameterized queries.
- `sourcing-history` — every search, who ran it, what it returned, who was added.
- `passive-discovery` — channel adapters (LinkedIn/GitHub/portfolio/ATS).

### 4.4 Database tables
| Table | Purpose |
|---|---|
| `applications` | candidate × job × status × source_platform × shortlist_rank × apply_date |
| `talent_pools` | id, org_id, name, criteria_json |
| `talent_pool_members` | pool × candidate × added_by × added_at |
| `saved_searches` | id, org_id, query_dsl, last_run_at |
| `sourcing_runs` | run_id, job_id, query, source, count, started/finished, status |
| `sourcing_results` | run_id × candidate_id × raw_score × added(bool) |
| `source_channels` | per-org configured channels and credentials |

### 4.5 APIs
- `POST /jobs/:id/applications` (manual add)
- `POST /candidates/apply` (candidate self-service)
- `POST /jobs/:id/sourcing/runs` `{query, sources[]}`
- `GET /sourcing/runs/:id`
- `POST /talent-pools`, `POST /talent-pools/:id/add`
- `POST /saved-searches`
- `GET /jobs/:id/sourcing/history`

### 4.6 Services
- Query DSL → adapter dispatch → dedupe-on-ingest → enrich-light → push to candidate ingestion (§2).
- Rate-limit + scraping etiquette (NFR security).
- "Outbound mode skips outreach gating? No." — sourcing always feeds the screening pipeline; outreach is its own gated stage (§5/§6).

### 4.7 AI / agent logic
- **Sourcing Agent** (a.k.a. Scraping & Searching Agent) — translates job spec + recruiter NL query into structured searches per channel; returns ranked candidate set with raw signals.

### 4.8 Inputs
Job + rubric + recruiter NL query + sources + filters.

### 4.9 Outputs
`applications` rows in `sourced` status, evidence references, sourcing run report.

### 4.10 Depends on
§2 (ingestion), §3 (job spec), §10 (skills/embeddings), §14 (audit).

### 4.11 Security / privacy / fairness / audit
- Respect `robots.txt` and channel ToS; per-org scraping limits.
- Sourced candidates flagged `passive` with no consent yet; cannot be outreached until §5/§6 approval.
- Filters that imply protected attributes (age proxies, gender, ethnicity) are blocked at the API layer with explainable error.

### 4.12 Build first
Inbound apply + manual add + ATS CSV import. Then one outbound channel (e.g., GitHub for engineering).

### 4.13 Build later
Multi-channel outbound (LinkedIn, niche boards), saved-search alerts, market-insights dashboards.

---

## 5. Screening and Matching Architecture

### 5.1 Responsibility
Score candidates against a job's rubric using verified evidence, produce a Top-K shortlist with **confidence**, **rationale**, and **explainable rank**, gated by a HITL approval before anyone is contacted or rejected.

### 5.2 Why it exists
Figure 3.7 of the doc — the heart of PATHS. Differentiator vs. ATS competitors (Greenhouse, Ashby) is **evidence-gated, explainable** ranking, not keyword overlap.

### 5.3 Modules
- `match-engine` — sub-score computation per dimension.
- `evidence-gate` — Sufficient Evidence Gate (penalize thin signals).
- `aggregator` — weighted combination per rubric.
- `constraints` — must-have hard fails.
- `fairness-rerank` — re-rank under fairness constraints.
- `confidence-calibration` — combine score with evidence sufficiency.
- `explanation-packet` — generates per-candidate rationale.
- `topk-publisher` — produces shortlist; requires HITL approve.

### 5.4 Database tables
| Table | Purpose |
|---|---|
| `match_runs` | run_id, job_id, rubric_version, started_by, status |
| `match_scores` | run × candidate × dimension × raw × weighted × evidence_count × confidence |
| `match_aggregate` | run × candidate × final_score × confidence × rank |
| `match_constraints_log` | run × candidate × rule × pass/fail × reason |
| `match_explanations` | run × candidate × packet_json (evidence ids, weights, penalties, flags) |
| `shortlists` | id, job_id, run_id, status (proposed/approved/rejected), approved_by, approved_at |
| `shortlist_items` | shortlist × candidate × position × note |

### 5.5 APIs
- `POST /jobs/:id/match/runs` `{rubric_version?}`
- `GET /match/runs/:id`
- `GET /match/runs/:id/candidates/:cid` (full explanation packet)
- `POST /jobs/:id/shortlists/propose`
- `POST /shortlists/:id/approve` (HITL)
- `POST /shortlists/:id/reject`
- `PATCH /shortlists/:id/items/:cid` (manual reorder/remove with reason)

### 5.6 Services
- Sub-scoring components per dimension: skill match, experience fit, project/portfolio, assessment, interview, culture/preference fit (Fig 3.7).
- Hard gates (must-have skills, work auth, location if hard) → exclude.
- Fairness constraint solver (selection-rate parity by group with merit preserved as much as possible).
- Confidence = f(score_signal_strength, evidence_count, source_diversity, recency).
- Explanation packet generator stamps **(a) evidence used, (b) sub-score breakdown, (c) penalties, (d) bias flags, (e) human-readable summary** — exactly the doc's spec.

### 5.7 AI / agent logic
- **Screening Agent** — orchestrates sub-scorers, calls LLM only for natural-language rationale; the **numbers come from deterministic scorers**, not from the LLM.

### 5.8 Inputs
Job + rubric + master profiles + evidence items + bias-guardrail-anonymized views.

### 5.9 Outputs
Match scores, shortlist proposal, explanation packets.

### 5.10 Depends on
§2, §3, §6 (anonymization), §10 (embeddings/RAG for skill semantic match), §14 (audit).

### 5.11 Security / privacy / fairness / audit
- Scoring runs only on anonymized projections (§6).
- Every score change writes to `match_runs` history.
- Fairness re-rank metrics persisted for audit.
- LLM rationale prompts logged with full input snapshot for replayability.

### 5.12 Build first
Skill match + experience fit + must-have gates + Top-K + explanation packet (rule-based).

### 5.13 Build later
Fairness-aware re-ranking solver, full multi-signal aggregation, model-based confidence calibration, what-if rubric simulators.

---

## 6. Bias and Fairness Architecture

### 6.1 Responsibility
Enforce **anonymization-before-evaluation**, separate **evidence from inference**, and provide fairness checks + audit trail + HITL approval throughout.

### 6.2 Why it exists
NFR §3.2.2.2.2; Egypt PDPL + EU AI Act + EEOC compliance (§2.2 doc). Without this layer the entire system is legally and ethically unshippable.

### 6.3 Modules
- `anonymizer` — produces a **Anonymized Candidate View** projection.
- `proxy-leakage-detector` — catches leakage in JD criteria, evidence extracts, prompts.
- `evidence-vs-inference-tagger` — every agent output must mark each statement as fact/inference.
- `fairness-monitor` — selection-rate, four-fifths rule, group disparity dashboards.
- `hitl-gates` — registry of approval checkpoints across the pipeline.
- `bias-audit-log` — append-only, separate from operational audit.
- `de-anonymization-service` — guarded reveal at outreach approval, logged.

### 6.4 Database tables
| Table | Purpose |
|---|---|
| `anonymized_views` | candidate × view_version × projected_json (no PII) |
| `bias_flags` | id, scope (job/run/candidate), rule, severity, status |
| `fairness_metrics` | scope × metric × group × value × period |
| `hitl_approvals` | id, action_type, target_id, requested_by, approver_id, decision, reason, ts |
| `de_anon_events` | candidate × requested_by × approver × purpose × granted_at |
| `bias_audit` | append-only forensic log |

### 6.5 APIs
- `GET /candidates/:id/anonymized?for=screening|interview`
- `POST /hitl/approvals/:id/decide`
- `GET /fairness/metrics?job_id=...`
- `POST /candidates/:id/deanonymize` `{purpose}` (gated)
- `GET /bias/flags?org_id=...`

### 6.6 Services
- Anonymizer redacts: name, email, phone, address, photo, age proxies, gender markers, ethnicity proxies, university name (configurable), employer names (optional setting).
- Proxy leakage detection on JD/criteria/prompts/outputs.
- Selection-rate and disparate-impact monitors per protected dimension (where allowed).
- Approval bus: any module can submit a request; recruiter UI later consumes it.

### 6.7 AI / agent logic
- **Bias Guardrail Agent** — runs at three points: (1) before scoring, (2) before any AI message generation, (3) before final decision, blocking unsafe transitions.

### 6.8 Inputs
Master profile, JD criteria, agent outputs, scoring runs.

### 6.9 Outputs
Anonymized views, flags, fairness metrics, approval requests/decisions, de-anon events.

### 6.10 Depends on
§1, §2, §3, §14.

### 6.11 Security / privacy / fairness / audit
- Anonymized view is its own DB column — never derived ad-hoc.
- HITL approvals are non-bypassable in code paths, not just UI.
- Bias audit log is write-once (immutable bucket / append-only DB partition).

### 6.12 Build first
Anonymizer + HITL approval bus + de-anonymization gating + base bias audit table.

### 6.13 Build later
Live fairness monitoring dashboards, automated proxy detection on free-text fields, calibration audits per model version.

---

## 7. Technical Assessment Architecture

### 7.1 Responsibility
Generate, deliver, and grade role-relevant assessments that act as a **gate before the technical interview** (Figure 3.8), with stored evidence and explainable pass/fail.

### 7.2 Why it exists
Documentation makes assessment a mandatory pre-interview filter for technical roles, with HITL approval to continue/stop.

### 7.3 Modules
- `assessment-templates` — coding/MCQ/case/take-home templates per role.
- `assessment-generator` — RAG-grounded, rubric-aligned generation.
- `assessment-delivery` — secure delivery to candidate, time-boxing, anti-cheating signals.
- `assessment-submission` — capture answers + artifacts.
- `auto-grader` — coding tests, unit tests, plagiarism check.
- `manual-grader` — interviewer scoring with rubric.
- `assessment-evidence` — stored answers + grader notes + AI rationale → `evidence_items`.

### 7.4 Database tables
| Table | Purpose |
|---|---|
| `assessment_templates` | id, type, role_family, rubric_id, content_json |
| `assessments` | id, application_id, template_id, status, due_at, generated_by(ai/human) |
| `assessment_questions` | assessment × question × rubric_dimension × weight |
| `assessment_submissions` | assessment × answer_json × started/finished_at × signals_json |
| `assessment_grades` | submission × grader(ai/human) × dimension × score × rationale |
| `assessment_artifacts` | code/files/links uploaded |

### 7.5 APIs
- `POST /assessments/templates`
- `POST /applications/:id/assessments` (generate)
- `GET /assessments/:id` (recruiter view)
- `GET /candidate/assessments/:token` (candidate-side)
- `POST /candidate/assessments/:token/submit`
- `POST /assessments/:id/grade` (auto + manual merge)
- `POST /assessments/:id/decision` `{pass|fail, note}` (HITL)

### 7.6 Services
- Template-driven generation; banks per skill; difficulty calibration.
- Coding sandbox runner (isolated, time + memory limits, language matrix).
- Plagiarism / similarity detection.
- Grader merges deterministic auto-scoring with optional human review.
- Pass/fail rationale generated **after** scores are computed; never the other way around.

### 7.7 AI / agent logic
- **Assessment Agent** — generates questions grounded in (job rubric + org KB + role best practice corpus); writes pass/fail rationale from concrete evidence.

### 7.8 Inputs
Job rubric, role family, candidate skill claims, anonymized identity.

### 7.9 Outputs
Assessment instance, submission, grades, evidence items, gating decision.

### 7.10 Depends on
§3 (rubric), §6 (anonymization), §10 (RAG for question grounding), §14 (audit).

### 7.11 Security / privacy / fairness / audit
- Assessment runs in sandboxed environments.
- Anti-cheating signals are advisory, not auto-disqualifying (HITL).
- Identical rubric and time across candidates for the same job (consistency).

### 7.12 Build first
MCQ + simple coding template, manual grade flow, pass/fail HITL gate.

### 7.13 Build later
Auto-graded multi-language sandbox, plagiarism, take-home with reviewer pool, AI-generated case studies.

---

## 8. Interview Architecture

### 8.1 Responsibility
Run **structured, rubric-aligned, RAG-grounded** interviews (HR + Technical) with scheduling, transcript capture, scorecards, and HITL summary review.

### 8.2 Why it exists
Literature review is unanimous: structured > unstructured. RAG keeps questions aligned to the org's policies and the role's evidence (Figure 3.8).

### 8.3 Modules
- `scheduler` — calendar sync, candidate self-select, timezone, reminders, reschedule.
- `interview-templates` — per role/stage; HR vs Technical.
- `question-generator` — Technical RAG Questions Agent + HR RAG Agent.
- `interview-rooms` — join links, recording consent.
- `transcript` — STT, speaker diarization, redaction.
- `notes` — interviewer live notes.
- `scorecards` — rubric-aligned scoring.
- `summarizer` — key points, strengths, weaknesses.

### 8.4 Database tables
| Table | Purpose |
|---|---|
| `interviews` | id, application_id, type (hr/tech/non-tech), stage, status, scheduled_at, ended_at, mode (onsite/video) |
| `interview_invitees` | interview × user × role (interviewer/observer) |
| `interview_slots` | proposed slots per interview |
| `interview_questions` | interview × question × rubric_dimension × source (rag/template) |
| `interview_transcripts` | interview × segment × speaker × text × ts |
| `interview_notes` | interview × user × text × ts |
| `interview_scorecards` | interview × interviewer × dimension × score × rationale |
| `interview_summaries` | interview × summary_json (strengths, weaknesses, evidence_ids) |
| `calendar_links` | per-user OAuth tokens for Google/Outlook |

### 8.5 APIs
- `POST /interviews` (create)
- `POST /interviews/:id/slots` / `POST /candidate/interviews/:token/select-slot`
- `POST /interviews/:id/questions/generate`
- `GET /interviews/:id`
- `POST /interviews/:id/notes`
- `POST /interviews/:id/scorecards`
- `POST /interviews/:id/transcript` (ingest)
- `POST /interviews/:id/summarize`
- `POST /interviews/:id/finalize` (HITL)

### 8.6 Services
- Scheduler: candidate-facing tokenized links, calendar OAuth, conflict resolution.
- RAG question generation grounded in (org KB + job rubric + candidate evidence).
- STT pipeline + redaction (PII, sensitive proxies).
- Summarizer that **cites evidence segment ids** in the summary, not free-text claims.

### 8.7 AI / agent logic
- **HR Interview Agent** (org/policy-grounded RAG questions, behavioral rubric).
- **Technical Interview Agent** (rubric + role + candidate evidence-grounded RAG questions).
- **Summary Agent** (strengths/weaknesses with citations).

### 8.8 Inputs
Job rubric, candidate evidence, org KB, schedule preferences, transcripts.

### 8.9 Outputs
Scheduled interview, questions, transcript, scorecards, summary.

### 8.10 Depends on
§3, §5, §6, §7, §10, §14.

### 8.11 Security / privacy / fairness / audit
- Recording requires explicit candidate consent.
- Same rubric for same role; deviation requires reason.
- Summaries store evidence segment ids → replayable.
- Affective/sentiment analysis is **off by default** per §2.2.2 risk note (ref [61]).

### 8.12 Build first
Manual scheduling, rubric-aligned templates, manual scorecards, manual summary.

### 8.13 Build later
Calendar OAuth, RAG question generation, STT transcripts, AI summary with citations, multi-interviewer aggregation.

---

## 9. Agentic AI Architecture

### 9.1 Orchestration spine
- **LangGraph** for stateful, durable, long-running flows with HITL checkpoints.
- **CrewAI** for multi-agent collaboration patterns inside a phase.
- **LangChain** for tool integrations and retrieval primitives.
- All agents run inside an **Agent Runtime** with: typed I/O schema, tool registry, retry/backoff, timeout, prompt templates versioned in the repo, full prompt+context+output logged to the **Agent Trace Store**.

### 9.2 Agent Roadmap Table
| # | Agent | Purpose | Input | Output | Tools | Data | Trigger | HITL? | Failure handling | Logs/Audit |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Candidate Profile Agent | Build/maintain master profile from raw sources | Raw CV/URLs/ATS row | Master profile + evidence items + provenance | CV parser, GitHub API, LinkedIn parser, portfolio fetcher, embeddings | candidate_sources, evidence_items, candidate_skills | New ingestion event | No (data) | Quarantine raw, mark fields low-confidence | Per-field provenance + agent trace |
| 2 | Sourcing Agent | Find candidates per job in outbound mode | Job + NL query + sources | sourcing_results + ranked candidate refs | Channel adapters, embeddings, dedupe | jobs, sourcing_runs, candidates | Recruiter starts run | Yes — recruiter approves before adding to pool | Per-channel retry, partial results | sourcing_runs + agent trace |
| 3 | Identity Resolution Agent | Merge/split candidate identities | New profile fragment | Merge proposals or auto-merges (if confidence ≥ τ) | Embeddings, deterministic rules | candidate_identities, dedupe_candidates | After ingestion | Yes for borderline | Park as `pending_review` | merge_history |
| 4 | Screening Agent | Score + rank Top-K with explanation | Anonymized profiles + job rubric | match_scores, shortlist proposal, explanation packets | Sub-scorers, embeddings, RAG | match_runs, evidence_items | Recruiter starts run; auto on apply | Yes — shortlist must be approved | Re-run with prior weights, mark inputs missing | match_runs trace |
| 5 | Bias Guardrail Agent | Block unsafe transitions | Any pipeline transition | allow/block + reasons + flags | Anonymizer, proxy detector, fairness solver | bias_flags, fairness_metrics | Pre-score, pre-message, pre-decision | Yes on `block` | Hard fail with reason | bias_audit |
| 6 | Contact Enrichment Agent | Resolve up-to-date contact channels | Candidate id | enriched_contacts + confidence + source | Hunter.io (MCP), email-validator | candidate_profile_fields | Pre-outreach for shortlist | No (data only) | Fall back to alt channel; flag | enrichment events |
| 7 | Outreach Agent | Personalized multi-channel messaging | De-anonymized candidate + job + org KB | Sent message + tracking | Email provider, LinkedIn API, RAG | outreach_messages, outreach_events | Shortlist approved + de-anon granted | Yes — recruiter approves message text | Retry/alt channel; throttle | message + delivery log |
| 8 | Scheduler Agent | Coordinate interviews/assessments | Candidate availability + interviewer calendars | Confirmed interview slot | Google/Outlook OAuth, timezone util | interviews, calendar_links | Candidate replies positively | No for routine; yes for conflicts | Propose alternates | scheduling log |
| 9 | Assessment Agent | Generate + grade assessments | Job rubric + role family + candidate evidence | Assessment instance + grades + rationale | RAG, sandboxed runners, rubric DSL | assessments, assessment_grades | After shortlist approval (technical roles) | Yes — pass/fail confirmation | Allow re-attempt with reason | assessment trace |
| 10 | HR Interview Agent | Generate HR-aligned questions, support evaluation | Org KB + role + candidate evidence | Questions + rubric prompts | RAG, vector store | interview_questions | Before HR interview | Yes — interviewer reviews questions | Fallback to template bank | interview trace |
| 11 | Technical Interview Agent | Generate role-specific technical questions | Rubric + role + candidate evidence | Questions, follow-ups, rubric prompts | RAG, code KB, embeddings | interview_questions | Before tech interview | Yes — interviewer reviews | Fallback to bank | interview trace |
| 12 | Summary Agent | Summarize interview + cite evidence | Transcript + notes + scorecards | Summary with evidence ids | LLM with citation enforcement | interview_summaries | After interview | Yes — interviewer signs off | Mark "needs review" | summary trace |
| 13 | Decision Support Agent | Aggregate signals into decision packet | Profile + scores + assessment + interviews + flags | Decision packet (recommendation + risks + evidence) | LLM, aggregator | decisions | Pipeline ready for decision | **Yes — final decision is human** | Block on missing critical signals | decision trace |
| 14 | IDP / Learning Path Agent | Build development plan from gaps | Decision packet + gaps + KB of learning resources | Growth plan | RAG, learning catalog | growth_plans | After hire OR after reject | Yes — recruiter/manager approves | Skip gracefully if no catalog | IDP trace |
| 15 | Audit & Compliance Agent | Continuous compliance checks | All audit streams | Compliance reports + alerts | Rule engine, retention scanner | audit_*, retention_jobs | Daily/event-based | No, but alerts to admins | Surface incidents | compliance reports |

### 9.3 Cross-cutting agent rules
- Every agent has a typed Pydantic I/O schema.
- Prompts are versioned files in the repo (`prompts/<agent>/<version>.md`).
- Outputs always include `evidence_ids[]` when referencing facts.
- Every LLM call logs `{prompt_hash, model, tokens, cost, latency, retrieved_doc_ids}`.
- Agents are idempotent given the same input + version; reruns produce diffable outputs.

---

## 10. RAG and Knowledge Architecture

### 10.1 Responsibility
Provide grounded retrieval for outreach personalization, interview question generation, decision rationale, and assessment grading — without hallucinating beyond the source.

### 10.2 Why it exists
The doc names RAG as the differentiator (§3.1, §3.3). Without strong grounding, generated content drifts and bias risk grows.

### 10.3 Knowledge bases
- **Org KB** — culture, policies, role expectations, evaluation criteria, FAQ.
- **Job KB** — JD, rubric, role-specific must-knows, prior good interview questions.
- **Candidate Evidence Store** — per-candidate `evidence_items` indexed.
- **Skills Knowledge Graph** — Neo4j graph of skills, parents, adjacency, versions.
- **Learning Catalog** — courses, paths, certifications mapped to skills.

### 10.4 Database / store layout
| Store | Use |
|---|---|
| PostgreSQL | source documents, metadata |
| Vector DB (pgvector → Weaviate/Pinecone) | chunk embeddings + metadata filters (org_id, kb_id, doc_id, tags) |
| Neo4j | skills graph + candidate↔skill↔job relations |
| Object storage | raw blobs (docs, audio, code) |

Tables: `kb_collections`, `kb_documents`, `kb_chunks`, `kb_embeddings`, `kb_acls`, `retrieval_runs`.

### 10.5 APIs
- `POST /kb/:scope/documents` (`scope = org|job|candidate`)
- `POST /kb/:scope/index/rebuild`
- `POST /retrieval/query` `{scope_filters, query, top_k}`
- `GET /retrieval/runs/:id`

### 10.6 Services
- Chunker (semantic + token-budget aware) + embedder.
- ACL filter applied at retrieval time (scope by org_id/job_id/candidate_id).
- Hybrid retrieval: BM25 + dense + filter; rerank.
- Grounding rules: every generated claim must be backed by ≥1 retrieved chunk; otherwise mark `unsupported` and downgrade.
- Hallucination guard: post-generation citation check; reject answers with no citations.

### 10.7 AI / agent logic
Used by Outreach, HR Interview, Technical Interview, Assessment, IDP, Decision Support agents.

### 10.8 Inputs
Documents (PDF, MD, HTML), JDs, rubric files, candidate evidence.

### 10.9 Outputs
Indexed chunks, retrieval results, grounded contexts.

### 10.10 Depends on
§1 (ACL), §2 (candidate evidence), §3 (job).

### 10.11 Security / privacy / fairness / audit
- Per-chunk ACLs (org/job/candidate) enforced server-side.
- Candidate evidence retrieval respects anonymization mode (`for=screening` returns redacted chunks).
- Retrieval runs logged for replayability.

### 10.12 Build first
Org KB + Job KB + simple chunker + pgvector + grounded retrieval API.

### 10.13 Build later
Skills KG in Neo4j, reranker, multi-modal evidence (code, screenshots), rolling re-index.

---

## 11. Decision Support Architecture

### 11.1 Responsibility
Combine all collected signals into a **Hiring Recommendation Package** for the human decision-maker, store the final decision with rationale, and trigger downstream IDP/feedback flows.

### 11.2 Why it exists
The doc explicitly defines a Decision Support Agent that produces a "package to send with the hiring decision" with rationale, risks, and compliance.

### 11.3 Modules
- `decision-aggregator`
- `risk-flagger`
- `recommendation-builder`
- `decision-recorder`
- `notifications`
- `cycle-restart` — if no Top-K passes, restart workflow.

### 11.4 Database tables
| Table | Purpose |
|---|---|
| `decisions` | application × proposed_decision × confidence × rationale × proposed_by(agent) |
| `decision_packets` | decision × evidence_summary × scorecards_summary × risks × fairness_summary × kb_refs |
| `final_decisions` | application × hire/reject/hold × decided_by × decided_at × reason × packet_id |
| `decision_audit` | append-only |
| `cycle_restarts` | job × reason × triggered_by × at |

### 11.5 APIs
- `POST /applications/:id/decision/propose`
- `GET /applications/:id/decision/packet`
- `POST /applications/:id/decision/finalize` `{decision, reason}` (HITL)
- `POST /jobs/:id/cycle/restart`

### 11.6 Services
- Aggregator pulls scores, assessment grades, interview scorecards, bias flags, evidence list.
- Risk flagger: missing assessment, low confidence, evidence conflicts, fairness flag, low interview agreement.
- Recommendation builder generates a structured packet (not free text only).
- Cycle restart triggers a new sourcing/screening run per documented behavior.

### 11.7 AI / agent logic
- **Decision Support Agent** — composes the packet; explicitly never finalizes.

### 11.8 Inputs
Profile, scores, assessment, interviews, flags.

### 11.9 Outputs
Decision packet, final decision record, downstream triggers.

### 11.10 Depends on
§2, §5, §7, §8, §6, §14.

### 11.11 Security / privacy / fairness / audit
- Final decisions write to immutable audit.
- "Hold" status supported to prevent forced binary outcomes.
- Decisions exportable for compliance review (PDF + JSON).

### 11.12 Build first
Aggregator + packet view + finalize endpoint with HITL.

### 11.13 Build later
Cross-candidate comparison view, model-based recommendation, calibration of recommended thresholds.

---

## 12. IDP and Feedback Architecture

### 12.1 Responsibility
Convert hiring outcomes into **Individual Development Plans** for hires and **constructive feedback + learning recommendations** for rejected candidates, closing the documented hiring↔development gap.

### 12.2 Why it exists
A core PATHS differentiator: 41% Egypt early-tech-turnover problem (doc §1.1).

### 12.3 Modules
- `gap-analysis`
- `learning-catalog`
- `growth-plan-builder`
- `candidate-feedback-builder`
- `tracking` — milestones, check-ins.

### 12.4 Database tables
| Table | Purpose |
|---|---|
| `growth_plans` | application × plan_json × goals × status × review_dates |
| `growth_plan_milestones` | plan × milestone × due × status × evidence |
| `learning_resources` | id, title, provider, skill_ids[], level, url |
| `candidate_feedback` | application × type (reject/coach) × content × delivered_at |
| `retention_signals` | hire × signal × value × ts |

### 12.5 APIs
- `POST /applications/:id/growth-plan/generate`
- `GET /growth-plans/:id`
- `PATCH /growth-plans/:id/milestones/:mid`
- `POST /applications/:id/feedback/generate`
- `POST /applications/:id/feedback/send`
- `POST /learning/resources` (admin)

### 12.6 Services
- Gap analysis = job rubric expectations − candidate verified evidence.
- Plan builder maps gaps → resources → milestones → review schedule.
- Feedback for rejects is **constructive, evidence-cited, never demoralizing**; delivery is opt-in and HITL.

### 12.7 AI / agent logic
- **IDP / Learning Path Agent**.

### 12.8 Inputs
Decision packet, gaps, learning catalog, candidate consent.

### 12.9 Outputs
Growth plan, milestones, feedback letter, retention tracking signals.

### 12.10 Depends on
§5, §7, §8, §11, §10 (KB of learning resources), §14.

### 12.11 Security / privacy / fairness / audit
- Feedback content reviewed by recruiter before send (HITL).
- No deficit framing on protected attributes.
- Candidate can opt out of feedback at any time.

### 12.12 Build first
Gap analysis + simple growth plan + manual feedback editor.

### 12.13 Build later
Learning catalog, automated milestone tracking, retention dashboards.

---

## 13. Reporting and Analytics Architecture

### 13.1 Responsibility
Operational + strategic dashboards: time-to-hire, cost-per-hire, response rate, funnel conversion, source effectiveness, bias metrics, assessment performance, recruiter efficiency, retention.

### 13.2 Why it exists
The doc's success metrics are explicit (§1.1 objectives). Without reporting, PATHS cannot prove its own value.

### 13.3 Modules
- `metrics-collectors` (event-driven)
- `aggregations` (daily/weekly rollups)
- `dashboards-api`
- `exports` (CSV/PDF)
- `alerts`

### 13.4 Database tables
| Table | Purpose |
|---|---|
| `events` | append-only event stream (entity, type, payload, ts) |
| `metric_definitions` | metric_id, formula, dims |
| `metric_aggregates` | metric × org × dim × period × value |
| `funnel_snapshots` | job × stage × count × period |
| `bias_metric_aggregates` | per protected dim aggregates |
| `cost_inputs` | per-hire cost components |

### 13.5 APIs
- `GET /analytics/time-to-hire?org_id=&job_id=`
- `GET /analytics/cost-per-hire`
- `GET /analytics/funnel?job_id=`
- `GET /analytics/sources`
- `GET /analytics/bias?dim=`
- `GET /analytics/recruiter-efficiency`
- `POST /analytics/exports`

### 13.6 Services
- Event collector consumes domain events from every module.
- Pre-aggregations to keep dashboards fast.
- Alert rules (bounce spike, scoring drift, fairness threshold breach).

### 13.7 AI / agent logic
None (pure analytics). Optional anomaly detection later.

### 13.8 Inputs
Domain events from all modules.

### 13.9 Outputs
Aggregates, dashboards JSON, exports, alerts.

### 13.10 Depends on
All other modules emit events.

### 13.11 Security / privacy / fairness / audit
- Aggregates only — never expose individual protected attributes.
- Org-scoped; cross-org analytics only at platform admin level.

### 13.12 Build first
Funnel + time-to-hire + outreach response rate.

### 13.13 Build later
Cost-per-hire, retention indicators, bias metric dashboards, anomaly alerts.

---

## 14. Audit, Compliance, and Security Architecture

### 14.1 Responsibility
Cross-cutting controls: audit logs, consents, access control, privacy, retention, explainability logs, HITL records, sensitive data handling.

### 14.2 Why it exists
Egypt PDPL (Law 151/2020), GDPR, EU AI Act, EEOC — all imposed by the doc's scope (§2.2, NFRs).

### 14.3 Modules
- `audit-bus` — every write goes through the bus → `audit_events`.
- `consent-mgr`
- `access-control` — RBAC + ABAC for candidate-level rules.
- `retention` — data lifecycle, deletion jobs.
- `explainability-store` — per-decision explanation packets archived.
- `hitl-records` — non-repudiable approvals.
- `sensitive-data-vault` — KMS-encrypted storage for PII / contact / docs.
- `dsr` — Data Subject Requests (access, rectification, erasure, portability).
- `breach-notification` — 72-hour reporting workflow.

### 14.4 Database tables
| Table | Purpose |
|---|---|
| `audit_events` | actor, action, target, before, after, ts, ip, request_id |
| `consents` | scope, purpose, lawful_basis, granted_at, revoked_at |
| `retention_jobs` | entity, policy, next_run, last_run |
| `dsr_requests` | type, candidate_id, status, due_at |
| `hitl_approvals` | shared with §6 |
| `decisions_archive` | snapshot of decision packets |
| `secret_references` | pointers to vault entries (no plaintext) |
| `breach_incidents` | severity, scope, status, notification_due_at |

### 14.5 APIs
- `GET /audit/events?scope=...`
- `POST /consents` / `DELETE /consents/:id`
- `POST /dsr/requests` (candidate-side)
- `GET /dsr/requests/:id`
- `POST /retention/policies`
- `GET /compliance/reports`

### 14.6 Services
- Audit writer SDK used by every module — non-bypassable.
- Retention scheduler runs delete/anonymize jobs per policy.
- DSR workflow: identity verify → fulfill within statutory window.
- Breach playbook automation.

### 14.7 AI / agent logic
- **Audit & Compliance Agent** — daily checks for missing audit entries, unconsented outreach, retention overruns.

### 14.8 Inputs
Every module's domain events; consents; legal policies.

### 14.9 Outputs
Audit trail, compliance reports, DSR fulfillment, breach reports, retention executions.

### 14.10 Depends on
§1.

### 14.11 Security / privacy / fairness / audit
- KMS for encryption at rest; mTLS internal; TLS 1.3 external.
- Secrets in Vault / cloud secret manager (no env files in prod).
- Prompt-injection defenses on every LLM input (sanitize, escape, role-check).
- Rate-limiting on outreach, scraping, public endpoints.

### 14.12 Build first
Audit bus, RBAC, consents, secrets vault.

### 14.13 Build later
DSR portal for candidates, compliance dashboards, breach automation.

---

## 15. Backend Architecture

### 15.1 Stack (per documentation)
- **API**: FastAPI (Python).
- **Workers**: Celery / RQ on Redis.
- **DBs**: PostgreSQL (transactional + pgvector), Neo4j (skills/identity graph), Redis (cache + queue), Vector DB (Weaviate/Pinecone for scale).
- **Object storage**: S3-compatible.
- **AI runtime**: LangGraph + CrewAI + LangChain.
- **DevOps**: Docker, GitHub Actions, OpenTelemetry, Prometheus, Grafana, Vault.

### 15.2 Recommended folder layout (high level)
```
paths-backend/
├── apps/
│   ├── api/                  # FastAPI app + routers
│   ├── workers/              # Celery/RQ workers
│   └── agents/               # LangGraph runtimes
├── modules/
│   ├── core_platform/        # auth, tenancy, iam, audit_core
│   ├── candidate_data/       # ingestion, parsing, normalization, identity, evidence
│   ├── jobs/                 # jobs, jd_parser, rubric, pipeline
│   ├── sourcing/             # inbound, outbound, pools, saved_searches
│   ├── screening/            # match_engine, evidence_gate, aggregator, fairness, explainer
│   ├── bias_fairness/        # anonymizer, hitl, fairness_monitor, bias_audit
│   ├── assessments/          # templates, generator, delivery, grading
│   ├── interviews/           # scheduler, templates, transcripts, scorecards, summaries
│   ├── decision_support/     # aggregator, packet, finalizer, cycle_restart
│   ├── idp_feedback/         # gap_analysis, growth_plan, candidate_feedback
│   ├── analytics/            # collectors, aggregates, dashboards
│   ├── audit_compliance/     # audit_bus, consent, retention, dsr
│   └── rag/                  # kb, chunker, embedder, retriever, ground_check
├── ai/
│   ├── agents/               # one folder per agent (schema, prompts, tools, runtime)
│   ├── tools/                # MCP-style tools (Hunter.io, GitHub, calendars, sandbox)
│   ├── prompts/              # versioned prompt templates
│   └── eval/                 # offline evaluation harness
├── integrations/
│   ├── ats/                  # CSV/JSON, partner APIs
│   ├── email/                # SES/SendGrid
│   ├── linkedin/, github/, portfolio/, hunter/
│   └── calendar/             # google, outlook
├── shared/
│   ├── schemas/              # Pydantic DTOs (single source of truth)
│   ├── repositories/         # SQLAlchemy + Neo4j repos
│   ├── events/               # event bus + types
│   ├── security/             # rbac, encryption, prompt_injection, rate_limit
│   ├── observability/        # otel, logging, tracing, metrics
│   └── utils/
├── migrations/               # Alembic + Neo4j migrations
├── infra/                    # docker, compose, k8s, terraform
└── tests/
```

### 15.3 Layered design per module
- **Controllers/Routes** (FastAPI routers) — thin, validate DTOs, call services.
- **Services** — business logic, transactions, calls repositories + AI services.
- **Repositories** — DB access; one repo per aggregate root.
- **DTOs/Schemas** — Pydantic models, also documented in OpenAPI.
- **Domain Events** — emitted by services, consumed by `analytics`, `audit`, `notifications`.
- **Background Jobs** — long-running (parsing, scoring, embedding, scraping, scheduled reports).
- **AI Service Layer** — typed wrappers around agents; never call LLMs from controllers.
- **Integration Layer** — adapters per external system; tested with contract tests.
- **Security Layer** — auth middleware, RBAC guard, tenant resolver, prompt-injection filters, rate limiter, audit writer.

### 15.4 Cross-cutting principles
- Outbound dependencies behind interfaces (no FastAPI controller imports `openai` directly).
- Idempotent agents and workers; every job has a deduplication key.
- Saga pattern for multi-step pipelines (sourcing → screening → outreach …) with compensations.
- Blue/green-friendly migrations.

---

## 16. Database Architecture

### 16.1 Entity map (consolidated)

```
                          +--------------------+
                          |   organizations    |
                          +---------+----------+
                                    |
            +-----------------------+----------------------+
            |                       |                      |
       +----v----+             +----v-----+         +------v------+
       |  users  |             |   jobs   |         | candidates  |
       +----+----+             +----+-----+         +------+------+
            |                       |                      |
   +--------v---------+      +------v------+        +------v---------+
   | org_memberships  |      | job_skills  |        | candidate_skills|
   +------------------+      +------+------+        +----------------+
                                    |                      |
                              +-----v-----+          +-----v-----+
                              |   skills  |<---------|  skills   |
                              +-----------+          +-----------+

       +-------------------+         +---------------------+
       |   applications    +---------+    assessments      |
       | (candidate × job) |         +---------------------+
       +---------+---------+
                 |
        +--------+----------+
        |                   |
  +-----v------+    +-------v-------+
  | interviews |    | growth_plans  |
  +------------+    +---------------+

  Cross-cutting: evidence_items, audit_events, consents,
                 hitl_approvals, decisions, kb_*, sourcing_*,
                 outreach_*, bias_*, fairness_*, retention_*
```

### 16.2 Many-to-many tables
- `job_skills`, `candidate_skills`, `org_memberships`, `role_permissions`, `talent_pool_members`, `interview_invitees`, `job_collaborators`.

### 16.3 Audit / status / history tables
- `audit_events`, `bias_audit`, `decision_audit`, `merge_history`, `jobs_history`, `match_runs`, `sourcing_runs`, `enrichment_events`, `outreach_events`, `dsr_requests`, `hitl_approvals`, `de_anon_events`, `breach_incidents`.

### 16.4 AI result tables
- `match_scores`, `match_aggregate`, `match_explanations`, `assessment_grades`, `interview_summaries`, `decision_packets`, `growth_plans`, `retrieval_runs`.

### 16.5 Evidence tables
- `evidence_items`, `candidate_sources`, `candidate_profile_fields`, `assessment_artifacts`, `interview_transcripts`.

### 16.6 Recommended indexing & extras
- pgvector indexes on embedding columns (`candidate_skills.embedding`, `kb_chunks.embedding`).
- Neo4j: `Candidate`, `Skill`, `Job`, `Organization`, `EvidenceItem` nodes; relationships `HAS_SKILL`, `REQUIRES`, `APPLIED_TO`, `RESOLVES_TO`, `MATCHED_BY`.
- Partition `audit_events` by month.
- Append-only buckets (object storage with object-lock) for `bias_audit` and decision archives.

---

## 17. API Architecture

| Group | Purpose | Main endpoints | Request | Response | Roles | Tables |
|---|---|---|---|---|---|---|
| **Auth** | Sessions, MFA, SSO | `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/mfa/*` | credentials/tokens | session/JWT | public | users, sessions |
| **Organization** | Tenant + settings | `/orgs`, `/orgs/:id`, `/orgs/:id/settings` | org payload | org/state | admin, super_admin | organizations, org_settings |
| **User/Role** | Members, roles, invites | `/orgs/:id/members`, `/invitations`, `/me` | role/perm payloads | user/perm sets | admin | users, org_memberships, roles, permissions |
| **Candidate** | Profile + evidence + dedupe | `/candidates`, `/candidates/:id`, `/candidates/import/*`, `/candidates/dedupe/*` | files/urls/ATS rows | master profile, evidence | recruiter, hiring_manager | candidates, evidence_items, candidate_skills |
| **Job** | Job lifecycle + rubric + pipeline | `/jobs`, `/jobs/:id/jd/parse`, `/jobs/:id/rubric`, `/jobs/:id/publish` | job spec | job + rubric | recruiter, hiring_manager | jobs, job_skills, job_rubric |
| **Application** | Tie candidate↔job + status | `/jobs/:id/applications`, `/applications/:id`, `/applications/:id/status` | apply/transition | application state | recruiter, candidate (apply) | applications |
| **Sourcing** | Inbound + outbound + pools | `/jobs/:id/sourcing/runs`, `/talent-pools`, `/saved-searches` | query/sources | runs/results | recruiter | sourcing_runs, talent_pools |
| **Screening** | Match runs + shortlist | `/jobs/:id/match/runs`, `/shortlists/:id/approve` | rubric_version | scores + packet | recruiter, hiring_manager | match_*, shortlists |
| **Bias/Fairness** | Anonymized views + HITL + flags | `/candidates/:id/anonymized`, `/hitl/approvals/:id/decide`, `/fairness/metrics` | approval payload | views/metrics | recruiter, hiring_manager, admin | anonymized_views, hitl_approvals, fairness_metrics |
| **Outreach** | Enrichment + messaging + tracking | `/applications/:id/outreach/messages`, `/outreach/events` (webhook) | message draft | sent state | recruiter | outreach_messages, outreach_events |
| **Assessment** | Templates + delivery + grading | `/assessments/templates`, `/applications/:id/assessments`, `/candidate/assessments/:token` | template/answers | assessment + grades | recruiter, candidate | assessments, assessment_* |
| **Interview** | Schedule + questions + scorecards + summary | `/interviews`, `/interviews/:id/questions/generate`, `/interviews/:id/finalize` | schedule/notes | interview state | recruiter, interviewer | interviews, interview_* |
| **Decision** | Packet + finalize + restart | `/applications/:id/decision/packet`, `/applications/:id/decision/finalize` | decision payload | final decision | hiring_manager | decisions, final_decisions |
| **IDP** | Growth plan + feedback | `/applications/:id/growth-plan/generate`, `/applications/:id/feedback/send` | plan/feedback | plan/feedback | recruiter, hiring_manager | growth_plans, candidate_feedback |
| **Analytics** | KPIs + funnels + bias metrics | `/analytics/*` | filters | aggregates | recruiter, hiring_manager, admin | metric_aggregates |
| **Audit** | Events + consents + DSR + retention | `/audit/events`, `/consents`, `/dsr/requests`, `/retention/policies` | filters/policies | events/state | admin, candidate (DSR) | audit_events, consents, dsr_requests |
| **RAG/KB** | Document mgmt + retrieval | `/kb/:scope/documents`, `/retrieval/query` | docs/queries | indexed/results | admin, agents | kb_* |

Cross-cutting: every endpoint requires `X-Org-Id` (or subdomain), JWT, and runs through the audit writer. Webhooks (email events, calendar, ATS) live under `/webhooks/*` with HMAC verification.

---

## 18. Workflow Architecture

### 18.1 Candidate application flow (Inbound)
1. Candidate submits CV via public application form (`POST /candidates/apply`).
2. Consent captured.
3. Ingestion + parsing + normalization (§2).
4. Identity resolution merges if match exists.
5. Evidence items written.
6. `application` row created, status `applied`.
7. Bias guardrail anonymizes.
8. Screening run (auto for inbound) computes score.
9. Candidate enters Top-K candidate queue if rubric passes.
10. Recruiter HITL approves shortlist.

### 18.2 Recruiter job creation flow
1. Recruiter creates draft job (`POST /jobs`).
2. Pastes JD → JD parser proposes structured fields, skills, criteria.
3. Recruiter confirms / edits skills, weights, must-haves, rubric, pipeline stages.
4. Adds collaborators (hiring manager, interviewers).
5. Publishes job → triggers Inbound application listener and/or Outbound sourcing setup.

### 18.3 Candidate sourcing flow (Outbound)
1. Recruiter writes NL query + selects channels.
2. Sourcing Agent runs adapters, dedupes against existing candidates.
3. Returns ranked raw matches with provenance.
4. Recruiter HITL adds selected to job pipeline.
5. New application rows created with `source = outbound`.

### 18.4 Screening and shortlist flow
1. `POST /jobs/:id/match/runs`.
2. Bias guardrail produces anonymized views.
3. Sub-scorers compute per-dimension scores; evidence gate applies penalties.
4. Aggregator + must-have gate + fairness re-rank → final ranked list.
5. Explanation packets generated.
6. Shortlist proposed → HITL approve.
7. On approve: outreach pipeline becomes available; on reject: optional reasons fed back.

### 18.5 Assessment flow (technical roles)
1. After shortlist approval, Assessment Agent generates assessment from rubric + role + RAG.
2. Recruiter HITL reviews assessment.
3. Candidate notified with tokenized link.
4. Candidate submits → auto-grader runs → manual grader optional.
5. Pass/fail rationale generated → HITL gating decision.
6. On pass, schedule technical interview.

### 18.6 Interview flow
1. Scheduler agent proposes slots; candidate selects.
2. Pre-interview: RAG agent generates questions; interviewer reviews (HITL).
3. Interview conducted; transcript captured (with consent).
4. Interviewer fills scorecard.
5. Summary Agent generates summary with citations; interviewer signs off.

### 18.7 Decision flow
1. Decision Support Agent compiles packet (scores + assessment + interviews + flags + evidence).
2. Hiring manager reviews; final HITL decision (`hire/reject/hold`).
3. Decision archived; notifications dispatched.
4. If `none of Top-K passes`: cycle restart triggered.

### 18.8 IDP / feedback flow
1. On `hire`: IDP Agent builds growth plan from gaps; manager approves.
2. On `reject` (with consent): feedback letter generated with constructive evidence-cited content; recruiter HITL reviews and sends.
3. Optional: candidate retained in talent pool for future re-match.

### 18.9 Admin / compliance flow
1. Admin defines retention policies, consent purposes, role assignments.
2. Audit & Compliance Agent runs daily checks; alerts on anomalies.
3. DSR portal handles candidate access/erasure requests within statutory windows.
4. Breach playbook executes 72-hour notification when triggered.

---

## 19. Build Roadmap

> Mapped to documented Work Plan (§1.4) but structured around production capability rather than calendar weeks.

### Phase 1 — Core Foundation
- **Goal**: Safe multi-tenant platform with audit/RBAC ready before any AI.
- **Build**: tenancy, users, roles/permissions, org memberships, audit bus, secrets vault, base FastAPI scaffolding, Postgres + Redis + Docker compose, observability baseline.
- **Modules**: §1, parts of §14.
- **Tables**: organizations, users, roles, permissions, role_permissions, org_memberships, audit_events, consents, secret_references.
- **APIs**: Auth, Org, User/Role, Audit (read).
- **Agents**: none.
- **Dependencies**: none.
- **Definition of done**: a recruiter user under an org can log in, manage members, every action is in `audit_events`, RBAC enforced on a sample resource.

### Phase 2 — Candidate and Job Data
- **Goal**: Master Candidate Profile + structured Jobs.
- **Build**: CV upload + parsing, evidence store, normalization, manual job CRUD, skills dictionary, JD parser proposal, identity resolution v1.
- **Modules**: §2, §3.
- **Tables**: candidates, candidate_sources, candidate_skills, evidence_items, jobs, job_skills, job_rubric, job_pipeline_stages.
- **APIs**: Candidate, Job, Application (basic).
- **Agents**: Candidate Profile Agent (parsing/normalization), JD-parsing subroutine.
- **Dependencies**: Phase 1.
- **DoD**: upload CV → master profile created → linked to a job; recruiter can build a job with rubric.

### Phase 3 — Matching and Screening
- **Goal**: Top-K shortlist with explanation packet.
- **Build**: sub-scorers, must-have gates, weighted aggregator, explanation generator, shortlist lifecycle.
- **Modules**: §5.
- **Tables**: match_runs, match_scores, match_aggregate, match_explanations, shortlists.
- **APIs**: Screening.
- **Agents**: Screening Agent (deterministic scorers + LLM rationale).
- **Dependencies**: Phase 2.
- **DoD**: given a job + candidates, system produces a ranked Top-K with evidence-backed rationales.

### Phase 4 — Human Approval and Bias Guardrails
- **Goal**: Anonymization-before-evaluation, HITL approval bus, de-anonymization gating.
- **Build**: anonymizer, HITL approval registry + APIs, bias_audit, de-anon flow, baseline fairness checks.
- **Modules**: §6.
- **Tables**: anonymized_views, bias_flags, hitl_approvals, de_anon_events, bias_audit.
- **APIs**: Bias/Fairness.
- **Agents**: Bias Guardrail Agent.
- **Dependencies**: Phases 1–3.
- **DoD**: scoring runs only on anonymized views; shortlist publish requires recorded HITL approval; de-anonymization is logged with purpose.

### Phase 5 — Assessments
- **Goal**: Pre-interview gate for technical roles.
- **Build**: templates, generator (RAG), candidate-side delivery, auto + manual grading, pass/fail HITL.
- **Modules**: §7.
- **Tables**: assessment_templates, assessments, assessment_questions, assessment_submissions, assessment_grades, assessment_artifacts.
- **APIs**: Assessment.
- **Agents**: Assessment Agent.
- **Dependencies**: Phases 2–4, basic RAG (subset of §10).
- **DoD**: shortlisted technical candidate receives an assessment, submits, gets graded, recruiter approves pass/fail.

### Phase 6 — Interviews and RAG
- **Goal**: Structured interviews grounded in org KB.
- **Build**: full RAG (KB + chunker + retriever + grounding), interview templates, scheduler v1 (manual + tokenized links), question generator, scorecards, summary agent.
- **Modules**: §8, §10.
- **Tables**: kb_*, interviews, interview_questions, interview_notes, interview_scorecards, interview_summaries.
- **APIs**: Interview, RAG/KB.
- **Agents**: HR Interview Agent, Technical Interview Agent, Summary Agent.
- **Dependencies**: Phases 2–5.
- **DoD**: a job has an org KB; pre-interview RAG generates rubric-aligned questions; interviewer fills scorecard; summary cites evidence.

### Phase 7 — Decision Support
- **Goal**: Aggregated decision packet + final human decision.
- **Build**: aggregator, packet builder, finalize endpoint, cycle restart, decisions archive.
- **Modules**: §11.
- **Tables**: decisions, decision_packets, final_decisions, decision_audit, cycle_restarts.
- **APIs**: Decision.
- **Agents**: Decision Support Agent.
- **Dependencies**: Phases 3, 5, 6.
- **DoD**: hiring manager opens packet, finalizes decision; if no Top-K passes, cycle restart triggers a new run.

### Phase 8 — IDP and Feedback
- **Goal**: Close the hiring↔development gap.
- **Build**: gap analysis, growth plan, milestones, candidate feedback for rejects.
- **Modules**: §12.
- **Tables**: growth_plans, growth_plan_milestones, learning_resources, candidate_feedback, retention_signals.
- **APIs**: IDP.
- **Agents**: IDP/Learning Path Agent.
- **Dependencies**: Phases 6–7.
- **DoD**: every hire gets an IDP; rejects can opt into feedback letters; both reviewed via HITL.

### Phase 9 — Analytics and Audit
- **Goal**: Measurable + auditable.
- **Build**: event collectors across all modules, aggregations, dashboards APIs, fairness metric aggregates, exports.
- **Modules**: §13, expansion of §14.
- **Tables**: events, metric_definitions, metric_aggregates, funnel_snapshots, bias_metric_aggregates, cost_inputs.
- **APIs**: Analytics, expanded Audit.
- **Agents**: Audit & Compliance Agent.
- **Dependencies**: Phases 1–8.
- **DoD**: time-to-hire / funnel / response rate / source effectiveness / bias metric dashboards live; daily compliance scan runs.

### Phase 10 — Production Hardening
- **Goal**: Ship it.
- **Build**: SSO/SAML, DSR portal, retention scheduler, breach automation, prompt-injection defense suite, rate-limit profiles, advanced fairness re-rank, full vector DB migration (Weaviate/Pinecone), Neo4j skills KG, blue/green deploy, runbooks.
- **Modules**: hardening across all.
- **Dependencies**: Phases 1–9.
- **DoD**: SOC-2-style controls in place; pen-tested; chaos-tested; documented SLOs.

---

## Final Recommendations

### A. Best starting point
Build Phase 1 first **and only after that**, in this exact order:
1. `core_platform` (orgs, users, roles, permissions, RBAC guard).
2. `audit_compliance.audit_bus` and `consents`.
3. The shared `event_bus`, `repositories`, `schemas`, `observability` packages.
4. A trivial `jobs` CRUD wired through RBAC + audit to **prove the platform plane works end-to-end** before any AI is touched.

### B. Correct build order
Phase 1 → 2 → 3 → **4 (non-skippable)** → 5 → 6 → 7 → 8 → 9 → 10.
You may parallelize within a phase, but never skip Phase 4. Bias/HITL is not a feature — it is the *substrate* for everything after Phase 3.

### C. Most important architecture decisions
1. **Anonymized view as a first-class DB column**, not a runtime redaction. This is the only reliable way to keep fairness laws enforced.
2. **Deterministic scorers + LLM rationale**, never LLM scoring. Numbers come from rules; the LLM only explains.
3. **Evidence-first data model**: every claim becomes an `evidence_item` with source, timestamp, confidence. No score can exist without referenced evidence ids.
4. **HITL approvals are non-bypassable in code paths**, not optional UI.
5. **Agent runtime separates orchestration (LangGraph) from agents (CrewAI cells) and tools (LangChain/MCP)** so any one can be swapped.
6. **Separate trust boundary for candidate-side endpoints** (different JWT audience, isolated database access patterns).
7. **Two-database strategy from day one**: Postgres for transactions, vector index alongside (start with pgvector, migrate to Weaviate/Pinecone in Phase 10). Add Neo4j when the skills graph exceeds what relational queries handle.
8. **Append-only audit + immutable bias audit bucket** (object lock) to satisfy Egypt PDPL / EU AI Act / EEOC posture.
9. **Saga-based pipeline orchestration** with compensations, because every step (sourcing, screening, outreach, assessment, interview, decision) must be resumable and auditable.
10. **Versioned prompts in repo + full prompt/response logging** — required for explainability and reproducibility of every AI decision.

### D. What should NOT be built yet
- Real-time, large-scale LinkedIn scraping (legal + technical risk; start with GitHub + ATS imports + manual add).
- Sentiment / facial / affective analysis on interviews (literature §2.2.2 ref [61] flags it as risky and weakly explainable).
- Auto-rejection without HITL.
- Cross-org analytics or any data sharing across tenants.
- Custom per-org permission editors (use fixed role catalog for v1).
- Reranker, model fine-tuning, custom embeddings, or learned fairness re-ranker before Phase 4–6 are stable.
- A standalone Skills KG in Neo4j before the relational skills model is solid.
- Browser-rendered portfolio scrapers (heavy + flaky) — accept URL + extract metadata + manual confirm in v1.
- Full DSR self-service portal — start with admin-mediated DSR fulfillment, expose to candidates in Phase 10.

### E. What the new frontend will need from the backend later
The frontend you'll build later only needs to consume what the backend already exposes. Plan for:

1. **Auth & session**: JWT + refresh, MFA endpoints, SSO redirect URLs.
2. **Tenant context** via `X-Org-Id` header (frontend reads from session) and per-tenant theming hooks (logo, brand color from `org_settings`).
3. **OpenAPI 3.1 schema** auto-generated from FastAPI as the single contract — frontend should generate a typed client from it.
4. **Stable resource URLs** for every entity (`/orgs/:id/jobs/:id/applications/:id`) so the frontend can deep-link.
5. **HITL approval inbox** endpoint (`GET /hitl/approvals?status=pending`) — central queue for all human gates; the frontend's most-used screen.
6. **Anonymized vs de-anonymized projections** with explicit `?view=anonymized|full` query params; the frontend UI must clearly indicate the mode it is showing.
7. **Explanation packets** as structured JSON (not free text), so the frontend can render evidence pills, sub-score bars, fairness flags.
8. **Realtime channel** (WebSocket or SSE) for: pipeline status changes, sourcing run progress, scoring completion, agent failures, new HITL approvals.
9. **Candidate-side tokenized endpoints** for assessments, interview scheduling, feedback acknowledgment — separate audience JWTs.
10. **File upload contracts**: presigned URLs for CV/portfolio/assessment artifacts (don't proxy through API).
11. **Search + filter DSL** for candidates/jobs/applications consistent across endpoints (cursor pagination + filter params).
12. **Webhooks signed payloads** for ATS sync (so partner ATS can notify the frontend's parent system if needed).
13. **Analytics endpoints return both raw series and pre-aggregated KPIs** so frontend can choose to render charts or single-value cards.
14. **i18n-ready strings**: server-side errors include `code` + `message_key`; frontend handles localization (Arabic + English at minimum given Egypt market).
15. **Audit-friendly responses**: every mutation returns the new state + an `audit_event_id` so the frontend can show "Action recorded · view in audit log".

---

*End of Blueprint.*
