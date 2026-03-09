# PolyON PostMessage Protocol — Console ↔ Module 통신 규격

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 대상: Console/Portal 개발자, Module SDK 구현자, AI 코딩 에이전트

---

## 1. 개요

Console/Portal(호스트)과 모듈 UI(iframe) 간 통신은 `window.postMessage` API를 사용합니다.

이 문서는 메시지 형식, 수명주기, 보안 규칙을 정의합니다.

---

## 2. 메시지 형식

모든 메시지는 다음 구조를 따릅니다:

```typescript
interface PolyonMessage {
  type: string;            // 'polyon:{action}' 형식
  [key: string]: any;      // 액션별 페이로드
}
```

### 2.1 네이밍 규칙

- 접두사: `polyon:` (네임스페이스 충돌 방지)
- 호스트 → iframe: `polyon:init`, `polyon:theme-change`, `polyon:token-refresh`
- iframe → 호스트: `polyon:ready`, `polyon:navigate`, `polyon:resize`, `polyon:notify`

---

## 3. 수명주기 (Lifecycle)

```
시간 ──────────────────────────────────────────────────>

호스트                              iframe (모듈 UI)
  │                                   │
  │  iframe 생성 (src 설정)           │
  │──────────────────────────────────>│
  │                                   │
  │                            JS 로드 + SDK init
  │                                   │
  │      polyon:ready                 │
  │<──────────────────────────────────│  ①
  │                                   │
  │      polyon:init                  │
  │──────────────────────────────────>│  ②
  │  { token, user, theme, context }  │
  │                                   │
  │         ... 양방향 통신 ...       │
  │                                   │
  │      polyon:destroy               │
  │──────────────────────────────────>│  ⑤ (선택)
  │                                   │
  │  iframe 제거                      │
  │──────────────────────────────────>│
```

---

## 4. 메시지 목록

### 4.1 iframe → 호스트

#### `polyon:ready`

모듈 SDK 초기화 완료. 호스트에 `polyon:init` 전송을 요청합니다.

```typescript
{
  type: 'polyon:ready',
  version: '1.0.0'        // SDK 버전
}
```

**호스트 처리:** `polyon:init` 응답 전송.

---

#### `polyon:navigate`

호스트의 SPA 라우터에 네비게이션을 요청합니다.

```typescript
{
  type: 'polyon:navigate',
  path: '/modules/chat/overview'    // 호스트 라우트 경로
}
```

**호스트 처리:** `window.location.hash` 또는 React Router `navigate()` 호출.

---

#### `polyon:resize`

iframe 콘텐츠 높이 변경을 알립니다.

```typescript
{
  type: 'polyon:resize',
  height: 1200             // px
}
```

**호스트 처리:** iframe 요소의 `style.height` 업데이트.

---

#### `polyon:notify`

호스트에 토스트 알림 표시를 요청합니다.

```typescript
{
  type: 'polyon:notify',
  kind: 'success',         // 'info' | 'success' | 'warning' | 'error'
  title: '파일 업로드 완료',
  subtitle: 'report.pdf (2.3MB)',
  timeout: 5000            // ms, 0이면 수동 닫기. 기본 5000
}
```

**호스트 처리:** Carbon `ToastNotification` 렌더링.

---

#### `polyon:open-external`

새 탭에서 외부 URL을 엽니다.

```typescript
{
  type: 'polyon:open-external',
  url: 'https://docs.polyon.io/drive'
}
```

**호스트 처리:** `window.open(url, '_blank')`.

---

### 4.2 호스트 → iframe

#### `polyon:init`

초기화 데이터를 전달합니다. `polyon:ready` 수신 후 전송.

```typescript
{
  type: 'polyon:init',
  token: 'eyJhbGciOiJSUzI1NiIsInR5cCI6Ikp...',  // JWT access_token
  user: {
    id: 'testuser1',                  // sAMAccountName
    displayName: '홍길동',
    email: 'testuser1@cmars.com',
    isAdmin: true,                    // Console이면 true
    groups: ['Domain Users', 'IT Team']
  },
  theme: 'light',                     // 'light' | 'dark'
  context: {
    moduleId: 'drive',
    host: 'console',                  // 'console' | 'portal'
    locale: 'ko-KR',
    timezone: 'Asia/Seoul',
    domain: 'cmars.com'
  }
}
```

---

#### `polyon:theme-change`

호스트 테마 변경 시 전달합니다.

```typescript
{
  type: 'polyon:theme-change',
  theme: 'dark'            // 'light' | 'dark'
}
```

---

#### `polyon:token-refresh`

호스트가 Keycloak 토큰을 리프레시한 후 새 토큰을 전달합니다.

```typescript
{
  type: 'polyon:token-refresh',
  token: 'eyJhbGciOiJSUzI1NiIs...'   // 새 JWT
}
```

---

#### `polyon:destroy`

호스트가 iframe을 제거하기 전 정리 신호를 보냅니다 (선택적).

```typescript
{
  type: 'polyon:destroy'
}
```

**모듈 처리:** 타이머 정리, 미저장 데이터 경고 등.

---

## 5. 보안 규칙

### 5.1 Origin 검증

```typescript
// 호스트 측 — iframe 메시지 수신 시
window.addEventListener('message', (event) => {
  // 허용된 origin만 처리
  const allowedOrigins = [
    'https://console.cmars.com',
    'https://portal.cmars.com',
    window.location.origin,  // 동일 도메인 프록시
  ];
  
  if (!allowedOrigins.includes(event.origin)) return;
  if (!event.data?.type?.startsWith('polyon:')) return;
  
  // 메시지 처리
});
```

```typescript
// 모듈 SDK 측 — 호스트 메시지 수신 시
window.addEventListener('message', (event) => {
  // parent origin 검증
  if (this.allowedOrigin && event.origin !== this.allowedOrigin) return;
  if (!event.data?.type?.startsWith('polyon:')) return;
  
  // 메시지 처리
});
```

### 5.2 iframe sandbox

```html
<iframe
  sandbox="allow-scripts allow-same-origin allow-forms allow-popups"
  ...
/>
```

| 속성 | 이유 |
|------|------|
| `allow-scripts` | 모듈 JS 실행 |
| `allow-same-origin` | 동일 도메인 쿠키/스토리지 접근 (인증) |
| `allow-forms` | 폼 제출 (파일 업로드 등) |
| `allow-popups` | 새 탭 열기 (외부 링크) |

### 5.3 토큰 관리

| 규칙 | 설명 |
|------|------|
| **메모리 보관** | SDK가 token을 변수로 보관. localStorage/sessionStorage 저장 금지 |
| **자동 갱신** | 호스트가 `polyon:token-refresh`로 갱신된 토큰 전달 |
| **만료 처리** | API 401 응답 시 SDK가 호스트에 갱신 요청 |

---

## 6. 에러 처리

### 6.1 초기화 타임아웃

SDK가 `polyon:ready` 전송 후 5초(기본값) 내에 `polyon:init`을 받지 못하면:

```typescript
// 모듈 SDK
sdk.ready.catch((err) => {
  console.error('PP SDK 초기화 타임아웃 — 호스트와 연결 실패');
  // 독립 모드로 폴백 (개발/디버그용)
});
```

### 6.2 호스트 미연결 (독립 실행)

모듈 UI를 iframe이 아닌 직접 브라우저에서 열면 호스트가 없습니다.
SDK는 이를 감지하여 독립 모드로 동작합니다:

```typescript
// SDK 내부
if (window.parent === window) {
  // 독립 모드 — 개발/디버그용
  // 토큰: localStorage 또는 수동 입력
  // 테마: 기본값
  console.warn('PP Module SDK: 독립 모드 (호스트 없음)');
}
```

---

## 7. 메시지 흐름 예시

### 7.1 Drive Admin 개요 페이지 로드

```
호스트(Console)                        iframe(Drive Admin)
    │                                      │
    │  iframe src 설정                     │
    │────────────────────────────────────>  │
    │                                      │
    │                               SDK init → polyon:ready
    │  <── polyon:ready ──────────────── │
    │                                      │
    │  ── polyon:init ──────────────────> │
    │  { token, user:{isAdmin:true},       │
    │    theme:'light', context }          │
    │                                      │
    │                               sdk.api.get('/api/v1/admin/status')
    │                               → GET /api/v1/admin/status
    │                                 Authorization: Bearer {token}
    │                                      │
    │                               렌더링 완료
    │  <── polyon:resize {height:900} ─── │
    │                                      │
    │  iframe height = 900px               │
```

### 7.2 다크모드 전환

```
호스트(Console)                        iframe(Drive Admin)
    │                                      │
    │  사용자가 다크모드 토글              │
    │                                      │
    │  ── polyon:theme-change ──────────> │
    │  { theme: 'dark' }                  │
    │                                      │
    │                               sdk.theme.onChange 콜백 실행
    │                               Carbon Theme → 'g100'
    │                                      │
    │  <── polyon:resize {height:920} ─── │
```

### 7.3 크로스 모듈 네비게이션

```
호스트(Console)                        iframe(Drive Admin)
    │                                      │
    │                               사용자가 "채팅에서 공유" 클릭
    │                               sdk.navigate('/modules/chat/overview')
    │                                      │
    │  <── polyon:navigate ──────────────  │
    │  { path: '/modules/chat/overview' }  │
    │                                      │
    │  Console Router.navigate(path)       │
    │  → Chat iframe 로드                  │
```

---

## 8. 버전 호환성

| SDK 버전 | 프로토콜 버전 | 호환성 |
|---------|-------------|--------|
| 1.x | v1 | 현재 |

`polyon:ready` 메시지에 `version` 필드를 포함하여 호스트가 프로토콜 버전을 확인합니다. 향후 하위 호환을 유지하되, 주 버전 변경 시 `polyon:ready` 핸드셰이크에서 버전 협상합니다.

---

## 참조

- [Module UI Spec](module-ui-spec.md) — 하이브리드 A/C 아키텍처
- [Module SDK Spec](module-sdk-spec.md) — @polyon/module-sdk 인터페이스
- [Admin API Spec](admin-api-spec.md) — 관리자 API
