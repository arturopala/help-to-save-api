# Help to Save API – Data Validation

Validation is **mixed: structural plus selective semantic rules**. Play JSON readers check that fields are present and have the right types. A hand-written `cats.Validated` layer then adds rules for header formats, names, phone numbers, email and enumerations, and collects every error into one message. Lengths, address and postcode formats, bank-account formats and the create-account NINO format are left to the back end and the account provider. The published contract, `resources/public/api/conf/2.0/application.yaml`, declares many patterns and length limits that the code does not enforce.

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Routing | Play routes, `String` params | `conf/app.routes` | 404 for unknown paths |
| 2 | Authentication | auth `authorise` with provider predicate | `Auth.authorised` / `handleFailure` | 401 / 403 / 500 (empty body) |
| 3 | Access type | `AccessType.fromCredentials` | `HelpToSaveController` | 403 `UNSUPPORTED_CREDENTIALS_PROVIDER` |
| 4 | Confidence level | `>= L200` (create account, Government Gateway only) | `HelpToSaveController.createAccount` | 401 text `Insufficient confidence level` |
| 5 | Identity binding | auth NINO vs path or body NINO | `HelpToSaveController.checkEligibility`, `CreateAccountBehaviour`, `storeEmailThenCreateAccount` | 403 (empty body, or `ACCESS_ERROR`) |
| 6 | Body parsing | Play `AnyContent` parser; `asJson` | `HelpToSaveApiServiceImpl` | Malformed JSON gives a 400 `FrameworkError`; a non-JSON body gives 400 `NO_JSON` |
| 7 | Fill-in (Government Gateway) | presence check of mandatory fields, then merge from auth | `CreateAccountField.missingMandatoryFields`, `fillInMissingDetailsGG` | 400 (channel missing or wrong type), 403 (NINO), 500 `MISSING_DATA` |
| 8 | JSON reader | `Reads[CreateAccountRequest]` | `CreateAccountRequest.reads`, `CreateAccountBody.reads(clientCode)` | 400 `VALIDATION_ERROR` "Could not parse JSON in request: …" |
| 9 | Headers | `APIHttpHeaderValidator` | service | 400 `VALIDATION_ERROR` |
| 10 | Field rules | `CreateAccountRequestValidator`, `EmailValidation`, `EligibilityRequestValidator` | service | 400 `VALIDATION_ERROR` (all errors joined) |
| 11 | Eligibility handshake | cached result by correlation id | `validateCorrelationId` | 400 / 500 |
| 12 | External bank check | back-end `validate-bank-details` | `validateBankDetails` | 400 `INVALID_BANK_DETAILS` / 500 |
| 13 | Upstream response parsing | `HttpResponseOps.parseJson` | service | 500 `SERVER_ERROR` |

Every `ApiError` body has the shape `{"code": "...", "message": "..."}`. `ApiValidationError(description)` produces code `VALIDATION_ERROR` with the message `Invalid request: <description>`.

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `nino` (`GET /eligibility/:nino`) | Any string at the routing stage. Later checked against `^((?!(BG\|GB\|KN\|NK\|NT\|TN\|ZZ)\|(D\|F\|I\|Q\|U\|V)[A-Z]\|[A-Z](D\|F\|I\|O\|Q\|U\|V))[A-Z]{2})[0-9]{6}[A-D]?$` (uppercase only, suffix optional) | `EligibilityRequestValidator.validateNino`, called from `validateCheckEligibilityRequest` **after** the enrolment-status call |
| `version`, `file` (`/api/conf/...`) | None (Play `Assets.at`) | `DocumentationController.yaml` |

For Government Gateway callers the path NINO must equal the auth NINO exactly (case-sensitive `===`), otherwise 403. For privileged callers, the regex is the only check.

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| `GET /eligibility`, `GET /eligibility/:nino`, `GET /account` | `Accept`, `Gov-Client-User-ID`, `Gov-Client-Timezone`, `Gov-Vendor-Version`, `Gov-Vendor-Instance-ID` | `Accept` must equal `application/vnd.hmrc.2.0+json`. The four Gov headers are checked for presence only. Errors are accumulated and returned as 400. |
| `POST /account` | Same, plus `Content-Type` | Content type must be `application/json`. The header checks run only **after** the body has parsed into `CreateAccountRequest`, so body errors are reported first. |
| all business endpoints | `Authorization` | Validated by auth |

⚠️ **Observation:** on the eligibility endpoints the header checks are skipped entirely when enrolment-status reports the person as enrolled. The response is 200 `{"accountExists":true}`.

## Request bodies

### Create account (`POST /account`)

**Reader**: `CreateAccountRequest.reads`. It reads `header` as `CreateAccountHeader` (Json.format) and then `body` with `CreateAccountBody.reads(header.clientCode)`. Errors are formatted by `JsErrorOps.prettyPrint` as `<path>: [<messages>]; ...`, and the response is 400 `Invalid request: Could not parse JSON in request: …`.

**What is checked**

- **Header**: all four fields are required.
  - `createdTimestamp` must parse with `yyyy-MM-dd HH:mm:ss z`.
  - `requestCorrelationId` must be a UUID.
  - `version` must match `^(\d\.)+\d+$` and be at most 10 characters long.
  - `clientCode` must match `^[A-Z0-9][A-Z0-9_-]+[A-Z0-9]$` and be at most 20 characters long.
- **Body**: the following must be present as strings: `nino`, `forename`, `surname`, `contactDetails` (`address1`, `address2`, `postcode`, `communicationPreference`) and `registrationChannel`. `dateOfBirth` must be present and match `yyyyMMdd`. `bankDetails` is optional, but if given it needs `accountNumber`, `sortCode` and `accountName`.
- **Forename**:
  - no leading special character
  - no two consecutive special characters
  - only `- & . , '` allowed as specials (spaces are ignored)
  - no digits
  - no apostrophe
- **Surname**: the same rules as forename, except that apostrophes are allowed. The last character may not be a special character.
- `communicationPreference` must be `00` or `02`, and `registrationChannel` must be `online` or `callCentre`.
- **Email** is required only when `communicationPreference = 02`. It is trimmed, then checked:
  - not blank, at most 254 characters in total
  - contains `@`
  - local part between 1 and 64 characters
  - domain part between 1 and 252 characters, containing a `.` with text before the first dot and after the last one

  The limits come from `email-validation.*`.
- **Phone number** (optional): at least one digit, no letters, and only `( ) - . + space` allowed as specials.
- **Conditional checks in the service**: when the preference is `02` and no email is present, the response is 400 "no email found…". This can't actually be reached, because the validator rejects the request first.

**What is not checked**

- The format of the body `nino`. For Government Gateway callers it is bound to the auth NINO instead; for privileged callers, to the NINO stored with the correlation id.
- Lengths of names, address lines, postcode and phone number. The contract has limits for all of them (26 / 300 / 35 / 10 / 15).
- Whether `dateOfBirth` is a plausible date (in the future, or an impossible age).
- The format of `countryCode`, `postcode`, `accountNumber`, `accountName` and `rollNumber`, and the format of `sortCode` (it is checked only by the external bank service).
- Whether `createdTimestamp` is recent. It is only parsed.
- Unknown fields, which are ignored. A `body.systemId` from the caller is ignored and replaced.

**Normalisation and derivation**

| Field | Rule | Code |
|---|---|---|
| `bankDetails.sortCode` | Strips the separators space, `-`, `–`, `−`, `—` and `_` | `CreateAccountBody.BankDetails.reads` |
| `body.systemId` | Set to `"MDTP-API-" + header.clientCode` | `CreateAccountBody.reads` |
| `email` | Trimmed only for validation; the untrimmed value is forwarded and stored | `EmailValidation.validate` |
| Missing forename / surname / DOB / NINO / address (Government Gateway) | Filled in from auth: `itmpName`, `itmpDateOfBirth` (falling back to `dateOfBirth`), `itmpAddress` (lines 1–2 and postcode are required, lines 3–5 and country are optional), auth NINO | `CreateAccountBehaviour.collate` |
| `communicationPreference` missing (Government Gateway, fill-in path only) | `online` gives `02`, `callCentre` gives `00` | `collate` |
| `email` missing (Government Gateway, fill-in path, `online`) | Taken from the auth `email` | `collate` |
| Eligibility reason | UC+WTC gives 8, WTC gives 7, otherwise 6 | `HelpToSaveApiServiceImpl.createAccount` |
| Email to back end | Base64-encoded | `EmailBehaviour.storeEmail` |

⚠️ **Observation:** defaults for `communicationPreference` and `email` are applied only when at least one *mandatory* field is missing. A Government Gateway request that supplies every mandatory field but leaves out `communicationPreference` fails with 400 instead of being defaulted.

⚠️ **Observation:** fill-in failures, where auth has no name or address, return **500** `MISSING_DATA`, not 400. The published contract documents this behaviour.

⚠️ **Observation:** a Government Gateway request that gives the body NINO with any non-string JSON type returns 400 "nino is not of expected type String", but only on the fill-in path.

### Eligibility and account retrieval

These endpoints take no body. The only input is the headers above and the path NINO.

## External validation services

| Service | Request | Decision | Failure behaviour |
|---|---|---|---|
| Bank details (back end `POST /help-to-save/validate-bank-details`) | `{nino, sortCode (normalised), accountNumber}` | `isValid: false` gives 400 `INVALID_BANK_DETAILS` | **Fails closed**: a non-200 response or an unparseable body gives 500. A connection exception isn't recovered and gives a `JsonErrorHandler` 500. The call is skipped when there are no `bankDetails`. |
| Eligibility handshake (Mongo `api-eligibility`) | `requestCorrelationId` | Must exist, its NINO must equal the body NINO, and it must be eligible. `accountExists` gives 409. | Missing entry gives 400 "requestCorrelationId(...) is not valid". NINO mismatch gives 400 "nino was not compatible with correlation Id". Not eligible gives 400. Mongo or parse error gives 500. |
| Enrolment status (back end) | NINO | `enrolled: true` gives "account exists" | **Fails open**: any error gives `None`, and the eligibility check goes ahead. |

The bank check and the eligibility handshake run in parallel. When the handshake says `AccountAlreadyExists` but the bank check returns `isValid:false`, the response is 400, not 409.

## Outbound payload validation

There is no outbound schema validation. The repository has no JSON Schema files except the published OpenAPI yaml, and the code never loads that. The create-account payload is the validated `CreateAccountBody` plus `eligibilityReason` and `source`. Formats are left to the back end and NS&I. Upstream 400s are passed back as 400 with the upstream `errorDetail` as the message.

PII masking: `util.maskNino` replaces `[A-Za-z]{2}[0-9]{6}[A-Za-z]` with `<NINO>` in logged upstream bodies. `LogMessageTransformer` adds the NINO to log lines only when `nino-logging.enabled = true`.

⚠️ **Observation:** the `maskNino` regex needs a suffix letter, so a NINO without a suffix in an upstream body would be logged unmasked.

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| eligibility-check 200 | `EligibilityCheckResponse` reads (`result`, `resultCode`, `reason`, `reasonCode` required), then the code mapping | 500 `SERVER_ERROR` and a PagerDuty log line |
| enrolment-status 200 | `{enrolled, itmpHtSFlag}`, both booleans and required | `None` (the flow continues) |
| create-account 400 | `CreateAccountErrorResponse {errorMessage, errorDetail}` | Falls back to 400 "request contained invalid or missing details" |
| account 200 | `HtsAccount` Json.format (all core fields required, dates ISO) | 500 |
| validate-bank-details 200 | `(json \ "isValid").as[Boolean]` inside a `Try` | 500 |
| Mongo cache entry | `data.eligibility` as `EligibilityResponseWithNINO` | 500 |

## Inbound callbacks and cache/opaque stores

There are no callbacks. The only store is the eligibility cache, which this service writes and which never takes caller-supplied JSON. Callers can only point at an entry by its correlation id, which must be a well-formed UUID.

## Error mapping summary

| Source | Exception / value | HTTP to caller |
|---|---|---|
| auth | `NoActiveSession` | 401 (empty body) |
| auth | `InternalError` | 500 (empty body) |
| auth | other `AuthorisationException` | 403 (empty body) |
| controller | unknown providerType | 403 `UNSUPPORTED_CREDENTIALS_PROVIDER` |
| controller | confidence level < 200 | 401 `Insufficient confidence level` (text) |
| controller / service | NINO missing or mismatched | 403 (empty body, or `ACCESS_ERROR`) |
| service | `ApiValidationError` | 400 |
| service | `ApiAccessError` | 403 |
| service | `ApiBackendError` | 500 |
| service | `CreateAccountSuccess(false)` / `(true)` | 201 / 409 |
| service | account `None` | 404 |
| Play body parser | malformed JSON | 400 (`JsonErrorHandler` body) |
| Play body parser | oversize body | 413 |
| `ApiError.writes` | `Json.parse` of an interpolated string fails | 500 (`JsonErrorHandler`) |
| validate-bank-details | connection exception (not recovered) | 500 (`JsonErrorHandler`) |

⚠️ **Observation:** `ApiError.writes` builds JSON by string interpolation (`Json.parse(s"""{ "code" : "${o.code}", ... }""")`). Several error messages echo caller input: `unknown client code $c`, `version has incorrect format: $v`, `Unknown registration channel: $r`, `Unknown communication preference: $c`, and the upstream `errorDetail`. If any of these contains `"`, `\` or a control character, `Json.parse` throws inside the result `Future`. `Auth.handleFailure` does not match that exception, so the caller gets a 500 where it should get a 400. A quote in the body can also inject extra JSON members into the error object.

## Recommendations

1. Replace the string-built `ApiError.writes` with `Json.obj("code" -> …, "message" -> …)`, and stop echoing raw caller input in messages.
2. Validate the NINO format, header, path and body, **before** making any upstream call. Run the header checks before the enrolment-status short-circuit.
3. Enforce the contract's length limits (names, address lines, postcode, phone) and formats (sort code `^[0-9]{6}$`, account number, roll number, country code) locally, so callers get a 400 and not a provider rejection.
4. Return 400, not 500, when Government Gateway fill-in can't find the required data in auth, or document this as intentional.
5. Apply the `communicationPreference` and email defaults independently of whether other mandatory fields are missing.
6. Add a `recover` to `validateBankDetails`, and send `X-Correlation-ID` on that call.
7. Store the enrolment-status "account exists" result in the eligibility cache, or map an unknown correlation id for an enrolled user to 409.
8. Widen `maskNino` to cover NINOs without a suffix.
