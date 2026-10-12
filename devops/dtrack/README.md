# OWASP Dependency-Track (SBOM 관리 시스템)

Proxmox Homelab 환경에서 SBOM(Software Bill of Materials)을 수집, 분석 및 관리하기 위한 OWASP Dependency-Track 구축 환경입니다.  
Maven 기반 Spring Boot 및 Node.js 기반 JavaScript/TypeScript 프로젝트의 취약점(CVE)을 지속적으로 추적할 수 있습니다.

---

## 1. 아키텍처 및 포트 구성

```
                       [ 클라이언트 / 브라우저 / CI/CD ]
                                      │
                         https://dtrack.club012.com
                                      ▼
                      [ Nginx Proxy Manager (NPM) ]
                        │                         │
                 (Path: /)                 (Path: /api/)
                        ▼                         ▼
               ┌─────────────────┐       ┌─────────────────┐
               │ dtrack-frontend │       │ dtrack-apiserver│
               │  (Port: 8081)   │       │  (Port: 8080)   │
               └─────────────────┘       └────────┬────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │    dtrack-db    │
                                         │ (Postgres 16)   │
                                         └─────────────────┘
```

| 서비스 | 컨테이너명 | 호스트 기본 포트 (`.env`) | 설명 |
|---|---|---|---|
| **Web Frontend** | `dtrack-frontend` | `8081` (`PORT_FRONTEND`) | 공식 웹 대시보드 UI |
| **API Server** | `dtrack-apiserver` | `8080` (`PORT_BACKEND`) | REST API 백엔드 및 취약점 분석 엔진 |
| **Database** | `dtrack-db` | 내부 전용 (5432) | PostgreSQL 16 메타데이터/취약점 저장소 |

---

## 2. 빠른 시작 (Quick Start)

### (1) 환경 변수 설정
```bash
cd devops/dtrack
cp .env.example .env
```
필요에 따라 `.env` 파일 내 비밀번호 및 포트 설정을 변경합니다:
```bash
# PostgreSQL 비밀번호 변경 권장
POSTGRES_PASSWORD=MySecurePassword123!

# 포트 변경 시 (기본값: Frontend 8081, Backend 8080)
PORT_FRONTEND=8081
PORT_BACKEND=8080
```

### (2) 서비스 시작
```bash
docker compose up -d
```

### (3) 컨테이너 상태 및 로그 확인
```bash
docker compose ps
docker compose logs -f dtrack-apiserver
```
> **참고**: API 서버는 최초 기동 시 NVD(National Vulnerability Database), GitHub Advisories, OSV 데이터베이스를 동기화하므로 초기 분석 완료까지 10~20분 정도 소요될 수 있습니다.

---

## 3. Nginx Proxy Manager (NPM) 리버스 프록시 설정

단일 도메인(`dtrack.club012.com`)을 통해 프론트엔드와 백엔드 API를 동시에 서빙하도록 설정합니다.

### Step 1: Proxy Host 추가
* **Domain Names**: `dtrack.club012.com`
* **Scheme**: `http`
* **Forward Hostname / IP**: `<도커_호스트_IP>`
* **Forward Port**: `8081` (Frontend 포트)
* **옵션 체크**:
  * [x] `Cache Assets`
  * [x] `Block Common Exploits`
  * [x] `Websockets Support`

### Step 2: SSL 탭 설정
* **SSL Certificate**: 유효한 인증서 선택
* **옵션 체크**:
  * [x] `Force SSL`
  * [x] `HTTP/2 Support`
  * [x] `HSTS Enabled`

### Step 3: Custom Locations 탭 설정 (중요)
API 호출(`/api/*`)을 백엔드 API 서버로 라우팅하고 대용량 SBOM 업로드를 허용합니다.
* **Define location**: `/api/`
* **Scheme**: `http`
* **Forward Hostname / IP**: `<도커_호스트_IP>`
* **Forward Port**: `8080` (Backend API 서버 포트)
* **Gear 아이콘 (고급 Nginx 지시문)**:
  ```nginx
  client_max_body_size 100M;
  proxy_read_timeout 300s;
  proxy_connect_timeout 300s;
  proxy_send_timeout 300s;
  ```

---

## 4. 초기 관리자 설정 및 CI 연동용 API Key 발급

1. 브라우저에서 `https://dtrack.club012.com` 접속.
2. 초기 로그인:
   - **Username**: `admin`
   - **Password**: `admin`
3. 최초 로그인 시 강제되는 비밀번호 변경 창에서 새 관리자 비밀번호 설정.
4. **CI/CD 전용 API Key 생성**:
   - 상단 메뉴: **Administration** > **Access Management** > **Teams**
   - **Create Team** 클릭:
     - Name: `CI-Automation`
   - 생성된 팀을 클릭하고 **Permissions** 탭에서 아래 권한 부여:
     - `BOM_UPLOAD` (필수: SBOM 업로드)
     - `PROJECT_CREATION_UPLOAD` (필수: 신규 프로젝트 자동 생성)
     - `VIEW_PORTFOLIO` (선택: 프로젝트 조회)
   - **API Keys** 탭에서 **Generate API Key** 클릭 후 키 복사.
5. 복사한 API Key를 GitLab CI 또는 Jenkins의 Secret 변수로 등록.

---

## 5. 프로젝트별 SBOM 생성 및 샘플 가이드

`samples/` 디렉터리에 각 언어 및 CI/CD 환경별 샘플 파일이 준비되어 있습니다.

### 5.1 Java / Spring Boot (Maven)
* **샘플 파일**: [`samples/maven-pom.xml.snippet`](samples/maven-pom.xml.snippet)
* `pom.xml`에 `cyclonedx-maven-plugin`을 추가합니다.
* 빌드 실행 시 `target/bom.json`이 자동 생성됩니다:
  ```bash
  mvn clean package -DskipTests
  ```

### 5.2 JavaScript / TypeScript (Node.js)
* **샘플 파일**: [`samples/package.json.snippet`](samples/package.json.snippet)
* `package.json`의 scripts에 등록하거나 `npx`로 실행합니다:
  ```bash
  npx @cyclonedx/cyclonedx-npm --output-file bom.json --output-format JSON --omit dev
  ```
  *(프로덕션 배포 대상 의존성만 분석하기 위해 `--omit dev` 적용)*

### 5.3 CI/CD 파이프라인 샘플
* **GitLab CI**:
  * Maven: [`samples/.gitlab-ci-maven.yml`](samples/.gitlab-ci-maven.yml)
  * Node.js: [`samples/.gitlab-ci-node.yml`](samples/.gitlab-ci-node.yml)
* **Jenkins**:
  * Maven: [`samples/Jenkinsfile-maven`](samples/Jenkinsfile-maven)
  * Node.js: [`samples/Jenkinsfile-node`](samples/Jenkinsfile-node)

---

## 6. 백업 및 유지관리 (Backup & Maintenance)

### PostgreSQL 데이터베이스 백업
```bash
docker exec -t dtrack-db pg_dump -U dtrack dtrack > dtrack_backup_$(date +%Y%m%d).sql
```

### PostgreSQL 데이터베이스 복구
```bash
cat dtrack_backup_20261012.sql | docker exec -i dtrack-db psql -U dtrack dtrack
```

### 볼륨 데이터 디렉터리
* `./volumes/db_data`: PostgreSQL 데이터베이스 파일
* `./volumes/apiserver_data`: 취약점 데이터베이스 캐시 및 Lucene 인덱스 파일
