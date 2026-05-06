# vk8s Validator Rules (v0.1)

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
