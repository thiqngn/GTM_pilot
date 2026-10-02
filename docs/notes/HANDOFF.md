# Handoff: GTM_pilot take-home (approved 2026-10-02)

## Who and what
- Candidate: Thi Nguyen (GitHub: thiqngn). Analytics engineer by own description; the role
  applied for is Data Engineering. Do not claim platform, Spark-at-scale or streaming experience.
- Task: graded take-home for a Data Engineering role at GoTyme/TymeX, "Lending Analytics Design."
  AI use confirmed allowed by HR.
- Requirement doc: docs/requirements.md (converted from the PDF; checked clean, tables intact).
- Deliverables per brief: (1) pipeline solution design, (2) Bronze/Silver/Gold data model,
  (3) 1–2 page decision log: identity strategy, cross-service joins with no FKs, governance
  assumptions, one rejected trade-off, one thing to confirm with a source owner.

## Environment state
- Repo: https://github.com/thiqngn/GTM_pilot (private). Local: D:\thi.q.nguyen\Claude\GTM_pilot.
- Repo-local git identity: 130969925+thiqngn@users.noreply.github.com. Never the work email.
- GH_TOKEN env var points at a different account and breaks pushes. Per terminal:
  Remove-Item Env:GH_TOKEN, then gh auth switch -u thiqngn if needed.
- mattpocock-skills plugin installed. Flow: /grill-with-docs → /to-spec → /to-tickets → /implement.
- Optional validation: Databricks Free Edition or DuckDB. Not required; deliverables are documents.

## Evidence base (what the submission may lean on)
Real work, Jul–Sep 2026, Fabric warehouse + dbt Core at a security vendor:
- dbt Core: 363 models, ~430 declared tests, 140 own commits, authored the layer standards doc.
  No dbt snapshots in the repo; history retention was built as version-based SCD2 in Python.
- Metric-definition work, Jun 2026, in claude.ai sessions (not in the scanned folders;
  verified by Thi): annualised sub-12-month renewable lines; moved ACV/ARR timing from close
  date to contract start date; parent-account region attribution; published before/after
  impact diff of +$121M across 793 opportunities. Migrated product attribution to quote lines
  as single source of truth under three numbered inclusion assumptions; a $281 residual
  traced to one exchange-rate difference.
- Version-based SCD2 for a source that returns blank records; carry-forward with provenance
  and age; production deploy gated against a same-vintage control; per-run audit table.
- Metric contract (source, measure, scope, grain, trigger, agreed date) + cause-bucketed
  reconciliation alert where every moved dollar gets one winning bucket and "unexplained"
  is never dropped. Daily Teams cards routed to the named product owner.
- Time-travel forensics: reproduced a transient guard alert to the minute; refresh-anchored
  vintages; UTC-vs-local timestamp lesson.
- Stage-history rebuild from two CRMs with explicit sentinels and valid_through.
- Field-retirement audit as a two-lens evidence pack with stated history gaps.
- Drift detectors: label registry with exempt sentinels; stripped-key collisions; copied
  fields diverging from the combined source.

Vocabulary only, never evidence (agreed): the NAB Domo demo (DMO/nab_*) and the grill-test
demo repo. They are the source of all Databricks, Unity Catalog, streaming and data-contract
wording in the folder.
Side engagements (Ben Line, SkinSeoul pilot, Zenlytic rollout): not used.

## Language rules
- Timing-rule anecdote, true version only: one metric's timing rule moved a deal into a
  different quarter from the others; the fix was a written rule and a published impact diff.
  No "lost trust" language.
- Patterns only in anything meant for the submission: no company, system or table names.

## Complexity rule
- Fallback chains, timeline rules and precedence orders are design-doc detail.
- The decision log gets one sentence of principle per item plus a pointer to the design doc.
- Coach flags any grill answer that adds mechanism without adding a principle.

## Strategy
Differentiate on business reasoning and governance patterns, not stack. Stack decision deferred
until semantics settle. Likely final: design on the TymeX stack (Databricks on AWS, Delta
Medallion, DMS CDC, Kafka, Unity Catalog), with every Databricks claim defensible from docs in
two minutes. Fabric transfers almost fully except Unity Catalog governance: study priority.
Gaps to fill from the schema and documentation, not from background: PII masking and audit
evidence, CDC/streaming ingestion, person-level identity resolution, lending domain.

## Reading of the brief
1. Funnel conversion: metric semantics. Whose status defines FUNDED; when the 14-day clock
   starts; which channel counts. Needs an in-progress bucket and exclusion of never-submitted.
2. Decision reproducibility: event time vs knowledge time. Inputs as of decided_at per
   decision_version. Open: which table is "declared income"; role of deprecated loan_decision_v1
   (retained to 2028-08, implies a 3-year horizon).
3. Offer expiry watchlist: operational feed with a morning SLA. offer_expires_at,
   offer_expired_at, is_deleted, absence of offer_acceptance.
4. PII proof: compliance as evidence. Who accessed what, what masking applied to each PII column
   at that time, lineage proving the funnel report read only pseudonymised layers.

Planted mess: deprecated table; free-text decision_outcome vs overall_status; pipe-delimited
risk_flags; comma-separated product_codes; channel copied across three services; soft deletes;
re-submission and re-decisioning; one offer row per application (UNIQUE) so offer history exists
only in CDC; at most one KYC request per type per application; four product identifiers with
no shared key (whitelist.product_codes, loan_application.product_code, product.product_key,
loan_offer.loan_product free text); kyc_result has no unique constraint on kyc_request_id, so
several results per request are possible; PII is wider than the brief's two examples
(date_of_birth, email, document s3_path, bureau_response_json, staff_id, changed_by, PEP inside
risk_flags); status history has no sequence column, so ordering is changed_at then id and ties
are possible.

## Pain behind the assignment
Four microservices, three teams, no shared FKs, joins now expected in the data platform.
Different teams produce different conversion numbers; the platform is asked to referee.
Mirrors the metric-definition work done at work: ACV/ARR timing moved from close date to
contract start, parent-account attribution, quote lines as single source of truth for product
attribution, stage vs forecast category, royalty exclusion. Same three questions every time:
whose definition, when to count, which entity to attribute to.

## Differentiators (patterns only in the submission; no company, system or table names)
- Metric definitions as dated contracts with a metric owner and a source owner. [verified]
- Every definition change ships with a published before/after impact diff and numbered
  assumptions. [verified: +$121M/793 opps diff; three inclusion assumptions; $281 residual]
- Reconciliation as a published dataset with an owner per disagreement; aggregate gates with
  two thresholds; never per-row refusal. [verified; "rate as KPI" is a design choice, not past work]
- Absent reading ≠ new version: CDC images in Bronze, first-observed values pinned in Silver,
  provenance column on every fallback. [verified pattern, new application]
- Timeline rule for drop-off reasons: first exit after last re-entry; unexplained never dropped;
  buckets + funded + in-progress sum to the cohort. [verified pattern, new application]
- Frozen cohort facts at maturity, daily recompute, restatement as a diff with a threshold.
  [verified pattern: refresh-anchored vintages, time-travel replay]
- Identity crosswalk as a Silver entity with provenance on every join. [design; key-alignment
  lessons are the evidence, not a prior crosswalk build]
- Compliance evidence pack as a Gold deliverable. [methodology shape verified; PII tooling is
  from docs]
- Scoped NRT: one alert stream (FAILED disbursement), daily batch primary. [schema-derived only]

## Assumptions with reasoning
- Volume: thousands of applications/day. application_number AURPL-YYYYMMDD-NNNNN caps daily
  sequence at 99,999. Daily batch default.
- Real-time: nothing in the four questions needs sub-minute data. NRT = minutes, one stream.
- Regulator: Philippine bank (BSP, Data Privacy Act). "CIC" exists in PH and VN; the doc has a
  Vietnamese word, so likely written in VN. State PH; one-line change.
- Reproducibility horizon: match loan_decision_v1 retention (2028-08). Covers only decisions
  after CDC capture began.
- Timezone: cohort month in Asia/Manila; per-service write zone is a confirm item; exposure =
  applications whose UTC and local dates fall in different months (8-hour window).

## Settled in grill rounds 1–2 (funnel)
Design-doc detail. The decision-log principles are at the end of this section.
- Anchor: submitted_at pinned at first observed value; fallback CDC status→SUBMITTED commit
  ts; then first SUBMITTED history row; anchor_source recorded. created_date rejected.
- FUNDED: application service says FUNDED (history or status). Timestamp: history changed_at,
  then CDC status→FUNDED commit ts, then first SUCCESS disbursed_at; funded_ts_source recorded.
  Disbursement never establishes funding. REVERSED stays funded for funnel, flagged.
- Clock: from submitted_at; 14 is a contract parameter.
- Channel: structural first (branch_applicant → BRANCH; applicant → MOBILE/WEB from column or
  DIGITAL_UNKNOWN). Both rows present: loan_application.branch_code breaks the tie. Neither:
  column with channel_source='COLUMN', else UNKNOWN as a published row. One channel column on
  the fact + assisted_acceptance flag; accepted_channel only on the acceptance fact.
- Reasons: timeline of exit events (DECLINED, OFFER_RETRACTED, WITHDRAWN, offer expiry,
  disbursement FAILED/REVERSED) and re-entry events (APPROVED/CONDITIONALLY_APPROVED after exit,
  offer re-issued, new acceptance, disbursement retry). Reason = first exit after last re-entry.
  Retraction at/after expiry → OFFER_EXPIRED with retracted_on_expiry. WITHDRAWN sub-cases:
  WITHDRAWN / OFFER_DECLINED_BY_CUSTOMER / CANCELLED_AFTER_SIGNING. Fine codes in Silver,
  roll-up to the brief's five in a Product-owned versioned mapping. In-progress is a sixth bucket.
- Decision version: latest decided_at ≤ cutoff for attributes; first version kept;
  re_decision_count. Legacy v1 for reproducibility only.
- Freeze: cohort fact written once at maturity with run id and lineage; daily recompute; diff
  published; restatement is a new version with a reason.
- Gates: flag threshold set by source owner (Lending squad), hold threshold by metric owner
  (Product); HELD row published, never silent.
- Grain: per application; person_sk = keyed hash of national_id (NOT NULL in both applicant
  tables; whitelist unique on it); customer_id is an attribute; prior_application_count.

Decision-log principles, one sentence each:
- Stages: the owning service's state says whether a stage was reached; the timestamp comes from
  the best evidence in a declared order, with provenance recorded.
- Reasons: the drop-off reason is the first exit after the last re-entry, evaluated at day 14;
  unexplained is never dropped and buckets sum to the cohort.
- Channel: structure beats a nullable column; UNKNOWN is published, never hidden.
- Freeze: a mature cohort is written once; restatement is a versioned diff, never an overwrite.
- Gates: disagreement is published per row and gated in aggregate; the source owner sets the
  flag level, the metric owner sets the hold level.
- Identity: one declared identifier, keyed hash, provenance on every cross-service join.

## Decisions reached earlier, still standing
- Cross-service key application_number; application_sk hashed from it.
- KYC for reproducibility: latest completed before decided_at per decision_version.
- Strip 'CUST-' from kyc_request.user_ref to match customer_id. Confirm items: borrower_id =
  customer_id (borrower_id is NOT NULL for branch applicants who have no customer_id);
  credit_assessment.applicant_id meaning (VARCHAR(36) vs branch_applicant BIGINT id).
- PII: raw Bronze to a break-glass group; keyed SHA-256 pseudonyms in Silver; UC column masks
  on remaining sensitive Gold columns. Proof = audit logs + lineage + grant history. Hash key in
  a secret scope backed by KMS; rotation breaks historical joins. [all from docs; study]
- Deliverables: README, one-page business summary, three Markdown docs with Mermaid (pipeline,
  data model with Delta DDL, decision log), one SQL file for the four questions plus a fifth.
  Fifth question (settled 2026-10-02): whitelist-to-application conversion. Rejected trade-off
  candidate: Databricks Workflows over AWS Glue for orchestration.

## Contested calls needing extra scrutiny
Reason-code roll-up (OFFER_RETRACTED placement), the timeline rule's edge cases, the Q4
evidence pack design. These decide whether the submission stands out.

## Guesses flagged to the grill, still open
Kafka lag magnitude; operational meaning of OFFER_RETRACTED; whether CONTRACT_SIGNED echoes
offer_acceptance; status history completeness; per-service timestamp zones; whether
submitted_at and the SUBMITTED history row share a transaction.
