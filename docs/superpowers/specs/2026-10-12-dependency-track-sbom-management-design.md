# Dependency-Track 기반 SBOM 관리 및 CI/CD 파이프라인 설계

- **작성일**: 2026-10-12
- **상태**: Review Pending
- **대상 경로**: `devops/dtrack/`

---

## 1. 개요 및 목적 (Overview & Goals)

본 문서는 Proxmox Homelab 환경에서 OWASP Dependency-Track을 Docker Compose 기반으로 구축하고, Maven 기반 Spring Boot 및 Node.js(JavaScript/TypeScript) 프로젝트의 SBOM(Software Bill of Materials) 생성 및 수집/분석을 자동화하기 위한 아키텍처 및 파이프라인 설계를 정의한다.

### 주요 목표
1. **Dependency-Track 스택 구축**: API Server, Web Frontend, PostgreSQL 데이터베이스를 격리된 Docker Compose 스택으로 구성.
2. **Nginx Proxy Manager(NPM) 연동**: 단일 공용 도메인(`dtrack.club012.com`)을 통해 웹 UI(`/`) 및 REST API(`/api/`)를 통합 서비스.
3. **포트 환경변수화**: Frontend 기본 포트 `8081`, Backend API Server 기본 포트 `8080`을 설정하되, `.env`를 통해 유연하게 변경 가능하도록 구현.
4. **프로젝트별 SBOM 생성 표준화**:
   - Java / Spring Boot (Maven): `cyclonedx-maven-plugin` 기반 CycloneDX JSON 생성
   - Node.js (JS/TS): `@cyclonedx/cyclonedx-npm` 기반 CycloneDX JSON 생성
5. **CI/CD 자동화**: GitLab CI 및 Jenkins 파이프라인에서 빌드 시 SBOM을 자동 생성하고 Dependency-Track API로 업로드.

---

## 2. 시스템 아키텍처 및 인프라 설계

### 2.1 네트워크 및 라우팅 토폴로지

```
                    [ 사용자 브라우저 / CI/CD Runner ]
                                    │
                     https://dtrack.club012.com
                                    ▼
                     [ Nginx Proxy Manager (NPM) ]
                                    │
          ┌─────────────────────────┴─────────────────────────┐
          │ (Path: /)                                         │ (Path: /api/)
          ▼                                                   ▼
┌───────────────────┐                               ┌───────────────────┐
│  dtrack-frontend  │                               │ dtrack-apiserver  │
│  (Host: ${PORT_   │                               │  (Host: ${PORT_   │
│   FRONTEND:-8081})│                               │   BACKEND:-8080}) │
└───────────────────┘                               └─────────┬─────────┘
                                                              │
                                                              ▼
                                                    ┌───────────────────┐
                                                    │     dtrack-db     │
                                                    │   (Postgres 16)   │
                                                    └───────────────────┘
```

### 2.2 Nginx Proxy Manager (NPM) 세부 설정

- **도메인**: `dtrack.club012.com`
- **SSL**: Let's Encrypt 또는 홈랩 내부 CA 인증서 적용 (`Force SSL`, `HTTP/2 Support`, `Websockets Support` 활성화)
- **메인 프록시 규칙 (Details)**:
  - Forward Hostname / IP: `<Docker_Host_IP>`
  - Forward Port: `8081` (Frontend)
- **Custom Location 규칙**:
  - Location: `/api/`
  - Forward Hostname / IP: `<Docker_Host_IP>`
  - Forward Port: `8080` (API Server)
  - Forward Path: `/api/` (API Server의 엔드포인트 `/api/v1/...` 경로 유지)
  - Client Max Body Size: `100M` (대용량 SBOM 파일 업로드 허용)

### 2.3 디렉터리 구조 (`devops/dtrack/`)

```
devops/dtrack/
├── .env.example
├── .gitignore
├── compose.yaml
├── README.md
└── samples/
    ├── maven-pom.xml.snippet
    ├── package.json.snippet
    ├── .gitlab-ci-maven.yml
    ├── .gitlab-ci-node.yml
    ├── Jenkinsfile-maven
    └── Jenkinsfile-node
```

### 2.4 컨테이너 구성 상세

1. **`dtrack-db` (PostgreSQL 16-alpine)**
   - 이미지: `postgres:16-alpine`
   - 환경변수: `POSTGRES_DB=${POSTGRES_DB:-dtrack}`, `POSTGRES_USER=${POSTGRES_USER:-dtrack}`, `POSTGRES_PASSWORD=${POSTGRES_PASSWORD}`
   - 볼륨: `./volumes/db_data:/var/lib/postgresql/data`
   - 재시작 정책: `unless-stopped`

2. **`dtrack-apiserver` (API Server & Analysis Engine)**
   - 이미지: `dependencytrack/apiserver:${DTRACK_VERSION:-4.11.7}`
   - 의존성: `dtrack-db` 헬스체크 통과 후 실행
   - 환경변수:
     - `ALPINE_DATABASE_MODE=external`
     - `ALPINE_DATABASE_URL=jdbc:postgresql://dtrack-db:5432/${POSTGRES_DB:-dtrack}`
     - `ALPINE_DATABASE_DRIVER=org.postgresql.Driver`
     - `ALPINE_DATABASE_USERNAME=${POSTGRES_USER:-dtrack}`
     - `ALPINE_DATABASE_PASSWORD=${POSTGRES_PASSWORD}`
     - `EXTRA_JVM_ARGUMENTS=-Xms2g -Xmx4g` (취약점 DB 동기화 및 그래프 분석을 위한 힙 메모리 할당)
   - 포트 매핑: `${PORT_BACKEND:-8080}:8080`
   - 볼륨: `./volumes/apiserver_data:/data`

3. **`dtrack-frontend` (Web UI)**
   - 이미지: `dependencytrack/frontend:${DTRACK_VERSION:-4.11.7}`
   - 환경변수:
     - `API_BASE_URL=https://dtrack.club012.com/api` (브라우저에서 호출하는 공용 API 엔드포인트)
   - 포트 매핑: `${PORT_FRONTEND:-8081}:8080`

---

## 3. 프로젝트별 SBOM 생성 사양

### 3.1 Java / Spring Boot (Maven)
- **도구**: `org.cyclonedx:cyclonedx-maven-plugin` (v2.8.0 이상 권장)
- **출력 포맷**: CycloneDX JSON v1.5 (`target/bom.json`)
- **생성 골(Goal)**: `makeAggregateBom` (멀티 모듈 및 단일 모듈 공용)
- **실행 명령**:
  ```bash
  mvn clean package -DskipTests
  # 또는 플러그인 설정 없이 ad-hoc 실행 시:
  mvn org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom -DoutputFormat=json -DoutputName=bom
  ```

### 3.2 JavaScript / TypeScript (Node.js)
- **도구**: `@cyclonedx/cyclonedx-npm` (v1.19.0 이상 권장)
- **출력 포맷**: CycloneDX JSON (`bom.json`)
- **의존성 정책**: `--omit dev` (프로덕션 런타임 의존성 위주 분석)
- **실행 명령**:
  ```bash
  npm ci
  npx @cyclonedx/cyclonedx-npm --output-file bom.json --output-format JSON --omit dev
  ```

---

## 4. CI/CD 파이프라인 연동 설계

### 4.1 Dependency-Track 인증 및 업로드 엔드포인트
- **엔드포인트**: `POST https://dtrack.club012.com/api/v1/bom`
- **인증 헤더**: `X-Api-Key: ${DTRACK_API_KEY}`
- **전송 방식**: `multipart/form-data`
- **요청 필드**:
  - `projectName`: 식별용 프로젝트명 (예: GitLab `${CI_PROJECT_NAME}`, Jenkins `${JOB_BASE_NAME}`)
  - `projectVersion`: 식별용 버전 (예: `${CI_COMMIT_REF_NAME}`, `${BUILD_NUMBER}`)
  - `autoCreate`: `true` (해당 프로젝트가 Dependency-Track에 없으면 자동 생성)
  - `bom`: SBOM 파일 바이너리 (`@bom.json`)

### 4.2 GitLab CI 파이프라인 흐름
1. `build` 스테이지:
   - 언어별 런타임 이미지(Maven/Node)에서 의존성 설치 및 `bom.json` 생성
   - `artifacts`에 `bom.json` 보관
2. `sbom` 스테이지:
   - `curlimages/curl` 또는 경량 이미지에서 `POST /api/v1/bom` 호출
   - CI/CD Variable (`DTRACK_API_URL`, `DTRACK_API_KEY`) 활용

### 4.3 Jenkins 파이프라인 흐름
1. Jenkins Declarative Pipeline 구성
2. Jenkins Credentials (`Secret text` 형식으로 API Key 관리) 주입
3. 빌드 및 SBOM 생성 스테이지 수행
4. 전송 방식:
   - **기본 Curl 방식**: 추가 플러그인 없이 범용적 실행
   - **플러그인 방식**: `Dependency-Track Plugin`을 이용한 동기식 검증 및 임계치 검사 가이드 포함

---

## 5. 보안 및 운영 관리 (Security & Operation)

1. **초기 관리자 비밀번호 변경**: 최초 로그인 시 기본 계정(`admin` / `admin`) 비밀번호를 즉시 변경.
2. **전용 팀 및 API Key 발급**:
   - Dependency-Track 관리자 메뉴 -> `Access Management` -> `Teams` 생성 (예: `CI-Automation`).
   - 권한: `BOM_UPLOAD`, `PROJECT_CREATION_UPLOAD`, `VIEW_PORTFOLIO` 부여.
   - API Key를 발급하여 CI/CD 저장소의 Secret으로 등록.
3. **데이터 백업**:
   - `volumes/db_data` (PostgreSQL DB) 주기적 백업 (pg_dump 권장).
   - `volumes/apiserver_data` (NIST NVD 미러 및 Lucene 인덱스).

---

## 6. 검증 계획 (Verification Plan)

1. **Docker Compose 기동 검증**:
   - `docker compose -f devops/dtrack/compose.yaml up -d` 실행
   - DB, API Server, Frontend 컨테이너 정상 실행 및 헬스체크 확인
2. **NPM 프록시 및 Web UI 접근 검증**:
   - `https://dtrack.club012.com` 접속 시 정상적인 로그인 화면 렌더링 확인
   - 브라우저 개발자 도구에서 `/api/v1/...` 호출 시 200 OK 응답 확인
3. **SBOM 수동 전송 테스트**:
   - 샘플 bom.json을 curl로 업로드하여 Dependency-Track 프로젝트 목록에 자동 등록 및 의존성 분석 확인
4. **CI/CD 샘플 스크립트 문법 검증**:
   - 제공되는 샘플 파일들의 문법 검사 및 README 안내 매뉴얼 정합성 확인
