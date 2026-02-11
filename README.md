# Python-Django-FastAPI

이 저장소는 **FastAPI 기반 항구/선박 API 실험 코드**와 **Django 기반 관리/도메인 앱**이 함께 들어있는 학습용 프로젝트입니다.

코드를 보면 한 가지 단일 서비스라기보다는, 아래 2가지 축으로 구성되어 있습니다.

- FastAPI 축: 항구 데이터 조회 + 선박 위치 조회 API (`main.py`, `main-sim.py`, `ship.py`)
- Django 축: 항구/선박 모델과 Admin 관리 UI (`myproject/`)

---

## 1) 프로젝트 구성 요약

```text
.
├── main.py                 # FastAPI: ISAM 파일 + 인덱스 기반 /ports, /ship
├── main-sim.py             # FastAPI: 메모리(fake) 데이터 + 선박 이동 시뮬레이션
├── ship.py                 # ISAM 데이터/인덱스 파일 생성 및 조회 스크립트
├── test.py                 # ISAM 바이너리 파일 덤프 확인 스크립트
├── requirements.txt        # Python 의존성 목록 (현재 UTF-16 형식)
└── myproject/
    ├── manage.py           # Django 진입점
    ├── myproject/
    │   ├── settings.py     # Django 설정 (MongoDB 연결, 앱 등록 등)
    │   └── urls.py         # 루트 URL
    └── myapp/
        ├── models.py       # ShippingPort, Ship 모델
        ├── admin.py        # Django Admin 등록
        ├── urls.py         # /hello/ 라우트
        └── views.py        # hello 뷰
```

---

## 2) FastAPI 코드 분석

### `main.py`

- `FastAPI(title="Port + Ship API Service")` 앱 생성.
- CORS 허용 origin은 로컬 프론트엔드 개발 주소 위주로 설정.
- `find_record(id)`:
  - `INDEX_FILE`에서 ID와 파일 오프셋을 찾고,
  - `DATA_FILE`에서 고정 길이 레코드(`RECORD_SIZE=200`)를 읽어 파싱.
  - `id`, `port`, `lat`, `lon` JSON으로 반환.
- 엔드포인트:
  - `GET /ports`: 인덱스 전체를 순회해 모든 항구 반환
  - `GET /ports/{port_id}`: 단건 조회, 없으면 404
  - `GET /ship`: 고정된 fake 선박 정보 반환

> 참고: `DATA_FILE`이 Windows 절대경로(`C:\...`)로 하드코딩되어 있어, Linux/macOS 환경에서는 기본값 그대로 실행 시 파일 접근 문제가 발생할 수 있습니다.

### `main-sim.py`

- 실제 파일 대신 메모리 상 fake 포트/선박 데이터를 사용.
- `distance()`에서 Haversine 공식을 통해 거리(km) 계산.
- `GET /ship` 호출마다 다음 경유지 방향으로 선박 위치를 보간 이동.
- 시각화/프론트엔드 연동 테스트에 유용한 형태.

### `ship.py`

- 간단한 ISAM 형태의 데이터 파일/인덱스 파일 생성 스크립트.
- `insert_record()`로 고정 길이 레코드를 작성하고, `id,pos` 인덱스를 텍스트로 저장.
- `find_record()`로 인덱스를 통해 랜덤 액세스 조회.

### `test.py`

- 바이너리 데이터 파일(`isam_data.dat`)을 레코드 단위로 덤프해 실제 저장 내용을 확인.

---

## 3) Django 코드 분석

### 핵심 설정 (`myproject/myproject/settings.py`)

- `INSTALLED_APPS`에 `jazzmin`, `django_mongoengine`, `myapp` 등록.
- MongoDB 연결 정보는 `MONGODB_DATABASES`에 설정 (`localhost:27017`, `testdb`).
- 세션 엔진을 `django_mongoengine.sessions`로 설정.
- `AUTH_USER_MODEL = "myapp.User"`가 지정되어 있음.

### 도메인 모델 (`myproject/myapp/models.py`)

- `ShippingPort(country, port, lat, lon)`
- `Ship(name, status, lat, lon)`
- Admin 목록 표시용 `__str__` 구현.

### 라우팅/뷰

- 루트 URL에서 `myapp.urls`를 include.
- 현재 사용자 기능 엔드포인트는 `GET /hello/` 하나로, `Hello, World!` 반환.

### Admin

- `ShippingPort`, `Ship`가 Django Admin에 등록되어 있으며,
- list display/search/filter 설정이 되어 있어 기본 관리 UI 동작이 잘 준비됨.

---

## 4) 빠른 실행 가이드

## (A) FastAPI 실행

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\\Scripts\\activate

# requirements.txt가 UTF-16이므로 설치가 실패하면 인코딩 변환 후 설치
python -c "from pathlib import Path; p=Path('requirements.txt'); p.write_text(p.read_text(encoding='utf-16'), encoding='utf-8')"
pip install -r requirements.txt

uvicorn main:app --reload
# 또는
uvicorn main-sim:app --reload
```

API 확인:

- `http://127.0.0.1:8000/docs`
- `GET /ports`, `GET /ports/{id}`, `GET /ship`

## (B) Django 실행

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cd myproject
python manage.py migrate
python manage.py runserver
```

확인:

- `http://127.0.0.1:8000/hello/`
- `http://127.0.0.1:8000/admin/`

---

## 5) 현재 코드에서 확인되는 주의사항

1. **`requirements.txt` 인코딩**
   - 현재 UTF-16으로 저장되어 있어 일반적인 `pip install -r requirements.txt`에서 문제가 날 수 있습니다.

2. **FastAPI 데이터 파일 경로 하드코딩**
   - `main.py`, `ship.py`, `test.py`에 Windows 절대 경로가 있어 OS 독립성이 낮습니다.

3. **Django 커스텀 유저 설정 불일치 가능성**
   - `AUTH_USER_MODEL = "myapp.User"` 설정이 있으나, 현재 `myapp/models.py`에는 `User` 모델이 없습니다.
   - 실제 실행 시 인증 관련 오류가 날 수 있으므로 모델 추가 또는 설정 수정이 필요합니다.

4. **Django + MongoEngine + Django ORM 혼합 구조**
   - 현재 모델은 `django.db.models.Model`(ORM)인데 설정은 MongoEngine 기반 요소를 포함합니다.
   - 의도한 아키텍처(순수 ORM vs MongoEngine 문서 모델)를 정리하면 유지보수성이 좋아집니다.

---

## 6) 추천 개선 방향

- 환경변수 기반 설정으로 경로/DB/시크릿 분리 (`.env`)
- FastAPI 코드에서 데이터 접근 계층 분리 (파일 파서, 서비스, 라우터)
- Django 사용자 모델/인증 구조 정합성 정리
- 테스트 코드 추가 (`pytest`, Django TestCase 확장)
- `README`에 실제 운영에 사용할 단일 실행 경로(예: FastAPI만, 또는 Django만)를 명확히 명시

---

필요하면 다음 단계로, 위 분석을 바탕으로

- **실행 가능한 단일 아키텍처로 정리**하거나,
- **Docker 기반 개발환경**(`docker-compose`)까지 구성해드릴 수 있습니다.
