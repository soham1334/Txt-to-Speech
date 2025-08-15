# 1. Introduction

This project comprises two core components: a **Crawler SDK** and a **Scheduler SDK**.  

The **Crawler SDK** is a collection of specialized, stateless data source components built with TypeScript and Node.js. Its primary function is to efficiently extract and standardize data from diverse external systems like Git repositories, Google Drive, and HTTP endpoints.  

The **Scheduler SDK** (simulated in this project) acts as the central orchestrator, managing the lifecycle and execution of data ingestion tasks. It determines when and which crawler should run, providing all necessary context for each task.  

> **Important Note:** In this project's current state, the Scheduler SDK is simulated. This means it utilizes an **in-memory database** for task definitions and webhook registrations. There is no real database or persistent storage layer configured for the scheduler's state. All task and webhook data will be reset upon application restart. This setup is ideal for demonstration and testing purposes.

---

# 2. Features

1. **Multi-Source Data Ingestion** – Supports data extraction from various external sources, including Git repositories, Google Drive, and HTTP/HTTPS endpoints.  
2. **Stateless Design** – Crawler components are designed to be stateless, which makes them highly scalable and resilient as they don't retain memory between executions.  
3. **Full Data Scans** – Capability to perform comprehensive initial data extractions from a source.  
4. **Incremental Data Syncs (Delta Sync)** – For supported sources (e.g., Google Drive), the system can fetch only changes since the last successful ingestion by utilizing continuation tokens.  
5. **Webhook-Driven Triggers** – Processes incoming webhook events (e.g., GitHub push events, Google Drive change notifications) to trigger targeted and real-time data ingestion.  
6. **Flexible Configuration** – Allows for easy configuration of crawler behavior and task definitions via YAML files and dynamic payloads.  
7. **Standardized Output** – All extracted data is transformed into a consistent `IngestionData` format, simplifying downstream processing.  
8. **Temporary Local Storage** – Git operations utilize temporary local paths for cloning and pulling, ensuring no persistent local data is left behind after a task completes.  
9. **Robust Error Handling & Logging** – Provides detailed logging and graceful error management across all stages of the ingestion process.  

---

# 3. Architecture Overview

The **Crawler SDK** operates as a set of stateless workers within a larger, loosely coupled architecture. It's designed to be invoked directly by an external **Scheduler SDK**.  

## Key Principles

- **Statelessness** – Each crawler instance created for a task execution is ephemeral. It doesn't maintain any long-term internal state. All necessary information (configuration, authentication, continuation tokens, webhook payloads) is passed to it at the point of invocation.  
- **Single Responsibility** – Each crawler (`git-crawler.ts`, `gdrive-crawler.ts`, `http-crawler.ts`) focuses solely on data extraction from its specific source.  
- **Separation of Concerns** – Webhook management (registration/deregistration) and payload processing are handled by dedicated utilities within the SDK, often invoked by the Scheduler SDK.  

## Interaction Flow

1. The Scheduler SDK (the orchestrator) determines when a task needs to run (e.g., cron schedule, webhook event, manual trigger).  
2. The Scheduler SDK retrieves the full `taskDefinition` (including source configuration, authentication details, and latest continuation tokens) from its persistent database.  
3. The Scheduler SDK directly instantiates a new instance of the relevant Crawler SDK component (e.g., `new GitCrawlerDataSource(...)`) and invokes its `execute()` method with a comprehensive `initialPayload`.  
4. The invoked Crawler SDK instance performs the data extraction from the external source based only on the provided payload.  
5. Upon completion, the Crawler SDK returns the `IngestionData` and a `GSStatus` (including any updated continuation tokens) back to the Scheduler SDK.  
6. The Scheduler SDK then persists these updated tokens and the task status in its database for future runs.  

This architecture ensures that the Crawler SDK remains lightweight, highly scalable, and resilient, as any instance can process any valid task request without needing prior history or complex state synchronization.

# 4. Project File Structure

The project follows a modular structure, separating concerns into distinct directories for source code, configurations, and build outputs.

```plaintext
.
├── config/                # Configuration files for the application and tasks
│   └── default.yaml        # Main configuration file (logs, tasks loading)
│
├── dist/                  # Compiled JavaScript output (from TypeScript)
│   └── ... (compiled .js files and copied config/events/mappings)
│
├── node_modules/          # Project dependencies
│
├── src/                   # Source code directory
│   ├── datasources/
│   │   ├── types/
│   │   │   ├── axios.ts                  # (Internal/Helper) Axios types
│   │   │   ├── git-crawler.ts            # Git Crawler implementation
│   │   │   ├── googledrive-crawler.ts    # Google Drive Crawler implementation
│   │   │   ├── http-crawler.ts           # HTTP Crawler implementation
│   │   │   └── teams-chat-crawler.ts     # (If Teams Chat Crawler is included)
│   │   ├── git-crawler.yaml              # Example config for Git Crawler (reference)
│   │   ├── googledrive-crawler.yaml      # Example config for GDrive Crawler (reference)
│   │   ├── http-crawler.yaml             # Example config for HTTP Crawler (reference)
│   │   └── teams-chat-crawler.yaml       # Example config for Teams Crawler (reference)
│   │
│   ├── events/                 # Godspeed event definitions
│   │   ├── cron-test.yaml        # Cron event definition
│   │   ├── gdrive-event.yaml     # GDrive webhook event definition
│   │   └── git_events.yaml       # GitHub webhook event definition
│   │
│   ├── eventsources/           # Godspeed event source types
│   │   └── types/
│   │       ├── cron.yaml           # Cron event source configuration
│   │       └── http.yaml           # HTTP event source configuration
│   │
│   ├── functions/
│   │   ├── Crawler/              # Webhook processing and external API utilities for Crawlers
│   │   │   ├── data-source-api-utils.ts       # Dispatches webhook API calls
│   │   │   ├── gdrive-api-utils.ts            # Google Drive webhook API calls
│   │   │   ├── git-api-utils.ts               # GitHub webhook API calls
│   │   │   └── processWebhookRequest.ts       # Central webhook payload processing
│   │   │
│   │   ├── my_bank_api/          # (Example/Placeholder for custom API functions)
│   │   │
│   │   ├── Scheduler/            # Core Scheduler SDK components
│   │   │   ├── FileSystemDestinationAdapter.ts      # Destination for saving crawled data locally
│   │   │   ├── GlobalIngestionLifecycleManager.ts   # Main Scheduler logic
│   │   │   ├── interfaces.ts                      # Shared interfaces (tasks, webhooks, data)
│   │   │   └── orchestrator.ts                    # Orchestrates single task execution
│   │   │
│   │   ├── TEST/                 # **Test/Simulation specific functions**
│   │   │   ├── final-test.ts                        # Main setup/simulation for Scheduler & Crawlers
│   │   │   ├── triggerIngestionManagerCronTasks.ts  # Handles cron-triggered task executions
│   │   │   └── triggerIngestionManagerWebhookTasks.ts # Handles webhook-triggered task executions
│   │   │
│   │   ├── Transformers/         # Data transformation functions
│   │   │   ├── code-metadata-extractor-transformer.ts
│   │   │   ├── gdrive-content-normalizer-transformer.ts
│   │   │   ├── generic-ingestion-preprocessor.ts
│   │   │   ├── html-to-plaintext-transformer.ts
│   │   │   ├── passthrough-transformer.ts
│   │   │   └── teams-message-cleaner-transformer.ts
│   │   │
│   │   └── validations/          # (Briefly: Request/Response Schema Validations)
│   │
│   └── index.ts                # Application entry point
│
├── .env                     # Environment variables (e.g., API keys, ngrok URLs)
├── package.json             # Project metadata and scripts
├── pnpm-lock.yaml           # Dependency lock file
├── tsconfig.json            # TypeScript compiler configuration

# 5. Setup & Usage Guide

This section provides a comprehensive, step-by-step guide to setting up and using your Crawler SDK and Scheduler SDK (Simulated). It covers dependency installation, environment configuration, task definition, and running tasks via various triggers, along with debugging tips.

## 5.1. Files Involved in Setup

The core files you'll interact with for setup and triggering are:

- **src/final-test.ts** – Main application setup file. Initializes the Scheduler SDK, registers default crawlers and destinations, loads tasks from configuration, and sets up event listeners.
- **src/functions/triggerIngestionManagerCronTasks.ts** – Acts as the Godspeed event handler for cron triggers. Invoked when a cron event is due.
- **src/functions/triggerIngestionManagerWebhookTasks.ts** – Serves as the Godspeed event handler for webhook triggers. Invoked when an external webhook notification is received.
- **config/events/cron-test.yaml** – Defines a cron event source, specifying the schedule for `triggerIngestionManagerCronTasks.ts`.
- **config/events/gdrive-event.yaml** – Defines the Google Drive webhook event source and endpoint for `triggerIngestionManagerWebhookTasks.ts`.
- **config/events/git_events.yaml** – Defines the GitHub webhook event source and endpoint for `triggerIngestionManagerWebhookTasks.ts`.

## 5.2. Installation

To use the Crawler SDK, you need to set up your Node.js environment and install the required dependencies.

### 5.2.1. Prerequisites

- Node.js (LTS version recommended)
- npm (Node Package Manager) or pnpm

### 5.2.2. Dependency Installation

Navigate to your project's root directory (where `package.json` is located) in your terminal and install the dependencies:

```bash
# Install runtime dependencies
npm install @godspeedsystems/core simple-git axios uuid googleapis google-auth-library cheerio
**Note on `@types` packages:**  
Some libraries (like `simple-git`, `googleapis`, and `google-auth-library`) bundle their TypeScript definitions directly, so explicit `@types/` packages for them might not be necessary. In some cases, installing them can cause conflicts (such as *E404 errors* during install).  

The command above includes only the `@types` packages that are typically required.

### 5.3. Define Tasks (`config/default.yaml`)

This section explains how to define ingestion tasks for the **Scheduler SDK**.  
Tasks are configured directly within your `config/default.yaml` file, serving as the central point for specifying their **source**, **destination**, and **trigger mechanisms**.

**Important:**  
Sensitive information such as API keys, service account paths, and webhook URLs **should NOT** be hardcoded directly in this YAML file.  
Instead, their values should be dynamically injected from environment variables (loaded via your `.env` file) by the `src/final-test.ts` file during application startup.  

This approach ensures:
- **Security** — credentials are never stored in source control.  
- **Flexibility** — you can easily switch configurations for different environments (development, staging, production).

### 5.3.1. Git Tasks (Cron + Webhook)

This subsection details the structure for defining tasks that interact with Git repositories, supporting both cron-based scheduling for full scans and webhook-based triggers for incremental updates.

---

#### Key Configuration Fields

- **id**: A unique identifier for this specific task (e.g., `webhook-git-clone`, `cron-git-clone`). This ID is used by the Scheduler SDK for tracking and management.
- **name**: A human-readable display name for the task. *(optional)*
- **enabled**: A boolean (`true`/`false`) to activate or deactivate the task. Only enabled tasks are considered for execution.
- **source**: Defines the data source for this task.
  - **pluginType**: Must be `git-crawler`. This tells the Scheduler SDK to use the Git Crawler component.
  - **config**: Contains specific settings for the Git Crawler:
    - **repoUrl**: The HTTPS URL of the Git repository to be crawled (e.g., `https://github.com/your-org/your-repo`). *(Mandatory)*
    - **branch**: The specific Git branch to track (e.g., `main`). Defaults to `main` if not specified.
    - **depth**: The depth of the Git clone (e.g., `1` for a shallow clone). Defaults to `1`.
- **destination**: Defines where the ingested data will be stored.
  - **pluginType**: Must be `file-system-destination`.
  - **config**: Contains settings for the destination, e.g., `outputPath: ./crawled_output/webhook-git`.
- **Optional Destination**:  
  If the `destination` block is omitted, the Scheduler SDK will not store the data locally. Instead, it will emit a `DATA_PROCESSED` event after the transformation stage.  
  This is useful for pipelines where data is consumed directly from events.
- **trigger**: Specifies how the task is initiated.
  - **type**: Can be `cron` or `webhook`.
  - **expression** (for cron type): A cron expression (e.g., `* * * * *` for every minute) defining the schedule.
  - **endpointId** (for webhook type): The local API endpoint that listens for incoming GitHub webhook notifications.  
    This must be `/webhook/github/` as per internal system logic.
  - **callbackurl** (for webhook type): The publicly accessible URL that you register with GitHub.  
    - If local, use a tool like ngrok (e.g., `https://your-ngrok-url.ngrok-free.app/api/v1/webhook/github/`).
    - If deployed, use your server’s public URL.
  - **credentials** (for webhook type): The GitHub Personal Access Token (PAT) used for authenticating webhook registration and Git operations.
- **currentStatus**: The initial status of the task when loaded, typically `SCHEDULED`.

---

#### Obtaining a GitHub Personal Access Token (PAT) 🔑

A PAT is required for your application to interact with GitHub's API (e.g., to register webhooks or clone private repositories).

1. **Log in to GitHub**: Go to [github.com](https://github.com) and log in.
2. **Navigate to Developer Settings**:  
   `Profile Picture` → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
3. **Generate New Token**:  
   Click **Generate new token** → **Generate new token (classic)**.
4. **Configure Token**:
   - **Name**: Give your token a descriptive name (e.g., `crawler-sdk-webhook`).
   - **Expiration**: Set an expiration date (e.g., `90 days`).
   - **Scopes**:  
     - Select **`admin:repo_hook`** (necessary for webhook creation/management).  
     - If private repo, also select **`repo`**.
5. **Generate Token**: Click **Generate token**.
6. **Copy Token**: Copy immediately — it will not be shown again. Store it securely (e.g., in your `.env` file).

---

#### Example `config/default.yaml` Snippet for Git Tasks

```yaml
# config/default.yaml

tasks:
  webhook-git-clone:
    id: webhook-git-clone
    name: Git Repository Webhook Clone
    enabled: true # Set to true to enable this task
    source:
      pluginType: git-crawler
      config:
        repoUrl: https://github.com/your-org/your-repo # Example repository URL
        branch: main
        depth: 1
    destination:
      pluginType: file-system-destination
      config: { outputPath: './crawled_output/webhook-git' }
    trigger:
      type: webhook
      # Values will be injected from process.env by final-test.ts
      credentials: "your_github_pat_placeholder" # From process.env.GITHUB_PAT
      endpointId: '/webhook/github/' # MANDATORY
      callbackurl: 'https://your-ngrok-url.ngrok-free.app/api/v1/webhook/github/' # From process.env.NGROK_GITHUB_WEBHOOK_URL or deployment URL
    currentStatus: SCHEDULED

  cron-git-clone:
    id: cron-git-clone
    name: Git Repository Cron Clone
    enabled: false # Set to true to enable this task
    source:
      pluginType: git-crawler
      config:
        repoUrl: https://github.com/your-org/your-repo
        branch: main
        depth: 1
    destination:
      pluginType: file-system-destination
      config: { outputPath: './crawled_output/cron-git' }
    trigger:
      type: cron
      expression: '* * * * *' # Every minute
    currentStatus: SCHEDULED

5.3.2. Google Drive Tasks (Cron + Webhook)

This subsection outlines the configuration for tasks designed to ingest data from Google Drive, supporting both scheduled full scans and real-time updates via webhooks.

Key Configuration Fields

id: A unique identifier for the task (e.g., my-cron-google-drive-crawl-task, my-webhook-google-drive-crawl-task).

name: A human-readable display name for the task.

enabled: Boolean (true/false) to activate or deactivate the task.

source: Defines the data source for this task.

pluginType: Must be googledrive-crawler.

config:

folderId: The ID of the Google Drive folder to crawl (mandatory).

authType: Must be service_account.

serviceAccountKeyPath: Path to your Google Service Account JSON key file.

userToImpersonateEmail: Email address of the user whose Google Drive account the service account will impersonate.

pageSize: Number of files retrieved per page (default: 100).

destination: Defines where ingested data will be stored.

pluginType: Must be file-system-destination.

config: Example: outputPath: ./crawled_output/cron-gdrive-data.

Optional Destination: If omitted, the Scheduler SDK emits a DATA_PROCESSED event without local storage.

trigger: Specifies how the task is initiated.

type: cron or webhook.

expression (for cron): Cron schedule (e.g., * * * * *).

endpointId (for webhook): Must be /webhook/gdrive/.

callbackurl (for webhook): Public HTTPS URL for Google Drive webhook registration.

Local: use ngrok (e.g., https://your-ngrok-url.ngrok-free.app/api/v1/webhook/gdrive/)

Server: use deployment public URL.

credentials (for webhook): Includes serviceAccountKeyPath or key string for authentication.

currentStatus: Initial status, typically SCHEDULED.

Obtaining Google Drive Credentials and Folder ID 🔑

Create a Google Service Account Key:

Go to Google Cloud Console.

Select your project (or create one).

Navigate: IAM & Admin → Service Accounts.

Click + CREATE SERVICE ACCOUNT and follow prompts.

Step 2: Assign a role (e.g., Viewer, Editor, or Drive-specific roles).

Step 3: + CREATE KEY → Choose JSON → CREATE.
A JSON key file is downloaded — store securely.

Enable Google Drive API:

In Google Cloud Console:
APIs & Services → Enabled APIs & services → + ENABLE APIS AND SERVICES.

Search Google Drive API → Click → ENABLE.

Get the Service Account Email:

Found in your Service Account details.

Example: my-service-account@your-project-id.iam.gserviceaccount.com.

Get the Google Drive Folder ID:

Open target Google Drive folder in browser.

URL format: https://drive.google.com/drive/folders/<FOLDER_ID>.

Share the Folder with Service Account:

Right-click folder → Share → Paste Service Account email.

Assign Viewer or Editor access.

Example config/default.yaml Snippet
# config/default.yaml

# ... (other global configs)

tasks:
  # ... (Git Tasks)

  my-cron-google-drive-crawl-task:
    id: my-cron-google-drive-crawl-task
    name: Google Drive Data Ingestion (Cron)
    enabled: true
    source:
      pluginType: googledrive-crawler
      config:
        serviceAccountKeyPath: "" 
        userToImpersonateEmail: ""
        folderId: ""
    destination:
      pluginType: file-system-destination
      config: { outputPath: './crawled_output/cron-gdrive-data' }
    trigger:
      type: cron
      expression: '* * * * *'
    currentStatus: SCHEDULED

  my-webhook-google-drive-crawl-task:
    id: my-webhook-google-drive-crawl-task
    name: Google Drive Data Ingestion (Webhook)
    enabled: false
    source:
      pluginType: googledrive-crawler
      config:
        serviceAccountKeyPath: "" 
        userToImpersonateEmail: ""
        folderId: ""
    destination:
      pluginType: file-system-destination
      config: { outputPath: './crawled_output/webhook-gdrive-data' }
    trigger:
      type: webhook
      endpointId: '/webhook/gdrive/'
      callbackurl: 'https://your-ngrok-url.ngrok-free.app/api/v1/webhook/gdrive/'
      credentials: { serviceAccountKeyPath: "" }
    currentStatus: SCHEDULED

