---
layout: post
title:  "Wiki.js Self-Hosted 구축 매뉴얼"
date:   2026-04-11 01:00:00 +0900
---

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
9. [Gitea 저장소 연동 (Docs as Code)](#9-gitea-저장소-연동-docs-as-code)
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
│ 소스 관리     │ Gitea [Self-Hosted] (버전 관리 / PR 코드 리뷰)│
│ 팀원 소통     │ Mattermost [Self-Hosted]                  │
│ 문서/공유     │ ✅ Wiki.js [Self-Hosted] ← 본 문서         │
│ 일정/태스크   │ Gitea Issues + Milestones [Self-Hosted]   │
│ CI/CD        │ Gitea Actions + Act Runner [Self-Hosted]   │
│ 상태 알림     │ Mattermost Incoming Webhook               │
└──────────────┴──────────────────────────────────────────┘
```

> ⚠️ **폐쇄망 환경 기준**: 본 가이드는 인터넷이 차단된 완전 폐쇄망 환경을 전제로 합니다.  
> 모든 서비스(Gitea 포함)는 내 서버에서만 동작하며, 외부 인터넷 연결이 불필요합니다.

### Wiki.js 핵심 특징

| 특징 | 설명 |
|------|------|
| 🔗 **Git 연동** | Gitea 저장소와 실시간 동기화 → 문서가 곧 코드 |
| ✍️ **Markdown 지원** | 개발자 친화적 Markdown 에디터 기본 제공 |
| 🌐 **다국어 지원** | 한국어 UI 지원 |
| 🔒 **접근 제어** | 사용자 그룹별 페이지 권한 관리 |
| 💰 **완전 무료** | AGPLv3 오픈소스, 기능 제한 없음 |

### 핵심 철학

> **"대화는 Mattermost에서, 기록은 Wiki.js에서."**  
> 모든 기술 결정 사항과 아키텍처 문서는 Wiki.js에 기록하고,  
> Gitea에 코드로 관리함으로써 서버 장애 시에도 완벽한 데이터 복구력을 확보합니다.  
> **모든 데이터는 내 서버 내부에서만 보관됩니다. (외부 유출 없음)**

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

> ℹ️ **Gitea 연동을 위해** Wiki.js 설치 완료 후 Step 3(Gitea 구축)을 먼저 완료해야 합니다.  
> Gitea가 구동 중이어야 9장의 저장소 연동을 진행할 수 있습니다.

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

## 9. Gitea 저장소 연동 (Docs as Code)

Wiki.js의 핵심 기능입니다. 위키 문서를 **내 서버의 Gitea 저장소**에 **자동으로 백업 및 동기화**합니다.  
GitHub 등 외부 서비스를 사용하지 않고, 폐쇄망 내부의 Gitea 서버만으로 완전히 운영됩니다.

> ⚠️ **사전 조건**: Gitea가 `http://<서버_IP>:3001`에서 정상 동작 중이어야 합니다.  
> Gitea 설치가 완료되지 않았다면 **[Step 3 — Gitea 구축]** 을 먼저 완료하세요.

### 9-1. Gitea에서 Wiki 전용 저장소 생성

Wiki.js 문서가 저장될 전용 Gitea 저장소를 먼저 생성합니다.

```
Gitea (http://<서버_IP>:3001) → 우측 상단 [+] → New Repository
```

| 항목 | 입력값 | 비고 |
|------|--------|------|
| Owner | `my-dev-team` (Organization) | 조직 소유로 생성 권장 |
| Repository Name | `wiki-docs` | 문서 전용 저장소명 |
| Visibility | `Private` | 팀 내부 전용 |
| Initialize Repository | ✅ 체크 | 초기 커밋 생성 필수 |
| Default Branch | `main` | |

**[Create Repository]** 클릭 → 저장소 생성 완료

### 9-2. Gitea PAT (Personal Access Token) 생성

Wiki.js가 Gitea 저장소에 접근하기 위한 인증 토큰을 생성합니다.

```
Gitea → 우측 상단 프로필 아이콘 → Settings
→ Applications → Access Tokens → [Generate New Token]
```

| 항목 | 입력값 |
|------|--------|
| Token Name | `wikijs-storage` |
| Expiration | `No expiration` (운영 환경 권장) 또는 원하는 기간 |
| Permissions — **repository** | **Read and Write** ✅ |

**[Generate Token]** 클릭 → **표시된 토큰 값을 즉시 복사** (재확인 불가)

```
예시: abc123def456xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ 토큰은 생성 직후 한 번만 표시됩니다. 반드시 안전한 곳에 복사해 두세요.

### 9-3. Wiki.js Storage — Gitea 연동 설정

```
관리자 콘솔 → Administration → Storage → Git 활성화
```

**Git Storage 설정값:**

| 설정 항목 | 입력값 | 설명 |
|-----------|--------|------|
| **Authentication Type** | `Basic` | ID/토큰 방식 인증 |
| **Repository URL** | `http://<서버_IP>:3001/<ORG>/wiki-docs.git` | Gitea 내부 주소 사용 |
| **Branch** | `main` | 동기화할 브랜치 |
| **Username** | Gitea 계정 아이디 | PAT를 발급한 계정 |
| **Password / Token** | 9-2에서 생성한 PAT 토큰 값 | |
| **Local Repository Path** | `/wiki/data/repo` | 컨테이너 내 경로 (기본값 유지) |

**Repository URL 입력 예시:**
```
http://192.168.xx.xxx:3001/my-dev-team/wiki-docs.git
```

> ℹ️ **폐쇄망 참고**: URL에 `https://` 대신 `http://`를 사용합니다.  
> Gitea가 내부 서버이므로 SSL 인증서 없이 HTTP로 접근하는 것이 정상입니다.

### 9-4. 동기화 방향 설정

| 방향 | 설명 |
|------|------|
| `Bi-directional` | Wiki ↔ Gitea 양방향 동기화 (권장) |
| `Push to remote` | Wiki → Gitea 단방향 (백업 목적) |
| `Pull from remote` | Gitea → Wiki 단방향 (읽기 전용) |

**[Save]** 클릭 후 **[Force Sync]** 버튼으로 초기 동기화 실행

### 9-5. 동작 확인

설정 저장 후 Wiki.js에서 문서를 작성하면:

```
Wiki.js 문서 저장
    ↓
Gitea 저장소(wiki-docs)에 자동 커밋
    ↓
.md 파일로 저장됨 (Wiki 경로 구조 그대로 유지)
```

**Gitea 저장소 커밋 예시:**
```
http://<서버_IP>:3001/my-dev-team/wiki-docs
    └── project/
    │   └── architecture.md   ← Wiki 경로 /project/architecture
    └── guide/
        └── setup.md          ← Wiki 경로 /guide/setup
```

**Gitea에서 확인:**
```
Gitea → my-dev-team/wiki-docs 저장소
→ Commits 탭에서 Wiki.js 자동 커밋 기록 확인 ✅
```

### 9-6. Gitea 저장소 연동 체크리스트

```
□ Gitea wiki-docs 저장소 생성 (Private · Initialize 포함)
□ Gitea PAT 생성 (wikijs-storage · repository Read/Write)
□ Wiki.js Storage → Git 활성화
□ Repository URL: http://<서버_IP>:3001/<ORG>/wiki-docs.git
□ Username / Token 입력
□ [Force Sync] 실행 → 오류 없음 확인
□ 테스트 문서 작성 후 Gitea 저장소 커밋 자동 생성 확인
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
| **Markdown** ✅ | 개발자, TA | 코드 블록, Gitea 연동 최적화 |
| Visual Editor | 비개발자 | WYSIWYG 방식 |
| Code | 고급 사용자 | 원시 HTML 편집 |

> **Markdown 에디터를 원칙으로 사용하세요.**  
> Gitea 연동 시 `.md` 파일로 저장되어 가독성이 보장됩니다.

### 10-3. 문서 경로 구조화 (중요)

문서 생성 시 **경로(Path)** 를 계층형으로 구성하면 Gitea에도 동일한 폴더 구조가 생성됩니다.

```
Wiki 경로 입력 예시:
project/backend/api-spec        → project/backend/api-spec.md (Gitea)
project/architecture/overview   → project/architecture/overview.md (Gitea)
guide/onboarding                → guide/onboarding.md (Gitea)
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
- [Gitea 저장소](http://<서버_IP>:3001/my-dev-team/my-app)
```

### 10-5. 문서 수정

```
페이지 우측 상단 [연필/편집] 아이콘 클릭
    ↓
수정 후 [저장] 클릭
    ↓
Gitea wiki-docs 저장소에 자동 커밋 (연동 시)
```

### 10-6. 문서 삭제

> ⚠️ 삭제 시 Gitea 저장소의 해당 `.md` 파일도 자동으로 제거됩니다.

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
http://192.168.xx.xxx:8065/hooks/xxxxxxxxxxxxxxxxxxxxxxxx
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

### ❌ Gitea 저장소 연동 동기화 실패

```bash
# Wiki.js 로그에서 Git 오류 확인
sudo docker compose logs wikijs | grep -i "git\|storage\|error"
```

**주요 원인 및 해결:**

| 원인 | 해결 방법 |
|------|-----------|
| PAT 토큰 만료 또는 권한 부족 | Gitea → Settings → Applications에서 신규 토큰 재발급 후 Wiki.js Storage에 재입력 |
| Repository URL 오류 | `http://<서버_IP>:3001/<ORG>/wiki-docs.git` 형식 및 `.git` 확장자 확인 |
| Gitea 서버 미기동 | `cd ~/gitea && sudo docker compose ps` 로 Gitea 상태 확인 |
| 브랜치명 불일치 | Gitea 저장소의 기본 브랜치가 `main`인지 확인 |
| HTTP 대신 HTTPS 입력 | 폐쇄망에서는 `http://` 사용. `https://`는 SSL 인증서 필요 |

**Gitea PAT 재발급 후 Wiki.js 재설정:**
```
1. Gitea → Settings → Applications → 기존 토큰 삭제
2. [Generate New Token] → repository: Read and Write
3. Wiki.js → Administration → Storage → Git
   → Password/Token 필드에 새 PAT 입력 → [Save]
4. [Force Sync] 클릭 → 오류 없음 확인
```

---

### ❌ Gitea 저장소에 커밋이 생성되지 않을 때

```bash
# Wiki.js Storage 설정 재확인
# 관리자 콘솔 → Administration → Storage → Git

# 수동 강제 동기화
# [Force Sync] 버튼 클릭

# Gitea wiki-docs 저장소 초기화 여부 확인 (빈 저장소면 push 실패)
# Gitea → wiki-docs 저장소 → 파일이 하나 이상 있는지 확인
# → 없으면 README.md라도 직접 생성 후 커밋 필요
```

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
| Wiki.js Git Storage 연동 | https://docs.requarks.io/storage/git |
| Gitea PAT 생성 | `http://<서버_IP>:3001` → Settings → Applications |
| Gitea 공식 문서 | https://docs.gitea.com |

---

## 🗺️ 다음 단계

```
✅ Step 1: Mattermost Self-Hosted 설치  (완료)
✅ Step 2: Wiki.js Self-Hosted 설치     ← 현재 문서
⬜ Step 3: Gitea Self-Hosted 구축       (소스코드 저장소 · GitHub 완전 대체)
⬜ Step 4: Gitea Issues · 마일스톤 구축 (태스크 관리 · 스프린트 운영)
⬜ Step 5: Act Runner 등록              (CI/CD 실행 환경 구축)
⬜ Step 6: 전체 통합 — Webhook 연동     (자동화 완성)
```

> ℹ️ **Wiki.js ↔ Gitea 연동 순서 안내**  
> 본 문서의 9장(Gitea 저장소 연동)은 Step 3(Gitea 구축) 완료 후 진행합니다.  
> Wiki.js 설치(Step 2)와 Gitea 설치(Step 3) 순서대로 진행한 뒤,  
> Gitea가 기동된 상태에서 9장 설정을 적용하세요.

---

<div align="center">

**본 문서는 소프트웨어 개발 프로젝트의 완전 무료 Self-Hosted 인프라 구축을 목적으로 작성되었습니다.**  
`Oracle Linux 8.10` · `Docker` · `Wiki.js 2.x` · `PostgreSQL 15` · `Gitea Self-Hosted`

*최초 작성일: 2026-04-13*  
*업데이트: 2026-04-15 — GitHub 환경 → Gitea 폐쇄망 환경으로 전환*  
*작성자: Kim Jong-in (Technical Architect)*

</div>
