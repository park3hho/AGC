---
name: researcher
description: 타오바오에서 팔 음료를 찾아 AGC 앱에 제품·사진·가격 조사를 입력하는 리서처 작업. "제품 N개 찾아줘", "리서치", "가격 조사", "타오바오에서 수집" 같은 요청에 쓴다. 제품 검색 → 사진 검색 → 제품 입력 → 가격 입력을 AGC API 로 처리한다.
---

# 리서처 AGC

사람이 앱(FE)에서 하는 일과 똑같이 한다: **제품을 찾아 제품을 추가하고**, 그 제품을 골라 **가격 조사를 입력**한다.
입력은 모두 AGC API 로 한다. API 스펙 원문은 `docs/api-contract.md` — 이 문서와 다르면 그쪽이 맞다.

---

## 0. 먼저: 무엇을 채워야 하나 (API 가 받는 모양)

검색하기 전에 아래 두 모양을 머리에 넣고, **이 칸들을 채우는 정보만** 모은다. 필수(✅) 칸을 못 채우면 그 상품은 버리고 다음 후보로 간다.

**제품 — `POST /products`**
```json
{
  "title_cn": "✅ 타오바오 상품의 중국어 제품명 (브랜드 + 제품명 + 맛/용량은 빼고 핵심만)",
  "title_ko": "한국어 판매용 이름 (예: 동방수엽 무가당 차)",
  "photo_url": "대표 사진 — §2 에서 from-url 로 저장한 /files/... 주소"
}
```
`code`(PR-YYMMDD-XXXX)·추가일은 BE 가 붙인다.

**가격 조사 — `POST /price-surveys`**
```json
{
  "product_code": "✅ 위에서 받은 PR-... 코드",
  "survey_date": "✅ 오늘 (KST, YYYY-MM-DD)",
  "taobao_url": "✅ 구매할 타오바오 상품 링크 (공식 판매몰 우선)",
  "photo_url": "제품 대표 사진 (보통 제품의 photo_url 그대로)",
  "options": [
    {"option1": "✅ 맛 이름 (한국어, 예: 재스민차)", "option2": "용량·입수 (예: 500ml×15병)", "price_cny": "✅ 39.90", "photo_url": "그 맛 사진 /files/..."}
  ],
  "coupang_min_price": 19900, "coupang_url": "https://...",
  "naver_min_price": null, "naver_url": null
}
```
- `options` 는 맛(option1) × 용량·입수(option2) 조합마다 한 줄. 같은 조합 두 번 금지.
- `price_cny` 는 그 옵션의 **타오바오 판매가(위안)**. 할인 쿠폰가가 아니라 페이지에 보이는 현재가.
- 쿠팡·네이버 칸은 없으면 비운다 (null).

---

## 1. 제품 검색

규칙:
- **검색·조사는 타오바오에서만** 한다 (s.taobao.com). 국내 비교만 예외 — 쿠팡·네이버쇼핑에서만, 없어도 됨.
- 구매 링크는 **공식 판매몰(旗舰店·官方旗舰店) 위주**. 공식몰이 없을 때만 판매량 많은 일반 판매자.
- **제조사를 확인**한다 (상품 상세의 品牌 / 生产商). 제조사를 모르면 `title_cn` 에 브랜드를 넣을 수 없으니 버린다.
- **음료 위주.**
- **신선식품 불가**: 과일, 유제품(우유·요구르트 등 냉장). **상온 장기보관 유제품(멸균우유 등)은 가능.**
- 주류 불가. 건강기능식품·약재 성분 음료(凉茶 등 약초 들어간 것)는 피한다 — 국내에서 의약품 성분으로 막힐 수 있다.
- 이미 앱에 있는 제품인지 먼저 본다: `GET /products?q=<브랜드나 제품명>`. 있으면 새로 만들지 말고 그 코드로 가격 조사만 추가한다.

타오바오 화면은 로그인이 필요하다. **브라우저 창(built-in browser)에서 사용자가 로그인해 둔 상태로만** 본다. 로그인 화면이 나오면 멈추고 사용자에게 로그인해 달라고 한다 — 비밀번호를 대신 넣지 않는다.

### 1-1. 캡챠·접속 차단 대응 (계정 보호)

캡챠를 **무시하고 다른 페이지로 넘어가지 않는다.** 캡챠가 떠 있는데 계속 페이지를 열면 그게 가장 봇처럼 보이고 계정 제한으로 이어진다. 캡챠가 떠 있으면 상세 이미지·参数信息 도 가려져서 읽은 값도 틀린다.

**확인 시점**: 검색, 다음 페이지, 상품 클릭(새 탭 포함) 등 페이지가 바뀔 때마다, 값을 읽기 **전에** 아래 `taobaoBlock(page)` 로 확인한다.

```js
async function taobaoBlock(page) {
  if (/login\.(taobao|tmall)\.com/.test(page.url())) return { type: 'login' };
  for (const f of page.frames()) {
    const h = await f.locator('[id^="nc_"][id$="_n1z"], .btn_slide').first().boundingBox().catch(() => null);
    if (h) return { type: 'slider', frame: f, handle: h };
    if (/_____tmd_____|punish/.test(f.url())) return { type: 'challenge', frame: f };
  }
  const t = await page.evaluate(() => (document.body?.innerText || '').slice(0, 4000));
  const m = t.match(/访问异常[^\n]*|请按住滑块[^\n]*|拖动到最右边[^\n]*|验证码[^\n]*/);
  return m ? { type: 'text', hint: m[0] } : null;
}
```

**걸리면** (`null` 이 아니면):
1. 그 자리에서 멈춘다. 탭을 닫거나, 다른 주소로 이동하거나, 새 탭을 열지 않는다.
2. **슬라이더**(请按住滑块，拖动到最右边): `captcha-solver` 스킬의 `captcha.drag` 로 한 번 푼다. 손잡이 가운데 → 트랙 오른쪽 끝 너머로, 단계는 30~50 사이 무작위, y 는 ±2px 흔든다.
   ```js
   const b = await taobaoBlock(page);
   const track = await b.frame.locator('[id^="nc_"][id$="__scale_text"], .nc_scale').first().boundingBox();
   const y = b.handle.y + b.handle.height / 2;
   await captcha.drag(page, { x: b.handle.x + b.handle.width / 2, y },
     { x: track.x + track.width + 8, y: y + (Math.random() * 4 - 2) }, { steps: 30 + Math.floor(Math.random() * 20) });
   ```
   3~5초 뒤 `taobaoBlock` 으로 다시 확인. "出错了 / 点击刷新" 이 나오면 그 글자를 눌러 새로 고친 뒤 **한 번만** 더 시도한다 (최대 2번).
3. **2번 실패하거나, 그림 고르기·글자 입력 같은 다른 캡챠**면 더 시도하지 않고 그날 타오바오 조사를 **중단**한다. 사용자는 캡챠를 직접 풀지 않으니 풀어 달라고 하거나 기다리지 않는다. 여기까지 입력한 결과와 막힌 화면(주소·캡챠 종류)을 보고하고 끝낸다. 로그인 화면은 위 규칙대로 사용자에게 로그인을 요청한다.
4. **통과한 뒤**: 1~2분 쉬고, 다음 10번의 동작은 간격을 두 배(10초 + 무작위)로 늘린다.
5. 한 번 작업에서 캡챠가 **3번** 나오면 그날 타오바오 조사는 중단하고 보고한다.

**예방**:
- 상품 상세는 주소로 바로 열지 않는다(访问异常 이 잘 뜬다). 검색창에 입력 → 결과 클릭으로 연다.
- 동작 사이 간격은 5초 + 무작위, 클릭 위치·누르는 시간도 무작위.
- 타오바오 탭은 한 번에 하나만. 서브 에이전트로 동시에 여러 타오바오 페이지를 열지 않는다 (같은 로그인 계정).
- 다 본 상품 탭은 닫고 다음으로 간다.

보고에 **캡챠 몇 번, 어떻게 넘겼는지**(자동 통과 / 사용자 해결 / 중단)를 적는다.

## 2. 사진 검색

규칙:
- **워터마크 없는 사진.** 판매자 로고·홍보 문구가 크게 박힌 사진은 쓰지 않는다.
- **수량이 정확해야 한다.** 옵션이 "15병"인데 사진에 1병만 있거나 24병이면 안 된다. **박스 사진은 상관없음** (박스가 그 수량 단위면 OK).
- **제품과 사진이 같은지 확인** — 패키지 디자인·용량 표기가 상품 페이지와 같아야 한다.
- **옵션(맛)별 사진을 최대한 모은다.** 못 찾은 맛은 `photo_url` 을 비운다 (억지로 다른 맛 사진을 넣지 않는다).

저장은 외부 주소를 그대로 쓰지 말고 **BE 에 저장**한다 (원본이 지워져도 남게):
```
POST /uploads/photos/from-url   {"url": "https://img.alicdn.com/...jpg"}
→ {"url": "/files/photos/2026/10/....jpg", ...}   ← 이 /files/... 를 photo_url 에 넣는다
```
타오바오 사진 주소가 `..._.webp` · `_400x400.jpg` 처럼 줄인 버전이면 접미사를 떼어 원본 주소로 저장한다.

## 3. 제품 입력

1. `GET /products?q=...` 로 중복 확인.
2. 없으면 `POST /products` → 응답의 `product.code` 를 기록.
3. 잘못 넣었으면 `PATCH /products/{id}` 로 고친다. 완전히 잘못된 제품이면 `DELETE /products/{id}` (소프트 삭제).

## 4. 가격 입력

1. 사람이 FE 에서 제품을 고르듯, §3 의 `product_code` 를 쓴다.
2. `POST /price-surveys` 로 조사 1건 + 옵션 전체를 한 번에 넣는다.
3. 고칠 게 있으면 `PATCH /price-surveys/{id}` (`options` 를 보내면 옵션 전체 교체). `survey_date`·제품은 못 바꾸므로 그럴 땐 `DELETE` 후 다시 등록.

---

## API 호출 방법

- 주소: `${AGC_API_URL:-http://127.0.0.1:8000}` (BE 가 이 PC 에서 돈다)
- 인증: 헤더 `X-API-Key`. 키는 환경변수 `AGC_API_KEY`, 없으면 파일 `~/.agc/api_key`.
  **키를 출력하거나 채팅·파일·커밋에 옮기지 않는다.** 명령 안에서 `$(cat ~/.agc/api_key)` 로만 읽는다.
- 키로는 제품·가격 조사·사진 API 만 된다. 다른 API 는 403 이 정상이다.
- **Windows Git Bash 에서 한글·중국어 JSON 을 `-d '...'` 로 보내면 깨진다** → JSON 을 UTF-8 파일로 쓰고 `--data-binary @파일` 로 보낸다 (파일은 scratchpad 에).

```bash
API="${AGC_API_URL:-http://127.0.0.1:8000}"
KEY="${AGC_API_KEY:-$(cat ~/.agc/api_key)}"
curl -s -H "X-API-Key: $KEY" "$API/products?q=东方树叶"
curl -s -H "X-API-Key: $KEY" -H "Content-Type: application/json; charset=utf-8" \
     --data-binary @product.json "$API/products"
```

## 끝나면 보고

제품마다 한 줄: 제품 코드 · 한국어 이름 · 제조사 · 타오바오 링크 · 옵션 수 · 가격 범위(위안) · 옵션 사진 몇 개 저장 · 국내 최저가(있으면).
버린 후보는 이유와 함께 (신선식품, 공식몰 없음, 사진 없음 등). 확인 못 한 것은 확인 못 했다고 쓴다.
