# API Contract

FE와 BE가 공유하는 API 스펙. 변경 시 이 문서를 먼저 수정한다.

## 공통
- Base URL: dev `http://localhost:8000` (FE 는 `VITE_API_URL` 로 지정)
- 인증: **세션 쿠키** (`session`, HttpOnly). `POST /auth/login` 이 발급하고 BE 만 읽는다 — FE 는 저장하지 않고 `credentials: 'include'` 로 보낸다.
  - 로그인 유지(`remember: true`): 쿠키 7일, **로그인이 필요한 요청마다 7일로 다시 연장** (슬라이딩).
  - 로그인 유지 안 함(`remember: false`): 브라우저 세션 쿠키 (브라우저를 닫으면 사라짐). 서버 쪽 유효기간은 똑같이 7일 슬라이딩.
  - production: `SameSite=None; Secure` (FE·BE 도메인이 다르므로 HTTPS 필수). dev: `SameSite=Lax`.
  - 요청마다 DB 에서 사용자를 다시 읽는다 → **비활성화·역할 변경·비밀번호 변경이 다음 요청부터 바로 반영**된다 (비활성화·비밀번호 변경 시 기존 세션은 401).
- 권한: 각 엔드포인트의 **권한** 줄. `없음` = 누구나, `로그인` = 로그인만, 그 밖은 필요한 Feature.
  | Role | 이름 | Feature |
  |---|---|---|
  | `admin` | 관리자 | `research`, `account_view`, `account_edit`, `user_manage` |
  | `member` | 직원 | `research`, `account_view` |

  Feature 문자열은 FE `constants/permissions.js` · BE `core/permissions.py` 와 같아야 한다.
- 형식: 요청·응답 모두 JSON. 성공 응답은 `{"status": "success", ...}`.
- 에러 형식: `{"detail": "한국어 메시지"}`
  | 코드 | 언제 |
  |---|---|
  | 400 | 로그인 실패 (아이디·비밀번호 틀림) · 규칙 위반 (자기 자신 비활성화 등) |
  | 401 | 로그인 안 됨 · 세션 만료 · 계정이 비활성화됨 · 비밀번호가 바뀜 |
  | 403 | 로그인은 됐지만 권한(Feature) 없음 · 비활성 계정으로 로그인 시도 |
  | 404 | 대상이 없음 |
  | 409 | 코드 발급 충돌이 재시도로도 풀리지 않음 · 하루 발급 한도(FFFF) 초과 · 이미 있는 아이디 |
  | 422 | 입력 검증 실패 — `detail` 이 `[{loc, msg, type}]` 배열 (FastAPI 기본) |
  | 500 | 서버 오류 (내부 메시지는 노출하지 않음) |
- 날짜: `survey_date` 는 `YYYY-MM-DD`. `created_at`·`updated_at` 은 UTC ISO 8601 (FE 에서 KST 로 표시).
- 위안 가격(`price_cny`)은 소수 오차를 막으려고 **문자열**로 주고받는다 (예: `"39.90"`). 요청에는 숫자도 받는다.
- 원화 가격(`*_min_price`)은 정수.
- 목록 조회 공통 쿼리: `page` (기본 1), `size` (기본 20, 최대 100). 응답에 `total` 포함.
- **코드는 BE 가 발급한다** — 요청에 `code` 를 보내도 무시된다.
  | 코드 | 형식 | 날짜 기준 | 예 |
  |---|---|---|---|
  | 제품 코드 | `PR-YYMMDD-XXXX` | 제품 추가일 (KST) | `PR-261018-0001` |
  | 조사 코드번호 | `RS-YYMMDD-XXXX` | 조사일 (`survey_date`) | `RS-261018-000A` |

  `XXXX` = 그 날짜의 순번, 16진수 대문자 4자리 (`0001` ~ `FFFF`, 하루 최대 65,535건).
  입력 화면에서는 `next-code` 로 미리 보여주되, **실제 코드는 등록 응답의 `code`** 를 쓴다 (그 사이 다른 등록이 있으면 달라진다).

---

## 엔드포인트

### GET /health
서버 + DB 연결 확인.
- 권한: 없음
- Response 200: `{"status": "success", "db": "ok"}`
- Response 503: DB 연결 실패

---

### POST /products — 제품 등록
- 권한: `research`
- Request
  ```json
  {
    "photo_url": "https://...jpg",
    "title_cn": "无线蓝牙耳机",
    "title_ko": "무선 블루투스 이어폰"
  }
  ```
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `photo_url` | | `http(s)://` 주소 또는 `POST /uploads/photos` 가 준 `/files/...` 경로, 1000자 이하 |
  | `title_cn` | ✅ | 1~300자 |
  | `title_ko` | | 300자 이하 |
- Response 200: `{"status": "success", "product": Product}`
- Response 409: 코드 발급 충돌 (드묾 — 다시 시도)

### GET /products/next-code — 다음 제품 코드 미리보기
- 권한: 로그인
- Response 200: `{"status": "success", "code": "PR-261018-0001"}`

### GET /products — 제품 목록
- 권한: 로그인
- Query: `q` (제품 코드·중국어 제목·한국어 제목 부분 검색), `page`, `size`
- 정렬: 최근 추가 순
- Response 200: `{"status": "success", "items": [Product + "survey_count"], "total": 12, "page": 1, "size": 20}`

### GET /products/{product_id} — 제품 상세
- 권한: 로그인
- Response 200: `{"status": "success", "product": Product, "price_surveys": [PriceSurveySummary]}` (조사일 최근 순)
- Response 404

**Product**
```json
{
  "id": 1, "code": "PR-261018-0001", "photo_url": null,
  "title_cn": "无线蓝牙耳机", "title_ko": "무선 블루투스 이어폰",
  "created_at": "2026-10-18T13:00:00+00:00", "updated_at": "2026-10-18T13:00:00+00:00"
}
```
`created_at` = 제품 코드 추가일.

---

### POST /price-surveys — 가격 조사 등록
조사 1건과 옵션 목록을 한 번에 저장한다 (전부 성공 또는 전부 실패).
- 권한: `research`
- Request
  ```json
  {
    "product_code": "PR-261018-0001",
    "survey_date": "2026-10-18",
    "photo_url": "https://...jpg",
    "taobao_url": "https://item.taobao.com/item.htm?id=...",
    "coupang_min_price": 19900,
    "coupang_url": "https://www.coupang.com/vp/products/...",
    "naver_min_price": null,
    "naver_url": null,
    "options": [
      {"option1": "검정", "option2": "L", "price_cny": "39.90"},
      {"option1": "흰색", "option2": null, "price_cny": 42}
    ]
  }
  ```
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `product_code` | ✅ | 등록된 제품 코드 (없으면 404) |
  | `survey_date` | ✅ | `YYYY-MM-DD` |
  | `photo_url` | | `http(s)://` 또는 `/files/...` (보통 연결된 제품 사진) |
  | `taobao_url` | ✅ | `http(s)://` |
  | `coupang_min_price` / `naver_min_price` | | 0 이상 정수 (원) |
  | `coupang_url` / `naver_url` | | `http(s)://` |
  | `options` | ✅ | 1개 이상, 같은 (옵션1, 옵션2) 조합 중복 불가 |
  | `options[].option1` | ✅ | 1~200자 |
  | `options[].option2` | | 200자 이하 |
  | `options[].price_cny` | ✅ | 0 초과, 소수 둘째 자리까지 |
- Response 200: `{"status": "success", "price_survey": PriceSurvey}`
- Response 404: 제품 코드 없음 · 409: 코드 발급 충돌 (드묾 — 다시 시도)

### GET /price-surveys/next-code — 다음 조사 코드번호 미리보기
- 권한: 로그인
- Query: `survey_date` (필수, `YYYY-MM-DD`)
- Response 200: `{"status": "success", "code": "RS-261018-0001"}`

### GET /price-surveys — 가격 조사 목록
- 권한: 로그인
- Query: `product_code` (해당 제품만), `page`, `size`
- 정렬: 조사일 최근 순
- Response 200: `{"status": "success", "items": [PriceSurveySummary], "total": 3, "page": 1, "size": 20}`

### GET /price-surveys/{survey_id} — 가격 조사 상세
- 권한: 로그인
- Response 200: `{"status": "success", "price_survey": PriceSurvey}`
- Response 404

**PriceSurvey**
```json
{
  "id": 1, "code": "RS-261018-0001", "survey_date": "2026-10-18",
  "product": {"id": 1, "code": "PR-261018-0001", "title_ko": "무선 블루투스 이어폰"},
  "photo_url": null, "taobao_url": "https://item.taobao.com/...",
  "coupang_min_price": 19900, "coupang_url": "https://...",
  "naver_min_price": null, "naver_url": null,
  "option_count": 2,
  "options": [
    {"id": 1, "option1": "검정", "option2": "L", "price_cny": "39.90"},
    {"id": 2, "option1": "흰색", "option2": null, "price_cny": "42.00"}
  ],
  "created_at": "...", "updated_at": "..."
}
```
**PriceSurveySummary** = PriceSurvey 에서 `options` 를 빼고 `min_price_cny` · `max_price_cny` (문자열)를 넣은 것.
`option_count` = 옵션 총 종류 (옵션 행 수로 계산).

---

### POST /uploads/photos — 사진 업로드
제품 등록 전에 먼저 올리고, 받은 `url` 을 `photo_url` 로 보낸다.
- 권한: `research`
- Request: `multipart/form-data`, 필드 `file`
  | 규칙 | |
  |---|---|
  | 형식 | JPG · PNG · WEBP · GIF (확장자가 아니라 파일 내용으로 판단) |
  | 크기 | 10MB 이하 |
- Response 200: `{"status": "success", "url": "/files/photos/2026/10/3a66e2fb….png", "size": 69}`
- Response 413: 10MB 초과 · 415: 이미지가 아님

### GET /files/{path} — 올린 사진 보기
- 권한: 없음
- 위 `url` 그대로 요청하면 사진 파일이 내려온다 (인증 없음).
- **`url` 은 `/files/...` 상대 경로다.** FE 는 API 주소를 앞에 붙여 띄운다 (`api/client.js` 의 `fileUrl()`).
- 저장 위치: BE 를 돌리는 PC 의 `BE/uploads/` (백업 대상).

---

### POST /auth/login — 로그인
- 권한: 없음
- Request: `{"username": "admin", "password": "********", "remember": true}`
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `username` | ✅ | 앞뒤 공백 제거 |
  | `password` | ✅ | |
  | `remember` | | 기본 `false`. `true` = 로그인 유지 (7일 슬라이딩) |
- Response 200: `{"status": "success", "user": AuthUser}` + `Set-Cookie: session=...`
- Response 400: 아이디나 비밀번호가 맞지 않음 (어느 쪽이 틀렸는지 알려주지 않는다)
  — 401 이 아닌 이유: FE `client.js` 는 401 을 받으면 로그인 화면으로 보내고 메시지를 덮어쓴다.
- Response 403: 비활성 계정 (비밀번호가 맞을 때만 알려준다)

### POST /auth/logout — 로그아웃
- 권한: 없음 (로그인 안 된 상태로 불러도 200)
- Response 200: `{"status": "success"}` — 세션 쿠키 삭제

### GET /auth/check — 로그인 확인
- 권한: 없음
- Response 200: `{"status": "success", "user": AuthUser}` (remember 세션이면 쿠키 연장)
- Response 401: 로그인 안 됨 · 세션 만료 · 비활성화됨

**AuthUser**
```json
{"id": 1, "username": "admin", "name": "관리자", "role": "admin"}
```

---

### GET /users — 사용자 목록
- 권한: `user_manage`
- 정렬: 먼저 만든 순
- Response 200: `{"status": "success", "items": [User]}`

### POST /users — 사용자 추가
- 권한: `user_manage`
- Request: `{"username": "kim", "name": "김직원", "password": "********", "role": "member"}`
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `username` | ✅ | 3~50자, 영문·숫자·`_`·`.`·`-`. 대소문자 구분 없이 저장 (소문자로 바꿈) |
  | `name` | ✅ | 1~50자 |
  | `password` | ✅ | 8자 이상, UTF-8 72바이트 이하 (bcrypt 한계 — 영문이면 72자) |
  | `role` | ✅ | `admin` · `member` |
- Response 200: `{"status": "success", "user": User}`
- Response 409: 이미 있는 아이디

### PATCH /users/{user_id} — 사용자 수정
보낸 필드만 바꾼다.
- 권한: `user_manage`
- Request: `{"active": false}` · `{"role": "admin"}` · `{"password": "********"}` (섞어도 됨)
  | 필드 | 규칙 |
  |---|---|
  | `active` | `false` = 사용 중지 → 그 사용자의 세션은 다음 요청부터 401 |
  | `role` | `admin` · `member` |
  | `password` | 규칙은 POST 와 같음. 바꾸면 그 사용자의 기존 세션은 모두 401 (다시 로그인) |
- Response 200: `{"status": "success", "user": User}`
- Response 400: 자기 자신을 비활성화하거나 자기 역할을 바꾸려 함 · 마지막 활성 관리자를 비활성화하거나 직원으로 바꾸려 함 · 바꿀 필드가 없음
- Response 404

**User**
```json
{
  "id": 2, "username": "kim", "name": "김직원", "role": "member", "active": true,
  "created_at": "2026-10-18T13:00:00+00:00", "updated_at": "2026-10-18T13:00:00+00:00"
}
```

---

### GET /company-accounts — 회사 계정 목록
타오바오·쿠팡·네이버 등 회사가 쓰는 외부 서비스 계정. **목록에는 비밀번호가 없다.**
- 권한: `account_view`
- Query: `usage` (선택 — 이 용도가 붙은 계정만, 정확히 일치)
- 정렬: 최근 추가 순
- Response 200: `{"status": "success", "items": [CompanyAccount]}`

### POST /company-accounts — 회사 계정 추가
- 권한: `account_edit`
- Request
  ```json
  {"platform": "타오바오", "login_id": "agc_buyer01", "password": "********", "usages": ["소싱"]}
  ```
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `platform` | ✅ | 1~100자 |
  | `login_id` | ✅ | 1~200자 |
  | `password` | ✅ | 1~500자. BE 가 암호화해서 저장 (응답·로그에 나오지 않음) |
  | `usages` | | 문자열 배열, 각 1~30자, 최대 10개, 중복은 하나로 합침. 기본 `[]` |
- Response 200: `{"status": "success", "account": CompanyAccount}`

### GET /company-accounts/{account_id}/password — 비밀번호 보기
한 건씩만 복호화해서 준다. **부를 때마다 열람 기록**(누가·언제·어느 계정·IP)이 남는다.
- 권한: `account_view`
- Response 200: `{"status": "success", "password": "..."}`
- Response 404

**CompanyAccount**
```json
{
  "id": 1, "platform": "타오바오", "login_id": "agc_buyer01", "usages": ["소싱"],
  "created_at": "2026-10-18T13:00:00+00:00", "updated_at": "2026-10-18T13:00:00+00:00"
}
```
