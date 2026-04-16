---
layout: post
title:  "사설 클라우드 인프라 구축 가이드"
date:   2026-04-13 12:00:00 +0900
---

# 📦 Phase 1: 사설 클라우드 인프라 구축 가이드

> **Proxmox VE · Ubuntu 22.04 · K3s · 4-VM 아키텍처**

| 항목 | 내용 |
|------|------|
| **버전** | v1.1 (2026-04-16 작성) |
| **대상** | Proxmox VE 9.x 환경 / Ubuntu 22.04 LTS |
| **사전 조건** | Proxmox VE 설치 완료 / 물리 서버 전용 확보 |
| **소요 시간** | 4 ~ 8 시간 (환경에 따라 상이) |

---

## 1. Phase 1 개요 및 목표

본 Phase는 이후 모든 서비스 배포의 기반이 되는 로컬 클라우드 인프라를 구성합니다.  
물리 서버 한 대를 Proxmox VE 하이퍼바이저로 운영하고, 그 위에 4개의 독립된 Ubuntu VM을 올려 **1대의 Master(vm-dev)와 3대의 Worker(vm-ai, vm-collab, vm-monitor)로 구성된 K3s 멀티 노드 클러스터**를 구축합니다.  
마지막으로 외부 PC(Windows)에서 **Lens GUI 도구**를 통해 전체 클러스터를 통합 관제합니다.

### 1-1. 아키텍처 개요

아래는 물리 서버 한 대 위에 구성되는 전체 레이어입니다.

| 레이어 | 기술 | 역할 |
|--------|------|------|
| **하이퍼바이저** | Proxmox VE 9.1 | 물리 서버 → 4개 독립 VM으로 분리 |
| **운영체제 (Guest OS)** | Ubuntu 22.04 LTS | 각 VM의 기반 OS |
| **컨테이너 오케스트레이션** | K3s (경량 Kubernetes) | Master/Worker 멀티 노드 클러스터 구성 |
| **통합 관제 도구** | Lens (외부 PC) | 전체 노드 및 리소스 GUI 모니터링 |

### 1-2. 클러스터 구성 구조

```
[ 물리 서버 - Proxmox VE ]
│
├── VM 100: vm-dev      → K3s Master (control-plane)
├── VM 101: vm-ai       → K3s Worker
├── VM 102: vm-collab   → K3s Worker
└── VM 103: vm-monitor  → K3s Worker
         ↑
    외부 Windows PC에서 Lens로 통합 관제
```

### 1-3. 구축 전략: 복제(Clone) 기반 효율화

4개의 VM을 처음부터 각각 설치하는 대신, vm-dev(VM 100)를 완벽히 구성한 뒤 나머지 3개를 Full Clone으로 복제합니다.  
설치 시간을 1/4로 줄이고 설정 오류 가능성도 최소화합니다.

| 단계 | 내용 |
|------|------|
| **STEP 1** | **vm-dev (VM 100) 생성 및 완전 설정** — OS 설치 → K3s 설치 → 기본 패키지 설정 |
| **STEP 2** | **vm-dev → 나머지 VM 복제 (Full Clone)** — vm-ai(101) · vm-collab(102) · vm-monitor(103) |
| **STEP 3** | **복제된 각 VM 개별화** — 고정 IP / 호스트네임 / K3s 노드 ID를 VM별로 변경 |

> 💡 **TIP**: Proxmox의 Full Clone은 디스크 이미지 전체를 복사하므로 원본 VM과 완전히 독립적입니다. Linked Clone과 달리 원본 삭제 후에도 정상 동작합니다.

---

## 2. 인프라 설계 사양

### 2-1. VM 구성표

실제 Proxmox 화면에서 확인되는 4개 VM의 사양입니다. (mycloud 노드, VM 100~103)

| VM 번호 | 호스트명 | K3s 역할 | 주요 역할 | CPU | RAM | Disk |
|---------|---------|---------|---------|-----|-----|------|
| 100 | vm-dev | **Master** (control-plane) | 원본 템플릿 / 개발 환경 | 2 Core | 4 GB | 30 GB |
| 101 | vm-ai | **Worker** | AI 서비스 (Ollama 등) | 2 Core | 8 GB | 96 GB+ |
| 102 | vm-collab | **Worker** | 협업 도구 (Gitea, DB) | 2 Core | 4 GB | 30 GB |
| 103 | vm-monitor | **Worker** | 관제 시스템 (Netdata) | 2 Core | 4 GB | 30 GB |

### 2-2. 고정 IP 설계

각 VM에는 네트워크 대역에 맞는 고정 IP를 할당합니다. 아래 표는 예시이며, 본인 공유기 환경에 맞게 조정하세요.

| VM 번호 | 호스트명 | 권장 고정 IP | 비고 |
|---------|---------|------------|------|
| 100 | vm-dev | 172.28.136.100 | Master / 원본 |
| 101 | vm-ai | 172.28.136.101 | Worker — AI 서비스 |
| 102 | vm-collab | 172.28.136.102 | Worker — 협업 도구 |
| 103 | vm-monitor | 172.28.136.103 | Worker — 관제 시스템 |

> ⚠️ **주의**: 공유기 DHCP 할당 범위와 충돌하지 않는 IP 대역을 선택하세요. 충돌 시 네트워크 연결이 끊길 수 있습니다.

---

## 3. 기초 VM (vm-dev) 생성 및 K3s 설치

vm-dev는 나머지 3개 VM의 원본(마스터 이미지) 역할을 합니다. 이 단계를 가장 꼼꼼하게 진행하세요.

### 3-1. Proxmox에서 VM 생성

Proxmox 웹 UI (`https://<서버IP>:8006`) 에 접속 후 우측 상단 [VM 생성] 버튼을 클릭합니다.

| 설정 탭 | 항목 | 입력값 |
|--------|------|--------|
| 일반 (General) | VM ID / Name | 100 / vm-dev |
| OS | ISO 이미지 | Ubuntu 22.04 LTS Server |
| 시스템 (System) | BIOS / SCSI 컨트롤러 | SeaBIOS / VirtIO SCSI |
| 디스크 (Disks) | 디스크 크기 | 30 GB |
| CPU | 코어 수 | 2 Core |
| 메모리 (Memory) | RAM | 4096 MB (4 GB) |
| 네트워크 | 모델 | VirtIO (Para-virtualized) |

> 💡 **TIP**: Proxmox 좌측 트리에서 mycloud → 우클릭 → [VM 생성]을 선택해도 됩니다. VM ID는 100번부터 순서대로 부여하면 관리가 편리합니다.

### 3-2. Ubuntu 22.04 LTS Server 설치

VM 콘솔(>_ 콘솔 탭)에서 Ubuntu 설치를 진행합니다. 아래 항목에 주의하세요.

| 설치 옵션 | 권장 설정 | 이유 |
|----------|----------|------|
| 설치 유형 | Ubuntu Server (Minimized 해제) | 필수 패키지 포함을 위해 |
| 네트워크 | 일단 DHCP로 진행 | 고정 IP는 설치 후 별도 설정 |
| 스토리지 | LVM 사용 (기본값 유지) | 나중에 growpart로 파티션 확장 가능 |
| OpenSSH | ✅ 설치 선택 | 이후 원격 접속을 위해 필수 |

### 3-3. K3s Master 설치 (vm-dev)

Ubuntu 설치 완료 후 아래 명령어로 K3s를 **Master 노드**로 설치합니다.

```bash
# K3s 설치 (인터넷 연결 필요)
curl -sfL https://get.k3s.io | sh -

# kubectl 접근 권한 설정
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# 설치 확인
sudo kubectl get nodes

# 편의를 위한 alias 설정 (선택)
echo 'alias k="sudo kubectl"' >> ~/.bashrc && source ~/.bashrc
```

> 💡 **TIP**: `kubectl get nodes` 결과에서 vm-dev가 `STATUS: Ready`, `ROLES: control-plane,master`로 나오면 K3s Master 설치가 정상 완료된 것입니다.

---

## 4. VM 복제 및 리소스 확장 (Scale-out)

vm-dev 구성이 완료되면 이를 원본으로 나머지 3개의 VM을 복제합니다.

### 4-1. Full Clone 생성

Proxmox 좌측 트리에서 100 (vm-dev) 를 우클릭 → [복제(Clone)] 를 선택합니다.

| 복제 대상 | VM ID | 이름 | Clone 방식 |
|----------|-------|------|-----------|
| vm-ai | 101 | vm-ai | Full Clone |
| vm-collab | 102 | vm-collab | Full Clone |
| vm-monitor | 103 | vm-monitor | Full Clone |

> ⚠️ **주의**: 반드시 'Full Clone'을 선택하세요. Linked Clone은 원본 VM에 의존적이어서 원본 삭제 시 문제가 발생합니다.

### 4-2. 디스크 및 메모리 리소스 조정

복제된 VM의 Proxmox [하드웨어] 탭에서 디스크와 메모리를 용도에 맞게 조정합니다.

| VM | 변경 항목 | 조정 방법 |
|----|----------|----------|
| 101 (vm-ai) | 디스크 96GB+, RAM 8GB | 하드웨어 탭 → 디스크/메모리 → 수정 |
| 102 (vm-collab) | 디스크 30GB, RAM 4GB | 기본 복제값 유지 가능 |
| 103 (vm-monitor) | 디스크 30GB, RAM 4GB | 기본 복제값 유지 가능 |

### 4-3. OS 파티션 확장

디스크를 증설한 경우(특히 vm-ai), 반드시 OS 내부에서 파티션을 확장해야 늘어난 용량을 실제로 사용할 수 있습니다.

```bash
# 파티션 확장 (디스크 증설 시 필수 - vm-ai에서 실행)
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# 확장 결과 확인
df -h /
```

> 💡 **TIP**: `growpart` 명령 실행 후 'CHANGED' 메시지가 나오면 성공입니다. `resize2fs`까지 완료해야 실제 파일시스템 크기가 늘어납니다.

---

## 5. 고정 IP 설정 및 네트워크 충돌 방지 ★ 핵심

복제된 VM들이 원본(vm-dev)과 동일한 네트워크 설정을 갖지 않도록 '네트워크 고정'과 'Cloud-init 간섭 차단'을 수행합니다.  
이 단계를 건너뛰면 VM 재부팅 시 IP가 원복되거나 충돌이 발생합니다.

### 5-1. Cloud-init 네트워크 관리 비활성화

Cloud-init이 재부팅 시 netplan 설정을 덮어쓰지 못하도록 네트워크 관리 기능을 끕니다.

```bash
# Cloud-init 네트워크 간섭 차단 설정 파일 생성
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

파일에 아래 내용을 입력 후 저장 (`Ctrl+X` → `Y` → `Enter`):

```
network: {config: disabled}
```

### 5-2. 고정 IP 할당

각 VM별로 아래 절차를 진행합니다. IP 주소만 VM에 맞게 변경하면 됩니다.

```bash
# netplan 설정 파일 열기
sudo nano /etc/netplan/50-cloud-init.yaml
```

아래 예시와 같이 `addresses` 값을 해당 VM의 고정 IP로 수정합니다:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses: [172.28.136.101/24]   # ← VM에 맞게 변경 (예: vm-ai는 .101)
      gateway4: 172.28.136.1            # ← 본인 공유기 게이트웨이
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
# 설정 적용
sudo netplan apply

# IP 적용 확인
ip addr show ens18
```

| VM | addresses 값 |
|----|-------------|
| vm-dev (100) | 172.28.136.100/24 |
| vm-ai (101) | 172.28.136.101/24 |
| vm-collab (102) | 172.28.136.102/24 |
| vm-monitor (103) | 172.28.136.103/24 |

---

## 6. 호스트네임 및 쿠버네티스 노드 갱신

복제된 VM이 원본(vm-dev)과 동일한 호스트네임과 K3s 노드 ID를 가지면 충돌이 발생합니다.  
각 복제 VM에서 아래 절차를 반드시 실행합니다.

### 6-1. 시스템 호스트네임 변경

각 VM에서 호스트명을 해당 VM의 이름으로 변경합니다. 아래는 vm-ai(101)에서 실행하는 예시입니다.

```bash
# 호스트네임 변경 (예: vm-ai)
sudo hostnamectl set-hostname vm-ai

# /etc/hosts 파일에서 내부 vm-dev 명칭도 교체
sudo nano /etc/hosts
# → '127.0.1.1  vm-dev' 를 '127.0.1.1  vm-ai' 로 수정
```

| VM | 실행 명령어 |
|----|-----------|
| vm-ai (101) | `sudo hostnamectl set-hostname vm-ai` |
| vm-collab (102) | `sudo hostnamectl set-hostname vm-collab` |
| vm-monitor (103) | `sudo hostnamectl set-hostname vm-monitor` |

### 6-2. K3s 노드 정보 초기화

복제된 노드가 기존 노드와 중복 인식되지 않도록 고유 노드 ID를 초기화합니다.  
이 작업은 복제된 모든 VM(vm-ai, vm-collab, vm-monitor)에서 각각 실행합니다.

```bash
# K3s 서비스 중지
sudo systemctl stop k3s

# 노드 고유 ID 및 패스워드 파일 삭제
sudo rm /etc/rancher/node/node-id /etc/rancher/node/password

# K3s 재시작 (새 ID로 자동 재생성)
sudo systemctl start k3s
```

이후 vm-dev(원본)에서 구형 이름의 노드가 남아있는 경우 삭제합니다:

```bash
# vm-dev 에서 실행 - 구형 노드 이름 정리
kubectl delete node vm-dev
```

> ⚠️ **중요**: `node-id` 파일을 삭제하지 않으면 K3s 클러스터에서 중복 노드 충돌이 발생합니다. 복제된 VM마다 반드시 실행하세요.

---

## 7. Master 노드 외부 노출 설정 (Lens 연동을 위한 필수 작업)

외부 Windows PC에서 Lens로 클러스터에 접속하려면, Master 노드(vm-dev)가 외부 IP를 허용하도록 설정해야 합니다.

### 7-1. K3s 서비스 설정 수정

`/etc/systemd/system/k3s.service` 파일의 `ExecStart` 섹션에 `--tls-san` 옵션을 추가합니다.

```bash
sudo nano /etc/systemd/system/k3s.service
```

`ExecStart` 라인에 다음 옵션을 추가합니다:

```
ExecStart=/usr/local/bin/k3s server --tls-san=172.28.136.100
```

> ⚠️ `172.28.136.100` 부분을 vm-dev의 실제 고정 IP로 변경하세요.

설정 변경 후 서비스 재시작:

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

### 7-2. Kubeconfig 추출 및 외부 접속용 가공 (Lens 등록용)

vm-dev의 `/etc/rancher/k3s/k3s.yaml` 파일을 기반으로 Lens에 등록할 설정 파일을 생성합니다.

```bash
# 원본 kubeconfig 내용 확인
sudo cat /etc/rancher/k3s/k3s.yaml
```

이 파일을 복사하여 아래 두 곳을 수정합니다:

1. **server 주소 변경**:  
   `https://127.0.0.1:6443` → `https://172.28.136.100:6443`

2. **TLS 인증 우회 설정 추가** (`cluster:` 섹션 하위):  
   `insecure-skip-tls-verify: true` 를 추가합니다.

3. **주의사항**: `insecure-skip-tls-verify` 사용 시, **기존 `certificate-authority-data` 라인은 반드시 삭제**해야 접속 충돌이 발생하지 않습니다.

수정된 kubeconfig 예시:

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true          # ← 추가
    # certificate-authority-data: ...       # ← 이 라인 삭제
    server: https://172.28.136.100:6443     # ← IP로 변경
  name: default
...
```

이 파일을 Windows PC로 복사한 후 Lens에 등록합니다.

---

## 8. Worker 노드 합류 (K3s Cluster Join)

### 8-1. Master 노드에서 토큰 확인

Worker 노드가 클러스터에 합류하려면 Master의 Join Token이 필요합니다.

```bash
# vm-dev (Master)에서 실행
sudo cat /var/lib/rancher/k3s/server/node-token
```

출력된 토큰 값을 복사해둡니다.

### 8-2. Worker 노드 합류 명령어

vm-ai, vm-collab, vm-monitor 각각에서 아래 명령어를 실행합니다.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://[VM1_IP]:6443 K3S_TOKEN=[YOUR_TOKEN] sh -
```

예시 (실제 IP와 토큰으로 치환):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://172.28.136.100:6443 K3S_TOKEN=K10abc...xyz sh -
```

### 8-3. 🛠️ [중요] 워커 노드 조인 트러블슈팅 — "샌드위치 방식" 초기화

워커 노드 합류 시 기존 K3s 잔재로 인해 서비스가 바로 올라오지 않을 수 있습니다.  
이때는 아래 **3단계 샌드위치 방식** 초기화가 필요합니다.

#### 1단계: 1차 삭제 시도

```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

> 최초 실행 시 파일이 없어 에러가 날 수 있으나, 무시하고 다음 단계로 진행합니다.

#### 2단계: 설치 시도 및 언인스톨 스크립트 확보

아래 합류 명령어를 실행합니다. 이 과정에서 이후 삭제에 필요한 언인스톨 스크립트가 시스템에 자동 생성됩니다.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://[VM1_IP]:6443 K3S_TOKEN=[YOUR_TOKEN] sh -
```

#### 3단계: 2차 삭제 및 최종 재설치

```bash
# 이제 언인스톨 스크립트가 존재하므로 정상 삭제됨
sudo /usr/local/bin/k3s-agent-uninstall.sh

# 최종 재설치 — 에러 없이 "Starting k3s-agent" 가 완료됨
curl -sfL https://get.k3s.io | K3S_URL=https://[VM1_IP]:6443 K3S_TOKEN=[YOUR_TOKEN] sh -
```

---

## 9. 최종 동작 확인

### 9-1. 각 VM에서 개별 노드 상태 확인

> ⚠️ **중요**: 아래 명령어는 **각 VM에 개별로 접속하여 실행**합니다.  
> 각 VM은 독립적인 K3s 인스턴스가 아닌 하나의 클러스터에 속하지만, **접속한 VM 기준으로 클러스터 전체 노드 상태**를 보여줍니다.  
> Master 노드(vm-dev)에서 실행할 경우 전체 4개 노드를 한번에 확인할 수 있습니다.

```bash
# Master 노드(vm-dev) 또는 각 VM에서 실행
kubectl get nodes -o wide
```

**vm-dev(Master)에서 실행 시 정상 출력 예시:**

```
NAME          STATUS   ROLES                  AGE   VERSION        INTERNAL-IP
vm-dev        Ready    control-plane,master   10d   v1.29.x+k3s1   172.28.136.100
vm-ai         Ready    <none>                 5d    v1.29.x+k3s1   172.28.136.101
vm-collab     Ready    <none>                 5d    v1.29.x+k3s1   172.28.136.102
vm-monitor    Ready    <none>                 5d    v1.29.x+k3s1   172.28.136.103
```

> 💡 **참고**: Worker 노드(vm-ai, vm-collab, vm-monitor)의 `ROLES`는 `<none>`으로 표시됩니다.  
> `control-plane,master`는 Master 노드(vm-dev)에만 표시됩니다.

**각 Worker VM에서 개별 실행 시**: 동일하게 클러스터 전체 4개 노드가 출력됩니다. (단, Worker VM에서 `kubectl` 실행 시 kubeconfig 경로 지정이 필요할 수 있습니다.)

### 9-2. 확인 기준

- ▸ `vm-dev`의 ROLES: `control-plane,master`
- ▸ 나머지 3개 노드의 ROLES: `<none>` (Worker 노드)
- ▸ 모든 노드의 STATUS: `Ready`
- ▸ INTERNAL-IP가 각각 고유한 IP 주소 (중복 없음)

### 9-3. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| STATUS가 NotReady | K3s 서비스 미실행 | `sudo systemctl restart k3s` |
| IP가 중복됨 | netplan 미적용 또는 cloud-init 간섭 | 5단계 절차 재확인 |
| 이름이 vm-dev로 동일 | 호스트네임 미변경 | 6-1 절차 재실행 후 K3s 재시작 |
| Worker ROLES가 `control-plane,master`로 표시 | 노드 ID 초기화 미수행 | 6-2 절차 재실행 |
| Worker 노드 조인 실패 | 기존 K3s 잔재 | 8-3 샌드위치 방식 초기화 수행 |

---

## 10. 통합 관제 — Lens 설정 및 확인

### 10-1. Lens란?

Lens는 쿠버네티스 클러스터를 시각적으로 관리하는 외부 GUI 도구입니다.  
Windows PC에 설치 후 vm-dev의 kubeconfig를 등록하면, 모든 노드의 상태를 한눈에 모니터링할 수 있습니다.

> 💡 **왜 Lens가 필요한가?**  
> `kubectl get nodes -o wide`는 VM에 직접 SSH 접속해서 실행해야 하며, 각 VM마다 개별 확인이 번거롭습니다.  
> Lens를 사용하면 **외부 PC에서 전체 클러스터를 실시간으로 GUI로 확인**할 수 있어 관제 효율이 크게 향상됩니다.

### 10-2. Lens 설치 및 클러스터 등록

1. Windows PC에서 [Lens 공식 사이트](https://k8slens.dev/)에 접속하여 설치합니다.
2. 7-2에서 가공한 kubeconfig 파일을 Windows PC로 복사합니다.
3. Lens 실행 → 좌측 상단 `+` (Add Cluster) 클릭 → kubeconfig 파일 경로 지정 또는 내용 붙여넣기.
4. 클러스터 등록 완료 후 연결을 확인합니다.

### 10-3. Lens에서 정상 상태 확인

모든 노드가 정상 합류하면 Lens의 **Nodes** 메뉴에서 다음과 같이 클러스터 구성이 확인됩니다.

**관제 리스트 (4 Items):**

| 노드명 | 역할 | 상태 |
|--------|------|------|
| vm-dev | control-plane, master | ✅ Ready (초록색) |
| vm-ai | Worker (리소스 제공) | ✅ Ready (초록색) |
| vm-collab | Worker (리소스 제공) | ✅ Ready (초록색) |
| vm-monitor | Worker (리소스 제공) | ✅ Ready (초록색) |

**최종 확인 기준:**

- ▸ Lens Nodes 메뉴에 4개 노드 표시 확인
- ▸ 모든 노드의 `Conditions`가 **Ready (초록색)** 임을 확인
- ▸ vm-dev만 `control-plane, master` 역할 표시, 나머지는 Worker로 표시

---

## 11. 스냅샷 촬영 — Phase 1 완료 상태 보존

구축이 완료된 최적의 상태를 보존하기 위해 Proxmox에서 각 VM의 스냅샷을 생성합니다.

### 11-1. Proxmox 스냅샷 생성 절차

| 단계 | 내용 |
|------|------|
| **STEP 1** | Proxmox 좌측 트리에서 스냅샷을 찍을 VM(100~103) 선택 |
| **STEP 2** | 상단 메뉴 탭 중 '스냅샷' 선택 |
| **STEP 3** | [스냅샷 생성] 버튼 클릭 |
| **STEP 4** | 아래 네이밍 규칙을 따라 이름 및 설명 입력 후 저장 |

### 11-2. 스냅샷 네이밍 규칙

| VM | 스냅샷 이름 (예시) | 설명 |
|----|-----------------|------|
| vm-dev (100) | `vm-dev-Base_k3s_Phase1` | 최초 VM 생성 (K3s 포함) |
| vm-ai (101) | `vm-ai-Base_k3s_Phase1` | 최초 VM 생성 (K3s 포함) |
| vm-collab (102) | `vm-collab-Base_k3s_Phase1` | 최초 VM 생성 (K3s 포함) |
| vm-monitor (103) | `vm-monitor-Base_k3s_Phase1` | 최초 VM 생성 (K3s 포함) |

> 💡 **TIP**: 스냅샷은 언제든 [롤백] 버튼으로 해당 시점의 상태로 되돌릴 수 있습니다. Phase 2 작업 전에도 반드시 스냅샷을 찍어두는 습관을 들이세요.

---

## 12. Phase 1 완료 체크리스트

아래 항목을 모두 확인한 뒤 Phase 2로 넘어가세요.

### 12-1. VM 생성 및 OS

- ☐ Proxmox 웹 UI (`https://<서버IP>:8006`) 접속 성공
- ☐ VM 100 (vm-dev) 생성 완료 — Ubuntu 22.04 + K3s Master 설치
- ☐ VM 101 (vm-ai) 생성 완료 — Ubuntu 22.04 + K3s Worker 합류
- ☐ VM 102 (vm-collab) 생성 완료 — Ubuntu 22.04 + K3s Worker 합류
- ☐ VM 103 (vm-monitor) 생성 완료 — Ubuntu 22.04 + K3s Worker 합류

### 12-2. 네트워크

- ☐ 각 VM에 고정 IP 할당 완료 (IP 중복 없음)
- ☐ Cloud-init 네트워크 간섭 차단 설정 완료
- ☐ 재부팅 후 고정 IP 유지 확인

### 12-3. K3s 클러스터

- ☐ vm-dev에서 `kubectl get nodes` → 4개 노드 모두 `STATUS: Ready` 확인
- ☐ vm-dev의 ROLES: `control-plane,master` 확인
- ☐ vm-ai / vm-collab / vm-monitor의 ROLES: `<none>` (Worker) 확인
- ☐ 각 노드의 INTERNAL-IP가 고유하게 표시됨 확인

### 12-4. 외부 관제 (Lens)

- ☐ vm-dev의 K3s 서비스 파일에 `--tls-san=[VM1_IP]` 추가 완료
- ☐ kubeconfig 파일 가공 완료 (server IP 변경, `insecure-skip-tls-verify: true` 추가, `certificate-authority-data` 삭제)
- ☐ Windows PC의 Lens에 클러스터 등록 완료
- ☐ Lens Nodes 메뉴에서 4개 노드 모두 `Ready (초록색)` 확인

### 12-5. 스냅샷

- ☐ VM 100 (vm-dev) 스냅샷 생성 완료
- ☐ VM 101 (vm-ai) 스냅샷 생성 완료
- ☐ VM 102 (vm-collab) 스냅샷 생성 완료
- ☐ VM 103 (vm-monitor) 스냅샷 생성 완료

---

## ✅ Phase 1 완료

**다음 단계**: Phase 2 — vm-dev에 Gitea (소스 관리) + Wiki.js (문서 관리) 배포

---

*작성일: 2026-04-16 | 버전: v1.1*
