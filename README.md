# Recruitment API Mock Server

WireMock-based mock server for the `AuBusinessLogicRecruitment` API, running in Docker.

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker + Docker Compose)

## Start the mock server

```bash
docker compose up
```

The API is available at **http://localhost:8080**

Stop with `Ctrl+C`, or run in the background with `docker compose up -d` (stop with `docker compose down`).

## Available endpoints

| Method | URL | Description |
|--------|-----|-------------|
| `GET` | `/v1/ping` | Health check — always returns 200 |
| `GET` | `/v1/job-postings` | List job postings |
| `GET` | `/v1/job-postings/{jobPostingId}` | Get job posting by UUID |
| `GET` | `/v1/recruitment-processes/{recruitmentId}` | Get recruitment process by integer ID |

Authentication headers are accepted but **not enforced** in the mock.

| URL | Description |
|-----|-------------|
| http://localhost:8081 | Swagger UI — interactive API browser |
| http://localhost:8080/recruitment-openapi.yaml | Raw OpenAPI spec |

### Example requests

```bash
curl http://localhost:8080/v1/ping

curl http://localhost:8080/v1/job-postings

curl http://localhost:8080/v1/job-postings/4b1c2551-9de3-4d18-9ae1-b963298d0bb9

curl http://localhost:8080/v1/recruitment-processes/20237
```

Unknown IDs return a `404` with a `application/problem+json` body.

---

## Adding more data

### Add a job posting to the list

Edit `wiremock/__files/job-postings-list.json` — add a new object to the `jobPostings` array and update `paging.returned` to match the new count.

### Add a new job posting detail / recruitment process

1. Create a response file in `wiremock/__files/`:
   - `job-posting-<uuid>.json` — copy an existing one and change the values
   - `recruitment-process-<id>.json` — same approach

2. Create a mapping file in `wiremock/mappings/`:
   - Copy `wiremock/mappings/get-job-posting-4b1c2551.json` (or `get-recruitment-process-20237.json`)
   - Update the `url` to the new ID and `bodyFileName` to point to the new response file

3. Restart the mock server: `docker compose restart`

### File layout

```
wiremock/
├── mappings/          # One JSON file per endpoint/ID — controls routing
│   ├── ping.json
│   ├── list-job-postings.json
│   ├── get-job-posting-4b1c2551.json
│   ├── get-job-posting-not-found.json
│   ├── get-recruitment-process-20237.json
│   └── get-recruitment-process-not-found.json
└── __files/           # Response body JSON files — edit these to change response data
    ├── job-postings-list.json
    ├── job-posting-4b1c2551-9de3-4d18-9ae1-b963298d0bb9.json
    └── recruitment-process-20237.json
```
