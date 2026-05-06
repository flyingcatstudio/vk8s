# vk8s UI Specification (v0.1)

대상: 단일 페이지 정적 HTML 앱 (loom과 동일 호스팅 패턴).
런타임: 브라우저만, 백엔드 없음. 프로젝트는 JSON 한 덩어리로 export/import.

---

## 1. 설계 원칙

| # | 원칙 | 의미 |
|---|---|---|
| 1 | **자유 편집 우선** | 트리·캔버스·인스펙터 어디서든 같은 데이터 수정. 마법사는 처음 시작과 Add Node에만. |
| 2 | **즉시 검증** | 매 편집 후 300ms 디바운스로 validator 재실행. 결과는 항상 가시. |
| 3 | **하나의 데이터, 여러 뷰** | Topology / Capacity / Dependency / Namespace / Bottleneck / Export — 같은 데이터를 다른 렌즈로. |
| 4 | **시야 우선** | 캔버스가 주, 패널은 보조. 좌·우 패널 토글로 캔버스 100%까지 확장 가능. |
| 5 | **로컬 우선** | 모든 상태 localStorage. 의도적 export까지 외부 통신 없음. |
| 6 | **loom과 통일된 톤** | 다크 테마, Pretendard/Outfit 폰트, accent #4d9cff. 사용자가 두 도구를 한 가족으로 인지. |

---

## 2. 글로벌 레이아웃

```
┌─────────────────────────────────────────────────────────────────────┐
│ 🧵 vk8s   ai-platform-2026   [hybrid]   ⚙️         [💾] [⤓] [⤒]    │  ← Top bar
├──────┬──────────────────────────────────────────────────┬───────────┤
│      │                                                    │           │
│ Tree │                  Canvas (현재 뷰)                  │ Inspector │
│      │                                                    │           │
│ 📁   │   [Topology] [Capacity] [Dependency] [Namespace]  │  선택된    │
│ ├ S  │   [Bottleneck] [Export]                            │  엔티티    │
│ ├ N  │                                                    │  속성      │
│ ├ C  │                                                    │  (폼)      │
│ └ W  │                                                    │           │
│      │                                                    │           │
├──────┴────────────────────────────────────────────────────┴───────────┤
│ ⚠ 3 errors  ▲ 7 warnings  ℹ 4 info       [Findings ▾]                │  ← Findings strip
└─────────────────────────────────────────────────────────────────────┘
```

### 영역
- **Top bar (48px)** — 프로젝트명, 모드 뱃지, 설정, 저장/Import/Export
- **Tree (left, 240px, collapsible)** — 엔티티 계층
- **Canvas (center, flex)** — 6개 뷰 중 선택된 뷰
- **Inspector (right, 360px, collapsible)** — 선택된 엔티티의 폼
- **Findings strip (bottom, 32px → expanded 240px)** — 검증 결과 요약 + 펼치면 상세

### 패널 토글 키
- `[` 좌측 트리 토글
- `]` 우측 인스펙터 토글
- `Ctrl/Cmd+B` Findings 펼치기

---

## 3. 시작 흐름 (Splash / New Project)

처음 진입 시 또는 New Project 시:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    vk8s — k8s capacity studio               │
│                                                             │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│   │              │  │              │  │              │    │
│   │   On-Prem    │  │    Cloud     │  │    Hybrid    │    │
│   │              │  │              │  │              │    │
│   │ IDC, 직접     │  │ AWS, NCP,   │  │ 둘 다         │    │
│   │ 보유 서버     │  │ Azure, GCP   │  │              │    │
│   └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                             │
│        프로젝트명 [ai-platform-2026          ]              │
│                                                             │
│   [Import existing JSON]              [Cancel] [Create →]   │
└─────────────────────────────────────────────────────────────┘
```

3택 카드 + 이름 입력 + (선택) Import. 결정 후 Project Dashboard로 진입.

---

## 4. Tree (좌측)

```
📁 ai-platform-2026  hybrid

▾ Sites (2)
   ◾ idc-seoul-1   onprem  84U / 32kW
   ◾ aws-apne2     cloud   AWS ap-northeast-2

▾ Nodes (4)
   ▾ idc-seoul-1
      ◽ node-gpu-1  Dell XE9680  6U  10.9kW
      ◽ node-gpu-2  Dell XE9680  6U  10.9kW
   ▾ aws-apne2
      ◽ node-api-1  AWS m6i.4xlarge
      ◽ node-api-2  AWS m6i.4xlarge

▾ Clusters (2)
   ▾ ai-prod
      ▾ namespaces (3)
         ◽ frontend
         ◽ backend  🔒 default-deny
         ◽ data     🔒 default-deny
      ▾ workloads (5)
         ◽ user-api      backend     ⚠
         ◽ web-frontend  frontend
         ◽ postgres      data        ⚠ SPOF
         ◽ redis         data
         ◽ kafka         data
      ▾ network policies (3)
   ▾ ai-dev (...)

[+ Add Site]  [+ Add Node]  [+ Add Cluster]
```

- 우클릭 → 컨텍스트 메뉴 (rename / delete / duplicate / move to cluster)
- 드래그앤드롭으로 노드 → 클러스터 이동
- 검색 박스 (`Ctrl/Cmd+K`)
- finding 있는 항목은 ⚠/❌ 인디케이터

---

## 5. Canvas — 6개 뷰

### 5.1 Topology View (기본)

```
┌──── Site: idc-seoul-1 ─────────────────────────────────┐
│  capacity:  84U  ████░░░░░░░ 12U used (14%)            │
│             32kW ████████░░░ 21.8kW used (68%)         │
│                                                        │
│   ┌─── node-gpu-1 ───────┐  ┌─── node-gpu-2 ───────┐ │
│   │ Dell XE9680  6U      │  │ Dell XE9680  6U      │ │
│   │ 112 vCPU  2048 GB    │  │ 112 vCPU  2048 GB    │ │
│   │ 🟩 GPU 8/8  H200 SXM │  │ 🟩 GPU 6/8  H200 SXM │ │
│   │ ⚡ 10.9 kW            │  │ ⚡ 10.9 kW            │ │
│   │                      │  │                      │ │
│   │  ▣ llm-infer-x4      │  │  ▣ llm-infer-x3      │ │
│   │  ▣ rag-api-x2        │  │  ▣ vector-db         │ │
│   └──────────────────────┘  └──────────────────────┘ │
└────────────────────────────────────────────────────────┘

┌──── Site: aws-apne2 ───────────────────────────────────┐
│   ┌─── node-api-1 ───────┐  ┌─── node-api-2 ───────┐ │
│   │ AWS m6i.4xlarge      │  │ AWS m6i.4xlarge      │ │
│   │ ▣ user-api-x2  ▣ ... │  │ ▣ user-api-x1  ▣ ... │ │
│   └──────────────────────┘  └──────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

- 박스-인-박스 (Site > Node > Pod)
- 박스 드래그로 위치 조정 (저장됨)
- 박스 클릭 → Inspector 채움
- 줌/팬 (`+`/`-`/`0`/마우스휠)
- 미니맵 우하단

### 5.2 Capacity View

각 노드를 진행률 바로 표현. 한눈에 사용률 파악.

```
node-gpu-1   Dell XE9680                                 6U
   CPU       ████████████████████░░░░░░░░░  64%  72/112c
   Memory    ██████████████████░░░░░░░░░░  60%  1228GB
   GPU       ████████████████████████████  100% 8/8
   VRAM      ████████████████████████████  100% 1128/1128GB
   Disk      ██████░░░░░░░░░░░░░░░░░░░░░░  20%  800GB
   Power     ██████████████████████████░░  91%  10.9kW  ⚠
                                                ──── 헤드룸 부족

node-gpu-2   Dell XE9680                                 6U
   CPU       ██████░░░░░░░░░░░░░░░░░░░░░░  20%
   GPU       ██████████████████████░░░░░░  75%  6/8
   ...
```

색상: 0-50% 녹, 50-80% 청, 80-90% 황, 90%+ 적.

### 5.3 Dependency Graph View

```
        ┌─────────┐
        │ inguser │  external traffic
        └────┬────┘
             │
        ┌────▼────────┐
        │ web-frontend│ frontend ns
        └────┬────────┘
             │ HTTP 8080
             │ 1000 RPS
        ┌────▼────────┐
        │ user-api    │ backend ns  ⚠ thread pool
        └────┬────────┘
             │ JDBC 5432
             │ 2000 calls/s
        ┌────▼────────┐
        │ postgres    │ data ns     ⚠ max_conn
        └─────────────┘
```

- 노드: 워크로드 박스
- 엣지: depends_on
- 엣지 색상: 녹 = NetworkPolicy 통과 / 적 = 차단 / 회 = NP 미정의
- 엣지 두께: RPS 비례
- 노드 색상: 병목 심각도 (녹/황/적)

### 5.4 Namespace View

각 클러스터를 NS 칸으로 분할.

```
┌──── Cluster: ai-prod ──────────────────────────────────────┐
│                                                            │
│ ┌── frontend ──────┐  ┌── backend 🔒 ─────┐  ┌── data 🔒 ─┐│
│ │                  │  │                   │  │            ││
│ │  ▣ web-frontend  │──▶  ▣ user-api      │──▶ ▣ postgres ││
│ │      x3          │  │      x3           │  │    x1     ⚠││
│ │                  │  │                   │  │            ││
│ │                  │  │                   │  │  ▣ redis  ││
│ │                  │  │                   │  │     x1     ││
│ └──────────────────┘  └───────────────────┘  └────────────┘│
│       no policies           ingress allow:        ingress: │
│                             frontend              from backend│
└────────────────────────────────────────────────────────────┘
```

- NS 박스 = 영역, default-deny면 🔒 자물쇠
- NS 간 화살표 = 허용된 트래픽
- 차단된 의존성은 빨간 점선 화살표 + ❌

### 5.5 Bottleneck View

테이블형. 병목 의심 워크로드만 모음.

```
WORKLOAD          NS        SEVERITY  RULE                            DETAIL
user-api          backend   ⚠ warn   THREAD_POOL_SATURATION          200T 부족, 257T 필요
user-api          backend   ⚠ warn   DB_CONNECTION_POOL_EXHAUSTED    pool 30, 호출량 80
postgres          data      ⚠ warn   DB_MAX_CONNECTIONS_EXCEEDED     90/100 연결
postgres          data      ⚠ warn   SPOF_RISK                       replicas=1, kind=DB
checkout-service  backend   ⚠ warn   TIMEOUT_CASCADE_RISK            client_to=2s < chain p99 3.4s
ml-trainer        ml        ⚠ warn   JVM_HEAP_TOO_SMALL              heap 2GB < req 8GB

[Apply auto-fix suggestions]   [Export bottleneck report .md]
```

행 클릭 → 해당 워크로드 inspector + 캔버스에서 강조.

### 5.6 Export View

```
┌─ Export Artifacts ─────────────────────────────────────────┐
│                                                            │
│  Target environment                                        │
│  ▣ AWS (Cloud sites)        ▣ On-prem (idc-seoul-1)       │
│                                                            │
│  ──────────────────────────────────────────────────────    │
│  Infra provisioning                                        │
│  ▢ Terraform (AWS)          → terraform/aws/               │
│  ▣ kubeadm scripts (onprem) → scripts/kubeadm/             │
│  ▢ MAAS preseed             → maas/                        │
│                                                            │
│  k8s manifests                                             │
│  ▣ Plain YAML               → manifests/                   │
│  ▣ Helm charts (per workload)→ charts/<workload>/         │
│  ▢ Umbrella chart           → charts/_umbrella/            │
│  ▣ NetworkPolicies          (in respective ns/)            │
│                                                            │
│  Reports                                                   │
│  ▣ Capacity report (.md)                                   │
│  ▣ Bottleneck report (.md)                                 │
│  ▢ Architecture diagram (.svg)                             │
│                                                            │
│              [Cancel]      [⤓ Download .zip]               │
└────────────────────────────────────────────────────────────┘
```

브라우저에서 `.zip` 생성 (jszip 등). 단일 파일 다운로드.

---

## 6. Inspector (우측)

선택된 엔티티에 따라 폼 자동 전환.

### 6.1 Node Inspector 예시

```
─ Node ────────────────────────  [×]
ID         node-gpu-1
Name       [GPU Node 1            ]
Source     ⦿ onprem  ⦾ cloud
Brand      [Dell ▾]
Model      [PowerEdge XE9680 ▾]   custom □
Site       [idc-seoul-1 ▾]
Role       [worker ▾]

─ Spec ─────────────
vCPU       [112    ]
Memory GB  [2048   ]
Form       [6U ▾]   Power [10800 W psu]

─ GPU ─────────────
Chassis: max 8 · SXM · NVSwitch mesh
Supported: H100-SXM, H200-SXM

   Model        Count  FF   NVLink
 1 [H200-SXM]  [8  ]  SXM  ▣
 [+ Add GPU]

─ Findings (this node) ──────────
⚠ SITE_POWER_HIGH_UTILIZATION (site)
   IDC-Seoul-1: 91% — 헤드룸 부족
```

- 모든 필드는 즉시 검증
- "Findings (this node)" 섹션 — 이 엔티티 관련 룰만 필터링
- 카탈로그 모델 선택 시 spec 자동 시드 (사용자 수정 가능 → custom 토글)

### 6.2 Workload Inspector 탭 구성

탭으로 분리 (정보량 많음):
- **Basic** — id, name, namespace, labels, replicas, kind
- **Resources** — CPU/Mem/GPU requests/limits, PVC
- **Runtime** — language/framework, concurrency, heap, max_connections
- **Traffic** — target_rps, latency, timeout
- **Dependencies** — depends_on 편집 (drop-down으로 워크로드 선택)
- **Scheduling** — nodeSelector, tolerations, anti-affinity
- **Helm** — chart_name, image, expose ports/ingress

각 탭에 해당 카테고리 finding만 표시.

---

## 7. Add Node Wizard (마법사)

`+ Add Node` 클릭 시 모달.

```
Step 1/4 — Source
  ⦿ On-Prem  ⦾ Cloud
  Site [idc-seoul-1 ▾]                          [Cancel] [Next →]

Step 2/4 — Brand & Category
  Brand:     [Dell] [HPE] [Lenovo] [Supermicro]
  Category:  [All] [General] [Compute] [Memory] [GPU] [Storage]
                                                [← Back] [Next →]

Step 3/4 — Model
  ┌──────────────────────────────────────────────────┐
  │ ⦿ PowerEdge R660       1U  General  no GPU       │
  │ ⦾ PowerEdge R760xa     2U  GPU  4× PCIe-FH      │
  │ ⦾ PowerEdge XE9680     6U  GPU  8× SXM   ★      │
  │ ⦾ PowerEdge XE8640     4U  GPU  4× SXM           │
  └──────────────────────────────────────────────────┘
  검색: [h200            🔍 ]                  [← Back] [Next →]

Step 4/4 — Configure
  Quantity      [×3]
  GPU model     [H200-SXM ▾]   Count [8]   NVLink ▣
  Customize:    □ override spec (vCPU/Mem 등)

  Validation:   ✓ chassis 호환  ✓ NVLink ok  ⚠ site 전력 32 → 65kW (over)

                                  [← Back] [✓ Add to project]
```

마지막 단계의 실시간 검증이 핵심. 추가 전에 capacity 초과 등 파악.

Cloud 모드는 Step 4가 단순 (인스턴스 타입이 GPU/스펙 고정).

---

## 8. Findings Strip & Panel

### 접힌 상태 (default, 32px)
```
⚠ 3 errors    ▲ 7 warnings   ℹ 4 info               [Findings ▾]
```

### 펼친 상태 (240px)
```
⚠ 3 errors    ▲ 7 warnings   ℹ 4 info       [filter ▾]   [Findings ▴]
─────────────────────────────────────────────────────────────────
⚠  WORKLOAD_DEPS_BLOCKED_BY_NETPOL    user-api → postgres : NP 차단
⚠  CLUSTER_NO_CONTROL_PLANE          ai-dev: control plane 없음
⚠  GPU_FORM_FACTOR_MISMATCH          node-3: PCIe-FH new but chassis SXM only
▲  THREAD_POOL_SATURATION             user-api: 257T 필요, 200T 가용
▲  DB_MAX_CONNECTIONS_EXCEEDED        postgres: 90/100
▲  SITE_POWER_HIGH_UTILIZATION        idc-seoul-1: 91%
▲  SPOF_RISK                          postgres: replicas=1, kind=DB
ℹ  RPS_NEVER_REACHED_BY_DEPS          web-frontend: 자동 추정 RPS=200
```

행 클릭 → 해당 엔티티 트리 + 인스펙터 + 캔버스 강조.
필터: severity / category / scope.

---

## 9. Visual Language

### Theme (loom과 통일)
```
--bg:        #08090c
--surface:   #111318
--surface2:  #181b23
--border:    #252a36
--text:      #e8ecf4
--text2:     #8892a4
--accent:    #4d9cff   (links, primary actions)
--green:     #34d399   (ok)
--yellow:    #fbbf24   (warn)
--red:       #f87171   (error)
--purple:    #a78bfa   (GPU)
--orange:    #fb923c   (power)
```

### Typography
- UI: Pretendard (한국어), Outfit (영문)
- Mono: JetBrains Mono (코드/스펙값)
- Sizes: 11/13/14/16/20/28

### Iconography
이모지 1차 (loom과 동일 — 의존성 X):
- 📁 project · 🏢 site · 🖥 node · 🎯 cluster · ▣ pod · 🔒 default-deny
- 🟩 gpu · ⚡ power · 📊 capacity · 🔗 dependency

후기에 SVG 아이콘 셋 도입 검토 (Lucide).

### 핵심 컴포넌트
- `.badge` — mode/severity 뱃지
- `.bar` — 진행률 (capacity)
- `.card` — 노드/사이트 박스
- `.chip` — 라벨 (k8s labels)
- `.kbd` — 단축키 표시
- `.toast` — 일시 알림 (저장 완료 등)

---

## 10. 키보드

| 키 | 동작 |
|---|---|
| `Ctrl/Cmd+S` | 저장 (localStorage) |
| `Ctrl/Cmd+Z` / `Shift+Z` | undo / redo |
| `Ctrl/Cmd+K` | 트리 검색 |
| `Ctrl/Cmd+B` | Findings 펼치기 |
| `[` / `]` | 좌/우 패널 토글 |
| `1`–`6` | 캔버스 뷰 전환 (Topology…Export) |
| `Esc` | 모달/마법사 취소 |
| `Del` | 선택 엔티티 삭제 (확인 다이얼로그) |
| `+` / `-` / `0` | 캔버스 줌 |

---

## 11. 반응형

- ≥1280px: 풀 3패널
- 1024–1280: 인스펙터 자동 접힘 (탭으로 토글)
- <1024: 트리도 접힘, 햄버거 메뉴 (모바일은 부차 — 데스크톱 도구)

---

## 12. 상태 & 영속

- **로컬 저장** — `localStorage["vk8s.project.<name>"]` JSON 직렬화
- **Multi-project** — 프로젝트 셀렉터 (Top bar)
- **Auto-save** — 편집 후 1초 디바운스
- **Undo stack** — 메모리 in-session, 50 단계
- **Import** — `.json` 파일 드래그앤드롭 또는 버튼
- **Schema migration** — 로드 시 `version` 확인, 0.1 → 0.2 자동 마이그레이션

---

## 13. 접근성

- 키보드 only 풀 동작
- aria-label 모든 인터랙티브 요소
- 색상 외 시그널 (severity는 아이콘+색상 둘 다)
- `prefers-reduced-motion` 존중

---

## 14. v0.1 구현 스코프

UI를 단계적으로:

| 단계 | 포함 | 비고 |
|---|---|---|
| **MVP** | Splash, Tree, Inspector, Topology, Capacity, Findings, Export(plain YAML+report) | 1주 |
| **+1** | Add Node Wizard, Dependency view, Namespace view, Helm export | +3일 |
| **+2** | Bottleneck view, Multi-project, Import | +2일 |
| **나중** | 다이어그램 SVG export, 시각 테마 토글, i18n(en) | v0.2+ |

---

## 15. 기술 결정 (제안)

- **단일 HTML 파일** — loom과 동일 (`index.html` 하나에 CSS/JS 내포)
- **상태 관리** — vanilla. 작은 reactive store 직접 구현 (~80 lines)
- **Canvas** — SVG (DOM 친화, 디버깅 쉬움). loom의 그래프 렌더 코드 재활용 검토
- **번들** — `build.js` 미니파이만. 빌드 도구 없음
- **테스트** — validator JS는 단위 테스트 (Node 빌트인 `node:test`)

---

## 16. 미정 / 다음 회의

이 문서는 v0.1 합의안. 다음 결정 필요:

- [ ] 다국어 (한/영) 우선순위?
- [ ] 실시간 협업 — 안 함 (정적 도구). 그러나 share via URL (압축 JSON to base64)? — 검토만
- [ ] 다이어그램 SVG export 포맷 (loom 호환?) — v0.2+
- [ ] 사용자 피드백 채널 (GitHub Issues로 충분?)

---

**다음 단계**: 이 스펙으로 OK면 `vk8s/index.html` MVP 시작. 변경/추가 의견 있으면 이 문서에 코멘트.
