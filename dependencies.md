# Help to Save API – Dependencies

All outbound HTTP calls use the bootstrap `HttpClientV2`, enabled through `uk.gov.hmrc.play.bootstrap.HttpClientV2Module`. URLs are built with the `url"..."` interpolator, which encodes interpolated query values. The `HeaderCarrier` is derived from the inbound request by `BackendController`, and http-verbs forwards its standard headers (`Authorization`, `X-Request-ID`, `X-Session-ID`, …) to internal hosts. The back-end base URL is assembled by hand in `HelpToSaveConnectorImpl` from `microservice.services.help-to-save.{protocol,host,port}`, with `/help-to-save` appended. The auth service URL is resolved by bootstrap's `AuthModule` from `microservice.services.auth`. The correlation header name comes from `microservice.correlationIdHeaderName` (`X-Correlation-ID`).

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | auth | HTTP (MDTP auth service) | `microservice.services.auth` | `Auth` trait (`AuthorisedFunctions`), `HelpToSaveController` |
| 2 | help-to-save (back end) | HTTP | `microservice.services.help-to-save` | `HelpToSaveConnector` / `HelpToSaveConnectorImpl`, called from `HelpToSaveApiServiceImpl`, `EmailBehaviour` |
| 3 | MongoDB | Datastore | `mongodb.uri`, `mongo-cache.expireAfter` | `EligibilityStore` / `MongoEligibilityStore` |
| 4 | API Platform | Inbound caller | `api.access.version-2.0.*` | `DocumentationController` |

---

## 1. auth

- **Base URL**: `microservice.services.auth`, which defaults locally to `localhost:8500`.
- **Scheme**: the inbound `Authorization` bearer token is passed on by `AuthConnector` (bootstrap `AuthModule`).

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Authorise + retrieve | `POST /auth/authorise` (bootstrap) | `Auth.authorised(retrievals)` | Predicate `AuthProviders(GovernmentGateway, PrivilegedApplication)`. Retrievals vary by endpoint (see the Auth section). |

**Failure mapping** (`Auth.handleFailure`):

| Exception | Result |
|---|---|
| `NoActiveSession` | 401, empty body |
| `InternalError` | 500, empty body |
| any other `AuthorisationException` | 403, empty body |

Any other exception propagates to `JsonErrorHandler`, which returns a 500.

## 2. help-to-save (back end)

- **Base URL**: `{protocol}://{host}:{port}/help-to-save`, which defaults locally to `http://localhost:7001/help-to-save`. The URL is computed once, when the connector is constructed.
- **Headers**: `X-Correlation-ID: <uuid>` on every call except validate-bank-details, plus the standard `HeaderCarrier` headers. There is no other service-to-service auth: the end user's bearer token is propagated.
- **Status handling**: `execute[HttpResponse]` never throws on non-2xx statuses. Status mapping is done in the service.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Enrolment status | `GET /help-to-save/enrolment-status?nino={nino}` | `HelpToSaveConnectorImpl.getUserEnrolmentStatus` | Fails open: a non-200 response, a parse error or an exception gives `None`, and the eligibility check goes ahead. `{enrolled:true}` short-circuits to `AccountAlreadyExists`. |
| Eligibility check | `GET /help-to-save/eligibility-check?nino={nino}` | `HelpToSaveConnectorImpl.checkEligibility` | 200 is parsed as `EligibilityCheckResponse`, and `(resultCode, reasonCode)` is mapped to an eligibility result (table below). Any other status or an exception gives 500 and a PagerDuty log line. |
| Create account | `POST /help-to-save/create-account` | `HelpToSaveConnectorImpl.createAccount` | Body `{payload: CreateAccountBody, eligibilityReason: 6\|7\|8, source: clientCode}`. 201 gives Created; 409 gives Conflict; 400 gives a 400 whose message is the upstream `errorDetail` (falling back to a generic message); anything else gives 500 and a PagerDuty log line. |
| Get account | `GET /help-to-save/{nino}/account?systemId=MDTP-API-client&correlationId={uuid}` | `HelpToSaveConnectorImpl.getAccount` | `systemId` comes from config key `system-id`. 200 is parsed as `HtsAccount` and mapped to `Account`; 404 gives 404; anything else gives 500. The NINO is pre-interpolated into the path string, so it is not encoded (it comes from auth). |
| Store email | `GET /help-to-save/store-email?email={base64}&nino={nino}` | `HelpToSaveConnectorImpl.storeEmail` | Called only when communicationPreference = `02`. A non-200 response or an exception gives 500 and a PagerDuty log line. |
| Validate bank details | `POST /help-to-save/validate-bank-details` | `HelpToSaveConnectorImpl.validateBankDetails` | Body `{nino, sortCode, accountNumber}`. 200 is read as `isValid`: `false` gives 400 `INVALID_BANK_DETAILS`; a non-200 response or an unparseable body gives 500. |

**Eligibility mapping** (`HelpToSaveApiServiceImpl.EligibilityCheckResponseOps.toApiEligibility`):

| resultCode | reasonCode | isEligible | hasWTC | hasUC |
|---|---|---|---|---|
| 1 | 6 | true | false | true |
| 1 | 7 | true | true | false |
| 1 | 8 | true | true | true |
| 2 | 3 | false | true | false |
| 2 | 4 | false | true | true |
| 2 | 5 | false | false | true |
| 2 | 9 | false | false | false |
| 3 | any | `accountExists: true` | | |
| other | | 500 (parse error) | | |

**Create-account eligibility reason** sent upstream: UC and WTC gives `8`, WTC only gives `7`, otherwise `6`.

**Usage notes**

- **Eligibility flow**: enrolment status, then validation of headers and the NINO, then the eligibility check, then the result is stored in Mongo. A Mongo write failure gives 500.
- **Create-account flow**:
  1. Look up the cached eligibility by `requestCorrelationId`, in parallel with bank validation.
  2. If the preference is `02`, call store-email.
  3. Call create-account.
- **Audit events**: none are sent explicitly.
- **Metrics** (`Metrics`):
  - `api.eligibility.time` / `.errors.count`
  - `api.create-account.{user-restricted|privileged}.time` / `.errors.count`
  - `api.create-account.<clientCode>.count`
  - `api.validate-bank-details-timer` / `-error.count`

⚠️ **Observation:** `validate-bank-details` is called without the `X-Correlation-ID` header, and its `Future` has no `recover`. A connection failure there skips the `ApiError` mapping and comes back as a `JsonErrorHandler` 500.

⚠️ **Observation:** store-email is a state-changing `GET` that carries the email (base64-encoded, not encrypted) and the NINO in the query string. They can therefore appear in access logs.

⚠️ **Observation:** the timers in `createAccountUserRestricted`, `createAccountPrivileged` and `performCheckEligibility` are stopped synchronously, before the returned `Future` completes. They measure how long it takes to build the future, not the latency of the call.

⚠️ **Observation:** the enrolment-status call runs **before** the header and NINO-format validation. A request with bad headers or a malformed path NINO still triggers an upstream call. If upstream reports the person as enrolled, the request gets 200 `{"accountExists":true}` without being validated at all.

## 3. Datastore (MongoDB)

| Collection | Repository | Key | Expiry | Purpose |
|---|---|---|---|---|
| `api-eligibility` | `MongoEligibilityStore` (hmrc-mongo `MongoCacheRepository`, `SimpleCacheId`) | `_id` = eligibility correlation id (UUID string). Data at `data.eligibility` = `{eligibilityResponse, nino}`. | TTL on the last-updated timestamp, from `mongo-cache.expireAfter` (1 hour) | Links an eligibility result to the create-account request that follows it |

- **Encryption**: none. The NINO and the eligibility result are stored in plain text.
- **What gets cached**:
  - Results from the eligibility-check call, including `resultCode 3` (an account exists), are stored.
  - The enrolment-status short-circuit is **not** stored. A create-account request that uses that correlation id gets 400 "requestCorrelationId(...) is not valid" instead of 409.
- **Reuse**: an entry is not deleted after it is used, so the same correlation id can be replayed until it expires.

## 4. Auth (guards)

| Guard | Used by | Requires | Retrieves |
|---|---|---|---|
| `authorised(v2Credentials)` | `checkEligibilityDeriveNino`, `checkEligibility`, `createAccount` | GovernmentGateway or PrivilegedApplication provider | `credentials`, mapped to an `AccessType`. Any other providerType gives 403 `UNSUPPORTED_CREDENTIALS_PROVIDER`. |
| nested `authorised(v2Nino)` | eligibility (Government Gateway) | same | `nino`. If it is missing, the request gets 403. |
| nested `authorised(userInfoRetrievals and v2Nino)` | create account (Government Gateway) | same, plus `ConfidenceLevel >= L200`, checked in code (otherwise 401 `Insufficient confidence level`) | dateOfBirth, itmpName, itmpDateOfBirth, itmpAddress, email, confidenceLevel, nino |
| `authorised(v2Nino)` | `getAccount` | same | `nino`. If it is missing, the request gets 403 (in practice, privileged applications). |

Each nested `authorised` call makes another round trip to auth.

## 5. Inbound callers and callbacks

- **API Platform gateway**: proxies the third-party traffic for `individuals/help-to-save` to this service's `/eligibility`, `/eligibility/:nino` and `/account` routes.
- **API Platform publisher**: reads `GET /api/definition` and `GET /api/conf/2.0/application.yaml`. These routes are unauthenticated, and the yaml route has CORS enabled.
- **No callbacks or webhooks** are registered.

## 6. Libraries

| Library | Version | Used for |
|---|---|---|
| `bootstrap-backend-play-30` | 10.7.0 | `HttpClientV2`, `AuthConnector`, `BackendController`, `JsonErrorHandler`, metrics, standard audit filter |
| `hmrc-mongo-play-30` | 2.14.0 | `MongoCacheRepository`, `PlayMongoModule` |
| `cats-core` | 2.13.0 | `Validated`/`EitherT` validation and composition |
| Play (sbt-plugin) | 3.0.10 | Framework, `CORSActionBuilder`, `Assets` |
| Scala | 3.7.1 | |

## 7. Local development

- `sbt run` serves on port **7004** (`PlayKeys.playDefaultPort`).
- auth: `localhost:8500`. help-to-save: `http://localhost:7001`. MongoDB: `mongodb://localhost:27017/help-to-save`.
- Test requests can be built with `help-to-save-test-admin-frontend`, according to the README.
- To preview the API spec locally, start the API documentation services with `sm2 --start API_DEFINITION THIRD_PARTY_APPLICATION …` (see the README) and point the preview at `http://localhost:7004/api/conf/2.0/application.yaml`.
- `conf/testOnlyDoNotUseInAppConf.routes` adds no test-only routes.
