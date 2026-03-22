# Copilot Instructions

## Running the Mock Server

```bash
docker compose up       # start on http://localhost:8080
docker compose down     # stop
```

## Repo Layout

- `recruitment-openapi.yaml` — OpenAPI 3.0.3 spec (source of truth for the API contract)
- `wiremock/mappings/` — WireMock stub routing rules (one JSON file per endpoint/ID)
- `wiremock/__files/` — Response body JSON files (edit these to change returned data)
- `docker-compose.yml` — Runs `wiremock/wiremock:latest` mounting the above directories

## What This Repo Is

A single OpenAPI 3.0.3 specification (`recruitment-openapi.yaml`) for `AuBusinessLogicRecruitment` — a business logic service at Aarhus University (AU) that wraps an external recruitment supplier (SRL). It exposes recruitment data so the external vendor can sync job postings and recruitment status with AU's internal systems.

## API Structure

Three endpoint groups defined in the spec:

| Tag | Endpoints |
|-----|-----------|
| `JobPostings` | `GET /v1/job-postings` (list), `GET /v1/job-postings/{jobPostingId}` |
| `RecruitmentProcesses` | `GET /v1/recruitment-processes/{recruitmentId}` |
| `Ping` | `GET /v1/ping` (health check, no auth) |

- `JobPosting` and `RecruitmentProcess` are linked: a `JobPostingListItem` contains a `recruitmentId`, and a `RecruitmentDetails` embeds a full `JobPostingDetails`.
- The list endpoint (`/v1/job-postings`) returns lightweight `JobPostingListItem`; the detail endpoints return the full `JobRecruitmentResponse` wrapping `RecruitmentDetails`.

## Key Conventions

**Bilingual fields** — Nearly all human-readable text fields come in Danish/English pairs using the `Da`/`En` suffix. Both are always `nullable: true`. Examples: `titleDa`/`titleEn`, `departmentNameDa`/`departmentNameEn`, `positionTypeDa`/`positionTypeEn`.

**Description split** — `LocalizedDescription` breaks job text into three typed sub-fields: `technicalText`, `hrText`, `auText`. This applies to both `descriptionDa` and `descriptionEn` on `JobPostingDetails`.

**Pagination** — List responses use an envelope with `{ jobPostings: [...], paging: { limit, offset, returned, hasMore } }`. The list endpoint accepts `limit` (1–5000, default 1000), `offset` (default 0), and `updatedSince` (ISO 8601 date-time).

**Error format** — All error responses use `AuProblemDetails` (supports both `application/problem+json` and `application/json`). Contains standard RFC 7807 fields plus AU-specific: `traceId`, `externalTraceId`, `errorCode`, `errorLogged`, `validationErrors[]`.

**Authentication** — Bearer token with OAuth2 scope `Recruitment.Read` required on all endpoints except `GET /v1/ping`.

**IDs** — `jobPostingId` is a UUID string; `recruitmentId` is an `int64` integer. `departmentCode` is an integer (e.g. `2847`). Contact AUIDs are strings (e.g. `'539754'`).

**recruitmentStatus** — Integer field on `JobPostingListItem`: `1 = Active`, `2 = Closed`.

**Servers** — Three environments defined: `api.au.dk` (prod), `api-test.au.dk` (test), `api-dev.au.dk` (dev).
