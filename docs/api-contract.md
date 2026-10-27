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
  | `admin` | 관리자 | `research`, `account_view`, `account_edit`, `user_manage`, `detail_review` |
  | `member` | 직원 | `research`, `account_view`, `detail_review` |

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
  | `photo_hover_url` | | 대표이미지(전환용) — 자사몰 상품 카드에 마우스를 올리면 바뀌는 두 번째 연출컷. 규칙은 `photo_url` 과 같음 (2026-10-27) |
  | `photos` | | 추가 사진(갤러리) 주소 목록 — 각 주소는 `photo_url` 과 같은 규칙, 같은 주소는 한 번, 최대 10장. 대표 사진(`photo_url`)은 넣지 않는다 (2026-10-25) |
  | `manufacturer` | | 제조사(공장 법인명), 300자 이하 (2026-10-27) |
  | `ingredients` | | 원재료, 5000자 이하. 맛마다 다르면 줄마다 `맛: 원재료` (2026-10-27) |
  | `allergens` | | 알레르기 유발 성분 (예: `우유, 대두`), 300자 이하 (2026-10-27) |
  | `shelf_life` | | 소비기한 (예: `12개월`), 100자 이하 (2026-10-27) |
  | `storage` | | 보관 방법, 300자 이하 (2026-10-27) |
  - 제품 정보 5개는 리서치 때 함께 찾아 넣는다. 상세페이지 초안(`build_template`)이 이 값으로 '제품 정보' 표를 채우고, 빈 값만 `[[...]]` 확인 표시로 남는다.
- Response 200: `{"status": "success", "product": Product}`
- Response 409: 코드 발급 충돌 (드묾 — 다시 시도)

### GET /products/next-code — 다음 제품 코드 미리보기
- 권한: 로그인
- Response 200: `{"status": "success", "code": "PR-261018-0001"}`

### GET /products — 제품 목록
- 권한: 로그인
- Query: `q` (제품 코드·중국어 제목·한국어 제목 부분 검색), `page`, `size`
- 정렬: 최근 추가 순
- Response 200: `{"status": "success", "items": [Product + "survey_count" + "latest_survey" + "priced_platforms"], "total": 12, "page": 1, "size": 20}`
  - `latest_survey`: 최근 가격 조사 요약 `{"id", "code", "survey_date", "option_count", "min_price_cny": "39.90"}` · 조사가 없으면 `null`
  - `priced_platforms`: 판매가를 저장한 판매처 키 목록 (예: `["coupang", "naver"]`). 모든 판매처가 들어 있으면 "가격 책정 완료" (2026-10-20 추가)

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
  | `photo_hover_url` | POST 와 같음. `null` 이면 지움 |
  | `photos` | POST 와 같음. 보내면 목록 전체를 바꾼다 (순서 그대로). `[]` 이면 비움 |
  | `manufacturer` · `ingredients` · `allergens` · `shelf_life` · `storage` | POST 와 같음. `null`·빈 값이면 지움 |
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
  "id": 1, "code": "PR-261018-0001", "photo_url": null, "photo_hover_url": null, "photos": [],
  "title_cn": "无线蓝牙耳机", "title_ko": "무선 블루투스 이어폰",
  "manufacturer": null, "ingredients": null, "allergens": null, "shelf_life": null, "storage": null,
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
  | `options[].barcode` | | 바코드(GTIN). 공백·하이픈을 지운 뒤 숫자 8~14자리만 (중국 상품은 보통 69로 시작하는 13자리). 모르면 비운다 — 임의 번호 금지. 쿠팡 공식 브랜드 상품 등록에 필요 (2026-10-27) |
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
- ⚠️ **`options` 를 보내면 옵션 행을 지우고 새 id 로 다시 만든다.** 그 옵션에 저장된 판매가(`option_price`)가 CASCADE 로 함께 지워지고, 마켓 등록(listing `items[].option_id`)과의 연결도 끊긴다 (2026-10-25 사고: 판매가 5개 제품 삭제 후 재계산 복구). 이름·사진만 고칠 때는 아래 옵션 단건 수정을 쓴다.

### PATCH /price-surveys/{survey_id}/options/{option_id} — 옵션 1개 제자리 수정
옵션 `id` 와 `price_cny` 는 그대로 두고 이름·사진만 바꾼다 → 저장된 판매가와 마켓 등록 연결이 유지된다. (2026-10-26)
- 권한: `research`
- Request: `{"option1": "서머블랙 포도"}` · `{"option2": "125g×6팩"}` · `{"photo_url": "/files/..."}` (보낸 필드만)
  | 필드 | 규칙 |
  |---|---|
  | `option1` | 1~200자, 비울 수 없음 |
  | `option2` | 200자 이하, `null` 이면 지움 |
  | `photo_url` | POST 의 `photo_url` 과 같음 |
  | `barcode` | POST 의 `options[].barcode` 와 같음. `null`·빈 값이면 지움 (2026-10-27) |
  | (`price_cny` 등 다른 필드) | 무시 — 가격 변경은 판매가 재계산이 필요해서 여기서 받지 않음 |
- Response 200: `{"status": "success", "price_survey": PriceSurvey}`
- Response 400 (바꿀 필드 없음) · 404 (조사 또는 그 조사의 옵션이 아님) · 409 (같은 (옵션1, 옵션2) 조합이 이미 있음)
- option2 의 용량 표기를 바꾸면 배송비 용량 계산(`utils/volume.py`, g = ml 로 읽음)도 그 문자열로 다시 계산된다.

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
    {"id": 1, "option1": "검정", "option2": "L", "price_cny": "39.90", "photo_url": "/files/photos/...", "barcode": "6901234567892"},
    {"id": 2, "option1": "흰색", "option2": null, "price_cny": "42.00", "photo_url": null, "barcode": null}
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
타오바오·쿠팡·네이버·식품안전나라 등 회사가 쓰는 외부 서비스 계정. **목록에는 비밀번호가 없다.** 삭제한 계정은 나오지 않는다.
- 권한: `account_view`
- Query: `usage` (선택 — 이 용도가 붙은 계정만, 정확히 일치)
- 정렬: 플랫폼 이름 순 → 최근 추가 순
- Response 200: `{"status": "success", "items": [CompanyAccount]}`

### POST /company-accounts — 회사 계정 추가
- 권한: `account_edit`
- Request
  ```json
  {"platform": "식품안전나라", "login_id": "agc01", "password": "********", "usages": ["인허가"], "url": "https://www.foodsafetykorea.go.kr"}
  ```
  | 필드 | 필수 | 규칙 |
  |---|---|---|
  | `platform` | ✅ | 1~100자, **자유 입력** (정해진 목록 없음. FE 제안 목록은 자동완성일 뿐) |
  | `login_id` | ✅ | 1~200자 |
  | `password` | ✅ | 1~500자. BE 가 암호화해서 저장 (응답·로그에 나오지 않음) |
  | `usages` | | 문자열 배열, 각 1~30자, 최대 10개, 중복은 하나로 합침. 기본 `[]` |
  | `memo` | | 비고 1000자 (예: 연동 휴대폰). 비우면 `null`. 비밀값은 넣지 않는다 (목록에 그대로 보인다) — 2026-10-25 |
  | `url` | | 홈페이지. http/https 만, 500자 이하. `www.…` 처럼 주소만 보내면 `https://` 를 붙여 저장. 빈 값·`null` = 없음 |
- Response 200: `{"status": "success", "account": CompanyAccount}` · 422 (주소 형식 오류 등)

### GET /company-accounts/{account_id} — 회사 계정 한 건 (수정 화면용)
- 권한: `account_view`
- Response 200: `{"status": "success", "account": CompanyAccount}` · 404 (없거나 삭제됨)

### PATCH /company-accounts/{account_id} — 회사 계정 수정
보낸 필드만 바꾼다.
- 권한: `account_edit`
- Request: POST 와 같은 필드, 전부 선택. `password` 는 **보냈을 때만** 새로 암호화해 바꾼다 (안 보내면 그대로). `url` · `memo` 에 `null`·`""` 을 보내면 지운다. `platform`·`login_id`·`usages` 에 `null` 은 422.
- Response 200: `{"status": "success", "account": CompanyAccount}` · 400 (바꿀 필드 없음) · 404

### DELETE /company-accounts/{account_id} — 회사 계정 삭제 (소프트)
목록·열람·수정에서 사라진다. 비밀번호 열람 기록을 남기려고 행은 지우지 않는다 (`deleted_at`).
- 권한: `account_edit`
- Response 200: `{"status": "success"}` · 404 (없거나 이미 삭제됨)

### GET /company-accounts/{account_id}/password — 비밀번호 보기
한 건씩만 복호화해서 준다. **부를 때마다 열람 기록**(누가·언제·어느 계정·IP)이 남는다.
- 권한: `account_view`
- Response 200: `{"status": "success", "password": "..."}`
- Response 404 (없거나 삭제됨)

**CompanyAccount**
```json
{
  "id": 1, "platform": "식품안전나라", "login_id": "agc01", "usages": ["인허가"], "memo": null,
  "url": "https://www.foodsafetykorea.go.kr",
  "created_at": "2026-10-18T13:00:00+00:00", "updated_at": "2026-10-18T13:00:00+00:00"
}
```
- `url` 은 없으면 `null`.

### API 키 (`/api-credentials`) — 2026-10-21
로그인 계정(`/company-accounts`)과 따로 둔다. 외부 API 키 묶음 — 공개값(앱 ID·업체코드 등)은 그대로 주고, **비밀값은 이름만** 준다. 값은 한 개씩 열람 API 로만 받고 열람 기록이 남는다. 권한은 회사 계정과 같다 (보기·열람 `account_view`, 추가·수정·삭제 `account_edit`).

| Method | Path | 설명 |
|---|---|---|
| GET | `/api-credentials` | 목록 (서비스 이름 순). `{"items": [ApiCredential]}` |
| POST | `/api-credentials` | 추가. `{"credential": ApiCredential}` |
| GET | `/api-credentials/{id}` | 한 건 (수정 화면용) · 404 |
| PATCH | `/api-credentials/{id}` | 수정 — 보낸 필드만. 400 (바꿀 것 없음 · 값 없는 새 비밀값) · 404 |
| DELETE | `/api-credentials/{id}` | 삭제 (소프트 — 열람 기록 보존) · 404 |
| GET | `/api-credentials/{id}/secrets/{secret_id}` | 비밀값 한 개 보기 `{"value": "..."}` — **부를 때마다 열람 기록** · 404 |

**Request (POST)**
```json
{
  "service": "쿠팡 OpenAPI", "url": "https://wing.coupang.com",
  "public_values": [{"label": "업체코드 (vendorId)", "value": "A0000000"}],
  "secrets": [{"label": "Access Key", "value": "..."}, {"label": "Secret Key", "value": "..."}],
  "issued_on": "2026-09-21", "expires_on": "2027-03-20",
  "reauth_from": null, "reauth_to": null,
  "allowed_ips": ["180.231.169.53"], "memo": "..."
}
```
| 필드 | 필수 | 규칙 |
|---|---|---|
| `service` | ✅ | 1~100자, 자유 입력 |
| `secrets` | ✅ | 1~10개, `label` 1~50자 겹치지 않게, `value` 1~2000자 (BE 가 암호화) |
| `public_values` | | 0~10개, `label`·`value` 모두 필수, `label` 겹치지 않게 |
| `url` | | 회사 계정 `url` 과 같은 규칙 |
| `issued_on` · `expires_on` | | 날짜. 발급일 > 만료일이면 422 |
| `reauth_from` · `reauth_to` | | 재인증 기간 (예: 네이버 커머스API 앱 인증). 시작 > 끝이면 422 |
| `allowed_ips` | | IP 또는 CIDR, 최대 20개 |
| `memo` | | 2000자 이하 |

- **PATCH 의 `secrets`**: 보내면 그 목록으로 통째로 바꾼다. `value` 를 비운(`null`·`""`) 항목은 **같은 `label` 의 기존 값을 그대로** 쓴다 (열람 기록도 이어짐). 기존에 없던 `label` 인데 값이 없으면 400. 목록에서 빠진 비밀값은 지워진다.

**ApiCredential**
```json
{
  "id": 2, "service": "쿠팡 OpenAPI", "url": "https://wing.coupang.com",
  "public_values": [{"label": "업체코드 (vendorId)", "value": "A0000000"}],
  "secrets": [{"id": 3, "label": "Access Key"}, {"id": 4, "label": "Secret Key"}],
  "issued_on": "2026-09-21", "expires_on": "2027-03-20", "reauth_from": null, "reauth_to": null,
  "allowed_ips": ["180.231.169.53"], "memo": "...",
  "created_at": "...", "updated_at": "..."
}
```
- 만료·재인증 알림(D-30 주의, D-14 위험)은 FE `utils/apiKey.keyStatuses` 가 계산한다.

### 회사 정보 › 기본 정보 (`/company-info`) — 2026-10-23
회사 정보 = 기본 정보 · 로그인 계정(`/company-accounts`) · API 키(`/api-credentials`). **기본 정보는 로그인도 비밀값도 아닌 회사 정보 전부** — 사업자등록번호, 인허가·신고 번호, 계좌, 연락처, 서류. 값은 숨기지 않고(열람 기록 없음) 그대로 준다. 만료일이 있어도 API 키가 아니다 (개념으로 나눈다: 비밀값·재인증·허용 IP 가 있으면 API 키). 권한: 보기·파일 받기 `account_view`, 추가·수정·삭제·파일 올리기 `account_edit`.

| Method | Path | 설명 |
|---|---|---|
| GET | `/company-info` | 목록 `{"items": [CompanyInfo], "links": [Link]}` — 분류 → `sort` → id 순 |
| POST | `/company-info` | 추가 `{"item": CompanyInfo}` · 409 (같은 `link` 이미 있음) |
| GET | `/company-info/{id}` | 한 건 (수정 화면용) · 404 |
| PATCH | `/company-info/{id}` | 수정 — 보낸 필드만 · 400 (바꿀 것 없음) · 404 · 409 |
| DELETE | `/company-info/{id}` | 삭제 (소프트, 파일도 같이 숨김) · 404 |
| POST | `/company-info/{id}/files` | 서류 파일 올리기 (multipart `file`) `{"file": InfoFile}` — PDF·JPG·PNG·WEBP, 20MB, 항목당 10개 · 413 · 415 |
| GET | `/company-info/{id}/files/{file_id}` | 파일 보기 (브라우저에서 바로 열림). **`/files/...` 공개 주소가 아니다** — 로그인 쿠키 필요 · 404 |
| DELETE | `/company-info/{id}/files/{file_id}` | 파일 삭제 · 404 |

**Request (POST)**
```json
{
  "category": "인허가·신고", "name": "전자상거래업자부호", "value": "K-26-043299",
  "details": [{"label": "발급처", "value": "유니패스"}],
  "issuer": "관세청", "issued_on": null, "expires_on": "2029-10-13", "remind": true,
  "url": "https://unipass.customs.go.kr", "link": "ecommerce_code", "sort": 30, "memo": "유효 3년"
}
```
| 필드 | 필수 | 규칙 |
|---|---|---|
| `category` | ✅ | 1~30자, 자유 입력 (FE 추천: 사업자 · 인허가·신고 · 계좌 · 연락처 · 서류) |
| `name` | ✅ | 1~100자 |
| `value` | | 대표 값 (번호·계좌번호 등) 500자 이하. 서류처럼 파일만 있는 항목은 비움 |
| `details` | | 세부 값 0~20개 `{label, value}` (은행·예금주·상호 등), `label` 겹치지 않게 |
| `issuer` | | 발급기관 100자 이하 |
| `issued_on` · `expires_on` | | 날짜. 발급일 > 만료일이면 422 |
| `remind` | | 기본 `true`. 만료가 가까우면 홈 할 일에 띄울지 (납세증명서처럼 필요할 때 다시 떼는 서류는 `false`) |
| `url` | | 회사 계정 `url` 과 같은 규칙 |
| `link` | | 앱에서 이 값을 가져다 쓰는 곳. `links` 의 `key` 중 하나 또는 `null`. 항목 하나에만 붙는다 (걸려 있으면 409) |
| `sort` | | 분류 안 순서 (작을수록 위), 기본 0 |
| `memo` | | 2000자 이하 |

**CompanyInfo**
```json
{
  "id": 5, "category": "인허가·신고", "name": "전자상거래업자부호", "value": "K-26-043299",
  "details": [], "issuer": "관세청", "issued_on": null, "expires_on": "2029-10-13", "remind": true,
  "url": "https://unipass.customs.go.kr", "link": "ecommerce_code", "sort": 30, "memo": "유효 3년",
  "files": [{"id": 9, "name": "전자상거래부호 캡쳐.png", "size": 48213, "content_type": "image/png", "created_at": "..."}],
  "created_at": "...", "updated_at": "..."
}
```
**Link** — `{"key": "food_agency_reg_no", "label": "수입식품등 인터넷구매대행업 영업등록번호", "used_by": "상세페이지 S6 등록번호"}`

| `link` | 쓰는 곳 |
|---|---|
| `biz_reg_no` | 사업자등록번호 — 상품 등록(예정) |
| `mail_order_no` | 통신판매업 신고번호 — 상품 등록(예정) |
| `food_agency_reg_no` | 구매대행업 영업등록번호 — **상세페이지 S6 등록번호** (`GET /detail-pages/common` 의 `reg_no` 가 이 값을 쓴다) |
| `ecommerce_code` | 전자상거래업자부호 — 통관(예정) |

- 만료 배지는 API 키와 같은 기준(D-30 주의, D-14 위험)으로 FE `utils/companyInfo.infoStatus` 가 계산한다. 홈 할 일에는 `remind: true` 인 것만.
- 파일은 공개 폴더(`/files`)가 아닌 `PRIVATE_DIR` 에 저장한다 (주민번호·계좌가 든 서류).

### 출고지·반품지 주소록 (`/shipping-addresses`) — 2026-10-22
실제 장소 1곳 = 1행. 마켓(네이버·쿠팡) 주소록에서 받은 번호를 `market_codes` 에 묶는다. **상품 등록은 기본 출고지·기본 반품지의 마켓 번호를 쓴다.** 구매대행은 출고지가 해외(배대지)여야 한다 — 쿠팡 '구매대행' 배송, 네이버 해외직구 표시 · 개인통관고유부호 입력창 모두 해외 출고지 기준. 권한: 보기 `research`, 추가·수정·삭제 `account_edit`.

| Method | Path | 설명 |
|---|---|---|
| GET | `/shipping-addresses` | 목록 (기본 출고지 → 기본 반품지 → 나머지). `{"items": [ShippingAddress], "couriers": [Courier]}` |
| POST | `/shipping-addresses` | 추가. `{"address": ShippingAddress}` |
| GET | `/shipping-addresses/{id}` | 한 건 · 404 |
| PATCH | `/shipping-addresses/{id}` | 수정 — 보낸 필드만. 합친 결과를 다시 검사 (422). 400 바꿀 것 없음 · 404 |
| DELETE | `/shipping-addresses/{id}` | 삭제 (소프트, 기본값도 풀림) · 404 |

**ShippingAddress**
```json
{
  "id": 3, "name": "집운 웨이하이 배대지", "country": "CN", "overseas": true,
  "zip_code": "264205", "address": "山东省 威海市 …", "address_detail": "韩国海运集运", "phone": "+86…",
  "uses": ["outbound"], "defaults": ["outbound"],
  "market_codes": [{"market": "쿠팡", "use": "outbound", "code": "25556899", "name": "타오바오 공식 해운 집운 웨이하이"}],
  "route": {
    "forwarder": "차이냐오(菜鸟)", "transport": "sea", "port": null, "courier": "CJGLS",
    "tracking_when": "웨이하이 출고·선적 후", "customs_ref": "H B/L — 집운 物流 상세", "confirmed": false, "note": null,
    "transport_label": "해운", "courier_name": "CJ대한통운", "courier_codes": {"쿠팡": "CJGLS", "네이버": "CJGLS"}
  },
  "memo": null, "created_at": "...", "updated_at": "..."
}
```
**Courier** `{"key": "LOTTE", "name": "롯데택배", "codes": {"쿠팡": "HYUNDAI", "네이버": "HYUNDAI"}}` — BE `core/shipping_config.COURIERS` 가 원천.
| 필드 | 필수 | 규칙 |
|---|---|---|
| `name` · `address` | ✅ | 100자 · 300자 |
| `country` | | 2글자 국가 코드 (기본 `KR`). `KR` 이 아니면 `overseas: true` |
| `uses` | ✅ | `outbound`(출고지) · `return`(반품지) 1개 이상 |
| `defaults` | | `uses` 의 부분집합. **용도마다 기본은 1곳** — 저장하면 다른 주소의 같은 기본값이 풀린다 |
| `market_codes` | | 최대 20개. `market` 자유 입력, `use` 는 `uses` 안에서, 같은 `market`+`use` 중복 불가 |
| `zip_code` · `address_detail` · `phone` · `memo` | | 비우면 `null` |
| `route` | | 배송 경로 (해외 출고지 → 해운/항공 → 통관 → 국내 택배). `null` = 없음/지움. 아래 표 |

**route** (2026-10-24) — 마켓 발송처리(송장 업로드)는 **국내 택배사 + 국내 송장번호**. 주문 발송처리는 `route.courier_codes[마켓]` 을 택배사 코드로 쓴다.
| 필드 | 규칙 |
|---|---|
| `forwarder` | 국제 물류사 (집운 업체) 100자 |
| `transport` | `sea`(해운, 기본) · `air`(항공) |
| `port` | 입항 항구 50자 |
| `courier` | 국내 택배사 — `couriers[].key` 중 하나 (아니면 422). 응답에 `courier_name` · `courier_codes`(마켓별 택배사 코드) 가 붙는다 |
| `tracking_when` | 국내 송장번호가 나오는 시점 200자 |
| `customs_ref` | UNI-PASS 통관 조회 번호(H B/L 등)와 확인 위치 200자 |
| `confirmed` | 배대지에 확인했는지 (기본 false — FE 점검에서 경고) |
| `note` | 2000자 |
- 응답 전용: `transport_label` · `courier_name` · `courier_codes` (요청에 보내도 무시).
- PATCH 로 `uses` 를 줄이면(그리고 `defaults` · `market_codes` 를 안 보내면) 빠진 용도의 기본값·마켓 번호도 같이 빠진다.
- **FE 는 주소 하나만 쓴다** (구매대행은 출고지·반품지 모두 배대지): 기본 출고지 주소를 골라 `uses`·`defaults` 를 둘 다로 저장한다. BE 는 여러 주소를 받을 수 있게 그대로 둔다.
- 점검(해외 주소인지, 마켓별 출고지·반품지 번호가 있는지)은 FE `utils/shippingAddress.addressChecks`.
- 마켓 주소록 직접 변경: 네이버 커머스API 는 주소록 조회만 있고 등록·수정·삭제 API 가 없다 (스마트스토어센터에서만). 쿠팡은 출고지·반품지 생성·수정 API 가 있다 (반품지는 굿스플로 택배사 계약 정보 필수).

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

- **묶음 판매 (2026-10-26)** — 판매 옵션 = 조사 옵션 × 묶음 수(`bundle_qty`). 묶음 1 = 조사 옵션 그대로 (`option_prices`, 지금까지와 같음).
  - 응답 `bundles`: 묶음 판매 옵션 = **10kg 제안**(`suggested`) + 어느 판매처든 판매가가 저장된 묶음(`saved`). 각 줄은 옵션 모양(`price_cny` · `volume_ml` · `weight_g` · `billable_kg` · `shipping_cny` · `over_limit` 모두 **묶음 전체 기준**, 박스 1개) +
    `key`("66x2") · `option_id` · `bundle_qty` · `market_option1`/`market_option2`(마켓에 올릴 이름, 개수 N배 "330ml×48캔") · `pack_note`("24캔×2") · `total_count`(48) · `over_duty_free`(상품가 > $150, 위안→달러 7.1 근사) · `sku`(`PR-…-66x2`).
  - 제안 규칙 (`core/shipping_config` 묶음 판매): 제품에서 가장 무거운 옵션(같은 용량이면 맛별 전부)도 `BUNDLE_TARGET_KG`(10kg) 미만이면, 10kg 이상이 되는 가장 작은 N (2~`BUNDLE_MAX_QTY` 10). 30kg 초과 · 면세 초과면 제안하지 않는다.
  - `Pricing.bundle_prices`: 묶음 2 이상 판매가 `[{option_id, bundle_qty, sale_price, listing_group, is_base}]`. `option_prices` 의 각 줄에도 `bundle_qty: 1` 이 붙는다.
  - **채택** (`product_bundle`, migration 26): 묶음을 판매 옵션으로 넣는 결정은 판매가와 따로 남긴다 (판매가는 사람이 정함). 응답 `bundles[].adopted`.
    `POST /products/{id}/pricing/bundles` `{option_id, bundle_qty(2~10)}` — 최신 조사 옵션만, 30kg 초과면 400. 다시 넣으면 같은 줄을 살림.
    `DELETE /products/{id}/pricing/bundles/{option_id}/{bundle_qty}` — 채택만 풂 (저장된 판매가는 가격 책정 저장으로 지움). 제품 목록 `bundle_unpriced` = 채택한 묶음 중 판매가 빈 칸 수 (판매처 × 묶음).
    2026-10-26 가격 책정 끝난 제품의 10kg 제안을 일괄 채택 (`source=rule`, 105개 제품 · 215묶음, 판매가 없음).
  - 마켓 판매자 코드(쿠팡 `externalVendorSku`) = `{제품 코드}-{옵션 id}` · 묶음이면 `{제품 코드}-{옵션 id}x{N}`. 쿠팡에서 불러오기와 주문 → 타오바오 구매 연결이 이 코드를 읽는다.
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
    "option_prices": [{"option_id": 11, "sale_price": 19900, "listing_group": null, "is_base": false}]
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
  | `option_prices` | ✅ | 그 조사의 옵션만, 같은 (옵션, `bundle_qty`) 중복 불가 (아니면 400). `bundle_qty` 1~10, 기본 1 (묶음 판매). `sale_price` 는 1 이상 정수 (원). 빠진 옵션은 판매가 없음으로 저장 |
  | `option_prices[].listing_group` | | (네이버) 이 옵션을 넣을 상품 번호 1~50 — 사람이 직접 옮긴 것만. `null`(기본) = 자동 (FE `naverListingPlan`: 한 상품에 다 들어가면 1개, 아니면 용량·입수별 → 가격으로 나눔) |
  | `option_prices[].is_base` | | (네이버) 그 상품의 대표 옵션(추가금 0원)으로 직접 지정. 기본 `false` = 자동 (한도 안에 드는 가장 싼 옵션) |
  | `base_option_id` | | **예전 방식 (2026-10-20 부터 안 씀 — 옵션별 `is_base`)** 대표 옵션 (네이버 추가금 0원). `option_prices` 에 판매가가 있는 옵션이어야 함 (아니면 400). `null`(기본) = 자동 — FE 가 추가금 한도를 만족하는 가장 싼 옵션을 고른다 |
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
  "option_prices": [{"option_id": 11, "sale_price": 19900, "listing_group": null, "is_base": false}],
  "base_option_id": null,
  "updated_at": "2026-10-19T01:00:00+00:00"
}
```
- 소수 값은 `price_cny` 처럼 **문자열**. `updated_at` 은 `saved: false` 면 `null`.
- `base_option_id` = 저장한 대표 옵션 (없으면 자동). 기준 조사가 바뀌었으면 판매가처럼 `null`.
  네이버 옵션가 한도 (할인 전 대표가 기준, 2026-10-20 확인): 1만 원 이상 −50%~+50% · 2천~1만 원 −50%~+100% · 2천 원 미만 0~+100%. FE `NAVER_ADDON_LIMITS`.
- `option_prices` 는 응답의 `options` 순서·개수와 같다 (판매가 없으면 `sale_price: null`).
- `option_prices[].listing_group` · `is_base` = 저장한 직접 지정값 (없으면 `null` · `false`). 네이버 상품 나눠 등록 — 기준 조사가 바뀌었으면 빈 값.
- 저장된 설정의 `survey_id` 가 기준 조사와 다르면(새 조사가 생김) 판매가는 옛 옵션 기준이라 전부 `null` 로 주고, `survey_id` 는 저장된 값 그대로 준다 → FE 는 "새 조사 기준으로 다시 저장" 안내.
- **차익 · 마진율은 BE 가 주지 않는다** — FE 가 계산한다 (BE `DESIGN.md` §6.3 공식).

---

### GET /exchange-rate — 환율 (현재 + 최근 기록 + 추이)
가격 책정 · 배송비가 같이 쓰는 위안 환율. BE 가 매일 오전 9시 자동으로 받아 저장한다 (아래 '자동 받기').
- 권한: `research`
- Query: `days` (선택, 1~400) — 주면 `trend` 도 내려준다
- 현재 환율 = 날짜(KST) 최근 기록. 기록이 없으면 기본값 200 (`source: "default"`, `date: null`).
- Response 200: `{"status": "success", "current": FxRate, "history": [FxRate], "trend": [FxPoint]}`
  - `history`: 날짜 최근 순 최대 30건 (하루 1건, 기본값은 들어가지 않음)
  - `trend` (`days` 를 줬을 때만): 최근 `days` 일 동안 **서버가 저장한 기록**, 오래된 → 최근. 외부 과거 데이터는 넣지 않는다. 31일보다 길면 하루 마지막 값 하나씩.
  - FxPoint: `{"at": "2026-10-21T00:14:54+00:00", "cny_krw": "198.5400", "source": "hana"}` — `at` = 저장 시각

### POST /exchange-rate/refresh — 지금 받아오기
환율을 바로 받아와 오늘(KST) 값으로 저장한다. **오늘 직접 입력한 값이 있어도 받은 값으로 바꾼다.**
- 권한: `research`
- 출처: 하나은행 고시 매매기준율(네이버 금융 데이터) → 안 되면 open.er-api. BE `core/pricing_config.FX_*`
- Response 200: `{"status": "success", "current": FxRate, "history": [FxRate], "fetched": {"cny_krw": "198.4400", "source": "hana", "observed_at": "2026-10-21T08:50:32+09:00"}}`
- Response 502: 모든 출처에서 못 받음 (마지막 저장값은 그대로)

**자동 받기**: BE 가 매일 오전 9시(KST)에 같은 출처에서 받아 오늘 값으로 저장한다 (실패하면 10분 간격 6번까지 다시). 서버가 켜질 때 오늘 값이 없고 9시가 지났으면 바로 받는다. 단 그날 직접 입력(`manual`)한 값이 있으면 자동으로는 덮어쓰지 않는다.
저장할 때마다(매일 9시 자동 · 지금 받아오기 · 직접 입력) `trend` 기록이 한 줄씩 쌓인다.

### PUT /exchange-rate — 오늘 환율 직접 입력
오늘(KST) 날짜로 저장한다. 같은 날 다시 넣으면 덮어쓴다 (하루 1건). 바로 현재 환율이 된다.
- 권한: `research`
- Request: `{"cny_krw": "201.35"}` — 0 초과 10000 이하, 소수 4자리까지 (원/위안). 숫자도 받는다
- Response 200: `{"status": "success", "current": FxRate}`

**FxRate**
```json
{"cny_krw": "201.3500", "date": "2026-10-20", "source": "manual", "updated_at": "2026-10-20T03:00:00+00:00"}
```
- `source`: `hana` (하나은행 고시, 자동) · `open_er_api` (open.er-api, 자동 예비) · `manual` (직접 입력 — 그날은 자동으로 안 바뀜) · `default` (기록이 없어 기본값 — DB 에 없는 값).

### GET /shipping-fees/rule — 배송비 요금표 (무게 구간별)
배송비 화면이 쓴다. 제품과 상관없이 요금 규칙 + 무게 구간별 위안 요금 + 현재 환율만 준다. 원화 환산은 FE 가 `fx` 로 한다 (환율을 바꾸면 원화도 바뀐다).
- 권한: `research`
- `brackets`: 첫 무게(1kg)부터 소형 한도(30kg)까지 0.5kg 단위. `max_volume_ml` = 이 구간에 들어가는 최대 용량 (`billable_kg` × 1000 − 박스 500g, 1ml = 1g)
- Response 200
  ```json
  {
    "status": "success",
    "fx": FxRate,
    "rule": {"service": "타오바오 집운-해운", "category": "식품", "first_kg": "1.0", "first_cny": "33.00", "step_kg": "0.5", "step_cny": "2.00", "max_kg": "30.0", "box_weight_g": 500},
    "brackets": [
      {"billable_kg": "1.0", "max_volume_ml": 500, "shipping_cny": "33.00"},
      {"billable_kg": "1.5", "max_volume_ml": 1000, "shipping_cny": "35.00"}
    ]
  }
  ```

### GET /shipping-fees — 배송비 (제품별 기준 조사 옵션)
> 2026-10-20 부터 배송비 화면은 위 `/shipping-fees/rule` 을 쓴다. 이 API 는 남겨 두지만 FE 화면에서는 쓰지 않는다 (옵션별 배송비는 가격 책정 응답에 들어 있다).
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
  - `volume_ml` = 옵션1 + 옵션2 이름에서 읽은 총 용량 (`245ml×24팩` → 5880, `1L×6팩` → 6000, 개수 없으면 1개). g·kg 로 적힌 옵션은 1g = 1ml 로 본다 (`205g×10팩` → 2050, 2026-10-24)
  - `weight_g` = `volume_ml` × 1g + 박스 500g
  - `billable_kg` = 1kg 미만은 1kg, 넘는 무게는 0.5kg 단위 올림
  - `shipping_cny` = 33 + (`billable_kg` − 1) ÷ 0.5 × 2 (위안)
  - `over_limit` = `billable_kg` 가 30kg 초과 (소형 한도 — 이 요금표로 못 보냄)
- 용량을 못 읽으면 `volume_ml` · `weight_g` · `billable_kg` · `shipping_cny` 가 `null` → 옵션 이름에 `245ml×24` 나 `205g×10` 처럼 적어야 계산된다.
- 부피 무게 · 관세는 반영하지 않는다.
---

## 상세페이지 (디자인) · 휴먼 체크 — 2026-10-21

가격 책정이 끝난 제품의 상세페이지(S1~S6 템플릿, 폭 860px)를 만들고 **사람이 한 번 확인(휴먼 체크)해서 승인**한다. 상품 등록은 승인된 상세페이지만 받는다. 규칙: BE `DESIGN.md` §6.4.
- 상태 `status`: `draft` 초안 · `review` 검수 대기 · `approved` 승인 · `changes` 수정 필요. 목록 필터에는 `none`(상세페이지 없음)도 쓴다.
- 확인 표시: 문자열 안의 `[[...]]` — 사람이 채우거나 확정해야 하는 자리. 남아 있어도 승인은 되고 FE 가 경고만 보여준다 (이미지에 그대로 나감).
- 문구 표기: 줄바꿈 = 줄 · `*글자*` = 파란 강조(헤드라인) · `_{글자}` = 아래첨자 (예: `VB_{6}`, 2026-10-22).
- **휴먼 체크 = 사람이 미리보기를 보고 '확인' 한 번** (2026-10-22 사용자 결정, 5개 체크리스트 폐지). '확인' = `approve` (상태 이름은 그대로 `approved` 승인).

### GET /detail-pages — 상세페이지 목록
가격 책정 완료(모든 판매처 판매가 저장) 제품 + 상세페이지 상태.
- 권한: `research`
- Query: `status` (`none` · `draft` · `review` · `approved` · `changes`, 없으면 전체 — 그 밖은 400), `q` (코드·한국어·중국어 이름 검색)
- 정렬: 제품 최근 추가 순. 페이지 넘김 없음 (FE 가 나눈다).
- Response 200: `{"status": "success", "items": [{"product": Product, "page": DetailPageSummary | null}], "total": 20, "counts": {"none": 19, "draft": 0, "review": 1, "approved": 0, "changes": 0}}` — `counts` 는 필터와 상관없이 전체 기준 (홈 진행 단계 숫자)

**DetailPageSummary**
```json
{
  "status": "review", "created_at": "…(디자인 시작 = 첫 저장, 2026-10-24 추가)", "updated_at": "…", "updated_by": {"id": 3, "name": "리서처 skill"},
  "reviewed_at": null, "reviewed_by": null, "review_note": null
}
```

### GET /detail-pages/{product_id} — 상세페이지 편집 화면 데이터
- 권한: `research`
- Response 200
  ```json
  {
    "status": "success",
    "product": Product,
    "page": DetailPage | null,
    "template": DetailContent,
    "photos": [{"url": "/files/photos/…", "label": "대표"}],
    "common": DetailCommon
  }
  ```
- `page: null` = 아직 없음. `template` = 제품 이름 · 최근 가격 조사 옵션(option1 = 맛, option2 = 용량·입수) · 옵션별 사진으로 채운 처음 값 (저장 안 된 상태, 사람이 쓸 칸은 `[[...]]`). 페이지가 있어도 준다 ("템플릿으로 다시 채우기"용).
- `photos` = 고를 수 있는 사진 (대표 사진 + 최근 조사 옵션 사진, 같은 파일 한 번).
- Response 404 (제품 없음)

**DetailPage** = DetailPageSummary + `{"id", "content": DetailContent, "review_checks": [..], "marker_count": 1, "content_hash": "sha256 hex", "exports": {"naver": DetailExport|null, "coupang": DetailExport|null}, "reviews": [{"action": "submit|approve|changes|reset", "user": {"id","name"}|null, "note", "checks": [], "created_at"}]}` (`reviews` 최근 10건, 최근 순)

**DetailContent** (모르는 키는 버리고 빠진 칸은 빈 값으로 저장)
```json
{
  "hero": {"eyebrow": "", "headline": "첫 줄\n*강조* 둘째 줄", "name": "", "name_sub": "", "badges": [{"value": "0kcal", "label": "무설탕"}], "photo_url": "/files/…"},
  "points": {"title": "이런 음료예요", "lead": "", "items": [{"title": "", "desc": "", "big": "0", "big_unit": "kcal / 병", "photo_url": null}]},
  "flavors": {"title": "맛을 골라주세요", "lead": "", "items": [{"name": "라임", "name_cn": "青柠口味", "desc": "", "chips": ["상큼"], "photo_url": "/files/…"}], "notes": [""],
              "gallery": {"title": "이렇게 보내드려요", "items": [{"photo_url": "/files/…", "caption": ""}]}},
  "info": {"title": "제품 정보", "rows": [{"label": "원재료", "value": ""}], "nutrition": [{"value": "0kcal", "label": "열량"}], "note": "",
           "label_photo_url": null, "label_caption": ""},
  "howto": {"title": "이렇게 드세요", "items": [{"icon": "cold", "title": "", "desc": ""}]}
}
```
- 줄바꿈 `\n` = 줄, `*글자*` = 파란 강조 (headline). 개수 한도: badges 4 · points 4 · flavors 12 · chips 4 · notes 4 · info rows 15 · nutrition 6 · howto 4 · gallery 4. `icon`: `cold` · `run` · `bottle` · `cup` · `clock` · `sun`.
- 사진 칸 (2026-10-25, 모두 선택 — 비면 안 그림, 값은 GET 의 `photos[].url` 중 아무거나):
  - `hero.photo_url`: S1 대표 사진. 원래 비율로 860 폭, 1100px 넘는 세로 사진만 위 기준으로 자른다.
  - `points.items[].photo_url`: 있으면 S2 사진 카드. 사진은 자르지 않고(contain) 비율로 배치 — 가로는 위 전체 폭, 세로·정사각은 글 옆 (2026-10-25b). 다른 사진 칸도 모두 자르지 않는다.
  - `flavors.items[].photo_url`: 맛 카드 사진 — 그 옵션 사진이 아니어도 된다. 없으면 글만 있는 카드.
  - `flavors.gallery`: S3 끝 구성·포장 사진 0~4장 (1장 = 전체 폭 원래 비율, 2장 이상 = 2열). `caption` 선택.
  - `info.label_photo_url` · `info.label_caption`: S4 정보 표 위 '현지 라벨 원본' 사진.
- 새 페이지 템플릿: `hero.photo_url` = 제품 대표 사진, 맛 카드 = 옵션별 사진, 포인트 사진은 비워 둔다(null — 자동으로 채우지 않음, 2026-10-25b).

### PUT /detail-pages/{product_id} — 저장 (없으면 초안으로 만든다)
- 권한: `research`
- Request: `{"content": DetailContent, "submit": false}` — `submit: true` 면 저장 후 검수 요청 (초안 · 수정 필요 → 검수 대기)
- **승인된 페이지의 content 가 바뀌면 검수 대기로 돌아간다** (`reset` 기록). 내용이 같으면 상태를 바꾸지 않는다.
- Response 200: `{"status": "success", "page": DetailPage}` · 404 (제품 없음) · 409 (첫 저장이 동시에 겹침)

### POST /detail-pages/{product_id}/approve — 확인 (휴먼 체크 = 승인)
- 권한: `detail_review` — **API 키 불가 (사람만)**
- Request: `{"note": null}` — `checks` 는 예전 호환용으로 받기만 하고 무시한다 (2026-10-22)
- `draft` · `review` · `changes` 어디서든 바로 `approved`. 등록번호 미입력은 막지 않는다 (FE 경고만).
- **409: 확인 표시 `[[...]]`가 내용 · 공통 안내(등록번호 칸 제외)에 남아 있음** — 메시지에 남은 칸 수와 자리(최대 8개)를 적는다 (2026-10-26 사용자 요청. 10-22 에 풀었던 조건을 다시 건다 — 그 사이 API 일괄 승인으로 표시가 남은 채 승인된 페이지가 이미지로 나갔음). `DetailPage.markers` = `[{path, where, text}]` (예: `S4 제품 정보 · 원재료`). 이미 승인된 페이지 상태는 바꾸지 않는다.
- 400: 이미 승인됨 · 404: 상세페이지 없음
- Response 200: `{"status": "success", "page": DetailPage}` — `reviewed_by` · `reviewed_at` 가 채워진다 (`review_checks` 는 예전 기록용, 새로는 null)

### POST /detail-pages/{product_id}/exports — 상세 이미지 조각 저장 — 2026-10-22 (형식 2026-10-24: 네이버 PNG · 쿠팡 JPG)
- 권한: `detail_review` — **API 키 불가 (사람만)**
- Request: `multipart/form-data` — `platform` (`naver` | `coupang`), `content_hash` (그릴 때의 `page.content_hash`, 64자), `template` (FE 템플릿 버전, 선택), `files` (네이버는 PNG · 쿠팡은 JPG 여러 개, 위에서부터 순서대로). 판매처 하나씩 올린다.
- 검사: 형식 = 판매처 형식 (`naver` PNG · `coupang` JPEG, 파일 내용으로 판단) · 조각 폭 = 판매처 폭 (`naver` 860 · `coupang` 780) · 높이 1~3000 · 장당 `naver` 20MB · `coupang` 10MB 이하 · `coupang` 높이 합계 30,000px 이하 · 1~20장 · 합계 45MB 이하.
- 400: 승인(`approved`) 아님 · 폭/높이/개수 안 맞음 · 409: 확인 표시 `[[...]]`가 남음 (approve 와 같은 검사, 2026-10-26) · `content_hash` 가 지금 내용과 다름(그 사이 바뀜) 또는 동시 저장 겹침 · 413: 장당/합계 크기 초과 · 415: 판매처 형식 아님 · 404
- Response 200: `{"status": "success", "export": DetailExport}`

**DetailExport** = `{"platform": "naver", "version": 1, "content_hash": "…", "current": true, "template": "s1-s6@2026-10-24", "files": [{"url": "/files/detail/PR-261019-0005/v1/PR-261019-0005_naver_01.png", "width": 860, "height": 2890, "size": 512345, "sha256": "…"}], "created_at", "created_by": {"id","name"}|null}`
- `current` = 이 이미지가 지금 content + 공통 안내로 그린 것. 판매처마다 마지막 버전만 `page.exports` 에 온다. 상품 등록은 `approved` + `current` 인 것만 쓴다.

### POST /detail-pages/{product_id}/request-changes — 수정 요청
- 권한: `detail_review`
- Request: `{"note": "원재료를 라임 라벨로 다시 확인해 주세요"}` (필수 — 비면 422)
- 검수 대기 · 승인 상태에서만 (아니면 400). 상태 → `changes`.
- Response 200: `{"status": "success", "page": DetailPage}`

### GET /detail-pages/common — S6 공통 안내 · 등록번호
- 권한: `research`
- Response 200: `{"status": "success", "common": DetailCommon}`

**DetailCommon**
```json
{
  "reg_no": "", "seller": "이도식품", "notice_title": "해외구매대행 상품 안내",
  "notice": [{"title": "배송 기간", "body": "결제 후 약 7~14일 [[가안]]"}],
  "reg_no_ready": false, "marker_count": 4, "updated_at": "…"
}
```
- `reg_no_ready` = 실제 등록번호가 들어 있음 (비었거나 `0`·`-` 만이면 false → FE 가 '등록번호 미입력' 경고, 승인은 됨). `marker_count` = 공통 안내에 남은 `[[...]]` 수.

### PUT /detail-pages/common — 공통 안내 저장
- 권한: `detail_review` (모든 상세페이지 · 등록번호에 걸려 있어 사람만)
- Request: `{"reg_no": "", "seller": "", "notice_title": "", "notice": [{"title": "", "body": ""}]}` (notice 최대 10)
- **값이 바뀌면 승인된 상세페이지를 전부 검수 대기로 되돌린다.**
- Response 200: `{"status": "success", "common": DetailCommon, "reset_count": 0}`

### 마켓 등록 (`/listings`) — 2026-10-24
제품을 판매처(쿠팡·네이버)에 올린 상품 1건 = 1행. 제품 1 : N (맛별로 나눠 올리면 여러 건). 판매 › 상품 › 상품 상세의 '마켓 등록'·'진행 기록(업로드 일시)'이 쓴다.

**Listing**
```json
{
  "id": 1, "product_id": 38, "platform": "coupang", "market_product_id": "16425945667",
  "title": "북빙양 귤맛 탄산음료 330ml x 6캔", "url": "https://www.coupang.com/vp/products/9790651423?vendorItemId=96228038011",
  "status": "승인완료", "listed_at": "2026-10-24T18:05:42+09:00",
  "items": [{"option_id": 63, "name": "330ml×6캔", "sale_price": 18400, "market_item_id": "96228038011", "sku": "PR-261020-0007-63"}],
  "source": "api", "synced_at": "…", "memo": null, "created_at": "…", "updated_at": "…"
}
```
- `market_product_id` = 쿠팡 sellerProductId · 네이버 원상품번호. (`platform`, `market_product_id`) unique.
- `listed_at` = 업로드 일시 (쿠팡은 셀러 상품 `createdAt`, KST). `status` = 마켓이 준 상태 그대로 (직접 입력이면 null).
- `source`: `api` (마켓에서 불러옴) · `manual` (직접 입력).

#### GET /listings — 목록
- 권한: 로그인. Query `product_id` (없으면 전체, 삭제 안 된 제품만)
- Response 200: `{"status": "success", "items": [Listing]}` — 제품 · 판매처 · 업로드 일시 순

#### POST /listings — 직접 추가
- 권한: `listing` (API 키 불가)
- Request: `{"product_id": 38, "platform": "naver", "market_product_id": "123", "title": null, "url": null, "listed_at": "2026-10-24T18:00:00+09:00", "memo": null}` — `market_product_id` 필수(공백 제거), url 은 http(s) 만
- 같은 번호가 살아 있으면 409, 지웠던 번호면 그 행을 살려 덮어쓴다.
- Response 200: `{"status": "success", "listing": Listing}`

#### PATCH /listings/{id} — 수정
- 권한: `listing`. Request: `title` · `url` · `listed_at` · `memo` 중 보낸 칸만. 판매처·상품번호는 못 바꾼다.

#### DELETE /listings/{id} — 삭제 (소프트)
- 권한: `listing`. AGC 기록만 지운다 (마켓 상품은 그대로). 지운 번호는 불러오기가 다시 살리지 않는다.

#### POST /listings/sync/coupang — 쿠팡에서 불러오기
- 권한: `listing`. Query `dry_run=true` 면 저장하지 않고 결과만.
- 쿠팡 셀러 상품 목록(`seller-products`, 100개씩) → 상품마다 상세 → 옵션 `externalVendorSku` 가 `{제품 코드}-{옵션 id}` (예: `PR-261020-0007-63`)이면 그 제품에 붙인다. 쿠팡 등록 때 SKU 를 이 모양으로 넣는다.
- url = `https://www.coupang.com/vp/products/{productId}?vendorItemId={첫 옵션}`.
- Response 200: `{"status": "success", "total": 3, "created": 3, "updated": 0, "skipped_deleted": 0, "unmatched": [{"market_product_id": "…", "title": "…"}], "items": [...]}`
- 쿠팡 호출 실패 502.

#### 상품 등록 배송 조건 (2026-10-25) — 등록 API 를 만들 때 적용
- **모든 등록은 무료배송** (배송비는 판매가에 포함). 쿠팡 `deliveryChargeType=FREE`, `deliveryCharge=0`, `freeShipOverAmount=0`, `deliveryChargeOnReturn=0`, `remoteAreaDeliverable=N`. 네이버 `deliveryInfo.deliveryFee.deliveryFeeType=FREE`, `baseFee=0`, 제주·도서산간 추가비 없음.
- 반품·교환비는 단순변심 실비만: 쿠팡 `returnCharge` 대표가 ≤ 20,000원 5,000 · 초과 10,000. 네이버 `claimDeliveryInfo.returnDeliveryFee` 같은 금액, `exchangeDeliveryFee` 는 2배(왕복).
- 자세한 표: BE DESIGN §6.7.

### 주문 (`/orders`) — 2026-10-24
마켓(네이버·쿠팡) 주문을 받아 저장하고, 새 주문·수취인 정보가 바뀐 주문은 **바로 통관부호를 검증**한다 (UNI-PASS API028). 권한 `order` (관리자·직원, API 키 불가). 수취인 전화·주소·통관부호와 마켓 원본 응답은 암호화 저장, 응답에는 가린 값만.

**고객 주문 번호 = 두 가지** — ① **자사 주문 코드** `code` = `OR-YYMMDD-XXXX` (날짜 = 마켓 주문일 KST, 16진 순번, `utils/code_gen`). 수집해서 새 주문을 만들 때 발급하고 `OrderEvent(kind=code_issued)` 를 남긴다. ② **마켓 주문번호** `order_no` (네이버 orderId · 쿠팡 orderId) + 줄 번호 `line_no`.

**추적성** — 주문에 무엇이 일어나든 `OrderEvent` 한 줄: `actor_type`(system 폴링 · agent API 키 · user 로그인) · `via` · `kind`(collected · changed · pccc_check …) · `summary` · `data`(보낸 값·받은 값, 개인정보는 가림). 지우거나 고치지 않는다. 수집 1번은 `OrderSyncRun`(구간 · 받은/새/변경/검증 건수 · 성공 여부).

| Method | Path | 설명 |
|---|---|---|
| GET | `/orders?stage=received&market=&pccc=&q=&page=&size=` | 목록 + `pccc_counts` + 마켓별 `last_runs` + `poll_minutes`. `q` = 자사 코드(`OR-…`) · 마켓 주문번호 · 줄 번호 검색 |
| POST | `/orders/collect` | 지금 가져오기. body `{"markets": ["naver","coupang"], "since_hours": null}` — 비우면 지난 수집 끝 −10분부터(처음이면 3일). 409 = 이미 수집 중 |
| GET | `/orders/sync-runs?market=&limit=` | 수집 기록 |
| GET | `/orders/{id}` | `{"order": Order, "events": [OrderEvent], "raw_masked": {...}, "receiver_full": ReceiverFull}` (events 최신 먼저) |
| POST | `/orders/{id}/pccc-check` | 통관부호 다시 검증 (사람) → 위와 같은 응답 |

- 마켓 → 줄: 네이버 = `productOrderId` 1개 (변경 상품 주문 조회 → 상세 조회), 쿠팡 = `shipmentBoxId` 안 옵션 1개 (발주서 목록, 상태 ACCEPT·INSTRUCT).
- 검증 입력: 이름 = 수취인, 전화 = 쿠팡 `overseaShippingInfoDto.ordererPhoneNumber`(안심번호 아님) · 네이버 `shippingAddress.tel1`, 우편번호 = 수취인 5자리. 옛 6자리 우편번호·휴대폰 아님·일회용 부호(인증번호 없음)는 `missing`.
- `pccc_status`: `ok` 일치 · `mismatch` 불일치(사유 `pccc_message`) · `missing` 자동 검증 못 함 · `error` 관세청 장애 · `pending`.
- **전화번호 두 가지 (2026-10-25, 덮어쓰지 않고 따로 저장)** — ① **통관용** `customs_phone` = 구매자가 통관 정보로 넣은 번호 (쿠팡 `overseaShippingInfoDto.ordererPhoneNumber` · 네이버 `shippingAddress.tel1`). 통관부호 검증·타오바오 한국 주소에 쓴다. ② **쇼핑몰 연락처** `contact_phone` = 쿠팡 `receiver.receiverNumber` → 없으면 `safeNumber`(안심번호 050…) · 네이버 `tel1`(없으면 `tel2`). 고객 연락·국내 배송용. UNI-PASS API028 은 일치 여부(`tCnt`)만 돌려주고 전화번호를 주지 않는다 — 두 번호 모두 마켓 주문 API 에서 온다.
- `Order.receiver` (목록·상세, 가린 값) `{name, phone(통관용), contact_phone, zip, address(앞 2단어), pccc}`.
- **ReceiverFull** (상세 전용, 원본 — 타오바오 주문·배송에 씀) `{"name", "zip", "address", "customs_phone", "contact_phone", "pccc"}`. 목록 응답에는 넣지 않는다.
- 자동 수집은 `ORDER_POLL_MINUTES`(기본 0 = 꺼짐).
#### GET /orders/{id} — `sourcing` (2026-10-26)
타오바오에서 살 것. 판매자 코드 `PR-…-{옵션 id}[x{N}]` → 없으면 마켓 등록 기록(`market_listing.items`)의 마켓 옵션 id → 제품 코드만 있으면 제품까지.
`{product_id, product_code, product_title, taobao_url, option_id, option1, option2, price_cny, bundle_qty, taobao_qty(= 주문 수량 × 묶음), matched_by(sku·listing·product_code)}` · 못 찾으면 `null`.
타오바오 구매기록 추가 화면이 옵션 이름 · 개수 · 단가 · 상품 링크를 미리 채운다.
#### POST /orders/{id}/confirm — 발주확인 (2026-10-25)
- 권한: `order`. 마켓에 '주문 확인'을 보낸다 → 고객 화면 결제완료 → **상품준비중** (즉시 취소가 막히고 취소 요청 → 판매자 승인으로 바뀜).
  - 쿠팡: `PUT /v2/providers/openapi/apis/api/v4/vendors/{vendorId}/ordersheets/acknowledgement` `{"vendorId", "shipmentBoxIds": [ship_unit_no]}` — `market_status = ACCEPT` 일 때만 (아니면 409 "먼저 다시 수집").
  - 네이버: `POST /v1/pay-order/seller/product-orders/confirm` `{"productOrderIds": [ship_unit_no]}` (30건씩).
- 성공: 같은 `market + ship_unit_no` 의 주문 줄 모두 `confirmed = true` (쿠팡은 `market_status = INSTRUCT`), 줄마다 OrderEvent `market_confirmed` (data.sent · received · changes).
- 이미 확인된 주문: 마켓에 다시 보내지 않고 200 + `"already": true`.
- 실패(취소 요청됨 · 상태 다름 · 마켓 오류): 409 + 마켓이 준 이유, OrderEvent `market_confirm_failed`. 취소된 주문은 409.
- Response 200: `GET /orders/{id}` 와 같은 모양 (+ `already`).

### 타오바오 구매 · 중국 내 배송 (`/taobao-purchases`) — 2026-10-24
- **등록(POST)은 발주확인된 주문만** (2026-10-25): `confirmed = false` 면 409 "먼저 마켓에 발주확인을 보내 주세요", 취소된 주문도 409. 수정·삭제는 그대로.
고객이 마켓에서 주문 → 우리가 타오바오에서 구매 → 판매처가 **집운 창고**까지 보내는 구간. **타오바오 주문번호 하나로 식별**하고, **고객 주문(MarketOrder) 1건에 붙는 기록**이다. 기본은 고객 주문 1건 = 타오바오 주문 1건 (같은 고객 주문에 여러 개 붙는 것은 막지 않음). 권한 `order`. 집운 쪽 매칭(입고 번호 등)·국제 발송은 범위 밖 — 나중에 따로.

**추적성** — 추가·수정·삭제마다 그 고객 주문의 `OrderEvent` 한 줄: `kind` = `purchase_created` · `purchase_changed` · `purchase_deleted`, `data.changes` = `{칸: {"from": 이전값, "to": 새값}}` (개인정보 없음). 값이 그대로면 기록하지 않는다.

| Method | Path | 설명 |
|---|---|---|
| GET | `/taobao-purchases?order_id=&cn_status=&q=&limit=` | 목록 (최근 먼저) `{"items": [Purchase], "cn_status": {키: 이름}}` — `q` = 타오바오 주문번호 검색 |
| POST | `/taobao-purchases` | 추가 `{"purchase": Purchase}` · 404 (없는 고객 주문) · 409 (같은 타오바오 주문번호가 이미 있음) |
| GET | `/taobao-purchases/{id}` | 한 건 · 404 |
| PATCH | `/taobao-purchases/{id}` | 수정 — 보낸 칸만 (붙은 고객 주문은 못 바꿈, 잘못 붙였으면 삭제 후 다시) · 400 · 404 · 409 |
| DELETE | `/taobao-purchases/{id}` | 삭제 (소프트, 기록은 남음) · 404 |

**Request (POST)** — `order_id` · `taobao_order_no` 만 필수, 나머지는 선택
```json
{
  "order_id": 2, "taobao_order_no": "4123456789012345678",
  "shop_name": "元气森林旗舰店", "item_url": "https://detail.tmall.com/item.htm?id=...",
  "items": [{"name": "라임 600ml×15", "qty": 1, "price_cny": "39.90"}],
  "paid_cny": "35.50", "purchased_at": "2026-10-24T18:00:00+09:00",
  "cn_status": "ordered", "shipped_at": null, "cn_courier": null, "cn_tracking_no": null, "arrived_at": null,
  "memo": null
}
```
| 필드 | 필수 | 규칙 |
|---|---|---|
| `order_id` | ✅ | 고객 주문 id |
| `taobao_order_no` | ✅ | 40자, 공백 제거. 살아 있는 구매끼리 겹치면 409 |
| `items` | | 0~30줄 `{name, qty 1~9999, price_cny}` — 옵션(맛·용량)·수량·단가 |
| `paid_cny` | | 실제 결제 위안 (할인·배송비 반영 합계) |
| `cn_status` | | `ordered` 주문함 · `shipped` 판매처 발송 · `arrived` 집운 창고 입고 · `problem` 문제 · `cancelled` 취소·환불. **`shipped`·`arrived` 로 바꿀 때 시각이 비어 있으면 지금 시각** |
| `cn_courier` · `cn_tracking_no` | | 중국 택배사 · 운송장 (운송장은 공백 제거) |

**Purchase** — 위 칸 + `id` · `order` `{id, code, market, order_no, line_no, product_name, option_name, quantity}` · `purchased_by` `{id, name}`(등록한 사람) · `status_source` (`manual` · 나중에 폴링이 채우면 `poll`) · `status_checked_at` · `created_at` · `updated_at`.

### 집운 기록 (`/jiyun-shipments`) — 2026-10-25
집운(배대지)에 배송 요청하고 배송비를 낸 기록 — 집운 번호(배송 요청 번호) 1개 = 1건, 고객 주문(MarketOrder) 1건에 붙는다 (여러 건 가능). 권한 `order`. 바꿀 때마다 그 고객 주문의 추적 기록 `jiyun_created` · `jiyun_changed` · `jiyun_deleted` (data: `jiyun_id` · `step: 국제 발송` · `refs`(집운 번호·H B/L·국내 택배·송장) · `values`/`changes` 이전값→새값).

| Method | Path | 설명 |
|---|---|---|
| GET | `/jiyun-shipments?order_id=&status=&q=&limit=` | 목록 (최신 먼저). `q` = 집운 번호 |
| POST | `/jiyun-shipments` | 추가 — `order_id` · `jiyun_no` 필수. 같은 집운 번호가 살아 있으면 409. 취소된 주문 409 |
| GET · PATCH · DELETE | `/jiyun-shipments/{id}` | 조회 · 보낸 필드만 수정 · 소프트 삭제 (기록은 남음) |

칸: `status`(requested 배송 요청 · dispatched 출고 · problem · cancelled — 출고로 바꾸고 출고 시각이 비면 지금) · `packages`(묶인 중국 운송장) · `weight_kg` · `fee_cny` · `transport`(sea·air) · `requested_at` · 실제 결제 `pay_method`·`pay_currency`·`pay_amount`·`charged_krw`·`charged_status`(none 미확인·approved·settled)·`charged_at`·`charged_source` (타오바오 구매기록과 같은 규칙) · `dispatched_at` · `hbl`(통관 조회 번호) · `kr_courier`(`core/shipping_config.COURIERS` 키) · `kr_tracking_no`(마켓 발송처리용) · `memo`.
