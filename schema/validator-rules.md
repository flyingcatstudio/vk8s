# vk8s Validator Rules (v0.3)

> v0.3 변경: `node.cluster_role` (단일 enum) → `node.cluster_roles` (배열). 룰의 control-plane / worker / etcd 카운트는 모두 `roles.includes("...")` 로 계산. dev 단일노드 클러스터에서 한 노드가 control-plane+worker를 동시에 갖는 패턴 지원.



검증 룰 카탈로그. 각 룰은 `id`, `severity`(`block` | `warn`), `scope`(검사 단위), 발동 조건, 메시지 템플릿을 가진다.

`block`은 저장/적용을 막고, `warn`은 경고 표시만 한다.

룰은 데이터 변경 시점(노드 추가/수정, GPU 설정, 워크로드 추가, Site capacity 변경)에 모두 재실행한다.

---

## 1. GPU 호환성 (Node 단위)

### `GPU_CHASSIS_NOT_SUPPORTED` — block
- **scope**: node
- **when**: `node.spec.gpu_chassis.max_count == 0` 인데 `node.spec.gpu` 가 비어있지 않음
- **msg**: `{node.name}: 이 새시({node.model})는 GPU를 지원하지 않습니다.`

### `GPU_COUNT_EXCEEDS_CHASSIS` — block
- **scope**: node
- **when**: `sum(node.spec.gpu[].count) > node.spec.gpu_chassis.max_count`
- **msg**: `{node.name}: GPU 총 {actual}장은 새시 max_count {limit} 초과.`

### `GPU_FORM_FACTOR_MISMATCH` — block
- **scope**: node, per gpu entry
- **when**: `gpu.form_factor` 가 `gpu_chassis.supported_form_factors` 에 포함되지 않음
- **msg**: `{node.name}: {gpu.model}의 폼팩터({gpu.form_factor})는 이 새시에서 지원되지 않음. 지원: {supported}`

### `GPU_MODEL_NOT_IN_CHASSIS_LIST` — block
- **scope**: node, per gpu entry
- **when**: `gpu.model` 이 `gpu_chassis.supported_models` 에 없음
- **msg**: `{node.name}: 이 새시 카탈로그에 {gpu.model} 항목이 없음. 지원 모델: {supported}`

### `GPU_MODEL_UNKNOWN` — block
- **scope**: node, per gpu entry
- **when**: `gpu.model` 이 `data/gpu-models.json` 에 없음
- **msg**: `알 수 없는 GPU 모델: {gpu.model}`

---

## 2. NVLink 룰 (Node 단위)

### `NVLINK_ON_NONE_TOPOLOGY` — block
- **scope**: node, per gpu entry
- **when**: `gpu.nvlink_enabled == true` 이고 `gpu_chassis.nvlink_topology == "none"`
- **msg**: `{node.name}: NVLink 미지원 새시에서 NVLink를 켤 수 없음.`

### `NVLINK_BRIDGE_PAIR_LIMIT` — warn
- **scope**: node, per gpu entry
- **when**: `gpu.nvlink_enabled == true`, `gpu_chassis.nvlink_topology == "bridge-pair"`, `gpu.count > 2 || gpu.count % 2 != 0`
- **msg**: `{node.name}: NVLink Bridge는 2장 페어링만 가능. 일부 GPU는 NVLink 비활성으로 동작.`

### `NVLINK_SXM_NOT_FULLY_POPULATED` — warn
- **scope**: node, per gpu entry
- **when**: `gpu.form_factor == "SXM"`, `gpu_chassis.nvlink_topology == "nvswitch-mesh"`, `gpu.count < gpu_chassis.max_count`
- **msg**: `{node.name}: SXM/NVSwitch 새시는 보통 fully-populated({max}장) 권장. 현재 {count}장.`

---

## 3. Site Capacity (Site 단위, onprem만)

### `SITE_RACK_UNITS_EXCEEDED` — block
- **scope**: site (onprem)
- **when**: `sum(node.form_factor)` > `site.capacity.rack_units` (form_factor 'Tower' = 0)
- **msg**: `{site.name}: {actual}U 요구, {limit}U 가용`

### `SITE_POWER_EXCEEDED` — block
- **scope**: site (onprem)
- **when**: `sum(node_power_w) / 1000 > site.capacity.power_kw`
  - `node_power_w = spec.power.typical_w_base + Σ(gpu.count × gpu_models[gpu.model].tdp_w) + (disks×10W)`
- **msg**: `{site.name}: {actual}kW 요구, {limit}kW 예산`

### `SITE_POWER_HIGH_UTILIZATION` — warn
- **scope**: site (onprem)
- **when**: power_used / power_kw >= 0.9
- **msg**: `{site.name}: 전력 사용률 {pct}% — 헤드룸 부족 (권장 80% 이하).`

### `SINGLE_NODE_EXCEEDS_TYPICAL_PDU` — warn
- **scope**: node (onprem)
- **when**: `node_power_w > 8000` (단일 PDU 한계 추정)
- **msg**: `{node.name}: {kw}kW 단일 노드 — 다중 PDU/회로 필요할 수 있음.`

### `GPU_NODE_NO_COOLING_INFO` — warn
- **scope**: site (onprem)
- **when**: site에 GPU 노드 있음, `site.capacity.cooling_kw` 미설정, GPU 부하 합 > 5kW
- **msg**: `{site.name}: GPU {kw}kW 부하인데 냉방 capacity 미입력.`

---

## 4. Cluster 토폴로지

### `CLUSTER_NO_CONTROL_PLANE` — block
- **scope**: cluster
- **when**: `node_ids` 에 `cluster_role == "control-plane"` 노드가 0개
- **msg**: `{cluster.name}: control plane 노드가 없음.`

### `HA_CONTROL_PLANE_NOT_ENOUGH` — block
- **scope**: cluster
- **when**: `cluster.ha_control_plane == true`, control-plane 노드 수 < 3
- **msg**: `{cluster.name}: HA control plane은 3대 이상 필요 (현재 {n}).`

### `HA_CONTROL_PLANE_EVEN_NUMBER` — warn
- **scope**: cluster
- **when**: control-plane 노드 수가 짝수
- **msg**: `{cluster.name}: control plane 짝수개({n})는 etcd 쿼럼에 비효율. 홀수 권장.`

### `NODE_IN_MULTIPLE_CLUSTERS` — block
- **scope**: project
- **when**: 한 Node.id가 두 개 이상의 cluster.node_ids 에 포함
- **msg**: `노드 {node.name}가 클러스터 여러 개에 속함: {clusters}`

### `CLUSTER_SPANS_ENVIRONMENTS` — warn
- **scope**: cluster
- **when**: 클러스터 노드들이 `kind` 다른 Site에 분산 (cloud + onprem 혼합)
- **msg**: `{cluster.name}: 한 클러스터가 cloud+onprem 양쪽에 걸침 — 네트워킹/지연 검토 필요.`

---

## 5. 워크로드 스케줄링 (Cluster 단위)

### `WORKLOAD_CPU_NOT_SCHEDULABLE` — block
- **scope**: cluster, per workload
- **when**: `workload.requests.cpu × replicas` > `Σ(worker node vCPU) - 다른 워크로드 합`
- **msg**: `{workload.name}: CPU {needed} core 요구, 가용 {available} core`

### `WORKLOAD_MEM_NOT_SCHEDULABLE` — block
- **scope**: cluster, per workload
- **when**: 동일 로직, memory_gb 기준
- **msg**: `{workload.name}: 메모리 {needed}GB 요구, 가용 {available}GB`

### `WORKLOAD_GPU_NO_NODE` — block
- **scope**: cluster, per workload
- **when**: `requests.gpu` 있는데 클러스터 내 GPU 노드 없음
- **msg**: `{workload.name}: GPU 요구하나 클러스터에 GPU 노드 없음.`

### `WORKLOAD_GPU_MODEL_NOT_AVAILABLE` — block
- **scope**: cluster, per workload
- **when**: `requests.gpu.model` 지정, 클러스터 내 해당 모델 GPU 노드 없음
- **msg**: `{workload.name}: {requested_model} 요구, 클러스터에 없음 (보유: {available_models}).`

### `WORKLOAD_GPU_VRAM_INSUFFICIENT` — block
- **scope**: cluster, per workload
- **when**: `requests.gpu.vram_gb` > 클러스터 내 모든 GPU의 vram_gb 최대값
- **msg**: `{workload.name}: VRAM {needed}GB 요구, 최대 {max}GB.`

### `WORKLOAD_REPLICAS_GT_NODES_WITH_AFFINITY` — warn
- **scope**: cluster, per workload
- **when**: `scheduling.anti_affinity == "spread-by-node"`, replicas > worker 노드 수
- **msg**: `{workload.name}: replicas {n} > 워커 노드 {m}. anti-affinity 만족 불가.`

---

## 6. Cluster Add-on 권장

### `GPU_NODE_NO_GPU_OPERATOR` — warn
- **scope**: cluster
- **when**: 클러스터에 GPU 노드 있음, `addons.gpu_operator != true`
- **msg**: `{cluster.name}: GPU 노드 보유 — NVIDIA GPU Operator 활성화 권장.`

### `NO_CNI_SELECTED` — warn
- **scope**: cluster
- **when**: `addons.cni == "None"` or 미설정
- **msg**: `{cluster.name}: CNI 미선택. Pod 네트워킹 동작 안함.`

### `NO_CSI_SELECTED_BUT_PVC_USED` — warn
- **scope**: cluster
- **when**: `addons.csi` 비어있음, 워크로드 중 `persistent_volumes` 사용
- **msg**: `{cluster.name}: PVC 사용 워크로드 있으나 CSI 미설정.`

---

## 7. 무결성

### `SITE_NODE_BACKREF_MISMATCH` — block
- **scope**: project
- **when**: `Node.site_id == S` 인데 `Site.node_ids` 에 해당 노드 없음 (또는 그 반대)
- **msg**: `Node {node}와 Site {site}의 양방향 참조 불일치.`

### `MODE_SITE_KIND_CONFLICT` — block
- **scope**: project
- **when**: `project.mode == "onprem"` 인데 `cloud` Site 존재 (또는 반대)
- **msg**: `프로젝트 모드 {mode}와 Site 종류({kinds}) 불일치.`

---

## 8. Namespace & NetworkPolicy

### `WORKLOAD_NAMESPACE_NOT_DECLARED` — block
- **scope**: cluster, per workload
- **when**: `workload.namespace` 값이 `cluster.namespaces[].name`에 없음
- **msg**: `{workload.name}: namespace '{ns}'가 클러스터에 정의되지 않음.`

### `NS_DEFAULT_DENY_NO_ALLOW_RULES` — warn
- **scope**: cluster, per namespace
- **when**: `namespace.default_deny == true`인데 해당 NS의 NetworkPolicy 중 ingress allow 룰 없음
- **msg**: `Namespace '{ns}': default-deny 적용되어 있고 allow 룰 없음 — 모든 트래픽 차단됨.`

### `WORKLOAD_DEPS_BLOCKED_BY_NETPOL` — block
- **scope**: cluster, per depends_on edge
- **when**: A→B 호출 경로가 B의 namespace NetworkPolicy에 의해 차단됨 (ingress 룰이 A의 ns/labels 매칭 안함)
- **msg**: `{a.name}({a.ns}) → {b.name}({b.ns}): NetworkPolicy '{policy}'에 의해 차단. allow 룰 추가 필요.`

### `WORKLOAD_EGRESS_BLOCKED_BY_NETPOL` — block
- **scope**: cluster, per depends_on edge
- **when**: A의 ns가 default_deny egress이고 A→B 허용 egress 룰 없음
- **msg**: `{a.name}: namespace '{ns}' egress default-deny — {b.name} 호출 불가.`

### `NETPOL_TARGETS_NONEXISTENT_POD` — warn
- **scope**: cluster, per network_policy
- **when**: `pod_selector.match_labels`가 namespace 내 어떤 워크로드 라벨에도 매칭되지 않음
- **msg**: `NetworkPolicy '{policy}': 매칭되는 pod 없음 (의도한 라벨 확인).`

### `NETPOL_FROM_NS_NONEXISTENT` — warn
- **scope**: cluster, per network_policy
- **when**: `ingress.from_namespaces`에 클러스터에 없는 namespace 참조
- **msg**: `NetworkPolicy '{policy}': from_namespaces에 존재하지 않는 ns '{ns}'.`

### `NS_RESOURCE_QUOTA_EXCEEDED_BY_WORKLOADS` — block
- **scope**: cluster, per namespace
- **when**: `Σ(workload.replicas × workload.requests.cpu)` > `namespace.resource_quota.cpu` (memory/gpu/pods 동일)
- **msg**: `Namespace '{ns}': CPU {needed}c 요구, quota {limit}c.`

### `PROD_NS_NO_DEFAULT_DENY` — warn
- **scope**: cluster, per namespace
- **when**: `namespace.labels.tier == "prod"` (or name contains "prod"), `default_deny == false`
- **msg**: `Namespace '{ns}': production 추정인데 default-deny 미설정 — 보안 권장.`

---

## 9. 의존성 그래프 (depends_on)

### `DEPENDS_ON_TARGET_NOT_FOUND` — block
- **scope**: cluster, per depends_on
- **when**: `target_workload_id`가 같은 cluster.workloads에 없음
- **msg**: `{workload.name}: 의존 대상 '{target}' 워크로드 없음.`

### `DEPENDS_ON_CYCLE` — warn
- **scope**: cluster
- **when**: depends_on 그래프에 사이클 발견
- **msg**: `의존성 사이클 감지: {a → b → c → a}. 순환 호출은 동기 호출 시 데드락 위험.`

### `DEPENDS_ON_CROSS_CLUSTER` — warn
- **scope**: project
- **when**: `target_workload_id`가 다른 클러스터에 위치
- **msg**: `{workload.name} → {target}: 다른 클러스터의 워크로드 호출 — 외부 노출 + 네트워크 비용 검토.`

---

## 10. 정적 병목 분석 (runtime + traffic_profile + depends_on)

병목 룰은 사용자 데이터(runtime/traffic) 기반의 추정이며 모두 **warn** 또는 **info**. 차단 안 함. Little's Law 기반.

> **표기**: `R` = target_rps, `S` = avg_request_ms (ms), `T` = max_threads, `N` = replicas, `pool` = connection_pool_per_replica.

### `THREAD_POOL_SATURATION` — warn
- **scope**: workload
- **when**: `runtime.concurrency.model == "thread-pool"` 그리고
  `R × (S/1000) > T × N × 0.8`
- **msg**: `{workload.name}: 동시 처리 {needed} 스레드 필요, 가용 {capacity} (사용률 {pct}%). Tomcat 스레드 고갈 가능.`
- **rationale**: Little's Law. 80%부터 latency 급증.

### `EVENT_LOOP_CPU_BOUND` — warn
- **scope**: workload
- **when**: `runtime.concurrency.model == "event-loop"` 그리고 `S > 50ms` 그리고 다운스트림 의존성 적음(< 2)
- **msg**: `{workload.name}: 평균 응답 {S}ms은 이벤트 루프에 비교적 길어 CPU 바운드 의심. worker/replicas 증대 검토.`

### `WORKER_POOL_SATURATION` — warn
- **scope**: workload
- **when**: `runtime.concurrency.model == "process-pool"` 그리고 `R × (S/1000) > workers × N × 0.8`
- **msg**: `{workload.name}: gunicorn worker 부족. 권장 {recommended_workers}, 현재 {workers}.`

### `DB_CONNECTION_POOL_EXHAUSTED` — warn
- **scope**: workload (caller)
- **when**: 다운스트림 dep에 대해 `dep.fanout_pct/100 × dep.calls_per_request × R × (dep.avg_call_ms/1000) > pool × N × 0.8`
- **msg**: `{a.name} → {b.name}: 호출 풀 {pool×N}로 부족 ({needed} 필요). HikariCP/HTTP 풀 고갈 가능.`

### `DB_MAX_CONNECTIONS_EXCEEDED` — warn
- **scope**: workload (DB target)
- **when**: target이 `runtime.max_connections` 가지고, `Σ(callers의 pool × replicas) > max_connections × 0.8`
- **msg**: `{db.name}: 인입 연결 {incoming} > max_connections {limit} 80%. PgBouncer/PgPool 권장.`

### `TIMEOUT_CASCADE_RISK` — warn
- **scope**: workload (caller chain)
- **when**: caller의 `client_timeout_ms < Σ(downstream chain의 p99 sum)`
- **msg**: `{a.name}: client_timeout={t}ms < 다운스트림 체인 p99 합 {sum}ms. 타임아웃 캐스케이드 가능.`

### `JVM_HEAP_TOO_SMALL` — warn
- **scope**: workload
- **when**: `runtime.language in [Java, Kotlin]` 그리고 `runtime.concurrency.heap_gb < requests.memory_gb × 0.5`
- **msg**: `{workload.name}: heap {h}GB가 memory request {m}GB의 절반 미만 — GC 부담 또는 메모리 낭비.`

### `JVM_HEAP_GT_REQUEST` — block
- **scope**: workload
- **when**: `runtime.concurrency.heap_gb > requests.memory_gb`
- **msg**: `{workload.name}: heap {h}GB > memory request {m}GB. OOMKilled 거의 확정.`

### `SPOF_RISK` — warn
- **scope**: workload
- **when**: `replicas == 1` 그리고 `category in [API, WAS, DB, Cache, Queue]`
- **msg**: `{workload.name}: replicas=1, 단일 장애점. 카테고리 {cat}는 다중화 권장.`

### `STATEFUL_NO_PVC` — warn
- **scope**: workload
- **when**: `category == "DB"` 또는 `kind == "StatefulSet"` 그리고 `persistent_volumes` 비어있음
- **msg**: `{workload.name}: stateful 워크로드인데 PVC 없음 — 데이터 손실 위험.`

### `RPS_NEVER_REACHED_BY_DEPS` — info
- **scope**: workload (downstream)
- **when**: target_rps 설정 X, depends_on 그래프 합산이 더 작음
- **msg**: `{workload.name}: 업스트림 의존성 합산 RPS {sum} — target_rps 자동 추정.`

### `KEEPALIVE_HIGH_RPS_DISABLED` — warn
- **scope**: workload
- **when**: `traffic_profile.target_rps > 100` 그리고 `keepalive == false`
- **msg**: `{workload.name}: RPS {n}인데 keepalive=false — TCP 핸드셰이크 오버헤드 큼.`

---

## 11. Helm Export 무결성

### `HELM_DUPLICATE_CHART_NAME` — block
- **scope**: project
- **when**: 둘 이상의 워크로드가 같은 `helm.chart_name` 사용
- **msg**: `chart_name '{name}' 중복: {workloads}`

### `HELM_INGRESS_NO_HOST` — warn
- **scope**: workload
- **when**: `helm.expose.ingress == true`, `helm.expose.ingress_host` 미설정
- **msg**: `{workload.name}: ingress 활성화됐는데 host 미설정 — wildcard로 export됨.`

### `HELM_NO_IMAGE_REPO` — warn
- **scope**: workload
- **when**: `helm.image_repo` 미설정
- **msg**: `{workload.name}: image_repo 미설정 — chart values에 placeholder만 들어감.`

---

## 12. 아키텍처 계층 규칙

계층은 `workload.tier`, 비어 있으면 **이름(제품) → 네임스페이스 → category** 순으로 도출한다.
`kube-system` 등 클러스터 런타임 네임스페이스의 워크로드는 아키텍처가 고른 계층이 아니므로 전부 제외한다.

흐름 계층은 `edge → entry → presentation → application → data`, 보조 계층은 `caching` · `messaging`,
횡단 계층은 `security` · `observability` · `delivery`. 횡단 계층이 걸린 호출은 방향을 따지지 않는다
(인증 조회·메트릭 수집은 어느 계층에서든 일어난다).

### `LAYER_UNCLASSIFIED` — info
- **scope**: workload
- **when**: 어느 규칙으로도 계층이 정해지지 않음 (보통 `category: "Other"` + 비표준 네임스페이스)
- **msg**: `{cluster.name}/{workload.name}: 아키텍처 계층 미분류 — 개념도에서 서비스 계층으로 묶입니다`

### `LAYER_DATA_EXPOSED` — warn
- **scope**: workload
- **when**: `tier ∈ {data, caching}` 이고 `network.service.type ∈ {LoadBalancer, NodePort}`
- **msg**: `{cluster.name}/{workload.name}: {tier} 계층이 {type}로 외부 노출 — Entry 계층을 우회합니다`

### `LAYER_SKIP_CALL` — warn
- **scope**: workload, per depends_on
- **when**: 호출 방향이 허용표에 없고 대상 계층의 order가 더 큼 (예: Presentation → Data)
- **msg**: `{from.name}({from.tier}) → {to.name}({to.tier}): 중간 계층을 건너뛴 호출`

### `LAYER_UPWARD_CALL` — warn
- **scope**: workload, per depends_on
- **when**: 대상 계층의 order가 더 작음 (예: Data → Presentation)
- **msg**: `{from.name}({from.tier}) → {to.name}({to.tier}): 하위 계층이 상위 계층을 호출`

### `LAYER_NO_ENTRY` — warn
- **scope**: cluster
- **when**: `ingress_rules`가 있거나 LoadBalancer/NodePort 서비스가 있는데, Entry 계층 워크로드도 `addons.ingress`도 이 클러스터를 가리키는 `site.l4_balancers`도 없음
- **msg**: `{cluster.name}: 외부 노출이 있는데 Entry 계층(Ingress·게이트웨이)이 없음`

---

## 13. VM · 베어메탈 그룹

`cluster.platform === "vm"` 인 그룹에서만 실행된다. 반대로 이 그룹에서는 k8s 룰
(`CLUSTER_NO_CONTROL_PLANE` · `NO_CNI_SELECTED` · `GPU_NODE_NO_GPU_OPERATOR` ·
`WORKLOAD_NAMESPACE_NOT_DECLARED` · `NETPOL_*` · `NS_*` · `WORKLOAD_STATEFUL_NO_ANTI_AFFINITY`)이
실행되지 않는다 — 고칠 수 없는 경고만 쌓이기 때문이다.

### `VM_NODE_NO_OS` — info
- **scope**: node
- **when**: vm 그룹에 속한 노드에 `node.os.distro` 미설정
- **msg**: `{cluster.name}/{node.name}: OS 미기재 — 발주서·설치 목록에 필요합니다`

### `VM_SERVICE_NO_SOFTWARE` — info
- **scope**: workload
- **when**: vm 그룹의 서비스에 `workload.software[]`가 비었거나 이름이 없음
- **msg**: `{cluster.name}/{workload.name}: 설치 소프트웨어 미기재 — 계층 판정이 category에만 의존합니다`

### `VM_HA_NO_VIP` — warn
- **scope**: workload
- **when**: `ha.mode ∈ {active-standby, active-active}` 인데 `ha.vip` 미설정
- **msg**: `{cluster.name}/{workload.name}: {mode}인데 VIP 미기재 — 절체 대상 주소가 없으면 구성이 성립하지 않습니다`

---

## 14. 물리 / 가상 (VM ↔ 호스트)

`node.virtualization.kind === "vm"` 인 노드에 적용. VM은 랙 U·전력 산정에서 빠지므로
물리 근거(호스트)가 없으면 용량 산정이 공중에 뜬다.

### `VM_NO_HOST` — info
- **scope**: node
- **when**: `kind=vm` 인데 `host_node_id` 미설정
- **msg**: `{node.name}: VM인데 호스트 미지정 — 랙 U·전력 산정에서 빠지며 물리 근거가 문서에 남지 않습니다`

### `VM_HOST_NOT_FOUND` — block
- **scope**: node
- **when**: `host_node_id`가 가리키는 노드가 없음
- **msg**: `{node.name}: 호스트 노드({id})를 찾을 수 없음`

### `VM_HOST_IS_VM` — warn
- **scope**: node
- **when**: 호스트로 지정한 노드도 `kind=vm`
- **msg**: `{node.name}: 호스트로 지정한 {host}도 VM입니다 — 물리 서버를 지정하세요`

### `VM_HOST_DIFFERENT_SITE` — warn
- **scope**: node
- **when**: 호스트가 다른 사이트에 있음
- **msg**: `{node.name}: 호스트 {host}가 다른 사이트에 있습니다`

### `VM_HOST_MEMORY_OVERCOMMIT` — block
- **scope**: node (호스트)
- **when**: 호스트에 올라간 VM들의 `memory_gb` 합 > 호스트 `memory_gb`
- **msg**: `{host}: VM {N}대의 메모리 합 {sum}GB > 호스트 {cap}GB (하이퍼바이저 오버헤드 제외)`
- 메모리 오버커밋은 하이퍼바이저가 스왑하거나 VM이 죽는다 — vCPU와 달리 정상 구성이 아니다.

### `VM_HOST_VCPU_OVERCOMMIT_HIGH` — warn
- **scope**: node (호스트)
- **when**: VM vCPU 합 > 호스트 vCPU × 4
- **msg**: `{host}: vCPU 오버커밋 {ratio}:1 ({sum}/{cap}) — 4:1 초과`
