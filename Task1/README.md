# Task 1. IT Landscape Map and Application Integration Diagram

| File | Contents |
|---|---|
| [it-landscape-map.drawio](it-landscape-map.drawio) / [.jpeg](it-landscape-map.jpeg) | Current IT landscape map. Rows are organizational units; columns are level-2 business capabilities from the Business Capability Map; cells show the systems and manual tools each unit uses. |
| [application-integration.drawio](application-integration.drawio) / [.jpeg](application-integration.jpeg) | Application integration diagram (AS-IS) with all participants of the deposit opening process. Each scenario has its own color and step prefix (see below). |

The `.drawio` files open in [app.diagrams.net](https://app.diagrams.net) or draw.io Desktop.

## Scenarios on the integration diagram

| Prefix | Scenario |
|---|---|
| **A** (blue) | Client called the call center first: inquiry → ABS → deposit manager sets the rate → SMS to the client → client comes to a branch |
| **B** (orange) | Client walks into a branch without a call: branch employee e-mails the back office, the deposit manager calculates the rate in Excel and replies by e-mail |
| **C** (purple) | Special rate (inside B): deposit manager requests client risk data from the credit department by e-mail |
| **D** (red) | Daily rate preparation: Central Bank rate → Excel → e-mail to the deposit back office |
| **F** (green) | Finish of A and B: branch employee creates the deposit in the ABS and uploads the signed documents |

## Business Capability Map used

| Sales and Marketing | Operational Process Support |
|---|---|
| Branch network sales | Deposit process servicing |
| Call center sales | Credit process servicing |
| Digital customer notifications | Contract management |

## What prevents deposits and savings accounts from becoming a fully digital product

1. **Manual pricing.** Rates are calculated in Excel from the Central Bank refinancing rate and sent by e-mail once a day. No system can return a rate through an API.
2. **The back office takes part in every deposit opening.** The front office requests the rate by e-mail. For a special rate, the deposit manager also has to request client risk data from the credit department. The client waits from 20 minutes to 1 hour.
3. **Deposit / credit data separation is organizational** (enforced via e-mail), not built into the systems, so special-rate calculation cannot be automated without changes.
4. **The internet bank is integrated with the ABS directly through the Oracle database.** There is no service layer (API) for new products, and the internet bank core depends on the vendor.
5. **Online channels barely sell.** The website only shows marketing information and has no integrations; the internet bank only supports payments and current accounts.
6. **Call centers are fragmented.** The call center system only passes inquiries to the ABS (the platform's CRM features are unused); the partner call center works from scripts with no integration with the bank.
7. **No modern mobile application.** The bank only has a website and a web internet bank; there is no mobile app (iOS / Android), which is the main channel for the younger clients the bank wants to attract.
