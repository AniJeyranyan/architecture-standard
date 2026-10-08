### **Task name:** Conceptual architecture of online deposit opening (MVP)
### **Author:** Ani Jeyranyan
### **Date:** 2026-10-08

### **Functional requirements**

Top-level use cases of the MVP. They are based on the FURPS+ table in [Task2](../Task2/FURPS+.md) (codes F1–F16) and on the AS-IS process in [Task1](../Task1). The numbers of integrations in brackets, such as (#14), refer to the arrows on the [container diagram](c4-container.png) and the "Integrations" table below.

| **#** | **Actors or systems** | **Use Case** | **Description** |
| :-: | :- | :- | :- |
| UC1 | Prospective client; Website; API Gateway; Catalog & Rate Service; Catalog Cache | View deposits and current rates on the website (F1) | 1. The visitor opens the deposits page (#1, #2).<br>2. The website backend requests the public product list and rates from the API Gateway (#3), which routes the call to the Catalog & Rate Service (#13).<br>3. The service reads the rates from the Catalog Cache (#17). The cache is filled from Kafka with the rates approved in the ABS (#21, see UC9), so the ABS is not called.<br>4. The website shows the deposits, the rates and the time the rates were last updated. |
| UC2 | Prospective client; Website; API Gateway; Deposit Application Service; Call Center System | Submit a deposit application on the website (F2, F12, F14) | 1. The visitor chooses a deposit and enters full name and phone number, ticks the personal data consent and passes the CAPTCHA (#1, #2).<br>2. The website backend sends the application to the API Gateway over mTLS (#3). The gateway checks the rate limits and the WAF rules and routes it to the Deposit Application Service (#14).<br>3. The service checks the data, saves the application with an idempotency key and an outbox record in one transaction (#18), and returns the application number. The website shows "Application No. X accepted, a manager will call you".<br>4. The outbox worker delivers the application to the Call Center System as a new CRM card (#25), with retries. If the Call Center System is down, the application waits in the outbox and is not lost. |
| UC3 | Call center manager; Call Center System; ABS; Deposit back office; ABS Integration Adapter; Kafka; Notification Service; SMS Gateway | Process a website application in the call center, offer special conditions (F3, F10) | 1. The manager opens the card of the website application in the Call Center System and calls the client (#32).<br>2. If the client agrees, the manager sends an inquiry to the ABS through the existing call center → ABS integration (#33). The inquiry now also carries the website application ID.<br>3. If special conditions are needed, the deposit back office sets the special rate in the ABS (#30). It sees only the client's risk category from the credit section, not the raw credit data (+R8).<br>4. The ABS writes a "rate confirmed" event into its outbox (#27). The adapter publishes it to Kafka (#24). The Notification Service sends an SMS with the rate and an invitation to a branch (#23, #34, #35), and the Deposit Application Service updates the status of the website application (#22). |
| UC4 | Prospective client; Branch employee; ABS Desktop Client; ABS; Kafka; Notification Service | Open a deposit in a branch for a new client from a website application (F4, +R2) | 1. The client comes to a branch with an ID document.<br>2. The branch employee identifies the client and finds the website application in the ABS by its number or phone (#29, #28). The agreed conditions from UC3 are already there, so no e-mail to the back office is needed.<br>3. The employee opens the deposit in the ABS, prints the documents, the client signs them, and the employee uploads the scans (as today).<br>4. The ABS publishes "deposit opened" (#27, #24). The client gets an SMS (#23, #34), and the website application is closed as converted (#22), which lets the bank measure the conversion of website applications. |
| UC5 | IB client; IB Web UI; Deposit Micro-frontend; API Gateway; IB Core; Catalog & Rate Service | View deposits with personalized rates in the IB (F5, F16, +R8) | 1. The client logs in to the IB as today (#4, #9) and opens the new "Deposits" section, which loads the Deposit Micro-frontend with single sign-on (#6, #5).<br>2. The micro-frontend calls the API Gateway with the OIDC access token issued by IB Core (#7). The gateway validates the token (#8).<br>3. The Catalog & Rate Service returns the products with personalized rates (#13): the base rate grid plus the premium for the client's risk category. Both are taken from the cache (#17). The ABS sends only the category, never the raw credit data (#21). |
| UC6 | IB client; Deposit Micro-frontend; API Gateway; Deposit Application Service; OTP Service; Notification Service; SMS Gateway; Kafka | Submit a deposit application in the IB and confirm it with an SMS code (F6, F7, F16, R4) | 1. The client chooses a deposit, a debit account and an amount. The micro-frontend shows the client's accounts and balances from IB Core (#7, #8).<br>2. The Deposit Application Service checks the amount against the product limits and the balance and creates a draft application (#14, #18).<br>3. The service requests an OTP for this application (#15). The OTP Service generates a 6-digit code bound to the application ID, amount and account. It stores only a hash with a 5-minute TTL (#19) and sends the SMS "Deposit X, amount Y, code Z" through the Notification Service (#16, #34, #35).<br>4. The client enters the code. The service checks it with the OTP Service (#15, max 3 attempts) and changes the status to "Submitted". The status change and the outbox event are saved in one transaction (#18).<br>5. The outbox relay publishes `ApplicationSubmitted` to Kafka (#22). The client immediately sees "Application No. X submitted". The ABS is not called synchronously. |
| UC7 | ABS Integration Adapter; ABS Integration API; ABS; Deposit back office; Kafka; Deposit Application Service; Notification Service | Back office processes the IB application and confirms the rate (F8, F9, F10, P4) | 1. The ABS Integration Adapter reads `deposit.applications` at a controlled rate (≤ 5 applications/s, slows down when ABS response time grows) and creates the application through the ABS Integration API (#24, #26, #27). Repeated messages do not create duplicates because of the idempotency key.<br>2. The deposit back office sees the new application in the application queue of the ABS Desktop Client (#30, #28), checks it and confirms the rate, or rejects the application with a reason.<br>3. The ABS writes the status change into the outbox. The adapter publishes it to `deposit.application-status` (#26, #24).<br>4. The Deposit Application Service updates the status (#22) and the Notification Service sends the "rate confirmed" SMS (#23, #34). |
| UC8 | Deposit back office; ABS; ABS Integration Adapter; Kafka; Deposit Application Service; Notification Service | Open the deposit and notify the client (F9, F10) | 1. After the rate is confirmed, the back office opens the deposit in the ABS: the deposit account is created and the amount is debited from the client's account (#30, #28). The ABS also checks the balance at this moment.<br>2. The ABS publishes "deposit opened" through the outbox and the adapter (#27, #26, #24).<br>3. The application gets the final status (#22) and the client gets the second SMS (#23, #34).<br>The design keeps steps 1–2 inside the ABS, so later automatic approval of standard applications does not change the channels. |
| UC9 | Credit back office; Deposit back office; ABS Desktop Client; ABS; ABS Integration Adapter; Kafka; Catalog & Rate Service | Maintain and publish deposit rates (F8, P6) | 1. Every day the credit back office enters the Bank of Russia key rate and the base rate grid in the new rate management screens of the ABS, instead of an Excel file (#31, #28). It also maintains the risk premiums per risk category.<br>2. The deposit back office reviews and approves the new rates in the ABS (#30). Access is role-based, and all changes are audited.<br>3. On approval, the ABS writes the rates into the outbox. The adapter publishes them to the compacted topic `deposit.rates`. The nightly job of the credit section publishes client risk categories to `client.rate-category` (#27, #26, #24).<br>4. The Catalog & Rate Service updates the cache (#21, #17). The website and the IB show the new rates within 5 minutes. |
| UC10 | IB client; Deposit Micro-frontend; API Gateway; Deposit Application Service | Track the application status in the IB (F11) | 1. The client opens "My applications" in the Deposits section (#5).<br>2. The micro-frontend requests the statuses (#7, #14). The Deposit Application Service returns them from its own database (#18), which is updated from Kafka events (#22), not by queries to the ABS.<br>3. Statuses: Submitted → Rate confirmed → Deposit opened, or Rejected with a reason. |

### **Non-functional requirements**

The architecturally significant requirements (ASR) from [Task2](../Task2/FURPS+.md). The code in brackets is the requirement code in the FURPS+ table.

| **#** | **Requirement** |
| :-: | :- |
| NFR1 | **Availability 99.9% 24/7** (≤ 43.8 min of downtime per month) for the website, the IB and the new deposit services, measured as the share of successful requests plus synthetic checks. Releases cause no downtime. (R1, R6) |
| NFR2 | **Two data centers.** If one DC fails, the services keep working in the other one at full peak load. RTO ≤ 15 min; RPO = 0 for applications already confirmed to the client. (R2, +R5) |
| NFR3 | **No synchronous dependency of the channels on the ABS.** The ABS database is overloaded and scales only vertically. Applications are accepted even when the ABS is down. Interaction goes through Kafka. (R3, R5, +R3) |
| NFR4 | **Controlled ABS load:** ≤ 5 applications/s from the queue with back-pressure, ≤ +5 percentage points of ABS database CPU at peak. (P4) |
| NFR5 | **Response time in milliseconds** at the API Gateway under target load: lists and rates p95 ≤ 100 ms; personalized rates p95 ≤ 200 ms; submit and OTP check p95 ≤ 300 ms; status p95 ≤ 150 ms; no request > 1 s. IB reference data p95 ≤ 100 ms (fixes the current > 1 s problem in payments). (P1, P2) |
| NFR6 | **Horizontal scaling** of all new services: stateless, ≥ 2 instances per DC, autoscaling, L7 balancing and GSLB between DCs. Target load: 200 RPS for rates on each channel, 500 RPS for IB reference data, 2× in load tests. (P3, P5) |
| NFR7 | **Reliable, exactly-once (for the client) delivery:** transactional outbox, Kafka `acks=all` with idempotent producer, replication factor 3, idempotent consumers, dead letter topics with alerts. (R4) |
| NFR8 | **Security:** TLS 1.2+/1.3 on all external channels, HSTS, WAF; mTLS between internal services and on the website → bank link; OIDC tokens (≤ 15 min) for deposit APIs; OTP bound to the operation; encryption at rest; secrets in a vault; append-only audit log kept 5 years; anti-bot protection and rate limits on the website. (F12–F16) |
| NFR9 | **Separation of deposit and credit data:** only the computed risk category leaves the credit section. (+R8) |
| NFR10 | **SMS confirmation and notifications are built by the bank's own team.** No changes to the ABS SMS module by the vendor. OTP SMS p95 ≤ 30 s, status SMS p95 ≤ 1 min, rate changes reach channels in ≤ 5 min. (F7, F10, S2, +R6, P6) |
| NFR11 | **Strangler approach for the IB:** deposit features are built as microservices next to the IB monolith; the monolith gets only minimal changes. (S1, +R4, +R10) |
| NFR12 | **Existing technologies first** (MS SQL, Oracle, .NET, React, PHP); Kafka for queues. Every new technology is justified in an ADR. (+R9, +R10) |
| NFR13 | **Extensibility:** new deposit products are configured without channel releases; API and event contracts are versioned (OpenAPI, AsyncAPI, schema registry). (S3, S6, S7) |
| NFR14 | **Observability:** central logs, metrics, OpenTelemetry tracing with a correlation ID through HTTP and Kafka headers, SLO alerts. (S4) |
| NFR15 | **Compliance:** 152-FZ (consent, personal data stored in Russia), Bank of Russia requirements and GOST R 57580.1. (+R7) |
| NFR16 | **UX:** the bank's design system, WCAG 2.1 AA, responsive web from 360 px, IB application in ≤ 3 screens. (U1–U5) |

### **Solution**

#### Diagrams

| File | Contents |
|---|---|
| [c4-context.png](c4-context.png) / [c4-context.drawio](c4-context.drawio) | C4 Level 1: System context. People, the bank systems that take part in online deposit opening, and external systems. |
| [c4-container.png](c4-container.png) / [c4-container.drawio](c4-container.drawio) | C4 Level 2: Containers. The Internet Bank and the ABS are shown in detail, as required; the Website is also detailed. Arrow numbers match the "Integrations" table below. The "Use case coverage" box at the bottom of the diagram lists the arrows each use case (UC1–UC10) goes through. |

The `.drawio` files open in [app.diagrams.net](https://app.diagrams.net) or draw.io Desktop. Color code on both diagrams: dark blue is new, light blue with an orange border is changed, grey is existing without changes, light grey with a dashed border is external. A dashed grey arrow is an existing integration that does not change.

#### Summary of the decision

1. **Deposit features of the IB are built as separate .NET 8 microservices next to the IB monolith (strangler pattern).** These are the Catalog & Rate Service, the Deposit Application Service, the OTP Service and the Notification Service, behind a new API Gateway. The IB monolith only gets small changes: a "Deposits" menu item, token issuing (OIDC) and reading reference data through an API. This is how we get Kafka (the IB platform cannot use it), independent scaling and fewer changes on the vendor platform. (NFR3, NFR6, NFR11)
2. **The website and the IB share the same deposit services.** The website backend calls the API Gateway for public rates and for application submission. There is one source of rates and one place where applications are stored, for both channels. (F1, F2)
3. **The ABS stays the master of rates, applications and deposits, and the back office works in the ABS.** Rate management and the rate approval process move from Excel and e-mail into new ABS screens (Delphi) and tables (Oracle), as the business proposed. Back-office work adds little load to the ABS. (F8, F9)
4. **The channels never call the ABS synchronously.** Data from the ABS reaches the channels as Kafka events: rates, risk categories, statuses and reference data. The channels keep their own read models (Redis, MS SQL). Applications go to the ABS through Kafka and the ABS Integration Adapter, at a rate the ABS team controls. (NFR3, NFR4, NFR5)
5. **The ABS is connected to Kafka through an outbox, not through vendor changes.** The ABS team (in-house) adds PL/SQL packages (the ABS Integration API) and an outbox table, which is written in the same transaction as the business change. The new ABS Integration Adapter reads the outbox and publishes events, and it consumes applications and calls the PL/SQL API. (NFR7)
6. **Deposit and credit data stay separated in the system.** A new view/job in the credit section computes only the client risk category. The deposit side and the IB receive the category, never the raw credit data. The personalized rate = base rate for the product and term + premium for the category. (NFR9)
7. **OTP and SMS notifications are built by the bank's team.** The OTP Service and the Notification Service send SMS directly through the existing SMS Gateway. The ABS SMS module is not used and not changed. (NFR10)
8. **All components run active-active in both DCs.** Kafka is one cluster stretched across the DCs (RF = 3, `min.insync.replicas = 2`). The MS SQL database uses Always On availability groups with synchronous commit, and Redis uses replication. GSLB sends traffic to both DCs. (NFR1, NFR2)

#### Containers

| Container | Technology | Status | Responsibility |
|---|---|---|---|
| Website Frontend | React.js | Changed | Deposit list with rates and update time, application form with consent and CAPTCHA. Design system. |
| Website Backend | PHP | Changed | Gets public rates and sends applications to the API Gateway over mTLS. Keeps no data. |
| Deposit Micro-frontend | React, design system | New | The "Deposits" section of the IB: deposits with personalized rates, application form, OTP entry, statuses. Opened from the IB with SSO. A separate front end means the deposit UI can be released without a release of the vendor monolith. |
| IB Web UI | ASP.NET MVC 4.5 | Changed | Existing IB pages. New: "Deposits" menu item. |
| IB Core | .NET 4.5, vendor platform, MS SQL | Changed | Login, payments, accounts (as today). New: issues OIDC access tokens for the deposit services (≤ 15 min) and publishes a JWKS endpoint; provides accounts and balances to the deposit services; reads reference data from the Catalog & Rate Service instead of the ABS database (fixes P2). |
| API Gateway + WAF | YARP on .NET 8 | New | Single entry point for the website and the micro-frontend: TLS termination, WAF, token validation, rate limits (website applications: 5 per hour per IP; the Deposit Application Service also allows 3 per day per phone number), routing, correlation ID, request metrics. |
| Catalog & Rate Service | .NET 8, Redis | New | Products, public rates, personalized rates, reference data. Product parameters (term, minimum and maximum amount, rates) come from the ABS together with the rates, because the deposit is opened there; texts and display order are configuration of this service. A new product therefore needs no release of the website or the IB (S6). Builds its read model in Redis from compacted Kafka topics, so it can always be rebuilt from Kafka. If rates are older than 24 hours, it marks the products as unavailable for applications (R5). |
| Deposit Application Service | .NET 8, MS SQL | New | Website leads and IB applications: checks, idempotency, status model, outbox. Sends website leads to the Call Center System. Consumes status events from the ABS. |
| OTP Service | .NET 8, Redis | New | 6-digit codes bound to the operation; only a hash is stored, TTL 5 min, 3 attempts, 1 resend per 60 s. |
| Notification Service | .NET 8, MS SQL | New | SMS templates, sending through the SMS Gateway, deduplication by event ID, sending log. Consumes status events; gets OTP requests synchronously from the OTP Service (an OTP must not wait in a queue). |
| Deposit Services DB | MS SQL, Always On | New | Applications, statuses, outbox, SMS log. Separate schema for each service. |
| Catalog Cache / OTP Store | Redis | New | In-memory read model of rates and reference data; OTP state with TTL. Two separate clusters, because they have different data and criticality. |
| Kafka | Apache Kafka, 2 DCs | New | Topics: `deposit.applications`, `deposit.application-status`, `deposit.rates` (compacted), `client.rate-category` (compacted), `reference-data` (compacted), DLQ topics. Schemas in a schema registry. |
| ABS Integration Adapter | .NET 8, ODP.NET | New | Kafka ↔ ABS bridge, owned by the ABS team. Throttled consumer with back-pressure, outbox relay. |
| ABS Integration API | PL/SQL packages + outbox table | New | The only entry point to the ABS for new integrations: create application, read status. Business changes and outbox records are written in the same transaction. |
| ABS Database | Oracle, PL/SQL | Changed | New tables and logic: rate grid and its versions, rate approval process, risk premiums, online applications with the channel and the website application ID. New view in the credit section that returns only the risk category. |
| ABS Desktop Client | Delphi | Changed | New screens: rate management, application queue, rate confirmation, search of website applications. Role-based access for the deposit and the credit back offices. |
| Call Center System | Vendor platform (Java, PostgreSQL) | Changed (configuration) | Uses the existing, unused CRM functions to store website leads; an inbound REST API for leads; the application ID in inquiries to the ABS. |
| SMS Gateway | — | Existing | Sends SMS through the telecom operator. |

#### Integrations

Arrow numbers on the [container diagram](c4-container.png).

| # | From → To | Data | Protocol / mode | Use cases |
|:-:|---|---|---|---|
| 1 | Prospective client → Website Frontend | Deposit pages, application form | HTTPS (TLS 1.2+/1.3) | UC1, UC2 |
| 2 | Website Frontend → Website Backend | Same | JSON over HTTPS | UC1, UC2 |
| 3 | Website Backend → API Gateway | Public rates; website application | REST, mTLS, sync | UC1, UC2 |
| 4 | IB client → IB Web UI | Login, existing IB | HTTPS | UC5 |
| 5 | IB client → Deposit Micro-frontend | Deposits, application, OTP, statuses | HTTPS | UC5, UC6, UC10 |
| 6 | IB Web UI → Deposit Micro-frontend | Opens the Deposits section, SSO | Browser redirect, OIDC | UC5 |
| 7 | Deposit Micro-frontend → API Gateway | Deposit API calls | REST + OIDC access token | UC5, UC6, UC10 |
| 8 | API Gateway → IB Core | Token validation (JWKS); client accounts and balances | REST, mTLS | UC5, UC6 |
| 9 | IB Web UI → IB Core | Existing IB functions | Existing | — |
| 10 | IB Core → IB Database | Existing IB data | SQL (existing) | — |
| 11 | IB Core → ABS Database | Payments and current accounts. **Not used for deposits.** | Direct SQL (existing) | — |
| 12 | IB Core → Catalog & Rate Service | Reference data for payments (fixes P2) | REST, mTLS | — |
| 13 | API Gateway → Catalog & Rate Service | Products, public and personalized rates | REST | UC1, UC5 |
| 14 | API Gateway → Deposit Application Service | Submit lead / application, confirm OTP, statuses | REST, idempotency key | UC2, UC6, UC10 |
| 15 | Deposit Application Service → OTP Service | Issue and verify OTP for an application | REST, sync | UC6 |
| 16 | OTP Service → Notification Service | Send OTP SMS (priority) | REST, sync | UC6 |
| 17 | Catalog & Rate Service → Catalog Cache | Read model | Redis protocol, TLS | UC1, UC5, UC9 |
| 18 | Deposit Application Service → Deposit Services DB | Applications, statuses, outbox | SQL, TLS | UC2, UC6, UC10 |
| 19 | OTP Service → OTP Store | Code hashes, attempt counters | Redis protocol, TLS | UC6 |
| 20 | Notification Service → Deposit Services DB | SMS log, deduplication | SQL, TLS | UC3, UC4, UC6–UC8 |
| 21 | Catalog & Rate Service → Kafka (consumer) | Consumes `deposit.rates`, `client.rate-category`, `reference-data` | Kafka consumer, async | UC1, UC5, UC9 |
| 22 | Deposit Application Service ↔ Kafka | Publishes `deposit.applications`; consumes `deposit.application-status` | Kafka, outbox, async | UC3, UC4, UC6–UC8, UC10 |
| 23 | Notification Service → Kafka (consumer) | Consumes `deposit.application-status` | Kafka consumer, async | UC3, UC4, UC7, UC8 |
| 24 | ABS Integration Adapter ↔ Kafka | Consumes applications; publishes statuses, rates, categories, reference data | Kafka, async, throttled | UC3, UC4, UC7–UC9 |
| 25 | Deposit Application Service → Call Center System | Website lead (name, phone, product, application ID) | REST, mTLS, outbox with retries | UC2 |
| 26 | ABS Integration Adapter → ABS Integration API | Create application; read outbox | SQL calls to PL/SQL packages (ODP.NET) | UC3, UC4, UC7–UC9 |
| 27 | ABS Integration API → ABS Database | Applications, deposits, rates, outbox | PL/SQL | UC3, UC4, UC7–UC9 |
| 28 | ABS Desktop Client → ABS Database | Existing functions + new screens | Existing client-server | UC3, UC4, UC7–UC9 |
| 29–31 | Branch employee, deposit back office, credit back office → ABS Desktop Client | Work in the ABS | UI | UC3, UC4, UC7–UC9 |
| 32 | Call center manager → Call Center System | Lead cards, calls | UI | UC3 |
| 33 | Call Center System → ABS Database | Inquiry, now with the website application ID | Existing integration, new field | UC3, UC4 |
| 34 | Notification Service → SMS Gateway | SMS | HTTPS API or SMPP 3.4 | UC3, UC4, UC6–UC8 |
| 35 | SMS Gateway → Telecom operator | SMS | Existing | UC3, UC4, UC6–UC8 |

Arrow numbers on the [context diagram](c4-context.png). The same flows are grouped by system there.

| # | From → To | Meaning | Use cases |
|:-:|---|---|---|
| 1 | Prospective client → Website | Views deposits, submits an application | UC1, UC2 |
| 2 | IB client → Internet Bank | Personalized rates, application, OTP, status | UC5, UC6, UC10 |
| 3 | Website → Internet Bank (deposit services) | Public rates, website applications; REST, mTLS | UC1, UC2 |
| 4 | Internet Bank → Call Center System | Website leads; REST | UC2 |
| 5 | Call center manager → Call Center System | Processes website leads | UC3 |
| 6 | Call Center System → ABS | Inquiry with the application ID (existing integration) | UC3, UC4 |
| 7 | Internet Bank ↔ ABS | Applications, statuses, rates, risk categories, reference data; asynchronous through Kafka | UC5–UC10 |
| 8 | Internet Bank → ABS | Existing direct SQL, only for payments and current accounts | — |
| 9 | Internet Bank → SMS Gateway | OTP and status SMS | UC3, UC4, UC6–UC8 |
| 10 | SMS Gateway → Telecom operator | SMS delivery | UC3, UC4, UC6–UC8 |
| 11–13 | Branch employee, deposit back office, credit back office → ABS | Deposit opening, rate approval, application processing, rate maintenance | UC3, UC4, UC7–UC9 |
| 14 | Credit back office → Bank of Russia | Reads the key rate for the daily rate grid (manual) | UC9 |
| 15 | Prospective client → Branch employee | Visits a branch for identification | UC4 |

#### Why these decisions and technologies

- **.NET 8 for the new services and the gateway (YARP).** The IB is a .NET platform, so the IB team already knows C#; .NET 8 runs on Linux and in containers, which the .NET 4.5 platform cannot do, and has mature Kafka (Confluent.Kafka), Redis, Oracle (ODP.NET) and OpenTelemetry clients. A YARP gateway keeps the stack in one language instead of adding a new product (+R9).
- **MS SQL for the application store.** The bank already runs MS SQL. Always On with synchronous commit gives RPO = 0 between DCs for confirmed applications (R2).
- **Kafka** is the bank's preferred broker (+R10). It replaces direct ABS reads: compacted topics hold the latest rates and categories, so a read model can be rebuilt at any time, and topic retention (≥ 7 days) covers long ABS outages (R5).
- **Redis** is a new technology and is justified here: personalized rates and reference data under 500 RPS with p95 ≤ 100 ms need an in-memory store, and OTPs need a TTL. A MS SQL cache was considered but it does not meet the latency targets as easily and gives no native TTL.
- **Transactional outbox** on both sides (MS SQL and Oracle) instead of writing to the database and to Kafka separately, so that an event is never lost or sent for a change that was rolled back (R4).
- **Outbox in Oracle + .NET adapter instead of CDC.** It uses only PL/SQL, which the in-house ABS team knows, and needs no new licenses (GoldenGate) or new Oracle-level tools (Debezium/LogMiner).
- **Container platform.** The new services are stateless containers, ≥ 2 instances per DC. The choice of a platform (Kubernetes/OpenShift or VMs behind a load balancer) is infrastructure, not part of this ADR. It should be decided in a separate ADR together with the infrastructure team (+R9).
- **Observability** (S4): OpenTelemetry in all .NET services and the adapter; the correlation ID from the gateway is passed in HTTP and Kafka headers, so one application can be traced from the website to the ABS.
- **Audit log** (F15): every new service and the adapter write audit records (who, what, old and new value, when in UTC, channel, IP, correlation ID) to an append-only table in the same transaction as the change; the ABS logs rate and application changes in its own audit tables. Audit records are shipped to the bank's central log store and kept for 5 years. Personal data in technical logs is masked.

#### IT teams that take part

| Team | Scope of work |
|---|---|
| Internet Bank team (+ IB platform vendor for small changes) | Deposit micro-frontend; Catalog & Rate, Deposit Application, OTP and Notification services; API Gateway; changes in IB Core: menu item, OIDC tokens, accounts API, reference data via API. |
| ABS team (in-house) | Rate management and approval in Oracle/PL/SQL and Delphi; application queue screens; risk category view in the credit section; ABS Integration API and outbox; ABS Integration Adapter; load tests of the ABS. |
| Website team | Deposit pages and application form in React/PHP; integration with the API Gateway. |
| Call center system team / vendor | CRM card configuration for website leads; inbound REST API; application ID in ABS inquiries. |
| Infrastructure / platform team | Kafka cluster across 2 DCs, schema registry, Redis, MS SQL Always On, container platform, GSLB / L7 balancers, observability stack, vault. |
| Information security | WAF rules, TLS/mTLS and certificates, OIDC setup, audit log, penetration test, 152-FZ and GOST R 57580.1 compliance check. |
| SMS Gateway support (bank IT) | Access for the Notification Service, SMS templates, sending limits. |
| Business: deposit and credit back offices, call center | New process in the ABS and the Call Center System, rate approval rules, SMS texts, acceptance testing. |

### **Alternatives**

| # | Alternative | Why it was not chosen |
|:-:|---|---|
| A1 | **Build the deposit features inside the IB monolith** and integrate with the ABS like today, through the Oracle database. | The IB platform cannot use Kafka (+R4); every change depends on the vendor; the monolith cannot be scaled separately and runs in one DC; direct reads from the overloaded ABS database break the latency and availability targets (P1, R1, +R3). |
| A2 | **Synchronous REST API on top of the ABS**, called by the channels for rates, applications and statuses. | Simpler, but the availability of the IB would depend on the ABS, and every rate request would load the ABS database, which can only scale vertically (R3, +R3). It also needs ABS load capacity that the ABS team says it does not have. |
| A3 | **A separate rate management system** (new microservice with its own UI for both back offices), instead of rate management in the ABS. | Gives a modern UI, but creates a second master of rates and applications next to the ABS, requires synchronization and a new UI for the back offices. The business proposed the ABS, where both back offices already work and where the deposits are opened. It can be reconsidered when rate management grows beyond the MVP. |
| A4 | **CDC from Oracle (Debezium or GoldenGate)** instead of the outbox table. | Needs no changes in the PL/SQL code, but needs new technology and licenses (+R9), publishes table-level changes instead of business events, and adds load on the database redo logs. |
| A5 | **Use the ABS SMS module** for OTP and notifications. | Requires changes by the vendor, which the business wants to avoid (+R6, S2), and puts OTP in the critical path of the overloaded ABS. |
| A6 | **Website sends applications directly to the Call Center System.** | Simpler, but the application is lost if the Call Center System is down, the link between the application and the deposit cannot be tracked (F4), and the website application has a different model than the IB one. |
| A7 | **Another broker** (RabbitMQ, IBM MQ) or **no broker** (database polling). | Kafka is the long-term choice of the bank (+R10); compacted topics and event replay are needed for read models. Polling the ABS is exactly the load we want to remove. |

**Disadvantages, limitations, risks**

| # | Risk / limitation | Impact | Mitigation |
|:-:|---|---|---|
| 1 | **Eventual consistency.** Rates, statuses and categories in the channels are copies; the client may see a rate that changed a few seconds ago, and the application status is not updated instantly. | A client can apply at an old rate. | The rate version is stored in the application; the ABS checks it and the back office confirms the final rate (UC7). Rates older than 24 h block applications (R5). The UI says "rates as of …". |
| 2 | **Changes on the IB vendor platform** (menu item, OIDC tokens, accounts API, reference data via API) may still need the vendor. | Time and cost; dependence on the vendor. | Keep the IB changes as small as possible and agree on them with the vendor early. Fallback: a separate identity provider with SSO from the IB session. |
| 3 | **New technologies for the bank** (Kafka, Redis, container platform, YARP) and a new operations model. | Learning curve, operations risk, cost of infrastructure in 2 DCs. | Start with a managed/supported distribution, training, runbooks (S3). Every new technology gets its own ADR. |
| 4 | **The ABS team becomes the critical path**: rate management, application queue, risk categories, outbox and adapter are all in the ABS. | Delays of the whole MVP. | Plan the ABS backlog first; define the event contracts (AsyncAPI) at the start so other teams can work with mocks. |
| 5 | **ABS remains a single point of capacity**: it scales only vertically; the throttle (5 applications/s) can create a backlog during campaigns. | Applications wait longer in the queue (the client already sees "Submitted"). | Back-pressure, queue monitoring, Kafka retention ≥ 7 days; load tests together with the ABS team; automated approval later. |
| 6 | **Availability of 99.9% for the whole IB is not reached by this ADR alone.** The IB monolith still runs in one DC; only the new deposit services are active-active. | The client cannot log in to the IB if the monolith's DC fails, even though the deposit services are up. | Move the IB monolith to active-passive in the backup DC as the next step (R1 in stages, as agreed in Task2). |
| 7 | **The existing direct SQL link** from IB Core to the ABS database stays for payments. | Load and coupling remain for the old functions. | Out of the MVP scope; the reference data part moves to the API now (#12); the rest moves to Kafka/API later. |
| 8 | **Distributed transactions across services** (application, OTP, outbox, ABS). | Complex error handling and debugging. | Outbox + idempotency keys + DLQ; end-to-end tracing with a correlation ID; runbooks for replaying DLQs. |
| 9 | **Personal data on the website** (name, phone of non-clients) moves through new components. | Regulatory risk (152-FZ). | Consent on the form, data stored only in Russia, masking in logs, retention rules for leads that did not become clients. |
| 10 | **Integration with the Call Center System** depends on its vendor providing an inbound API and CRM configuration. | Website leads can be delayed. | Outbox with retries; if the API is late, a temporary manual lead list for call center managers. |
