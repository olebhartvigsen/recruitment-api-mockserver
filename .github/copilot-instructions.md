# Copilot Instructions

## Running the Mock Server

```bash
docker compose up       # start on http://localhost:8080
docker compose down     # stop
docker compose restart  # pick up mapping/file changes
```

| URL | Description |
|-----|-------------|
| http://localhost:8080 | WireMock API |
| http://localhost:8081 | Swagger UI |
| http://localhost:8082 | Redoc |

## What This Repo Is

A WireMock-based mock server for `AuBusinessLogicRecruitment` — a business logic service at Aarhus University (AU) that wraps the external recruitment supplier SRL. It exposes recruitment data so SRL can sync job postings and status with AU's internal systems.

## API Structure

| Tag | Endpoints |
|-----|-----------|
| `JobPostings` | `GET /v1/job-postings` (list), `GET /v1/job-postings/{jobPostingId}` |
| `RecruitmentProcesses` | `GET /v1/recruitment-processes/{recruitmentId}` |
| `Ping` | `GET /v1/ping` (health check, no auth) |

- `jobPostingId` is a UUID string; `recruitmentId` is an `int64` integer.
- The list endpoint returns lightweight `JobPostingListItem`; the detail endpoints return full `JobRecruitmentResponse` wrapping `RecruitmentDetails`.

## File Layout

```
recruitment-openapi.yaml               # Source of truth for API contract (prod/test/dev servers)
wiremock/
├── __files/
│   ├── recruitment-openapi.yaml       # Copy served to Swagger UI — has localhost:8080 server only
│   ├── job-postings-list.json         # Response for GET /v1/job-postings
│   ├── job-posting-<uuid>.json        # Response for GET /v1/job-postings/{uuid}
│   └── recruitment-process-<id>.json  # Response for GET /v1/recruitment-processes/{id}
└── mappings/
    ├── list-job-postings.json
    ├── get-job-posting-<8-char-hex>.json      # 8-char prefix of the UUID
    ├── get-job-posting-not-found.json         # Fallback 404, priority: 10
    ├── get-recruitment-process-<id>.json
    └── get-recruitment-process-not-found.json # Fallback 404, priority: 10
dokumenter/
├── au-enheder.json        # 1314 AU department/unit reference entries
└── au-stillingstyper.json # 98 AU position type reference entries
```

**Two separate YAML files**: `wiremock/__files/recruitment-openapi.yaml` is the one served to Swagger UI and has `servers: [{url: http://localhost:8080}]` only. The root `recruitment-openapi.yaml` keeps the real prod/test/dev server URLs. Always keep both in sync when changing the API spec.

## Data Sync Rules (Critical)

Every job posting exists in **three places** that must always be kept in sync:

1. `wiremock/__files/job-posting-<uuid>.json` — full detail response
2. `wiremock/__files/recruitment-process-<id>.json` — mirrors the job-posting file; same nested structure under `recruitment`
3. `wiremock/__files/job-postings-list.json` — contains a lightweight summary item in the `jobPostings` array

When adding or updating a job posting, update all three. Also update `paging.returned` in `job-postings-list.json` to match the array length.

### Adding a new job posting

1. Create `wiremock/__files/job-posting-<uuid>.json` (copy an existing one)
2. Create `wiremock/__files/recruitment-process-<recruitmentId>.json` (same data)
3. Add summary item to `wiremock/__files/job-postings-list.json`
4. Create `wiremock/mappings/get-job-posting-<first-8-chars-of-uuid>.json`
5. Create `wiremock/mappings/get-recruitment-process-<recruitmentId>.json`
6. Run `docker compose restart`

### Mapping file format

```json
{
  "priority": 1,
  "request": { "method": "GET", "url": "/v1/job-postings/<uuid>" },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "bodyFileName": "job-posting-<uuid>.json"
  }
}
```

Exact-match stubs use `"url"` with `priority: 1`. The fallback 404s use `"urlPathPattern"` regex with `priority: 10`.

## Key Conventions

**Bilingual fields** — Human-readable text fields come in Da/En pairs (`titleDa`/`titleEn`, `departmentNameDa`/`departmentNameEn`, etc.). Both are always `nullable: true`. Set the correct language and leave the other `null`.

**Language detection** — The `languages` array (e.g. `["da"]` or `["en"]`) and which description field is populated must match. Use the **title field as the primary language signal**: if `titleEn` is set and `titleDa` is null → English posting → use `descriptionEn.technicalText`, set `descriptionDa` to null. Danish boilerplate footers in English job text should not affect this classification.

**Description fields** — Only `technicalText` is populated. `hrText` and `auText` are always `null`. This applies to both `descriptionDa` and `descriptionEn`.

**shortText** — `shortTextDa` always contains the "Ledig stilling ved X" Danish boilerplate. `shortTextEn` is always `null` regardless of the posting language.

**Pagination** — List response envelope: `{ jobPostings: [...], paging: { limit, offset, returned, hasMore } }`.

**Error format** — `AuProblemDetails` (RFC 7807 + AU extensions: `traceId`, `externalTraceId`, `errorCode`, `errorLogged`, `validationErrors[]`). Content-type: `application/problem+json`.

**Authentication** — Bearer token with OAuth2 scope `Recruitment.Read`. Not enforced in the mock but must be documented in the spec.

**recruitmentStatus** — `1 = Active`, `2 = Closed`.

**Postal codes** — Stored as integers. Reference: https://github.com/tjconcept/postal-codes-denmark. Aarhus C = 8000, Aarhus N = 8200.

**Department codes** — Integer values. Reference: `dokumenter/au-enheder.json` (`enhedKode` field).

**Position type codes** — String values. Reference: `dokumenter/au-stillingstyper.json` (`stillingstypeKode` field).

**Contact AUIDs** — Strings (e.g. `"539754"`). Both `technicalContactAuid` and `hrContactAuid` use the same AUID.
