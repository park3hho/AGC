# API Contract

FE와 BE가 공유하는 API 스펙. 변경 시 이 문서를 먼저 수정한다.

## 공통
- Base URL: dev `http://localhost:8000` (FE 는 `VITE_API_URL` 로 지정)
- 인증: **세션 쿠키** (`session`, HttpOnly). `POST /auth/login` 이 발급하고 BE 만 읽는다 — FE 는 저장하지 않고 `credentials: 'include'` 로 보낸다.
  - 로그인 유지(`remember: true`): 쿠키 7일, **로그인이 필요한 요청마다 7일로 다시 연장** (슬라이딩).
  - 로그인 유지 안 함(`remember: false`): 브라우저 세션 쿠키 (브라우저를 닫으면 사라짐). 서버 쪽 유효기간은 똑같이 7일 슬라이딩.
  - production: `SameSite=None; Secure` (FE·BE 도메인이 다르므로 HTTPS 필수). dev: `SameSite=Lax`.
  - 요청마다 DB 에서 사용자를 다시 읽는다 → **비활성화·역할 변경·비밀번호 변경이 다음 요청부터 바로 반영**된다 (비활성화·비밀번호 변경 시 기존 세션은 401).
- 인증 (자동화용): **API 키** — 헤더 `X-API-Key: agc_...`. 리서처 skill 같은 프로그램이 쓴다. 쿠키·세션을 만들지 않는다.
  - 키는 사용자 1명에 묶인다 (보통 리서처 전용 `member` 계정). 그 사용자가 비활성화되면 키도 바로 401.
  - **키로 쓸 수 있는 것은 `로그인` · `research` 권한 엔드포인트뿐**이다. 그 밖(`account_view`, `user_manage` 등)은 사용자 역할과 상관없이 403 — 키가 새도 회사 계정 비밀번호·사용자 관리에는 닿지 않는다.
  - 발급·목록·폐기는 API 가 아니라 **BE PC 에서 스크립트로만** 한다 (`python -m scripts.api_key`, BE `DESIGN.md` §5). 키 원문은 발급할 때 한 번만 보이고, DB 에는 해시만 남는다.
  - 키가 틀렸거나 폐기됐으면 401.
- 삭제는 **소프트 삭제**다 — DB 에서 지우지 않고 `deleted_at` 을 채운다. 삭제된 것은 목록·상세·검색에 나오지 않고(상세는 404), 코드(`PR-`·`RS-`)는 다시 쓰지 않는다. 되살리기 API 는 없다 (DB 에는 남아 있음).
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
- Response 404 (없거나 삭제됨)

### PATCH /products/{product_id} — 제품 수정
보낸 필드만 바꾼다. 코드·추가일은 바꿀 수 없다.
- 권한: `research`
- Request: `{"title_ko": "무선 이어폰"}` · `{"photo_url": "/files/..."}` (섞어도 됨)
  | 필드 | 규칙 |
  |---|---|
  | `title_cn` | 1~300자. `null`·빈 값 불가 |
  | `title_ko` | 300자 이하. `null` 이면 지움 |
  | `photo_url` | POST 와 같음. `null` 이면 지움 |
- Response 200: `{"status": "success", "product": Product}`
- Response 400: 바꿀 필드가 없음 · 404 (없거나 삭제됨)

### DELETE /products/{product_id} — 제품 삭제 (소프트)
- 권한: `research`
- 그 제품의 가격 조사도 함께 목록에서 사라진다 (조사 자체는 그대로 DB 에 남음). 삭제된 제품 코드로는 가격 조사를 등록할 수 없다 (404).
- Response 200: `{"status": "success"}`
- Response 404 (없거나 이미 삭제됨)

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
      {"option1": "검정", "option2": "L", "price_cny": "39.90", "photo_url": "/files/photos/..."},
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
  | `options[].photo_url` | | 옵션별 사진 (예: 맛별 사진). 규칙은 `photo_url` 과 같음 |
- Response 200: `{"status": "success", "price_survey": PriceSurvey}`
- Response 404: 제품 코드 없음(삭제된 제품 포함) · 409: 코드 발급 충돌 (드묾 — 다시 시도)

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
- Response 404 (없거나 삭제됨, 제품이 삭제된 경우 포함)

### PATCH /price-surveys/{survey_id} — 가격 조사 수정
보낸 필드만 바꾼다. `options` 를 보내면 **옵션 목록 전체를 그 내용으로 바꾼다** (부분 수정 아님 — 옵션 `id` 가 새로 바뀐다).
조사 코드번호가 조사일 기준이라 `survey_date` 와 `product_code` 는 바꿀 수 없다 → 잘못 넣었으면 삭제 후 다시 등록.
- 권한: `research`
- Request: `{"coupang_min_price": 18900}` · `{"options": [...]}` (섞어도 됨)
  | 필드 | 규칙 |
  |---|---|
  | `taobao_url` | POST 와 같음. `null`·빈 값 불가 |
  | `photo_url` · `coupang_min_price` · `coupang_url` · `naver_min_price` · `naver_url` | POST 와 같음. `null` 이면 지움 |
  | `options` | POST 와 같음 (1개 이상, 중복 불가) |
- Response 200: `{"status": "success", "price_survey": PriceSurvey}`
- Response 400: 바꿀 필드가 없음 · 404 (없거나 삭제됨)

### DELETE /price-surveys/{survey_id} — 가격 조사 삭제 (소프트)
- 권한: `research`
- Response 200: `{"status": "success"}`
- Response 404 (없거나 이미 삭제됨)

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
    {"id": 1, "option1": "검정", "option2": "L", "price_cny": "39.90", "photo_url": "/files/photos/..."},
    {"id": 2, "option1": "흰색", "option2": null, "price_cny": "42.00", "photo_url": null}
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

### POST /uploads/photos/from-url — 사진 주소로 저장
인터넷 사진 주소를 주면 **BE 가 직접 받아서** 업로드와 똑같이 저장한다. 외부 주소를 그대로 `photo_url` 로 쓰면 원본이 지워지거나 막혔을 때 사진이 사라지므로, 리서치 사진은 이걸로 저장해서 쓴다.
- 권한: `research`
- Request: `{"url": "https://img.alicdn.com/...jpg"}`
  | 규칙 | |
  |---|---|
  | 주소 | `http(s)://`, 2000자 이하. **사설·내부 주소(127.x, 10.x, 192.168.x, 169.254.x 등)는 거부** — 서버 내부를 들여다보는 요청 차단 |
  | 리다이렉트 | 최대 3번, 옮겨 간 주소도 같은 규칙으로 검사 |
  | 시간 | 15초 안에 다 받아야 함 |
  | 형식·크기 | `POST /uploads/photos` 와 같음 (내용으로 판단, 10MB 이하) |
- Response 200: `{"status": "success", "url": "/files/photos/2026/10/….jpg", "size": 48213, "source_url": "https://img.alicdn.com/...jpg"}`
- Response 400: 주소 형식이 틀림 · 내부 주소 · 413: 10MB 초과 · 415: 이미지가 아님 · 502: 원본 서버에서 받지 못함 (응답 오류·시간 초과)

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

---

### GET /products/{product_id}/pricing — 가격 책정 화면 데이터
제품 1개의 기준 옵션(최신 가격 조사)과 판매처별 가격 설정을 한 번에 준다.
- 권한: `research`
- 기준 조사 = 그 제품의 살아 있는 조사 중 `survey_date` 최근 → `id` 큰 순 1건. 조사가 없으면 `survey: null`, `options: []`.
- 판매처 설정이 아직 없으면 기본값으로 채워서 주고 `saved: false` (DB 에 만들지 않는다).
  기본값: `fx_rate` 200 · `fx_markup_pct` 0 · `shipping_fee` 10000 · `payment_fee_rate` 0 · `platform_fee_rate` `null` (사용자가 넣어야 함) · 판매가 전부 `null`.
- Response 200
  ```json
  {
    "status": "success",
    "product": Product,
    "survey": {"id": 3, "code": "RS-261018-0003", "survey_date": "2026-10-18"},
    "options": [{"id": 11, "option1": "복숭아", "option2": "480ml×12", "price_cny": "39.90", "photo_url": null}],
    "pricings": {"coupang": Pricing, "naver": Pricing}
  }
  ```
- Response 404 (제품이 없거나 삭제됨)

### PUT /products/{product_id}/pricing/{platform} — 판매처 가격 저장
`platform` = `coupang` | `naver`. 그 판매처 설정과 옵션 판매가를 **통째로 바꾼다** (전부 성공 또는 전부 실패). 다른 판매처 값은 건드리지 않는다.
- 권한: `research`
- Request
  ```json
  {
    "survey_id": 3,
    "fx_rate": "200", "fx_markup_pct": "0",
    "shipping_fee": 10000,
    "platform_fee_rate": "10.8", "payment_fee_rate": "0",
    "option_prices": [{"option_id": 11, "sale_price": 19900}]
  }
  ```
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `survey_id` | ✅ | 이 제품의 살아 있는 조사 (아니면 400) |
  | `fx_rate` | ✅ | 0 초과, 소수 4자리까지 (원/위안) |
  | `fx_markup_pct` | | 0~100, 소수 2자리. 기본 0 |
  | `shipping_fee` | ✅ | 0 이상 정수 (원) |
  | `platform_fee_rate` | ✅ | 0~100, 소수 2자리 (%) |
  | `payment_fee_rate` | | 0~100, 소수 2자리. 기본 0 |
  | `option_prices` | ✅ | 그 조사의 옵션만, 같은 옵션 중복 불가 (아니면 400). `sale_price` 는 1 이상 정수 (원). 빠진 옵션은 판매가 없음으로 저장 |
- `fx_date` 는 환율 API 연동 전까지 `null` 로 저장한다.
- Response 200: `{"status": "success", "pricing": Pricing}`
- Response 400 (조사·옵션이 안 맞음, 옵션 중복) · 404 (제품 없음, 알 수 없는 `platform`) · 409 (같은 판매처 첫 저장이 동시에 겹침 — 다시 시도)
- 응답 `pricing` 은 GET 과 같은 규칙으로 만든다 (옵션 순서 = 현재 기준 조사 `options`, 기준 조사가 아닌 조사로 저장했으면 판매가 전부 `null`).

**Pricing**
```json
{
  "platform": "coupang", "saved": true, "survey_id": 3,
  "fx_rate": "200.0000", "fx_date": null, "fx_markup_pct": "0.00",
  "shipping_fee": 10000, "platform_fee_rate": "10.80", "payment_fee_rate": "0.00",
  "option_prices": [{"option_id": 11, "sale_price": 19900}],
  "updated_at": "2026-10-19T01:00:00+00:00"
}
```
- 소수 값은 `price_cny` 처럼 **문자열**. `updated_at` 은 `saved: false` 면 `null`.
- `option_prices` 는 응답의 `options` 순서·개수와 같다 (판매가 없으면 `sale_price: null`).
- 저장된 설정의 `survey_id` 가 기준 조사와 다르면(새 조사가 생김) 판매가는 옛 옵션 기준이라 전부 `null` 로 주고, `survey_id` 는 저장된 값 그대로 준다 → FE 는 "새 조사 기준으로 다시 저장" 안내.
- **차익 · 마진율은 BE 가 주지 않는다** — FE 가 계산한다 (BE `DESIGN.md` §6.3 공식).

---

### GET /exchange-rate — 환율 (현재 + 최근 기록)
가격 책정 · 배송비가 같이 쓰는 위안 환율. 지금은 사람이 직접 넣는다 (환율 API 연동은 나중에).
- 권한: `research`
- 현재 환율 = 날짜(KST) 최근 기록. 기록이 없으면 기본값 200 (`source: "default"`, `date: null`).
- Response 200: `{"status": "success", "current": FxRate, "history": [FxRate]}` — `history` 는 날짜 최근 순 최대 30건 (기본값은 들어가지 않음)

### PUT /exchange-rate — 오늘 환율 직접 입력
오늘(KST) 날짜로 저장한다. 같은 날 다시 넣으면 덮어쓴다 (하루 1건). 바로 현재 환율이 된다.
- 권한: `research`
- Request: `{"cny_krw": "201.35"}` — 0 초과 10000 이하, 소수 4자리까지 (원/위안). 숫자도 받는다
- Response 200: `{"status": "success", "current": FxRate}`

**FxRate**
```json
{"cny_krw": "201.3500", "date": "2026-10-20", "source": "manual", "updated_at": "2026-10-20T03:00:00+00:00"}
```
- `source`: `manual` (직접 입력) · `default` (기록이 없어 기본값 — DB 에 없는 값). 환율 API 를 붙이면 API 이름이 추가된다.

### GET /shipping-fees — 배송비 (제품별 기준 조사 옵션)
옵션 이름의 용량으로 무게를 잡고 집운 식품 요금표로 계산한 배송비. 원화 환산은 FE 가 `fx` 로 한다.
- 권한: `research`
- Query: `page`, `size` (공통)
- 정렬: 제품 최근 추가 순. 기준 조사 = 가격 책정과 같은 규칙 (조사일 최근 → id 큰 순). 조사가 없는 제품은 `survey: null`, `options: []`.
- Response 200
  ```json
  {
    "status": "success",
    "fx": FxRate,
    "rule": {
      "service": "타오바오 집운-해운", "category": "식품",
      "first_kg": "1.0", "first_cny": "33.00", "step_kg": "0.5", "step_cny": "2.00",
      "max_kg": "30.0", "box_weight_g": 500
    },
    "items": [{"product": Product, "survey": {"id": 12, "code": "RS-261019-0006", "survey_date": "2026-10-19"}, "options": [ShippingOption]}],
    "total": 8, "page": 1, "size": 20
  }
  ```

**ShippingOption** (가격 책정 `options` 와 같은 모양)
```json
{
  "id": 32, "option1": "코코넛밀크", "option2": "245ml×24팩", "price_cny": "88.80", "photo_url": null,
  "volume_ml": 5880, "weight_g": 6380, "billable_kg": "6.5", "shipping_cny": "55.00", "over_limit": false
}
```
- 계산 규칙 (BE `core/shipping_config.py`, 2026-10-20 타오바오 집운-해운 식품 요금표 확인)
  - `volume_ml` = 옵션1 + 옵션2 이름에서 읽은 총 용량 (`245ml×24팩` → 5880, `1L×6팩` → 6000, 개수 없으면 1개)
  - `weight_g` = `volume_ml` × 1g + 박스 500g
  - `billable_kg` = 1kg 미만은 1kg, 넘는 무게는 0.5kg 단위 올림
  - `shipping_cny` = 33 + (`billable_kg` − 1) ÷ 0.5 × 2 (위안)
  - `over_limit` = `billable_kg` 가 30kg 초과 (소형 한도 — 이 요금표로 못 보냄)
- 용량을 못 읽으면 `volume_ml` · `weight_g` · `billable_kg` · `shipping_cny` 가 `null` → 옵션 이름에 `245ml×24` 처럼 적어야 계산된다.
- 부피 무게 · 관세는 반영하지 않는다.
