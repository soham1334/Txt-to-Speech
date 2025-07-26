# Plugin-Based Data Ingester SDK – Detailed Architecture & Roadmap

## 🔰 Introduction

The **Plugin-Based Data Ingester SDK** is a modular and extensible framework for building and orchestrating data ingestion pipelines. Designed with a plugin-first architecture, it allows developers to pull data from various sources—such as **web pages**, **Git repositories**, and **cloud storage**—and push it into structured destinations like **APIs**. The SDK supports both **scheduled (cron)** and **event-based (webhook)** triggers, enabling flexible automation workflows.

This architecture document describes the system’s design, key components, technical constraints, and a week-by-week implementation roadmap. It is intended for **developers** building or extending plugins, **DevOps teams** managing deployment and integration, and **technical leads or product stakeholders** seeking a deeper understanding of the system’s structure and delivery plan.

---

## 1. ✅ High-Level Architecture

```
+---------------------------------------------------------------+
|                     Global Trigger Manager                    |
|    (Schedules & triggers ingestion via Cron/Webhook API)      |
+---------------------------------------------------------------+
                         |
                         ▼
+---------------------------------------------------------------+
|                    Ingestion Job Manager                      |
|  (Manages and monitors orchestrators for all ingestion jobs)  |
+---------------------------------------------------------------+
                         |
                         ▼
+-------------------------- Ingestor ---------------------------+
|           Orchestrates one job: source -> transform -> dest   |
+---------------------------------------------------------------+
           ▲                                    ▼
+----------------------+           +-----------------------------+
|     Source Plugin     |           |     Destination Plugin     |
| - Web, Git, Cloud     |           | - API Endpoint             |
| - Fetches data        |           | - Sends data               |
+----------------------+           +-----------------------------+
```

## 🎯 Key Components Summary

---

### 🔁 Global Trigger Manager

The **Global Trigger Manager** is responsible for starting ingestion jobs based on external triggers. These triggers can come in two main forms:

- **Cron Jobs:** Scheduled executions defined using cron expressions. For example, a user might configure a data ingestion task to run every hour or every day .
- **Webhooks:** An HTTP API endpoint is exposed, and when it's called (usually by another system), it initiates the ingestion process.

This component ensures that jobs start at the right time or in response to specific external events. It also validates and prevents duplicate triggers for the same job when one is already running. Think of it as the gatekeeper that decides _when_ to start ingestion.

---

### 🧠 Ingestion Job Manager

The **Ingestion Job Manager** maintains a registry of all ingestion jobs in the system. For each job, it keeps track of metadata like:

- The job's unique ID
- Its current state (idle, running, completed, failed)
- The associated source and destination configurations
- Last run time, success/failure logs, etc.

It provides programmatic functions like starting or stopping a job, querying job status, and removing or updating jobs. This manager is essential for lifecycle control, enabling you to manage multiple ingestion jobs in a coordinated way. It also plays a key role in monitoring and debugging.

---

### ⚙️ Ingestor

The **Ingestor** is the core orchestration engine of the SDK. It takes a source plugin and a destination plugin and executes the ingestion pipeline:

1. Initializes and uses the source plugin to fetch the data
2. Optionally applies a transformation or processing step
3. Sends the fetched/processed data to the destination plugin

This component is designed to be lightweight, modular, and reusable. Each Ingestor instance handles a single run of a job, making it highly testable and predictable. It’s the "worker" that does the actual work of pulling and pushing data.

---

### 🌐 Source Plugin

A **Source Plugin** is a module that knows how to fetch data from a specific type of source. The SDK supports several types of source plugins:

- **Web Source Plugin:** Crawls web pages via sitemaps or recursive links
- **Git Source Plugin:** Fetches data from version control systems like Git
- **Cloud Storage Plugin:** Pulls files from providers like Google Drive or AWS S3

Each plugin implements a common interface to ensure consistency, regardless of data source. They encapsulate logic for authentication, discovery, and data retrieval, abstracting away the complexity from the core system.

---

### 📤 Destination Plugin

The **Destination Plugin** is responsible for sending data to a configured target endpoint. Initially, the SDK will support:

- A **generic API endpoint** that accepts HTTP POST requests
- **Authentication mechanisms**, such as tokens or API keys
- **Batching** capabilities to improve throughput and reduce network overhead

Destination plugins follow a uniform interface, making them easy to replace or extend. In the future, additional destinations like databases, cloud functions, or message queues can be supported.

---

## 2. 🧰 Low-Level Architecture

### 📁 SDK Project Structure

```text
plugin-ingester-sdk/
project-root/
├── src/
│   ├── datasources/
│   │   ├── CloudStorage-crawler/
│   │   │   ├── providers/
│   │   │   │   ├── AzureBlobProvider.ts
│   │   │   │   ├── GDriveProvider.ts
│   │   │   │   └── S3Provider.ts
│   │   │   ├── CloudStorageCrawler.ts
│   │   │   ├── index.ts
│   │   │   └── types.ts
│   │   ├── git-crawler/
│   │   │   ├── GitCrawler.ts
│   │   │   └── index.ts
│   │   ├── HTTP-crawler/
│   │   │   ├── HTTPCrawler.ts
│   │   │   └── index.ts
|   |   |
│   │   └── types/
│   │       ├── axios.ts
│   │       └── api.yaml
│   ├── events/
│   │   ├── handleGitIngestion.ts
│   │   ├── handleHttpIngestion.ts
│   │   └── handleCloudIngestion.ts
│   ├── eventsources/
│   │   ├── cron
|   |   |   ├── git.ts
│   │   │   ├── http.ts
│   │   │   └── cloud.ts
│   │   └── webhook/
│   │       ├── git.ts
│   │       ├── http.ts
│   │       └── cloud.ts
│   ├── functions/
│   │   ├── ingestGit.ts
│   │   ├── ingestHttp.ts
│   │   └── ingestCloud.ts
│   ├── ingester/
│   │   ├── IngesterJobManager.ts
│   │   ├── ingesterOrchestrator.ts
│   │   └── GlobalTriggerManager.ts
|   ├── utils/
│   │   ├── bufferStream.ts
│   │   └── event-emitters/
│   │       ├── GitCrawlerEventEmitter.ts
│   │       ├── HTTPCrawlerEventEmitter.ts
│   │       └── CloudStorageEventEmitter.ts
│   └── index.ts
├── config/
│   ├── default.yaml
│   └── custom-environment-variables.yaml
├── .env
├── package.json
└── README.md

```

---

## 📌 Component-Level Details

---

### 📌 `GlobalTriggerManager.ts`

This file is responsible for managing **how and when** ingestion jobs are triggered. It supports two main mechanisms:

- **Cron Scheduling**: Uses libraries like `node-cron` to schedule ingestion jobs periodically based on cron expressions. For example, a job can be configured to run every hour or once a day.
- **Webhooks**: Exposes HTTP endpoints via Express, allowing external systems to trigger ingestion jobs on-demand (e.g., when a GitHub push event occurs).

The main responsibilities of this component include:

- Registering new ingestion jobs with associated triggers.
- Starting or stopping jobs based on schedule or webhook events.
- Preventing duplicate runs by checking if a job is already running.
- Retrying or re-triggering jobs by ID.

This acts as the centralized event entry point for the ingestion lifecycle.

---

### 📌 `IngesterJobManager.ts`

This acts as the **central job controller** and registry that keeps track of every configured ingestion job.

Key responsibilities:

- Maintain a list of all active and scheduled jobs.
- Track job metadata including:
  - Unique job ID
  - Associated source and destination configurations
  - Last execution time
  - Current job status (idle, running, completed, failed)
- Provide an API-like interface to manage jobs programmatically:

```js
registerJob(jobConfig); // Adds a new job
startJob(jobId); // Starts execution of a job
stopJob(jobId); // Stops a running job
getJobStatus(jobId); // Retrieves status and metadata of a job
```

This module ensures that jobs can be dynamically added, removed, or monitored at runtime.

---

### 📌 `ingesterOrchestrator.ts`

`OrchestrateCrawl` is the core class responsible for executing a single ingestion job. It:

1. Dynamically loads the crawler plugin via `pluginPath`.
2. Creates a default event emitters (`git`, `http`, or `cloud`) if none is provided.
3. Calls `connect()` (if present) and then `execute()` on the plugin.
4. Emits `crawler:success` or `crawler:failure` events.

It supports:

- Custom or default event emitters (`Git`, `HTTP`, `CloudStorage`)
- Basic logging via listeners
- Future extensibility for retries, transforms, or tracking.

---

### 📌 `SourcePlugin.ts` Interface

This is a base interface that all source plugins (Git, Web, Cloud, etc.) implement. It defines the contract for how the SDK expects to interact with any data source.

```ts
class SourcePlugin {
  constructor(config) {}

  async connect() {
    // Optional: setup credentials, validate connection
  }

  async execute() {
    throw new Error("Not Implemented");
  }
}
```

Each source plugin implement `execute()` to return raw content. `connect()` can be used to set up API tokens or auth headers.

---

<!-- ### 📌 `DestinationPlugin.ts` Interface

This is the standard contract that all output/destination plugins must follow. It enables consistent sending of data to any destination like an API, database, or message queue.

```js
class DestinationPlugin {
  constructor(config) {}

  async initialize() {
    // Optional: setup connection or auth
  }

  async sendData(data) {
    throw new Error("Not Implemented");
  }

  async sendBatch(dataArray) {
    // Optional: optimized batch sending
  }

  async getStatus() {
    // Optional: reports connectivity or send stats
  }
}
```

- `sendData(data)` is required — handles single data unit transmission.
- `sendBatch(dataArray)` is optional but useful for performance.
- `getStatus()` can help with monitoring or debugging.

--- -->

---

### 📁 `events/`

These files define **event handler functions** that respond when a trigger (cron or webhook) is fired. They act as middlemen between the event and ingestion logic.

- **`handleGitIngestion.ts`**  
  Triggers the Git crawler pipeline. Parses job config and invokes `ingestGit`.

- **`handleHttpIngestion.ts`**  
  Activated on HTTP API crawling triggers. Passes config to `ingestHttp`.

- **`handleCloudIngestion.ts`**  
  Manages cloud storage ingestion. Detects the cloud provider and triggers `ingestCloud`.

---

### 📁 `eventsources/`

Defines **how ingestion jobs are triggered**, either via scheduled (cron) or real-time (webhook) mechanisms.

#### 📁 `cron/`

Used to **periodically trigger** ingestion at scheduled intervals.

- **`git.ts`**  
  Schedules recurring Git repo crawl jobs.

- **`http.ts`**  
  Periodically polls an HTTP API and starts the ingestion.

- **`cloud.ts`**  
  Repeatedly scans a Cloud Storage bucket/folder for new files.

#### 📁 `webhook/`

Used when an **external service calls** the system to trigger ingestion.

- **`git.ts`**  
  Listens for Git webhook events like push/merge and triggers ingestion.

- **`http.ts`**  
  Accepts webhook calls for API crawling, useful for on-demand fetches.

- **`cloud.ts`**  
  Listens for events from cloud storage systems like file uploads or changes.

---

### 📁 `functions/`

Contains **core logic** that connects to the source, fetches data, and sends it to a destination.

- **`ingestGit.ts`**  
  Orchestrates the GitCrawler. Connects, pulls data, and passes it to the destination plugin.

- **`ingestHttp.ts`**  
  Configures and runs the HTTPCrawler. Handles API authentication, headers, and parsing.

- **`ingestCloud.ts`**  
  Runs the cloud storage crawler based on provider (S3, GDrive, Azure), fetches file content.

---

## 🧰 Component Breakdown: `utils/`

The `utils/` directory includes helpers and custom event emitters used to provide observability and status tracking across the data ingestion lifecycle.

---

### 📄 `bufferStream.ts`

- **Purpose**: Converts a `Readable` stream (e.g., from S3 or GDrive) to a `Buffer`.
- **Use Case**: Required when binary content from cloud sources needs to be stored, parsed, or passed downstream.

---

### 📁 `event-emitters/`

#### 📄 `GitCrawlerEventEmitter.ts`

- **Purpose**: EventEmitter for Git ingestion.
- **Events Emitted**:
  - `gitcrawler:success` — Triggered on successful Git repo crawling.
  - `gitcrawler:failure` — Triggered on failure, logs error and config.
- **Use Case**: Observability into Git crawling process; plugs into Git ingestion pipeline.

#### 📄 `HTTPCrawlerEventEmitter.ts`

- **Purpose**: Emits events for HTTP data crawling (e.g., APIs, RSS feeds).
- **Events Emitted**:
  - `crawler:success` — API or HTTP response successfully fetched and parsed.
  - `crawler:failure` — Crawling failed due to network or logic error.
- **Use Case**: Debugging HTTP pipeline, logging success/error details with config.

#### 📄 `CloudStorageEventEmitter.ts`

- **Purpose**: Handles generic and provider-specific events for cloud-based ingestion (S3, GDrive, Azure).
- **Events Emitted**:
  - `cloudstorage:success`, `cloudstorage:failure`, `cloudstorage:progress`, `cloudstorage:file`
  - Provider-level: `s3:success`, `gdrive:failure`, etc.
- **Use Case**: Granular observability into multi-provider cloud ingestion. Logs both success data and error context.

---

## 🗓️ Implementation Roadmap (3 Weeks)

---

### 🔹 Week 1: Core Architecture & Orchestration Foundation

**Objective:** Establish project scaffolding, define interfaces, and implement core orchestrator and job manager.

1. **Project Setup**

   - Initialize project (`npm init` or `pnpm init`)
   - Setup TypeScript, ESLint, Prettier, and testing framework (Jest/Vitest)
   - Create directory structure:
     - `/events`, `/eventsources`, `/functions`, `/utils`, etc.

2. **Define Plugin Interfaces**

   - Implement `SourcePlugin.ts` and `DestinationPlugin.ts` base interfaces
     - Define `connect()`, `execute()`, `sendData()`, `sendBatch()`, `getStatus()`

3. **Build Orchestrator**

   - Implement `ingesterOrchestrator.ts` (or `OrchestrateCrawl`)
     - Accept plugin path, emitter, and config
     - Dynamically load and execute source plugin
     - Hook emitter events (e.g., success/failure)

4. **Job Management Layer**

   - Implement `IngesterJobManager.ts`
     - Register, start, stop, and get job status
     - Maintain in-memory job registry with metadata (ID, status, timestamps)

5. **Trigger Management Layer**

   - Build `GlobalTriggerManager.ts`
     - Integrate `node-cron` for schedule-based triggering
     - Spin up Express server for webhook routes
     - Route events to `/events/handle*.ts` handlers

6. **Utility Setup**
   - Implement `logger.ts` using `Pino` or `Winston`
   - Create `bufferStream.ts` for stream-to-buffer conversion

---

### 🔹 Week 2: Plugins, EventSources & Ingestion Logic

**Objective:** Implement ingestion pipelines, crawling logic, and event source configurations.

1. **Core Ingestion Functions**

   - `ingestGit.ts` — clone and read Git files
   - `ingestHttp.ts` — make HTTP request, parse response
   - `ingestCloud.ts` — connect to S3/GDrive, fetch files

2. **Source Plugins**

   - Create Git, HTTP, and Cloud source plugin classes using shared interface
     - Implement `connect()` and `execute()` methods

3. **Destination Plugin**

   - Accept data and `POST` to external API with headers
   - Implement batching and basic retry logic

4. **Event Emitters**

   - `GitCrawlerEventEmitter.ts`, `HTTPCrawlerEventEmitter.ts`, `CloudStorageEventEmitter.ts`
     - Emit and log success/failure events
     - Attach to orchestrator for runtime observability

5. **Event Source Triggers**

   - Implement `eventsources/cron/*.ts` and `webhook/*.ts` files
     - Git: trigger via cron/webhook on repo changes
     - HTTP: API polling and on-demand via webhook
     - Cloud: file presence or upload trigger

6. **Ingestion Event Handlers**
   - `handleGitIngestion.ts`, `handleHttpIngestion.ts`, `handleCloudIngestion.ts`
     - Bridge event to orchestrator
     - Pass config and emitter

---

### 🔹 Week 3: CLI, Integration Testing & Documentation

**Objective:** Finalize the SDK with a test CLI, end-to-end validations, docs, and packaging.

1. **CLI Tool / Express API**

   - Build simple CLI using `commander` (e.g., `ingest run <job>`)
   - Optionally: Add lightweight Express server for job control endpoints

2. **Plugin Loader System**

   - Dynamic loading of source/destination plugins via job config
   - Allow for scalable plugin extension

3. **End-to-End Testing**

   - Simulate Git/HTTP/Cloud to API ingestion
   - Validate:
     - Job orchestration
     - Triggers and handlers
     - Event logs and error paths

4. **Documentation**

   - `README.md`:
     - Setup, usage, CLI reference
     - Plugin dev guide
     - Architecture diagram (optional)
   - Inline code comments for all exported functions

5. **Publish SDK**

   - Finalize `package.json` with types/entry points
   - Tag and publish to NPM (scoped/private if needed)
   - Create GitHub release

6. **Optional Enhancements**
   - Add `transform()` hook in `ingesterOrchestrator.ts` before sending data
   - Implement persistent job store (SQLite or JSON file)

---

## ✅ Final Deliverables

- 📦 Installable SDK via NPM
- ⚙️ Orchestrator + Job Manager + Event Triggers
- 🌐 Web, Git, and Cloud plugins
- 🧪 Ingestion test suite (E2E & unit)
- 🛠️ CLI or Express API interface
- 📚 Docs: README + plugin guide
- 🔬 80%+ test coverage
