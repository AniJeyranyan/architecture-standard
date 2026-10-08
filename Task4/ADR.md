### **Task name:** Deposit rates for the bank and the partner call centers (change to the online deposit opening MVP)
### **Author:** Ani Jeyranyan
### **Date:** 2026-10-09

### **Context**

Most clients of bank "Standart" are elderly. When the marketing campaign for online deposits starts, many of them will call the call center to ask about the deposit conditions instead of using the website or the IB. The bank wants to handle this with two changes:

1. **Bank call center.** Operators must be able to consult clients at least on the current deposit rates, so the Call Center System needs access to the rates.
2. **Partner call center.** To avoid overloading the bank's call center, overflow calls go to a partner call center. Its operators must also quote the current rates. The partner works in its own information system outside the bank. It can receive files (over SFTP), but it cannot call APIs or offer them.

The decision builds on the MVP architecture from [Task3](../Task3/ADR.md). Rates are kept and approved in the ABS (Task3 UC9). The ABS Integration Adapter publishes them to Kafka (`deposit.rates`), and the Catalog & Rate Service keeps the published copy that the website and the IB read. Today (Task1) the partner call center works from scripts and has no integration with the bank, and the bank's Call Center System only passes inquiries to the ABS. In Task2 (+R1, F2) the partner call center was out of the MVP scope; this change adds it to the MVP, but only for consulting on rates.

### **Functional requirements**

#### Use cases

Arrow numbers in brackets refer to the [component diagram](c4-component.png) and to the "Component integrations" table below.

| **#** | **Actors or systems** | **Use Case** | **Description** |
| :-: | :- | :- | :- |
| UC1 | Credit and deposit back office; ABS; ABS Integration Adapter; Kafka; Catalog & Rate Service | Publish approved rates for the call centers | 1. The back office enters and approves the daily rates in the ABS, as in Task3 UC9. The adapter publishes them to `deposit.rates` (#7).<br>2. The Rates Kafka Consumer updates the read model (#8, #9). When every rate of the approved set has arrived (the event carries the rate set version and the number of items), it notifies the Rate Snapshot Builder (#10).<br>3. The builder creates snapshot version N. The snapshot holds the public products, the rate grid by term and amount, the conditions (payout, capitalization, top-ups, partial withdrawal, early closure rate), the approval time and `valid_until` = approval time + 24 h. Personal and special rates are left out. The builder saves the snapshot in the Catalog Cache (#11).<br>4. The Snapshot Publisher publishes the snapshot to the compacted topic `deposit.rate-snapshot` (#12, #13). |
| UC2 | Client; Bank call center operator; Call Center System | Consult a client on current rates in the bank call center | 1. A client calls the bank and asks about deposits.<br>2. The operator opens the Operator Rates Screen in the Call Center System (#1). The screen reads the latest snapshot from the local Rate Reference Table (#2), so it does not depend on the bank API during the call.<br>3. The operator filters by product, term, amount and currency and sees the rate, the conditions and "rates as of <time>".<br>4. If `valid_until` has passed, the screen shows a warning and the operator does not quote a rate. The operator offers a callback instead.<br>5. If the client wants to open a deposit or asks for a special rate, the operator follows the existing process: an inquiry to the ABS (Task3 UC3), or directions to the website, the IB or a branch. |
| UC3 | Call Center System; API Gateway; Catalog & Rate Service | Synchronize rates to the Call Center System | 1. Every 5 minutes the Rate Sync Job calls `GET /v1/call-center/rates` with `If-None-Match: <current version>` (#4). The call goes over mTLS with an OAuth2 client-credentials token (scope `rates.read`).<br>2. The API Gateway checks the token and the rate limit and routes the call (#5). The Call Center Rates API reads the snapshot from the Catalog Cache (#6).<br>3. If nothing changed, the API returns `304 Not Modified`. If there is a new version, it returns `200` with the full snapshot and its `ETag`.<br>4. The job saves the new version in the Rate Reference Table in one transaction (#3) and keeps older versions as history.<br>5. On an error the job keeps the last version and retries in the next cycle. After 3 failed cycles in a row, the Call Center System raises an alert. |
| UC4 | Kafka; Rate File Export Service; Secrets Vault; Partner SFTP Server | Deliver a rate file to the partner when rates change | 1. The Snapshot Consumer receives snapshot version N from `deposit.rate-snapshot` (#14). If version N was already delivered, it skips it (idempotency) (#15).<br>2. The File Builder and Signer creates the file set: the CSV file in the agreed format (see "Rate file specification"), a `.sha256` checksum file and a detached signature `.sig`. The signing key never leaves the vault or HSM (#17).<br>3. The SFTP Delivery Worker uploads each file as `*.part` and renames it when the upload is complete, so the partner never reads a half-written file (#18, #19). The connection goes out through the DMZ egress, uses an SSH key and checks the pinned host key.<br>4. The delivery result is saved in the export log. A copy of every file sent is archived for 5 years (#20, #21).<br>5. If the upload fails, the worker retries 5 times with exponential backoff (1, 2, 4, 8, 16 minutes) and then raises an alert. |
| UC5 | Rate File Export Service | Daily file and delivery check | 1. Every day at 08:00 (Moscow time) the Export Scheduler sends the current snapshot again as a new file. The version stays the same; the file has a new generation time (#16). The partner then knows that its rates are still valid on days when they did not change.<br>2. At 08:30 the scheduler checks the export log (#24). If no file was delivered today, or if the latest snapshot version has not been delivered within 15 minutes after approval, it raises an alert to the support team. |
| UC6 | Client; Partner call center operator; Partner Call Center System | Consult a client on current rates in the partner call center | 1. A call is routed to the partner when the bank's call center queue is overloaded (telephony overflow routing).<br>2. The Partner Call Center System picks up the file with the highest version from its SFTP inbound folder (#22). It checks the checksum and the signature and rejects a file that fails the check.<br>3. The operator sees the rates, the conditions, "rates as of" and `valid_until` (#23). After `valid_until`, the operator does not quote rates (the partner's system should hide them) and transfers the client to the bank's call center or arranges a callback.<br>4. If the client wants to open a deposit, the operator directs them to the website application form, the IB or a branch. Callback requests from the partner to the bank are part of the target solution, not the MVP (see the [roadmap](roadmap.png)). |
| UC7 | Support engineer; Rate File Export Service; Back office | Resend a file or correct rates | 1. If the partner has lost a file or its SFTP server was down for longer than the retries, the support engineer triggers a resend of the latest version through the Admin API (#24, #16). The action is written to the audit log.<br>2. If rates were approved with a mistake, the back office approves corrected rates in the ABS. The corrected set gets a new version and reaches both call centers through UC1, UC3 and UC4 without manual steps: within 10 minutes in the bank call center and within 15 minutes for the partner. |

#### Functional requirements

| **#** | **Requirement** | **Use cases** |
| :-: | :- | :- |
| FR1 | Bank call center operators see the current public rates and conditions of every deposit product in the Call Center System, with the snapshot version and "rates as of". | UC2, UC3 |
| FR2 | Rates in both call centers come from the same source as on the website and in the IB: approved in the ABS and published by the Catalog & Rate Service. No manual copying, no e-mail, no XLS. | UC1, UC3, UC4 |
| FR3 | The operator can filter by product, term, amount and currency and see the rate for those parameters. | UC2 |
| FR4 | Rates older than 24 hours (`valid_until` has passed) are clearly marked, and operators of both call centers are told not to quote them. This is the same rule as on the website and in the IB (Task2 R5). | UC2, UC6 |
| FR5 | The partner receives a full snapshot file on every rate change and once a day at 08:00, even if nothing changed. | UC4, UC5 |
| FR6 | The partner file is machine-readable and follows a fixed, versioned specification (CSV). It comes with a checksum and a signature, so the partner can check that the file is complete and comes from the bank. | UC4, UC6 |
| FR7 | Every delivery is logged. A failed delivery raises an alert, and support can resend the latest file. | UC4, UC5, UC7 |
| FR8 | Only public rates are sent to the call centers. Personal rates, special rates, risk categories and client data are never sent to the Call Center System or the partner (Task2 +R8, 152-FZ). | UC1 |
| FR9 | Every file sent to the partner is archived for 5 years, so the bank can show which rates the partner had at any moment, for example when handling a complaint. | UC4 |
| FR10 | A new deposit product appears in both call centers without changes to the Call Center System, the export service or the partner's import. The snapshot and the file are data-driven (Task2 S6). | UC1, UC3, UC4 |

### **Non-functional requirements**

| **#** | **Requirement** |
| :-: | :- |
| NFR1 | **Freshness.** Bank call center: new rates are visible ≤ 10 minutes after approval (p95; the 5-minute publication target of Task2 P6 plus one 5-minute sync cycle). Partner: the file is on the partner's SFTP server ≤ 15 minutes after approval (p95). The daily file is delivered by 08:30. |
| NFR2 | **Availability.** The Call Center Rates API is part of the deposit services and has the same 99.9% target (Task2 R1), in 2 DCs. Operators read a local copy, so consulting keeps working during an outage of the bank API or of the partner link, as long as the rates are still valid (24 h). The Rate File Export Service is asynchronous; its target is ≥ 99.5% plus the delivery-time targets in NFR1, and it runs in both DCs with one active scheduler (cluster lock in MS SQL). |
| NFR3 | **No extra load on the ABS and no growth with call volume.** The call centers never call the ABS for rates. Load on the bank API is one request per 5 minutes from the Call Center System (plus manual refreshes, limited to 1 per second per client at the gateway). It does not depend on the number of operators or calls. Snapshot size ≤ 1 MB. |
| NFR4 | **Performance.** The Operator Rates Screen opens and filters in ≤ 1 s (local database). The Call Center Rates API: p95 ≤ 200 ms, p99 ≤ 400 ms; a `304` response p95 ≤ 50 ms. |
| NFR5 | **Security.** Call Center System → bank: mTLS (TLS 1.2+) plus an OAuth2 client-credentials token with the `rates.read` scope only. Bank → partner: SFTP only (SSH, password login disabled), ed25519 or RSA ≥ 3072 keys, a pinned partner host key, an IP allow-list on both sides, a chrooted write-only folder for the bank, and outbound connections only from the bank's DMZ. Keys and certificates are kept in the vault and rotated at least once a year. The files contain no personal data. (Task2 F12, F13, +R7) |
| NFR6 | **Integrity.** SHA-256 checksum and a detached CMS signature. The algorithm is agreed in the contract: GOST R 34.10-2012 where the bank's information security rules require it, otherwise RSA-PSS or ECDSA. Versions only grow. The partner shows only the highest valid version and rejects files that fail the check. |
| NFR7 | **Reliability.** Idempotent processing by snapshot version in the Call Center System and in the export service. Atomic upload (`*.part`, then rename). 5 retries with exponential backoff, then an alert. The compacted Kafka topic lets every consumer rebuild the latest state after a failure (Task2 R4). |
| NFR8 | **Observability.** Metrics per consumer (Call Center System, partner): last delivered snapshot version and the age of the rates. Alerts: a consumer is more than 30 minutes behind the latest version; no partner file by 08:30; 3 failed sync cycles in the Call Center System. Correlation ID from the snapshot version through Kafka headers to the export log (Task2 S4). |
| NFR9 | **Audit.** Archive of all files sent (5 years). Resends and key rotations are written to the append-only audit log (Task2 F15). |
| NFR10 | **Existing technologies.** .NET 8, Kafka, Redis, MS SQL and the vault from Task3. The only new library is an SFTP client (SSH.NET). No new products or licenses (Task2 +R9). |
| NFR11 | **Extensibility.** The file format has a `format_version` column. Within a version only backward-compatible changes are allowed (new columns at the end), and an old version is supported for 6 months after a new one (Task2 S7). Adding another partner means adding one more delivery target in configuration, not a code change. |
| NFR12 | **Data format.** Dates and times use ISO 8601 with an offset (e.g. `2027-02-10T09:15:00+03:00`), rates are decimals with a dot and up to 2 decimal places, amounts are decimals with an ISO 4217 currency code (Task2 +R11). |

### **Solution**

#### Diagrams

| File | Contents |
|---|---|
| [c4-context.png](c4-context.png) / [c4-context.drawio](c4-context.drawio) | C4 Level 1: System context. Only the people and systems that take part in the new case. The website and the IB channels do not change and are left out. |
| [c4-component.png](c4-component.png) / [c4-component.drawio](c4-component.drawio) | C4 Level 3: Components of the three containers that change: Call Center System, Catalog & Rate Service and the new Rate File Export Service, plus the containers from Task3 they use. |
| [roadmap.png](roadmap.png) / [roadmap.drawio](roadmap.drawio) | RoadMap: the MVP in 6 months, including this case, and the target solution in 12 months. |

Color code as in Task3: dark blue is new, light blue with an orange border is changed, grey is existing without changes, light grey with a dashed border is external. A dashed grey arrow is an integration from the Task3 MVP that this case reuses without changes.

#### Summary of the decision

1. **One source of rates for all channels.** The call centers get the same approved rates as the website and the IB: ABS → Kafka → Catalog & Rate Service. The ABS gets no new functions and no extra load; only its existing `deposit.rates` event is extended. (FR2, NFR3)
2. **A versioned "call center snapshot".** The Catalog & Rate Service builds a complete snapshot of the public rates and conditions with a version and `valid_until` each time an approved rate set is complete. Both call centers receive exactly this snapshot. Bank and partner operators therefore always quote the same numbers, and the export log shows which version each of them had. (FR1, FR6, FR9)
3. **Bank call center: pull with a local copy.** The Call Center System pulls the snapshot through the API Gateway every 5 minutes, with an `ETag`, and stores it in its own table. The operator screen reads only the local copy. API load does not grow with the number of calls, and a bank API outage does not stop consulting. (NFR2, NFR3, NFR4)
4. **Partner call center: signed CSV files pushed over SFTP.** The new Rate File Export Service consumes the snapshot from the compacted topic `deposit.rate-snapshot`, builds a CSV file with a checksum and a signature, and pushes it to the partner's SFTP server. Files are sent on every rate change and every morning. The bank only opens outbound connections and controls delivery and retries. (FR5–FR7, NFR5–NFR7)
5. **Separate container for file export.** File generation, signing keys and the DMZ connection to an external party are kept out of the Catalog & Rate Service, which serves 200+ RPS for the channels. The export service can fail, be released or change its security rules without affecting the website and the IB.
6. **Only public data leaves the deposit services.** Personal and special rates stay in the IB and the ABS process. The call center files contain no personal data, so 152-FZ does not apply to the partner exchange. Integrity matters more than confidentiality here: a wrong rate quoted to a client is a financial and reputational risk. (FR8, NFR6)
7. **Same validity rule everywhere.** A snapshot is valid for 24 hours after approval, the same rule as on the website and in the IB. Expired rates are marked and are not quoted. (FR4)

#### Containers and components

| Container / component | Technology | Status | Responsibility |
|---|---|---|---|
| **Catalog & Rate Service** | .NET 8 | Changed | |
| – Rates Kafka Consumer | Confluent.Kafka | Changed | As in Task3, plus: detects that an approved rate set is complete (rate set version and number of items in the event) and notifies the Snapshot Builder. |
| – Rate Snapshot Builder | .NET | New | Builds the versioned call center snapshot (public products, rate grid, conditions, `valid_until`) and saves it in the Catalog Cache. |
| – Call Center Rates API | ASP.NET Core | New | `GET /v1/call-center/rates`: full snapshot as JSON; `ETag` = version; `304` if unchanged. OpenAPI 3 contract. |
| – Snapshot Publisher | Confluent.Kafka | New | Publishes each new snapshot to `deposit.rate-snapshot` (compacted, key `public`, idempotent producer). Consumers ignore versions they have already processed. |
| **Rate File Export Service** | .NET 8 worker, MS SQL | New | Owned by the Internet Bank team, deployed in both DCs. |
| – Snapshot Consumer | Confluent.Kafka | New | Consumes `deposit.rate-snapshot`; skips versions already delivered. |
| – File Builder and Signer | .NET, vault / HSM API | New | CSV in the agreed format, SHA-256 checksum file, detached CMS signature. |
| – SFTP Delivery Worker | SSH.NET | New | Atomic upload to the partner, retries with backoff, alerts. Connects through the DMZ egress. |
| – Export Scheduler and Admin API | Quartz.NET (cluster lock in MS SQL); internal REST | New | Daily file at 08:00, delivery check at 08:30, manual resend (role "support", audited), health and status. |
| – Export Log Repository | EF Core, schema `rate_export` in the Deposit Services DB | New | Delivery status per version and target, archive of the files sent (5 years). |
| **Call Center System** | Vendor platform (Java, PostgreSQL) | Changed (configuration and small extensions by the vendor) | |
| – Rate Sync Job | CRM scheduled integration job | New | Pulls the snapshot every 5 minutes with `If-None-Match`, saves new versions, alerts after 3 failed cycles. |
| – Rate Reference Table | PostgreSQL | New | Current snapshot and version history. |
| – Operator Rates Screen | CRM screen or widget, bank design system | New | Filter by product, term, amount and currency; "rates as of"; warning for expired rates. |
| API Gateway + WAF | YARP (Task3) | Existing, new route | Route `/v1/call-center/rates`, OAuth2 client for the Call Center System, mTLS client certificate, rate limit. |
| Kafka | Task3 | Existing, new topic | New compacted topic `deposit.rate-snapshot` (RF 3, ACL: write by the Catalog & Rate Service, read by the Rate File Export Service), schema in the schema registry. |
| Catalog Cache, Deposit Services DB, Secrets Vault | Task3 | Existing | New key for the snapshot; new schema `rate_export`; signing key and SSH key. |
| ABS, ABS Integration Adapter | Task3 | Changed (event contract only) | No new functions. The existing `deposit.rates` event is extended with the rate set version, the approval time, the number of items and the product conditions. The contract is agreed before the contracts are frozen (MS1 on the roadmap); see ABS-1 and ABS-2 in tasks.md. |
| Partner SFTP Server, Partner Call Center System | Partner | External | Inbound folder for the bank; import and display of the rates. Built by the partner according to the file specification. |

#### Rate file specification (version 1)

| Item | Value |
|---|---|
| File set | `standart_deposit_rates_<yyyyMMddTHHmmss>_v<version>.csv`, the same name + `.sha256`, the same name + `.sig` |
| Delivery | Push to the partner's SFTP folder `/inbound/standart/rates/`; upload as `*.part`, then rename. The partner deletes files older than 30 days; the bank keeps its own archive. |
| Encoding and format | CSV (RFC 4180), UTF-8 with BOM (so the file also opens correctly in Excel as a manual fallback), separator `;`, decimal point `.`, header row |
| Columns | `format_version; snapshot_version; generated_at; approved_at; valid_until; product_code; product_name; currency; term_from_days; term_to_days; amount_from; amount_to; rate_percent; payout (MONTHLY / END_OF_TERM); capitalization (Y/N); top_up (Y/N); partial_withdrawal (Y/N); early_closure_rate_percent; note` |
| Rules | Each file is a full snapshot, not a delta. The partner uses only the highest `snapshot_version` with a valid checksum and signature, and shows no rates after `valid_until`. |

#### Context integrations

Arrow numbers on the [context diagram](c4-context.png).

| # | From → To | Meaning | Use cases |
|:-:|---|---|---|
| 1 | Client → Bank call center operator | Calls about deposit conditions (phone) | UC2 |
| 2 | Client → Partner call center operator | Overflow calls are routed to the partner by the telephony | UC6 |
| 3 | Bank call center operator → Call Center System | Looks up current rates | UC2 |
| 4 | Partner call center operator → Partner Call Center System | Looks up current rates | UC6 |
| 5 | Back office → ABS | Enters and approves daily rates (Task3 UC9, unchanged) | UC1 |
| 6 | ABS → Internet Bank: deposit services | Approved rates through Kafka `deposit.rates` (Task3); the event is extended with the rate set version, the number of items and the conditions | UC1 |
| 7 | Call Center System → Internet Bank: deposit services | Rate snapshot; REST, mTLS + OAuth2, every 5 minutes, ETag | UC3 |
| 8 | Internet Bank: deposit services → Partner Call Center System | Signed CSV rate files; SFTP push on every approval and daily at 08:00 | UC4, UC5, UC7 |

#### Component integrations

Arrow numbers on the [component diagram](c4-component.png). Arrows point from the component that starts the call (the "Data" column says what is read or sent); Kafka arrows show the event flow.

| # | From → To | Data | Protocol / mode | Use cases |
|:-:|---|---|---|---|
| 1 | Bank call center operator → Operator Rates Screen | Search for rates | UI | UC2 |
| 2 | Operator Rates Screen → Rate Reference Table | Reads the latest snapshot | SQL (inside the vendor platform) | UC2 |
| 3 | Rate Sync Job → Rate Reference Table | New snapshot version | SQL, one transaction | UC3 |
| 4 | Rate Sync Job → API Gateway | `GET /v1/call-center/rates`, `If-None-Match` | REST, mTLS + OAuth2 client credentials, every 5 min | UC3 |
| 5 | API Gateway → Call Center Rates API | Same | REST | UC3 |
| 6 | Call Center Rates API → Catalog Cache | Read the snapshot | Redis protocol, TLS | UC3 |
| 7 | ABS Integration Adapter → Kafka | `deposit.rates` (Task3, extended schema) | Kafka, async | UC1 |
| 8 | Kafka → Rates Kafka Consumer | `deposit.rates` (Task3, extended schema) | Kafka, async | UC1 |
| 9 | Rates Kafka Consumer → Catalog Cache | Read model update (Task3) | Redis protocol | UC1 |
| 10 | Rates Kafka Consumer → Rate Snapshot Builder | "Rate set version complete" | In-process | UC1 |
| 11 | Rate Snapshot Builder → Catalog Cache | Snapshot + version | Redis protocol | UC1 |
| 12 | Rate Snapshot Builder → Snapshot Publisher | New snapshot | In-process | UC1 |
| 13 | Snapshot Publisher → Kafka | `deposit.rate-snapshot` | Kafka, compacted, idempotent producer | UC1 |
| 14 | Kafka → Snapshot Consumer | `deposit.rate-snapshot` | Kafka, async | UC4 |
| 15 | Snapshot Consumer → File Builder and Signer | New version (deduplicated) | In-process | UC4 |
| 16 | Export Scheduler and Admin API → File Builder and Signer | Daily file at 08:00; manual resend | In-process; internal REST for support | UC5, UC7 |
| 17 | File Builder and Signer → Secrets Vault / HSM | Sign the file hash | Vault API, mTLS | UC4 |
| 18 | File Builder and Signer → SFTP Delivery Worker | File set (CSV, `.sha256`, `.sig`) | In-process | UC4 |
| 19 | SFTP Delivery Worker → Partner SFTP Server | Upload + rename | SFTP (SSH), key authentication, through the DMZ egress | UC4, UC5, UC7 |
| 20 | SFTP Delivery Worker → Export Log Repository | Delivery status | In-process | UC4, UC5 |
| 21 | Export Log Repository → Deposit Services DB | Export log, file archive | SQL, TLS | UC4, UC5 |
| 22 | Partner Call Center System → Partner SFTP Server | Picks up the latest file | Partner side | UC6 |
| 23 | Partner call center operator → Partner Call Center System | Looks up rates | UI (partner side) | UC6 |
| 24 | Export Scheduler and Admin API → Export Log Repository | Delivery check at 08:30; latest delivered version for a resend | In-process | UC5, UC7 |

#### Why these decisions and technologies

- **Pull with ETag for the Call Center System** instead of push or Kafka. The vendor platform can run a scheduled REST job but should not become a Kafka consumer of internal bank topics. One request every 5 minutes is negligible, and `304` answers are almost free. Rates change about once a day, so up to 10 minutes of delay is acceptable.
- **A snapshot event (`deposit.rate-snapshot`) for the export service** instead of a second REST client. The export service reacts to new rates within seconds and needs no polling. Because the topic is compacted, the latest snapshot can always be rebuilt after a restart.
- **Push over SFTP from the bank** instead of a bank-hosted SFTP server that the partner pulls from. No inbound connections into the bank. The bank knows the exact time each file was delivered and owns the retries. The partner said it is ready to receive files.
- **CSV + checksum + signature.** CSV is the simplest format that any partner system can import and that a person can open in Excel in an emergency. The signature protects against a wrong or modified file in the partner's system, which matters because operators quote these numbers to clients.
- **SSH.NET, Quartz.NET** are mature .NET libraries. They fit the .NET 8 stack of the deposit services, so no new platform is needed (+R9).

#### IT teams that take part

| Team | Scope of work |
|---|---|
| Internet Bank team | Snapshot Builder, Call Center Rates API, Snapshot Publisher; the new Rate File Export Service; gateway route. |
| ABS team | Make sure the `deposit.rates` contract contains the rate set version, the number of items and the product conditions; take part in end-to-end tests. |
| Call center system team / vendor | Rate Sync Job, Rate Reference Table, Operator Rates Screen, monitoring. |
| Infrastructure / platform team | Kafka topic and ACLs, DMZ egress to the partner, NAT address for the partner's allow-list, deployment of the new service in 2 DCs. |
| Information security | Threat model, SSH keys and signing certificate in the vault, security assessment of the partner connection, information exchange annex to the partner contract. |
| Telephony / call center operations | Overflow routing to the partner call center. |
| Partner call center (external) | SFTP endpoint, file import with checks, rate screen for operators, operator training. |
| Business | File specification and SLA with the partner, call scripts, training, acceptance. |

The detailed list of tasks for each system is in [tasks.md](tasks.md), and the order of the work is on the [roadmap](roadmap.png).

### **Alternatives**

| # | Alternative | Why it was not chosen |
|:-:|---|---|
| A1 | **The Call Center System consumes Kafka (`deposit.rates`) directly.** | Needs a Kafka client and schema registry access in the vendor platform and a network path from the vendor system into the internal Kafka cluster. It couples the call center to the internal event schema, which is meant for the deposit services. |
| A2 | **Embed a bank web page with the rates in the operator workspace (iframe)** instead of a local copy. | No vendor work and always fresh, but the vendor CRM may not allow it, consulting stops when the bank API is down, and API load grows with every call. It can serve as a temporary fallback if the vendor is late (risk 2). |
| A3 | **On-demand API call for every operator request** (no local copy). | Load grows with the call volume, which is exactly what will peak during the campaign, and an outage of the bank API stops consulting. |
| A4 | **Bank-hosted SFTP server in the DMZ; the partner pulls files.** | Opens inbound connections into the bank, the bank has to run an internet-facing server, and it cannot tell whether the partner picked up the file. Kept as a fallback if the partner cannot host an SFTP server. |
| A5 | **Rates for the partner by e-mail (XLS), or partner operators use the public website.** | E-mail is the manual process the MVP removes: no integrity, no audit, delays (Task2 F8). The public website shows rates but not all the conditions operators need, and the bank cannot show which rates the partner had. The website is a reasonable last-resort fallback for the partner's operators. |
| A6 | **The ABS generates and sends the rate files** (PL/SQL job). | Adds work to the ABS team, which is already on the critical path (Task3 risk 4). Files and SFTP are not its stack, and the ABS gets extra load. |
| A7 | **File export inside the Catalog & Rate Service.** | One container less, but it mixes the high-load channel API with batch file work, signing keys and a connection to an external party. Security zones, scaling and releases would be tied together. |
| A8 | **A Managed File Transfer (MFT) product.** | A good long-term option when there are many partners and file flows, but it is a new product with a license (+R9) for one small file a day. It should be considered in the target architecture. |

**Disadvantages, limitations, risks**

| # | Risk / limitation | Impact | Mitigation |
|:-:|---|---|---|
| 1 | **The bank cannot see what partner operators actually see.** The import and display are in the partner's system. | The partner may show an old or wrong file. | SLA and file specification in the contract; `valid_until` in every file; checksum and signature checks; acceptance test of the partner's import. Target: an acknowledgement file from the partner. |
| 2 | **The vendor of the Call Center System** must build the sync job and the screen. | Release 1 may be delayed. | Agree the scope with the vendor in M1. Temporary fallback: an embedded read-only rate page (A2). |
| 3 | **Up to 10 minutes (bank) and 15 minutes (partner) of delay** after rates are approved. | A client may hear the previous rate right after a change. | Rates change about once a day; operators say "rates as of …"; the final rate is fixed in the ABS when the deposit is opened (Task3 risk 1). |
| 4 | **Operators may ignore the expiry warning.** | Expired rates are quoted. | The partner's system hides expired rates; scripts and training; quality control of calls. |
| 5 | **Key management between two organizations** (SSH keys, signing certificate). | Delivery stops after an uncoordinated key rotation. | A documented rotation procedure with an overlap period; expiry monitoring in the vault. |
| 6 | **Network changes on the partner side** (IP address, host key). | Delivery fails. | Retries and alerts within minutes (NFR8); the daily 08:30 check; contacts and change procedure in the SLA. |
| 7 | **The snapshot depends on the completeness signal from the ABS** (rate set version and number of items). | A partial rate set could be published. | Agree the event contract before MS1. The builder publishes only complete sets, and contract tests check this. |
| 8 | **The case covers consulting only.** Elderly clients may still want to open a deposit by phone. | Calls end with "please go to the website or a branch". | Callback requests from the partner and application status for operators are in the target solution (months 7–12 on the roadmap). |
| 9 | **One more container to run** (Rate File Export Service). | Operations effort. | Same stack and platform as the other deposit services; runbooks for resend and key rotation (Task2 S3). |
