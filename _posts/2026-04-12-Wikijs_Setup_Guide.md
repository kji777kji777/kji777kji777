# 📚 Wiki.js 구축 및 기술 자산화 가이드

> **Oracle Linux 8.10 + Docker 기반 Wiki.js Self-Hosted 완전 설치 매뉴얼**  
> "Docs as Code" — 문서를 코드처럼 관리하는 기술 자산화 환경 구축

---

## 📋 목차

1. [Wiki.js란 무엇인가](#1-wikijs란-무엇인가)
2. [사전 요구사항](#2-사전-요구사항)
3. [디렉터리 구조 및 권한 설정](#3-디렉터리-구조-및-권한-설정)
4. [docker-compose.yml 구성](#4-docker-composeyml-구성)
5. [방화벽 설정](#5-방화벽-설정)
6. [Wiki.js 실행 및 상태 확인](#6-wikijs-실행-및-상태-확인)
7. [초기 설정 (Setup Wizard)](#7-초기-설정-setup-wizard)
8. [한글화 (로컬라이징)](#8-한글화-로컬라이징)
9. [GitHub 저장소 연동 (Docs as Code)](#9-github-저장소-연동-docs-as-code)
10. [기본 사용법](#10-기본-사용법)
11. [Mattermost 연동 (알림 자동화)](#11-mattermost-연동-알림-자동화)
12. [운영 명령어 치트시트](#12-운영-명령어-치트시트)
13. [문제 해결 (Troubleshooting)](#13-문제-해결-troubleshooting)

---

## 1. Wiki.js란 무엇인가

Wiki.js는 **Node.js 기반의 오픈소스 위키 플랫폼**으로, 팀의 기술 문서를 체계적으로 관리하기 위한 최적의 Self-Hosted 솔루션입니다.

### 개발 인프라 내 역할

```
┌─────────────────────────────────────────────────────────┐
│              개발 관리 인프라 전체 구성                      │
├──────────────┬──────────────────────────────────────────┤
│ 영역          │ 도구                                      │
├──────────────┼──────────────────────────────────────────┤
│ 소스 관리     │ GitHub (버전 관리 / PR 코드 리뷰)            │
│ 팀원 소통     │ Mattermost [Self-Hosted]                  │
│ 문서/공유     │ ✅ Wiki.js [Self-Hosted] ← 본 문서         │
│ 일정/태스크   │ GitHub Projects (칸반 보드)                 │
│ CI/CD        │ GitHub Actions + Self-hosted Runner        │
│ 상태 알림     │ Mattermost Incoming Webhook               │
└──────────────┴──────────────────────────────────────────┘
```

### Wiki.js 핵심 특징

| 특징 | 설명 |
|------|------|
| 🔗 **Git 연동** | GitHub 저장소와 실시간 동기화 → 문서가 곧 코드 |
| ✍️ **Markdown 지원** | 개발자 친화적 Markdown 에디터 기본 제공 |
| 🌐 **다국어 지원** | 한국어 UI 지원 |
| 🔒 **접근 제어** | 사용자 그룹별 페이지 권한 관리 |
| 💰 **완전 무료** | AGPLv3 오픈소스, 기능 제한 없음 |

### 핵심 철학

> **"대화는 Mattermost에서, 기록은 Wiki.js에서."**  
> 모든 기술 결정 사항과 아키텍처 문서는 Wiki.js에 기록하고,  
> GitHub에 코드로 관리함으로써 서버 장애 시에도 완벽한 데이터 복구력을 확보합니다.

---

## 2. 사전 요구사항

### 시스템 환경

```
OS       : Oracle Linux 8.10 (RHEL 계열)
아키텍처  : x86_64
RAM      : 최소 1GB (권장 2GB 이상)
Disk     : 최소 10GB 여유 공간
네트워크  : 포트 3000 외부 접근 가능
```

### 사전 완료 조건

> Docker 및 Docker Compose가 이미 설치되어 있어야 합니다.  
> 미설치 상태라면 **[Mattermost 설치 가이드 3장]** 을 먼저 참고하세요.

```bash
# 설치 여부 확인
docker --version
docker compose version
```

---

## 3. 디렉터리 구조 및 권한 설정

### 3-1. 작업 디렉터리 생성

```bash
# 홈 디렉터리 기준으로 wikijs 폴더 생성
mkdir -p ~/wikijs
cd ~/wikijs

# 필수 하위 디렉터리 생성
mkdir -p config data/db
```

### 3-2. 디렉터리 구조

```
~/wikijs/
├── docker-compose.yml   ← 컨테이너 정의 파일
├── config/              ← Wiki.js 설정 파일
└── data/
    └── db/              ← PostgreSQL 데이터베이스 파일
```

### 3-3. 디렉터리 권한 설정

> Wiki.js 및 PostgreSQL 컨테이너가 파일을 읽고 쓸 수 있도록 아래 명령을 **반드시** 실행하세요.

```bash
sudo chmod -R 777 /home/rt/ta/wikijs
```

또는 현재 경로 기준으로 실행:

```bash
cd ~/wikijs
sudo chmod -R 777 .
```

> ⚠️ **보안 참고**: `777`은 초기 구동 및 테스트 환경용입니다.  
> 운영 환경에서는 적절한 소유권(`chown`) 설정을 권장합니다.

---

## 4. docker-compose.yml 구성

### 4-1. 파일 작성

```bash
vi ~/wikijs/docker-compose.yml
```

### 4-2. 전체 내용

```yaml
version: "3"

services:
  # ─────────────────────────────────
  # PostgreSQL 데이터베이스
  # ─────────────────────────────────
  wiki-db:
    image: postgres:15-alpine
    restart: always
    environment:
      - POSTGRES_DB=wikijs
      - POSTGRES_USER=wikiuser
      - POSTGRES_PASSWORD=wiki_password      # ← 운영 시 반드시 변경
    volumes:
      - ./data/db:/var/lib/postgresql/data

  # ─────────────────────────────────
  # Wiki.js 애플리케이션 서버
  # ─────────────────────────────────
  wikijs:
    image: requarks/wiki:2
    restart: always
    depends_on:
      - wiki-db
    environment:
      - DB_TYPE=postgres
      - DB_HOST=wiki-db
      - DB_PORT=5432
      - DB_USER=wikiuser
      - DB_PASS=wiki_password               # ← wiki-db의 POSTGRES_PASSWORD와 동일
      - DB_NAME=wikijs
    ports:
      - "3000:3000"
```

### 4-3. 구성 요소 설명

| 항목 | 값 | 설명 |
|------|-----|------|
| `postgres:15-alpine` | DB 이미지 | 경량 Alpine 기반 PostgreSQL 15 |
| `requarks/wiki:2` | 앱 이미지 | Wiki.js 2.x 공식 이미지 |
| `3000:3000` | 포트 | 호스트:컨테이너 포트 매핑 |
| `POSTGRES_DB` | `wikijs` | 데이터베이스 이름 |
| `POSTGRES_USER` | `wikiuser` | DB 접속 계정 |
| `POSTGRES_PASSWORD` | `wiki_password` | DB 비밀번호 **(운영 시 변경 필수)** |
| `DB_TYPE` | `postgres` | Wiki.js DB 드라이버 종류 |
| `DB_HOST` | `wiki-db` | 컨테이너 내부 DB 서비스명 |
| `restart: always` | 재시작 정책 | 서버 재부팅 시 자동 재시작 |

> ⚠️ **주의**: `wiki-db`의 `POSTGRES_PASSWORD`와 `wikijs`의 `DB_PASS` 값이 **반드시 동일**해야 합니다.

---

## 5. 방화벽 설정

Wiki.js는 기본적으로 포트 `3000`을 사용합니다.  
Oracle Linux 8.10의 `firewalld`에서 해당 포트를 허용해야 합니다.

```bash
# 포트 3000 영구 허용
sudo firewall-cmd --permanent --add-port=3000/tcp

# 방화벽 규칙 즉시 적용
sudo firewall-cmd --reload
```

**적용 확인:**

```bash
sudo firewall-cmd --list-ports
```

**예상 출력:**
```
3000/tcp
```

---

## 6. Wiki.js 실행 및 상태 확인

### 6-1. 컨테이너 시작

```bash
cd ~/wikijs

# 백그라운드로 실행 (-d: detached mode)
sudo docker compose up -d
```

**정상 실행 시 예상 출력:**
```
[+] Running 3/3
 ✔ Network wikijs_default    Created
 ✔ Container wikijs-wiki-db-1  Started
 ✔ Container wikijs-wikijs-1   Started
```

> ℹ️ `WARN[0000] version is obsolete` 경고는 `version: "3"` 항목에 대한 안내로,  
> 동작에는 전혀 영향이 없습니다. 무시해도 됩니다.

### 6-2. 실행 상태 확인

```bash
# 컨테이너 상태 확인
docker compose ps -a
```

**예상 출력:**
```
NAME               IMAGE                COMMAND                  SERVICE   STATUS         PORTS
wikijs-wiki-db-1   postgres:15-alpine   "docker-entrypoint.s…"  wiki-db   Up X seconds   5432/tcp
wikijs-wikijs-1    requarks/wiki:2      "docker-entrypoint.s…"  wikijs    Up X seconds   0.0.0.0:3000->3000/tcp, 3443/tcp
```

### 6-3. 로그 확인

```bash
# Wiki.js 최근 로그 30줄 확인
sudo docker compose logs wikijs | tail -n 30

# 실시간 로그 스트리밍
sudo docker compose logs -f wikijs

# DB 로그 확인
sudo docker compose logs wiki-db
```

### 6-4. 웹 브라우저 접속

```
http://<서버_IP>:3000
```

**정상 접속 화면:**

> Wiki.js 로고와 함께 "환영합니다!" 메시지 및  
> **[홈 문서 만들기]**, **[관리]** 버튼이 표시되면 성공입니다. ✅

---

## 7. 초기 설정 (Setup Wizard)

### 7-1. 관리자 계정 생성

최초 접속 시 Setup Wizard가 자동으로 실행됩니다.

1. 브라우저에서 `http://<서버_IP>:3000` 접속
2. **관리자 이메일** 입력
3. **관리자 비밀번호** 설정 (8자 이상)
4. **[설치 완료]** 버튼 클릭

### 7-2. 관리자 콘솔 접근

```
http://<서버_IP>:3000/a
또는
[관리] 버튼 클릭
```

---

## 8. 한글화 (로컬라이징)

### 8-1. 언어 팩 적용 경로

```
관리자 콘솔 → Site → Locale
Administration > Site > Locale
```

### 8-2. 설정 방법

1. 관리자 콘솔 접속 (`http://<서버_IP>:3000/a`)
2. 좌측 메뉴 **Site** 클릭
3. **Locale** 탭 선택
4. Locale 드롭다운에서 **Korean** 선택
5. **[Apply]** 클릭 → 페이지 새로고침

> 언어 팩 적용 후 관리 UI 전체가 한국어로 변경됩니다.

---

## 9. GitHub 저장소 연동 (Docs as Code)

Wiki.js의 핵심 기능입니다. 위키 문서를 GitHub 저장소에 **자동으로 백업 및 동기화**합니다.

### 9-1. GitHub PAT(Personal Access Token) 생성

1. GitHub 접속 → **Settings** → **Developer settings**
2. **Personal access tokens** → **Fine-grained tokens** → **Generate new token**
3. 아래 권한 설정:

| 권한 항목 | 설정 값 |
|-----------|---------|
| Repository access | 연동할 특정 저장소 선택 |
| **Contents** | **Read and write** ✅ |
| Metadata | Read-only (자동 포함) |

4. **[Generate token]** 클릭 → **토큰 값 복사 (재확인 불가)**

### 9-2. Wiki.js Storage 설정

```
관리자 콘솔 → Storage → Git 활성화
```

| 설정 항목 | 입력 값 | 설명 |
|-----------|---------|------|
| **Authentication Type** | `Basic` | ID/토큰 방식 인증 |
| **Repository URL** | `https://github.com/username/repo.git` | 연동할 저장소 주소 |
| **Branch** | `main` | 동기화할 브랜치 |
| **Username** | GitHub 사용자명 | |
| **Password / Token** | 생성한 PAT 토큰 | |
| **Local Repository Path** | `/wiki/data/repo` | 컨테이너 내 경로 (기본값 유지) |

### 9-3. 동기화 방향 설정

| 방향 | 설명 |
|------|------|
| `Bi-directional` | Wiki ↔ GitHub 양방향 동기화 (권장) |
| `Push to remote` | Wiki → GitHub 단방향 (백업 목적) |
| `Pull from remote` | GitHub → Wiki 단방향 (읽기 전용) |

### 9-4. 동작 확인

설정 저장 후 Wiki.js에서 문서를 작성하면:

```
Wiki.js 문서 저장
    ↓
GitHub 저장소에 자동 커밋
    ↓
.md 파일로 저장됨 (경로 구조 그대로 유지)
```

**GitHub 커밋 예시:**
```
docs/project/architecture.md  ← Wiki 경로 /project/architecture
docs/guide/setup.md           ← Wiki 경로 /guide/setup
```

---

## 10. 기본 사용법

### 10-1. 홈 문서 생성

1. Wiki.js 첫 화면에서 **[홈 문서 만들기]** 클릭
2. 에디터 타입 선택 → **Markdown** 선택 (권장)
3. 내용 작성 후 **[저장]** 클릭

### 10-2. 에디터 선택 가이드

| 에디터 | 권장 대상 | 특징 |
|--------|-----------|------|
| **Markdown** ✅ | 개발자, TA | 코드 블록, GitHub 연동 최적화 |
| Visual Editor | 비개발자 | WYSIWYG 방식 |
| Code | 고급 사용자 | 원시 HTML 편집 |

> **Markdown 에디터를 원칙으로 사용하세요.**  
> GitHub 연동 시 `.md` 파일로 저장되어 가독성이 보장됩니다.

### 10-3. 문서 경로 구조화 (중요)

문서 생성 시 **경로(Path)** 를 계층형으로 구성하면 GitHub에도 동일한 폴더 구조가 생성됩니다.

```
Wiki 경로 입력 예시:
project/backend/api-spec        → docs/project/backend/api-spec.md
project/architecture/overview   → docs/project/architecture/overview.md
guide/onboarding                → docs/guide/onboarding.md
```

**추천 경로 구조:**

```
/
├── project/
│   ├── architecture/    아키텍처 설계 문서
│   ├── backend/         백엔드 API 명세
│   ├── frontend/        프론트엔드 가이드
│   └── database/        DB 스키마 및 ERD
├── guide/
│   ├── onboarding/      신규 팀원 온보딩
│   └── convention/      코딩 컨벤션
└── ops/
    ├── setup/           서버 설치 매뉴얼
    └── runbook/         운영 절차서
```

### 10-4. 문서 작성 (Markdown 에디터)

```markdown
# 문서 제목

## 개요
프로젝트 아키텍처에 대한 설명입니다.

## 구성 요소

| 컴포넌트 | 기술 스택 | 설명 |
|----------|-----------|------|
| API 서버  | Node.js   | REST API 제공 |
| DB        | PostgreSQL| 데이터 저장 |

## 설치 방법

```bash
docker compose up -d
```

## 참고 링크
- [GitHub 저장소](https://github.com/username/repo)
```

### 10-5. 문서 수정

```
페이지 우측 상단 [연필/편집] 아이콘 클릭
    ↓
수정 후 [저장] 클릭
    ↓
GitHub에 자동 커밋 (연동 시)
```

### 10-6. 문서 삭제

> ⚠️ 삭제 시 GitHub 저장소의 해당 `.md` 파일도 자동으로 제거됩니다.

```
관리자 콘솔 → 페이지 메뉴 → 삭제할 문서 선택 → 삭제
```

### 10-7. 검색 기능

- 상단 검색창에서 전체 문서 내 키워드 검색
- 제목 및 본문 내용 동시 검색

### 10-8. 사용자 및 그룹 관리

```
관리자 콘솔 → Users → 사용자 초대 또는 생성
관리자 콘솔 → Groups → 권한 그룹 생성 및 페이지 접근 권한 설정
```

**추천 그룹 구성:**

| 그룹명 | 권한 | 대상 |
|--------|------|------|
| `Admins` | 전체 관리 | TA, 인프라 담당 |
| `Developers` | 읽기/쓰기 | 개발자 전체 |
| `Readers` | 읽기 전용 | 이해관계자 |

---

## 11. Mattermost 연동 (알림 자동화)

Wiki.js에서 문서가 생성/수정될 때 Mattermost 채널로 알림을 전송합니다.

### 11-1. Mattermost Incoming Webhook URL 준비

> Mattermost에서 Webhook을 먼저 생성해야 합니다.  
> 참고: **[Mattermost 설치 가이드 9-2장]**

```
http://192.168.93.138:8065/hooks/xxxxxxxxxxxxxxxxxxxxxxxx
```

### 11-2. Wiki.js Webhook 설정

```
관리자 콘솔 → Administration → Webhooks
```

| 설정 항목 | 입력 값 |
|-----------|---------|
| **URL** | Mattermost Incoming Webhook URL |
| **Type** | Slack-compatible (Mattermost는 Slack 포맷 호환) |
| **Events** | `page:created`, `page:updated` 선택 |

### 11-3. 알림 메시지 예시

```
📄 [Wiki.js] 새 문서가 등록되었습니다.
제목: API 명세서 v2.0
경로: /project/backend/api-spec
작성자: kim.developer
```

---

## 12. 운영 명령어 치트시트

```bash
# ─── 서비스 제어 ───────────────────────────────────────────
sudo docker compose up -d          # 백그라운드 시작
sudo docker compose down           # 완전 종료 (데이터 유지)
sudo docker compose restart        # 전체 재시작
sudo docker compose restart wikijs # Wiki.js만 재시작

# ─── 상태 확인 ────────────────────────────────────────────
docker compose ps -a               # 컨테이너 상태
sudo docker ps -a                  # 전체 컨테이너 목록
sudo docker compose logs wikijs | tail -n 30  # 최근 로그

# ─── 실시간 모니터링 ──────────────────────────────────────
sudo docker compose logs -f wikijs # 실시간 로그
sudo docker stats                  # CPU/메모리 사용량

# ─── 네트워크 확인 ────────────────────────────────────────
sudo netstat -tulpn | grep 3000    # 포트 리스닝 확인
sudo firewall-cmd --list-ports     # 방화벽 허용 포트 목록

# ─── 유지보수 ─────────────────────────────────────────────
sudo docker compose pull           # 이미지 최신 버전 업데이트
sudo docker image prune -f         # 미사용 이미지 정리

# ─── 백업 ─────────────────────────────────────────────────
tar -czf wikijs_backup_$(date +%Y%m%d).tar.gz \
  ~/wikijs/data \
  ~/wikijs/config
```

---

## 13. 문제 해결 (Troubleshooting)

### ❌ docker-compose.yml을 찾을 수 없다는 오류

```
no configuration file provided: not found
```

**원인:** 명령 실행 위치가 `docker-compose.yml`이 있는 디렉터리가 아님

```bash
# 반드시 docker-compose.yml이 있는 디렉터리에서 실행
cd ~/wikijs
sudo docker compose up -d
```

---

### ❌ 컨테이너가 시작되지 않을 때

```bash
# 상세 로그 확인
sudo docker compose logs --tail=50

# DB 컨테이너 상태 우선 확인
sudo docker compose logs wiki-db
```

**주요 원인:**
- `data/db/` 디렉터리 권한 부족 → `sudo chmod -R 777 ~/wikijs`
- DB 비밀번호 불일치 → `POSTGRES_PASSWORD`와 `DB_PASS` 동일한지 확인

---

### ❌ 브라우저에서 접속이 안 될 때

```bash
# 1. 컨테이너 실행 여부 확인
sudo docker compose ps

# 2. 포트 바인딩 확인
sudo netstat -tulpn | grep 3000

# 3. 방화벽 규칙 확인
sudo firewall-cmd --list-ports

# 4. 포트 재등록
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

---

### ❌ GitHub 연동 동기화 실패

```bash
# Wiki.js 로그에서 Git 오류 확인
sudo docker compose logs wikijs | grep -i "git\|storage\|error"
```

**주요 원인 및 해결:**

| 원인 | 해결 방법 |
|------|-----------|
| PAT 토큰 권한 부족 | GitHub → Token 설정에서 `Contents: Read and write` 확인 |
| 저장소 URL 오류 | `.git` 확장자 포함 여부, HTTPS URL 사용 여부 확인 |
| 브랜치명 불일치 | `main` vs `master` 확인 |
| 네트워크 차단 | 서버에서 `curl https://github.com` 연결 테스트 |

---

### ❌ `version is obsolete` 경고

```
WARN[0000] /home/rt/ta/wikijs/docker-compose.yml: `version` is obsolete
```

**설명:** Docker Compose V2에서 `version` 키가 더 이상 필요하지 않아 발생하는 경고입니다.  
**동작에는 전혀 영향이 없으며**, 무시해도 됩니다.  
제거하려면 `docker-compose.yml`에서 `version: "3"` 줄을 삭제하면 됩니다.

---

### ❌ Setup Wizard가 반복해서 표시될 때

DB 초기화 후 재설정 필요한 상황입니다.

```bash
# 컨테이너 및 데이터 초기화 (주의: 데이터 삭제됨)
sudo docker compose down
sudo rm -rf ~/wikijs/data/db/*
sudo docker compose up -d
```

---

## 📚 참고 자료

| 자료 | URL |
|------|-----|
| Wiki.js 공식 문서 | https://docs.requarks.io |
| Wiki.js Docker 설치 | https://docs.requarks.io/install/docker |
| Wiki.js GitHub 연동 | https://docs.requarks.io/storage/git |
| GitHub Fine-grained PAT | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens |

---

## 🗺️ 다음 단계

```
✅ Step 1: Mattermost Self-Hosted 설치  (완료)
✅ Step 2: Wiki.js Self-Hosted 설치     ← 현재 문서
⬜ Step 3: GitHub Actions Self-hosted Runner 등록
⬜ Step 4: GitHub Actions ↔ Mattermost Webhook 연동
⬜ Step 5: GitHub Projects 칸반 보드 구성
```

---

<div align="center">

**본 문서는 소프트웨어 개발 프로젝트의 무료 Self-Hosted 인프라 구축을 목적으로 작성되었습니다.**  
`Oracle Linux 8.10` · `Docker` · `Wiki.js 2.x` · `PostgreSQL 15` · `GitHub Storage`

*작성일: 2026-04-13*  
*작성자: Kim Jong-in (Technical Architect)*

</div>
