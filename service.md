# Help to Save API – Summary

## Purpose

This service is the back end of HMRC's **Help to Save API** (`individuals/help-to-save`, version 2.0), published on the HMRC API Platform. Help to Save is a government savings scheme for working people on low incomes. The API lets approved third-party software, outside HMRC's own digital platform, check whether a person can open a Help to Save account, open the account for them, and read the account's details.

Callers reach the service only through the API Platform gateway. They act either **on behalf of a signed-in individual** (user-restricted access through a Government Gateway login) or as a **trusted privileged application** that works for any person. The service has no UI. It holds no account data of its own: every business operation is passed on to the internal Help to Save back end, which talks to the scheme's systems of record (the eligibility service and the savings account provider, NS&I). The only data it keeps is a short-lived record of recent eligibility results, which links an eligibility check to the account-creation request that follows it.

## Main Capabilities

### 1. Eligibility checking

- A user-restricted caller can check the signed-in person's eligibility. The person's National Insurance number (NINO) always comes from their login, never from the caller.
- A caller can also check a NINO given in the URL:
  - for user-restricted callers, the NINO must match the signed-in person's NINO, or the request is refused
  - privileged applications may check any NINO
- Before running a full check, the service asks whether the person already has an account. If they do, it answers "account exists" straight away.
- Otherwise the back end's eligibility decision is mapped into a simple answer: *eligible or not*, plus whether the person receives Working Tax Credit and/or Universal Credit. If an account already exists, the answer is just "account exists". Any result combination it doesn't recognise counts as a back-end error.
- Every eligibility response carries a **correlation id**. The service stores the result against that id, together with the NINO, for one hour. Account creation must present the id.

### 2. Account creation

- The request holds a *header* (client code, version, timestamp, correlation id) and a *body* (the person's name, date of birth, NINO, address, contact preference, registration channel, and optionally nominated bank details).
- **Eligibility gate**: the correlation id must point to a stored eligibility result for the same NINO that is less than an hour old and says *eligible*. If the stored result says an account already exists, the service reports a conflict and creates nothing.
- **Identity binding**: for user-restricted callers, any NINO in the body must match the signed-in person's NINO, and the person's login must be at confidence level 200 or higher.
- **Pre-population**: for user-restricted callers, missing name, date of birth, NINO and address are filled in from the person's HMRC record. The contact preference and email can be defaulted from the registration channel (online means email, call centre means post).
- **Field rules**:
  - names may not contain digits, may not start with a special character and may not contain two special characters in a row; forenames may not contain apostrophes
  - phone numbers must contain a digit and no letters
  - the communication preference must be email (`02`) or post (`00`), and an email address is required and checked when the preference is email
  - the registration channel must be `online` or `callCentre`
  - the client code and version must match their formats
- **Bank details**, when given, are checked by a bank-account validation service (account number and sort code). Invalid details reject the request.
- If the person chose email contact, the email address is stored in the back end before the account is created.
- The service derives the eligibility reason sent to the account provider from the stored result: Universal Credit, Working Tax Credit, or both. It also records which client application asked.
- Outcomes are *created*, *already had an account*, *validation error* (including the account provider's own validation messages) and *server error*.

### 3. Account details

- Returns the signed-in person's account: account number, balance, how much can still be paid in this month, whether it is closed, bonus terms (dates and estimated bonus) and nominated bank account details.
- The NINO always comes from the login. If the person has no account, the service returns "not found".

### 4. API Platform publication

- Publishes the API definition (context, version, access type, enabled flag) and the API specification documents that the API Platform reads to publish documentation and enforce access.

## Cross-cutting Concerns

- **Authentication and authorisation**: every business endpoint needs a valid bearer token from either Government Gateway (individual) or a Privileged Application. Login types other than these two are refused. NINO-based operations are tied to the signed-in identity, except for privileged eligibility checks and account creation.
- **Fraud-prevention headers**: the versioned `Accept` header and four `Gov-Client-*`/`Gov-Vendor-*` headers must be present. Only their presence is checked, not their content.
- **Validation and transformation**: the service checks request structure and a set of field-format rules, removes separators from bank sort codes, and fills in or defaults missing details from the person's HMRC record. Most length and format limits are left to the back end and the account provider.
- **Data protection**: NINOs are kept out of logs by default (a config flag can turn NINO logging on) and are masked when back-end responses are logged. The email address is base64-encoded when passed to the back end. Eligibility results, with their NINO, are cached for one hour without encryption.
- **Monitoring**: the service keeps call timers and error counters per operation, and a counter of account creations per client code. Unexpected back-end failures are written to the log as "PagerDuty" alert lines.
- **Auditing**: the service raises no explicit audit events. Only the platform's standard request auditing applies.
- **Feature toggles**: the API version's access type and enabled flag, NINO logging, and the eligibility cache lifetime.

## Dependencies (generic)

| Kind | Role |
|---|---|
| Identity and authorisation service | Validates tokens; provides login type, NINO, confidence level, name, date of birth, address and email |
| Help to Save back end | Eligibility check, enrolment status, account creation, account retrieval, email storage, bank-detail validation (it fronts the eligibility service and the NS&I account provider) |
| Document database | One-hour cache of eligibility results keyed by correlation id |
| API Platform | Routes third-party traffic to the service and reads its API definition |

## In One Sentence

A thin, private API Platform gateway that lets approved third parties check Help to Save eligibility, open accounts and view account details for HMRC customers, by enforcing identity binding, an eligibility-before-creation handshake and request validation before calling the internal Help to Save back end.
