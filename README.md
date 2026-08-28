# vk8s — k8s capacity studio

가상 Kubernetes 환경(IDC / Cloud / Hybrid)을 설계하고, GPU VRAM까지 인지하는
스케줄 시뮬레이션 · 의존성 그래프 · 정적 병목 분석 · 서비스 준비 체크리스트 ·
K8s YAML / Helm / Docker Compose / Project 리포트 산출을 한 페이지에서 수행하는 **브라우저 단독** 도구.

- **백엔드 없음** · 모든 상태는 `localStorage`. 의도적 Export 외에 외부 통신 0.
- **즉시 검증** · 매 편집 후 300ms 디바운스로 validator 재실행.
- **하나의 데이터 · 여덟 개의 렌즈** · Topology / Architecture / Capacity / Dependency / Namespace / Bottleneck / Checklist / Export 모두 같은 JSON을 다른 관점으로 본다.

라이브 데모: `https://aiotool.net/vk8s/` (또는 로컬에서 `index.html`을 더블클릭).

---

## 목차

1. [Quick Start](#1-quick-start)
2. [4계층 데이터 모델](#2-4계층-데이터-모델)
3. [상단 / 좌측 트리 / 우측 인스펙터](#3-상단--좌측-트리--우측-인스펙터)
4. [뷰 1 — Topology](#4-뷰-1--topology)
5. [뷰 2 — Architecture](#5-뷰-2--architecture)
6. [뷰 3 — Capacity (스케줄 시뮬레이션)](#6-뷰-3--capacity-스케줄-시뮬레이션)
7. [뷰 4 — Dependency Graph](#7-뷰-4--dependency-graph)
8. [뷰 5 — Namespace & NetworkPolicy](#8-뷰-5--namespace--networkpolicy)
9. [뷰 6 — Bottleneck (정적 병목 분석)](#9-뷰-6--bottleneck-정적-병목-분석)
10. [뷰 7 — Checklist (서비스 준비 점검)](#10-뷰-7--checklist-서비스-준비-점검)
11. [뷰 8 — 산출물 (SI 납품 문서)](#11-뷰-8--산출물-si-납품-문서)
12. [뷰 9 — Export (K8s YAML / Helm / Compose / Report)](#12-뷰-9--export-k8s-yaml--helm--compose--report)
13. [Workload Inspector — 5탭 + HA](#13-workload-inspector--5탭--ha)
14. [GPU 모델링 — chassis · 실장착 · VRAM 소비 · MIG](#14-gpu-모델링--chassis--실장착--vram-소비--mig)
15. [Quick Deploy 템플릿 & Project 시드](#15-quick-deploy-템플릿--project-시드)
16. [Validator — 검증 룰 카탈로그](#16-validator--검증-룰-카탈로그)
17. [카탈로그 데이터 (GPU · Runtime · Node 스펙)](#17-카탈로그-데이터-gpu--runtime--node-스펙)
18. [스키마 & 버전](#18-스키마--버전)
19. [영속성 · Import / Export](#19-영속성--import--export)
20. [키보드 단축키](#20-키보드-단축키)
21. [디렉터리 구조](#21-디렉터리-구조)
22. [라이선스](#22-라이선스)

---

## 1. Quick Start

```bash
# 1) 그냥 index.html을 더블클릭하거나
open vk8s/index.html

# 2) 정적 호스팅 (예: aiotool.net) 하위 경로로 배포
# 백엔드/빌드 단계 없음.
```

처음 진입하면 Splash에서 `On-Prem / Cloud / Hybrid` 중 하나를 고르고 프로젝트명을 입력하거나,
**Import existing JSON**으로 기존 프로젝트를 복원한다. 또는 **📋 Templates**에서 아키텍처 템플릿
(3-Tier Web / Microservices / LLM Inference / Hybrid Cloud / Multi-Zone HA)을 한 번에 시드할 수 있다.
Project Dashboard에 들어가면 바로 Sites · Nodes · Clusters · Workloads를 자유롭게 편집할 수 있다.

---

## 2. 4계층 데이터 모델

```
Project ──┬── Sites       ── Storage / L4 LB / NetworkDevice / ExternalService
          ├── Nodes       (spec: form_factor, vcpu, memory_gb, disks, gpu, gpu_chassis, power)
          ├── Clusters    ── Namespaces / NetworkPolicies / Workloads / Addons
          │                  (platform=vm 이면 서버 그룹 — Namespaces·NetPol·Addons 없음, §13.3)
          ├── site_links  (사이트 간 VPN / leased line — 대역폭·지연·암호화)
          └── (catalog)   ── 사전정의된 node-spec, gpu-model, runtime-profile
```

- **Site** — 물리 IDC 또는 클라우드 리전. onprem이면 `capacity.rack_units / power_kw / cooling_kw`를 갖는다. 사이트는 `storages` · `l4_balancers` · `network_devices` · `external_services`를 보유.
- **site_links** — 프로젝트 최상위 배열. 두 사이트를 VPN/leased line으로 연결(대역폭·지연·암호화) — Hybrid/Multi-site 토폴로지에서 사용.
- **Node** — 단일 서버. `cluster_roles`에 `control-plane` / `worker` 부여. site에 속하고 cluster에 멤버로 들어감.
- **Cluster** — 배포 그룹. `platform: k8s | vm` (§13.3). `node_ids`로 노드 멤버 관리. CNI / CSI / GPU Operator / Local PV / Ingress 등 addon, HA 옵션, 네임스페이스, 네트워크 정책, 워크로드 보유.
- **Workload** — Deployment / StatefulSet / DaemonSet / Job / CronJob 단위. replicas, requests/limits, GPU 요청, PVC, runtime(언어·프레임워크·동시성 모델), 네트워크(Service · Ingress), `depends_on`, **HA 토폴로지(stateful)**.

스키마 정의: [`schema/`](./schema/) · 자세한 설명: [`schema/README.md`](./schema/README.md).

---

## 3. 상단 / 좌측 트리 / 우측 인스펙터

```
┌──────────────────────────────────────────────────────────────────┐
│ 🧵 vk8s   project-name   [hybrid]                  [💾] [⤓] [⤒]   │  ← Top bar
├────────┬──────────────────────────────────────────┬──────────────┤
│ Tree   │  Canvas (9개 뷰 중 하나)                  │ Inspector   │
│ Sites  │  [Topology][Architecture][Capacity][Dep] │ 선택된       │
│ Nodes  │  [Namespace][Bottleneck][Checklist][Exp] │ 엔티티 폼    │
│ Cluster│                                          │              │
│ Workld │                                          │              │
├────────┴──────────────────────────────────────────┴──────────────┤
│ ⚠ N errors  ▲ M warnings  ℹ K info       [Findings ▾]            │  ← Findings strip
└──────────────────────────────────────────────────────────────────┘
```

- **Top bar** — 프로젝트명 · 모드 뱃지(`onprem`/`cloud`/`hybrid`) · 저장/Import/Export.
- **좌측 트리** — Sites · Nodes · Clusters · Workloads (열고 닫기 가능). 클릭하면 인스펙터에 폼이 뜸.
- **인스펙터** — 선택된 엔티티의 모든 필드를 폼으로 노출. `🔒` 토글로 우발 편집/삭제 보호 가능.
- **클러스터 복사 / 붙여넣기** — 트리에서 클러스터를 통째로 복제(워크로드 + 노드 함께)하거나, 클립보드로 복사해 다른 프로젝트에 붙여넣기(`pasteClusterIntoProject`). 붙여넣을 때 노드 IP는 대상 서브넷 기준으로 재할당.
- **Findings strip** — error/warn/info 건수를 항상 표시. 펼치면 상세 + 클릭 시 해당 엔티티로 점프.

UI 명세 원문: [`docs/ui-spec.md`](./docs/ui-spec.md).

---

## 4. 뷰 1 — Topology

물리/논리 토폴로지를 site → node → cluster → workload 순으로 그래프로 본다.

- **3개 레이아웃**: `grid` (자동), `free` (자유 배치 — 드래그로 이동), `diagram` (SVG 다이어그램).
- **레이어 토글**: Compute · Network · Services · Ingress · Sites 각각 켜고 끄기.
- **다이어그램 모드 export**: 현재 뷰를 **SVG / PNG**로 저장 (범례 포함 — §5 참조).
- 같은 cluster의 node-cluster 연결선, l4_balancer → cluster 라인, storage → workload PVC 라인 등 자동 라우팅.

---

## 5. 뷰 2 — Architecture

워크로드를 카테고리(Frontend/API/WAS/DB/Cache/Queue/ML-Inference/...) 또는 zone(라벨 기준)으로 묶어 보는 *논리* 아키텍처.

- `archGroupByZone` 토글 — 라벨 기반 그룹 ↔ 카테고리 그룹.
- ingress → frontend → API → DB 흐름이 한눈에 들어오게 화살표 라우팅.
- **노드 색상 토글** — `category`(워크로드 종류별 고정 팔레트) ↔ `namespace`(네임스페이스별 golden-angle 자동 색상)로 전환. namespace는 표기 정규화 후 색을 매핑. 화면 범례 chip 행과 **SVG/PNG export에 구워지는 범례가 항상 동일**하게 유지된다(`legendEntriesFor`).
- **크로스클러스터 표시** — 다른 클러스터 워크로드를 호출하는 의존성은 ghost endpoint로 표시되고, Architecture 상단의 refs 박스로 외부 참조를 요약. 워크로드별 `dep_layer` override로 레인 위치를 수동 조정 가능.

---

## 6. 뷰 3 — Capacity (스케줄 시뮬레이션)

워크로드 replica 단위로 **first-fit 스케줄 시뮬레이션**을 돌려 어디에 어떻게 배치되는지 보여준다.

- 워커 노드 카드마다 막대 5종: **CPU / Memory / GPU / VRAM / PSU(onprem만)**.
- 배치 성공한 pod 목록과 노드별 자원 차감 결과 표시.
- 상단 요약: `N/M pods placed`, `✓ all placed` 또는 `⊗ K unschedulable`.

### 6.1 Unschedulable pod에 대한 사유 breakdown

배치 실패한 pod 카드에 다음을 노출:

- **요청 라인** — `cpu · mem · GPU(count or VRAM)`.
- **사유 요약** — 어떤 차원이 몇 개 워커를 막았는지: `gpu 3/3 · H200-SXM 3/3` 식.
- **노드별 상세** — `▣ node-name   cpu 16/32 · H200-SXM 1/2` — 클릭 시 노드 디테일로 점프.
- **worker 0대인 경우** — `cluster_roles에 'worker' 추가 필요` 힌트.

`simulateScheduling()` (in `index.html`) → `renderUnscheduled()`.

### 6.2 VRAM 기반 GPU 스케줄

워크로드의 GPU 요청을 **카드 개수가 아닌 VRAM(GB) 소비량**으로 모델링한다.
vLLM 같은 추론 워크로드는 노드의 H100/H200 VRAM 풀에서 GB 단위로 잘라 쓰는 주체이지,
*카드를 장착하는* 주체가 아니라는 사실을 반영.

- 워크로드 Resources 탭의 GPU 입력 = `GPU VRAM (GB / pod)`.
- model 드롭다운은 *해당 cluster의 worker가 실제 장착한 모델*로만 좁힘 — 미스매치 자체를 차단.
- `(any)` 옵션은 노드의 모든 모델 VRAM 합산 풀에서 소비.
- legacy `gpu.count` 기반 워크로드는 그대로 동작 (자동 fallback).
- 자세한 분기 로직은 §14 참조.

---

## 7. 뷰 4 — Dependency Graph

`workload.depends_on`을 좌→우 layered DAG로 시각화. 진입점(아무도 호출 안 하는 노드)이 가장 왼쪽,
싱크(leaf)가 오른쪽. 점선·주황은 cycle (back edge).

### 7.1 노드를 가로지르지 않는 edge 라우팅

2-layer 이상 건너뛰는 edge는 중간 layer의 노드 row를 회피하는 *gutter lane*으로 우회.
사전 패스로 intervening 노드들의 y 중심을 모은 뒤 *위/아래/사이 gap* 중 (y1+y2)/2에 가장 가까운
lane을 선택, cubic bezier control point를 overshoot시켜 곡선 중점이 lane에 정확히 닿게 한다.
overshoot가 viewBox를 벗어나지 않게 SVG 영역도 함께 확장.

cluster 밴드(§7.3)가 켜져 있으면 회피 대상은 *같은 밴드 안의* 노드로 한정된다.
중간 layer가 모든 밴드를 관통하기 때문에, 전체를 회피 대상으로 잡으면 클러스터 내부 edge가
다이어그램 전체 높이를 가로지르는 곡선이 되어버린다. 밴드를 넘나드는 edge는 어차피 긴 대각선이라
lane 라우팅을 건너뛰고 직접 bezier로 그린다.

### 7.2 Cluster 필터

상단에 cluster 탭 (`All (N) / cluster-A (n1) / cluster-B (n2) ...`).
선택 시 해당 cluster의 워크로드만 노출. cross-cluster 의존이 숨겨졌으면
`▲ cross-cluster 의존 N개가 숨겨져 있습니다 — 'All' 탭에서 확인하세요` 배너로 명시.

### 7.3 Cluster 밴드 (클러스터 우선 구분)

**클러스터가 1차 구분 기준.** 노드를 클러스터별 가로 밴드(swimlane)로 나눈 뒤 그 안에서 layer를 배치한다.
layer 자체는 그래프 전체에서 공유되므로 `entry / L1 / L2 … / sink` 컬럼은 모든 밴드에서 세로로 정렬되고,
같은 클러스터의 워크로드는 절대 흩어지지 않는다.

- 밴드 순서 = 프로젝트의 클러스터 순서 (상단 탭 순서와 동일).
- 밴드 높이 = 그 밴드에서 가장 많은 노드를 가진 layer 기준. layer별로 밴드 안에서 세로 중앙 정렬.
- barycenter 교차 최소화는 밴드 *내부* 순서만 바꾼다 (클러스터 rank가 항상 우선).
- cross-cluster ghost endpoint(§7.2 필터 사용 시)도 자기 클러스터 밴드에 놓이므로,
  트래픽이 밴드를 벗어나는 지점이 그대로 눈에 보인다.
- 밴드 테두리는 무채색(점선 + 클러스터명·노드 수)으로만 그린다 — 박스 색은 category/namespace
  색상 표준 전용이라 겹치지 않게 한다 (밴드에 클러스터별 색을 입히지 않는 이유).

우측 상단 체크박스 `cluster 밴드 (클러스터별 구분)`로 토글 (클러스터 2개 이상일 때만 노출, 기본 ON).
끄면 예전처럼 클러스터를 섞은 채 layer만으로 배치한다. 토글 시 viewBox는 리셋(Fit)된다.

### 7.4 워크로드별 그래프 숨김

`workload.hide_from_dep_graph: true`면 해당 워크로드 노드와 연결된 edge가 그래프에서 제외.
검증·export·다른 뷰에는 전혀 영향 없음 — 순수 시각화 옵션. workload Inspector → Basic 탭에서 토글.

### 7.5 Edge label 토글

View-local 옵션 (프로젝트 데이터에는 저장 안 됨). 우측 상단 체크박스
`edge label (protocol·latency)`로 라벨(`HTTP / 15ms` 등)을 한 번에 끄고 켤 수 있다.

---

## 8. 뷰 5 — Namespace & NetworkPolicy

namespace 별 워크로드 분포, ResourceQuota, NetworkPolicy(ingress/egress) 시각화.

- namespace에 `default_deny: true`면 모든 egress/ingress가 명시적 allow 필요.
- `prod`로 시작하는 namespace에 default_deny가 없으면 경고.
- workload의 `depends_on` 호출이 NetworkPolicy로 차단되면 block 룰 발동
  (`WORKLOAD_DEPS_BLOCKED_BY_NETPOL` 등 — §16 참조).

---

## 9. 뷰 6 — Bottleneck (정적 병목 분석)

`runtime.concurrency` (thread-pool / async-await / event-loop / worker-pool / process) +
`traffic_profile` (RPS, avg_ms, peak_factor) + `depends_on` (call_ms, fanout_pct, calls_per_request)을 입력 삼아
**Little's Law** 기반으로 워크로드별 병목 카드를 그린다.

### 5개 룰

1. **Compute capacity (Little's Law)** — `in_flight = RPS × avg_ms / 1000`, `per_replica` capacity vs 요구치.
2. **Caller pool → target** — 호출자 connection pool 고갈 여부.
3. **Downstream RPS → target** — target의 RPS budget 초과.
4. **Inbound conn budget (as DB target)** — DB max_connections 대비 caller 합계.
5. **Latency budget** — 워크로드 자체 `avg_ms` + Σ(downstream call_ms × calls_per_request) > p99 budget.

각 카드는 **transparency block**으로 입력 변수 + 산식 + 도출된 값 + 권장 액션을 모두 노출
(rule transparency 커밋). validator 룰 카탈로그(§16)와 1:1 대응.

---

## 10. 뷰 7 — Checklist (서비스 준비 점검)

현재 프로젝트 구성(클러스터·노드·워크로드·HA·애드온·네트워크·스토리지)을 읽어 **서비스 오픈 전 검증해야 할 운영 점검 항목**을 자동 생성한다. Findings(§16)가 *설정 오류*를 잡는다면, Checklist는 *실제 동작 테스트 절차*("primary를 죽이고 replica 승격을 확인")를 나열 — 둘은 보완 관계.

### 10.1 두 범위 × 그룹

- **🌐 전체 (프로젝트 공통)** — CI/CD & 배포 파이프라인 · 멀티 클러스터 / DR · 릴리즈 준비 / 운영.
- **☸ 클러스터별** — 클러스터 부트스트랩 · **HA 구성 / 장애 조치 테스트** · 네트워크 & 보안 · 스토리지 & 백업 · 용량 & 오토스케일링 · 관측성(모니터링·로깅) · 인그레스 & 외부 노출.

각 그룹은 선언적 `CHECKLIST_PROJECT_GROUPS` / `CHECKLIST_CLUSTER_GROUPS` 배열로 정의되며, `_buildChecklist(p)`가 구성에서 항목을 도출한다.

### 10.2 구성 도출 항목

`config` 배지가 붙은 항목은 실제 구성에서 자동 생성된다. 예:

- `ha_control_plane`이면 → "control-plane 1대 강제 종료 시 API 서버·스케줄링 유지 확인", "etcd 쿼럼 유지 확인".
- workload `ha.mode = primary-replica` → "primary 강제 종료 시 replica 자동 승격(failover) 확인", `sentinel` → "Sentinel 쿼럼(N)으로 failover 확인", `sharded-cluster` → "shard 장애 시 replica 승격 & 슬롯 재분배 확인".
- `anti_affinity` 미설정 HA 워크로드 → 경고성 점검 항목.
- PVC / ingress TLS / HPA / default-deny NetworkPolicy / 레지스트리 external_service 등도 각각 항목을 만든다.

도출 항목엔 `↗`가 붙어 해당 워크로드·클러스터로 바로 이동(`selectEntity`).

### 10.3 편집 · 삭제 · 사용자 항목 추가

모든 항목(구성도출·표준·직접추가)에 동일하게 적용:

- **✎ 수정** — 인라인으로 항목 텍스트를 덮어쓴다. 구성도출 항목은 `✎수정됨` 표시 + `↺`로 기본값 복원.
- **✕ 숨기기** — 구성도출 항목은 재생성되므로 *소프트 삭제*(hidden 플래그). 헤더의 **"☐ 숨긴 항목 N"** 토글 → `↩ 복원`으로 되살림.
- **🗑 삭제** — 직접 추가한 항목은 확인 후 완전 삭제.
- **＋ 항목 추가** — 그룹마다 사용자 정의 항목 추가(`직접추가` 배지).

### 10.4 진행률 · 필터 · Export

- 전체 / 범위별 / 그룹별 진행률 바, 체크박스 + 메모.
- **"☐ 남은 항목만"** 필터로 완료 항목 숨김.
- **⬇ Markdown** — `- [x] / [ ]` 체크박스 마크다운.
- **⬇ Excel** — 진짜 `.xlsx`(범위·그룹·항목·유형·완료·메모 6열, 헤더 굵게+고정). 기존 `_zipStore`로 OOXML 직접 생성, inline string.

Markdown·Excel 모두 `_buildChecklist`를 소비하므로 수정/삭제/추가가 자동 반영된다.

### 10.5 저장

체크 상태·메모·수정·숨김은 `project.checklist[key] = {done, note, text, hidden}`(엔티티 **id** 기반 안정 키), 사용자 항목은 `project.checklist_custom`에 저장 → save / load / import / export 모두 함께 라운드트립.

---

## 11. 뷰 8 — 산출물 (SI 납품 문서)

**현재 구성에서 도출한 납품 문서를 화면에서 검토·수정한 뒤 내보내는 탭.**
Export(§12)가 기계가 먹는 산출물(kubectl / helm / docker compose)이라면, 이 탭은 사람이 읽고
납품하는 문서다. 그래서 다운로드 버튼을 이 탭 안에만 둬서 **눈으로 확인한 뒤 내보내는 흐름**을 강제한다.

산출물은 `DELIVERABLES` 레지스트리에 등록된다 — 새 산출물을 추가할 때는 배열에 한 줄과 렌더 함수
하나면 되고 상단 탭은 늘어나지 않는다 (Export 뷰의 `tabs`, `REPORT_SECTIONS`와 같은 관용구).

### 11.1 아키텍처 개념도

워크로드를 전부 나열하는 Architecture 뷰(§5)의 한 단계 위 추상. **행 = 아키텍처 계층(§13.2),
열 = 구역 또는 클러스터**인 격자로 그린다. 두 축은 직교한다 — 관리형 RDS는 "Data 계층인데 외부
구역"이지 정체불명의 외부 서비스가 아니다.

#### 행 — 계층

행 목록·순서는 `TIERS` 하나가 정한다. 흐름/보조 계층은 order 순으로 쌓되 같은 order에서는
흐름을 먼저 둔다(Application → Data 화살표가 중간에 낀 캐시·큐에 끊기지 않게). 횡단 계층은
열로 쪼개지 않고 전체 폭 밴드로 그리고 흐름 화살표도 붙이지 않는다.

```
사용자 · 채널            (액터 — 계층 아님)
  ↓
Network & Edge          network_devices(cdn·dns·waf·ddos·firewall·vpn) · site_links
  ↓
Entry / Routing         l4_balancers · Entry 계층 워크로드 (없으면 addons.ingress)
  ↓
Presentation            category로 묶어 `Frontend ×3`
  ↓
Application             `API/WAS ×7` — 미분류 워크로드도 여기 남겨 눈에 보이게 둔다
  ↓
Data                    + PVC가 실제로 가리키는 사이트 스토리지
Caching / Messaging     (보조 — 화살표 없이 아래에 나란히)
─────
Security & Auth         (횡단 — 전체 폭)
Observability & Mgmt
Delivery & Operations
```

횡단 계층 박스는 네임스페이스 역할(`PLATFORM_NS_ROLES`)로 색·라벨을 매기고, 역할이 안 잡히면
계층 이름으로 묶는다(`Platform` 뭉텅이가 되지 않게).

`kube-system` 등 클러스터 런타임 워크로드는 그리지 않는다 — CoreDNS·kube-proxy는 아키텍처가
고른 계층이 아니라 쿠버네티스가 그렇게 동작하는 방식이라, 클러스터가 그 역할을 대신한다.

#### 열 — `열 구분 [구역 | 클러스터]`

| 모드 | 열 |
|---|---|
| **구역** (기본) | `외부 구역` · `내부 구역`. **외부 = 우리가 운영하지 않는 것** — `external_services`와 `site_links`. 나머지는 전부 내부 |
| **클러스터** | 클러스터별 열 + `공용`(여러 클러스터가 쓰는 스토리지 등) + `외부` |

열 폭은 내용에 맞춰 정해진다 — 외부 구역은 대개 좁고 내부가 넓다. 열이 둘 이상일 때만 상단에
열 머리글이 붙고, 흐름 화살표도 양쪽 행에 내용이 있는 열마다 따로 그린다(앱은 내부인데 DB는
외부인 경우처럼 한 열도 못 그리면 가운데 화살표 하나로 대신한다).

### 11.2 검토 · 수정

**그림에서 짚어서 고친다.** 요소가 수십 개가 되면 전체 목록을 훑어 한 줄을 찾는 편집은 못 쓴다.
박스를 클릭하면 다이어그램 아래 패널이 **그 요소 하나만** 보여준다.

```
■ API/WAS ×1   [Application] [내부 구역] [자동 도출]        원본 보기 →  ✕
라벨  [                    ]   ← 비우면 도출값(placeholder)으로 복귀
비고  [                    ]
계층  [자동 — Application ▾]   구역  [자동 — 내부 구역 ▾]
[👁 개념도에서 제외]  [↺ 이 요소 초기화]  [🗑 삭제(직접 추가만)]
```

- **드래그 앤 드롭으로 계층 이동** — 박스를 끌어 다른 계층 행에 놓으면 그 계층으로 옮겨진다.
  드래그 중 대상 행이 강조되고, 옮긴 박스에는 `✥` 표시가 붙는다. 4px 이하로 움직이면 클릭(선택)으로 친다.
  **비어 있는 계층으로 옮길 때는** 패널의 계층 셀렉트를 쓴다 (11개 행이 전부 들어 있다).
- **계층 · 구역은 그림에서의 자리** — 워크로드의 의미론적 계층(`workload.tier` §13.2)과는 별개다.
  셀렉트의 첫 항목이 도출값(`자동 — Application`)이라 언제든 되돌릴 수 있다.
- **제외한 요소**는 그림에서 사라져 클릭할 수 없으므로, 다이어그램 바로 아래 `제외된 요소 N` 칩으로
  되돌린다. 되돌리면 그 요소가 자동으로 선택된다.
- **＋ 요소** — 툴바에서 계층을 고르고 추가한다(점선 테두리). 추가 즉시 선택되어 바로 이름을 적을 수 있다.
- **↺ 재도출** — 손댄 내용을 전부 지우고 현재 구성에서 다시 도출.

선택 링·드롭 하이라이트 같은 **편집용 장식은 `data-editor`로 표시해 내보낼 때 제거**한다 —
화면에서 고르고 있던 흔적이 제안서에 구워지면 안 된다.

수정 내용은 `project.concept`에 **엔티티 id 기반 안정 키**로 저장된다(Checklist와 동일한 방식).
요소 자체는 매번 config에서 다시 도출되므로, 구성이 바뀌어도 손댄 라벨·비고·계층·구역·숨김은 유지된다.
6레인 시절에 추가한 요소는 새 계층 행으로 자동 이동한다(`service`→Application, `platform`→Observability).

### 11.3 내보내기

- **배경 `[화면 | 인쇄]`** — 앱은 다크 테마지만 산출물은 흰 종이로 나간다. `인쇄`를 고르면 직렬화
  시점에 팔레트를 흰 배경·어두운 글씨로 갈아끼워 굽는다(테마 전환이 아니라 export 전용).
  이 토글은 **모든 다이어그램 뷰**(Topology diagram · Architecture · Dependency · 개념도)에 함께 붙는다.
- **SVG / PNG** — 범례와 작성일이 그림 안에 함께 구워진다 (화면과 내보낸 파일이 동일).
- **Markdown** — 레인별 요소 표(구분 / 요소 / 수량 / 비고). 숨긴 요소는 본문에서 빠지고 하단에 제외 목록으로 남는다.

---

## 12. 뷰 9 — Export (K8s YAML / Helm / Compose / Report)

4개 탭 + 마스터 다운로드 버튼.

### 12.1 K8s Manifests (`*.yaml`)

`kubectl apply -f` 가능한 multi-doc YAML. 포함 리소스:

- Namespace · ResourceQuota · NetworkPolicy
- Deployment · StatefulSet (PVC 마운트 포함)
- Service (ClusterIP / NodePort / LoadBalancer)
- Ingress
- PersistentVolumeClaim
- `nvidia.com/gpu` resource limit (count 기반 — §14의 caveat 참조)

### 12.2 Helm Chart (`*.zip`)

워크로드별 umbrella chart 패턴.

- `Chart.yaml` · `values.yaml` · `templates/` · `_helpers.tpl`
- 워크로드별 values 분리. CLI에서 `helm install -f values.yaml`.
- ZIP은 store-only(deflate 미사용) → 브라우저만으로 zip 생성 가능.
- `HELM_DUPLICATE_CHART_NAME` / `HELM_INGRESS_NO_HOST` / `HELM_NO_IMAGE_REPO` 검증 동시 동작.

### 12.3 Docker Compose (`*.yml`)

단일 호스트용 **best-effort** 변환. 워크로드 → service, 내부 포트는 `expose`·외부 노출은 `publish`, `depends_on` / volume / 리소스 매핑.

- Operator / HA / Ingress / 오토스케일 등 k8s 고유 기능은 단일 호스트 근사치로 빠지며, 각 service 주석에 누락 항목을 명시.
- `docker compose up -d`로 로컬 구동.

### 12.4 Project Report (`*.md`)

발주서/제안서에 그대로 첨부할 수 있는 마크다운. **섹션 선택형** — Overview / Node Specs / Site Capacity / GPU Plan / Network Plan / Cluster Config / Architecture / Workload Summary / Workload Detail / Validation 중 원하는 것만 체크(`REPORT_SECTIONS` + `state.ui.reportHiddenSections`).

- 사이트별 U·kW 산정과 잔여 헤드룸
- GPU 플랜 (모델별 카드/VRAM)
- 네트워크 플랜 (L4 LB · CIDR · CNI)
- 검증 요약 (error/warn 건수와 코드)

### 12.5 ⬇ Download everything (`*.zip`)

K8s YAML · Docker Compose · 전체 Project Report · Helm chart를 한 ZIP에 묶어 다운로드(리포트는 항상 전체 섹션). 파일명에 날짜 stamp.

---

## 13. Workload Inspector — 5탭 + HA

| 탭 | 내용 |
|---|---|
| **Basic** | name, category, kind, **architecture tier(§13.2)**, replicas, namespace, cluster (read-only), 🔒 lock, **Dependency 그래프에서 숨기기**, **HA 토폴로지(stateful 카테고리)** |
| **Resources** | requests/limits (CPU·Memory), **GPU VRAM(GB)** + model 드롭다운(cluster 장착 모델만), PVC (size_gb · access_mode · storage_class · target_storage), Cluster total 합산 |
| **Runtime** | 언어/프레임워크/버전, concurrency 모델(thread-pool / async-await / event-loop / worker-pool / process), thread/connection pool 사이즈, traffic profile (RPS · avg_ms · peak_factor) |
| **Network** | Service (type · ports), Ingress (host · path · TLS), HPA(min/max · cpu_target_pct) |
| **Deps** | `depends_on[]` 편집 — target_workload_id · protocol · port · avg_call_ms · fanout_pct · calls_per_request |

### 13.1 HA 토폴로지 (stateful 워크로드)

DB / Cache / Queue 카테고리에서만 Basic 탭에 노출 (`workload.ha`). replica 수만으로 추측하지 않고 복제 토폴로지를 명시한다:

- **mode** — `none` / `primary-replica`(Postgres 류) / `sentinel`(Redis Sentinel) / `sharded-cluster`(Redis Cluster · MongoDB).
- **primaries · replicas_per_primary · quorum_count** — 데이터 pod와 쿼럼 pod 수. `replicas`는 토폴로지에서 자동 산출.
- **operator** — 자유 입력(예: `cloudnative-pg`, `bitnami-redis`, `strimzi-kafka-operator`) — export 라운드트립을 견디는 문서용.

이 값은 인스펙터·validator(HA 룰)·capacity 시뮬(쿼럼 pod footprint)·export·**Checklist(§10) HA 장애조치 항목**이 공유한다.

---

### 13.2 아키텍처 계층 (`workload.tier`)

인프라 아키텍처의 표준 계층. 개념도(§11.1) 배치와 **계층 규칙 validator(§16)**가 이 값을 공유한다.

| 성격 | 계층 | 그림에서 |
|---|---|---|
| **flow** — 트래픽이 위→아래로 지남 | `edge` → `entry` → `presentation` → `application` → `data` | 세로 스택 + 화살표. 호출 방향 검증 대상 |
| **support** — Application 옆에 붙음 | `caching` · `messaging` | 흐름 옆에 나란히 |
| **crosscut** — 전 계층에 걸침 | `security` · `observability` · `delivery` | 별도 밴드. 흐름 화살표 없음 |

횡단 계층을 흐름에 끼워 넣으면 화살표가 거짓말을 한다(인증 조회·메트릭 수집은 어느 계층에서든 일어남).
`delivery`(GitOps·CI·레지스트리·백업)는 런타임 계층이 아니라 딜리버리 파이프라인이라 횡단으로 둔다.

#### 판정 순서 — 3단

```
1. 사용자 지정 (workload.tier)   — 있으면 무조건 이것
2. 이름 · 제품 매칭               — redis→caching, kafka→messaging, prometheus→observability …
3. 구조 필드                      — 네임스페이스(monitoring·ingress-nginx…) → category
```

인스펙터 Basic 탭 셀렉트의 첫 항목이 도출 결과를 그대로 보여준다 (`자동 — Application (category)`).
잘못 잡혔을 때만 고르면 되고, 비우면 다시 자동으로 돌아간다 — `dep_layer`와 같은 방식.

**이름만으로는 부족하다.** 업무 서비스는 이름에 제품 신호가 없고(`user-service`, `배치서버`),
워크로드가 아닌 엔티티는 이름이 자의적이며(`F5` · `FW-A` · `nas-a`) `kind`가 정확하다.
그래서 제품 별칭은 **보수적으로** 유지한다 — 애매한 낱말(`elasticsearch` · `gateway` · `registry`)은
일부러 빼서 네임스페이스·category가 가르게 한다. `llm-api-gateway`는 이름에 gateway가 들어가지만
실제로는 FastAPI 서비스라 Application이 맞다.

`kube-system` 등 클러스터 런타임 네임스페이스는 계층을 매기지 않는다 — 개념도에서 빠지고
미분류 경고에서도 제외된다.

#### 워크로드가 아닌 엔티티

| 소스 | 매핑 | 지금 쓰이는 곳 |
|---|---|---|
| `network_devices.kind` | cdn·dns·waf·ddos·firewall·vpn → `edge`, switch·router → 없음 | 개념도 **접근 계층**의 "네트워크 경계" 박스에 넣을 장비를 고른다 (스위치·라우터는 물리 배선이라 제외) |
| `l4_balancers` | `entry` | 개념도 접근 계층 박스 · `LAYER_NO_ENTRY` 판정 (사이트 L4가 진입점인 클러스터를 오탐하지 않게) |
| `external_services.kind` | database·object_store → `data`, cache → `caching`, queue → `messaging`, registry·backup → `delivery`, monitoring·logging → `observability`, secret_store·identity → `security` | 개념도 **외부 연동** 레인의 범례·설명 (`관리형 Redis` → `Caching (외부)`). 레인 자체는 외부 연동에 그대로 둔다 |

사이트 경계에서 종단되는 VPN은 접근 계층의 경계 장비로 본다. 사용자 접속용 VPN 게이트웨이를
따로 모델링한다면 그 워크로드의 `tier`를 `security`로 지정하면 된다.

---

### 13.3 VM · 베어메탈 그룹 (`cluster.platform`)

온프레미스 설계에는 k8s 밖의 요소가 늘 섞인다 — 레거시 WAS, Oracle, 배치 서버, 파일 서버.
이를 가짜 클러스터 + 가짜 파드로 욱여넣으면 네임스페이스·Ingress·Deployment 같은 무의미한
필드가 따라붙는다. `cluster.platform`을 `vm`으로 두면 그 필드들이 데이터에서도 UI에서도 빠지고,
나머지 축(노드 · 서비스 · 의존 · 계층 · 용량)은 그대로 공유한다.

| | `platform: "k8s"` | `platform: "vm"` |
|---|---|---|
| 트리 · 인스펙터 | ⎈ Cluster | ▤ VM Group |
| 하위 항목 | Workloads (pod) | **서비스** (서버에 설치되는 것) |
| 갖는 것 | k8s_version · addons · namespaces · network_policies · ingress_rules · pod/service CIDR | node_subnet만 |
| 노드 행 | control-plane / worker / etcd 역할 | **OS 표시** (`node.os`) |
| 워크로드 필드 | kind(Deployment/StatefulSet) · namespace · HPA | **설치 소프트웨어** · 서버 대수 |
| HA | primary-replica · sentinel · sharded-cluster | **active-standby · active-active + VIP** |
| Quick Deploy | pod 템플릿 | 같은 템플릿이 **설치 미들웨어**로 들어감 (kind·namespace 제거) |
| Export | K8s YAML · Helm · Compose 대상 | **제외** (리포트·개념도·체크리스트에는 포함) |
| Namespace 뷰 | 대상 | 제외 |
| Validator | k8s 룰 전체 | k8s 룰 제외 + VM 룰 (§16) |

플랫폼을 되돌려도 기존 k8s 설정은 지워지지 않는다 — vm인 동안 UI·검증·export가 그 필드를
읽지 않을 뿐이다.

#### 설치 소프트웨어가 계층을 정한다

`workload.software[]`(이름 · 버전 · 포트)는 단순 기록이 아니라 **계층 판정의 입력**이다(§13.2).
워크로드 이름이 업무명이라 제품 신호가 없을 때 여기서 잡힌다.

```
정산서비스  [WAS]  software: Tomcat 9.0 · Apache 2.4   → Application (설치 SW)
원장DB      [DB]   software: Oracle 19c                → Data       (설치 SW)
```

#### 이중화

`active-standby`(운영 1 + 대기 1)와 `active-active`(N대 동시)는 VIP 페일오버가 전제라
VIP를 비우면 `VM_HA_NO_VIP` 경고가 뜬다. 대수는 Capacity 시뮬레이션에 그대로 반영된다.

#### 산출물

- **개념도** — VM 서비스도 계층 행에 그대로 들어간다. 열 구분을 `클러스터`로 두면 `▤ 그룹명`으로 표시된다.
- **Project Report** — Deployment Groups 섹션에 플랫폼 · 서버 OS · 설치 소프트웨어 · 이중화가 나간다. Node Specs 표에 OS 열 추가.
- **Checklist** — VM 그룹은 전용 3개 그룹(서버·OS 준비 / 미들웨어 설치·기동 / 이중화·절체 시험)이 나온다. k8s 부트스트랩·인그레스·오토스케일 항목은 나오지 않는다.

---

## 14. GPU 모델링 — chassis · 실장착 · VRAM 소비 · MIG

세 계층으로 나뉘어 검증된다.

1. **chassis** (`node.spec.gpu_chassis`): 노드가 *허용*하는 GPU. `max_count`, `supported_form_factors`,
   `supported_models`, `nvlink_topology(none/bridge/nvswitch-mesh)`. 발주 가능한 슬롯 정의.
2. **실장착** (`node.spec.gpu[]`): 실제 꽂혀 있는 카드. `{ model, count, multi_pod_sharing?, mig_enabled?, mig_profile? }[]`.
   - chassis 룰로 검증 (`GPU_CHASSIS_NOT_SUPPORTED` · `GPU_COUNT_EXCEEDS_CHASSIS` · `GPU_FORM_FACTOR_MISMATCH` · `GPU_MODEL_NOT_IN_CHASSIS_LIST`).
   - NVLink 토폴로지가 `none`인데 카드가 NVLink-capable인 경우 `NVLINK_ON_NONE_TOPOLOGY` 등 추가 검증.
   - **MIG** — A100/H100 등 MIG 지원 모델은 `mig_enabled` + `mig_profile`(예: `1g.10gb`, `3g.40gb`)을 지정. 프로파일 목록은 `data/gpu-models.json`의 `mig_profiles`에서 시드되고, model 변경 시 호환 안 되면 자동 reset. *표시·문서용* — 현재 스케줄 시뮬·export에는 미반영.
   - **pod 공유** (`multi_pod_sharing`): 아래 §14.2.
3. **워크로드 소비** (`workload.resources.requests.gpu`):
   - **VRAM 모드 (권장)** — `{ vram_gb, model?, breakdown? }` (count는 schema 호환 위해 1로 자동).
   - **count 모드 (legacy)** — `{ count, model? }`. 카드 N장을 통째로 점유.
   - 모드 분기: `vram_gb > 0`이면 VRAM 모드.
   - **모델 내역** (`breakdown[{ name, vram_gb, port? }]`) — 하나의 pod가 카드 한 장 안에서 모델을 여러 개
     서빙하는 구성 (예: vLLM 31B `:8000` + 8B `:8001`). 합계가 `vram_gb`로 자동 계산된다. GPU는 원래 여러
     프로세스를 동시에 수용하므로 MIG나 device plugin 설정이 **필요 없다**.

### 14.1 카드 단위 배치 (GPU card ledger)

배치의 단위는 노드의 VRAM 합계가 아니라 **물리 카드 한 장**이다. 시뮬레이터는 노드의 GPU 행을 카드 배열로
펼친 뒤(`count: 2` → 카드 2장) 카드별 잔여 VRAM에 bin-packing 한다.

- `vram_gb ≤ 카드 VRAM` → 여유 있는 카드에 best-fit 배치 (작은 pod가 새 카드를 열지 않고 채워짐)
- `vram_gb > 카드 VRAM` → `ceil(요청 / 카드)`장을 **빈 카드로 묶어** 점유 (tensor-parallel 전제, 같은 노드·같은 모델)
- count 모드 → N장을 통째로 점유

Capacity 뷰에 카드별 점유가 그대로 표시된다 — `GPU#0 72/141GB · vllm-multi`, `GPU#1 0/141GB (free)`.

### 14.2 pod 공유 모드 (`node.spec.gpu[].multi_pod_sharing`)

**서로 다른 pod 여러 개가 한 카드를 나눠 쓸 수 있는가**를 정하는 값. GPU가 여러 프로세스를 돌릴 수 있느냐와는
다른 층위의 문제다 — 그건 언제나 가능하고, 이 값은 *쿠버네티스 device plugin이 카드를 몇 개의 pod에 배정할 수
있는지*만 결정한다.

| 값 | 의미 | 시뮬레이터 |
|---|---|---|
| `none` (기본) | 스톡 device plugin — 카드 1장 = pod 1개 | 카드당 pod 1개 |
| `time-slicing` | device plugin이 카드를 N개로 광고. 메모리 격리 없음 | 카드 VRAM 한도 내에서 여러 pod |
| `mps` | CUDA MPS — 커널 동시 실행 + 클라이언트별 메모리 상한 | 위와 동일 |
| `mig` | 하드웨어 분할 | 현재 `none`으로 계산 (미구현) |

`none`인 상태에서 서로 다른 pod를 한 카드에 올리려 하면 배치가 실패하고 사유가 표시된다:
`card-sharing: 카드 2장 모두 다른 pod가 점유 (VRAM은 133GB 남음) · 노드 GPU 행의 공유 모드 필요`.
실제 클러스터에서 두 번째 pod가 Pending에 걸리는 상황을 그대로 재현한 것이다.

관련 findings — `GPU_CARD_UNDERUSED`(카드 절반 이하만 쓰는 요청), `GPU_SPANS_CARDS`(카드를 넘는 요청).

**Caveat** — K8s YAML/Helm export는 `nvidia.com/gpu` 정수만 emit할 수 있다 (K8s API의 한계).
VRAM 모드 워크로드는 `ceil(vram_gb / 카드 VRAM)`장으로 환산되어 나간다 (모델 미지정 시 1). 한 카드를 여러 pod가
공유하는 계획은 device plugin의 time-slicing/MPS 설정이 별도로 필요하며, 그 ConfigMap은 export에 포함되지 않는다.

---

## 15. Quick Deploy 템플릿 & Project 시드

### 15.1 Quick Deploy 워크로드 (한 클릭 추가)

- **DB**: Postgres / MySQL / MongoDB
- **Cache**: Redis
- **Queue**: Kafka / RabbitMQ
- **Search**: Elasticsearch
- **ML**: vLLM
- 각 템플릿에 PVC 시드, Service port, replicas, HA 기본값 포함.

### 15.2 아키텍처 템플릿 (📋 Templates)

전체 프로젝트를 한 번에 시드하는 1-click 템플릿:

- **3-Tier Web (HA)** — Frontend×3 + API(Spring)×3 + 외부 RDS/Redis, F5 L4 + nginx-ingress.
- **Microservices Demo** — Online Boutique 풍 8개 서비스 + 의존 그래프, Cilium · Ceph.
- **LLM Inference Platform** — XE9680(H200×8) + vLLM(Llama 70B) + Embedding + Qdrant + Redis.
- **Hybrid Cloud (HA)** — IDC 3노드(데이터) + AWS 3노드(웹), VPN site_link, 양쪽 HA control plane.
- **Multi-Zone HA Production** — 3 zone에 분산된 단일 클러스터, 풀 메시 leased line, anti-affinity zone 분산.

### 15.3 Project 시드 (모드)

처음 만들 때 셋 중 선택:

- **On-Prem** — Dell XE9680 + R660 기반의 GPU/CP 노드 구성 예시.
- **Cloud** — AWS m6i / g5 등 EC2 인스턴스 시드.
- **Hybrid** — 둘 다.

---

## 16. Validator — 검증 룰 카탈로그

300ms 디바운스로 매 편집 후 재실행. 결과는 Findings strip → 상세 패널.
모든 룰 정의: [`schema/validator-rules.md`](./schema/validator-rules.md).

### 카테고리

| 카테고리 | 대표 룰 (일부) |
|---|---|
| **GPU 호환성** | `GPU_CHASSIS_NOT_SUPPORTED` · `GPU_COUNT_EXCEEDS_CHASSIS` · `GPU_FORM_FACTOR_MISMATCH` · `GPU_MODEL_NOT_IN_CHASSIS_LIST` · `GPU_MODEL_UNKNOWN` |
| **NVLink** | `NVLINK_ON_NONE_TOPOLOGY` · `NVLINK_BRIDGE_PAIR_LIMIT` · `NVLINK_SXM_NOT_FULLY_POPULATED` |
| **Site Capacity** | `SITE_RACK_UNITS_EXCEEDED` · `SITE_POWER_EXCEEDED` · `SITE_POWER_HIGH_UTILIZATION` · `SINGLE_NODE_EXCEEDS_TYPICAL_PDU` · `GPU_NODE_NO_COOLING_INFO` |
| **Cluster 토폴로지** | `CLUSTER_NO_CONTROL_PLANE` · `HA_CONTROL_PLANE_NOT_ENOUGH` · `HA_CONTROL_PLANE_EVEN_NUMBER` · `NODE_IN_MULTIPLE_CLUSTERS` · `CLUSTER_SPANS_ENVIRONMENTS` |
| **워크로드 스케줄링** | `WORKLOAD_CPU_NOT_SCHEDULABLE` · `WORKLOAD_MEM_NOT_SCHEDULABLE` · `WORKLOAD_GPU_NO_NODE` · `WORKLOAD_GPU_INSUFFICIENT` · `WORKLOAD_GPU_MODEL_NOT_AVAILABLE` · `WORKLOAD_GPU_VRAM_INSUFFICIENT` · `WORKLOAD_GPU_MODEL_VRAM_INSUFFICIENT` |
| **HA 토폴로지** | `WORKLOAD_STATEFUL_NO_HA` · `WORKLOAD_STATEFUL_NO_ANTI_AFFINITY` · `WORKLOAD_SENTINEL_QUORUM_LOW` · `WORKLOAD_SENTINEL_QUORUM_EVEN` · `WORKLOAD_SHARDED_TOO_SMALL` · `WORKLOAD_HA_NO_STORAGE` |
| **Addon 권장** | `GPU_NODE_NO_GPU_OPERATOR` · `NO_CNI_SELECTED` · `NO_CSI_SELECTED_BUT_PVC_USED` |
| **무결성** | `SITE_NODE_BACKREF_MISMATCH` · `MODE_SITE_KIND_CONFLICT` |
| **Namespace / NetPol** | `WORKLOAD_NAMESPACE_NOT_DECLARED` · `WORKLOAD_DEPS_BLOCKED_BY_NETPOL` · `WORKLOAD_EGRESS_BLOCKED_BY_NETPOL` · `NS_DEFAULT_DENY_NO_ALLOW_RULES` · `NS_RESOURCE_QUOTA_EXCEEDED_BY_WORKLOADS` · `PROD_NS_NO_DEFAULT_DENY` · `NETPOL_TARGETS_NONEXISTENT_POD` · `NETPOL_FROM_NS_NONEXISTENT` |
| **Dependency** | `DEPENDS_ON_TARGET_NOT_FOUND` · `DEPENDS_ON_CYCLE` · `DEPENDS_ON_CROSS_CLUSTER` |
| **정적 병목** | `THREAD_POOL_SATURATION` · `EVENT_LOOP_CPU_BOUND` · `WORKER_POOL_SATURATION` · `DB_CONNECTION_POOL_EXHAUSTED` · `DB_MAX_CONNECTIONS_EXCEEDED` · `TIMEOUT_CASCADE_RISK` · `JVM_HEAP_TOO_SMALL` · `JVM_HEAP_GT_REQUEST` · `SPOF_RISK` · `STATEFUL_NO_PVC` · `RPS_NEVER_REACHED_BY_DEPS` · `KEEPALIVE_HIGH_RPS_DISABLED` |
| **Helm Export** | `HELM_DUPLICATE_CHART_NAME` · `HELM_INGRESS_NO_HOST` · `HELM_NO_IMAGE_REPO` |
| **VM 그룹** | `VM_NODE_NO_OS` · `VM_SERVICE_NO_SOFTWARE` · `VM_HA_NO_VIP` — platform=vm 그룹 전용. 대신 control-plane·CNI·GPU Operator·네임스페이스·NetworkPolicy 룰은 이 그룹에서 실행되지 않는다 |
| **계층 규칙** | `LAYER_UNCLASSIFIED` · `LAYER_DATA_EXPOSED` · `LAYER_SKIP_CALL` · `LAYER_UPWARD_CALL` · `LAYER_NO_ENTRY` — 아키텍처 계층(§13.2) 기반. 프론트가 DB를 직접 부르거나 Data 계층이 외부 노출되는 구성을 잡는다. 전부 `warn`/`info` — 계층은 설계 판단이라 저장을 막지 않는다 |

각 룰은 `error(block)` / `warn` / `info` 중 하나의 severity를 갖는다.

> Checklist(§10)는 validator와 별개다 — validator는 *설정 오류*를, Checklist는 *운영 테스트 절차*를 다룬다.

---

## 17. 카탈로그 데이터 (GPU · Runtime · Node 스펙)

### 17.1 GPU 모델 — `data/gpu-models.json`

NVIDIA Blackwell Ultra(B300) / Blackwell(B200·B100·RTX PRO 6000) / Hopper(H200·H100) / Ada(L40S·L40·L4) / Ampere(A100) 라인업. 각 항목:

```json
"H200-SXM": {
  "vendor": "NVIDIA", "family": "Hopper", "form_factor": "SXM",
  "vram_gb": 141, "tdp_w": 700, "fp16_tflops": 1979,
  "released": 2024, "nvlink_capable": true,
  "mig_capable": true,
  "mig_profiles": [
    { "name": "1g.18gb", "count": 7, "vram_gb": 18 },
    { "name": "3g.71gb", "count": 2, "vram_gb": 71 },
    { "name": "7g.141gb", "count": 1, "vram_gb": 141 }
  ]
}
```

### 17.2 Runtime profile — `data/runtime-profiles.json`

언어/프레임워크별 동시성 모델 디폴트 (thread-pool 크기, async-await 효율, GIL, JVM heap 비율 등).
워크로드 Runtime 탭에서 자동 시드되어 Bottleneck 룰의 입력으로 쓰임.

### 17.3 Node 카탈로그 — `data/catalog/*.json`

Dell PowerEdge, AWS EC2, NCP, Azure VM 등 사전 정의된 노드 스펙.
`chassis`, `form_factor`, `vcpu`, `memory_gb`, `disks`, `gpu_chassis`, `power.psu_max_w`까지 채워져 있어
"Add Node" 마법사에서 한 번에 시드 가능.

스키마 정의: [`schema/catalog-entry.schema.json`](./schema/catalog-entry.schema.json).

---

## 18. 스키마 & 버전

JSON Schema (Draft 2020-12) 7개:

| 파일 | 정의 |
|---|---|
| `project.schema.json` | 최상위 — name, mode, sites, nodes, clusters, site_links |
| `site.schema.json` | site + capacity + storages + l4_balancers + network_devices + external_services |
| `node.schema.json` | 단일 서버 — site_id, cluster_roles, spec(↓) |
| `node-spec.schema.json` | spec 본체 — vcpu, memory, disks, gpu(_chassis), power, form_factor |
| `cluster.schema.json` | k8s cluster — node_ids, addons, namespaces, network_policies, workloads, ha_control_plane |
| `workload.schema.json` | workload + resources + runtime + network + depends_on + persistent_volumes + hide_from_dep_graph |
| `catalog-entry.schema.json` | 사전 정의된 노드 스펙 항목 |

버전: `v0.7` (`SCHEMA_VERSION` 상수). 0.6→0.7에서 **서비스 준비 Checklist** 상태(`project.checklist` + `project.checklist_custom`)가 추가됐다. `migrateProject()`가 구버전 프로젝트를 로드 시 자동 마이그레이션.

> **런타임 모델 ⊃ 정적 스키마** — 위 `.json` 파일은 코어 형상만 정의한다. 이후 추가된 `workload.ha`(§13.1) · `node.spec.gpu[].mig_enabled/mig_profile`(§14) · `project.checklist`(§10) · `project.concept`(§11)은 현재 `index.html` 런타임 모델에만 존재하며 정적 스키마 파일에는 아직 미반영. 자세한 변경 이력은 [`schema/README.md`](./schema/README.md).

---

## 19. 영속성 · Import / Export

- **저장**: 모든 편집은 자동으로 `localStorage['vk8s.project.current']`에 직렬화 (`saveSoon()` 디바운스). Checklist 체크/메모/사용자 항목도 프로젝트 JSON에 함께 저장.
- **Import**: 상단 ⤓ 버튼 — JSON 파일을 로드해 현재 프로젝트 교체.
- **Export (project)**: 상단 ⤒ 버튼 — 현재 프로젝트를 단일 JSON으로 다운로드.
- **산출물 (개념도 등)**: §11 참조. · **Export (YAML/Helm/Compose/Report)**: §12 참조.
- **공장 초기화**: Splash에서 `Reset` (모든 localStorage 삭제).

외부로 나가는 통신은 export 다운로드 외에 **없음** (보안망/에어갭 환경 친화).

---

## 20. 키보드 단축키

| 키 | 동작 |
|---|---|
| `[` | 좌측 트리 토글 |
| `]` | 우측 인스펙터 토글 |
| `Ctrl/Cmd + B` | Findings 패널 펼치기/접기 |
| `Esc` | 선택 해제 |

자세히는 [`docs/ui-spec.md` §10](./docs/ui-spec.md).

---

## 21. 디렉터리 구조

```
vk8s/
├── index.html                    # 단일 SPA — 모든 뷰/로직/스타일 포함
├── README.md                     # (이 문서)
├── data/
│   ├── gpu-models.json           # GPU 카탈로그 (+ mig_profiles)
│   ├── runtime-profiles.json     # 언어·프레임워크 디폴트
│   └── catalog/*.json            # 노드 스펙 카탈로그
├── docs/
│   └── ui-spec.md                # UI 설계 명세
└── schema/
    ├── README.md                 # 스키마 개요 + 변경 이력
    ├── validator-rules.md        # 모든 검증 룰
    ├── project.schema.json
    ├── site.schema.json
    ├── node.schema.json
    ├── node-spec.schema.json
    ├── cluster.schema.json
    ├── workload.schema.json
    └── catalog-entry.schema.json
```

---

## 22. 라이선스

[LICENSE](./LICENSE) 참조.

FCStudio · `https://github.com/flyingcatstudio/vk8s`
