# SIMS 데이터 모델 상세

> 2026-10-12 확정, 2026-10-13 ProductModel 필드 보완 (통합기종명·형식명·D365 기종코드). models.py는 이 문서를 기준으로 작성한다. 구조를 바꾸면 이 문서를 먼저 고친다.

## 1. 한눈에 보기

```text
ProductModel (모델 1개 = 1줄)
   │ 1
   │
   │ N
SpecValue (모델 × 항목 = 값) ── N : 1 ── SpecItem (항목 사전)

DownloadLog (엑셀 다운로드 기록, 독립)
```

- 저장은 **세로(long)**: 항목 하나당 SpecValue 한 줄
- 화면은 **가로(wide)**: 행 = 항목, 열 = 모델로 pivot
- 앱: `specs` 하나에 4개 테이블 모두 둔다

## 2. ProductModel

모델 1개 = 1줄. 사내 표준 기종 구분 단위(국가·출력·엔진·변속·ROPS 조합)를 따른다.

| 필드 | Django 타입 | 필수 | 설명 | 예 |
|---|---|---|---|---|
| `category` | CharField(choices) | ✅ | 기종 | `tractor` |
| `name` | CharField(50) | ✅ | 판매 모델명 | `T25` |
| `unified_name` | CharField(20) | | 통합기종명 (국가 공통 이름, 변형·시장 묶음용) | `T1025` |
| `model_type_name` | CharField(30) | | 형식명에서 타이어 자리를 뺀 이름 (MI와 1:1) | `T1025HREU` |
| `market` | CharField(choices) | ✅ | 사양 국가군 | `KR` / `NA` / `EU` |
| `rated_hp` | PositiveIntegerField | | 공칭 마력 (마력 드롭박스). 작업기는 비움 | `25` |
| `engine_maker` | CharField(50) | | 엔진 제조사 (엔진 드롭박스) | `Yanmar` |
| `engine_model` | CharField(50) | | 엔진 모델 | `3TNV80F` |
| `transmission` | CharField(50) | | 변속 방식 (자유 텍스트) | `HST` |
| `rops_type` | CharField(50) | | ROPS / 캐빈 형태 | `2주식 ROPS` |
| `status` | CharField(choices) | ✅ | 개발 / 양산 / 단종 | `양산` |
| `mi_code` | CharField(20) | | 사내 표준 기종 구분 코드(MI). 확보 시 입력 | `TAAEC025DA` |
| `d365_item_codes` | TextField | | D365 기종코드, 쉼표 구분. 타이어 변형마다 코드가 달라 한 줄에 여러 개 (매출·클레임 데이터 연결용) | `TA00074843, TA00072430` |
| `product_code` | CharField(10) | | 사내 제품코드 (확보 시 입력) | |
| `series` | CharField(10) | | 시리즈 (확보 시 입력) | |
| `aliases` | TextField | | 다른 이름들, 쉼표 구분 (사내 모델코드, 판매처 모델명, 구명칭) | |
| `created_at` / `updated_at` | DateTimeField | 자동 | | |

- **중복 금지**: `(category, market, name)` 조합은 하나만
- **변경 이력**: simple-history 적용
- 변속·ROPS는 사내 코드 목록이 실제 데이터와 다른 경우가 있어 고정 코드가 아닌 텍스트로 저장
- 형식명 규칙 (2024년 이후 신규 이름): `통합기종명 + 변속 + 구조물 + 시장 + 타이어` (예: `T1025 H R EU 0`)
  - 변속: M 기계식 / H HST / P 파워셔틀, 구조물: R Rear ROPS / F Front ROPS / C Cabin, 타이어: 0 없음 / 1 AGRI / 3 TURF / 4 INDI
  - **타이어는 옵션**이라 ProductModel을 나누지 않는다 → `model_type_name`은 타이어 자리를 뺀 값
  - 옛 기종명(예: `T255NHEU1`)은 시리즈마다 접미사 뜻이 달라 파싱하지 않고 `aliases`에 넣는다

## 3. SpecItem (항목 사전)

| 필드 | Django 타입 | 필수 | 설명 | 예 |
|---|---|---|---|---|
| `category` | CharField(choices) | ✅ | 이 항목이 속한 기종 | `tractor` |
| `kind` | CharField(choices) | ✅ | 사양 / 옵션 | `spec` |
| `section` | CharField(50) | ✅ | 화면 묶음 | `엔진` |
| `code` | CharField(50) | ✅ | 영문 소문자_밑줄. **한 번 정하면 바꾸지 않음** | `gross_power` |
| `name_ko` / `name_en` | CharField(100) | ✅ | 화면 표시명 | `엔진 출력` / `Gross engine power` |
| `unit` | CharField(20) | | 저장 단위 (metric) | `kW` |
| `value_type` | CharField(choices) | ✅ | 숫자 / 텍스트 | `number` |
| `display_order` | PositiveIntegerField | ✅ | 화면 표시 순서 | `1` |
| `source_code` | CharField(20) | | 원천 시스템 옵션 코드 (옵션 연동 시) | |

- **중복 금지**: `(category, code)` 조합은 하나만
- 화면 이름을 바꾸고 싶으면 `name_ko`만 고친다 (`code`는 그대로)

## 4. SpecValue (모델 × 항목 = 값)

| 필드 | Django 타입 | 필수 | 설명 |
|---|---|---|---|
| `product_model` | ForeignKey(ProductModel, CASCADE) | ✅ | 모델 삭제 시 값도 함께 삭제 |
| `spec_item` | ForeignKey(SpecItem, **PROTECT**) | ✅ | 값이 남아 있는 항목은 삭제 불가 (실수로 항목 지워 데이터 날리는 것 방지) |
| `value_number` | DecimalField(12, 3) | | 숫자 항목 값 (metric) |
| `value_text` | CharField(255) | | 텍스트 항목 값 |

- **중복 금지**: `(product_model, spec_item)` 조합은 하나만
- **검증**: 숫자 항목은 `value_number`만, 텍스트 항목은 `value_text`만 채워져야 저장됨
- **변경 이력**: simple-history 적용
- `FloatField`가 아닌 `DecimalField`를 쓰는 이유: 실수 오차(예: `2215.4699999998`) 없이 원본 값 그대로 저장
- 옵션 값의 기본/선택 구분은 텍스트로: `할로겐 (기본) / LED (선택)`

## 5. DownloadLog

| 필드 | Django 타입 | 설명 |
|---|---|---|
| `user` | ForeignKey(User, SET_NULL) | 계정이 삭제돼도 기록은 남김 |
| `user_name` | CharField(100) | 다운로드 당시 이름 (스냅샷) |
| `downloaded_at` | DateTimeField(auto_now_add) | |
| `conditions` | TextField | 조회 조건 (예: `국가=국내, 마력=20~30`) |
| `model_names` | TextField | 대상 모델 목록 (스냅샷) |

- 대상 모델을 ForeignKey가 아닌 텍스트로 저장하는 이유: 나중에 모델명이 바뀌거나 삭제돼도 "그때 무엇을 받았는지"가 그대로 남아야 하므로

## 6. 단위 변환 (화면에서 계산)

| 저장 단위 | 표시 단위 | 변환 |
|---|---|---|
| kW | ps | × 1.35962 |
| kW | hp | × 1.34102 |
| mm | in | ÷ 25.4 |
| kg | lb | × 2.20462 |
| L | gal (US) | ÷ 3.78541 |
| km/h | mph | ÷ 1.60934 |
| L/min | gpm (US) | ÷ 3.78541 |
| cc | cu.in | ÷ 16.3871 |

## 7. 이관 규칙

- 원본의 `24.6 ps | 18.4 kW` 같은 이중 표기 → metric 값(18.4, kW)만 저장
- `-`, `N/A`, 빈칸 → SpecValue를 만들지 않음
- `(L) 2390 mm / (R) 2370 mm`처럼 한 칸에 두 값 → 항목을 나눠 각각 저장
- 엔진 제조사·모델, 변속, 마력 → SpecValue가 아니라 ProductModel 컬럼으로 (중복 저장 금지)
- `- 단종` 같은 상태 표기 → 모델명에서 떼어 `status`로
- 이관은 `bulk_create_with_history` + 이관 전용 계정으로 수행해, 이관 데이터와 사람의 수정을 이력에서 구분

## 8. 지금은 만들지 않는 것

| 항목 | 시점 |
|---|---|
| 작업기 ↔ 트랙터 적합 관계 테이블 | 작업기 확장 시 (5번째 테이블) |
| 옵션 조합·주문 사양 코드 | 필요 시 |
| 개별 기계 번호 매핑 | 클레임 분석 연계 시 |
