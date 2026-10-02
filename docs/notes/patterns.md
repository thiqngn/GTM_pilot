# Reusable patterns (planning material, not for submission)

Each pattern: what I built / the problem it solved / where it lands in the lending schema.
Submission-safe wording: no company, system or table names from past work. Lending table and
column names are fine. Verified evidence only; demo and interview-prep material excluded.

Specific figures (dollar amounts, record counts, level counts) stay in this file. In the
submission and in interviews, describe magnitude without numbers.

## 1. An absent reading is not a new version
**Built:** Version-retaining SCD2 keyed on the business key, with a grace period before a
disappearance closes a version, and a carry-forward that marks every carried row with
provenance and age. Settled rule: carry indefinitely, expose age, mark the row.
**Problem:** A source API intermittently returned a record with its key present and every
business column blank. Overwrite-in-place SCD2 destroyed the last good values, and the record
silently dropped out of a report that filtered on one of the blanked flags.
**Lending:** Silver pins the first observed non-null `submitted_at` and treats a later null or
rewrite as a published defect, not a new anchor. The single `loan_offer` row per application
(UNIQUE on `application_number`, soft delete via `is_deleted`) gets its history from CDC
versions. Every fallback timestamp carries a `*_source` provenance column.

## 2. History you do not capture cannot be reconstructed later
**Built:** Nothing, which is the point. I had to ask a vendor for a history of record
create/delete events, flag toggles and line changes to diagnose sync failures, because the
source kept none and the pipeline had not kept its own.
**Problem:** Without an append-only record of what arrived and when, every incident became an
argument about whether the source was wrong or the pipeline broke it.
**Lending:** Append-only Bronze with CDC operation type and commit timestamp is non-negotiable.
It is the only thing that makes "even if the source records have since changed" answerable,
and the only history for `kyc_request` (UNIQUE on application and type, so a REFRESH overwrites
in place), `loan_offer`, and `submitted_at`. State the cost of starting capture late: anything
before the first captured image is NOT_CAPTURED and labelled so.

## 3. A metric is a dated contract with an owner
**Built:** A header block stating source, measure, scope, grain, views and trigger, agreed with
the consuming team on a stated date. Every number the alert published reconciled to it. Each
alert card named the responsible owner, read live from the owning system.
**Problem:** The same measure was being summed three different ways by three teams, and an
alert built on any one of them would have been disputed on day one.
**Lending:** The funnel definition, the 14-day parameter, the reason-code roll-up, the flag and
hold thresholds and the restatement threshold live in one metric contract. Product is metric
owner, the Lending squad is source owner, the platform executes and sets neither.

## 4. Change the definition, publish the diff, number the assumptions
**Built:** Moved revenue timing from close date to contract start date, annualised short
renewable lines and re-attributed region to the parent account. Shipped a before/after impact
diff (+$121M across 793 records) rather than a new number. Separately migrated product
attribution to line items as the single source of truth under three numbered inclusion
assumptions; a $281 residual was traced to one exchange-rate difference and written down.
**Problem:** One metric's timing rule moved a deal into a different quarter from the other
metrics. The fix was a written rule and a published impact diff, so every consumer could see
which records moved and why.
**Lending:** Every version of the roll-up mapping, the 14-day parameter or the channel rule
ships with a cohort-level before/after diff. Inclusion rules (never-submitted excluded, test
applications excluded, pre-cutover excluded) are numbered assumptions in the decision log, and
residuals are traced, not rounded away.

## 5. One source of truth per fact; every other source becomes a check
**Built:** Declared line items the single source for product attribution and demoted the header
field to a reconciliation check, instead of blending the two.
**Problem:** Two places held "the same" value with different grain and timing, and blending them
produced numbers nobody could trace to either.
**Lending:** `application_status_history` is the source for funnel stages; `disbursement` is
the money truth and a reconciliation check, never a second definition of FUNDED.
`loan_application.channel` is the dimension; `credit_assessment.channel` and `accepted_channel`
are checks or attributes. `kyc_result.overall_status` is the verdict; `decision_outcome` is
carried free text with a mismatch flag.

## 6. Disagreement is a published dataset, not a blocker
**Built:** Mismatch tables with an owner on every row: stage versus forecast category, quote
lines versus CRM lines, two bookings tables that should agree. Header-versus-detail amounts
disagreed on 138 of about 28k records, roughly $34.7M in aggregate; published, not resolved
silently. The number still shipped; disagreement was a quality signal, not a filter.
**Problem:** Refusing to publish until sources agreed would have made the metric hostage to the
worst integration. Quietly picking one side hid the problem from the team that could fix it.
**Lending:** FUNDED without SUCCESS and SUCCESS without FUNDED; anchor versus first SUBMITTED
row; channel column versus applicant-table presence; retraction versus `offer_expired_at`.
Amounts are expected to disagree across hops (`requested_amount`, `approved_amount_cents`,
`principal_amount`, `disbursed_amount`); each hop gets a tolerance and the mismatch is published
with an owner per disagreement type. Gate in aggregate with two thresholds, never per row.

## 7. Every unit of movement gets a cause, and "unexplained" is never dropped
**Built:** Detected every possible cause per moved record, resolved them to one winning bucket
by a root-cause-first priority (twelve levels), kept the rest as contributing causes, proved the
buckets reconciled to total movement, and kept an explicit "unexplained" bucket.
**Problem:** Causes overlapped heavily; most moved records tripped many detectors that were
echoes of one root event. Without a precedence rule the explanation was wrong in a plausible way.
**Lending:** Fine drop-off codes derived from the application timeline, exit and re-entry
events, first exit after last re-entry at cutoff, and the requirement that funded plus buckets
plus in-progress sum to the cohort. The roll-up to the brief's five is a mapping, not logic.

## 8. Refresh-anchored vintages and point-in-time replay
**Built:** Compared snapshots anchored to the data's own refresh stamp rather than the clock,
and reproduced a transient alert to the minute by replaying the warehouse at an instant. Found a
race between a recalculation write and a report run; nothing was lost, only mis-bucketed.
Learned the hard way that alert run IDs were in local time while the warehouse was UTC.
**Problem:** Clock-anchored comparisons lied when a report froze, and intraday swing exceeded the
alert threshold many times over.
**Lending:** Cohorts are written once at maturity with run id and lineage; today's recompute is
diffed against the frozen fact and the diff is the restatement dataset. Decision reproducibility
is "inputs as of `decided_at`" with capture time on the record. Cohort month in bank-local time,
timestamps stored in UTC with the source zone recorded.

## 9. Guards fail visibly, gates compare like with like, every alert has a fixer
**Built:** Found that an informational-severity guard logged as success and hid a freeze. A
deploy gate compared against a stale backup during a sync lag and rolled back for the wrong
reason; re-gated against a same-vintage control run. Freshness checks on source tables against
a six-hour threshold, amount mismatches above a one-dollar tolerance, each alert routed to the
team that fixes it, with de-duplication so an unresolved condition does not re-fire every hour.
**Problem:** A control that cannot fail is not a control, and an alert with no reader is noise.
**Lending:** A held cohort publishes a HELD row, never silence. CDC lag alert goes to the
platform team; amount tolerances per hop go to the owning service team; the FAILED disbursement
stream goes to ops. Alert plumbing is itself monitored.

## 10. A transition log needs explicit sentinels for the impossible cases
**Built:** Rebuilt stage history from two systems into one transition log with `valid_through`
by LEAD and the clock stopping on terminal states. For closed records with no recorded close,
chose per case between the real prior stage, a named sentinel, and an honest NULL, and wrote
down why.
**Problem:** Migrated records had single-row closes with no journey, and live records said
closed while the trail never recorded it. Guessing either way would have been wrong silently.
**Lending:** Missing SUBMITTED or FUNDED rows in `application_status_history`; UNKNOWN channel
as a published row; DIGITAL_UNKNOWN rather than a guess at MOBILE or WEB; status history ordered
by `changed_at` then `id` because there is no sequence column. Open confirm item: whether an
application can return to APPLYING after submission (`last_submitted_at` is documented as
"after edit", which implies it can), in which case `APPLYING_EXPIRED` can occur with a
`submitted_at` set and needs its own fine code inside the cohort rather than exclusion.

## 11. Business key over system ID; same name is not the same entity
**Built:** A quoting tool's API returned different system IDs for the same quote number, so I
asked the vendor whether the quote number could be canonical and keyed history on the composite
business key instead of the regenerating surrogate. Separately: lookups joined on a
space-stripped key collided ("Healthcare" and "Health Care"), and split copies of a field
diverged from the combined source field they were derived from.
**Problem:** System IDs regenerate, copies drift, and two fields with the same name in two
systems are not the same entity until someone proves it.
**Lending:** `application_number` over `loan_application.id` as the cross-service key.
`borrower_id` and `credit_assessment.applicant_id` are unconfirmed until an owner says what
they reference (`borrower_id` is NOT NULL for branch applicants who have no `customer_id`;
`applicant_id` is VARCHAR(36) while `branch_applicant.id` is BIGINT). Strip the `CUST-` prefix
from `kyc_request.user_ref` with the rule written down. Channel copied across three services is
a check, not a second dimension. Person key is a keyed hash of one declared identifier.

## 12. Evidence states its own gaps and exclusions
**Built:** A field-retirement audit with two independent lenses (downstream consumers scanned
comment-stripped and alias-aware; staleness from audit tables) that said per field where history
was unavailable, and gave a verdict only when both lenses agreed. In a diff write-up, excluded
one placeholder category because of a known sync issue and said so in the first paragraph.
**Problem:** An evidence pack that hides its blind spots gets caught by the first auditor who
asks "and what about the period before that?" A quiet filter looks like a finding when someone
else reruns the query.
**Lending:** The regulator proof combines access logs, masking state at the time, and lineage,
and states that it covers only the period after CDC and audit capture began. Test applications
and known-bad periods are excluded by a documented, numbered rule, never a quiet WHERE clause.
Legacy `loan_decision_v1` is served as-is and graded NOT_CAPTURED, never reconstructed.
