# CLAUDE.md — SIMS 작업 가이드

> Claude Code가 이 저장소에서 작업할 때 따르는 규칙입니다.
> 이 저장소는 **공개 저장소**이므로 사내 정보(도메인, 서버 경로, 사내 시스템명, 인원)는 이 파일에 쓰지 않고,
> 아래 개인 파일(저장소 밖)에서 불러옵니다. 파일이 없는 환경에서는 이 줄은 무시됩니다.

@~/.claude/sims-internal.md

---

## 1. 프로젝트 요약

SIMS(Specification Information Management System, 사양 정보 관리 시스템)는 농기계(트랙터·콤바인·이앙기·엔진·작업기) 제품 사양을 하나의 DB로 통합하고, 임직원이 모델·국가별 최신 사양을 조회하는 사내 웹 시스템이다.

- **1차 범위(MVP)**: 트랙터 사양 조회
- **1차 기능**: 국가·마력·엔진 드롭박스 AND 조회, 검색 필터, 모델 비교(2~10개), 단위 전환(metric ↔ imperial), 엑셀 다운로드
- **현재 단계**: Hello SIMS (빈 프로젝트로 배포 파이프라인 검증)

## 2. 개발자와 협업하는 방식 (가장 중요)

개발자는 데이터 분석가 출신으로 Python 중급, Django·Vue·Docker는 이 프로젝트로 익히는 중이다.

- **응답은 한국어**로 한다. 코드·기술 용어는 영어 그대로 쓴다.
- **한 번에 다 만들지 않는다.** 작업을 단계로 나눠 계획을 먼저 보여주고, 확인을 받은 뒤 진행한다.
- 코드를 쓰기 전에 **무엇을, 왜** 하는지 먼저 설명하고, 쓴 뒤에는 동작 원리를 짧게 정리한다.
- 더 나은 방법이 있으면 **현재 방식의 문제 → 대안 → 이유** 순으로 먼저 제안한다.
- 선택지가 있으면 비교표와 함께 **추천안과 근거**를 준다.
- 6개월 뒤 개발자가 혼자 다시 봐도 이해할 수 있는 코드를 우선한다. 짧은 코드보다 명확한 코드.

## 3. 기술 스택

| 영역 | 사용 기술 |
|---|---|
| Language | Python 3.13 (로컬 PC와 Docker 이미지 버전을 반드시 동일하게) |
| Backend | Django, Django REST Framework |
| Frontend | Vue 3, Vite, Vue Router, Pinia, axios, scoped CSS |
| DB | PostgreSQL |
| Auth | django-allauth + Microsoft Entra ID (단일 테넌트) |
| Audit | django-simple-history |
| Static | WhiteNoise (Vue 빌드 결과를 Django가 서빙) |
| Infra | Docker 멀티스테이지 빌드, Docker Compose, Reverse Proxy |
| Data migration | pandas, openpyxl |

- Django 버전은 사내 기존 시스템과 맞춘다 (확정 전까지 임의로 올리거나 바꾸지 않는다).
- **새 라이브러리를 추가하기 전에** 왜 필요한지, 대안은 무엇인지 설명하고 확인을 받는다.

## 4. 저장소 구조

```text
SIMS/
├── SIMS-Backend/          # Django + DRF (Admin, 데이터 이관 스크립트 포함)
├── SIMS-Frontend/         # Vue 3 + Vite
├── Dockerfile             # 멀티스테이지: Node로 Vue 빌드 → Python 이미지로 복사
├── docker-compose.yml     # app + PostgreSQL (비밀값은 .env로 주입)
├── .env.example           # 환경 변수 목록 (값 없음)
├── .dockerignore
├── .gitignore
├── README.md
└── CLAUDE.md
```

- 배포 파이프라인 스크립트(Jenkins)는 저장소가 아닌 Jenkins Job 내부에서 관리한다. 저장소에 Jenkinsfile을 만들지 않는다.

## 5. 자주 쓰는 명령어

> Hello SIMS 단계에서 실제 명령어로 확정·갱신한다.

```bash
# 전체 환경 실행 (운영과 같은 구성으로 로컬 통합 테스트)
docker compose up -d --build

# Django
python manage.py makemigrations
python manage.py migrate
python manage.py runserver

# Vue (SIMS-Frontend/)
npm install
npm run dev
npm run build
```

## 6. 아키텍처 원칙

- 컨테이너는 **app(Django) + PostgreSQL 2개**. 프론트 전용 웹서버 컨테이너를 만들지 않는다.
- Vue 화면, API, Django Admin **모두 로그인 필수**. 로그인 없이 접근 가능한 화면·API를 만들지 않는다.
- API는 `/api/` 하위에 둔다. Vue는 같은 도메인에서 API를 호출한다 (CORS 설정 최소화).
- 데이터 수정 화면은 Vue로 만들지 않고 **Django Admin**을 사용한다.
- 애플리케이션 포트는 외부에 노출하지 않는다 (`ports` 미사용, 리버스 프록시 경유).

## 7. 데이터 모델 원칙

- 분류 체계: **제품군 → 시리즈 → 제품코드 → MI(기종구분)**. 사양의 기본 단위는 MI(국가·마력·엔진·변속·ROPS 조합)다.
- 마케팅 모델명(예: 판매처·국가별 이름)은 MI에 연결되는 **별칭(alias)** 으로 관리한다.
- 사양 항목은 컬럼으로 하드코딩하지 않고 **항목 마스터(섹션 > 항목, 국문·영문명, 단위, 데이터 타입)** 로 관리한다. 기종이 늘어도 테이블 구조를 바꾸지 않기 위해서다.
- 수치는 **metric만 저장**하고 imperial은 화면 표시 시 변환한다. 둘 다 저장하지 않는다.
- 드롭박스·필터에 쓰는 값(마력 등)은 텍스트가 아닌 **숫자 필드**로 저장한다.
- django-simple-history는 사람이 수정하는 핵심 모델(사양 값, 모델명 별칭)에만 적용한다.
- 엑셀 이관은 `bulk_create_with_history` + **이관 전용 계정**으로 수행해, 이관 데이터와 사람의 수정을 구분한다.

## 8. 코드 작성 규칙

- **주석은 한국어로 상세히**: 각 블록이 무엇을 하는지, 왜 이렇게 했는지.
- 함수·변수명은 의미가 바로 드러나게 (`convert_mm_to_inch`, `spec_values_by_market`).
- Python은 PEP 8, 함수에는 type hint와 짧은 docstring을 단다.
- 예상 가능한 오류(파일 없음, 형식 오류, 빈 값, DB 연결 실패)는 미리 처리하고, 사용자에게 보이는 메시지는 이해할 수 있게 쓴다.
- 단위 변환, 엑셀 파서처럼 **틀리면 데이터가 오염되는 로직은 테스트를 함께 작성**한다.
- 과도한 추상화·디자인 패턴을 피한다. 단순하게 풀 수 있으면 단순하게.
- 요청받지 않은 대규모 리팩터링을 하지 않는다.
- 이미 적용된 migration 파일을 임의로 수정·삭제하지 않는다.

## 9. Git 규칙

| 브랜치 | 용도 |
|---|---|
| `master` | 운영 배포 (push 시 자동 배포) |
| `develop` | 통합 개발, 배포 전 로컬 테스트 |
| `front` | 프론트엔드 작업 |
| `back` | 백엔드 작업 |

- 커밋 메시지 접두어: `feature:` `fix:` `docs:` `refactor:` `chore:` (예: `feature: 트랙터 사양 조회 API 추가`)
- `master`, `develop`에서 직접 개발하지 않는다. 작업 전 `develop` 최신 내용을 반영한다.
- **`master`로의 merge·push는 Claude가 하지 않는다.** `master` push는 곧 운영 배포이므로 개발자가 직접 확인 후 수행한다.
- commit·push 등 되돌리기 어려운 작업은 실행 전에 무엇을 할지 보여주고 확인을 받는다.

## 10. 보안 규칙 (반드시 지킬 것)

- `.env`, 비밀번호, API 키, Client Secret을 **커밋하거나 코드에 하드코딩하지 않는다.** 설정값은 환경 변수로만 읽는다.
- **실제 사양 데이터(엑셀, CSV, DB 덤프)를 저장소에 커밋하지 않는다.** 사양 데이터는 대외비다. 테스트에는 가짜 데이터를 쓴다.
- 사내 도메인, 서버 경로, 사내 시스템명, 인원 이름을 공개되는 파일(README, CLAUDE.md, 코드 주석)에 쓰지 않는다.
- 운영 설정: `DEBUG=False`, `ALLOWED_HOSTS`·`CSRF_TRUSTED_ORIGINS`는 환경 변수로 제한.
- 엑셀 다운로드 기능은 다운로드 기록(누가·언제·무엇을)과 파일 내 다운로드자·일시·대외비 표기를 포함한다.
