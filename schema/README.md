# vk8s Schema (v0.5)

vk8s 프로젝트의 데이터 모델 정의. JSON Schema 2020-12 기준.

## 파일 구조

```
schema/
├── README.md                      ← 이 문서
├── project.schema.json            ← 루트. version + mode + sites[] + nodes[] + clusters[]
├── site.schema.json               ← 배포 환경 (onprem IDC / cloud region)
├── node.schema.json               ← 노드 본체 (id, brand, model, role)
├── node-spec.schema.json          ← 하드웨어 스펙 (vCPU/RAM/disk/NIC/form_factor/power/GPU)
├── cluster.schema.json            ← 클러스터 (node_ids/addons/namespaces/network_policies/workloads)
├── workload.schema.json           ← 서비스 (resources/runtime/traffic/depends_on/helm)
├── catalog-entry.schema.json      ← 브랜드 프리셋 (서버/인스턴스 카탈로그)
└── validator-rules.md             ← 검증 룰 명세 (block/warn/info)
```

```
data/
├── gpu-models.json                ← GPU 레퍼런스 (TDP, VRAM, NVLink)
├── runtime-profiles.json          ← 런타임 프리셋 (Spring 17, Node, FastAPI, Go, Postgres ...)
└── catalog/                       ← 추후 채울 예정
    ├── cloud/      (aws.json, ncp.json)
    └── onprem/     (dell.json, hpe.json, lenovo.json, supermicro.json)
```

## 엔티티 관계

```
Project (mode: onprem|cloud|hybrid)
├── Sites[]                ← 배포 환경. onprem만 capacity (U/kW/cooling)
├── Nodes[]                ← site_id로 Site와 연결
│   └── spec               ← catalog-entry와 shape 공유 ($ref)
└── Clusters[]
    ├── node_ids[]
    ├── namespaces[]       ← NS 정의 + default_deny + resource_quota
    ├── network_policies[] ← NS간 통신 제어
    └── Workloads[]
        ├── namespace      ← cluster.namespaces[].name 중 하나
        ├── runtime        ← runtime-profiles.json 시드
        ├── traffic_profile← 병목 분석 입력
        ├── depends_on[]   ← 다운스트림 호출 그래프
        └── helm           ← chart export 메타
```

## 핵심 설계 결정

### 1. node-spec은 catalog와 node가 공유
양쪽이 `$ref`로 같은 shape — 카탈로그 항목 = "미리 채워진 spec".

### 2. GPU = chassis(허용) + gpu(실장착)
- `gpu_chassis`: 새시 호환성 (카탈로그 정의)
- `gpu`: 사용자 장착 구성
- Validator가 1U 새시에 H200 같은 비현실 구성 차단

### 3. capacity는 onprem 전용
`Site.capacity` (rack_units, power_kw, cooling_kw)는 cloud Site에서는 무시.

### 4. Hybrid 모드
한 프로젝트에 onprem + cloud Site 공존 가능. 한 클러스터가 양쪽 걸치면 warn.

### 5. 가격은 미포함
부정확한 가격은 신뢰를 잃어 v0.1부터 제외.

### 6. 병목은 정적 룰만 (v0.2)
Little's Law 기반 산수 룰. 디스크리트 이벤트 시뮬레이션은 v0.3+ 검토.

### 7. NetworkPolicy는 인라인
Cluster 내 `namespaces[]`/`network_policies[]` 인라인. NetworkPolicy 별도 파일 안 만듦 (호스팅 스코프가 cluster이므로).

### 8. Helm은 워크로드 단위
Workload 1개 = chart 1개. Export 시 umbrella chart로 묶을지는 export 옵션.

## v0.4 → v0.5 변경

- **Site.l4_balancers[]**: L4 로드밸런서 (F5/Citrix/MetalLB 등). vip, throughput, target_cluster_ids.
- **Site.storages[]**: 외부 스토리지 (NAS/SAN/Ceph/Object). protocol, endpoint, csi_driver, capacity_tb.
- 모두 **Site 레벨**에 위치 (IDC 인프라 물리적 자원). 클러스터는 target_cluster_ids로 연결.

## v0.3 → v0.4 변경

- **Cluster.network** 신규 (CIDR + DNS): `pod_cidr`, `service_cidr`, `node_subnet`, `dns_domain`, `lb_pool`
- **Cluster.ingress_rules[]** 신규: 호스트/경로 → workload service 매핑 (nginx-ingress 등이 사용)
- **Node.network** 신규: `ip` (cluster.network.node_subnet에서 자동 할당) + 옵션으로 다중 NIC `interfaces[]`
- **Workload.network.service** 신규: type (None/ClusterIP/NodePort/LoadBalancer), `cluster_ip`, `lb_ip`, `ports[]` (port/target_port/protocol/node_port)
- 기존 `workload.helm.expose`와 별도로 유지 (helm은 export 메타, network는 모델)

## v0.2 → v0.3 변경

- `Node.cluster_role` (단일 enum) → `Node.cluster_roles` (배열). dev 단일노드 클러스터의 control-plane+worker 겸용 지원. 빈 배열 = unassigned.

## v0.1 → v0.2 변경

- Cluster: `namespaces[]`, `network_policies[]` 추가
- Workload: `namespace`, `labels`, `runtime`, `traffic_profile`, `depends_on[]`, `helm` 추가
- 신규 데이터: `data/runtime-profiles.json`
- Validator: NS/NP 룰 (8장), depends_on 룰 (9장), 병목 룰 (10장), Helm 룰 (11장) 추가

## 사용 예시

```json
{
  "version": "0.3",
  "name": "ai-platform-2026",
  "mode": "hybrid",
  "sites": [
    {
      "id": "idc-seoul-1", "name": "IDC Seoul 1", "kind": "onprem",
      "capacity": { "rack_units": 84, "power_kw": 32, "cooling_kw": 36 },
      "node_ids": ["node-gpu-1"]
    }
  ],
  "nodes": [
    {
      "id": "node-gpu-1", "source": "onprem", "brand": "Dell",
      "model": "PowerEdge-XE9680", "site_id": "idc-seoul-1",
      "cluster_roles": ["worker"],
      "spec": {
        "vcpu": 112, "memory_gb": 2048, "form_factor": "6U",
        "power": { "psu_max_w": 10800, "typical_w_base": 2500, "psu_redundancy": "N+N" },
        "gpu_chassis": {
          "max_count": 8, "supported_form_factors": ["SXM"],
          "supported_models": ["H100-SXM", "H200-SXM"],
          "nvlink_topology": "nvswitch-mesh"
        },
        "gpu": [{ "model": "H200-SXM", "form_factor": "SXM", "count": 8, "nvlink_enabled": true }]
      }
    }
  ],
  "clusters": [
    {
      "id": "ai-prod", "name": "AI Production",
      "k8s_version": "1.30.4", "node_ids": ["node-gpu-1"],
      "addons": { "cni": "Calico", "gpu_operator": true },
      "namespaces": [
        { "name": "frontend", "labels": { "tier": "edge" } },
        { "name": "backend", "labels": { "tier": "core" }, "default_deny": true },
        { "name": "data", "labels": { "tier": "stateful" }, "default_deny": true }
      ],
      "network_policies": [
        {
          "id": "data-from-backend", "name": "data-from-backend",
          "namespace": "data",
          "pod_selector": { "match_labels": { "app": "postgres" } },
          "ingress": [
            { "from_namespaces": ["backend"], "ports": [{ "port": 5432, "protocol": "TCP" }] }
          ]
        }
      ],
      "workloads": [
        {
          "id": "user-api", "name": "User API",
          "category": "API", "namespace": "backend",
          "labels": { "app": "user-api", "tier": "core" },
          "replicas": 3,
          "resources": { "requests": { "cpu": 2, "memory_gb": 4 } },
          "runtime": {
            "language": "Java", "framework": "Spring-Boot", "version": "17",
            "concurrency": { "model": "thread-pool", "max_threads": 200, "heap_gb": 3 },
            "connection_pool_per_replica": 10
          },
          "traffic_profile": {
            "target_rps": 500, "avg_request_ms": 80, "p99_request_ms": 300,
            "client_timeout_ms": 3000
          },
          "depends_on": [
            {
              "target_workload_id": "postgres",
              "protocol": "JDBC", "port": 5432,
              "avg_call_ms": 15, "fanout_pct": 100, "calls_per_request": 2
            }
          ],
          "helm": {
            "chart_name": "user-api", "chart_version": "0.1.0",
            "image_repo": "registry.example.com/user-api",
            "expose": {
              "service": "ClusterIP",
              "ports": [{ "name": "http", "port": 8080, "target_port": 8080 }]
            }
          }
        },
        {
          "id": "postgres", "name": "Postgres",
          "category": "DB", "namespace": "data",
          "labels": { "app": "postgres" },
          "kind": "StatefulSet", "replicas": 1,
          "resources": { "requests": { "cpu": 4, "memory_gb": 16 } },
          "runtime": {
            "language": "Other", "framework": "PostgreSQL", "version": "16",
            "max_connections": 100
          },
          "persistent_volumes": [{ "size_gb": 500, "access_mode": "RWO" }]
        }
      ]
    }
  ]
}
```

## 다음 단계

1. UI 설계 (`vk8s/docs/ui-spec.md`) — 화면/인터랙션/정보계층
2. 카탈로그 데이터 입력 (`data/catalog/`)
3. JS 모듈로 validator + 병목 분석 + Helm export 구현
