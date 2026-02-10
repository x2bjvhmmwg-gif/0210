# 02/10 수업 정리 (Docker · OAuth · Kafka 연계)

## 📌 수업 목표

* Docker Compose를 이용해 **여러 서비스(Kafka, Redis, DB, Backend, Frontend)** 를 하나의 네트워크로 구성한다.
* FastAPI 기반 백엔드에서 **OAuth 2.0 (카카오 로그인)** 흐름을 이해하고 구현한다.
* Kafka를 이용해 **서비스 간 직접 통신을 제거한 이벤트 기반 구조**를 이해한다.

---

## 🧱 전체 아키텍처 개요

```
[ Browser ]
     │
     ▼
[ App3 (React, :80) ]
     │ API 호출
     ▼
[ App1 (FastAPI, :8000) ] ──▶ Kafka ──▶ App2 (FastAPI)
     │
     ├─ Redis (세션/캐시)
     └─ MariaDB (영속 데이터)
```

* 모든 컨테이너는 `my-bridge` 네트워크로 연결됨
* 컨테이너 간 통신은 **IP가 아닌 서비스 이름** 사용

---

## 🐳 Docker Compose 사용 이유

* 다수의 컨테이너를 한 번에 관리
* 환경변수 / 네트워크 / 의존성 관리 용이
* `docker run` 대비 **가독성과 재현성 향상**

---

## 📦 서비스 구성 설명

### 1️⃣ Kafka

* 역할: 서비스 간 이벤트 전달 (비동기 통신)
* 핵심 포인트:

  * `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092`
  * 다른 컨테이너에서는 `kafka:9092`로 접근

---

### 2️⃣ Redis

* 역할: 세션, 캐시, 임시 데이터 저장
* DB 접근 최소화를 위한 보조 저장소

---

### 3️⃣ MariaDB

* 역할: 영속 데이터 저장
* 볼륨 사용 목적:

  * 컨테이너 삭제 후에도 데이터 유지
* `initdb.d`를 통해 초기 테이블 자동 생성 가능

---

### 4️⃣ App1 / App2 (FastAPI)

* 공통 이미지: `uv:1`
* 코드만 다르게 마운트

```yaml
volumes:
  - ./app1:/workspace
```

* 개발용 구조:

  * 컨테이너는 환경만 제공
  * 서버 실행은 수동 (`sleep infinity`)

```bash
docker exec -it app1 bash
uv run fastapi run
```

---

### 5️⃣ App3 (React)

* 포트: `80:5173`
* OAuth Redirect URI 대응을 위해 80 포트 사용
* 개발 중 `npm run dev` 수동 실행

---

## 🔐 카카오 OAuth 로그인 흐름

### 1️⃣ 로그인 요청 (Redirect)

```python
@app.get("/login/kakao")
async def kakao_login():
    kakao_auth_url = (
        f"https://kauth.kakao.com/oauth/authorize?"
        f"client_id={KAKAO_REST_API_KEY}&"
        f"redirect_uri={KAKAO_REDIRECT_URI}&"
        f"response_type=code"
    )
    return RedirectResponse(kakao_auth_url)
```

* 브라우저를 **카카오 로그인 페이지로 이동**시킨다.
* 비밀번호는 우리 서버를 거치지 않는다.

📎 공식 문서: Kakao OAuth 2.0 Authorization Code

* [https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-code](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-code)

---

### 2️⃣ 콜백 처리 + 토큰 발급

```python
async def get_token(client, code: str):
    return await client.post(
        "https://kauth.kakao.com/oauth/token",
        data={
            "grant_type": "authorization_code",
            "client_id": KAKAO_REST_API_KEY,
            "redirect_uri": KAKAO_REDIRECT_URI,
            "code": code,
            "client_secret": KAKAO_CLIENT_SECRET,
        },
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
```

```python
@app.get("/oauth/callback/kakao")
async def kakao_callback(code: str):
    async with httpx.AsyncClient() as client:
        token_response = await get_token(client, code)
        tokens = token_response.json()
        return tokens
```

* `code`는 **1회용 인증 코드**
* Access Token / Refresh Token 발급

📎 공식 문서: Kakao Access Token 발급

* [https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-token](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-token)

---

### 3️⃣ 사용자 정보 조회

```python
async def get_user_info(client, access_token: str):
    return await client.get(
        "https://kapi.kakao.com/v2/user/me",
        headers={"Authorization": f"Bearer {access_token}"}
    )
```

```python
@app.get("/kakao/me")
async def kakao_user_info(access_token: str):
    async with httpx.AsyncClient() as client:
        response = await get_user_info(client, access_token)
        user_info = response.json()
        return user_info
```

* Access Token을 **Authorization 헤더**에 포함

📎 공식 문서: Kakao 사용자 정보 조회

* [https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#req-user-info](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#req-user-info)

---

## 📚 FastAPI / HTTP 관련 공식 문서 참고

* FastAPI 공식 사이트

  * [https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/)

* FastAPI CORS 설정

  * [https://fastapi.tiangolo.com/tutorial/cors/](https://fastapi.tiangolo.com/tutorial/cors/)

* FastAPI RedirectResponse

  * [https://fastapi.tiangolo.com/advanced/custom-response/#redirectresponse](https://fastapi.tiangolo.com/advanced/custom-response/#redirectresponse)

* httpx AsyncClient

  * [https://www.python-httpx.org/async/](https://www.python-httpx.org/async/)

---

## ⚠️ 수업 중 발생한 주요 이슈 & 포인트

* Windows에서 생성한 `.venv`를 Docker(Linux)에 마운트하면 오류 발생

  * 해결: `.venv`는 컨테이너 내부에서 생성
* 컨테이너 간 통신 시 `localhost` 사용 ❌

  * 반드시 서비스 이름 사용 ⭕

---

## 🎯 수업 핵심 정리

* Docker Compose 기반 멀티 서비스 환경 구성
* OAuth 2.0 인증 흐름 이해
* Kafka를 통한 이벤트 기반 아키텍처 입문
* 개발 환경과 운영 환경 분리 개념 학습

---

## 📝 한 줄 요약 (발표/면접용)

> Docker Compose로 인프라를 구성하고, OAuth 인증과 Kafka를 활용해 서비스 간 결합도를 낮춘 구조를 실습했다.
