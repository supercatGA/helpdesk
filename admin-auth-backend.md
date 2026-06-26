# 호야 헬프데스크 — 어드민 접근코드 백엔드 연동 가이드

어드민 게이트가 이제 **서버에서 접근코드를 검증**할 수 있습니다.
사이트(설정 탭 → "인증 서버 URL")에 아래 백엔드의 URL을 넣으면, 게이트가 그 서버로
`POST {code}` 를 보내고 `{ "ok": true }` 응답을 받을 때만 입장됩니다.
URL을 비워두면 기존처럼 폴백 코드(`ADMIN_PIN`, 기본 `supercat`)로 동작합니다.

> 보안 메모: 단일 HTML 파일이라 어드민 화면/데이터는 소스에 포함됩니다. 접근코드를
> 서버에서 검증하면 "코드 노출"은 막을 수 있지만, 기술적으로 소스를 뜯어보는 사용자까지
> 완전히 차단하려면 어드민 화면과 데이터를 서버 뒤로 옮기는 별도 구조가 필요합니다.
> 사내 운영용 1차 차단으로는 아래 방식으로 충분합니다.

---

## 옵션 A — Cloudflare Workers (추천, 무료 · CORS 자동 처리)

1. https://dash.cloudflare.com → **Workers & Pages** → **Create Worker**
2. 아래 코드 붙여넣기 → `ADMIN_CODE` 값을 원하는 접근코드로 변경 → **Deploy**
3. 배포된 주소(`https://xxxx.workers.dev`)를 복사
4. 사이트 어드민 → **설정 → 인증 서버 URL** 에 붙여넣고 저장

```js
const ADMIN_CODE = "여기에-원하는-접근코드";

const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type",
};

export default {
  async fetch(req) {
    if (req.method === "OPTIONS") return new Response(null, { headers: cors });
    if (req.method !== "POST")
      return new Response("POST only", { status: 405, headers: cors });
    let code = "";
    try { code = (await req.json()).code; }
    catch { try { code = JSON.parse(await req.text()).code; } catch {} }
    const ok = code === ADMIN_CODE;
    return new Response(JSON.stringify({ ok }), {
      headers: { ...cors, "Content-Type": "application/json" },
    });
  },
};
```

---

## 옵션 B — Google Apps Script (구글 계정만 있으면 무료)

1. https://script.google.com → **새 프로젝트**
2. 아래 코드 붙여넣기 → `ADMIN_CODE` 변경
3. **배포 → 새 배포 → 유형: 웹 앱** → 실행 대상: 나, 액세스 권한: **모든 사용자** → 배포
4. 발급된 **웹 앱 URL** 복사 → 사이트 **설정 → 인증 서버 URL** 에 붙여넣고 저장

```js
var ADMIN_CODE = "여기에-원하는-접근코드";

function doPost(e) {
  var code = "";
  try { code = JSON.parse(e.postData.contents).code; } catch (err) {}
  var ok = code === ADMIN_CODE;
  return ContentService
    .createTextOutput(JSON.stringify({ ok: ok }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

> 참고: 사이트는 프리플라이트(OPTIONS)를 피하려고 `Content-Type: text/plain` 으로
> 단순 요청을 보냅니다. 위 두 백엔드 모두 본문(JSON 문자열)을 파싱하도록 되어 있어
> 그대로 동작합니다.

---

## 동작 방식 요약

- 사이트 게이트 입력값 → `POST { "code": "<입력값>" }`
- 백엔드가 코드 비교 → `{ "ok": true }` 또는 `{ "ok": false }`
- `ok:true` 면 입장, 아니면 "접근코드가 올바르지 않아요", 서버 미응답이면 "인증 서버에 연결할 수 없어요"
- 코드는 더 이상 사이트 소스에 하드코딩되지 않습니다(인증 서버 URL 설정 시).

비밀번호 회전이 필요하면 백엔드의 `ADMIN_CODE` 만 바꾸면 됩니다(사이트 재배포 불필요).
