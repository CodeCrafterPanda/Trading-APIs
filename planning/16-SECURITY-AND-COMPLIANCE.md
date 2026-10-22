# Security, market rules and launch evidence

This document distinguishes researched requirements from account certification. Research was completed 21 October 2026 and this pack issued 22 October. No account has been approved, connected or traded by this task. The initial assumption is an investor's own self-developed platform; provider/commercial use needs a different review.

## Retail algo framework

SEBI's September 2025 extension states the framework applies to all stockbrokers from **1 April 2026**. NSE standards and later clarifications distinguish below-threshold self-developed algorithms, above-threshold registration and provider algorithms. All client API orders are treated as algo orders for tagging; staying below a rate threshold does not remove identity, tagging, IP, authentication or broker risk requirements. [SEBI extension](https://www.sebi.gov.in/sebi_data/attachdocs/sep-2025/1759232056254.pdf), [NSE standards](https://nsearchives.nseindia.com/content/circulars/INVG67858.pdf), [NSE FAQ](https://nsearchives.nseindia.com/web/sites/default/files/inline-files/FAQ_Retail%20Algo_03112025_NSE.pdf).

NSE's April 2026 system-audit terms repeat daily logout and whitelisted IP controls, distinguish tech-savvy clients from provider hosting, and clarify generic tagging below threshold versus registration above threshold. The referenced initial threshold is not exceeding 10 orders/second, with scope/counting clarified through exchange/broker evidence. This is not a daily trade cap. Broker endpoint limits remain separately enforced. [NSE audit terms, clause 27](https://nsearchives.nseindia.com/content/circulars/INSP73850.pdf).

ComplianceProfile stores account category, broker app type, exchanges/segments, approved primary/secondary static egress, algo tag/name/registration, order-message counting policy, allowed session/order types, effective dates, evidence references, reviewer and review expiry. Unknown required evidence blocks mutations. Provider/black-box/third-party operation cannot inherit the private-client profile.

## Effective-dated sessions

| Rule set | Effective status at research date | Required implementation |
|---|---|---|
| Current CAS-related changes | Live from 3 August 2026 | Instrument eligibility and phase-specific rules |
| Current NSE normal derivatives close | Current timings page lists 15:40 | Do not assume universal 15:30 |
| Upstox current CAS sequence | Eligible cash continuous close 15:15; auction phases thereafter, random close near 15:28–15:30 | Consume phase/eligibility; stop/GTT/IOC/slicing restrictions |
| SEBI 15 October circular | Most changes effective 7 December 2026 | Pending rule set, not activated early |
| Carryover paragraph 4.8 | Effective 8 February 2027 | Separate later activation |

Current evidence: [NSE timings](https://www.nseindia.com/static/market-data/market-timings), [NSE July circular/live date](https://nsearchives.nseindia.com/content/circulars/FAOP75472.pdf), [current CAS SOP](https://nsearchives.nseindia.com/content/circulars/CMTR73362.pdf), [Upstox CAS support](https://upstox.com/developer/api-documentation/announcements/closing-auction-session/).

The prospective October circular schedules CAS 15:30–15:40, stock derivatives to 15:45, index derivatives to 15:45 except expiring contracts at 15:30, and other settlement/post-close changes. Preserve future rules with activation gates requiring operative exchange notices and broker support. [SEBI October circular](https://www.sebi.gov.in/sebi_data/attachdocs/oct-2026/1792079571420.pdf).

NSE's revised derivatives pre-open page explicitly permits algo market orders in its initial five-minute phase, while older FAQ wording is broader. Therefore order-type policy is effective-dated and phase-specific, with LIMIT/SL-LIMIT initial automation and separately certified exceptions. Do not infer global permission from a schema enum or a generic prohibition from an older paragraph. [NSE derivatives pre-open](https://www.nseindia.com/static/products-services/equity-derivatives-pre-open-session).

BSE's primary notice fetch was blocked during research. BSE-specific current operational circulars, exact session behavior and approval route are an external gate; NSE text is not substituted as proof. Upstox's BSE_FO behavior is documented broker evidence only. Holidays, special sessions, halts and broker square-off deadlines require daily validated calendar refresh.

## Credential and application security

Secrets use encrypted server-side storage with envelope encryption and a managed key service. Store token metadata separately from secret values. Each adapter receives only its account's necessary secret handle; UI and analytical workers receive no broker credentials. Restrict egress to required broker endpoints and approved static addresses. Rotate keys/tokens under a single credential owner.

Default auth uses broker-managed OAuth/consent/2FA. Documented TOTP endpoints can be supported behind an approved capability, but unattended PIN/TOTP seed storage is disabled by default. Operator-supplied secret inputs are ephemeral and redacted. Dhan token-origin renewal restrictions and Upstox token-class/expiry rules are defined in [01](01-API-RESEARCH.md); do not invent refresh grants.

Platform login uses strong authentication, short-lived secure HttpOnly/SameSite sessions, CSRF protection for cookie-authenticated mutations and role scopes: VIEW, OPERATE, CONFIGURE_RISK, ADMINISTER_CREDENTIALS. Risk-limit increases and strategy activation are auditable owner actions. This is application authorization, not repeated conversational approval for routine implementation work.

Validate OAuth state/nonce and registered redirects. Never log authorization URLs/query strings containing secrets. Redact headers, request bodies, crash dumps and diagnostic payloads. Encrypted backups must not include plaintext keys alongside ciphertext. Dependency/SBOM review, signed release artifacts and least-privilege service identities are release gates.

## Webhooks and untrusted input

Upstox's inspected webhook page supplies no verified signing scheme; Dhan postback signing was likewise not established. Use TLS, a controlled ingress, random endpoint secret if broker configuration permits, strict body limits/schema/account validation, replay-deduplication hashes and rate controls. These measures do not turn an unsigned payload into cryptographic broker proof. Corroborate suspicious state through authenticated stream/REST before releasing risk. Never let webhook fields trigger arbitrary URLs, commands or new orders. [Upstox webhook](https://upstox.com/developer/api-documentation/webhook/), [Dhan postback](https://dhanhq.co/docs/v2/postback/).

## Audit, retention and data rights

Audit intent, strategy/config versions, data quality, risk checks, command payload hash, broker results, fills, modifications, cancels, protection, operator actions and reconciliation. Append-only exports include UTC time, actor and trace; secret values are excluded. Five-year application retention is a proposed operational policy pending applicable legal review; broker audit requirements are not automatically identical to private application duties.

Broker data access does not establish public redistribution, commercial dashboards, index constituent/weight licensing or perpetual storage rights. Keep license evidence by dataset and use case. Public/multi-user exposure is blocked until cleared. [NSE index licensing](https://www.niftyindices.com/offerings/index-licensing).

## Launch gate register

| Gate | Owner/evidence required | Blocks |
|---|---|---|
| G01 Account route | Owner/broker confirm self-developed vs provider/business category | All execution |
| G02 Auth and IP | Account identity, token flow, approved egress, daily logout behavior | Account mutations |
| G03 Algo controls | Tag/name/registration and message-count scope confirmed | Automated/API execution |
| G04 Segment/products | Enabled NSE/BSE products, shorts, delivery authorization | Affected routes |
| G05 Data entitlement | Paid/free tier, mode/connection quotas, deep data | Affected feed/scanners |
| G06 Units/schema | Actual tick/price/volume/Greek/time fixtures | Affected calculations/orders |
| G07 Session/BSE | Operative calendars/circulars and square-off deadlines | Affected venue/session |
| G08 Protections | Native stop/OCO/GTT/partial-fill/modify behavior | Strategies requiring them |
| G09 Data rights | Storage/display/derived/index licensing evidence | Disallowed use/distribution |
| G10 Durability/recovery | Fencing, synchronous ledger, restore/reconciliation tests | Live mutations |
| G11 Live conformance | Separately authorized commissioning evidence per broker/product | Production enablement |
| G12 Risk values | Owner-approved account/global limits and emergency policy | All execution |

Each gate has status OPEN|VERIFIED|EXPIRED|REVOKED, evidence URL/artifact hash, reviewer, verified_at and next_review_at. No gate is marked verified by this planning pack. Read-only functionality can proceed where its own permissions are satisfied; live routes remain fail-closed.

## Acceptance

Security tests cover token leakage, role/CSRF boundaries, webhook spoofing/replay, invalid OAuth state, secret rotation and revoked credentials. Operational tests cover future calendar activation, random auction close, special holidays, static-IP mismatch and expired compliance evidence. Restore/fencing drills prove an old gateway cannot continue sending after takeover. An audit reviewer must reconstruct any order from its recorded evidence without accessing a live secret.
