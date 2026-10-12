# Dependency-Track SBOM Management Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Proxmox Homelab 환경의 `devops/dtrack/` 디렉터리에 Docker Compose 기반 OWASP Dependency-Track 스택을 구성하고, Maven Spring Boot 및 Node.js 프로젝트를 위한 SBOM 생성 스니펫과 GitLab CI / Jenkins 파이프라인 연동 샘플을 구축한다.

**Architecture:** Docker Compose 기반의 3계층(PostgreSQL 16, API Server, Web Frontend) 구조를 배포하며, 외부 Nginx Proxy Manager(NPM)를 통해 단일 도메인 `dtrack.club012.com`의 `/`는 Frontend(기본 포트 8081), `/api/`는 Backend API Server(기본 포트 8080)로 역방향 프록시 라우팅한다. 프로젝트 빌드 단계에서 CycloneDX JSON 형식의 SBOM을 생성하여 REST API(`/api/v1/bom`)로 전송하는 CI/CD 템플릿을 제공한다.

**Tech Stack:** Docker Compose, PostgreSQL 16, OWASP Dependency-Track (v4.11.x), CycloneDX Maven Plugin, @cyclonedx/cyclonedx-npm, GitLab CI, Jenkins (Declarative Pipeline)

**Spec:** `docs/superpowers/specs/2026-10-12-dependency-track-sbom-management-design.md`

## Global Constraints

- 스택 디렉터리 위치: `devops/dtrack/`
- Frontend 외부 노출 포트 환경변수: `PORT_FRONTEND` (기본값: `8081`)
- Backend API Server 외부 노출 포트 환경변수: `PORT_BACKEND` (기본값: `8080`)
- Frontend의 `API_BASE_URL`: `https://dtrack.club012.com/api`
- SBOM 포맷: CycloneDX JSON v1.5 / v1.6
- API 서버 메모리: `-Xms2g -Xmx4g`
- NPM 프록시 라우팅: 루트 `/` -> Frontend(`8081`), 커스텀 로케이션 `/api/` -> Backend(`8080`)

## Review Focus

1. **대용량 SBOM 업로드 시 프록시 제한 (HTTP 413 Payload Too Large)**: NPM Custom Location `/api/`에 `client_max_body_size 100M;` 설정이 안내되어야 함.
2. **API 경로 프리픽스 유실 (HTTP 404)**: NPM Custom Location `/api/`가 백엔드로 포워딩될 때 `/api/` 경로가 유지되어 백엔드의 `/api/v1/...` 엔드포인트와 일치해야 함.
3. **API Key 권한 부족 (HTTP 403 Forbidden)**: CI 파이프라인에서 사용하는 API Key에 `BOM_UPLOAD` 및 `PROJECT_CREATION_UPLOAD` 권한이 필수적임을 명시해야 함.
4. **PostgreSQL 데이터베이스 헬스체크 대기**: API 서버가 DB 준비 전에 부팅되어 실패하지 않도록 `depends_on`에 `condition: service_healthy` 적용.
5. **Node.js 개발 의존성 오염**: Node.js SBOM 추출 시 배포 런타임에 포함되지 않는 devDependencies가 취약점 경고를 유발하지 않도록 `--omit dev` 플래그 적용.

---

### Task 1: Setup Dependency-Track Docker Compose Stack & Environment Configuration

**Files:**
- Create: `devops/dtrack/.env.example`
- Create: `devops/dtrack/.gitignore`
- Create: `devops/dtrack/compose.yaml`

**Interfaces:**
- Consumes: 호스트 환경변수 (`PORT_FRONTEND`, `PORT_BACKEND`, `POSTGRES_PASSWORD`, etc.)
- Produces: Docker Compose 스택 (`dtrack-db`, `dtrack-apiserver`, `dtrack-frontend`)

- [ ] **Step 1: Create `devops/dtrack/.gitignore`**
  Ignore local `.env`, volumes directory, and runtime data:
  ```gitignore
  .env
  volumes/
  ```

- [ ] **Step 2: Create `devops/dtrack/.env.example`**
  Define default configurable parameters:
  - `DTRACK_VERSION=4.11.7`
  - `PORT_FRONTEND=8081`
  - `PORT_BACKEND=8080`
  - `POSTGRES_DB=dtrack`
  - `POSTGRES_USER=dtrack`
  - `POSTGRES_PASSWORD=ChangeThisPassword123!`
  - `API_BASE_URL=https://dtrack.club012.com/api`

- [ ] **Step 3: Create `devops/dtrack/compose.yaml`**
  Define services:
  - `dtrack-db`: `postgres:16-alpine`, healthcheck with `pg_isready`, persistent volume `./volumes/db_data:/var/lib/postgresql/data`.
  - `dtrack-apiserver`: `dependencytrack/apiserver:${DTRACK_VERSION:-4.11.7}`, depends_on `dtrack-db` (healthy), `EXTRA_JVM_ARGUMENTS=-Xms2g -Xmx4g`, ports `${PORT_BACKEND:-8080}:8080`, volume `./volumes/apiserver_data:/data`.
  - `dtrack-frontend`: `dependencytrack/frontend:${DTRACK_VERSION:-4.11.7}`, environment `API_BASE_URL=${API_BASE_URL:-https://dtrack.club012.com/api}`, ports `${PORT_FRONTEND:-8081}:8080`.

- [ ] **Step 4: Verify Docker Compose syntax**
  Run: `docker compose -f devops/dtrack/compose.yaml config`
  Expected: Valid YAML output without configuration errors.

- [ ] **Step 5: Commit**
  ```bash
  git add devops/dtrack/.gitignore devops/dtrack/.env.example devops/dtrack/compose.yaml
  git commit -m "feat(dtrack): add docker compose stack and environment configuration"
  ```

---

### Task 2: Create Project SBOM Generation Snippets (Maven & Node.js)

**Files:**
- Create: `devops/dtrack/samples/maven-pom.xml.snippet`
- Create: `devops/dtrack/samples/package.json.snippet`

**Interfaces:**
- Consumes: Maven pom.xml 및 Node.js package.json
- Produces: `target/bom.json` (Maven) 및 `bom.json` (Node.js)

- [ ] **Step 1: Create `devops/dtrack/samples/maven-pom.xml.snippet`**
  Provide complete XML block for `cyclonedx-maven-plugin` (v2.8.0), goal `makeAggregateBom`, schemaVersion `1.5`, outputFormat `json`, outputName `bom`.
  Include comments explaining single-module (`makeBom`) vs multi-module (`makeAggregateBom`) usage.

- [ ] **Step 2: Create `devops/dtrack/samples/package.json.snippet`**
  Provide JSON snippet demonstrating:
  - `"scripts": { "sbom": "cyclonedx-npm --output-file bom.json --output-format JSON --omit dev" }`
  - `"devDependencies": { "@cyclonedx/cyclonedx-npm": "^1.19.0" }`
  - Alternative one-line `npx` command description.

- [ ] **Step 3: Validate snippet formatting**
  Verify XML well-formedness and JSON validity of snippets.

- [ ] **Step 4: Commit**
  ```bash
  git add devops/dtrack/samples/maven-pom.xml.snippet devops/dtrack/samples/package.json.snippet
  git commit -m "feat(dtrack): add Maven and Node.js SBOM generation snippets"
  ```

---

### Task 3: Create CI/CD Pipeline Samples for GitLab CI & Jenkins

**Files:**
- Create: `devops/dtrack/samples/.gitlab-ci-maven.yml`
- Create: `devops/dtrack/samples/.gitlab-ci-node.yml`
- Create: `devops/dtrack/samples/Jenkinsfile-maven`
- Create: `devops/dtrack/samples/Jenkinsfile-node`

**Interfaces:**
- Consumes: `bom.json`, `DTRACK_API_URL` (`https://dtrack.club012.com/api`), `DTRACK_API_KEY`
- Produces: SBOM 업로드 HTTP 요청 (`POST ${DTRACK_API_URL}/v1/bom` with multipart payload)

- [ ] **Step 1: Create `devops/dtrack/samples/.gitlab-ci-maven.yml`**
  Define multi-stage pipeline (`build`, `sbom`):
  - Maven build & `target/bom.json` artifact export
  - `curl` stage sending `bom.json` to `${DTRACK_API_URL}/v1/bom` with `autoCreate=true`, `projectName=${CI_PROJECT_NAME}`, `projectVersion=${CI_COMMIT_REF_NAME}`.

- [ ] **Step 2: Create `devops/dtrack/samples/.gitlab-ci-node.yml`**
  Define multi-stage pipeline (`build`, `sbom`):
  - `node:20-alpine` based `npm ci` and `npx @cyclonedx/cyclonedx-npm ...`
  - Extraction of version from `package.json`
  - `curl` stage sending `bom.json` to Dependency-Track.

- [ ] **Step 3: Create `devops/dtrack/samples/Jenkinsfile-maven`**
  Define Declarative Pipeline for Maven:
  - Environment with credentials (`dependency-track-api-key`)
  - Stage: `Build & Generate SBOM` (`mvn clean package -DskipTests`)
  - Stage: `Upload SBOM` using `curl` (with optional `dependencyTrackPublisher` commented guide)
  - Post: archiveArtifacts `target/bom.json`.

- [ ] **Step 4: Create `devops/dtrack/samples/Jenkinsfile-node`**
  Define Declarative Pipeline for Node.js:
  - Stage: `Build & Generate SBOM` (`npx @cyclonedx/cyclonedx-npm`)
  - Stage: `Upload SBOM` using `curl` with extracted version from `package.json`
  - Post: archiveArtifacts `bom.json`.

- [ ] **Step 5: Commit**
  ```bash
  git add devops/dtrack/samples/.gitlab-ci-maven.yml devops/dtrack/samples/.gitlab-ci-node.yml devops/dtrack/samples/Jenkinsfile-maven devops/dtrack/samples/Jenkinsfile-node
  git commit -m "feat(dtrack): add GitLab CI and Jenkins pipeline samples"
  ```

---

### Task 4: Create Comprehensive README and Operations Guide

**Files:**
- Create: `devops/dtrack/README.md`

**Interfaces:**
- Documents: Quickstart, NPM Reverse Proxy Guide, API Key Issue Guide, Usage Instructions.

- [ ] **Step 1: Write `devops/dtrack/README.md`**
  Include:
  1. Service Architecture & Port Overview (Frontend: 8081, Backend: 8080)
  2. Setup & Execution (`cp .env.example .env && docker compose up -d`)
  3. Nginx Proxy Manager (NPM) 상세 설정 가이드:
     - Details: `dtrack.club012.com` -> Host IP : `8081`
     - Custom Locations: `/api/` -> Host IP : `8080`, Forward Path `/api/`, `client_max_body_size 100M;`
  4. 초기 관리자 계정 변경 및 CI 연동용 API Key 생성 절차
  5. Maven 및 Node.js SBOM 생성 및 CI/CD 연동 샘플 파일 사용법
  6. 백업 및 유지관리 가이드 (PostgreSQL 백업 명령어 포함)

- [ ] **Step 2: Commit**
  ```bash
  git add devops/dtrack/README.md
  git commit -m "docs(dtrack): add comprehensive setup and operations guide"
  ```

---

## Plan Self-Review Check

- **Spec coverage:** Compose 스택, .env 환경변수, 포트 8081/8080, NPM 라우팅, Maven/Node.js SBOM 플러그인, GitLab CI/Jenkins 샘플, 운영 가이드 모두 포함됨.
- **Review Focus coverage:** NPM client_max_body_size, /api/ 경로 보존, API Key RBAC, PostgreSQL 헬스체크, --omit dev 플러그인 플래그 모두 태스크별 검증 단계에 반영됨.
- **Independence:** 각 태스크는 명확한 입출력과 파일 범위를 가지며 단계적으로 커밋 가능함.
