
# DScholars Technical Blueprint

## Architecture Overview

DScholars is a lightweight, scalable web + mobile product that exposes a student-friendly academic search and reading experience. The system combines:

•	A responsive frontend (web and mobile)

•	A backend API layer that orchestrates search, 
summarization, authentication, and data storage.

•	Integration with external scholarly APIs and the AI/NLP service for summaries.

•	A persistence layer for user data, saved items, and usage metrics.

##### High-level Flow

DScholars is a single product surface, available on the web and mobile, which enables learners to discover, read, and cite scholarly work in plain English. It has three logical layers:

•	A frontend layer where users would then connect and search-essentially, the web app, browser extension, and mobile app.

•	The back-end retrieves scholarly metadata/content from an outside scholarly API or an existing search index and then invokes an AI summarization service, if necessary.

•	A data & integration layer that includes a database, cache, object storage, and external scholarly + AI APIs.

•	We favor a shared backend so that web, extension, and mobile clients all use the same business logic and the same data.

```
css
 
 <--HTTPS--> [API Gateway / Backend (Express / Node)] 

       ↕                                      ↕
    Local cache                             [AI/NLP Service]
       ↕                                      ↕

[CDN / Cache (Redis)]            [Scholarly AP(Semantic Scholar, CrossRef, PubMed)]

       ↕                                      ↕

   [Database: MongoDB / Firebase] ----------> [Storage: S3 / Cloud Storage]
```

## Frontend, Backend, and Database Stack

### Frontend 
Target: Web Responsive & Mobile iOS and Android.

#### Framework:
•	Web: React (via Create React App or Vite) with either a component library or Tailwind CSS.

•	Mobile: React Native (with shared components where possible), or Flutter if preferred.

#### Responsibilities:
•	Search and filtering UI, results list, summary view, citation actions, library management, onboarding/tutorial.

•	Local state and caching for immediate responsiveness.

•	Authentication Flows: Google OAuth, guided onboarding.

### Backend
•	Language & Framework: Node.js + Express (or Fastify). Alternative: Python + FastAPI if the team prefers Python.

•	RESTful API Style: The first set of endpoints will be in RESTful format, but GraphQL is also an option for complicated queries.

#### Responsibilities:
•	Coordinate search requests and apply business logic: free-first policy, "free paper only" filter.

•	Call external scholarly APIs to fetch metadata and links.

•	Orchestrate calls to AI summarizer to provide plain-English summaries.

•	Generate and serve citations (formatting).

•	User management, preferences, saved library, metrics.

•	Rate limiting, caching, and error handling.

#### Deployment: 
Containerized, Docker; may be orchestrated on a cloud provider like AWS ECS / Fargate, GCP Cloud Run, or Kubernetes.

### Database and Storage

Primary DB: MongoDB (Atlas) or Firebase Firestore.
   
   •	Stores user profiles, saved papers, user libraries, usage metrics, and annotations.

#### Search/Indexing: 
Use external scholarly APIs as the source of truth; optionally maintain a lightweight internal index for fast faceted search and user-specific signals.

#### Cache: 
Redis is used for query caching, session caching, and rate-limiting counters.

#### Blob Storage: 
S3-compatible storage for any persisted PDFs, user-uploaded files, and generated artifacts - for example, generated summary objects.

#### AI Models:
Serverless, using an external provider such as OpenAI/Anthropic/in-house transformer serving. 

#### Store:
 summary outputs in DB to optimize repeat requests.

## How Components Communicate

All inter-service communication uses HTTPS with JSON payloads. Where low latency or streaming is needed, the design can incorporate WebSockets for live progress updates when summaries are being generated.

### Typical request flows
Search (user asks a query)

1. Frontend sends `POST /api/search with { query, filters }`.

2. Backend checks Redis cache for results.

#### If cache miss:

* Backend queries scholarly APIs (Semantic Scholar / CrossRef / PubMed) for metadata and links.

* Optionally fetches open-access PDFs or links to full text.

* Optionally checks internal search index for cached matches.

4. Backend returns a list of result cards (title, authors, year, source, isOpenAccess boolean, preview/abstract).

5. Frontend shows results. For each card, user can request `GET /api/summary?paperId=xxx.`

#### Summary generation

1. Frontend calls `POST /api/summary with { paperId }`.

2. Backend either returns precomputed summary from DB or:

* Fetches paper abstract/text.

* Sends the text to the AI summarization service (via API).

* Receives summary, stores it in DB, and returns it.

3. Frontend shows summary and one-click citation actions.

#### Save and Library

1. Frontend `POST /api/library with { userId, paperId, tags }`.

2. Backend persists record in MongoDB and returns updated library listing.

#### Citation copy

* `Endpoint GET /api/citation?paperId=xxx&format=apa` returns formatted citation string.

#### Authentication and Security

* Use OAuth2 with Google sign-in to support academic emails but do not require a school email for read access (optional for premium or researcher features).

* Use JWT access tokens for session management with short expiry.

* Rate-limit unauthenticated and authenticated endpoints differently.

## Why This Approach Is Technically Feasible

* Proven building blocks: The stacks proposed (React/React Native, Node.js, MongoDB, Redis, S3) are widely used and well-supported with many libraries. They enable faster delivery for an MVP.

* External scholarly data: APIs such as Semantic Scholar, CrossRef, and PubMed offer metadata and abstracts without needing to crawl academic publisher sites. This reduces legal and technical complexity.

* AI summarization as a service: Using an API (OpenAI, Anthropic, or other) lets you add summarization without building and maintaining large ML models.

* Incremental rollout: Start with metadata + abstracts (open access) and add deeper integrations later (e.g., institutional access, paywalled previews).

* Scalability: Caching hot queries in Redis and caching AI results avoids re-costly summarization. Containerized deployment and managed DB services allow horizontal scaling.

* Compliance and ethics: By prioritizing open access materials and not redistributing copyrighted PDFs, the approach minimizes legal risk. 

Note: Summaries should always link back to the source.

## Data Models
User

```
json

 {
  "userId": "string",
  "email": "string",
  "name": "string",
  "preferences": {
    "defaultCitationFormat": "apa",
    "freeOnly": true
  },
  "library": ["paperId1", "paperId2"]
}

```
### Paper (cached metadata)

```
{   Json

  "paperId": "string",
  "title": "string",
  "authors": ["string"],
  "year": 2023,
  "abstract": "string",
  "source": "Semantic Scholar",
  "isOpenAccess": true,
  "links": {
    "pdf": "https://...",
    "doi": "https://doi.org/..."
  },
  "summary": {
    "text": "plain english summary",
    "createdAt": "ISODate"
  }
}

```
### LibraryItem

```
Json
{
  "userId": "string",
  "paperId": "string",
  "tags": ["health", "policy"],
  "savedAt": "ISODate"
}

```
## Security and Privacy Considerations

* Use HTTPS everywhere.

* Store sensitive secrets (API keys) securely in a secrets manager.

* Minimize storage of raw full-text paywalled documents; store only metadata and summaries.

* Comply with GDPR-like principles: allow users to delete their data (right to be forgotten).

* Rate-limit AI summarization calls and monitor costs.





