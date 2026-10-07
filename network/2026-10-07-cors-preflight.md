# CORS와 Preflight

## 질문
`http://localhost:5173`에서 돌아가는 프론트엔드가 API 서버에 장바구니 추가 요청을 보낸다.

```javascript
fetch('https://api.myshop.com/cart', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: 'Bearer abc123',
  },
  credentials: 'include',
  body: JSON.stringify({ itemId: 1 }),
});
```

```
Access to fetch at 'https://api.myshop.com/cart' from origin 'http://localhost:5173'
has been blocked by CORS policy: Response to preflight request doesn't pass access
control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

1. CORS란 무엇이고, 브라우저는 왜 이런 요청을 막는가? 아래 중 `http://localhost:5173`과 같은 출처는?
   - (a) `http://localhost:3000`
   - (b) `https://localhost:5173`
   - (c) `http://127.0.0.1:5173`
   - (d) `http://localhost:5173/api/cart`
2. preflight 요청이란 무엇이고, 이 요청에서는 왜 발생했는가? preflight가 발생하지 않는 "단순 요청"의 조건은?
3. 프론트엔드와 백엔드 중 어디서 고쳐야 하는가? 서버 응답에 어떤 헤더가 필요한가? (`credentials: 'include'` 고려)
4. CORS 에러가 났을 때 요청은 실제로 서버에 도착했는가? CORS로 서버를 보호할 수 있는가?
5. (보너스) 서버 코드를 건드리지 않고 개발 환경에서만 이 문제를 피하는 방법은? 왜 통하는가?

## 내 답변 (처음 생각)
1. CORS가 정확히 무엇의 약자인지는 모르겠다. 요청을 막는 이유는 보안을 보호하기 위해서다. same origin은 포트, 도메인, 스키마(cart)까지 동일해야 한다.
2. 브라우저가 본 요청을 보내기 전, 서버가 이 요청을 안전하게 받아줄 수 있는지 확인하기 위해 보내는 요청이다. 단순 요청은 GET, HEAD, POST에 header가 없어야 할 것이다.
3. 백엔드에서 고쳐야 한다. 올바른 응답 헤더를 보내주도록.
4. 보내졌을 것이다. 서버 보호가 아니고 사용자 보호다.
5. 개발 서버에 프록시를 설정하면 되지 않을까? 정확히는 모르겠다.

## 보완된 개념

### 1. 막는 건 SOP이고, CORS는 그걸 "풀어주는" 규칙이다
- **SOP(Same-Origin Policy, 동일 출처 정책)**: 브라우저는 기본적으로 다른 출처의 응답을 JS가 읽지 못하게 막는다.
- **CORS(Cross-Origin Resource Sharing, 교차 출처 리소스 공유)**: 서버가 응답 헤더로 "이 출처는 읽어도 된다"고 허락하면 SOP의 예외를 만들어 주는 규칙이다.

즉 "CORS가 막는다"는 말은 정확하지 않다. **SOP가 막고, CORS 허락이 없어서 풀리지 않은 것**이다.

**왜 막는가 (구체적으로)**
사용자가 `bank.com`에 로그인한 상태로 `evil.com`에 들어갔다고 하자. SOP가 없으면 `evil.com`의 JS가 `fetch('https://bank.com/account', { credentials: 'include' })`로 **사용자의 쿠키를 실어** 요청하고, 그 응답(계좌 정보)을 읽어서 빼돌릴 수 있다. SOP는 이걸 막는다.

**출처(origin) = scheme + host + port**
처음 답변에서 "스키마(cart)"라고 했는데, 스키마(scheme)는 `http`/`https`이고 `/cart`는 **경로(path)**다. **경로는 출처에 포함되지 않는다.**

| URL | 같은 출처? | 이유 |
|---|---|---|
| (a) `http://localhost:3000` | X | port 다름 (5173 vs 3000) |
| (b) `https://localhost:5173` | X | scheme 다름 (http vs https) |
| (c) `http://127.0.0.1:5173` | X | host 문자열이 다름 (같은 컴퓨터를 가리켜도 다른 출처) |
| (d) `http://localhost:5173/api/cart` | **O** | path만 다름 |

### 2. Preflight와 단순 요청 조건
**Preflight**는 본 요청 전에 브라우저가 자동으로 보내는 `OPTIONS` 요청이다. "이 메서드와 헤더로 보내도 되는지" 먼저 묻는다.

```http
OPTIONS /cart HTTP/1.1
Origin: http://localhost:5173
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```

**단순 요청 조건 (모두 만족해야 preflight 생략)**
"header가 없어야 한다"가 아니라, **허용된 헤더만 써야 한다.**
- 메서드: `GET`, `HEAD`, `POST` 중 하나
- 헤더: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` 등 **CORS-safelisted 헤더**만
- `Content-Type`: `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` 중 하나

**이 요청에서 preflight가 발생한 이유 (2가지)**
1. `Content-Type: application/json` (단순 요청에서 허용되지 않는 값)
2. `Authorization` 헤더 (safelisted 헤더가 아님)

> 단순 요청 조건은 HTML `<form>`이 원래 보낼 수 있던 요청과 같다. 폼으로 이미 보낼 수 있던 요청은 새로 막을 이유가 없어서 preflight를 생략한다고 이해하면 외우기 쉽다.

### 3. 백엔드에서 고친다. 필요한 헤더
**preflight(OPTIONS) 응답** (2xx 상태여야 함)
```http
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600
```

**본 요청(POST) 응답**에도 다시 필요하다.
```http
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Credentials: true
Vary: Origin
```

**`credentials: 'include'` 때문에 주의할 점**
- `Access-Control-Allow-Origin: *`(와일드카드)는 **쓸 수 없다.** 쿠키 등 자격 증명이 포함된 요청은 정확한 출처를 적어야 한다.
- `Access-Control-Allow-Credentials: true`가 있어야 브라우저가 응답을 JS에 넘겨준다.
- 허용 출처가 여러 개면 서버가 요청의 `Origin`을 허용 목록과 비교해서 그대로 돌려준다. 이때 캐시가 섞이지 않도록 `Vary: Origin`을 붙인다.
- `Access-Control-Max-Age`: preflight 결과를 캐시할 시간(초). 매 요청마다 OPTIONS가 가는 걸 줄인다.
- 다른 사이트로 쿠키를 보내려면 쿠키 자체에도 `SameSite=None; Secure`가 필요하다.

### 4. 이번 경우엔 POST가 서버에 도착하지 않았다
처음 답변(보내졌을 것이다)은 **단순 요청일 때만** 맞다.

| 경우 | 서버에 도착한 요청 | 결과 |
|---|---|---|
| 단순 요청 | 본 요청이 도착하고 **서버가 처리까지 한다** | 브라우저가 응답을 JS에 넘겨주지 않을 뿐 |
| preflight 요청 (이번 경우) | `OPTIONS`만 도착 | preflight가 실패해서 **본 `POST`는 아예 보내지 않는다** |

"서버 보호가 아니라 사용자 보호"라는 결론은 맞다.
- CORS는 **브라우저만 지키는 규칙**이다. `curl`, Postman, 다른 서버에서는 CORS와 상관없이 요청하고 응답을 읽을 수 있다.
- 단순 요청은 서버까지 도착해서 처리되므로, CORS로 **CSRF(사이트 간 요청 위조)**를 막을 수 없다. CSRF는 `SameSite` 쿠키나 CSRF 토큰으로 따로 막아야 한다.

### 5. 개발 서버 프록시가 정답이다
SOP는 **브라우저**의 규칙이다. 서버끼리 주고받는 요청에는 적용되지 않는다. 이 점을 이용한다.

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'https://api.myshop.com',
        changeOrigin: true, // Host 헤더를 target 기준으로 바꿔준다
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
});
```

```javascript
fetch('/api/cart', { method: 'POST', ... }); // 같은 출처(localhost:5173)로 보낸다
```

```
브라우저 → localhost:5173/api/cart   (같은 출처라 SOP 문제 없음)
           Vite 서버 → api.myshop.com/cart   (서버끼리 통신이라 SOP 적용 안 됨)
```

- 개발 서버에서만 동작한다. 배포 환경에서는 nginx 같은 리버스 프록시로 같은 출처로 묶거나, 서버가 CORS 헤더를 제대로 보내야 한다.

## 추가 심화 질문

### Q1. `<img>`, `<script>`는 다른 출처에서도 왜 잘 불러와지나요?
SOP는 다른 출처의 리소스를 **불러오는 것(embed)**은 허용하고, JS로 **내용을 읽는 것**을 막는다. 그래서 `<img src="다른 출처">`는 화면에 보이지만, 그 이미지를 `<canvas>`에 그린 뒤 `getImageData()`로 픽셀을 읽으려 하면 막힌다. (`crossorigin` 속성 + 서버의 CORS 헤더가 필요)

### Q2. CORS 에러인데 Network 탭에서는 200이 찍혀 있어요. 왜 그런가요?
단순 요청은 서버가 실제로 처리하고 200으로 응답했다. 다만 응답에 `Access-Control-Allow-Origin`이 없어서 **브라우저가 JS에 넘겨주지 않은 것**이다. 서버는 정상 동작했으니 서버 로그만 보면 문제를 찾을 수 없다.

## 참고
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
- [MDN: Preflight request](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request)
- [MDN: CORS-safelisted request header](https://developer.mozilla.org/en-US/docs/Glossary/CORS-safelisted_request_header)
- [Vite: server.proxy](https://vite.dev/config/server-options#server-proxy)
