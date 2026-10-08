# Task 4. Tasks for each system

Large tasks for planning the work described in [ADR.md](ADR.md) (rates for the bank and partner call centers). The "Roadmap" column gives the month on the [roadmap](roadmap.png) (M1 = November 2026). "Depends on" lists the tasks that must be finished first. Estimates are rough team-weeks for planning and must be checked with each team.

## 1. ABS (ABS team)

No new functions in the ABS. Rate approval and publication are delivered by the Task3 MVP (UC9).

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| ABS-1 | Extend the `deposit.rates` event contract: rate set version, approval time, number of items in the set, product conditions (payout, capitalization, top-ups, partial withdrawal, early closure rate). | AsyncAPI contract agreed and in the schema registry. | 1 | — | M1 (before MS1) |
| ABS-2 | Publish the extended event from the ABS outbox through the ABS Integration Adapter. | Complete rate sets reach Kafka. | 1 | ABS-1, Task3 rate management | M2–M3 (MS2) |
| ABS-3 | Take part in end-to-end tests: approve, correct and re-approve rates; check freshness in both call centers. | Test report. | 0.5 | CR-3, EXP-5, CCS-4 | M4 |

## 2. Catalog & Rate Service and API Gateway (Internet Bank team)

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| CR-1 | Detect complete rate sets in the Rates Kafka Consumer (version + number of items) and build the versioned call center snapshot (public products, rate grid, conditions, `valid_until`, no personal or special rates). | Rate Snapshot Builder; snapshot in the Catalog Cache. | 2 | ABS-1, Task3 Catalog & Rate Service | M3 |
| CR-2 | Call Center Rates API `GET /v1/call-center/rates` with `ETag` / `304`; OpenAPI 3 contract; contract tests. | API published in the test environment. | 1.5 | CR-1 | M3 |
| CR-3 | Snapshot Publisher to the compacted topic `deposit.rate-snapshot` (idempotent producer, schema in the registry, contract tests). | Snapshot events in Kafka. | 1 | CR-1, PLT-1 | M3 |
| CR-4 | API Gateway: route, OAuth2 client for the Call Center System (scope `rates.read`), mTLS client certificate, rate limit 1 request/s. | Call Center System can call the API. | 0.5 | CR-2, SEC-2 | M3 |
| CR-5 | Observability: snapshot version and rate age per consumer, alerts on lag > 30 min. | Dashboards and alerts. | 0.5 | CR-2, CR-3 | M4 |

## 3. Rate File Export Service (Internet Bank team, new container)

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| EXP-1 | Service skeleton on the Task3 platform (.NET 8 worker, CI/CD, 2 DCs, OpenTelemetry, health checks); schema `rate_export` in the Deposit Services DB. | Empty service deployed in test. | 1 | Task3 platform | M3 |
| EXP-2 | Snapshot Consumer with deduplication by version; File Builder (CSV by the specification, `.sha256`); signing through the vault / HSM. | File set built for each new version. | 2 | CR-3, BIZ-2, SEC-2 | M3–M4 |
| EXP-3 | SFTP Delivery Worker: atomic upload (`*.part` → rename), host key pinning, 5 retries with backoff, alerts; export log and 5-year archive. | Files delivered to a test SFTP server. | 2 | EXP-2, PLT-2 | M4 |
| EXP-4 | Export Scheduler and Admin API: daily file at 08:00, delivery check at 08:30, manual resend (role "support", audit log); single active scheduler through a MS SQL cluster lock. | Scheduled and manual delivery. | 1 | EXP-3 | M4 |
| EXP-5 | Test exchange and parallel run with the partner (2 weeks); runbooks: resend, key rotation, partner connection failure. | Partner confirms import; runbooks in the wiki. | 1 | EXP-4, PRT-2 | M4 (MS3) |

## 4. Call Center System (call center system team + vendor)

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| CCS-1 | Agree the scope and cost with the vendor; choose how to build the job and the screen on the platform (or the iframe fallback, ADR A2). | Signed change request. | 1 | — | M1–M2 |
| CCS-2 | Rate Reference Table and Rate Sync Job: every 5 min, `If-None-Match`, save new versions in one transaction, alert after 3 failed cycles; OAuth2 client and mTLS certificate. | Rates are synced from the test API. | 2 | CR-4, CCS-1 | M3–M4 |
| CCS-3 | Operator Rates Screen in the bank design system: filters by product, term, amount and currency; "rates as of"; warning for expired rates. | Screen accepted by call center supervisors. | 2 | CCS-2 | M4 |
| CCS-4 | Acceptance with operators, update of the call scripts, go-live. | Bank operators consult on rates. | 1 | CCS-3, BIZ-3 | M4 (MS3) |

## 5. Kafka and platform (infrastructure team)

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| PLT-1 | Topic `deposit.rate-snapshot`: compacted, RF 3, `min.insync.replicas` = 2, ACLs (write: Catalog & Rate Service, read: Rate File Export Service). | Topic in all environments. | 0.5 | Task3 Kafka | M3 |
| PLT-2 | DMZ egress from the export service to the partner's SFTP server: firewall rules, fixed NAT address for the partner's allow-list, in both DCs. | Network path tested. | 1 | SEC-1 | M3 |

## 6. Information security

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| SEC-1 | Threat model of the partner exchange; security assessment of the partner; information exchange annex to the contract (keys, signature algorithm, incident contacts). | Approved by information security. | 1.5 | BIZ-2 | M2–M3 |
| SEC-2 | Keys and certificates: SSH key pair, signing certificate (GOST R 34.10-2012 if required), client certificate and OAuth2 client for the Call Center System. All kept in the vault, with a rotation procedure. | Keys issued; procedure documented. | 1 | SEC-1 | M3 |
| SEC-3 | Security check of Release 1 before go-live: new gateway route, public rates on the website, outbound SFTP link to the partner (scan, configuration review, test of key and host-key checks). The full pen test of the MVP follows in M5–M6. | No open critical or high findings. | 0.5 | SEC-2, CR-4, EXP-3 | M3–M4 (before MS3) |

## 7. Partner call center (external)

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| PRT-1 | SFTP endpoint for the bank: account with key authentication, chrooted write-only folder, IP allow-list. | Endpoint reachable from the bank. | 1 | BIZ-2, PLT-2 | M3 |
| PRT-2 | Import into the partner's system: take the highest valid version, check checksum and signature, hide rates after `valid_until`, show rates to operators. | Import works on test files. | 3 | PRT-1, BIZ-2 | M3–M4 |
| PRT-3 | Operator training and scripts; go-live. | Partner operators consult on rates. | 1 | PRT-2, BIZ-3 | M4 (MS3) |

## 8. Telephony and call center operations

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| TEL-1 | Overflow routing of calls to the partner call center (queue thresholds, working hours, transfer back to the bank). | Routing rules tested. | 2 | BIZ-2 | M3–M4 |

## 9. Business and PMO

| ID | Task | Result | Estimate | Depends on | Roadmap |
|---|---|---|---|---|---|
| BIZ-1 | Confirm the scope: public rates only; validity rule (24 h); freshness targets. | Signed requirements. | 0.5 | — | M1 |
| BIZ-2 | Contract and SLA with the partner, including the rate file specification (version 1) and delivery times. | Signed contract and specification. | 3 | BIZ-1 | M1–M2 |
| BIZ-3 | Call scripts for deposit consultations (including "rates as of" and expired rates), training of bank and partner operators. | Trained operators. | 2 | CCS-3, PRT-2 | M4 |
| BIZ-4 | Start the marketing campaign only after Release 1 (call centers ready) and Release 2 (MVP go-live). | Campaign plan aligned with MS3–MS5. | — | MS3, MS4 | M7 |

## Tasks from the Task3 MVP on the same roadmap

The roadmap also includes the main MVP work from [Task3](../Task3/ADR.md). These tasks are prerequisites for this case or run in parallel with it:

| System | Main tasks | Roadmap |
|---|---|---|
| ABS | Rate management and approval (screens, rate grid, roles); ABS Integration API, outbox, adapter; application queue and statuses; risk category view; load test with throttling. | M1–M5 |
| Deposit services | API Gateway and contracts; Catalog & Rate Service; Deposit Application Service; OTP and Notification services. | M1–M5 |
| IB Core and Web UI | OIDC tokens, accounts API, "Deposits" menu (vendor); deposit micro-frontend; reference data through the API. | M2–M5 |
| Website | Rates page; application form with consent and CAPTCHA. | M3–M5 |
| Call Center System | Website lead API and CRM cards. | M4–M5 |
| Infrastructure and security | Kafka in 2 DCs, schema registry, vault, observability; container platform, Redis, MS SQL Always On, GSLB, WAF; pen test and compliance check. | M1–M6 |
| QA and release | Contract and integration tests, load tests at 2× target load, UAT, pilot, go-live. | M4–M7 |

## Target solution (months 7–12)

| System | Task |
|---|---|
| ABS | Automatic approval of standard applications; move IB payments off direct SQL to the ABS (API / Kafka). |
| Deposit services | Application status API for the call centers. |
| Call Center System | Application status for operators (a client calls "where is my application?"); personal rate for an identified caller. |
| Rate File Export Service and partner | Inbound callback request files from the partner (SFTP) that become leads in the Call Center System; discovery of an API or a shared CRM workspace instead of files. |
| Infrastructure | IB monolith active-passive in the backup DC (99.9% for the whole IB). |
| Website | More deposit products online (configuration only). |
| Architecture | Discovery: mobile app; a pricing engine outside the ABS (Task3 alternative A3); an MFT product if the number of file exchanges grows. |
