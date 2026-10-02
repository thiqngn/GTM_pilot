# Tricky parts of the schema and brief (planning material, not for submission)

One line per trap, with the table or column that proves it. Grouped by where it bites.
Status: identified across grill rounds 1–4 and the planning sessions, as of 2026-10-02.

## Cohort and stages
- `loan_application.submitted_at` is nullable on a mutable row; `last_submitted_at` moves on every edit. Anchor needs pinning and a provenance column.
- `application_status_history` has no sequence column; order is `changed_at` then `id`, and same-timestamp ties are possible.
- Nothing guarantees `application_status_history` is complete; a missing SUBMITTED or FUNDED row silently drops a cohort member or a conversion.
- No `funded_at` column anywhere; FUNDED exists only as a status, a history row, or a `disbursement` SUCCESS.
- `disbursement` is one-to-many on `loan_offer_id` with no uniqueness, so PENDING→FAILED→SUCCESS retries and REVERSED rows coexist; the brief never says whether REVERSED un-funds.
- Pre-submission `APPLYING`, `APPLYING_EXPIRED` and `WITHDRAWN` rows have no anchor and sit outside the funnel, but the business will ask for them (started-to-submitted rate).
- OPEN confirm item: `last_submitted_at` is documented as "most recent re-submission (after edit)", which implies an application can return to APPLYING after submission. If so, `APPLYING_EXPIRED` can occur with `submitted_at` set, belongs in the cohort, and needs its own fine code (clock said no). CONTEXT.md currently records "cannot occur post-submission" as settled; it should be reopened.
- 14-day window near month end: a cohort is not mature until month end plus 14 days; publishing early restates.
- All `DATETIME` columns carry no zone across four services; cohort month, the 14-day window, "each morning" and "Q1" all depend on an assumed zone. Manila is UTC+8, so the ambiguous window is eight hours, not one.
- `application_number` embeds a date (AURPL-YYYYMMDD-NNNNN): a volume cap (99,999/day) and a free timezone test against `created_date`.
- Re-decisioning: `credit_assessment` is UNIQUE on (`application_number`, `decision_version`); v1 APPROVED then v2 DECLINED must not restate a mature cohort.

## Reasons and the status enum
- The brief's five reasons (decline, withdrawn, expired, failed-disbursement, offer-not-accepted) do not map onto the `status` enum; no "offer expired" or "disbursement failed" status exists.
- `OFFER_RETRACTED` is in the enum, absent from the brief, and may be the application-side echo of offer expiry; `offer_expired_at` at or before the retraction `changed_at` is the test.
- `CONDITIONALLY_APPROVED` is a stage, not an outcome; nothing says what it means operationally.
- `CONTRACT_SIGNED` has no documented link to `offer_acceptance`; assumed echo, confirm item.
- `WITHDRAWN` can happen before an offer, after an offer, or after acceptance; only timeline state distinguishes them.
- Expiry then re-decision (v2 DECLINED on day 12) and decline then re-approval both break any bucket-rank precedence; reasons must be ordered by event time with re-entry.
- `loan_offer.is_deleted` flipping with no acceptance is a bank-initiated exit with no application status at all.

## Offers and disbursement
- `loan_offer` is UNIQUE on `application_number` regardless of `is_deleted`; a re-offer must update the row in place, so offer history exists only in CDC.
- `offer_expires_at` is documented as "config: offered_at + 7d"; the config can change, so expiry must be read from the row, not recomputed.
- `offer_expired_at` is "actual expiry if customer never acted"; NULL is ambiguous between accepted, still open, and expiry job not run.
- The watchlist (question 3) is an operational feed with a morning SLA, not analytics: expiring in next 24h and unaccepted, plus expired without action in the last 7 days; `is_deleted` must be honoured and `offer_acceptance` absence is the test.
- `disbursed_to_account` is already a masked account number and `mandate_account_hash` / `account_number_hash` are already hashed: PII arrives pre-masked inconsistently, so the proof must say which columns were masked where.
- `credit_assessment.mandate_debit_day` / `mandate_account_hash` duplicate `mandate.debit_day` / `account_number_hash`: another copy across services that can disagree.

## Decision reproducibility
- "Even if the source records have since changed" (brief Q2) is only answerable with append-only capture; and the platform can prove what sources contained, not what the engine read (Honesty Boundary).
- Declared income has two homes: `applicant.monthly_gross_income` (at application) and `bank_statement_parse.declared_monthly_income` (at statement consent); which one the engine used is not recorded.
- Employer has two homes: `applicant.employer_name` and `credit_assessment.employer_name` / `employer_sector` (engine copy); `branch_applicant` has none.
- `bureau_check` has no `decision_version` link; join is latest `fetched_at` at or before `decided_at`, ambiguous when two fetches fall in the read window.
- `kyc_request` is UNIQUE on (`application_reference`, `kyc_type`): at most one FULL, REFRESH, RE_SUBMIT per application, and a second REFRESH overwrites in place.
- `kyc_result` has no unique constraint on `kyc_request_id`; several results per request are possible, and `completed_at` is nullable.
- `kyc_result.decision_outcome` is free text that "sometimes differs from" `overall_status`; never the verdict.
- `risk_flags` is pipe-delimited (`PEP|ADVERSE_MEDIA`) and carries sensitive categories.
- `decided_at` defaults to CURRENT_TIMESTAMP: write time, not necessarily commit or read time; confirm item.
- `decision_reason_code` is undocumented; `affordability_ratio` is stored but the formula is not. The Affordability Back-solve is a design-doc diagnostic only (an unreproducible ratio is a finding about missing inputs); it stays out of the decision log.
- `loan_decision_v1` is deprecated with a different schema (`approved_amount` DECIMAL vs `approved_amount_cents`, no version, no reasons), retained to 2028-08: served as-is, graded NOT_CAPTURED, never used for the funnel.
- Reproducibility horizon: only decisions after capture began; the brief's retention note implies a three-year regulatory horizon.

## Identity and cross-service joins
- No shared FKs across the four databases; every join is by `application_number` / `application_reference`, a string with no referential guarantee.
- `applicant.customer_id` is non-unique; `kyc_request.user_ref` is `CUST-<uuid>` and must be prefix-stripped.
- `branch_applicant` has no `customer_id`, yet `loan_offer.borrower_id` is NOT NULL for branch applications; `borrower_id = customer_id` is unconfirmed.
- `credit_assessment.applicant_id` is VARCHAR(36) while `branch_applicant.id` is BIGINT; what it references for branch applications is unknown.
- `national_id` is the only identifier NOT NULL in both applicant tables and the `whitelist` unique key; its normalisation before hashing is open.
- FK direction runs from applicant tables to `loan_application`, so an application with no applicant row, or with both, is legal; `branch_code` on `loan_application` is the tie-breaker.
- Channel lives in four places (`loan_application.channel`, applicant-table presence, `credit_assessment.channel`, `offer_acceptance.accepted_channel`) with no guarantee of agreement.

## Product and reference data
- Four product identifiers with no shared key: `whitelist.product_codes` (comma-separated), `loan_application.product_code`, `product.product_key`, `loan_offer.loan_product` (free-text display name).
- Units differ by service: `*_cents` BIGINT in decisioning, DECIMAL(12,2) in application, DECIMAL(15,2) in disbursement; interest as basis points in one and percent in another.
- `list_of_value` is a generic typed lookup with a self-FK hierarchy, `metadata_json` blob and an `enabled` flag; `applicant.marital_status` is free text "see list_of_value".
- Blob columns: `bureau_response_json`, `installment_schedule.schedule_json`, `list_of_value.metadata_json`, plus delimited `risk_flags` and `product_codes`.
- `whitelist` is a hidden funnel stage: pre-approved people who never apply; `enabled` and `pre_approved_limit` make whitelist-to-application conversion the natural fifth question.

## PII and governance
- PII is far wider than the brief's two examples: `national_id`, `mobile_number`, `first_name`/`last_name`, `date_of_birth`, `email`, `employer_name`, `document_upload.s3_path`, `bureau_response_json`, `staff_id`, `changed_by`, PEP inside `risk_flags`, and the pre-hashed account columns.
- Question 4 asks for proof after the fact ("for the funnel report in Q1"): evidence of access, masking state at the time, and lineage, not a description of configuration.
- The person key is a keyed hash, so key custody and rotation are part of the masking proof; rotation breaks historical joins.
- Soft deletes (`is_deleted`, `whitelist.enabled`, `list_of_value.enabled`) mean "deleted" rows persist and must be handled in every layer.
- `document_upload` has two completion timestamps (`verified_at` auto, `verification_completed_at` manual) and a `MANUAL_REVIEW` status: human-in-the-loop timing.

## Brief and deliverables
- The NFR names real-time and near-real-time "for operational monitoring and decisioning" with no concrete use case; the platform is not in the decision path, so NRT must be justified from the schema (FAILED disbursement alert) or declined.
- The decision log is limited to 1–2 pages; mechanism belongs in the design docs, one sentence of principle per item in the log.
- "You can make any assumptions" but every one must be written down; the regulator and timezone assumptions carry the most risk.
- The document mixes Philippine (GoTyme) and Vietnamese signals (a Vietnamese word in the stack table, "CIC" exists in both); state the regulator assumption as a one-line change.
- The schema example dates (2026) sit after the 2025-08 decisioning cutover, so the funnel is post-cutover only.
- Different teams already produce different conversion numbers; the platform is being asked to referee, so ownership of definitions is part of the deliverable, not an aside.
