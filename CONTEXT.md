# CONTEXT.md — Lending Analytics take-home: settled terms and decisions

Status key: **SETTLED** (agreed in grilling), **OPEN** (under discussion), **ASSUMED** (no source-owner to ask; reason recorded).
Every term cites the schema field or brief line it rests on. This file is a glossary of business meaning; pipeline mechanics live elsewhere.

## 1. Ubiquitous language

### Grain and cohort

**Application** — SETTLED
One row of `loan_application`, identified by `application_number`. The grain of the funnel. A person may hold several.
_Avoid_: loan (that is the funded outcome), case, request.

**Stage Evidence Principle** — SETTLED
For every funnel stage: the owning service's own state decides whether the stage was reached; the stage timestamp comes from the best available evidence in a declared order; the fact records which source supplied it (`*_source`), and the share of rows on each fallback is a published quality metric. Another service's record never establishes that a stage happened; it may only lend a timestamp, and that case is a published history-completeness defect.

**Object-Owner Exit Principle** — SETTLED
The application service decides stages. The service that owns an object decides exits from that object: offer expiry and offer soft-delete are disbursement-service exits; a DECLINED `decision_version` is a decision-service exit at its `decided_at`. The application service's matching transition is the echo; a missing echo is a published disagreement. When the echo exists, the earlier event wins by timeline.

**Submission Anchor** — SETTLED
The instant an Application entered the funnel. Evidence order: first non-null `loan_application.submitted_at` ever observed; else the change-capture commit time of the status change to SUBMITTED; else the first `application_status_history` row with `to_status = 'SUBMITTED'`. Provenance in `anchor_source`. A later rewrite of `submitted_at` is a published defect, not a new anchor.
_Avoid_: last_submitted_at (an attribute, with `resubmission_count`), created_date (can precede submission by days).

**Cohort Month** — SETTLED
Calendar month of the Submission Anchor in bank-local time (Asia/Manila). Pre-submission applications (`APPLYING`, `APPLYING_EXPIRED`, or `WITHDRAWN` with no anchor) are outside the denominator; they belong to a separate started-to-submitted rate for Product.

**Funnel Window** — SETTLED
Submission Anchor plus 14 days. The 14 is a parameter of the metric contract. The window does not reset on re-submission because time the bank spends asking for corrections is real customer time.

**Cutoff** — SETTLED
The instant the Funnel Window closes. Every funnel outcome is evaluated using only events timestamped at or before the Cutoff.

### Stages and outcomes

**Funded (funnel sense)** — SETTLED
The application service says FUNDED (in `application_status_history` or current `status`) at or before the Cutoff. Timestamp evidence order: history `changed_at`; else change-capture commit time of the status change; else `disbursed_at` of the first SUCCESS `disbursement`. Provenance in `funded_ts_source`. A later REVERSED disbursement does not un-fund for the funnel; it sets a reversal flag. Commercial metrics net reversals out; funnel and commercial are different metrics with different rules.
_Avoid_: disbursement SUCCESS as the definition. SUCCESS with no FUNDED anywhere in the application service is not funded and goes to the Disagreement Dataset.

**Application Timeline** — SETTLED
One ordered sequence of events per Application drawn from all four services, each event classed as progress, exit, or re-entry.
- Exit: transitions to DECLINED, OFFER_RETRACTED, WITHDRAWN; a DECLINED `credit_assessment` version; offer expiry (`offer_expired_at`, or `offer_expires_at` passing with no acceptance); `loan_offer.is_deleted` flipping to 1 with no acceptance (OFFER_WITHDRAWN, bank-initiated); disbursement FAILED or REVERSED.
- Re-entry: APPROVED or CONDITIONALLY_APPROVED after an exit; an offer re-issued (a change to the single `loan_offer` row); a new `offer_acceptance`; a disbursement retry.
- Not an exit, published as a disagreement: offer soft-delete after acceptance or after FUNDED; FUNDED after a DECLINED version inside the window (funded wins for the funnel; highest severity, because that is a control failure, not lag).

**Drop-off Reason (fine code)** — SETTLED
Outcome of an Application at Cutoff. If Funded within the window: FUNDED. Otherwise the first exit event after the last re-entry (or after submission if none). If no un-reversed exit: IN_PROGRESS. Codes: FUNDED, CREDIT_DECLINED, OFFER_RETRACTED, OFFER_WITHDRAWN, WITHDRAWN, OFFER_DECLINED_BY_CUSTOMER, CANCELLED_AFTER_SIGNING, OFFER_EXPIRED, DISBURSEMENT_FAILED, IN_PROGRESS.
Sub-cases of a WITHDRAWN transition, derived from timeline state at that moment: no offer yet → WITHDRAWN; offer and no acceptance → OFFER_DECLINED_BY_CUSTOMER; acceptance exists → CANCELLED_AFTER_SIGNING.
Retraction on expiry: `offer_expired_at` at or before the OFFER_RETRACTED `changed_at` means expiry is the earlier exit → OFFER_EXPIRED with `retracted_on_expiry = true`.
`APPLYING_EXPIRED` cannot occur post-submission, so the brief's "expired" means offer expiry only.
_Avoid_: bucket-rank precedence (rejected: mis-buckets expiry followed by re-decision, and decline followed by re-approval).

**Drop-off Reason (roll-up)** — SETTLED
The brief's five buckets plus IN_PROGRESS, produced from fine codes by a versioned mapping owned by Product. Organising axis is "who said no": bank (CREDIT_DECLINED, OFFER_RETRACTED, OFFER_WITHDRAWN), customer (WITHDRAWN, OFFER_DECLINED_BY_CUSTOMER, CANCELLED_AFTER_SIGNING), clock (OFFER_EXPIRED), ops (DISBURSEMENT_FAILED), nobody yet (IN_PROGRESS). Offers expire at 7 days, so an open offer at day 14 signals a slow pipeline; the IN_PROGRESS size is a KPI.

**Decision of Record (funnel)** — SETTLED
Latest `credit_assessment.decision_version` with `decided_at` at or before Cutoff. Supplies decision attributes; does not decide the bucket. First-version attributes and `re_decision_count` kept separately for approval-rate diagnostics. Latest-ever rejected because it restates a mature cohort.

### Channel

**Intake Channel** — SETTLED
Derived structurally: `branch_applicant` present → BRANCH; `applicant` present → digital, refined to MOBILE or WEB by `loan_application.channel`, else DIGITAL_UNKNOWN. Both present: `loan_application.branch_code` breaks the tie (documented NULL for non-branch). Neither present: `loan_application.channel` with `channel_source = 'COLUMN'`, else UNKNOWN. UNKNOWN is a published row with a target of zero, because hiding it stops the breakdown summing to the cohort. The single channel dimension on the funnel fact.
_Avoid_: `credit_assessment.channel` (a copy; checked, never used), `accepted_channel` as a funnel dimension.

**Assisted Acceptance** — SETTLED
Boolean on the funnel fact: `offer_acceptance.accepted_channel` differs from Intake Channel. The accepted channel lives only on the acceptance fact, so there is never a second channel column to mis-sum.

### Publication

**Mature Cohort** — SETTLED
A Cohort Month whose last possible Funnel Window has closed (month end plus 14 days, bank-local). Written once as an immutable fact with run id and input lineage; the artifact a regulator is shown.

**Restatement** — SETTLED
The difference between a Mature Cohort's frozen fact and today's recomputation. Published as a dataset with its own rate; above the restatement threshold a new version is published with a reason, never an overwrite. No new event can change a mature outcome, so the restatement rate measures pipeline lateness.

**Quality Gate** — SETTLED
Two thresholds in the metric contract. Flag threshold, set by the Lending squad as source data owner: above it the cohort publishes with a quality flag column shown on the certified dashboard. Hold threshold, set by Product as metric owner: above it the cohort publishes a HELD row with the reason, so absence is visible. The platform executes and reports against both; it sets neither. The restatement threshold sits in the same contract under the same owners.

**Disagreement Dataset** — SETTLED
Published table of rows where two sources that should agree do not, each row carrying the owning team. Rows stay in the metric; disagreement is a quality signal, not a filter. Disagreements that cross a month boundary carry their own flag, because those move a cohort.

### Identity

**Person Key** — SETTLED (normalisation rule in §6)
Keyed hash of `national_id`. The only identifier NOT NULL in both `applicant` and `branch_applicant`; `whitelist` already uses it as its unique key, so the bank itself treats it as the person. `customer_id` is an attribute of digital applicants only.

## 2. Funnel conversion (analytics question 1)

Metric: of Applications with Submission Anchor in Cohort Month, by Intake Channel, the share whose Drop-off Reason is FUNDED, with the remainder broken down by roll-up bucket. Unit of count is the Application; the fact carries the application key and `prior_application_count`; the person is reached through the Application-Person Bridge (§8, Deletion) so a customer-outcome view can be built separately without the frozen fact ever carrying Person Key.

What the question is really testing: an anchor that cannot drift, a stage definition that survives source disagreement, channel attribution across a cross-channel journey, and a drop-off taxonomy that is mutually exclusive and exhaustive when the brief's five reasons do not map onto the status enum.

Status: SETTLED. Residuals resolved by the Object-Owner Exit Principle.

### 2.1 Conversion diagnostics (brief: approval, offer-acceptance, disbursement rates)

**Stage Rate** — SETTLED
Per-stage rates in the metric contract, each with a named denominator and its own maturity window, published with a still-open share so the breakdown sums, cohorted by Submission Anchor month. The 14-day Funnel Window is the funnel's headline choice and does not leak into stage rates.
- First-decision approval rate: version one approvals over applications with at least one decision version; measures the engine. Final approval rate: Decision of Record approvals over the same denominator; measures the business. An application decided twice counts once in each.
- Offer-acceptance rate: per application, acceptances over applications with an offer, because the business rate is about customers. Offer re-issue rate per Offer Version is a separate diagnostic (re-issued-then-accepted is one acceptance in the first, two issues in the second).
- Disbursement rate: SUCCESS over acceptances (adjacent stage); SUCCESS over offers available as the composite.

**Stage Maturity** — SETTLED
Offers mature at scheduled expiry plus the measured lateness of the expiry job. Decisions mature at a window derived from the observed distribution (the point past which a first decision almost never arrives), set in the contract and reviewed. Each rate publishes once mature and restates under the Restatement rule.

### 2.2 Commercial performance (brief: disbursement amount, funded-loan count, mix)

**Funded Loan (commercial sense)** — SETTLED
The disbursement service owns the commercial fact: money is owned by the service that moved it. Grain is one row per `disbursement`. A funded loan exists when a disbursement reached SUCCESS; amount is `disbursed_amount`; date is `disbursed_at`. Tenor and rate come from the signed offer (`loan_offer.tenor_months`, `interest_rate_pct`) because that is the contract, with the decision's approved values as a published check. Channel is Intake Channel with the Assisted Acceptance flag, same as the funnel, so both products attribute the same way. Amounts conformed to decimal in Silver with the cents unit noted on decision columns.
_Avoid_: application status FUNDED as the commercial definition (that is the funnel's).

**Reversal** — SETTLED
A REVERSED disbursement is a negative row on the reversal date, never an adjustment to the original: a closed month never restates. Monthly funded-loan count is SUCCESS rows minus reversals, each counted when it happened.

Published checks: commercial funded count vs funnel funded count per cohort (the difference is lag plus reversals plus dropped events); `principal_amount` vs `disbursed_amount` (the schema allows them to differ and the brief does not say why).

## 3. Decision reproducibility (question 2)

What the question is really testing: whether the candidate knows the difference between what the sources contained at decision time and what the engine read, and whether they claim the second when they can only prove the first.

**Honesty Boundary** — SETTLED
The platform proves what the sources contained and what the engine copied. It cannot prove what the engine read unless the engine says so.

**Decision Input Record** — SETTLED
One record per (`application_number`, `decision_version`), immutable, with a content hash over the input fields and two timestamps: `decided_at` and `snapshot_captured_at`, so capture lag is on the record rather than assumed away. "For any given application_number" is served by a view defaulting to the latest version and listing the others. Re-derivation from row history runs on a schedule; a hash mismatch is published, never corrected in place. A late-arriving change inside the Read Window opens a new record version flagged `late_input`; the old version stays. Same rule as the cohort freeze: superseded with a reason, never overwritten.

**Evidence Grade** — SETTLED
Per input on the Decision Input Record: ENGINE_SNAPSHOT (decision service emitted its inputs as an event), ENGINE_COPY (field the decision row itself carries), SOURCE_AS_OF (reconstructed from source history), NOT_CAPTURED. The engine's copies are the primary evidence for what it saw; as-of reconstruction is the evidence for what the sources held; the diff is the finding, and its rate is published per input. When an inputs-snapshot event exists it becomes primary and reconstruction becomes the check.

**Read Window** — SETTLED
Per decision version, the span from the earliest input read timestamp (`bureau_check.fetched_at`, `kyc_result.completed_at`, `employment_verified_at`, `income_verified_at`) to `decided_at`. Inputs with a read timestamp are reconstructed as-of that timestamp; applicant profile fields, which have none, as-of `decided_at`. Any source change inside the window sets `ambiguous_input` naming the field. A null read timestamp on a version is a published completeness defect. No invented tolerance.

**Input rules** — SETTLED
- KYC verdict: `kyc_result.overall_status` (coded, not null). `decision_outcome` carried as free text with a mismatch flag, never used as the verdict. The `kyc_result` for a version is the latest `completed_at` at or before that version's `decided_at`, with `kyc_type` carried so a REFRESH between versions is visible as the reason a verdict changed. Because `kyc_request` is unique on (application, type), a second REFRESH overwrites in place and the version the engine saw exists only in change history.
- Bureau score: no engine copy, so SOURCE_AS_OF only: latest `bureau_check` with `fetched_at` at or before `decided_at`, carrying row id and `fetched_at`, ambiguity flag when more than one fetch falls inside the Read Window.
- Employer: decision row `employer_name`/`employer_sector` are ENGINE_COPY; the `applicant` row as-of is SOURCE_AS_OF; diff published.
- Declared income: both candidates carried (`applicant.monthly_gross_income` declared at application; `bank_statement_parse.declared_monthly_income` declared at statement consent). `income_source_used` is evidenced, not guessed, by the **Affordability Back-solve**: the stored `affordability_ratio` is reproduced from instalment and exactly one candidate income. Outcomes: APPLICANT, STATEMENT, AGREE (the two declared values are equal), UNDETERMINED (both reproduce within DECIMAL(5,4) rounding), NONE (neither; a published finding about the engine). The margin is stored so a confirmed formula can re-grade without re-deriving.
  Declines have no instalment, so: `decision_reason_code` not income-related → NOT_DECISIVE; otherwise UNDETERMINED with both candidates SOURCE_AS_OF. When an approved sibling version exists on the same application, its evidenced source is carried as `income_source_inferred`, graded inference never evidence, flagged if a KYC refresh landed between the versions. Back-solving declines from requested terms is rejected: it assumes the engine computes affordability on requested terms, which cannot be shown.

**Decision Input Record: access** — SETTLED
One table, inputs stored clear, column masks by reader group: the analyst role sees income as a band, employer masked, risk flags as a count; the compliance role sees clear. The content hash is over clear values and proves integrity (masked view and clear view are one record). It does not prove masking happened; that proof is mask-definition history plus the access log (§5). The record carries no direct identifier and, being a frozen artefact, no Person Key either (§8, Deletion); the person is reached through the Application-Person Bridge, and resolving a Person Key to an identifier is a logged break-glass read of raw data.

**Legacy decisions** — SETTLED
`loan_decision_v1` rows served as-is, graded NOT_CAPTURED, labelled pre-cutover, retained immutable to at least 2028-08 per the schema's note; reproducibility records inherit the retention. No reconstruction from current rows: that is the one thing a regulator could catch. Never used for the funnel.

## 4. Offer expiry watchlist (question 3)

What the question is really testing: the operational use case from the non-functional requirements. The consumer acts on the data, needs it fresh at a fixed hour, and needs the fields the analyst must never see.

**Watchlist Anchor** — SETTLED
Fixed 06:00 bank-local, regardless of when the job ran; `run_at` recorded. A late run must not change which offers are listed. "Next 24 hours" is (anchor, anchor + 24h]. "Past 7 days" is the seven bank-local calendar days before the anchor day, because ops reads a calendar.

**Expired Without Action** — SETTLED
The offer reached expiry with no acceptance and no earlier exit event on the Application Timeline. Reuses the funnel's exit classification so the two products never disagree. A retracted, withdrawn or soft-deleted offer is closed, not expired, and does not belong on a list whose purpose is a call to a customer who can still say yes; `exit_reason` shows why a case left. Bank-side closures the customer may not have been told about are a second feed from the same timeline, not this list.

**Expiry Pending** — SETTLED
`offer_expires_at` has passed and `offer_expired_at` is still null. Listed at the top: if the source's expiry job has not run, the customer may still be able to accept, so it is the most valuable call of the morning. The gap between the two columns is published as a lateness metric on the disbursement service's expiry job.

**Offer Version** — SETTLED
Watchlist grain is (`loan_offer.id`, offer version from change history), because re-issue overwrites the single row per application. A re-issue after expiry is two rows across two mornings linked by the id: one offer id, two offers for a list that prompts action.

**Freshness** — SETTLED
Daily computation at the anchor with `data_as_of` on every row and a stated freshness target; change capture is continuous so the data is minutes old. The ops view anti-joins acceptances that landed after the run, so a 05:55 signature removes the row before ops opens the list. The one near-real-time stream in the design is the FAILED disbursement alert.

**Watchlist Snapshot** — SETTLED
Every morning's list persisted immutable with `run_id`, `anchor_at`, `data_as_of` and the suppression applied, so anyone can later show what ops was told. Joined later to outcomes (accepted after listing or not) so the list's effectiveness is measured, not assumed.

**Watchlist access** — SETTLED
Watchlist and funnel read one governed source. The ops role sees name and `mobile_number` through column-level access; the analyst role sees masked values; the access log records which principal read which column in which query, so the watchlist is evidence the access model works, not a leak path. The leak path is export: downloads from the ops view are disabled or logged under the same identity. A token-lookup design is rejected: friction with nothing the log doesn't already give.

Confirm: whether acceptance remains possible between scheduled and actual expiry (EXPIRY_PENDING is top of the list regardless). Ops consumes a governed view; a file only as a logged export if asked.

## 5. PII masking proof (question 4)

What the question is really testing: whether the candidate answers with an assertion ("we have masking policies") or with evidence that survives a hostile check. "Demonstrate", "before any analyst accessed" and "for the funnel report in Q1" each carry a trap: enforcement vs definition, ordering in time per principal, and scope.

**Evidence Classes** — SETTLED
Every item in the proof pack is labelled one of three. Assertion: policy and mask definitions. Evidence: the access log (of access) and the Probe (of enforcement). Inference: the join of access log to mask-definition history, because the log records principal, statement, tables, columns and time, not which mask fired. (To verify: exactly which events the catalog's audit tables record for mask DDL.)

**Probe** — SETTLED
A synthetic principal in the analyst group runs a fixed query against every masked column daily; the output is hashed and written to the Evidence Store with the run time. For every day of Q1 there is a stored masked output whether or not a real analyst queried. The compliance owner reviews probe output monthly and the review is logged: a probe nobody reads is not a control. Whether a regulator accepts a synthetic probe as enforcement evidence is an assumption to confirm with compliance.

**Negative Proof** — SETTLED
Two halves. The door was locked: deny-by-default grants with grant history showing no analyst principal held a raw-layer privilege in Q1, resolved against group membership history at grant time. Nobody walked through: the access log scan for any analyst principal against any raw object in Q1, expected empty, with the break-glass log listing every exception and its ticket. Covers the platform team unasked: pipelines run as service principals with no interactive login, and every human read of raw data in the quarter is listed with justification. A long list is a finding about the platform team and belongs in the pack.

**Analyst (for the proof)** — SETTLED
The role that executed the query, not the person. Column masks test group membership at query time, so the enforceable question is whether the mask was attached at the time of every query. For the narrative, group membership change events are materialised into a versioned dimension under the platform's own retention, because the identity provider's retention is not the platform's to set.

**Proof Scope** — SETTLED
Every analyst read, in Q1 calendar time, of any table in the input lineage recorded by the Mature Cohort runs for January to March. The frozen lineage was written at run time, not reconstructed, which is what makes the scope defensible; the catalog's lineage is the independent second lens. The aggregate artefacts themselves are included and noted as containing no PII by construction. Reading the scope as "the report only" is a dodge; reading it as "every table in Q1" is unbounded.

**Derived Copies** — SETTLED
Analysts create persistent objects only in a per-analyst sandbox schema, so a copy made from a masked table holds only what the copier could see. Detective control: lineage on every derived object plus a scheduled scan of sandbox schemas for identifier patterns, hits published. BI tools read with the viewer's own identity, no imported extracts of masked columns; downloads from governed views are logged. Stated limit: a screenshot or a typed value cannot be proven, and the pack says so.

**Evidence Store** — SETTLED
Audit logs delivered to object storage in a separate security-owned account with write-once retention, no platform-team write access, retained at least to 2028-08 (the horizon the legacy table's note implies) and proposed longer. The catalog's own audit tables are the working copy, not the evidence copy (to verify: their retention, believed around a year). A hash chain is unnecessary when the store is write-once.

**Person Key custody** — SETTLED
The hash key is a control: held in a security-owned secrets store under a customer-managed key, readable only by the pipeline service principal, never by a human. Every table carrying Person Key carries `key_version`. Rotation is compromise-driven, not scheduled, because it re-keys every row from raw and republishes; the procedure is documented and rehearsed once. Joining Person Key back to `national_id` is a raw read, so break-glass, ticketed, and listed in the Negative Proof.

**PII Classification** — ASSUMED (one confirm item with the data protection officer covering income, employer, pseudonymised keys, whitelist non-applicants and document paths)
Direct identifiers (national id, name, mobile, email, date of birth) are pseudonymised in Silver. Income, employer and KYC risk flags are sensitive attributes: stored clear, column-masked by role. Pseudonymised data is still personal data under the Data Privacy Act (as understood), so Person Key is governed as such; key custody and the raw-layer deny are the two controls.

**PII outside the named columns** — SETTLED
- Whitelist non-applicants: the funnel needs only people who applied, so non-applicant rows never leave the raw layer; Silver holds Person Key, enabled, limit and product codes with no name or mobile. Their consent and retention is the strictest position in the data, assumed to follow the whitelist's own consent rather than the lending horizon.
- Blobs (`bureau_response_json`, `metadata_json`, `schedule_json`): raw-layer only, break-glass, never promoted until parsed into classified columns.
- Document paths (`document_upload.s3_path`): treated as PII because the path may embed an identifier; masked in Silver. The object store it points at is outside the platform boundary; the platform never copies documents, and the decision log says the store needs the same access model under its own owner.
- Risk flags (`kyc_result.risk_flags`): the most sensitive attribute present. Analysts see a count, compliance sees the flags.

## 6. Identity across the four services

Summary: the application number is the only cross-service join key; every service-local UUID is an attribute whose agreement is measured, never a key; the person is the normalised national id under a keyed hash.

**Application System of Record** — SETTLED (stated as an assumption, supported three ways)
The application service defines whether an Application exists: `application_number` is UNIQUE only in `loan_application`; the KYC service's own comment calls its copy a foreign application number; the brief names the application service as the first step in the funnel.

**Orphan** — SETTLED
A `kyc_request`, `credit_assessment` or `loan_offer` whose application number has no `loan_application` row. Kept in Silver, excluded from the funnel denominator, published as a defect to the team that wrote it (either a number the application service never issued, or ahead of a capture gap). Reproducibility still answers for an orphaned decision, flagged application-unknown, because a regulator asks by application number and does not care which service dropped the row.

**Join Normalisation** — SETTLED
Trim and uppercase before every join on application number; rows that join only after normalisation carry `join_normalised = true`, rate published per service. The number is validated against the pattern the schema comment gives (prefix, date, five-digit sequence) and outliers published rather than guessing other valid forms. The rule is part of the key definition and versioned with it.

**Service-local UUIDs** — SETTLED
`credit_assessment.applicant_id`, `loan_offer.borrower_id` and the UUID inside `kyc_request.user_ref` (`CUST-` stripped) are attributes, never identity. Each is tested from data (a UUID repeating across applications is a customer id; one that never repeats is an applicant id), joined to both candidates, match rate published, and disagreement with the application service's `customer_id` published with no effect on the join. `applicant.id` is rejected as a person key because it is a per-application surrogate and fragments a repeat customer. For branch applications these columns are NOT NULL yet the application service recorded no UUID, so what they hold is tested from data and confirmed with owners; if it matches nothing, a service outside the four mints customer identity (see Customer Master Gap).

**KYC scoping** — SETTLED
KYC belongs to the Application through `application_reference` only. The KYC service's uniqueness on (reference, type) says it scopes requests to the application, so a FULL from a prior application never counts for this one; REFRESH exists precisely to reuse prior KYC under this application's reference.

**Person Key normalisation** — SETTLED
Trim, uppercase, strip separators, then keyed hash; `key_version` covers both the secret and the rule. Published: the count of raw values that collapsed into one key (with the collapsed set reviewable, because normalisation can merge two legitimately different identifiers as easily as reunite one person) and outliers against the dominant pattern derived from the data. No national id format is assumed. Up-front check: whether `whitelist`, unique on this column, holds values differing only by formatting; if so the application does not normalise and the collapse rate will be material.

**Product** — SETTLED
A conformed product dimension with a versioned mapping table owned by Product from each service's form (`loan_application.product_code`, `whitelist.product_codes` split into a bridge, `product.product_key`, free-text `loan_offer.loan_product` by exact string, unmapped published). The funnel uses the application's product because the funnel starts before any decision exists; commercial mix uses the decision's product via `product_id` because that is what was offered and funded. Disagreement between the two is published and is also a business event (applied for one product, approved for another).

**Pre-approved** — SETTLED
An enabled `whitelist` row for the Person Key at the Submission Anchor whose product codes include the application's product, resolved from change history (the table has no history of its own). `pre_approved_limit` at the anchor and the ratio of requested amount to limit are carried as attributes; their analysis is out of scope, the attributes are not. "At any time" rejected: it restates.

**Customer Master Gap** — SETTLED
No service in the brief owns `customer_id`. Person Key is the person; `customer_id` is an attribute. One Person Key with several customer ids, or one customer id with several national ids, are published to the application team as defects with no resolution in the platform. A customer dimension stub keyed on `customer_id` with a source column is the seam, so a customer master plugs in without reshaping the facts.

## 7. Source disagreements

SETTLED principles:
- Publish, don't withhold: a disagreeing row stays in the metric and lands in the Disagreement Dataset with an owner. Refusing to count until sources agree makes the metric hostage to the worst integration.
- Gate at the aggregate, not the row: see Quality Gate.
- Quarantine is rejected for the funnel because it changes the denominator and the funnel stops reconciling to application counts.

Registered checks and owners:
- Submission Anchor vs first SUBMITTED history row, beyond a mechanical tolerance (minutes; the two writes should share a transaction, to confirm). Owner: application team.
- Funded (application service) vs `disbursement_status = 'SUCCESS'`, both directions. Owner: Lending squad.
- `loan_application.channel` vs applicant-table presence; both tables present; neither present. Owner: application team.
- `credit_assessment.channel` vs Intake Channel.
- Status OFFER_RETRACTED vs `offer_expired_at` (agreement surfaced as `retracted_on_expiry`).
- Fallback rates for every `*_source` provenance column.
- Commercial funded count vs funnel funded count per cohort; `principal_amount` vs `disbursed_amount`; decision product vs application product; offer tenor/rate vs approved tenor/rate.
- Skew between `decided_at` and the change-capture commit timestamp (measured, not asked); whether `is_deleted` ever flips back and whether `offered_at` moves (measured, not asked).

**Enum Registry** — SETTLED
Every enumerated column gets a registry row seeded from the schema comments, owned by the source team, versioned, acknowledged by the team. Enforcement is detection, because an acknowledgement does not stop a deploy. An unknown value classifies as UNMAPPED, visible in the breakdown so it still sums, published as a contract violation, and its share passes through the Quality Gate so a new terminal status cannot silently inflate a bucket for long.
_Avoid_: failing the pipeline (a late cohort hides the drift from the people who need it); mapping unknowns to IN_PROGRESS (wrong and quiet).

**Operational Streams** — SETTLED
The platform is downstream of decisions and never upstream. Near-real-time scope: FAILED disbursement (a customer is waiting for money), EXPIRY_PENDING to the morning list, and a severity-ranked disagreement alert whose top severity is a disbursement after a DECLINED decision version. Serving features to the credit engine is recorded as a possible future use with one consequence: the moment the platform's output becomes an input, the Decision Input Record must snapshot it like every other input and the platform's own lag becomes part of the engine's evidence.


## 8. Gaps in the brief and assumptions made

### Design dependencies (ASSUMED, stated in the decision log)
- Change capture delivers every row change with before-and-after images, so first-observed values can be pinned, rewrites detected, status changes carry a commit timestamp even when `application_status_history` is incomplete, and tables with no history of their own (`whitelist`, `kyc_request` REFRESH overwrites, the single `loan_offer` row) can be read as-of. If capture started after go-live, the initial load supplies the first value.
- Event propagation lag is seconds to minutes, immaterial against a 14-day window. A dropped event is what the Funded reconciliation check catches.
- The funnel starts at the first month with complete change capture; the brief's example application number is dated 2026, after the 2025-08 decisioning cutover.
- Cohort Month and the Watchlist Anchor are bank-local (Asia/Manila). Timestamps are held in UTC with the source zone recorded per service. Until each service's write convention is confirmed, the published exposure is the count of applications whose UTC date and bank-local date fall in different months (the last eight hours of each UTC month). The application service's convention can be tested from the date embedded in `application_number` against `created_date` near midnight.
- A regulatory retention obligation takes precedence over an erasure request for the retention period.

### Retention and deletion — SETTLED
Per layer. Raw is retained to the regulatory horizon with no deletion, because it is the evidence. Decision Input Records sit under the same basis and carry no direct identifier. Silver and Gold are subject to deletion requests. A single horizon for all lending data is rejected: the whitelist non-applicants and the frozen aggregates sit at opposite ends of the obligation.

**Application-Person Bridge** — SETTLED
Person Key lives in no frozen artefact. Frozen facts (Mature Cohort, Decision Input Record, Watchlist Snapshot) carry the application key only and never change; a bridge table maps application to Person Key. A deletion request hashes the supplied identifier under every `key_version`, deletes the bridge rows and the person-level Silver rows, and tombstones personal attributes in non-frozen tables. Aggregates are untouched and were never personal.

### The two forced choices in the decision log — SETTLED
- Rejected trade-off: refusing to count a row until the sources agree. It looks rigorous, it makes the metric hostage to the worst integration, and a referee who will not publish is not a referee. Replaced by the per-row canonical source, the Disagreement Dataset with an owner per disagreement, and the two-threshold Quality Gate at cohort grain. Bucket-rank precedence appears as a rejected option inside the funnel design, where the two scenarios that broke it are the explanation.
- Confirm item: whether the decision service can emit an inputs-snapshot event keyed on (`application_number`, `decision_version`). It changes the Evidence Grade of an entire deliverable and is incremental for the owner because the decision row already copies four inputs. Runner-up, one line: who mints `customer_id`, since branch applicants carry one in the KYC and disbursement services but not in the application service, and no customer master is among the four.

### Confirm with source owners (carried, none blocking)
- Operational meaning of OFFER_RETRACTED and whether CONTRACT_SIGNED echoes `offer_acceptance` (the timeline rule is robust to either reading).
- Completeness of `application_status_history`; whether `submitted_at` and the SUBMITTED history row share a transaction (sets the tolerance).
- Each service's timestamp zone; actual event lag.
- Affordability formula and what instalment the engine uses for declines (ships with UNDETERMINED and NOT_DECISIVE).
- Whether acceptance remains possible between scheduled and actual expiry (EXPIRY_PENDING is top of the list regardless).
- What `applicant_id` and `user_ref` hold for branch applications; whether a product mismatch between application and decision is operationally normal.
- Data protection officer, one item: classification of income, employer, pseudonymised keys, whitelist non-applicants, document paths; and whether regulatory retention overrides erasure.
- Compliance: whether a synthetic Probe is accepted as enforcement evidence (delivered regardless; it costs one job).

### Moved from ask to measure
- Whether a soft-deleted offer is re-issued on the same row: change history shows whether `is_deleted` flips back and whether `offered_at` moves; the design handles both.
- Whether `decided_at` is write time or commit time: skew against the capture commit timestamp is computed and published; the Read Window uses evidenced timestamps, so skew matters only if large, and then it is a finding.

### Settled in the sweep
- Ops consumes a governed view. A file only as a logged export if ops asks, and the ask is recorded.

### Vendor facts to verify in the engineering round
- Which audit events the catalog records for mask DDL; audit-table retention (believed around a year, so not the evidence copy).

## 9. Deferred: stack / engineering round

Business semantics closed on Thi's side after round 7. Opens when Thi calls it, after the vendor facts above are verified.
