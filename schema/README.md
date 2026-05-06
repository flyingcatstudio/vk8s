# vk8s Schema (v0.1)

vk8s 프로젝트의 데이터 모델 정의. JSON Schema 2020-12 기준.

## 파일 구조

```
schema/
├── README.md                      ← 이 문서
├── project.schema.json            ← 루트. mode + sites[] + clusters[]
├── site.schema.json               ← 배포 환경 (onprem IDC / cloud region)
├── node.schema.json               ← 노드 본체 (id, brand, model, role)
├── node-spec.schema.json          ← 하드웨어 스펙 (vCPU/RAM/disk/NIC/form_factor/power/GPU)
├── cluster.schema.json            ← k8s 클러스터 (node_ids[], addons, workloads[])
├── workload.schema.json           ← 서비스 (resources/scheduling/HPA)
├── catalog-entry.schema.json      ← 카탈로그 항목 (브랜드 프리셋)
└── validator-rules.md             ← 검증 룰 명세 (block/warn)
```

```
data/
├── gpu-models.json                ← GPU 레퍼런스 (TDP, VRAM, NVLink 가능 여부)
└── catalog/                       ← 추후 채울 예정 (Phase B)
    ├── cloud/
    │   ├── aws.json
    │   └── ncp.json
    └── onprem/
        ├── dell.json
        ├── hpe.json
        ├── lenovo.json
        └── supermicro.json
```

## 엔티티 관계

```
Project (mode: onprem|cloud|hybrid)
├── Sites[]                ← 배포 환경
│   └── (capacity)         ← onprem만: rack_units, power_kw, cooling_kw
├── Nodes[]                ← 모든 노드. site_id로 Site와 연결
│   └── spec               ← node-spec 공유 (catalog와 동일 shape)
└── Clusters[]
    ├── node_ids[]         ← 어떤 Node를 묶을지 (한 노드 = 한 클러스터)
    └── Workloads[]        ← 이 클러스터에 띄울 서비스
```

- **Node ↔ Site**: `Node.site_id` / `Site.node_ids` 양방향 (validator가 일치 검증)
- **Node ↔ Cluster**: `Cluster.node_ids` 단방향 (한 노드는 한 클러스터에만)
- **Catalog ↔ Node**: 카탈로그 항목을 선택하면 `spec`이 Node로 복사됨. 이후 사용자가 수정 가능 (`custom: true`).

## 핵심 설계 결정

### 1. spec은 catalog와 node가 공유
`node-spec.schema.json`을 양쪽에서 `$ref` — 카탈로그 항목과 실제 노드의 shape이 같음. 카탈로그는 단순히 "미리 채워진 spec".

### 2. GPU는 chassis ↔ installation 분리
- `gpu_chassis`: 새시가 받아줄 수 있는 한계 (카탈로그가 정의)
- `gpu`: 사용자가 실제 장착한 구성 (사용자가 수정)
- Validator가 둘을 대조해서 호환성 검증 (1U 새시에 H200 8장 같은 비현실적 구성 차단)

### 3. Capacity는 onprem 전용
`Site.capacity` 필드는 cloud Site에서는 무시. 클라우드는 사실상 무한 가정.

### 4. Hybrid 모드
한 프로젝트에 onprem + cloud Site 공존 가능. 다만 한 클러스터가 양쪽에 걸치면 warn (`CLUSTER_SPANS_ENVIRONMENTS`) — 보통 클러스터는 환경별로 분리.

### 5. 가격은 미포함
가격은 자주 변하고 부정확하면 신뢰를 잃어 v0.1에서 제외. capacity와 spec 데이터로만 의사결정.

## 사용 예시 (project 인스턴스 스니펫)

```json
{
  "version": "0.1",
  "name": "ai-platform-2026",
  "mode": "hybrid",
  "sites": [
    {
      "id": "idc-seoul-1",
      "name": "IDC Seoul 1",
      "kind": "onprem",
      "capacity": { "rack_units": 84, "power_kw": 32, "cooling_kw": 36 },
      "node_ids": ["node-gpu-1"]
    },
    {
      "id": "aws-apne2",
      "name": "AWS Seoul",
      "kind": "cloud",
      "cloud_provider": "AWS",
      "region": "ap-northeast-2",
      "node_ids": ["node-api-1", "node-api-2"]
    }
  ],
  "nodes": [
    {
      "id": "node-gpu-1",
      "source": "onprem",
      "brand": "Dell",
      "model": "PowerEdge-XE9680",
      "site_id": "idc-seoul-1",
      "cluster_role": "worker",
      "spec": {
        "vcpu": 112,
        "memory_gb": 2048,
        "form_factor": "6U",
        "power": { "psu_max_w": 10800, "typical_w_base": 2500, "psu_redundancy": "N+N" },
        "gpu_chassis": {
          "max_count": 8,
          "supported_form_factors": ["SXM"],
          "supported_models": ["H100-SXM", "H200-SXM"],
          "nvlink_topology": "nvswitch-mesh"
        },
        "gpu": [{ "model": "H200-SXM", "form_factor": "SXM", "count": 8, "nvlink_enabled": true }]
      }
    }
  ],
  "clusters": [
    {
      "id": "ai-prod",
      "name": "AI Production",
      "environment_tag": "onprem-gpu",
      "k8s_version": "1.30.4",
      "node_ids": ["node-gpu-1"],
      "addons": { "cni": "Calico", "gpu_operator": true },
      "workloads": [
        {
          "id": "llm-inference",
          "name": "Llama-3-70B Inference",
          "category": "ML-Inference",
          "replicas": 2,
          "resources": {
            "requests": {
              "cpu": 16,
              "memory_gb": 256,
              "gpu": { "count": 4, "model": "H200-SXM" }
            }
          }
        }
      ]
    }
  ]
}
```

## 다음 단계 (Phase B)

1. `data/catalog/onprem/dell.json` 등 브랜드별 프리셋 채우기
2. JS 모듈로 validator 구현 (`src/validator.js`)
3. UI 프로토타입 (single-page index.html, loom과 동일 패턴)
