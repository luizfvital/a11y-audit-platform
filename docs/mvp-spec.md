# Accessibility Audit Platform - MVP Specification

## Purpose

This document is the canonical MVP specification for the Accessibility Audit Platform backend.

It combines the previously separate scope, architecture, domain model, report run lifecycle, and audit execution flow documents into a single source of truth.

The MVP is intentionally backend-first and API-first. It exists to validate that the platform can perform automated accessibility audits, normalize the results, persist the audit domain, and expose a stable backend contract before any dedicated client is required.

## MVP Objective

The MVP backend must allow a developer to:

1. Configure an application to audit, including its screens
2. Create a report configuration
3. Trigger an accessibility scan
4. Execute axe-core using Playwright
5. Retrieve normalized accessibility findings through the API

The MVP proves that the audit engine, normalization pipeline, persistence model, and API design work end to end through a stable backend contract.

## In Scope

The MVP backend must support:

- project creation and listing
- application creation and update within a project
- screen management within the application context
- report creation with selected screens
- optional report authentication configuration
- report run triggering
- sequential scan execution
- report run lifecycle tracking
- normalized finding generation
- guideline reference data
- retrieval of run status and findings
- backend-owned persistence for the audit domain
- OpenAPI documentation, Swagger UI, Postman validation, and automated API tests

## Out of Scope

The MVP explicitly excludes:

- dashboards, charts, trend analysis, and advanced reporting
- issue assignment, remediation workflows, and finding lifecycle management
- PDF, CSV, or Excel export
- automatic crawling or screen discovery
- scheduled scans
- queue-based or parallel scan execution
- a required full client application
- advanced retry strategies or partial-success handling

Historical report runs may exist in the MVP, but advanced reporting over that history is out of scope.

## Core Architectural Decisions

The MVP follows these decisions:

- the platform is API-first, so the backend owns the domain contract consumed by any current or future client
- the backend is the system of record for the MVP audit domain
- scan execution, normalization, and result retrieval live in the backend
- screens remain domain entities, but the MVP public API manages them through the parent application
- findings are exposed as normalized backend records, not raw axe-core payloads
- the initial implementation may run scans sequentially inside the same Node application, while preserving a path to a separate worker later

This structure keeps the MVP simple without coupling the domain to a specific UI.

## High-Level Architecture

```text
Client Application or Integration (future)
        |
        | REST API
        v
Node Backend API
        |
        | create/read/update audit resources
        | trigger report runs
        v
Audit Runner
        |
        | browser automation
        v
Playwright + axe-core
        |
        | normalized results
        v
Persistence / Retrieval Layer
```

## System Components

### Client Applications and Integrations (Future)

Any future frontend or integration consumes the backend through REST APIs.

Responsibilities:

- create and edit projects, applications, screens, and reports through the API
- trigger report runs
- retrieve run status, summaries, findings, and guideline data
- provide future dashboards or workflow-specific UX

### Node Backend API

The backend API is the entry point for the platform.

Responsibilities:

- expose REST endpoints
- validate requests
- create, read, and update audit-domain resources
- manage screens within the application context
- create report runs
- orchestrate audit execution
- persist normalized data
- expose run status, findings, summaries, and guidelines
- publish OpenAPI and Swagger documentation

### Audit Runner

The audit runner performs the actual scan work.

Responsibilities:

- launch browser instances
- apply device and viewport configuration
- navigate to target URLs
- perform optional authentication flows
- wait for page readiness
- execute axe-core
- collect raw results
- hand results to normalization logic

At the beginning, the runner may live inside the same Node service as the API. Later, it can be extracted into a dedicated worker.

### Audit Engine

The audit engine combines:

- Playwright for browser automation, page navigation, authentication, interaction, and viewport simulation
- axe-core for accessibility analysis and machine-readable violations

### Persistence Layer

The backend owns persistence for:

- Projects
- Applications
- Screens
- Reports
- ReportRuns
- Guidelines
- Findings

PostgreSQL is the recommended MVP persistence technology.

## Domain Model

```text
Project
   |
   +-- Application
         |
         +-- Screen
         |
         +-- Report
               +-- selected Screens
               |
               +-- ReportRun
                     |
                     +-- Finding
                           +-- Screen reference
                           +-- Guideline reference
```

### Core Principles

- A Project represents a customer initiative.
- A Project contains one or more Applications.
- An Application contains one or more Screens and one or more Reports.
- A Report is a reusable audit configuration.
- A ReportRun is one execution instance of a Report.
- A Finding is one normalized accessibility issue detected during a report run.
- A Guideline represents the accessibility rule or success criterion referenced by findings.
- The public contract should remain stable regardless of whether the consumer is a web frontend, low-code frontend, or another integration.

## Entity Definitions

### Project

Description:
A top-level container representing a customer project.

Required MVP fields:

- `id`
- `name`
- `customerName`
- `createdAt`
- `updatedAt`

Relationships:

- one Project has many Applications

### Application

Description:
A target application inside a project that can be configured for accessibility audits.

Required MVP fields:

- `id`
- `projectId`
- `name`
- `device`
- `width`
- `height`
- `waitTimeMs`
- `accessibilityTarget`
- `createdAt`
- `updatedAt`

Relationships:

- one Application belongs to one Project
- one Application has many Screens
- one Application has many Reports

Notes:

- current product flows require targeted accessibility standard, version, and levels
- MVP device values may include `desktop`, `tablet`, and `phone`
- MVP accessibility target values may include `WCAG` with versions `2.0`, `2.1`, or `2.2` and one or more levels `A`, `AA`, `AAA`
- application create and update operations may carry a nested screen collection

### Screen

Description:
A single auditable page or route inside an application.

Required MVP fields:

- `id`
- `applicationId`
- `name`
- `url`
- `createdAt`
- `updatedAt`

Relationships:

- one Screen belongs to one Application
- one Screen can be included in many Reports
- one Screen can appear in many Findings through ReportRuns

Notes:

- a screen is defined minimally by a human-readable name and target URL
- screens are managed through the parent Application in the MVP public API rather than as a standalone top-level resource

### Report

Description:
A reusable audit definition for an application.

A Report is a configuration object, not an execution object.

Required MVP fields:

- `id`
- `applicationId`
- `name`
- `authenticationEnabled`
- `authentication` (optional object)
- `authentication.loginUrl` (optional)
- `authentication.username` (optional)
- `authentication.password` (optional, sensitive, write-only)
- `selectedScreenIds`
- `createdAt`
- `updatedAt`

Relationships:

- one Report belongs to one Application
- one Report selects one or more Screens
- one Report has many ReportRuns

Notes:

- selected screens are chosen from the report application's screen collection
- authentication support may be only partially implemented in the MVP, but the model should allow it
- authentication data is grouped under the `authentication` object rather than flattened on the report
- `authentication.password` may be sent in write operations, but it should not be returned in report responses

### ReportScreen

Description:
A join relationship representing which screens are included in a report.

Required MVP fields:

- `reportId`
- `screenId`

Relationships:

- many Reports can include many Screens

Notes:

- this many-to-many relationship should exist in the domain model
- it does not need to be exposed as a first-class API resource in the MVP
- the public API may manage screen selection through the `Report` payload

### ReportRun

Description:
A single execution instance of a report.

Required MVP fields:

- `id`
- `reportId`
- `status`
- `startedAt` (optional until run starts)
- `finishedAt` (optional until run ends)
- `errorMessage` (optional)
- `summary` (optional)
- `summary.totalFindings` (optional)
- `summary.screensScanned` (optional)
- `summary.screensPlanned` (optional)
- `summary.findingsByImpact` (optional)
- `createdAt`
- `updatedAt`

Relationships:

- one ReportRun belongs to one Report
- one ReportRun has many Findings

Notes:

- minimum expected statuses are `pending`, `running`, `completed`, and `failed`
- multiple runs may exist over time for the same report to support historical retrieval later

### Guideline

Description:
A structured accessibility rule or success criterion used to classify findings.

Required MVP fields:

- `id`
- `code`
- `name`
- `level`
- `standard`
- `createdAt`
- `updatedAt`

Relationships:

- one Guideline can be referenced by many Findings

Notes:

- guideline data may be seeded from a static dataset rather than manually managed
- expected examples include codes such as `1.1.1`, levels such as `A`, and standards such as `WCAG`

### Finding

Description:
A normalized accessibility issue produced by a report run.

Required MVP fields:

- `id`
- `reportRunId`
- `screenId`
- `guidelineId`
- `ruleCode`
- `message`
- `impact` (optional if available)
- `severity` (optional if modeled separately)
- `htmlSnippet` (optional)
- `selector`
- `createdAt`

Relationships:

- one Finding belongs to one ReportRun
- one Finding belongs to one Screen
- one Finding references one Guideline

Notes:

- findings are read-only scan results in the MVP
- the normalized contract should not expose raw axe-core payloads as the primary result shape
- raw payloads may be stored internally in the future for debugging, but that is out of scope for the MVP contract

## Report Run Lifecycle

The lifecycle applies to the execution of a Report, not to report configuration itself.

### States

- `pending`
- `running`
- `completed`
- `failed`

### State Meanings

`pending`

- the run record exists
- the system accepted the execution request
- browser automation and scanning have not started yet

`running`

- one or more screens are being processed
- Playwright is opening pages
- axe-core analysis is being executed
- findings may already exist internally, but the run is not finalized

`completed`

- all intended screens were processed
- findings were normalized
- results were stored successfully
- summary metrics were finalized
- terminal state

`failed`

- execution started or was about to start
- a fatal error prevented successful completion
- the run ended without a valid completed result set
- terminal state

### Allowed Transitions

- `pending -> running`
- `running -> completed`
- `running -> failed`

No other transitions are valid in the MVP.

### Invalid Transitions

- `pending -> completed`
- `pending -> failed`
- `completed -> running`
- `completed -> failed`
- `failed -> running`
- `failed -> completed`

Once a run reaches `completed` or `failed`, it must not transition again. If the same report is executed again, the system must create a new `ReportRun`.

### Transition Triggers

`pending -> running`

- happens when the backend starts processing the run
- execution start timestamp may be recorded

`running -> completed`

- happens when all selected screens were processed
- findings were normalized and stored
- summary data was finalized without fatal errors

`running -> failed`

- happens when a fatal runtime error interrupts the audit flow
- required processing cannot continue safely
- findings cannot be finalized correctly

### Lifecycle Diagram

```text
pending
   |
   v
running
  | \
  |  \
  v   v
failed completed
```

## Audit Execution Flow

### Preconditions

Before a report can be executed in the MVP, the following must already exist:

- a Project
- an Application inside that project
- one or more Screens inside the application
- a Report associated with the application
- one or more selected screens linked to the report

The Application provides execution settings such as:

- device type
- viewport width
- viewport height
- wait time before scanning
- accessibility target level

The Report may also include optional authentication data.

### High-Level Flow

```text
Project
  -> Application
    -> Screens
      -> Report
        -> ReportRun created
          -> pending
            -> running
              -> Playwright opens each selected screen
              -> wait configured time
              -> axe-core scans page
              -> raw results collected
              -> results normalized into Findings
              -> Findings stored
            -> completed or failed
              -> results exposed for retrieval
```

### Execution Steps

1. A Report is defined for an existing Application with selected screen IDs and optional authentication configuration.
2. When execution is requested, the backend creates a new `ReportRun` with initial status `pending`.
3. When the backend begins processing, the run transitions to `running`, records execution timestamps as needed, resolves report configuration, and loads the selected screens.
4. For each selected screen, the runner opens the target page with Playwright, applies the configured viewport and runtime settings, performs authentication if supported and enabled, waits for the configured stabilization time, and runs axe-core.
5. axe-core returns raw accessibility analysis results for the scanned screen.
6. The backend normalizes raw results into `Finding` records by mapping tool-specific output into the stable domain model.
7. Each Finding is linked to the current ReportRun, the originating Screen, and the related Guideline.
8. Findings are stored in the backend-owned database together with their run, screen, and guideline references.
9. After all selected screens are processed successfully, the backend finalizes the run summary and transitions the run to `completed`.
10. If a fatal error prevents successful completion, the run transitions to `failed` and may store high-level error information.
11. Once the run reaches a terminal state, the backend exposes run status, summary data, and normalized findings for retrieval.

### Normalization Rules

The MVP normalization approach should follow these rules:

- findings are normalized backend records, not raw axe-core objects
- findings reference domain entities instead of only embedded strings
- guideline mapping is explicit
- screen association is explicit
- run association is explicit

Expected normalized data may include:

- rule code
- message or explanation
- selector path
- relevant HTML snippet or element context
- impact or severity information when available

### Failure Handling

Examples of high-level failure conditions include:

- browser automation cannot continue
- required page processing fails fatally
- result normalization cannot complete safely
- findings cannot be stored correctly

The MVP does not define retries or partial-success strategies.

## API Design Principles

The backend follows a resource-oriented API structure.

### Resource Responsibilities

Projects:

- top-level business container
- own applications

Applications:

- audit target and runtime configuration boundary
- own screens and reports
- store execution defaults such as device, viewport, wait time, and accessibility target

Reports:

- reusable audit definitions
- define which application-owned screens are selected for execution
- optionally store authentication configuration
- act as the parent resource from which report runs are created

ReportRuns:

- execution instances of reports
- expose lifecycle state, timestamps, and summary data
- provide the retrieval boundary for findings

Findings:

- normalized accessibility issues produced by a specific run
- read-only in the MVP

Guidelines:

- read-only reference data used to classify findings

### Naming Rules

Use these conventions consistently:

- plural resource names in URLs
- kebab-case path segments
- nested routes when the parent-child relationship is part of the resource identity
- `report-runs` instead of shorthand aliases such as `/runs`

### Recommended MVP Endpoint Groups

```text
Projects
POST /projects
GET /projects
GET /projects/{projectId}
PATCH /projects/{projectId}

Applications
POST /projects/{projectId}/applications
GET /projects/{projectId}/applications
GET /applications/{applicationId}
PATCH /applications/{applicationId}
PUT /applications/{applicationId}/screens

Reports
POST /applications/{applicationId}/reports
GET /applications/{applicationId}/reports
GET /reports/{reportId}
PATCH /reports/{reportId}

ReportRuns
POST /reports/{reportId}/report-runs
GET /reports/{reportId}/report-runs
GET /report-runs/{reportRunId}

Findings
GET /report-runs/{reportRunId}/findings
GET /findings/{findingId}

Guidelines
GET /guidelines
GET /guidelines/{guidelineId}
```

Application creation may include an initial `screens` collection in the request body. `GET /applications/{applicationId}` may return the current screen collection so screens can be managed within the same application workflow.

### API Contract Expectations

- OpenAPI is the source API contract
- Swagger UI is used for interactive exploration
- examples should demonstrate one coherent MVP workflow across projects, applications, screens, reports, report runs, findings, and guidelines
- validation and execution error examples should be operation-specific

## Persistence Strategy

The MVP backend owns the audit-domain database.

The database stores:

- projects
- applications
- application-owned screens
- reports
- report runs
- guidelines
- findings

PostgreSQL is the recommended database for the MVP. The key requirement is that persistence remains reasonably isolated from the API contract and audit execution logic so the implementation can evolve later.

## Validation and Testing Strategy

The MVP includes several validation layers:

1. Swagger UI for interactive API exploration
2. Postman collections for manual end-to-end workflow validation
3. Playwright API tests for automated endpoint validation
4. real audit validation to confirm that Playwright and axe-core work correctly on real pages

### Definition of Done

The MVP backend is considered complete when this workflow works end to end:

1. create a project
2. create an application with one or more screens
3. create a report
4. trigger a report run
5. Playwright loads each selected screen
6. axe-core performs the scan
7. findings are collected and normalized
8. findings are linked to screens, report runs, and guideline references
9. the API returns normalized results

Additionally:

- API documentation is available via Swagger
- Postman collections validate the full flow
- automated API tests run successfully

## Repository and Runtime Architecture

### Development-Time Shape

```text
Swagger UI / Postman
        |
        v
Dockerized Node Backend
        |- API layer
        |- Audit runner
        \- persistence access
                |
                v
            PostgreSQL
```

### Future Production-Oriented Shape

```text
Client Application or Integration
        |
        v
Node API
   |- PostgreSQL
   \- Job Queue
           |
           v
   One or More Audit Workers
           |
           v
   Playwright + axe-core
```

### Repository Structure

```text
apps/
  api/
  worker/

packages/
  audit-engine/
  contracts/
  shared/

tests/
  api/
  fixtures/

docs/
postman/
openapi.yaml
docker-compose.yml
```

Responsibilities:

- `apps/api` owns HTTP and API exposure
- `apps/worker` owns background execution
- `packages/audit-engine` owns Playwright and axe-core execution logic
- `packages/contracts` owns shared schemas and domain models
- `tests/` owns validation coverage

### Containerization

The backend should run in Docker containers to provide:

- consistent local setup
- reliable Playwright environment
- easier CI execution
- easier future scaling

Containers should include:

- Node runtime
- Playwright dependencies
- browser runtime support
- PostgreSQL connectivity

## Security Considerations

Important MVP security concerns include:

- secure handling of authentication credentials
- isolation of browser execution
- safe Docker configuration
- controlled access to internal environments being audited

These concerns become more important once authenticated scanning is introduced.

## Technology Stack

- backend API: Node.js
- browser automation: Playwright
- accessibility engine: axe-core
- database: PostgreSQL
- API documentation: OpenAPI and Swagger
- manual API validation: Swagger UI and Postman
- automated API validation: Playwright API tests
- containerization: Docker

## Future Evolution

Potential later enhancements include:

- authenticated application scanning beyond the MVP baseline
- report history comparison
- trend analysis
- export functionality
- scheduled scans
- CI integration
- multiple worker execution
- issue lifecycle management

The project should start as simply as possible while preserving a path to queue-based execution, worker separation, and larger-scale reporting in later phases.
