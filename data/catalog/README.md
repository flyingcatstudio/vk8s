# Catalog

브랜드별 서버/인스턴스 프리셋. Add Node 마법사의 데이터 소스.

## 구조

```
catalog/
├── cloud/
│   ├── aws.json       ← EC2 (m6i, c7i, r7i, i4i, g6, g6e, p4d, p5, p5e, p6-b200)
│   └── ncp.json       ← NCP Standard / CPU / Memory / GPU
└── onprem/
    ├── dell.json      ← PowerEdge R660, R760, R760xa, XE8640, XE9680, XE9685L (Blackwell)
    ├── hpe.json       ← ProLiant DL360/DL380/DL380a Gen11, Cray XD670, Cray XD685 (Blackwell)
    ├── lenovo.json    ← ThinkSystem SR630/SR650/SR675/SR685a V3, SC777 V4 (Blackwell, Neptune)
    └── supermicro.json← AS-4125GS-TNRT, SYS-821GE-TNHR, SYS-821GV-TNR, AS-A21GE-NBRT (Blackwell)
```

## 파일 포맷

```jsonc
{
  "$schema_version": "0.2",
  "brand": "Dell",
  "source": "onprem",
  "notes": "...",
  "entries": [
    { /* CatalogEntry */ },
    ...
  ]
}
```

각 `entries[i]`는 `schema/catalog-entry.schema.json` 준수.

## 스코프 (v0.1)

- 대표 모델 위주, 모든 SKU 망라 X
- 가격 정보 제외
- onprem: 16~17세대(2023~2025), GPU는 B200/B100/H100/H200/A100/L40S/L40/L4 위주
- cloud: 대표 인스턴스 타입만 (전체 패밀리 X)

## 데이터 갱신 정책

- 사양 변경(GPU 옵션 추가, 신규 세대 출시)은 vendor 자료 기준
- 커밋 메시지에 출처/연도 명시
- 사용자 추가 모델은 `Node.custom: true`로 처리, 카탈로그에 영향 없음

## 추가하지 않은 브랜드 (v0.2+)

- Azure, GCP (Cloud)
- IBM Power (PPC64LE는 별도 추적)
- Cisco UCS, Inspur, NVIDIA DGX (직접 OEM 라인)
- Korean OEM: 삼성SDS, KT Cloud, 카카오엔터프라이즈
