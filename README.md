# vk8s — k8s capacity studio

가상 Kubernetes 환경(IDC / Cloud / Hybrid)을 설계하고, GPU VRAM까지 인지하는
스케줄 시뮬레이션 · 의존성 그래프 · 정적 병목 분석 · K8s YAML / Helm / Capacity 리포트
산출을 한 페이지에서 수행하는 **브라우저 단독** 도구.

- **백엔드 없음** · 모든 상태는 `localStorage`. 의도적 Export 외에 외부 통신 0.
- **즉시 검증** · 매 편집 후 300ms 디바운스로 validator 재실행.
- **하나의 데이터 · 일곱 개의 렌즈** · Topology / Architecture / Capacity / Dependency / Namespace / Bottleneck / Export 모두 같은 JSON을 다른 관점으로 본다.

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
10. [뷰 7 — Export (K8s YAML / Helm / Capacity)](#10-뷰-7--export-k8s-yaml--helm--capacity)
11. [Workload Inspector — 5탭](#11-workload-inspector--5탭)
12. [GPU 모델링 — chassis · 실장착 · VRAM 소비](#12-gpu-모델링--chassis--실장착--vram-소비)
13. [Quick Deploy 템플릿 & Project 시드](#13-quick-deploy-템플릿--project-시드)
14. [Validator — 검증 룰 카탈로그](#14-validator--검증-룰-카탈로그)
15. [카탈로그 데이터 (GPU · Runtime · Node 스펙)](#15-카탈로그-데이터-gpu--runtime--node-스펙)
16. [스키마 & 버전](#16-스키마--버전)
17. [영속성 · Import / Export](#17-영속성--import--export)
18. [키보드 단축키](#18-키보드-단축키)
19. [디렉터리 구조](#19-디렉터리-구조)
20. [라이선스](#20-라이선스)

---

## 1. Quick Start

```bash
# 1) 그냥 index.html을 더블클릭하거나
open vk8s/index.html

# 2) 정적 호스팅 (예: aiotool.net) 하위 경로로 배포
# 백엔드/빌드 단계 없음.
```

처음 진입하면 Splash에서 `On-Prem / Cloud / Hybrid` 중 하나를 고르고 프로젝트명을 입력하거나,
**Import existing JSON**으로 기존 프로젝트를 복원한다. Project Dashboard에 들어가면
바로 Sites · Nodes · Clusters · Workloads를 자유롭게 편집할 수 있다.

---

## 2. 4계층 데이터 모델

```
Project ──┬── Sites      ── Storage / L4 LB / NetworkDevice / ExternalService
          ├── Nodes      (spec: form_factor, vcpu, memory_gb, disks, gpu, gpu_chassis, power)
          ├── Clusters   ── Namespaces / NetworkPolicies / Workloads / Addons
          └── (catalog)  ── 사전정의된 node-spec, gpu-model, runtime-profile
```

- **Site** — 물리 IDC 또는 클라우드 리전. onprem이면 `capacity.rack_units / power_kw / cooling_kw`를 갖는다.
- **Node** — 단일 서버. `cluster_roles`에 `control-plane` / `worker` 부여. site에 속하고 cluster에 멤버로 들어감.
- **Cluster** — k8s 클러스터. `node_ids`로 노드 멤버 관리. CNI / CSI / GPU Operator / Local PV / Ingress 등 addon, HA 옵션, 네임스페이스, 네트워크 정책, 워크로드 보유.
- **Workload** — Deployment / StatefulSet / DaemonSet / Job / CronJob 단위. replicas, requests/limits, GPU 요청, PVC, runtime(언어·프레임워크·동시성 모델), 네트워크(Service · Ingress), `depends_on`.

스키마 정의: [`schema/`](./schema/) · 자세한 설명: [`schema/README.md`](./schema/README.md).

---

## 3. 상단 / 좌측 트리 / 우측 인스펙터

```
┌──────────────────────────────────────────────────────────────────┐
│ 🧵 vk8s   project-name   [hybrid]                  [💾] [⤓] [⤒]   │  ← Top bar
├────────┬──────────────────────────────────────────┬──────────────┤
│ Tree   │  Canvas (7개 뷰 중 하나)                  │ Inspector   │
│ Sites  │  [Topology][Architecture][Capacity][Dep] │ 선택된       │
│ Nodes  │  [Namespace][Bottleneck][Export]         │ 엔티티 폼    │
│ Cluster│                                          │              │
│ Workld │                                          │              │
├────────┴──────────────────────────────────────────┴──────────────┤
│ ⚠ N errors  ▲ M warnings  ℹ K info       [Findings ▾]            │  ← Findings strip
└──────────────────────────────────────────────────────────────────┘
```

- **Top bar** — 프로젝트명 · 모드 뱃지(`onprem`/`cloud`/`hybrid`) · 저장/Import/Export.
- **좌측 트리** — Sites · Nodes · Clusters · Workloads (열고 닫기 가능). 클릭하면 인스펙터에 폼이 뜸.
- **인스펙터** — 선택된 엔티티의 모든 필드를 폼으로 노출. `🔒` 토글로 우발 편집/삭제 보호 가능.
- **Findings strip** — error/warn/info 건수를 항상 표시. 펼치면 상세 + 클릭 시 해당 엔티티로 점프.

UI 명세 원문: [`docs/ui-spec.md`](./docs/ui-spec.md).

---

## 4. 뷰 1 — Topology

물리/논리 토폴로지를 site → node → cluster → workload 순으로 그래프로 본다.

- **3개 레이아웃**: `grid` (자동), `free` (자유 배치 — 드래그로 이동), `diagram` (SVG 다이어그램).
- **레이어 토글**: Compute · Network · Services · Ingress · Sites 각각 켜고 끄기.
- **다이어그램 모드 export**: 현재 뷰를 **SVG / PNG**로 저장 (`Bottleneck rule transparency` 커밋 이후).
- 같은 cluster의 node-cluster 연결선, l4_balancer → cluster 라인, storage → workload PVC 라인 등 자동 라우팅.

---

## 5. 뷰 2 — Architecture

워크로드를 카테고리(Frontend/API/WAS/DB/Cache/Queue/ML-Inference/...) 또는 zone(라벨 기준)으로 묶어 보는 *논리* 아키텍처.

- `archGroupByZone` 토글 — 라벨 기반 그룹 ↔ 카테고리 그룹.
- ingress → frontend → API → DB 흐름이 한눈에 들어오게 화살표 라우팅.

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
- 자세한 분기 로직은 §12 참조.

---

## 7. 뷰 4 — Dependency Graph

`workload.depends_on`을 좌→우 layered DAG로 시각화. 진입점(아무도 호출 안 하는 노드)이 가장 왼쪽,
싱크(leaf)가 오른쪽. 점선·주황은 cycle (back edge).

### 7.1 노드를 가로지르지 않는 edge 라우팅

2-layer 이상 건너뛰는 edge는 중간 layer의 노드 row를 회피하는 *gutter lane*으로 우회.
사전 패스로 intervening 노드들의 y 중심을 모은 뒤 *위/아래/사이 gap* 중 (y1+y2)/2에 가장 가까운
lane을 선택, cubic bezier control point를 overshoot시켜 곡선 중점이 lane에 정확히 닿게 한다.
overshoot가 viewBox를 벗어나지 않게 SVG 영역도 함께 확장.

### 7.2 Cluster 필터

상단에 cluster 탭 (`All (N) / cluster-A (n1) / cluster-B (n2) ...`).
선택 시 해당 cluster의 워크로드만 노출. cross-cluster 의존이 숨겨졌으면
`▲ cross-cluster 의존 N개가 숨겨져 있습니다 — 'All' 탭에서 확인하세요` 배너로 명시.

### 7.3 워크로드별 그래프 숨김

`workload.hide_from_dep_graph: true`면 해당 워크로드 노드와 연결된 edge가 그래프에서 제외.
검증·export·다른 뷰에는 전혀 영향 없음 — 순수 시각화 옵션. workload Inspector → Basic 탭에서 토글.

### 7.4 Edge label 토글

View-local 옵션 (프로젝트 데이터에는 저장 안 됨). 우측 상단 체크박스
`edge label (protocol·latency)`로 라벨(`HTTP / 15ms` 등)을 한 번에 끄고 켤 수 있다.

---

## 8. 뷰 5 — Namespace & NetworkPolicy

namespace 별 워크로드 분포, ResourceQuota, NetworkPolicy(ingress/egress) 시각화.

- namespace에 `default_deny: true`면 모든 egress/ingress가 명시적 allow 필요.
- `prod`로 시작하는 namespace에 default_deny가 없으면 경고.
- workload의 `depends_on` 호출이 NetworkPolicy로 차단되면 block 룰 발동
  (`WORKLOAD_DEPS_BLOCKED_BY_NETPOL` 등 — §14 참조).

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
(rule transparency 커밋). validator 룰 카탈로그(§14)와 1:1 대응.

---

## 10. 뷰 7 — Export (K8s YAML / Helm / Capacity)

3개 탭 + 마스터 다운로드 버튼.

### 10.1 K8s Manifests (`*.yaml`)

`kubectl apply -f` 가능한 multi-doc YAML. 포함 리소스:

- Namespace · ResourceQuota · NetworkPolicy
- Deployment · StatefulSet (PVC 마운트 포함)
- Service (ClusterIP / NodePort / LoadBalancer)
- Ingress
- PersistentVolumeClaim
- `nvidia.com/gpu` resource limit (count 기반 — §12의 caveat 참조)

### 10.2 Helm Chart (`*.zip`)

워크로드별 umbrella chart 패턴.

- `Chart.yaml` · `values.yaml` · `templates/` · `_helpers.tpl`
- 워크로드별 values 분리. CLI에서 `helm install -f values.yaml`.
- ZIP은 store-only(deflate 미사용) → 브라우저만으로 zip 생성 가능.
- `HELM_DUPLICATE_CHART_NAME` / `HELM_INGRESS_NO_HOST` / `HELM_NO_IMAGE_REPO` 검증 동시 동작.

### 10.3 Capacity Report (`*.md`)

발주서/제안서에 그대로 첨부할 수 있는 마크다운.

- 사이트별 U·kW 산정과 잔여 헤드룸
- GPU 플랜 (모델별 카드/VRAM)
- 네트워크 플랜 (L4 LB · CIDR · CNI)
- 검증 요약 (error/warn 건수와 코드)

### 10.4 ⬇ Download everything (`*.zip`)

위 3개를 한 ZIP에 묶어 다운로드. 파일명에 날짜 stamp.

---

## 11. Workload Inspector — 5탭

| 탭 | 내용 |
|---|---|
| **Basic** | name, category, kind, replicas, namespace, cluster (read-only), 🔒 lock, **Dependency 그래프에서 숨기기** |
| **Resources** | requests/limits (CPU·Memory), **GPU VRAM(GB)** + model 드롭다운(cluster 장착 모델만), PVC (size_gb · access_mode · storage_class · target_storage), Cluster total 합산 |
| **Runtime** | 언어/프레임워크/버전, concurrency 모델(thread-pool / async-await / event-loop / worker-pool / process), thread/connection pool 사이즈, traffic profile (RPS · avg_ms · peak_factor) |
| **Network** | Service (type · ports), Ingress (host · path · TLS), HPA(min/max · cpu_target_pct) |
| **Deps** | `depends_on[]` 편집 — target_workload_id · protocol · port · avg_call_ms · fanout_pct · calls_per_request |

---

## 12. GPU 모델링 — chassis · 실장착 · VRAM 소비

세 계층으로 나뉘어 검증된다.

1. **chassis** (`node.spec.gpu_chassis`): 노드가 *허용*하는 GPU. `max_count`, `supported_form_factors`,
   `supported_models`, `nvlink_topology(none/bridge/nvswitch-mesh)`. 발주 가능한 슬롯 정의.
2. **실장착** (`node.spec.gpu[]`): 실제 꽂혀 있는 카드. `{ model, count }[]`.
   - chassis 룰로 검증 (`GPU_CHASSIS_NOT_SUPPORTED` · `GPU_COUNT_EXCEEDS_CHASSIS` · `GPU_FORM_FACTOR_MISMATCH` · `GPU_MODEL_NOT_IN_CHASSIS_LIST`).
   - NVLink 토폴로지가 `none`인데 카드가 NVLink-capable인 경우 `NVLINK_ON_NONE_TOPOLOGY` 등 추가 검증.
3. **워크로드 소비** (`workload.resources.requests.gpu`):
   - **VRAM 모드 (권장)** — `{ vram_gb, model? }` (count는 schema 호환 위해 1로 자동). 시뮬레이터는 노드의 모델별 VRAM 풀에서 GB로 차감.
   - **count 모드 (legacy)** — `{ count, model? }`. 카드 단위 매칭.
   - 모드 분기: `vram_gb > 0`이면 VRAM 모드.

**Caveat** — K8s YAML/Helm export는 여전히 `nvidia.com/gpu: count` 정수만 emit (실 K8s API의 한계).
VRAM 모드 워크로드는 export 시 `nvidia.com/gpu: 1`이 됨. 시뮬레이션과 시각화는 VRAM 기반으로 정확.

---

## 13. Quick Deploy 템플릿 & Project 시드

### 13.1 Quick Deploy 워크로드 (한 클릭 추가)

- **DB**: Postgres / MySQL / MongoDB
- **Cache**: Redis
- **Queue**: Kafka / RabbitMQ
- **Search**: Elasticsearch
- **ML**: vLLM
- 각 템플릿에 PVC 시드, Service port, replicas, HA 기본값 포함.

### 13.2 Project 시드

처음 만들 때 셋 중 선택:

- **On-Prem** — Dell XE9680 + R660 기반의 GPU/CP 노드 구성 예시.
- **Cloud** — AWS m6i / g5 등 EC2 인스턴스 시드.
- **Hybrid** — 둘 다.

또는 ML 추론 플랫폼 1-click: `LLM Inference (HA)` — 3× R660 cp + 1× XE9680(H200×8) + vLLM + Embedding + Qdrant + Redis + API gateway.

---

## 14. Validator — 검증 룰 카탈로그

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
| **Addon 권장** | `GPU_NODE_NO_GPU_OPERATOR` · `NO_CNI_SELECTED` · `NO_CSI_SELECTED_BUT_PVC_USED` |
| **무결성** | `SITE_NODE_BACKREF_MISMATCH` · `MODE_SITE_KIND_CONFLICT` |
| **Namespace / NetPol** | `WORKLOAD_NAMESPACE_NOT_DECLARED` · `WORKLOAD_DEPS_BLOCKED_BY_NETPOL` · `WORKLOAD_EGRESS_BLOCKED_BY_NETPOL` · `NS_DEFAULT_DENY_NO_ALLOW_RULES` · `NS_RESOURCE_QUOTA_EXCEEDED_BY_WORKLOADS` · `PROD_NS_NO_DEFAULT_DENY` · `NETPOL_TARGETS_NONEXISTENT_POD` · `NETPOL_FROM_NS_NONEXISTENT` |
| **Dependency** | `DEPENDS_ON_TARGET_NOT_FOUND` · `DEPENDS_ON_CYCLE` · `DEPENDS_ON_CROSS_CLUSTER` |
| **정적 병목** | `THREAD_POOL_SATURATION` · `EVENT_LOOP_CPU_BOUND` · `WORKER_POOL_SATURATION` · `DB_CONNECTION_POOL_EXHAUSTED` · `DB_MAX_CONNECTIONS_EXCEEDED` · `TIMEOUT_CASCADE_RISK` · `JVM_HEAP_TOO_SMALL` · `JVM_HEAP_GT_REQUEST` · `SPOF_RISK` · `STATEFUL_NO_PVC` · `RPS_NEVER_REACHED_BY_DEPS` · `KEEPALIVE_HIGH_RPS_DISABLED` |
| **Helm Export** | `HELM_DUPLICATE_CHART_NAME` · `HELM_INGRESS_NO_HOST` · `HELM_NO_IMAGE_REPO` |

각 룰은 `error(block)` / `warn` / `info` 중 하나의 severity를 갖는다.

---

## 15. 카탈로그 데이터 (GPU · Runtime · Node 스펙)

### 15.1 GPU 모델 — `data/gpu-models.json`

NVIDIA Hopper / Ampere / Ada 라인업 + 일부 AMD MI. 각 항목:

```json
"H200-SXM": {
  "vendor": "NVIDIA", "family": "Hopper", "form_factor": "SXM",
  "vram_gb": 141, "tdp_w": 700, "fp16_tflops": 1979,
  "released": 2024, "nvlink_capable": true
}
```

### 15.2 Runtime profile — `data/runtime-profiles.json`

언어/프레임워크별 동시성 모델 디폴트 (thread-pool 크기, async-await 효율, GIL, JVM heap 비율 등).
워크로드 Runtime 탭에서 자동 시드되어 Bottleneck 룰의 입력으로 쓰임.

### 15.3 Node 카탈로그 — `data/catalog/*.json`

Dell PowerEdge, AWS EC2, NCP, Azure VM 등 사전 정의된 노드 스펙.
`chassis`, `form_factor`, `vcpu`, `memory_gb`, `disks`, `gpu_chassis`, `power.psu_max_w`까지 채워져 있어
"Add Node" 마법사에서 한 번에 시드 가능.

스키마 정의: [`schema/catalog-entry.schema.json`](./schema/catalog-entry.schema.json).

---

## 16. 스키마 & 버전

JSON Schema (Draft 2020-12) 7개:

| 파일 | 정의 |
|---|---|
| `project.schema.json` | 최상위 — name, mode, sites, nodes, clusters |
| `site.schema.json` | site + capacity + storages + l4_balancers + network_devices + external_services |
| `node.schema.json` | 단일 서버 — site_id, cluster_roles, spec(↓) |
| `node-spec.schema.json` | spec 본체 — vcpu, memory, disks, gpu(_chassis), power, form_factor |
| `cluster.schema.json` | k8s cluster — node_ids, addons, namespaces, network_policies, workloads, ha |
| `workload.schema.json` | workload + resources + runtime + network + depends_on + persistent_volumes + hide_from_dep_graph |
| `catalog-entry.schema.json` | 사전 정의된 노드 스펙 항목 |

버전: `v0.6` (`SCHEMA_VERSION` 상수). 자세한 변경 이력은 [`schema/README.md`](./schema/README.md).

---

## 17. 영속성 · Import / Export

- **저장**: 모든 편집은 자동으로 `localStorage['vk8s.project.current']`에 직렬화 (`saveSoon()` 디바운스).
- **Import**: 상단 ⤓ 버튼 — JSON 파일을 로드해 현재 프로젝트 교체.
- **Export (project)**: 상단 ⤒ 버튼 — 현재 프로젝트를 단일 JSON으로 다운로드.
- **Export (산출물)**: §10 참조.
- **공장 초기화**: Splash에서 `Reset` (모든 localStorage 삭제).

외부로 나가는 통신은 export 다운로드 외에 **없음** (보안망/에어갭 환경 친화).

---

## 18. 키보드 단축키

| 키 | 동작 |
|---|---|
| `[` | 좌측 트리 토글 |
| `]` | 우측 인스펙터 토글 |
| `Ctrl/Cmd + B` | Findings 패널 펼치기/접기 |
| `Esc` | 선택 해제 |

자세히는 [`docs/ui-spec.md` §10](./docs/ui-spec.md).

---

## 19. 디렉터리 구조

```
vk8s/
├── index.html                    # 단일 SPA — 모든 뷰/로직/스타일 포함
├── README.md                     # (이 문서)
├── data/
│   ├── gpu-models.json           # GPU 카탈로그
│   ├── runtime-profiles.json     # 언어·프레임워크 디폴트
│   └── catalog/*.json            # 노드 스펙 카탈로그
├── docs/
│   └── ui-spec.md                # UI 설계 명세 (519 lines)
└── schema/
    ├── README.md                 # 스키마 개요 + 변경 이력
    ├── validator-rules.md        # 모든 검증 룰 (337 lines)
    ├── project.schema.json
    ├── site.schema.json
    ├── node.schema.json
    ├── node-spec.schema.json
    ├── cluster.schema.json
    ├── workload.schema.json
    └── catalog-entry.schema.json
```

---

## 20. 라이선스

[LICENSE](./LICENSE) 참조.

FCStudio · `https://github.com/flyingcatstudio/vk8s`
