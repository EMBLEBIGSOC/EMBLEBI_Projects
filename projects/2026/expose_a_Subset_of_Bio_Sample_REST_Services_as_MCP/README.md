# AI-Assisted BioSamples Workflows with Model Context Protocol

**Contributor:** Dineshkumar Anbalagan  
**Organisation:** EMBL-EBI  
**Programme:** Google Summer of Code 2026  
**Mentors:** Dipayan Gupta, Senthilnathan Vijayaraja, Vishnukumar Balavenkataraman kadhirvelu

---

## Project summary

This Google Summer of Code project explores how the **Model Context Protocol (MCP)** can make EMBL-EBI BioSamples workflows easier to use from AI assistants and other MCP-compatible clients.

The work was developed as three connected components rather than a single monolithic application:

1. **BioSamples MCP Server** — search, retrieve, and submit BioSamples records.
2. **JSON Schema Store MCP Server** — retrieve BioSamples checklist schemas, extract structured sample metadata from natural language, and validate drafts.
3. **Authentication Portal** — authenticate Webin users and securely make short-lived authentication tokens available to the submission workflow through Redis.

Together, these components form an end-to-end workflow in which a user can describe a biological sample in natural language, create a checklist-aware draft, identify missing or invalid fields, inspect existing BioSamples records, authenticate when required, and submit a reviewed sample.

---

## What was built

### 1. BioSamples MCP Server

The `biosamples-mcp` project provides the MCP-facing integration with the EMBL-EBI BioSamples API.

Implemented MCP tools include:

- `biosamples_serverinfo` — reports server information and supported capabilities.
- `biosamples_searchsamples` — searches BioSamples using free-text queries, pagination, structured filters, and date ranges.
- `biosamples_getsample` — retrieves a BioSamples record by accession.
- `biosamples_submitsample` — submits a validated and user-confirmed BioSamples payload.

The search workflow supports BioSamples filter types including:

- Attribute (`attr`)
- Accession (`acc`)
- Relationship (`rel`)
- Reverse relationship (`rrel`)
- Domain (`dom`)
- Name (`name`)
- External data (`extd`)
- Release/update date ranges

Submission is deliberately treated differently from read-only operations. It requires an explicit Webin ID, retrieves the corresponding cached authentication token, and does not cache the write operation.

The server also includes:

- Dynamic MCP tool registration
- Dependency injection
- Middleware-based execution
- Structured logging
- Redis-backed response caching for cacheable tools
- Centralised BioSamples API error handling
- Separate request context and tool-execution layers
- GitHub Actions test workflow

---

### 2. JSON Schema Store MCP Server

The `json-schema-store-mcp` project adds checklist awareness and AI-assisted sample preparation.

Implemented MCP tools include:

- `jsonschema_serverinfo`
- `jsonschema_extract_sample_draft`
- `jsonschema_validate_sample_draft`

The extraction workflow accepts a natural-language biological sample description and retrieves the requested BioSamples checklist schema from EMBL-EBI. The schema and user description are then used to generate structured BioSamples-compatible JSON.

The extraction logic is designed to:

- Preserve checklist property names
- Avoid inventing missing user data
- Omit unsupported optional fields
- Keep missing required fields absent so they can be detected during validation
- Normalise biological information where appropriate
- Generate `taxId` and organism information when the organism can be confidently identified
- Retain extraction metadata for generated values

When no checklist is supplied, the server uses **`ERC000011`** as the default checklist.

Validation uses **JSON Schema Draft 7** and reports:

- Whether the draft is valid
- Missing required fields
- Invalid fields
- Validation rule involved
- Expected value
- Actual value
- User-friendly validation messages

The project also includes:

- EMBL-EBI JSON Schema API integration
- Google Gemini-based metadata extraction
- Dynamic MCP tool loading
- Dependency injection
- Middleware execution pipeline
- Optional Redis caching
- Structured API errors
- GitHub Actions test workflow

---

### 3. Webin Authentication Portal

The `python_login_portal` project provides the browser-based authentication step needed before a protected BioSamples submission.

It is implemented with **FastAPI**, **Jinja2**, **HTTPX**, sessions, and **Redis**.

The portal:

- Displays a Webin login form
- Sends credentials to the configured Webin token endpoint
- Handles invalid credentials and upstream connection failures
- Normalises the Webin username before authentication
- Stores the returned token in Redis using a deterministic SHA-256-based key
- Uses a short token lifetime
- Returns the user to a success screen after authentication
- Reports authentication-storage failures without exposing the token

The cached token is later retrieved by the BioSamples MCP submission tool using the user's Webin ID, connecting the browser authentication flow with the MCP submission flow.

---

## End-to-end workflow

The three components work together as one BioSamples assistant workflow:

```text
                    Natural-language sample description
                                    |
                                    v
                    JSON Schema Store MCP
                      - Fetch checklist schema
                      - Extract structured metadata
                      - Build BioSamples draft
                                    |
                                    v
                    Checklist validation
                      - Missing required fields
                      - Invalid values
                      - Clarification information
                                    |
                                    v
                    User reviews / completes the sample
                                    |
                                    v
                    BioSamples MCP
                      - Search existing samples
                      - Retrieve samples by accession
                      - Prepare for submission
                                    |
                                    v
                    Authentication required?
                            |                 |
                           No                Yes
                            |                 |
                            |          FastAPI Login Portal
                            |          - Webin authentication
                            |          - Token stored in Redis
                            |                 |
                            +-----------------+
                                    |
                                    v
                    User-confirmed BioSamples submission
```

A key design goal was to keep **read operations, AI-assisted preparation, validation, authentication, and write operations clearly separated** while still allowing them to participate in one conversational workflow.

---

## Architecture and engineering work

Across the MCP projects, the implementation follows a reusable layered structure:

```text
MCP Client
   |
   v
Dynamic Tool Loader
   |
   v
MCP Tool Orchestrator
   |
   v
Execution Pipeline
   |
   +--> Logging Middleware
   +--> Authentication Middleware
   +--> Cache Middleware
   |
   v
Tool Service
   |
   v
Domain / Adapter Layer
   |
   v
EMBL-EBI APIs / Redis / LLM service
```

Important engineering work completed during the project includes:

- Asynchronous API communication with `httpx`
- MCP tool metadata and JSON input schemas
- Decorator-based component and tool registration
- Dynamic service discovery
- Dependency injection
- Reusable middleware architecture
- Request-scoped execution context
- Redis caching with configurable TTL
- Cache bypass for submission operations
- Structured application logging
- Structured API error models with retryability information
- Unit testing of adapters, middleware, tools, domain services, loaders, and orchestration
- CI test workflows for both MCP repositories

---

## Project stats

Across the submitted source archives:

- **3 connected applications**
- **7 MCP tools**
  - 4 BioSamples tools
  - 3 JSON Schema tools
- **123 test functions** across the three projects
  - 58 in `biosamples-mcp`
  - 37 in `json-schema-store-mcp`
  - 28 in `python_login_portal`
- **3 GitHub Actions test workflows**
- **Python 3.11+** for the MCP servers
- Redis-backed caching/authentication with a **300-second default TTL**

---

## Code

### BioSamples MCP

- **Upstream repository:** 
  - https://github.com/EBIBioSamples/biosamples-mcp
- **Contributor fork:** 
  - https://github.com/Dinesh-kumar-Anbalagan/biosamples-mcp
- **Development branch:**
  - https://github.com/Dinesh-kumar-Anbalagan/biosamples-mcp/tree/feature/bioSampleMCPTools
  - https://github.com/Dinesh-kumar-Anbalagan/biosamples-mcp/tree/feature/bioSample_bugfix_inital_changes
- **Peer Reviews:** 
  - https://github.com/EBIBioSamples/biosamples-mcp/pull/1
  - https://github.com/EBIBioSamples/biosamples-mcp/pull/3
                          

### JSON Schema Store MCP
- **Upstream repository:** 
  - https://github.com/EBIBioSamples/json-schema-store-mcp
- **Contributor fork:** 
  - https://github.com/Dinesh-kumar-Anbalagan/json-schema-store-mcp
- **Development branch:**
  - https://github.com/Dinesh-kumar-Anbalagan/json-schema-store-mcp/tree/feature/Json_Schema_core_code
- **Peer Reviews:** 
  - https://github.com/EBIBioSamples/json-schema-store-mcp/pulls

### Webin Authentication Portal
- **Upstream repository:** 
  - https://github.com/EBIBioSamples/mcp-auth
- **Contributor fork:** 
  - https://github.com/Dinesh-kumar-Anbalagan/mcp-auth
- **Development branch:**
  - https://github.com/Dinesh-kumar-Anbalagan/mcp-auth/tree/main
- **Peer Reviews:** 
  - https://github.com/EBIBioSamples/mcp-auth/pull/1

---

## Current state

The project currently demonstrates the major building blocks needed for an MCP-assisted BioSamples workflow:

- BioSamples search is implemented.
- Sample retrieval by accession is implemented.
- Natural-language draft extraction is implemented.
- Checklist retrieval and Draft 7 validation are implemented.
- Missing and invalid fields are surfaced in structured validation output.
- Authenticated BioSamples submission is implemented.
- A browser-based Webin authentication flow is connected to submission through Redis.
- Read-only MCP calls can use response caching while submission bypasses cache.
- Automated tests cover the main architectural layers.

The three projects therefore provide a working foundation for conversational BioSamples discovery, sample preparation, validation, authentication, and submission.

---

## What's left / future work

Possible next steps include:

- Deploying the authentication portal behind production HTTPS
- Adding full end-to-end integration tests spanning all three services.
- Expanding testing against additional BioSamples checklist schemas.
- Adding richer clarification-question generation for incomplete biological metadata.
- Packaging the three components for simpler local or container-based deployment.
- Continuing API and error-handling hardening for production use.
- Merging or publishing the remaining development branches upstream where appropriate.

---

## Challenges and learnings

### Designing an MCP architecture instead of a collection of API wrappers

One of the main challenges was deciding how tool discovery, middleware, dependency injection, API adapters, and domain logic should fit together. Separating these responsibilities made it possible to reuse the same execution architecture across both MCP servers.

### Turning natural language into schema-constrained biological metadata

LLM extraction alone is not sufficient for a scientific submission workflow. The generated data has to respect checklist field names without inventing values simply to make the schema pass. Separating extraction from deterministic JSON Schema validation made the workflow safer and easier to reason about.

### Handling missing required metadata

A useful draft should represent what the user actually said. Missing required fields therefore remain missing during extraction and are reported later by the validator instead of being filled with guessed or placeholder values.

### Supporting BioSamples filtering correctly

BioSamples exposes several specialised filter formats rather than a single generic filter. Building a dedicated filter layer made it possible to support attributes, accessions, relationships, reverse relationships, domains, names, external references, and date ranges consistently.

### Adding authentication to an MCP submission workflow

Read-only tools can be called directly, but submitting a sample requires user authentication. Connecting an MCP tool call to a browser login flow required a separate authentication service and a shared short-lived Redis token store.

### Treating writes differently from reads

Caching is useful for search, retrieval, and other repeatable operations, but it must not accidentally replay a submission. The middleware therefore explicitly excludes the BioSamples submission tool from response caching.

### Error handling across external services

The project communicates with BioSamples, the JSON Schema Store, Redis, Webin authentication, and an LLM service. Each has different failure modes. Building structured error handling and graceful fallbacks was an important part of making the workflow understandable to both MCP clients and users.

### Working across multiple services

The project evolved into three cooperating applications. Keeping their responsibilities separate while ensuring that checklist validation, authentication, token lookup, and submission still worked as one workflow was one of the most valuable architectural lessons from the project.

---

## Technologies used

- Python
- Model Context Protocol (MCP)
- FastAPI
- HTTPX
- Redis
- dependency-injector
- JSON Schema / Draft 7
- Google Gemini API
- Jinja2
- pytest
- pytest-asyncio
- GitHub Actions

---

## Acknowledgements

This project was completed as part of **Google Summer of Code 2026** with **EMBL-EBI**.

I would like to thank my mentors for their guidance, technical feedback, code reviews, and support throughout the project.
