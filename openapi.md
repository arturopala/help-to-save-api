# Help to Save API – HTTP API

The machine-readable specification is in [`openapi.json`](openapi.json), an OpenAPI 3.0.3 document with **8 operations on 7 paths**. It was generated from an endpoint table built by reading:

- `conf/prod.routes`, `conf/app.routes` and `conf/definition.routes`, plus bootstrap's `health.Routes` for `/ping/ping`
- `HelpToSaveController`, `DocumentationController` and `Auth`
- `HelpToSaveApiServiceImpl`, the validators and the request/response models

Paths are given as the microservice serves them. On the API Platform the business endpoints sit under `/individuals/help-to-save` (API version 2.0, access type from `api.access.version-2.0.type`, which is `INTERNAL` by default). The separately maintained public spec, `resources/public/api/conf/2.0/application.yaml`, describes the platform view. Where its patterns differ from the code, see [validation.md](validation.md) and [identifiers.md](identifiers.md).

## Conventions that apply to every endpoint

| Concern | Behaviour |
|---|---|
| Authentication | Business endpoints call auth `authorise` with `AuthProviders(GovernmentGateway, PrivilegedApplication)` and the inbound bearer token. API Platform, asset and ping routes are unauthenticated. |
| Request body parsing | Play's default `AnyContent` parser, then `request.body.asJson`. A malformed JSON body with a JSON content type gets 400 from `JsonErrorHandler` (`{statusCode, message}`). A non-JSON body gets 400 `NO_JSON`. |
| Parameter binding | The only path parameter is the `nino` string; routing never rejects it. Format checks (NINO regex, headers) run in the service and return 400 `VALIDATION_ERROR`. |
| Required headers | `Accept: application/vnd.hmrc.2.0+json`, `Gov-Client-User-ID`, `Gov-Client-Timezone`, `Gov-Vendor-Version` and `Gov-Vendor-Instance-ID` (presence only). `Content-Type: application/json` is also required on `POST /account`. |
| Responses | Not passed through: upstream responses are mapped to the service's own shapes. Errors are `{"code","message"}` (`ApiError`), except auth failures and 404, which have empty bodies. |
| Error-status mapping | `ApiValidationError` gives 400, `ApiAccessError` 403 and `ApiBackendError` 500. Auth `NoActiveSession` gives 401, other auth exceptions 403, and auth `InternalError` 500. |
| Correlation | Eligibility responses carry `X-Correlation-ID`. Create account must send it back as `header.requestCorrelationId`, within 1 hour. |
| Size limits | Play defaults; there are no overrides in `application.conf`. An oversize body gets 413. |
| Auditing | Only bootstrap's standard request auditing; the service sends no explicit audit events. |

## Access-controlled endpoints

| Endpoint | Check | Failure (status and body) |
|---|---|---|
| all business endpoints | Valid session from GovernmentGateway or PrivilegedApplication | 401 empty (no session); 403 empty (other auth failures); 500 empty (auth internal error) |
| `GET /eligibility`, `GET /eligibility/{nino}`, `POST /account` | `credentials.providerType` is `GovernmentGateway` or `PrivilegedApplication` | 403 `{"code":"UNSUPPORTED_CREDENTIALS_PROVIDER",…}` |
| `GET /eligibility` | Government Gateway only, and auth must have a NINO | 403 empty |
| `GET /eligibility/{nino}` | Government Gateway: the path NINO must equal the auth NINO. Privileged: none. | 403 empty |
| `POST /account` (Government Gateway) | Confidence level ≥ 200 | 401 `Insufficient confidence level` (text/plain) |
| `POST /account` (Government Gateway) | The body NINO, if present, equals the auth NINO; auth NINO required on the fill-in path | 403 `{"code":"ACCESS_ERROR","message":"access not allowed"}` |
| `POST /account` (both) | The correlation id's stored NINO equals the body NINO, and the result is eligible | 400 `VALIDATION_ERROR` |
| `GET /account` | Auth must have a NINO (so privileged applications are effectively rejected) | 403 empty |

## Back-end services

All business calls go to the **help-to-save** back end (`microservice.services.help-to-save`, base path `/help-to-save`):

| Endpoint | Upstream calls (in order) |
|---|---|
| `GET /eligibility`, `GET /eligibility/{nino}` | `GET /enrolment-status?nino=`, then (if not enrolled) `GET /eligibility-check?nino=`, then a Mongo write |
| `POST /account` | Mongo read of the correlation id, in parallel with `POST /validate-bank-details` (if `bankDetails` given), then `GET /store-email?email=&nino=` (if preference `02`), then `POST /create-account` |
| `GET /account` | `GET /{nino}/account?systemId=&correlationId=` |

See [dependencies.md](dependencies.md) for headers and status mapping.

## Endpoints with non-standard behaviour

- **`GET /eligibility`, `GET /eligibility/{nino}`**:
  - Add an `X-Correlation-ID` response header to every response produced after authentication, including 4xx and 5xx responses.
  - If enrolment-status says the person is enrolled, the response is 200 `{"accountExists":true}` straight away. Header and NINO validation are skipped and nothing is cached.
- **`GET /eligibility`** returns 403 for privileged applications, because there is no NINO to derive.
- **`POST /account`**:
  - A success has no body: 201 for created, 409 if the account already existed.
  - For Government Gateway callers, missing name, date of birth, NINO and address are merged in from auth. Missing data comes back as **500** `MISSING_DATA`, not 400.
  - Validation order: confidence level, then body JSON, then fill-in, then the JSON reader, then headers plus field rules (accumulated), then the eligibility handshake and the bank check.
  - An upstream 400 is passed back as 400 with the upstream `errorDetail` as the message.
- **`GET /account`** returns 404 with an empty body when there is no account. `blockedFromPayment` is always `false`, and `headroom` is the upstream `canPayInThisMonth`.
- **Error bodies**: `ApiError` JSON is built by string interpolation. Messages that echo caller input containing `"` or `\` turn a would-be 400 into a 500 from `JsonErrorHandler`. This can be reached through `header.clientCode`, `header.version`, `body.registrationChannel`, `contactDetails.communicationPreference`, or the upstream `errorDetail`.
- **Unreachable status**: `POST /account` returns 400 "no email found in the request body but communicationPreference is 02" only in theory, because the validator rejects a missing email first.
- **`GET /api/conf/{version}/{file}`** is CORS-enabled and serves classpath files under `/public/api/conf/{version}`. `/assets/{file}` serves the same `/public` tree.

## Endpoint catalogue

### Eligibility

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| GET | `/eligibility` | Check eligibility of the signed-in user (NINO from auth) | Government Gateway only (privileged → 403); NINO from auth | `help-to-save` → `GET /help-to-save/eligibility-check?nino={nino}` |
| GET | `/eligibility/{nino}` | Check eligibility for a given NINO | Government Gateway (path NINO must equal auth NINO) or Privileged Application (any NINO) | `help-to-save` → `GET /help-to-save/eligibility-check?nino={nino}` |

### Accounts

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| POST | `/account` | Create a Help to Save account | Government Gateway with confidence level ≥ 200 (NINO bound to auth) or Privileged Application | `help-to-save` → `POST /help-to-save/create-account` |
| GET | `/account` | Get the signed-in user's account details | Any supported provider; NINO must be present in auth (else 403) | `help-to-save` → `GET /help-to-save/{nino}/account?systemId=MDTP-API-client&correlationId={uuid}` |

### API Platform

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| GET | `/api/definition` | API Platform definition (context, version 2.0, access type, enabled flag) | — | — (served locally) |
| GET | `/api/conf/{version}/{file}` | Serve API specification files (e.g. 2.0/application.yaml) | — | — (served locally) |

### Platform

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| GET | `/assets/{file}` | Static assets from /public (Play Assets controller) | — | — (served locally) |
| GET | `/ping/ping` | Health check (bootstrap health.Routes) | — | — (served locally) |
