# SIMS 데이터 모델 (최종 확정안)

> 2026-10-20 확정. **models.py는 이 문서를 기준으로 작성한다.** 구조를 바꿀 때는 이 문서를 먼저 고치고 개발자 확인을 받는다.
> 사내 코드 체계가 들어 있으므로 **사내 Gitea 전용**이다. 공개 저장소에 올리지 않는다.

---

## 1. 설계 전제

- 사양 데이터의 수정·신규 입력은 **드물다.** 속도보다 **정확성과 추적 가능성**이 중요하다.
- 단순 조회 시스템이다. 가장 무거운 화면(모델 10개 비교)도 10 × 약 56항목 = 약 560줄이라 별도 최적화는 필요 없다.
- 원칙: **같은 값을 두 곳에 저장하지 않는다.** 다른 칸에서 계산할 수 있는 값은 저장하지 않고 property로 계산한다.

## 2. 전체 구조

```text
            ProductModel  (모델 1개 = 1줄, MI 단위)
                 │ 1
                 │
                 │ N
SpecItem ──1:N── SpecValue  (모델 × 항목 = 값)
(항목 사전)

DownloadLog  (엑셀 다운로드 기록, 독립)
```

- `specs` 앱 하나에 4개 테이블. 선택지는 `specs/choices.py` 한 곳에서 정의해 모든 테이블이 같이 쓴다.
- 저장은 세로(항목당 1줄), 화면은 가로(pivot).
- 변경 이력(django-simple-history): **ProductModel, SpecItem, SpecValue**. DownloadLog는 수정 불가라 이력 불필요.

---

## 3. 선택지 (`specs/choices.py`)

값(DB 저장)은 **회사 코드 그대로**, 표시(화면)는 한글/영문. MI 코드의 자리별 글자와 같은 값이라 FSC를 받으면 자동 대조할 수 있다.

### 기종 `CATEGORY` (인터페이스 항목 PA제품군·PA 상세 제품군)

| 값 | 표시 |
|-|-|
| `T` | 트랙터 |
| `C` | 콤바인 |
| `R` | 이앙기 |
| `E` | 엔진 |
| `W10` | 로더 |
| `W20` | 모어 |
| `W30` | 백호 |

### 시장 `MARKET` (MI 4번째 자리)

| 값 | 표시 |
|-|-|
| `K` | 한국 |
| `N` | 북미 |
| `E` | 유럽 |
| `O` | 오세아니아 |
| `A` | 아시아 |
| `S` | 남미 |

1차는 K·N·E만 사용. 기대스펙통합판 "유럽 및 기타" 시트의 모델은 1차엔 `E`로 넣는다.

### 판매처 `BRAND`

| 값 | 표시 |
|-|-|
| `TYM` | TYM |
| `RK` | Rural King |
| `BADBOY` | Bad Boy |
| `BRANSON` | Branson |
| `YANMAR` | YANMAR |
| `MCCORMICK` | McCormick |
| `ISEKI` | ISEKI |
| `ETC` | 기타 |

기본값 `TYM`. 판매처는 인터페이스 항목·옵션 마스터의 칼라(CL) 코드 체계를 참고.

### 엔진 제조사 `ENGINE_MAKER` (MI 5번째 자리)

| 값 | 표시 | 값 | 표시 |
|-|-|-|-|
| `A` | Deutz | `I` | Cummins |
| `B` | TYM | `J` | Mitsubishi |
| `C` | YANMAR | `K` | Mahindra |
| `D` | Doosan | `L` | IVECO |
| `E` | John Deere | `M` | Kohler |
| `F` | Perkins | `N` | Honda |
| `H` | Daedong | `X` | 기타 |

### 변속 방식 `TRANSMISSION` (MI 9번째 자리)

| 값 | 표시 | 값 | 표시 |
|-|-|-|-|
| `A` | Synchro | `F` | MCT |
| `B` | Synchro-Shuttle | `G` | 6단 Power-Shift |
| `C` | Power-Shuttle | `H` | 8단 Power-Shift |
| `D` | HST | `I` | 기계식 (콤바인) |
| `E` | CVT | `J` | Constant mesh |
| | | `X` | 기타 |

### 보호구조물 `ROPS_TYPE` (MI 10번째 자리)

| 값 | 표시 |
|-|-|
| `A` | 2주식 ROPS |
| `B` | 4주식 Cabin |
| `C` | 5주식 Cabin |
| `D` | 6주식 Cabin |
| `E` | Non-Cabin (콤바인·이앙기) |
| `F` | Cabin (콤바인) |
| `G` | 2주식 Front ROPS |
| `X` | 기타 |

### 상태 `STATUS`

| 값 | 표시 |
|-|-|
| `development` | 개발 |
| `production` | 양산 |
| `discontinued` | 단종 |

### 항목 구분 `KIND` / 값 타입 `VALUE_TYPE`

| `KIND` 값 | 표시 | `VALUE_TYPE` 값 | 표시 |
|-|-|-|-|
| `spec` | 사양 | `number` | 숫자 |
| `option` | 옵션 | `text` | 텍스트 |

### 단위 `UNIT` (저장 단위, metric)

값은 영문·숫자·밑줄만 쓴다 (특수문자 `·`, `/`, `°`를 값에 쓰지 않음).

| 분류 | 값 → 표시 | imperial 변환 표시 |
|-|-|-|
| 출력 | `kW` → kW | ps (× 1.35962), hp (× 1.34102) |
| 길이 | `mm` → mm, `m` → m | in (mm ÷ 25.4), ft (m × 3.28084) |
| 무게 | `kg` → kg | lb (× 2.20462) |
| 힘 | `N` → N, `kN` → kN, `kgf` → kgf | lbf (N ÷ 4.44822, kN × 224.809, kgf × 2.20462) |
| 토크 | `N_m` → N·m, `kgf_m` → kgf·m | lbf·ft (N·m × 0.737562, kgf·m × 7.23301) |
| 부피 | `L` → L, `cc` → cc | gal (L ÷ 3.78541), cu.in (cc ÷ 16.3871) |
| 유량 | `L_min` → L/min | gpm (÷ 3.78541) |
| 속도 | `km_h` → km/h | mph (÷ 1.60934) |
| 압력 | `bar` → bar, `MPa` → MPa, `kgf_cm2` → kgf/cm² | psi (bar × 14.5038, MPa × 145.038, kgf/cm² × 14.2233) |
| 온도 | `degC` → °C | °F (× 9/5 + 32) |
| 회전수 | `rpm` → rpm | 변환 없음 |
| 각도 | `deg` → ° | 변환 없음 |
| 시간 | `s` → s, `h` → h | 변환 없음 |
| 연료 소비율 | `g_kWh` → g/kWh | 변환 없음 |
| 전기 | `V`, `A`, `Ah`, `W` | 변환 없음 |
| 비율 | `pct` → % | 변환 없음 |
| 없음 | `''` (빈 값) | 개수·텍스트 항목 |

새 단위는 **선택지와 변환 규칙을 함께** 추가한다.

### 출처 `SOURCE`

| 값 | 표시 |
|-|-|
| `spec_sheet` | 기대스펙통합판 |
| `product_spec` | 제품사양서 |
| `manual` | 수기 입력 |
| `other` | 기타 |

파일 버전 등 세부 내용은 `note`에 적는다.

### 다운로드 종류 `DOWNLOAD_TYPE`

| 값 | 표시 |
|-|-|
| `search` | 조회 결과 |
| `compare` | 비교 |
| `detail` | 상세 |

---

## 4. ProductModel (모델 1개 = 1줄)

**한 줄 = MI 단위 = 형식명에서 타이어 자리를 뺀 것** (예: `T1025HREU` ↔ `TAAEC025DA`).
국가·판매처·엔진·변속·보호구조물이 다르면 별도 줄, 타이어 등 옵션만 다르면 같은 줄.

| 필드 | 한글명 | Django 타입 | 필수 | 예 (T25 한국) |
|-|-|-|-|-|
| `category` | 기종 | CharField(10, CATEGORY) | ✅ | `T` |
| `brand` | 판매처 | CharField(20, BRAND, 기본 `TYM`) | ✅ | `TYM` |
| `market` | 시장 | CharField(1, MARKET) | ✅ | `K` |
| `name` | 판매명 | CharField(50) | ✅ | T25 |
| `unified_name` | 통합기종명 | CharField(20, blank) | | T1025 |
| `model_type_name` | 형식명 (타이어 제외) | CharField(30, blank) | | (D365 수신 후) |
| `aliases` | 별칭 | TextField(blank) | | 옛 기종명, 판매처 모델명 (쉼표 구분) |
| `rated_ps` | 마력 (PS) | DecimalField(5,1, null, blank) | | 24.6 |
| `engine_maker` | 엔진 제조사 | CharField(1, ENGINE_MAKER, blank) | | `C` |
| `engine_model` | 엔진 모델 | CharField(50, blank) | | 3TNV80F |
| `transmission` | 변속 방식 | CharField(1, TRANSMISSION, blank) | | `B` |
| `rops_type` | 보호구조물 | CharField(1, ROPS_TYPE, blank) | | `A` |
| `status` | 상태 | CharField(20, STATUS, 기본 `production`) | ✅ | `production` |
| `mi_code` | MI 코드 | CharField(10, blank) | | (FSC 앞 10자리, 수신 후) |
| `d365_item_codes` | D365 기종코드 | TextField(blank) | | (쉼표 구분, 타이어 변형별 여러 개) |
| `note` | 비고 | TextField(blank) | | "기대스펙 헤더 K65 / 모델행 T65 불일치" |
| `created_at` / `updated_at` | 생성·수정 일시 | DateTimeField(auto) | 자동 | |

### 규칙

- **중복 금지**: `UniqueConstraint(category, market, brand, name, transmission, rops_type)`
- **"모름"은 빈 문자열(`''`)로 저장, NULL 금지** (선택지·텍스트 필드 전부). PostgreSQL은 NULL끼리 다른 값으로 봐서 중복 기준이 무력화된다. `rated_ps`(숫자)만 NULL 허용.
- 기대스펙통합판은 한 열에 ROPS·캐빈 값이 섞여 있어, 이관 초기엔 `rops_type`이 빈 값일 수 있다 (MI 자료 수신 후 분리).
- **모델은 삭제하지 않고 `status=단종`으로 처리한다.** 삭제하면 SpecValue가 함께 삭제되므로(CASCADE) Admin 삭제 권한은 시스템 관리자(superuser)만.

### 계산 속성 (저장하지 않음)

| property | 계산 | 예 |
|-|-|-|
| `product_code` | `mi_code[:3]` | TAA |
| `series` | `unified_name`의 두 번째 글자 (숫자일 때) | T**1**025 → 1 |
| `hp_class` | `rated_ps`를 10 단위로 내림 | 24.6 → 20 (20마력대) |

### 검증 (`clean()`)

- `mi_code`가 있으면 10자리여야 하고, 값이 채워진 필드와 자리별로 일치해야 한다:
  4번째 = `market`, 5번째 = `engine_maker`, 9번째 = `transmission`, 10번째 = `rops_type` (빈 필드는 비교하지 않음)
- 불일치 시 어느 자리가 왜 다른지 한국어로 안내

### `__str__`

`T25 (한국 · TYM · Synchro-Shuttle · 2주식 ROPS)` 형태. 변속·보호구조물이 비어 있으면 생략.

---

## 5. SpecItem (항목 사전)

| 필드 | 한글명 | Django 타입 | 필수 | 예 |
|-|-|-|-|-|
| `category` | 기종 | CharField(10, CATEGORY) | ✅ | `T` |
| `kind` | 구분 | CharField(10, KIND, 기본 `spec`) | ✅ | `spec` |
| `section` | 섹션 | CharField(50) | ✅ | 엔진 |
| `code` | 항목 코드 | CharField(50) | ✅ | `gross_power` |
| `name_ko` | 국문명 | CharField(100) | ✅ | 엔진 출력 |
| `name_en` | 영문명 | CharField(100) | ✅ | Gross engine power |
| `unit` | 저장 단위 | CharField(10, UNIT, blank) | | `kW` |
| `value_type` | 값 타입 | CharField(10, VALUE_TYPE) | ✅ | `number` |
| `display_order` | 표시 순서 | PositiveIntegerField | ✅ | 1 |
| `source_label` | 원본 표기 | TextField(blank) | | `S; ENGINE; Gross engine power \| S; 엔진; 엔진 출력` |
| `source_code` | 원천 코드 | CharField(20, blank) | | `CN11` (옵션일 때) |
| `is_active` | 사용 여부 | BooleanField(기본 True) | ✅ | True |

### 규칙

- **중복 금지**: `UniqueConstraint(category, code)`
- `code`: 영문 소문자로 시작, 소문자·숫자·밑줄만 (정규식 검증 `^[a-z][a-z0-9_]*$`). **한 번 정하면 바꾸지 않는다.** 화면 이름은 `name_ko`만 고친다.
- 섹션 순서는 별도 칸 없이 `display_order`로 정한다 (엔진 1~10, 동력전달 11~20 …).
- `source_label`은 원본 엑셀의 행 이름. 파일마다 표기가 다르면 `|`로 구분해 여러 개 저장 (이관 스크립트는 하나만 맞아도 인식).
- 한 칸에 두 값(L/R, F/R, 내경×행정)은 항목을 나누고, 나뉜 항목은 같은 `source_label`을 가진다.
- 안 쓰는 항목은 삭제하지 않고 `is_active=False` (값이 있는 항목은 PROTECT로 삭제 불가).
- 변경 이력 적용: 단위·타입을 바꾸면 모든 모델의 값 해석이 바뀌므로 추적 필요.

---

## 6. SpecValue (모델 × 항목 = 값)

| 필드 | 한글명 | Django 타입 | 필수 |
|-|-|-|-|
| `product_model` | 모델 | ForeignKey(ProductModel, CASCADE, related_name="spec_values") | ✅ |
| `spec_item` | 항목 | ForeignKey(SpecItem, **PROTECT**, related_name="values") | ✅ |
| `value_number` | 숫자값 | DecimalField(12,3, null, blank) | |
| `value_text` | 텍스트값 | CharField(255, blank) | |
| `source` | 출처 | CharField(20, SOURCE, 기본 `spec_sheet`) | ✅ |
| `note` | 비고 | TextField(blank) | |

### 검증 (`clean()`, 저장 시 자동)

| 규칙 | 막는 실수 |
|-|-|
| `UniqueConstraint(product_model, spec_item)` | 같은 값 두 번 입력 |
| 숫자 항목 → `value_number`만, 텍스트 항목 → `value_text`만 | 숫자 칸에 "18.4kW" |
| 둘 다 비어 있으면 저장 불가 | 빈 줄 (값이 없으면 줄을 만들지 않음) |
| `product_model.category == spec_item.category` | 트랙터에 로더 항목 연결 |
| `spec_item.is_active == False`인 항목에 **새** 값 추가 불가 (기존 값 수정은 허용) | 숨긴 항목에 실수로 입력 |

- `FloatField`가 아닌 `DecimalField`: 실수 오차(예: `2215.4699999998`) 없이 저장
- 옵션 값의 기본/선택 구분은 텍스트로: `할로겐 (기본) / LED (선택)`

### 예시

| 모델 | 항목 | 숫자값 | 텍스트값 | 출처 | 비고 |
|-|-|-|-|-|-|
| T25 한국 | 엔진 출력 (kW) | 18.4 | | 기대스펙통합판 | |
| T25 한국 | 변속 단수 | | F2 × R2 | 기대스펙통합판 | |
| T70 한국 | 전고 (mm) | 2540 | | 기대스펙통합판 | EN 100in / KO 106.88in 불일치 → mm 기준 |

---

## 7. DownloadLog (엑셀 다운로드 기록)

| 필드 | 한글명 | Django 타입 |
|-|-|-|
| `user` | 사용자 | ForeignKey(User, **SET_NULL**, null) — 계정이 삭제돼도 기록 유지 |
| `user_name` | 이름 (당시) | CharField(100) |
| `user_email` | 이메일 (당시) | CharField(254) |
| `downloaded_at` | 다운로드 일시 | DateTimeField(auto_now_add) |
| `download_type` | 다운로드 종류 | CharField(10, DOWNLOAD_TYPE) |
| `conditions` | 조회 조건 | TextField(blank) |
| `model_names` | 대상 모델 | TextField — 당시 모델명 스냅샷 |

- 이름·이메일·모델명은 **스냅샷 텍스트**: 나중에 이름이 바뀌거나 모델이 삭제돼도 "그때 무엇을 받았는지"가 남아야 한다.
- Admin은 **조회 전용** (추가·수정·삭제 불가, 시스템 관리자 포함).
- 파일명은 규칙 `SIMS_{종류}_{YYYYMMDD_HHMM}.xlsx`로 만들어 `download_type` + `downloaded_at`으로 재구성 가능.
- 접속 IP, 화면 조회 기록은 남기지 않는다 (필요해지면 추가).

---

## 8. 이관 규칙 (참고)

- `24.6 ps | 18.4 kW` 같은 이중 표기 → metric 값(18.4, kW)만 저장
- `-`, N/A, 빈칸 → SpecValue를 만들지 않음
- `(L) 2390 mm / (R) 2370 mm` → 좌·우 항목에 각각 저장
- 엔진 제조사·모델, 변속, 마력 → SpecValue가 아니라 ProductModel 컬럼으로 (중복 저장 금지)
- `- 단종` 표기 → 모델명에서 떼어 `status=discontinued`
- 기대스펙의 "Mechanical" 등 표현 → 선택지 코드로 변환 (변환표는 이관 스크립트에서 관리)
- 이관은 `bulk_create_with_history` + **이관 전용 계정**으로 수행해 사람의 수정과 구분

## 9. 나중에 추가할 것

| 항목 | 시점 | 내용 |
|-|-|-|
| 적합 정보 테이블 | 3차 (작업기) | `implement`(ProductModel, 로더·백호), `tractor`(ProductModel, 비어도 됨), `tractor_name`(타사 트랙터 이름), `note`. 기존 4개 테이블 변경 없이 추가만 |
| 조수 필터 컬럼 | 콤바인·이앙기 추가 시 | 인터페이스 항목 PA 상세 제품군(C04 4조 등) 기준 |
| 접속 IP | 보안 요구 시 | DownloadLog에 빈칸 허용 필드 추가 |
