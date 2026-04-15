---
layout: post
title:  "Gitea 구축 및 운영 가이드"
date:   2026-04-12 01:00:00 +0900
---

# 🦊 Gitea 구축 및 운영 가이드

> **폐쇄망 완전 자립 — GitHub 없이 내 서버에서 Git 저장소 완전 운영**  
> TA/개발자를 위한 Gitea 초기 설치부터 조직 관리 · 이슈 · CI/CD 연동까지 완전 매뉴얼

---

## 📋 목차

1. [Gitea란 무엇인가](#1-gitea란-무엇인가)
2. [사전 요구사항](#2-사전-요구사항)
3. [Docker로 Gitea 설치](#3-docker로-gitea-설치)
4. [초기 설정 마법사 (Install Wizard)](#4-초기-설정-마법사-install-wizard)
5. [관리자 초기 설정](#5-관리자-초기-설정)
6. [조직(Organization) 및 팀(Team) 구성](#6-조직organization-및-팀team-구성)
7. [저장소(Repository) 생성 및 관리](#7-저장소repository-생성-및-관리)
8. [이슈(Issue) · 마일스톤 · 라벨 관리](#8-이슈issue--마일스톤--라벨-관리)
9. [Pull Request 및 코드 리뷰](#9-pull-request-및-코드-리뷰)
10. [Gitea Actions — CI/CD 자동화](#10-gitea-actions--cicd-자동화)
11. [Mattermost Webhook 연동](#11-mattermost-webhook-연동)
12. [사용자 관리 및 권한 설정](#12-사용자-관리-및-권한-설정)
13. [팀 협업 실전 운영 가이드](#13-팀-협업-실전-운영-가이드)
14. [운영 치트시트](#14-운영-치트시트)
15. [문제 해결 (Troubleshooting)](#15-문제-해결-troubleshooting)

---

## 1. Gitea란 무엇인가

Gitea는 **Go 언어로 작성된 경량 Self-Hosted Git 서비스**입니다.  
GitHub과 동일한 UI·기능을 제공하면서 단일 바이너리로 내 서버에서 완전히 운영할 수 있습니다.  
인터넷이 차단된 폐쇄망 환경에서 GitHub을 100% 대체할 수 있는 가장 현실적인 선택입니다.

### 폐쇄망 개발 인프라 내 역할

```
┌─────────────────────────────────────────────────────────┐
│         폐쇄망 Self-Hosted 개발 관리 인프라 구성             │
├──────────────┬──────────────────────────────────────────┤
│ 영역          │ 도구                                      │
├──────────────┼──────────────────────────────────────────┤
│ 소스 관리     │ ✅ Gitea [Self-Hosted] ← 본 문서           │
│ 팀원 소통     │ Mattermost [Self-Hosted]                  │
│ 문서/공유     │ Wiki.js [Self-Hosted]                     │
│ 일정/태스크   │ Gitea Issues + Milestones                 │
│ CI/CD        │ Gitea Actions + Self-hosted Runner        │
│ 상태 알림     │ Mattermost Incoming Webhook               │
└──────────────┴──────────────────────────────────────────┘
```

### GitHub vs Gitea 비교

| 항목 | GitHub | Gitea |
|------|--------|-------|
| **운영 방식** | SaaS (외부 서버) | Self-Hosted (내 서버) |
| **인터넷 필요** | 필수 | ❌ 불필요 (폐쇄망 가능) |
| **데이터 위치** | GitHub 서버 | 내 서버 (완전 통제) |
| **비용** | 팀 규모 증가 시 유료 | **완전 무료** |
| **저장소 수** | 플랜 제한 | **무제한** |
| **사용자 수** | 플랜 제한 | **무제한** |
| **CI/CD** | GitHub Actions (월 2,000분 제한) | Gitea Actions (제한 없음) |
| **리소스** | — | RAM 150MB ~ (매우 경량) |

### Gitea 핵심 특징

| 특징 | 설명 |
|------|------|
| 🦊 **GitHub 호환 UI** | GitHub과 거의 동일한 인터페이스 — 학습 비용 최소 |
| 🏠 **완전 Self-Hosted** | 데이터가 내 서버에서만 존재, 외부 유출 없음 |
| ⚡ **초경량** | RAM 150MB, 단일 바이너리로 실행 |
| 🔄 **Gitea Actions** | GitHub Actions 워크플로우 문법 100% 호환 |
| 🔗 **Webhook 지원** | Mattermost · Slack 등 외부 알림 시스템 연동 |
| 🛡️ **완전 무료** | MIT 라이선스 오픈소스 |

### 핵심 개념 정리

```
Gitea 서버 (Self-Hosted)
    ├── Organization (조직)
    │       ├── Team (팀 — 권한 그룹)
    │       └── Repository (저장소)
    │               ├── Issue (이슈 · 태스크)
    │               ├── Pull Request (코드 리뷰)
    │               ├── Milestone (마일스톤)
    │               ├── Wiki (저장소별 위키)
    │               └── Actions (CI/CD 워크플로우)
    └── User (개인 사용자)
            └── Repository (개인 저장소)
```

---

## 2. 사전 요구사항

### 서버 환경

| 항목 | 최소 사양 | 권장 사양 |
|------|----------|----------|
| **OS** | Oracle Linux 8.10 | Oracle Linux 8.10 |
| **CPU** | 1 Core | 2 Core 이상 |
| **RAM** | 512 MB | 2 GB 이상 |
| **디스크** | 10 GB | 50 GB 이상 (저장소 규모에 따라) |
| **Docker** | 20.10 이상 | 최신 버전 |
| **Docker Compose** | v2.0 이상 | 최신 버전 |

### 사전 패키지 확인

```bash
# Docker 설치 확인
docker --version
sudo docker compose version

# git 설치 확인
git --version

# 미설치 시
sudo dnf install -y git
```

### 네트워크 및 포트 계획

| 서비스 | 포트 | 프로토콜 | 용도 |
|--------|------|----------|------|
| Gitea 웹 UI | `3001` | HTTP | 웹 브라우저 접속 |
| Gitea SSH | `2222` | TCP | Git SSH 클론·푸시 |
| Gitea DB | `5432` | TCP | 내부 전용 |

> 포트 `3000`은 Wiki.js가 사용 중이므로 Gitea는 `3001`을 사용합니다.

### 용어 정리

| 용어 | 설명 |
|------|------|
| **Repository** | 소스코드를 저장하는 Git 저장소 |
| **Organization** | 여러 저장소를 묶어 팀 단위로 관리하는 그룹 |
| **Team** | Organization 내 권한을 공유하는 사용자 그룹 |
| **Issue** | 개발 태스크, 버그, 기능 요청의 단위 |
| **Milestone** | 버전 또는 스프린트 단위 목표 |
| **Pull Request** | 브랜치 병합 요청 및 코드 리뷰 |
| **Webhook** | 이벤트 발생 시 외부 시스템으로 알림을 전송하는 HTTP 콜백 |
| **Actions Runner** | Gitea Actions 워크플로우를 실행하는 에이전트 |

---

## 3. Docker로 Gitea 설치

### 3-1. 디렉터리 생성

```bash
# Gitea 작업 디렉터리 생성
mkdir -p ~/gitea/{data,config,db}

# 권한 설정
sudo chmod -R 777 ~/gitea
```

### 3-2. docker-compose.yml 작성

```bash
cd ~/gitea
vi docker-compose.yml
```

```yaml
# ~/gitea/docker-compose.yml
version: "3"

networks:
  gitea-net:
    driver: bridge

services:

  # ─── PostgreSQL 데이터베이스 ──────────────────────────────
  gitea-db:
    image: postgres:15-alpine
    container_name: gitea-db
    restart: always
    networks:
      - gitea-net
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea_password        # ← 반드시 변경
      - POSTGRES_DB=gitea
    volumes:
      - ./db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gitea"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── Gitea 서버 ──────────────────────────────────────────
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    networks:
      - gitea-net
    depends_on:
      gitea-db:
        condition: service_healthy
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=gitea-db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_password  # ← DB 비밀번호와 동일하게 변경
    ports:
      - "3001:3000"     # 웹 UI (호스트 3001 → 컨테이너 3000)
      - "2222:22"       # SSH  (호스트 2222 → 컨테이너 22)
    volumes:
      - ./data:/data
      - ./config:/etc/gitea
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
```

### 3-3. 컨테이너 실행

```bash
cd ~/gitea
sudo docker compose up -d

# 실행 상태 확인
sudo docker compose ps
sudo docker compose logs -f gitea   # 로그 실시간 확인 (Ctrl+C로 종료)
```

**정상 기동 로그 확인:**
```
gitea  | 2026/04/13 ... server.go:xxx: HTTP Listener: 0.0.0.0:3000
```

### 3-4. 방화벽 포트 허용

```bash
# 웹 UI 포트
sudo firewall-cmd --permanent --add-port=3001/tcp

# SSH 포트
sudo firewall-cmd --permanent --add-port=2222/tcp

# 방화벽 재로드
sudo firewall-cmd --reload

# 허용 포트 확인
sudo firewall-cmd --list-ports
```

### 3-5. 접속 확인

```
브라우저에서 접속:
http://<서버_IP>:3001

초기 설정 마법사 화면이 나타나면 설치 성공 ✅
```

---

## 4. 초기 설정 마법사 (Install Wizard)

첫 접속 시 **Install Gitea** 설정 페이지가 표시됩니다. 아래 항목을 순서대로 설정합니다.

### 4-1. 데이터베이스 설정

| 항목 | 입력값 |
|------|--------|
| **Database Type** | `PostgreSQL` |
| **Host** | `gitea-db:5432` |
| **Username** | `gitea` |
| **Password** | `gitea_password` (docker-compose.yml에서 설정한 값) |
| **Database Name** | `gitea` |

### 4-2. 일반 설정

| 항목 | 입력값 | 설명 |
|------|--------|------|
| **Site Title** | `우리팀 Gitea` | 브라우저 탭 및 상단에 표시되는 이름 |
| **Repository Root Path** | `/data/gitea/repositories` | 기본값 유지 |
| **Git Hooks Path** | `/data/gitea/hooks` | 기본값 유지 |
| **LFS Root Path** | `/data/gitea/lfs` | 기본값 유지 |
| **Run As Username** | `git` | 기본값 유지 |

### 4-3. 서버 도메인 설정

> ⚠️ **폐쇄망 환경에서 가장 중요한 설정입니다.** 잘못 입력 시 Clone URL이 틀어집니다.

| 항목 | 입력값 | 설명 |
|------|--------|------|
| **Server Domain** | `192.168.93.138` | 서버의 실제 IP 주소 |
| **Gitea HTTP Listen Port** | `3000` | 컨테이너 내부 포트 (변경 금지) |
| **Gitea Base URL** | `http://192.168.93.138:3001/` | 외부에서 접속하는 실제 URL |
| **SSH Server Port** | `2222` | 호스트 SSH 포트 |

### 4-4. 선택 기능 설정

| 항목 | 권장값 | 설명 |
|------|--------|------|
| **Email Settings** | 폐쇄망은 비워둠 | SMTP 서버가 있을 경우 설정 |
| **Enable Registration** | ✅ 활성화 | 초기 팀원 가입용 (이후 비활성화 권장) |
| **Require Sign-In** | ✅ 활성화 | 로그인 없이는 접근 불가 |
| **Enable Git Hooks** | ✅ 활성화 | 서버사이드 Git Hook 사용 |
| **Disable CDN** | ✅ 활성화 | **폐쇄망 필수** — 외부 CDN 차단 |

### 4-5. 관리자 계정 생성

```
Administrator Account Settings 섹션:

Admin Username : admin          ← 영문·숫자만 사용
Admin Password : Admin@1234!    ← 8자 이상, 특수문자 포함
Confirm Password : Admin@1234!
Admin Email    : admin@local.dev
```

> 이 계정이 Gitea 최고 관리자 계정이 됩니다. 비밀번호는 반드시 안전한 곳에 기록합니다.

### 4-6. 설치 완료

```
[Install Gitea] 버튼 클릭
→ 설치 완료 후 로그인 화면으로 자동 이동
→ 위에서 생성한 admin 계정으로 로그인
```

---

## 5. 관리자 초기 설정

설치 완료 후 관리자 콘솔에서 필수 설정을 진행합니다.

### 5-1. 관리자 콘솔 진입

```
우측 상단 프로필 아이콘 → Site Administration
또는 직접 접속: http://<서버_IP>:3001/-/admin
```

### 5-2. 보안 설정 — 회원가입 제한

초기 팀원 가입이 완료되면 무분별한 가입을 차단합니다.

```
Site Administration → Settings → User Settings

□ Disable Self-Registration    → 체크 (외부 가입 차단)
□ Require Sign-In To View Pages → 체크 (로그인 없이 열람 차단)

[Save] 클릭
```

### 5-3. Git 저장소 기본 설정

```
Site Administration → Settings → Repository Settings

Default Branch       : main          ← master 대신 main 사용
Repository Max Size  : 0             ← 0은 무제한
Enable Git LFS       : ✅ 체크       ← 대용량 파일 지원

[Save] 클릭
```

### 5-4. Git 기본 설정

```
Site Administration → Settings → Git Settings

Default Git Merge Style  : Merge     ← PR 머지 방식 기본값
Max Git Diff Lines       : 1000      ← diff 표시 최대 줄 수

[Save] 클릭
```

### 5-5. Webhook 허용 주소 설정 (Mattermost 연동 필수)

폐쇄망에서 내부 IP로 Webhook을 전송하려면 반드시 설정합니다.

```
Site Administration → Settings → Webhook Settings

Allowed Hosts : 192.168.93.138    ← Mattermost 서버 IP
                (쉼표로 복수 IP 허용 가능: 192.168.93.138, 192.168.93.0/24)

[Save] 클릭
```

### 5-6. 초기 설정 체크리스트

```
□ 관리자 계정 로그인 확인
□ 회원가입 제한 설정 (팀원 가입 완료 후)
□ 기본 브랜치 main으로 변경
□ Webhook 허용 주소 등록
□ 팀원 계정 생성 또는 가입 안내
```

---

## 6. 조직(Organization) 및 팀(Team) 구성

개인 저장소 대신 **Organization으로 팀 단위 관리**하면 권한 관리와 프로젝트 분리가 편리합니다.

### 6-1. Organization 생성

```
우측 상단 [+] → New Organization
```

| 항목 | 입력 예시 | 설명 |
|------|-----------|------|
| **Organization Name** | `my-dev-team` | URL에 사용됨 — 영문·숫자·하이픈만 |
| **Visibility** | `Private` | 조직 내 멤버만 접근 |

```
[Create Organization] 클릭
```

### 6-2. Team 구성

Organization 내에 역할별 Team을 구성합니다.

```
Organization 페이지 → Teams 탭 → [New Team]
```

**권장 Team 구성:**

| Team 이름 | 권한 | 대상 |
|-----------|------|------|
| `Owners` | Owner | 조직 관리자 (자동 생성) |
| `Developers` | Write | 일반 개발자 — push·PR 가능 |
| `Reviewers` | Write | 시니어 개발자 — 리뷰·머지 권한 |
| `Viewers` | Read | 열람 전용 — 외부 협력사 등 |

**Team 생성 설정:**

```
Team Name       : Developers
Description     : 개발팀 — 저장소 쓰기 권한
Permission      : Write
                  ├── Read  : 저장소 열람
                  ├── Write : push · 이슈 · PR
                  └── Admin : 저장소 설정 변경

[Create Team] 클릭
```

### 6-3. Team에 멤버 추가

```
Team 페이지 → Members 탭
→ 사용자명 검색 → [Add Team Member]
```

### 6-4. Team에 저장소 연결

```
Team 페이지 → Repositories 탭
→ 저장소명 검색 → [Add Team Repository]
```

### 6-5. Organization 구조 예시

```
Organization: my-dev-team
    ├── Team: Owners
    │       └── Members: admin
    ├── Team: Developers
    │       ├── Members: dev-kim, dev-lee, dev-park
    │       └── Repositories: my-app, my-api, my-infra
    └── Team: Viewers
            └── Members: pm-choi
```

---

## 7. 저장소(Repository) 생성 및 관리

### 7-1. 저장소 생성

```
우측 상단 [+] → New Repository
```

| 항목 | 입력 예시 | 설명 |
|------|-----------|------|
| **Owner** | `my-dev-team` | Organization 선택 (권장) |
| **Repository Name** | `my-app` | 영문·숫자·하이픈 |
| **Description** | `메인 애플리케이션 서버` | 저장소 설명 |
| **Visibility** | `Private` | 팀 내부용 |
| **Initialize Repository** | ✅ 체크 | README.md 자동 생성 |
| **Default Branch** | `main` | 기본 브랜치 |
| **.gitignore** | 프로젝트 언어 선택 | Node, Python, Java 등 |
| **License** | 필요 시 선택 | 오픈소스 아닌 경우 생략 |

```
[Create Repository] 클릭
```

### 7-2. 로컬에서 저장소 Clone

**HTTPS 방식 (기본):**

```bash
git clone http://<서버_IP>:3001/<ORG>/<REPO>.git

# 예시
git clone http://192.168.93.138:3001/my-dev-team/my-app.git
```

**SSH 방식 (SSH 키 등록 후):**

```bash
git clone ssh://git@<서버_IP>:2222/<ORG>/<REPO>.git

# 예시
git clone ssh://git@192.168.93.138:2222/my-dev-team/my-app.git
```

### 7-3. SSH 키 등록

SSH 방식으로 인증 없이 push·pull하려면 SSH 공개키를 등록합니다.

**로컬 PC에서 SSH 키 생성:**

```bash
# SSH 키 생성 (이미 있으면 생략)
ssh-keygen -t ed25519 -C "gitea-key"

# 공개키 내용 복사
cat ~/.ssh/id_ed25519.pub
```

**Gitea에 공개키 등록:**

```
우측 상단 프로필 → Settings → SSH / GPG Keys
→ [Add Key]
→ Key Name : my-laptop
→ Content  : (위에서 복사한 공개키 전체 붙여넣기)
→ [Add Key]
```

**SSH 연결 테스트:**

```bash
ssh -T git@<서버_IP> -p 2222

# 성공 시 출력
# Hi <username>! You've successfully authenticated with Gitea.
```

### 7-4. 브랜치 보호 규칙 설정

`main` 브랜치에 직접 push를 막고 PR을 통한 코드 리뷰를 강제합니다.

```
저장소 → Settings → Branches → [Add Branch Protection Rule]

Branch Name Pattern     : main
□ Protect Branch        : ✅ 체크
□ Block Force Push      : ✅ 체크
□ Require Merge Request : ✅ 체크 (PR 필수)
□ Required Approvals    : 1     (최소 승인 수)

[Save] 클릭
```

### 7-5. 저장소 기본 라벨 생성

이슈 분류를 위한 라벨을 표준화합니다.

```
저장소 → Issues → Labels → [Create Default Labels]
또는 [New Label]로 개별 생성
```

**권장 라벨 구성:**

| 라벨명 | 색상 코드 | 의미 |
|--------|----------|------|
| `feature` | `#0075ca` | 새 기능 개발 |
| `bug` | `#d73a4a` | 버그 수정 |
| `docs` | `#0052cc` | 문서 작업 |
| `infra` | `#e4e669` | 인프라·DevOps |
| `refactor` | `#cfd3d7` | 코드 개선 |
| `hotfix` | `#b60205` | 긴급 수정 |
| `blocked` | `#d93f0b` | 다른 작업에 의해 차단됨 |

---

## 8. 이슈(Issue) · 마일스톤 · 라벨 관리

### 8-1. 이슈 생성

```
저장소 → Issues 탭 → [New Issue]
```

**이슈 작성 시 설정 항목 (우측 사이드바):**

```
Assignees  → 담당자 지정
Labels     → 유형 라벨 선택 (feature, bug 등)
Milestone  → 속하는 마일스톤 선택
Due Date   → 마감일 설정
```

### 8-2. 이슈 템플릿 설정

팀 전체가 일관된 형식으로 이슈를 작성하도록 템플릿을 등록합니다.

**템플릿 파일 생성 위치:**

```
저장소 루트/.gitea/ISSUE_TEMPLATE/
```

**기능 개발 템플릿** (`.gitea/ISSUE_TEMPLATE/feature.md`):

```markdown
---
name: ✨ 기능 개발
about: 새로운 기능 개발 요청
title: "[FEAT] "
labels: feature
---

## 📌 개요
<!-- 어떤 기능을 개발하는지 한 줄로 요약하세요 -->

## 🎯 목적 및 배경
<!-- 왜 이 기능이 필요한지 설명하세요 -->

## ✅ 완료 조건 (Acceptance Criteria)
- [ ] 조건 1
- [ ] 조건 2
- [ ] 조건 3

## 📋 작업 목록 (Sub Tasks)
- [ ] 세부 작업 1
- [ ] 세부 작업 2

## 🔗 관련 문서
<!-- Wiki.js 링크, 설계 문서 등 -->
```

**버그 리포트 템플릿** (`.gitea/ISSUE_TEMPLATE/bug.md`):

```markdown
---
name: 🐛 버그 리포트
about: 버그 신고 및 수정 요청
title: "[BUG] "
labels: bug
---

## 🐛 버그 설명

## 🔁 재현 방법
1.
2.
3.

## ✅ 기대 동작

## ❌ 실제 동작

## 🖥️ 환경 정보
- OS:
- 버전:

## 📸 스크린샷
```

### 8-3. 마일스톤 생성

마일스톤은 **스프린트** 또는 **릴리즈 버전** 단위로 이슈를 묶어 진척도를 추적합니다.

```
저장소 → Issues → Milestones → [New Milestone]
```

| 항목 | 입력 예시 |
|------|-----------|
| **Title** | `Sprint 1` 또는 `v1.0.0` |
| **Description** | `2026년 4월 3주차 스프린트` |
| **Due Date** | `2026-04-25` |

**마일스톤 진척도 확인:**

```
저장소 → Issues → Milestones
→ 각 마일스톤의 Open/Closed 이슈 비율로 진척도 자동 표시
```

---

## 9. Pull Request 및 코드 리뷰

### 9-1. PR 생성 흐름

```
1. 이슈 기반 브랜치 생성
   git checkout -b feature/42-user-auth

2. 코드 작성 및 커밋
   git add .
   git commit -m "feat: #42 사용자 로그인 JWT 인증 구현"

3. 원격 저장소에 push
   git push origin feature/42-user-auth

4. Gitea 웹에서 PR 생성
   저장소 → Pull Requests → [New Pull Request]
   → base: main ← compare: feature/42-user-auth
```

### 9-2. PR 템플릿 설정

```
저장소 루트/.gitea/PULL_REQUEST_TEMPLATE.md
```

```markdown
## 🔗 관련 이슈
Close #이슈번호

## 📝 변경 내용
<!-- 어떤 변경을 했는지 요약 -->

## ✅ 테스트 확인
- [ ] 로컬 테스트 완료
- [ ] 기존 기능 영향 없음 확인

## 📸 스크린샷 (UI 변경 시)

## 💬 리뷰어에게
<!-- 리뷰 시 참고할 사항 -->
```

> PR 본문에 `Close #42` 형식으로 작성하면 PR 머지 시 **이슈가 자동으로 Close**됩니다.

### 9-3. 이슈 자동 Close 키워드

```
Close #42       ← 권장
Closes #42
Fix #42
Fixes #42
Resolve #42
Resolves #42
```

### 9-4. 코드 리뷰 진행

```
PR 페이지 → Files Changed 탭
→ 변경된 코드 라인 클릭 → 댓글 작성
→ [Start Review] → 전체 리뷰 완료 후 [Submit Review]

리뷰 결과:
  ✅ Approve      : 승인 — 머지 가능
  💬 Comment      : 의견만 남김
  ❌ Request Changes : 수정 요청 — 재검토 필요
```

### 9-5. PR 머지

```
PR 페이지 → 필요 승인 수 충족 확인
→ [Merge Pull Request] 클릭

머지 방식:
  Merge Commit    : 모든 커밋 기록 유지 (기본값)
  Squash and Merge: 커밋을 하나로 합쳐 머지
  Rebase and Merge: 선형 히스토리 유지
```

---

## 10. Gitea Actions — CI/CD 자동화

Gitea Actions는 GitHub Actions와 **완전히 동일한 YAML 문법**을 사용합니다.  
기존 GitHub Actions 워크플로우를 그대로 재사용할 수 있습니다.

### 10-1. Gitea Actions 활성화

```
Site Administration → Settings → Repository Settings
→ Enable Repository Actions : ✅ 체크
→ [Save]
```

### 10-2. Act Runner 설치 (Self-hosted)

Gitea Actions를 실행하는 Runner를 서버에 설치합니다.

**Runner 바이너리 다운로드:**

```bash
# act_runner 다운로드 (최신 버전 확인: https://gitea.com/gitea/act_runner/releases)
RUNNER_VERSION="0.2.11"

wget -O act_runner \
  https://gitea.com/gitea/act_runner/releases/download/v${RUNNER_VERSION}/act_runner-${RUNNER_VERSION}-linux-amd64

# 실행 권한 부여
chmod +x act_runner

# 시스템 경로로 이동
sudo mv act_runner /usr/local/bin/act_runner

# 버전 확인
act_runner --version
```

> ⚠️ **폐쇄망 환경**: 인터넷 접속이 불가능할 경우, 인터넷이 되는 PC에서 바이너리를 다운로드한 후 서버로 복사합니다.
> ```bash
> scp act_runner user@<서버_IP>:~/
> ```

**Runner 전용 디렉터리 생성:**

```bash
sudo mkdir -p /opt/act-runner
sudo chown runner:runner /opt/act-runner
cd /opt/act-runner
```

### 10-3. Runner 등록 토큰 발급

```
Site Administration → Actions → Runners → [Create new Runner]
→ 표시된 Registration Token 복사
```

또는 저장소 단위 Runner:

```
저장소 → Settings → Actions → Runners → [Create new Runner]
→ Registration Token 복사
```

### 10-4. Runner 등록

```bash
cd /opt/act-runner

# Runner 설정 파일 생성
act_runner register \
  --instance http://<서버_IP>:3001 \
  --token <REGISTRATION_TOKEN> \
  --name oracle-linux-runner \
  --labels self-hosted,linux,oracle-linux \
  --no-interactive
```

### 10-5. Runner systemd 서비스 등록

```bash
# 서비스 파일 생성
sudo tee /etc/systemd/system/act-runner.service << 'EOF'
[Unit]
Description=Gitea Act Runner
After=network.target

[Service]
Type=simple
User=runner
WorkingDirectory=/opt/act-runner
ExecStart=/usr/local/bin/act_runner daemon
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 서비스 등록 및 시작
sudo systemctl daemon-reload
sudo systemctl enable act-runner
sudo systemctl start act-runner

# 상태 확인
sudo systemctl status act-runner
```

### 10-6. 기본 CI 워크플로우 예시

```yaml
# .gitea/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    runs-on: self-hosted

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: Node.js 의존성 설치
        run: npm ci

      - name: 빌드
        run: npm run build

      - name: 테스트
        run: npm test

      - name: 빌드 성공 알림
        if: success()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"✅ **빌드 성공** \`${{ gitea.repository }}\`\n브랜치: \`${{ gitea.ref_name }}\`\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}

      - name: 빌드 실패 알림
        if: failure()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"❌ **빌드 실패** \`${{ gitea.repository }}\`\n브랜치: \`${{ gitea.ref_name }}\`\n즉시 확인 필요!\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

> **GitHub Actions와의 차이**: `${{ github.xxx }}` 대신 `${{ gitea.xxx }}`를 사용합니다.

### 10-7. Secrets 등록

```
저장소 → Settings → Actions → Secrets → [Add Secret]

Name  : MATTERMOST_WEBHOOK_URL
Value : http://192.168.93.138:8065/hooks/xxxxxxxxxx
```

---

## 11. Mattermost Webhook 연동

Gitea 이벤트(push, PR, 이슈)를 Mattermost 채널로 자동 전송합니다.

### 11-1. Mattermost Incoming Webhook URL 준비

```
Mattermost → Integrations → Incoming Webhooks → [Add Incoming Webhook]
→ Channel: #dev-alert
→ [Save] → Webhook URL 복사
```

### 11-2. Gitea 저장소 Webhook 설정

```
저장소 → Settings → Webhooks → [Add Webhook] → Gitea 선택
```

| 항목 | 입력값 |
|------|--------|
| **Target URL** | `http://192.168.93.138:8065/hooks/xxxxxxxxxx` |
| **HTTP Method** | `POST` |
| **Content Type** | `application/json` |
| **Secret** | (비워둠 또는 임의 문자열) |
| **Trigger On** | 아래 체크 목록 참고 |

**권장 트리거 이벤트:**

```
✅ Push            ← 코드 push 시 알림
✅ Pull Request    ← PR 생성·머지·댓글 시 알림
✅ Issues          ← 이슈 생성·완료 시 알림
✅ Issue Comment   ← 이슈 댓글 시 알림
```

### 11-3. Webhook 동작 테스트

```
Webhooks 목록 → 해당 Webhook → [Test Delivery]
→ Mattermost #dev-alert 채널에 테스트 메시지 수신 확인
```

### 11-4. Gitea Actions로 상세 알림 (커스터마이즈)

Webhook으로 전송되는 기본 메시지 대신, Actions로 더 세밀하게 포맷을 제어할 수 있습니다.

```yaml
# .gitea/workflows/notify.yml
name: Mattermost Notification

on:
  issues:
    types: [opened, closed]
  pull_request:
    types: [opened, closed]

jobs:
  notify:
    runs-on: self-hosted
    steps:
      - name: 이슈/PR 알림 전송
        run: |
          EVENT="${{ gitea.event_name }}"
          ACTION="${{ gitea.event.action }}"

          if [ "$EVENT" = "issues" ] && [ "$ACTION" = "opened" ]; then
            MSG="🆕 **새 이슈**: [${{ gitea.event.issue.title }}](${{ gitea.event.issue.html_url }})"
          elif [ "$EVENT" = "issues" ] && [ "$ACTION" = "closed" ]; then
            MSG="✅ **이슈 완료**: [${{ gitea.event.issue.title }}](${{ gitea.event.issue.html_url }})"
          elif [ "$EVENT" = "pull_request" ] && [ "$ACTION" = "opened" ]; then
            MSG="🔀 **PR 오픈**: [${{ gitea.event.pull_request.title }}](${{ gitea.event.pull_request.html_url }})"
          elif [ "$EVENT" = "pull_request" ] && [ "$ACTION" = "closed" ]; then
            MSG="🎉 **PR 머지**: [${{ gitea.event.pull_request.title }}](${{ gitea.event.pull_request.html_url }})"
          fi

          [ -n "$MSG" ] && curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"$MSG\n저장소: \`${{ gitea.repository }}\`\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

---

## 12. 사용자 관리 및 권한 설정

### 12-1. 사용자 계정 생성 (관리자)

팀원 계정을 관리자가 직접 생성하는 방법입니다.

```
Site Administration → User Management → [Create User Account]

Username   : dev-kim
Email      : dev-kim@local.dev
Password   : (임시 비밀번호 — 팀원에게 전달 후 변경 요청)
```

또는 팀원이 직접 가입 후 관리자가 승인:

```
Site Administration → User Management → 가입 대기 사용자 확인
```

### 12-2. 사용자 권한 수준

| 권한 | 설명 | 설정 위치 |
|------|------|-----------|
| **Site Admin** | 전체 서버 관리 | Site Administration → User Management |
| **Organization Owner** | 조직 전체 관리 | Organization → Teams → Owners |
| **Team Write** | 저장소 push · PR · 이슈 | Organization → Teams |
| **Team Read** | 저장소 열람만 | Organization → Teams |
| **Repository Collaborator** | 특정 저장소 개별 권한 | 저장소 → Settings → Collaborators |

### 12-3. 팀원 추가 절차

```
1. 관리자가 계정 생성 또는 팀원 자가 가입
2. Organization → Teams → Developers → Members → [Add Member]
3. 팀원이 SSH 키 등록 (저장소 SSH 클론용)
4. 로컬 git 설정 안내:
```

**팀원에게 안내할 로컬 git 초기 설정:**

```bash
# Git 사용자 정보 설정
git config --global user.name "Kim Jong-in"
git config --global user.email "dev-kim@local.dev"

# Gitea 서버 기본 URL 설정 (HTTPS 인증 정보 캐시)
git config --global credential.helper store

# 저장소 Clone (첫 회 ID/PW 입력 후 자동 저장)
git clone http://192.168.93.138:3001/my-dev-team/my-app.git
```

---

## 13. 팀 협업 실전 운영 가이드

### 13-1. 브랜치 전략 (Git Flow 간소화)

```
main          ← 운영 배포 브랜치 (직접 push 금지 — 브랜치 보호 필수)
  └── develop ← 개발 통합 브랜치
        ├── feature/이슈번호-기능명   예) feature/42-user-auth
        ├── bugfix/이슈번호-버그명    예) bugfix/55-login-error
        └── hotfix/이슈번호-긴급명   예) hotfix/60-payment-fix
```

**브랜치 생성 규칙:**

```bash
# 이슈 #42 기능 개발 브랜치 생성
git checkout develop
git pull origin develop
git checkout -b feature/42-user-auth

# 작업 후 커밋
git add .
git commit -m "feat: #42 사용자 로그인 JWT 인증 구현"

# push
git push origin feature/42-user-auth
```

### 13-2. 커밋 메시지 컨벤션

```
[타입]: #이슈번호 작업 내용 요약

타입 목록:
  feat     : 새 기능 추가
  fix      : 버그 수정
  docs     : 문서 수정
  style    : 코드 포맷팅
  refactor : 코드 리팩터링
  test     : 테스트 코드
  infra    : 인프라·빌드 설정
  chore    : 기타 (패키지 업데이트 등)
```

**예시:**

```bash
git commit -m "feat: #42 사용자 로그인 JWT 인증 구현"
git commit -m "fix: #55 로그인 페이지 무한 리다이렉트 수정"
git commit -m "docs: #60 API 명세서 Swagger 추가"
```

> 커밋 메시지에 `#이슈번호`를 포함하면 Gitea에서 해당 이슈와 커밋이 **자동으로 연결**됩니다.

### 13-3. 스프린트 운영 사이클

```
월요일 (스프린트 시작)
  └── 스프린트 계획 회의
       ├── 마일스톤 생성 또는 선택 (Sprint N)
       ├── 이슈 생성 및 우선순위 설정
       ├── 마일스톤에 이슈 할당
       └── 담당자(Assignee) 배정

매일 (데일리 스탠드업)
  └── Issues 탭의 Open 이슈 기준으로 현황 공유
       ├── 진행 중 이슈 상태 공유
       ├── 블로킹(blocked) 이슈 공유
       └── 오늘 진행할 이슈 확인

금요일 (스프린트 종료)
  └── 마일스톤 페이지에서 진척도 확인
       ├── Closed 이슈 비율 확인
       ├── 미완료 이슈 → 다음 마일스톤으로 이전
       └── 다음 스프린트 마일스톤 생성
```

### 13-4. 이슈 검색 필터 활용

```
저장소 → Issues 탭 검색창에서 필터 조합:

is:open assignee:@me           ← 내가 담당한 열린 이슈
is:open label:bug              ← 열린 버그 이슈
milestone:Sprint-1 is:open     ← Sprint 1의 미완료 이슈
is:closed label:feature        ← 완료된 기능 개발 이슈
```

---

## 14. 운영 치트시트

### 서비스 관리 명령어

```bash
# Gitea 서비스 상태 확인
cd ~/gitea && sudo docker compose ps

# Gitea 시작 / 중지 / 재시작
sudo docker compose up -d
sudo docker compose down
sudo docker compose restart

# Gitea 로그 확인
sudo docker compose logs -f gitea
sudo docker compose logs -f gitea --tail=50

# Act Runner 상태 확인
sudo systemctl status act-runner
sudo systemctl restart act-runner
journalctl -u act-runner -n 50
```

### 전체 시스템 상태 한 번에 확인

```bash
echo "=== Gitea ==="
cd ~/gitea && sudo docker compose ps

echo ""
echo "=== Act Runner ==="
sudo systemctl status act-runner --no-pager | grep -E "Active|Loaded"

echo ""
echo "=== 포트 리스닝 ==="
sudo netstat -tulpn | grep -E "3001|2222"

echo ""
echo "=== 디스크 사용량 ==="
du -sh ~/gitea/
```

### 정기 백업

```bash
#!/bin/bash
# /opt/scripts/gitea-backup.sh
# cron: 0 2 * * * /opt/scripts/gitea-backup.sh

BACKUP_DIR="/opt/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Gitea 데이터 백업 (저장소 · 설정 · DB)
tar -czf $BACKUP_DIR/gitea_data_$DATE.tar.gz ~/gitea/data ~/gitea/config
tar -czf $BACKUP_DIR/gitea_db_$DATE.tar.gz ~/gitea/db

# 30일 이상 된 백업 자동 삭제
find $BACKUP_DIR -name "gitea_*.tar.gz" -mtime +30 -delete

echo "[$DATE] Gitea 백업 완료: $BACKUP_DIR"
```

```bash
# 실행 권한 부여 및 cron 등록
chmod +x /opt/scripts/gitea-backup.sh
crontab -e
# 0 2 * * * /opt/scripts/gitea-backup.sh >> /var/log/gitea-backup.log 2>&1
```

### Gitea CLI (Gitea 내장 도구)

```bash
# 컨테이너 내부에서 관리 명령 실행
sudo docker exec -it gitea bash

# 관리자 비밀번호 재설정 (컨테이너 내부)
gitea admin user change-password --username admin --password NewPassword123!

# 사용자 목록 확인
gitea admin user list

# 저장소 인덱스 재생성
gitea admin index-repos
```

---

## 15. 문제 해결 (Troubleshooting)

### ❌ 웹 페이지에 접속되지 않을 때

```bash
# 1. 컨테이너 실행 상태 확인
sudo docker compose ps
sudo docker compose logs gitea --tail=30

# 2. 포트 리스닝 확인
sudo netstat -tulpn | grep 3001

# 3. 방화벽 확인
sudo firewall-cmd --list-ports

# 4. 컨테이너 재시작
cd ~/gitea && sudo docker compose restart
```

---

### ❌ SSH clone 시 "Connection refused" 오류

```bash
# 1. SSH 포트(2222) 방화벽 허용 확인
sudo firewall-cmd --list-ports | grep 2222

# 2. 미허용 시 추가
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload

# 3. 컨테이너 포트 매핑 확인
sudo docker ps | grep gitea

# 4. SSH 연결 테스트
ssh -T git@<서버_IP> -p 2222 -v
```

---

### ❌ Push 시 "403 Forbidden" 오류

```bash
# 원인: 브랜치 보호 규칙에 의해 직접 push 차단됨
# → 해당 브랜치로 PR을 생성하여 머지해야 합니다.

# 원인 2: 사용자 권한 부족
# → Organization Team의 권한이 Write 이상인지 확인
# Organization → Teams → 해당 팀 → Permission: Write 확인

# 원인 3: HTTPS 인증 정보 오류
git config --global credential.helper store
git pull   # ID/PW 재입력 후 자동 저장
```

---

### ❌ Gitea Actions 워크플로우가 실행되지 않을 때

```bash
# 1. Actions 기능 활성화 확인
# Site Administration → Settings → Repository Settings
# → Enable Repository Actions: 체크 확인

# 2. Act Runner 상태 확인
sudo systemctl status act-runner
journalctl -u act-runner -n 50

# 3. Runner 등록 상태 확인
# Site Administration → Actions → Runners
# → 등록된 Runner가 Idle 상태인지 확인

# 4. 워크플로우 파일 경로 확인
# → .gitea/workflows/*.yml  (GitHub: .github/workflows/)
ls -la .gitea/workflows/
```

---

### ❌ Mattermost Webhook 전송이 되지 않을 때

```bash
# 1. Webhook 허용 주소 등록 확인 (폐쇄망에서 필수)
# Site Administration → Settings → Webhook Settings
# → Allowed Hosts에 Mattermost 서버 IP 등록 확인

# 2. Webhook URL 직접 테스트 (Gitea 서버에서)
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{"text": "Gitea Webhook 테스트"}' \
  http://192.168.93.138:8065/hooks/xxxxxxxxxx

# 3. Gitea Webhook 최근 전송 기록 확인
# 저장소 → Settings → Webhooks → 해당 Webhook → [Recent Deliveries]
# → 전송 기록과 응답 코드(200이면 정상) 확인
```

---

### ❌ 서버 재부팅 후 Gitea가 자동 시작되지 않을 때

```bash
# docker-compose.yml에 restart: always가 설정되어 있어야 함
grep "restart" ~/gitea/docker-compose.yml
# restart: always 출력 확인

# Docker 서비스 자동 시작 활성화
sudo systemctl enable docker

# 수동으로 다시 시작
cd ~/gitea && sudo docker compose up -d
```

---

## 📚 참고 자료

| 자료 | URL |
|------|-----|
| Gitea 공식 문서 | https://docs.gitea.com |
| Gitea Docker 설치 가이드 | https://docs.gitea.com/installation/install-with-docker |
| Act Runner 릴리즈 | https://gitea.com/gitea/act_runner/releases |
| Gitea Actions 문서 | https://docs.gitea.com/usage/actions/overview |
| Gitea Webhook 문서 | https://docs.gitea.com/usage/webhooks |

---

## 🗺️ 구축 완료 현황

```
✅ Gitea Self-Hosted 설치        (폐쇄망 Git 서버)
   ├── Organization · Team 구성
   ├── 저장소 생성 · 브랜치 보호 설정
   ├── 이슈 · 마일스톤 · 라벨 설정
   ├── Pull Request · 코드 리뷰 운영
   ├── Gitea Actions CI/CD 파이프라인
   └── Mattermost Webhook 알림 연동

다음 구축 단계 (참고):
⬜ Mattermost Self-Hosted 구축   (팀 소통 채널)
⬜ Wiki.js Self-Hosted 구축      (기술 문서 위키)
⬜ 전체 통합 — Webhook 연동      (자동화 완성)
```

---

<div align="center">

**본 문서는 폐쇄망 환경의 소프트웨어 개발 프로젝트 무료 Self-Hosted 인프라 구축을 목적으로 작성되었습니다.**  
`Oracle Linux 8.10` · `Docker` · `Gitea` · `Gitea Actions` · `Act Runner` · `Mattermost Webhook`

*작성일: 2026-04-15*  
*작성자: Kim Jong-in (Technical Architect)*

</div>
