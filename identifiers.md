# Help to Save API – Business Identifiers

This document lists every identifier the service receives, derives, stores or sends on, and compares the validation rules from three sources:

- **Local**: checks in this repository (regexes, validators, readers).
- **Library**: checks built into domain types from dependencies. This service uses **none**: the NINO is a plain `String` (`type NINO = String`), and `uk.gov.hmrc.domain.Nino` is not used.
- **Contract**: patterns in the published API specification, `resources/public/api/conf/2.0/application.yaml`. The code never loads that file.

**Terminology:**

- In code, *NINO* appears as `nino`, `urlNino`, `bodyNINO` and `retrievedNINO`.
- The *eligibility correlation id* appears as `correlationId` (eligibility), as the response header `X-Correlation-ID`, and as `header.requestCorrelationId` (create account). All three are the same value.
- The *client code* (`header.clientCode`) also appears as `source` and inside `systemId`.

## Summary table

| Identifier | Identifies | Format (contract) | Local validation | Enters via | Main uses |
|---|---|---|---|---|---|
| NINO | Individual (tax/NI party) | `^(([A-CEGHJ-PR-TW-Z][A-CEGHJ-NPR-TW-Z])\d{6}[A-D]\|XX999999X)$`, length 9 | Path: local regex (suffix optional). Body: **none**. Auth: none (trusted). | Auth `nino` (trusted); path `/eligibility/:nino`; body `body.nino` | Upstream query/path; stored in the Mongo cache; bound to the correlation id; create payload; bank-check payload |
| Eligibility correlation id | One eligibility result / create-account handshake | `^[0-9a-f-]{1,36}$` | `UUID` reader | Generated (`UUID.randomUUID`); returned as `X-Correlation-ID`; comes back as `header.requestCorrelationId` | Mongo `_id`; `X-Correlation-ID` to upstream; logs |
| Get-account correlation id | One get-account call | — | — | Generated per call | `X-Correlation-ID` header and `correlationId` query param upstream |
| Client code | Calling third-party application | `^[A-Z0-9][A-Z0-9_-]+[A-Z0-9]$`, max 20 | Same regex, max 20 | `header.clientCode` | `source` upstream; `systemId = MDTP-API-<code>`; metric name; logs |
| System id | Originating system | — | — | Derived from the client code (create); config `system-id` (`MDTP-API-client`, get account) | `body.systemId` upstream; `systemId` query param |
| Request version | Caller's message version | `^[0-9]+.[0-9]+$`, max 5 | `^(\d\.)+\d+$`, max 10 | `header.version` | Validation and logs only |
| Credentials (providerId, providerType) | Auth login | — | providerType must be `GovernmentGateway` or `PrivilegedApplication` | Auth (trusted) | Access-type decision; providerId logged on rejection |
| Help to Save account number | NS&I savings account | maxLength 30 | — | Upstream response | Returned as `accountNumber` |
| Bank account number / sort code / roll number | Nominated bank account | `^.{1,8}$` / `^[0-9]{6}$` / `^([0-9a-zA-Z\/\.-]{4,18})$` | Separators stripped from the sort code; nothing else | Body `bankDetails`; upstream `nba*` | Bank-check payload; create payload; returned as `nba*` |
| Fraud-prevention ids (`Gov-Client-User-ID`, `Gov-Vendor-Instance-ID`, …) | End user / vendor instance | Defined by the HMRC fraud-prevention spec | Presence only | Request headers | Not used further by this service (forwarded only if `HeaderCarrier` allow-lists them) |

## Party identifiers: NINO

**Lifecycle**: issued by DWP/HMRC. It reaches the service in three ways:

1. **Auth retrieval** (`v2.Retrievals.nino`). This is trusted, and it is the only source for `GET /eligibility` and `GET /account`.
2. **Path** `/eligibility/:nino`. This is untrusted. Government Gateway callers must have it equal the auth NINO (exact, case-sensitive). Privileged callers may send any value.
3. **Body** `body.nino` on create account. This is untrusted:
   - **Government Gateway**: if present, it must equal the auth NINO (`fillInMissingDetailsGG` / `storeEmailThenCreateAccount`). If absent, it is filled in from auth.
   - **Privileged**: required. Its only binding is that it must equal the NINO stored with the eligibility correlation id.

**Uses**

- `GET /help-to-save/enrolment-status?nino=` and `GET /help-to-save/eligibility-check?nino=` (query parameters, URL-encoded by the `url` interpolator).
- `GET /help-to-save/{nino}/account`. The NINO is pre-interpolated into a plain string, so it is **not encoded**. It comes from auth.
- `GET /help-to-save/store-email?nino=`, and the `nino` field of the validate-bank-details payload.
- `payload.nino` of the create-account payload, which goes to NS&I.
- Mongo `api-eligibility` document, `data.eligibility.nino`, in plain text.
- Logs: added only when `nino-logging.enabled = true`. Upstream bodies are masked with `maskNino`.

**Validation comparison**

| Source | Rule | Applied to |
|---|---|---|
| Local | `^((?!(BG\|GB\|KN\|NK\|NT\|TN\|ZZ)\|(D\|F\|I\|Q\|U\|V)[A-Z]\|[A-Z](D\|F\|I\|O\|Q\|U\|V))[A-Z]{2})[0-9]{6}[A-D]?$` | Path NINO only, after the enrolment-status call |
| Library | none | — |
| Contract | `^(([A-CEGHJ-PR-TW-Z][A-CEGHJ-NPR-TW-Z])([0-9]{2})([0-9]{2})([0-9]{2})([A-D]{1})\|((XX)(99)(99)(99)(X)))$`, length 9 | `body.nino` (documentation only) |
| Log masking | `[A-Za-z]{2}[0-9]{6}[A-Za-z]` | Logged upstream bodies |

**Mismatches**, found by evaluating the regexes against format examples:

| Example shape | Local | Contract | Mask |
|---|---|---|---|
| `AB123456C` (standard) | ✅ | ✅ | ✅ |
| `AB123456` (no suffix) | ✅ | ❌ | ❌ (not masked) |
| `GB…`, `ZZ…` (invalid administrative prefixes) | ❌ | ✅ | ✅ |
| `XX999999X` (placeholder) | ❌ | ✅ | ✅ |
| lowercase | ❌ | ❌ | ✅ |

⚠️ **Observation:** the body NINO on create account has no local format check at all. Its validity rests on being equal to the auth NINO or to the NINO stored with a correlation id, which in turn was validated only on the `/eligibility/:nino` path.

⚠️ **Observation:** in `storeEmailThenCreateAccount`, the comparison `retrievedNINO.forall(_ === body.nino)` passes when auth returns **no** NINO and the body supplies every mandatory field, because the fill-in path, which would return 403, is skipped. The correlation-id NINO binding is then the only protection for a Government Gateway session without a NINO.

⚠️ **Observation:** on `/eligibility/:nino` the path NINO is sent to enrolment-status **before** it is validated. A malformed value still reaches the back end, URL-encoded.

## Entity and technical identifiers

### Eligibility correlation id

- **Issuer**: this service. It is created by `UUID.randomUUID()` in `checkEligibility*` and returned in the `X-Correlation-ID` response header (header name from `microservice.correlationIdHeaderName`).
- **Stored**: as the Mongo `_id` in `api-eligibility`, with a 1-hour TTL (`mongo-cache.expireAfter`), but only when the eligibility-check call ran. The enrolment-status short-circuit doesn't store one.
- **Comes back** as `header.requestCorrelationId`. It is parsed as a `java.util.UUID` (so it must be well-formed) and then looked up. The stored NINO must equal `body.nino`.
- **Forwarded** as `X-Correlation-ID` on store-email and create-account. That means the eligibility-check call, the email store and the account creation share one correlation id upstream.
- **Contract**: `^[0-9a-f-]{1,36}$`. The local UUID parser also accepts uppercase hex and non-canonical forms that Java's `UUID.fromString` allows.

⚠️ **Observation:** the entry is not consumed after a successful create. It can be replayed until it expires, and every replay is forwarded to upstream create-account again. What upstream returns for a replay (presumably 409) is not visible in this repository.

### Get-account correlation id

Generated for each call in `getAccount`. It is sent both as the `X-Correlation-ID` header and as the `correlationId` query parameter, and it is never returned to the caller.

### Client code and system id

- `header.clientCode` is caller-supplied. Local and contract rules are the same (the regex, max 20).
- It is used as:
  - `source` in the create-account payload
  - part of `body.systemId = "MDTP-API-" + clientCode`, overriding any caller value
  - a part of the metric name `api.create-account.<clientCode>.count`
- The code is **not** checked against the authenticated application, so a privileged application can claim any client code.
- Get account uses the fixed `systemId` from config key `system-id`.

⚠️ **Observation:** the client code is self-declared and goes straight into a metric name, so the number of distinct metrics is limited only by what the regex allows.

### Request version

The local rule `^(\d\.)+\d+$` allows only one digit before each dot (`1.0`, `1.2.3`) and up to 10 characters. The contract `^[0-9]+.[0-9]+$`, max 5, allows `10.0` but not `1.2.3`. The two rules accept different values.

### Account and bank identifiers

- **Account number**: issued by NS&I, returned as received from the back end, with no validation.
- **Nominated bank account details**, in `bankDetails` on input and `nba*` on output:
  - The sort code has the separators space, `-`, `–`, `−`, `—` and `_` removed, then goes to the external bank check without a local format check.
  - The account number, account name and roll number are not validated locally. The contract declares `^.{1,8}$`, `^.{2,35}$` and `^([0-9a-zA-Z\/\.-]{4,18})$`.

### Credentials

The providerType decides the access type. If it is unrecognised, the whole `Option[Credentials]`, including the providerId (the user's credential id), is written to a warning log.

## Party identifiers table

| Identifier | Frontend key(s) | Upstream key (per flow) | Local rule | Contract rule |
|---|---|---|---|---|
| NINO | path `{nino}`; `body.nino`; auth `nino` | enrolment/eligibility/store-email: query `nino`; get account: path segment; create: `payload.nino`; bank check: `nino` | Regex on the path only | 9 characters, prefix rules, suffix A–D, or `XX999999X` |
| Bank sort code | `bankDetails.sortCode` | `payload.bankDetails.sortCode`; bank check `sortCode` | Separators stripped | `^[0-9]{6}$` |
| Bank account number | `bankDetails.accountNumber` | `payload.bankDetails.accountNumber`; bank check `accountNumber` | none | `^.{1,8}$` |
| Roll number | `bankDetails.rollNumber` | `payload.bankDetails.rollNumber` | none | `^([0-9a-zA-Z\/\.-]{4,18})$` |

**Sort-code normalisation**: `sortCode.filterNot(Set(' ', '-', '–', '−', '—', '_').contains)`, applied in `CreateAccountBody.BankDetails.reads`. Other characters are kept.

## Validation coverage matrix

| Identifier | Format checked locally? | Checked against the authenticated user? | Encoded when put in a URL? |
|---|---|---|---|
| NINO (auth) | ❌ (trusted) | n/a | ⚠️ query: yes; get-account path: no (trusted source) |
| NINO (path) | ⚠️ yes, but after the first upstream call | ✅ Government Gateway; ❌ privileged (by design) | ✅ query parameter |
| NINO (body) | ❌ | ✅ Government Gateway when auth has a NINO; ⚠️ skipped when auth has none; privileged: via correlation id only | n/a (JSON body) |
| Eligibility correlation id | ✅ UUID | ⚠️ indirectly (stored NINO = body NINO) | n/a (header / Mongo key) |
| Client code | ✅ regex + length | ❌ (not tied to the API application) | n/a (body / metric name) |
| Version | ⚠️ differs from contract | n/a | n/a |
| Sort code | ⚠️ normalised only; external check | n/a | n/a (JSON body) |
| Account number / roll number | ❌ | n/a | n/a |

## Recommendations

1. Introduce one NINO rule, aligned with the contract or with `uk.gov.hmrc.domain.Nino`. Apply it to the path **and** the body NINO before any upstream call, and normalise the case.
2. In `storeEmailThenCreateAccount`, reject Government Gateway requests when auth returns no NINO, instead of letting `forall` pass on `None`.
3. Mark eligibility cache entries as used, or delete them, after a successful create, and store the enrolment-status short-circuit result too.
4. Make the local `version` rule match the contract, and validate sort code, account number and roll number against the contract patterns.
5. Consider tying the client code to the authenticated application (for example the API Platform application id), or at least stop using it unbounded in metric names.
6. Widen `maskNino` to match NINOs without a suffix, and avoid logging `Credentials.providerId`.
7. Build the get-account URL with the `url` interpolator's path-segment encoding (`url"$htsBaseUrl/$nino/account"`), so the NINO is encoded even though it comes from auth.
