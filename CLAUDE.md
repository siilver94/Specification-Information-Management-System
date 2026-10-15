# CLAUDE.md — SIMS 작업 가이드

> Claude Code가 이 저장소에서 작업할 때 따르는 규칙입니다.
> **이 파일은 공개 저장소에 올라가는 버전입니다.** 사내 정보(도메인, 서버 경로, 사내 시스템명, 원본 파일명, 인원)는 이 파일·README·코드 주석에 쓰지 않고, 아래 개인 파일(저장소 밖)에서 불러옵니다. 파일이 없는 환경에서는 이 줄은 무시됩니다.

@~/.claude/sims-internal.md

---

## 1. 프로젝트 요약

SIMS(Specification Information Management System, 사양 정보 관리 시스템)는 농기계(트랙터·콤바인·이앙기·엔진·작업기) 제품 사양을 하나의 DB로 통합하고, 임직원이 모델·국가별 최신 사양을 조회하는 사내 웹 시스템이다.

* **1차 범위(MVP)**: 트랙터 사양 조회
* **1차 기능**: 국가·마력·엔진 드롭박스 AND 조회, 검색 필터, 모델 비교(2~10개), 단위 전환(metric ↔ imperial), 엑셀 다운로드
* **현재 단계**: Hello SIMS 완료 → 데이터 모델 확정 → **다음: `specs` 앱 + models.py 구현**
  * 완료: Django 프로젝트(`sims`) + `accounts` 앱, Vue 프로젝트(Vite), 환경 변수 분리, Dockerfile·docker-compose, Jenkins Job, `master` push → 자동 배포 → 컨테이너 기동 성공
  * 남음: Django 모델 구현, PostgreSQL 연결, 비밀키·`DEBUG=False` 적용, WhiteNoise로 Vue 서빙, 로그인(allauth), 리버스 프록시 연결

### 현재 코드 상태

**Backend**
* `settings.py`는 보안 값을 환경 변수에서 읽는다: `DJANGO_DEBUG`(기본 `False`), `DJANGO_SECRET_KEY`, `DJANGO_ALLOWED_HOSTS`, `DJANGO_CSRF_TRUSTED_ORIGINS`
  * `DEBUG=False`인데 `DJANGO_SECRET_KEY`가 없으면 `ImproperlyConfigured`로 바로 실패한다. `DEBUG=True`면 실행할 때마다 임시 키를 만든다.
  * `LANGUAGE_CODE='ko-kr'`, `TIME_ZONE='Asia/Seoul'`, `STATIC_ROOT=BASE_DIR/'staticfiles'`
* DB는 아직 SQLite다. 컨테이너 안에 생기므로 **재배포하면 데이터가 초기화된다.**
* `INSTALLED_APPS`에 `accounts`, `rest_framework` 등록 완료. 프로젝트 모델·migration은 아직 없다.

**Frontend**
* `npm create vue`로 만든 초기 상태다. 예시 파일(`HelloWorld.vue`, `TheWelcome.vue`, `WelcomeItem.vue`, `components/icons/*`, `stores/counter.js`, `views/AboutView.vue`, `assets/logo.svg`)은 실제 화면을 만들 때 삭제한다.
* `@`는 `src/`를 가리키는 경로 별칭이다. Django API 호출용 Vite proxy 설정은 아직 없다.
* Docker 이미지에 Vue 빌드 결과가 들어가지만, 아직 Django가 서빙하지 않는다 (WhiteNoise 단계에서 연결).

**배포**
* 서비스 이름·컨테이너 이름: `sims`. Jenkins 스크립트의 서비스 이름과 반드시 일치시킨다.
* `migrate`는 Dockerfile `CMD`가 컨테이너 시작 때 실행한다. Jenkins에서 따로 돌리지 않는다.
* 현재 테스트 단계라 포트를 직접 노출하고 `DJANGO_DEBUG=True`로 실행 중이다 (임시). 리버스 프록시 연결 시 정리한다.

### 트러블슈팅 기록

|증상|원인|해결|
|-|-|-|
|`no such service: web`|Jenkins 스크립트와 compose의 서비스 이름 불일치|compose 서비스 이름(`sims`)과 Jenkins 명령을 일치|
|서버에서 `COPY ... not found`|Windows에서 폴더명 대소문자만 바꾸면 Git이 감지 못해 Linux에 옛 이름으로 clone됨|`git mv A tmp && git mv tmp B`로 두 번 옮김|
|빌드 성공 후 `exit code 137`, 에러 메시지 없음|컨테이너 메인 프로세스가 죽으면서 그 안의 `exec` 명령이 강제 종료됨|서버에서 `docker logs --tail 30 sims`로 실제 에러 확인|
|`.env`의 비밀키 값이 깨짐|docker compose가 `.env` 값 안의 `$`를 변수로 해석|`secrets.token_urlsafe(50)`처럼 `$`가 없는 키 사용|

---

## 다음 작업: Django 모델 구현

설계는 확정됐다. **필드 상세는 `docs/DATA_MODEL.md`를 기준으로 한다.**

1. `specs` 앱 생성, `INSTALLED_APPS` 등록
2. models.py: ProductModel → SpecItem → SpecValue → DownloadLog 순으로 하나씩 (각 단계에서 개발자 확인)
3. django-simple-history 설치(버전 고정, §3 규칙) 후 ProductModel·SpecValue에 적용
4. Admin 등록: ProductModel 화면에 SpecValue를 inline 표로
5. 로컬 SQLite에서 트랙터 모델 1~2개를 Admin으로 수기 입력해 구조 검증
6. PostgreSQL 컨테이너 + 볼륨 연결 (실제 데이터 이관 전 필수)

원본 데이터는 `data/`(로컬 전용, 대외비)에 있다. 본기 통합 사양표(시장별 트랙터 시트 + 콤바인·이앙기·엔진), 작업기 사양 엑셀 10종, 사내 제품사양서·옵션 마스터가 있다. **파일 내용은 커밋하지 않는다.**

---

## 2. 개발자와 협업하는 방식 (가장 중요)

개발자는 데이터 분석가 출신으로 Python 중급, Django·Vue·Docker는 이 프로젝트로 익히는 중이다.

* **응답은 한국어**로 한다. 코드·기술 용어는 영어 그대로 쓴다.
* **한 번에 다 만들지 않는다.** 작업을 단계로 나눠 계획을 먼저 보여주고, 확인을 받은 뒤 진행한다.
* 코드를 쓰기 전에 **무엇을, 왜** 하는지 먼저 설명하고, 쓴 뒤에는 동작 원리를 짧게 정리한다.
* 더 나은 방법이 있으면 **현재 방식의 문제 → 대안 → 이유** 순으로 먼저 제안한다.
* 선택지가 있으면 비교표와 함께 **추천안과 근거**를 준다.
* 설계 선택 시 "번거로움"보다 **"초보자가 구현·유지보수할 수 있는가"** 를 중요한 기준으로 본다.
* 6개월 뒤 개발자가 혼자 다시 봐도 이해할 수 있는 코드를 우선한다. 짧은 코드보다 명확한 코드.

## 3. 기술 스택

|영역|사용 기술|
|-|-|
|Language|Python 3.13 (로컬 PC와 Docker 이미지 버전을 반드시 동일하게)|
|Backend|Django 6.1.x, Django REST Framework 3.18.x|
|WSGI 서버|waitress|
|Frontend|Vue 3.5, Vite, Vue Router, Pinia, axios, scoped CSS|
|Node|`package.json`의 `engines` 기준, 로컬과 Docker 빌드 이미지를 같은 메이저 버전으로|
|DB|PostgreSQL (현재는 SQLite, 이관 전 전환)|
|Auth|django-allauth + Microsoft Entra ID (단일 테넌트)|
|Audit|django-simple-history|
|Static|WhiteNoise (Vue 빌드 결과를 Django가 서빙)|
|Infra|Docker 멀티스테이지 빌드, Docker Compose, Reverse Proxy, Jenkins|
|Data migration|pandas, openpyxl|

* **Django는 6.1.x 시리즈로 고정**한다 (사내 기준). 마이너 버전은 올리지 않되, 보안 패치(6.1.x)는 반영한다.
* Backend 패키지는 `BackEnd/requirements.txt`에 버전을 고정해서 관리한다.
* **새 라이브러리를 추가하기 전에** 왜 필요한지, 대안은 무엇인지 설명하고 확인을 받는다.

## 4. 저장소 구조

```text
Specification-Information-Management-System/
├── BackEnd/
│   ├── venv/                  # Python 가상환경 (커밋 제외)
│   ├── requirements.txt
│   └── sims/                  # Django 프로젝트 루트 (manage.py, Docker에서는 /app)
│       ├── manage.py
│       ├── sims/              # 프로젝트 설정
│       ├── accounts/          # 로그인·사용자
│       └── specs/             # 사양 데이터 (예정)
├── FrontEnd/
│   └── sims/                  # Vue 3 + Vite 프로젝트 루트
├── docs/
│   └── DATA_MODEL.md          # 데이터 모델 상세 (models.py의 기준)
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .dockerignore
├── .gitignore
├── data/                      # 원본 사양 자료 (로컬 전용, 대외비 — 커밋 금지)
└── CLAUDE.md
```

* **폴더 이름은 `BackEnd`/`FrontEnd`(대문자 E)다.** 서버(Linux)는 대소문자를 구분한다.
* Django 경로가 `BackEnd/sims/sims/`처럼 `sims`가 겹친다. 바깥은 프로젝트 루트, 안쪽은 설정 폴더다.
* 배포 파이프라인 스크립트(Jenkins)는 Jenkins Job 내부에서 관리한다. 저장소에 Jenkinsfile을 만들지 않는다.

## 5. 자주 쓰는 명령어

```bash
# 가상환경 활성화 — Django 명령 전에 먼저 실행
source BackEnd/venv/Scripts/activate      # Git Bash
# BackEnd\venv\Scripts\Activate.ps1      # PowerShell

# 로컬 개발 시 DEBUG 켜기 (settings.py 기본값이 False)
export DJANGO_DEBUG=True                  # Git Bash
# $env:DJANGO_DEBUG="True"                # PowerShell

# Django (BackEnd/sims/ 에서)
cd BackEnd/sims
pip install -r ../requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py runserver

# Vue (FrontEnd/sims/ 에서)
cd FrontEnd/sims
npm install
npm run dev
npm run build

# 전체 환경 (저장소 루트, .env 필요)
docker compose up -d --build sims
```

## 6. 아키텍처 원칙

* 컨테이너는 **app(Django) + PostgreSQL 2개**. 프론트 전용 웹서버 컨테이너를 만들지 않는다.
* Vue 화면, API, Django Admin **모두 로그인 필수**.
* API는 `/api/` 하위에 둔다. Vue는 같은 도메인에서 API를 호출한다.
* 데이터 수정 화면은 Vue로 만들지 않고 **Django Admin**을 사용한다.
* 애플리케이션 포트는 외부에 노출하지 않는다 (리버스 프록시 경유). 현재 테스트용 노출은 리버스 프록시 연결 시 제거한다.

## 7. 데이터 모델 원칙 (확정 — 상세는 `docs/DATA_MODEL.md`)

* **구조**: `specs` 앱에 세로 테이블 4개 — `ProductModel` / `SpecItem` / `SpecValue` / `DownloadLog`. 저장은 세로(항목당 1줄), 화면은 가로(pivot).
* **배제한 안**: 정규화 테이블 9개, 사양을 JSON 필드에 담는 방식, 기종별 넓은 테이블. 구조 변경이 필요해 보이면 코드 전에 개발자에게 먼저 제안한다.
* **ProductModel 1줄 = 사내 표준 기종 구분(MI) 단위**: 국가·마력·엔진·변속·ROPS가 다르면 별도 줄, 옵션만 다르면 같은 줄.
* MI 요소(`market`, `rated_hp`, `engine_maker`, `engine_model`, `transmission`, `rops_type`)는 **ProductModel 컬럼**(조회 필터)이다. 나머지 상세 사양은 SpecValue. **같은 값을 두 곳에 저장하지 않는다.**
* 기종·시장·엔진·별칭은 별도 테이블 없이 컬럼으로 둔다. `aliases`는 쉼표 구분 텍스트(사내 모델코드, 판매처 모델명, 구명칭).
* 사양 항목은 **SpecItem(항목 사전)** 으로 관리한다. `code`는 영문 소문자_밑줄이며 한 번 정하면 바꾸지 않는다.
* 수치는 **metric만 저장**(출력은 kW), `DecimalField` 사용. imperial·ps·hp는 화면에서 변환한다.
* 값이 없으면(`-`, N/A, 빈칸) SpecValue를 만들지 않는다. 한 칸에 두 값(L/R, F/R)은 항목을 나눈다.
* SpecValue: `(product_model, spec_item)` 유일, `spec_item`은 `PROTECT`, `value_type`에 맞는 칸만 채워지도록 `clean()`에서 검증한다.
* 옵션도 SpecItem(`kind=option`)/SpecValue로 저장한다(2차). 기본/선택은 텍스트로: `할로겐 (기본) / LED (선택)`.
* django-simple-history는 `ProductModel`, `SpecValue`에만 적용한다. 엑셀 이관은 `bulk_create_with_history` + **이관 전용 계정**으로 수행한다.
* `mi_code`, `product_code`, `series`는 지금은 비워둔다 (판매 모델명 ↔ 사내 코드 매칭 자료 확보 후 채움).

## 8. 코드 작성 규칙

* **주석은 한국어로 상세히**: 각 블록이 무엇을 하는지, 왜 이렇게 했는지.
* 함수·변수명은 의미가 바로 드러나게 (`convert_mm_to_inch`, `spec_values_by_market`).
* Python은 PEP 8, 함수에는 type hint와 짧은 docstring을 단다.
* 예상 가능한 오류는 미리 처리하고, 사용자에게 보이는 메시지는 이해할 수 있게 쓴다.
* 단위 변환, 엑셀 파서처럼 **틀리면 데이터가 오염되는 로직은 테스트를 함께 작성**한다.
* 과도한 추상화를 피한다. 요청받지 않은 대규모 리팩터링을 하지 않는다.
* 이미 적용된 migration 파일을 임의로 수정·삭제하지 않는다.

## 9. Git 규칙

|브랜치|용도|
|-|-|
|`master`|운영 배포 (push 시 자동 배포)|
|`develop`|통합 개발, 배포 전 로컬 테스트|
|`frontend`|프론트엔드 작업|
|`backend`|백엔드 작업|

* 커밋 메시지 접두어: `feature:` `fix:` `docs:` `refactor:` `chore:` (예: `feature: ProductModel 모델 추가`)
* `master`, `develop`에서 직접 개발하지 않는다. 작업 전 `develop` 최신 내용을 반영한다.
* **`master`로의 merge·push는 Claude가 하지 않는다.** `master` push는 곧 운영 배포다.
* commit·push 등 되돌리기 어려운 작업은 실행 전에 무엇을 할지 보여주고 확인을 받는다.

## 10. 보안 규칙 (반드시 지킬 것)

* `.env`, 비밀번호, API 키, Client Secret, `SECRET_KEY`를 **커밋하거나 코드에 하드코딩하지 않는다.**
* **실제 사양 데이터(엑셀, CSV, DB 덤프)와 사내 문서(pptx, docx, pdf)를 커밋하지 않는다.** `data/`에 두고 `.gitignore`로 제외한다. 테스트에는 가짜 데이터를 쓴다.
* **사내 도메인, 서버 경로, 사내 시스템명, 원본 파일명, 인원 이름을 공개되는 파일(README, CLAUDE.md, docs, 코드 주석)에 쓰지 않는다.** 이런 정보는 `~/.claude/sims-internal.md`에만 둔다.
* 운영 설정: `DEBUG=False`, `ALLOWED_HOSTS`·`CSRF_TRUSTED_ORIGINS`는 환경 변수로 제한.
* 엑셀 다운로드 기능은 다운로드 기록(누가·언제·무엇을)과 파일 내 다운로드자·일시·대외비 표기를 포함한다.
