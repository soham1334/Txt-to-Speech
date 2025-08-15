# Crawler SDK

A robust and extensible SDK for ingesting data from various external sources such as Git repositories, Google Drive, and HTTP endpoints.  
Designed to be stateless, the crawlers integrate seamlessly with an external **Scheduler SDK** for orchestration and task management.

---

## Table of Contents
1. [Introduction](#1-introduction)  
2. [Features](#2-features)  
3. [Architecture Overview](#3-architecture-overview)  
4. [Installation](#4-installation)  
    - [4.1. Prerequisites](#41-prerequisites)  
    - [4.2. Dependency Installation](#42-dependency-installation)  
5. [Usage](#5-usage)  
    - [5.1. General Concept](#51-general-concept)  
    - [5.2. Configuration](#52-configuration)  
    - [5.3. Output Format (IngestionData)](#53-output-format-ingestiondata)  
6. [Crawler Types](#6-crawler-types)  
    - [6.1. Git Crawler](#61-git-crawler-git-crawlerts)  
    - [6.2. Google Drive Crawler](#62-google-drive-crawler-gdrive-crawlerts)  
    - [6.3. HTTP Crawler](#63-http-crawler-http-crawlerts)  
7. [Webhook Processing & Utilities](#7-webhook-processing--utilities)  
8. [Testing](#8-testing)  
9. [License](#9-license)  

---

## 1. Introduction
The **Crawler SDK** is a collection of specialized data source components built with TypeScript and Node.js.  
Its primary function is to efficiently extract and standardize data from diverse external systems.  

**Key points:**
- Stateless by design — invoked by an external Scheduler SDK
- Modular, testable, and extensible for integration into ingestion pipelines

---

## 2. Features
- **Multi-Source Data Ingestion** – Git, Google Drive, HTTP(S)  
- **Stateless Design** – High scalability and resilience  
- **Full Scans & Incremental Syncs** (delta sync with continuation tokens)  
- **Webhook-Driven Triggers** – GitHub push, Google Drive changes  
- **Flexible Configuration** – YAML + JSON payloads  
- **Standardized Output Format** – `IngestionData`  
- **Temporary Local Storage** – Cleaned after execution  
- **Robust Logging & Error Handling**

---

## 3. Architecture Overview
The Crawler SDK operates as **stateless workers** in a larger orchestrated system.

### Key Principles
- **Statelessness** – No persistent internal state
- **Single Responsibility** – Each crawler handles one source type
- **Separation of Concerns** – Webhook management in utilities

### Interaction Flow
1. Scheduler SDK decides when to run a task (cron/webhook/manual)  
2. Scheduler SDK fetches the full `taskDefinition` from DB  
3. Instantiates the correct Crawler SDK component  
4. Calls `execute()` with payload  
5. Crawler fetches & returns `IngestionData` + updated tokens  
6. Scheduler SDK persists status/tokens for next run  

---

## 4. Installation

### 4.1. Prerequisites
- Node.js (LTS recommended)  
- npm or pnpm

### 4.2. Dependency Installation
```bash
# Runtime dependencies
npm install @godspeedsystems/core simple-git axios uuid googleapis google-auth-library cheerio

# Dev dependencies
npm install --save-dev jest ts-jest @types/jest ts-node @types/axios @types/cheerio
```
## 5. Usage

### 5.1. General Concept
```ts
const crawlerInstance = new DataSource({ config: taskDefinition.source.config });
await crawlerInstance.initClient();
const result = await crawlerInstance.execute(context, initialPayload);
```
### 5.2. Configuration

**Example: Git Crawler** (`config/datasources/git-crawler.yaml`)
```yaml
type: git-crawler
branch: main
depth: 1
```
**Example: Google Drive Crawler** (`config/datasources/gdrive-crawler.yaml`)
```yaml
type: googledrive-crawler
authType: service_account
serviceAccountKeyPath: ./path/to/service-account-key.json
pageSize: 100
```
**Example: HTTP Crawler** (`config/datasources/http-crawler.yaml`)
```yaml
type: http-crawler
maxDepth: 2
recursiveCrawling: true
allowedDomains:
  - example.com
```
### 5.3. Output Format (IngestionData)
```typescript
export interface IngestionData {
  id: string;
  content: string | Buffer | object;
  metadata?: {
    filename?: string;
    relativePath?: string;
    changeType?: 'UPSERT' | 'DELETE' | 'UNKNOWN' | 'full_scan';
  };
  fetchedAt?: Date;
  url?: string;
  statusCode?: number;
}
```
## 6. Crawler Types

### 6.1. Git Crawler (`git-crawler.ts`)
- **Full Scan** – Clone & read all files  
- **Webhook Trigger** – Handle GitHub push payloads  
- **Graceful Handling** – Warnings for missing commits  

### 6.2. Google Drive Crawler (`gdrive-crawler.ts`)
- **Full Scan** – List & fetch all files  
- **Delta Sync** – Uses `startPageToken` for incremental changes  
- **Token Management** – Returns `newStartPageToken`  

### 6.3. HTTP Crawler (`http-crawler.ts`)
- **Recursive Crawling** (if enabled)  
- **Domain & Path Filters**  
- **Content & Metadata Extraction**  

## 7. Webhook Processing & Utilities
- `processWebhookRequest.ts` – Validation & payload extraction  
- `data-source-api-utils.ts` – Dispatch register/deregister calls  
- `gdrive-api-utils.ts` – Google Drive webhook API calls  
- `git-api-utils.ts` – GitHub webhook API calls  

## 8. Testing
- **Framework:** Jest + ts-jest  
- **Mocking:** All external deps mocked  
- **Structure:** `test/` mirrors `src/`  

**Run tests:**
```bash
npm test
```
## 9. License

This project is licensed under the ISC License – see [LICENSE](LICENSE) for details.
