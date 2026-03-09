# PolyON Module SDK Spec — @polyon/module-sdk

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 대상: 모듈 UI 개발자, AI 코딩 에이전트

---

## 1. 개요

`@polyon/module-sdk`는 모듈 UI(iframe)와 Console/Portal(호스트) 간 통신을 위한 JavaScript SDK입니다.

모듈 UI는 iframe 안에서 렌더링되므로 Console/Portal의 인증 토큰, 테마, 네비게이션에 직접 접근할 수 없습니다. Module SDK가 **postMessage 프로토콜**을 추상화하여 이를 해결합니다.

```
Console/Portal (호스트)          iframe (모듈 UI)
  │                                │
  │                          PolyonSDK.init()
  │                                │
  │<── polyon:ready ───────────────│  SDK 초기화 완료 신호
  │                                │
  │── polyon:init ────────────────>│  token, theme, user 전달
  │                                │
  │    sdk.api.get('/api/v1/...')   │  인증된 API 호출
  │    sdk.theme.onChange(...)      │  테마 변경 구독
  │    sdk.navigate('/chat/...')   │  크로스 모듈 네비게이션
  │                                │
  │<── polyon:navigate ────────────│  호스트에 네비게이션 요청
  │<── polyon:resize ──────────────│  iframe 높이 조절
```

---

## 2. 설치

### 2.1 npm 패키지 (번들 포함)

```bash
npm install @polyon/module-sdk
```

```tsx
import { PolyonSDK } from '@polyon/module-sdk';
const sdk = PolyonSDK.init();
```

### 2.2 CDN (script 태그)

```html
<script src="https://cdn.polyon.io/module-sdk/v1/sdk.js"></script>
<script>
  const sdk = PolyonSDK.init();
</script>
```

### 2.3 셀프호스팅

Module SDK를 모듈 이미지에 포함:

```
/admin/static/polyon-sdk.js    ← SDK 파일 복사
```

```html
<script src="static/polyon-sdk.js"></script>
```

---

## 3. API Reference

### 3.1 초기화

```typescript
const sdk = PolyonSDK.init(options?: InitOptions);

interface InitOptions {
  /** 호스트 origin 제한 (보안). 생략 시 '*' */
  allowedOrigin?: string;
  /** 초기화 타임아웃 (ms). 기본값 5000 */
  timeoutMs?: number;
  /** 자동 높이 리사이즈 활성화. 기본값 true */
  autoResize?: boolean;
}
```

- `init()` 호출 시 호스트(Console/Portal)에 `polyon:ready` 메시지 전송
- 호스트가 `polyon:init` 응답으로 token, theme, user 정보 전달
- 초기화 완료 후 SDK 사용 가능

```typescript
const sdk = PolyonSDK.init();

// 초기화 완료 대기 (Promise)
await sdk.ready;
console.log('SDK 초기화 완료:', sdk.user);
```

### 3.2 인증 (auth)

```typescript
// 현재 JWT 토큰 (호스트에서 전달받은 것)
const token: string = sdk.auth.getToken();

// 현재 사용자 정보
const user: PolyonUser = sdk.auth.getUser();

interface PolyonUser {
  id: string;              // sAMAccountName
  displayName: string;     // AD displayName
  email: string;           // mail
  isAdmin: boolean;        // 관리자 여부
  groups: string[];        // AD memberOf
}

// 토큰 갱신 이벤트 구독 (호스트가 리프레시 시 자동 전달)
sdk.auth.onTokenRefresh((newToken: string) => {
  console.log('토큰 갱신됨');
});
```

### 3.3 API 호출 (api)

SDK가 Authorization 헤더를 자동 포함합니다:

```typescript
// GET 요청
const status = await sdk.api.get('/api/v1/admin/status');
// → GET http://polyon-drive:8080/api/v1/admin/status
//   Authorization: Bearer {token}

// POST 요청
const result = await sdk.api.post('/api/v1/admin/quota/testuser1', {
  limit_bytes: 10737418240
});

// PUT 요청
await sdk.api.put('/api/v1/admin/settings/default_quota_bytes', {
  value: '10737418240'
});

// DELETE 요청
await sdk.api.delete('/api/v1/trash/123');
```

**API 호출은 모듈 자체 백엔드로 직접 갑니다** (Core 프록시 경유 아님).
iframe이 이미 Ingress를 통해 모듈 서비스로 라우팅되어 있으므로,
상대 경로(`/api/v1/...`)가 모듈 백엔드로 연결됩니다.

```typescript
interface ApiClient {
  get<T>(path: string, options?: RequestOptions): Promise<T>;
  post<T>(path: string, body?: any, options?: RequestOptions): Promise<T>;
  put<T>(path: string, body?: any, options?: RequestOptions): Promise<T>;
  delete<T>(path: string, options?: RequestOptions): Promise<T>;
}

interface RequestOptions {
  headers?: Record<string, string>;
  timeout?: number;
}
```

### 3.4 테마 (theme)

```typescript
// 현재 테마
const current: 'light' | 'dark' = sdk.theme.current;

// 테마 변경 구독 (Console에서 다크모드 전환 시)
sdk.theme.onChange((theme: 'light' | 'dark') => {
  // Carbon: theme === 'dark' ? 'g100' : 'white'
  document.documentElement.setAttribute('data-carbon-theme', 
    theme === 'dark' ? 'g100' : 'white');
});
```

### 3.5 네비게이션 (navigate)

```typescript
// 호스트(Console/Portal)에 네비게이션 요청
sdk.navigate('/modules/chat/overview');
// → Console이 SPA 라우팅으로 해당 경로로 이동

// 외부 URL 열기
sdk.openExternal('https://docs.polyon.io');
// → 새 탭에서 열기
```

### 3.6 알림 (notify)

```typescript
// 호스트에 토스트 알림 요청
sdk.notify({
  kind: 'success',         // 'info' | 'success' | 'warning' | 'error'
  title: '파일 업로드 완료',
  subtitle: 'report.pdf (2.3MB)',
  timeout: 5000,           // ms, 0이면 수동 닫기
});
```

### 3.7 리사이즈 (resize)

```typescript
// 자동 리사이즈 (기본 활성화)
// SDK가 ResizeObserver로 콘텐츠 높이 감지 → 호스트에 전달 → iframe 높이 조절

// 수동 리사이즈
sdk.resize(800);  // iframe 높이를 800px로 요청

// 자동 리사이즈 비활성화
const sdk = PolyonSDK.init({ autoResize: false });
```

### 3.8 모듈 컨텍스트 (context)

```typescript
// Console이 전달하는 컨텍스트 정보
const ctx: ModuleContext = sdk.context;

interface ModuleContext {
  moduleId: string;        // 'drive'
  host: 'console' | 'portal';  // 호스트 구분
  locale: string;          // 'ko-KR'
  timezone: string;        // 'Asia/Seoul'
  domain: string;          // 'cmars.com'
}
```

---

## 4. React Hook (권장)

```tsx
// @polyon/module-sdk/react
import { usePolyon, usePolyonApi, usePolyonTheme } from '@polyon/module-sdk/react';

function Overview() {
  const { user, context } = usePolyon();
  const { theme, carbonTheme } = usePolyonTheme();  // carbonTheme: 'white' | 'g100'
  const api = usePolyonApi();

  const { data: status } = useQuery('status', () => api.get('/api/v1/admin/status'));

  return (
    <Theme theme={carbonTheme}>
      <h2>{user.displayName}님의 Drive 관리</h2>
      <p>상태: {status?.status}</p>
    </Theme>
  );
}
```

---

## 5. Admin 인증 흐름

### 5.1 Console → iframe Admin 인증

```
1. Console에서 사용자가 Keycloak OIDC 로그인 (admin realm)
2. Console이 JWT access_token 보유
3. 모듈 메뉴 클릭 → ModuleSector가 iframe 생성
4. iframe 내 Module SDK가 polyon:ready 전송
5. Console이 polyon:init으로 JWT 토큰 + admin 플래그 전달
6. Module SDK가 API 호출 시 Authorization: Bearer {token} 자동 포함
7. 모듈 백엔드가 JWT 검증 또는 X-Polyon-Admin 헤더 확인
```

### 5.2 Portal → iframe User 인증

```
1. Portal에서 사용자가 Keycloak OIDC 로그인 (polyon realm)
2. Portal이 JWT access_token 보유
3. 모듈 메뉴 클릭 → ModuleSector가 iframe 생성
4. 동일한 SDK 초기화 흐름
5. 모듈 백엔드가 JWT 또는 LDAP bind로 사용자 인증
```

---

## 6. 보안 고려사항

| 항목 | 대응 |
|------|------|
| **Origin 검증** | SDK는 `allowedOrigin` 옵션으로 호스트 origin 제한 가능 |
| **토큰 노출** | postMessage로만 전달, localStorage 저장 금지 권장 |
| **iframe sandbox** | `allow-scripts allow-same-origin allow-forms allow-popups` |
| **XSS** | 모듈 UI는 자체 격리. Console/Portal DOM에 접근 불가 |
| **CSRF** | JWT Bearer 토큰 사용 (쿠키 기반 아님) |

---

## 7. 구현 참고 — SDK 내부 동작

```typescript
// SDK 내부 (간략화)
class PolyonSDK {
  private token: string = '';
  private user: PolyonUser | null = null;
  private themeValue: 'light' | 'dark' = 'light';
  private themeListeners: Array<(t: string) => void> = [];

  static init(options?: InitOptions): PolyonSDK {
    const sdk = new PolyonSDK();
    
    // 호스트 메시지 수신
    window.addEventListener('message', (event) => {
      const { data } = event;
      
      if (data.type === 'polyon:init') {
        sdk.token = data.token;
        sdk.user = data.user;
        sdk.themeValue = data.theme;
        sdk.context = data.context;
        sdk._resolveReady();
      }
      
      if (data.type === 'polyon:theme-change') {
        sdk.themeValue = data.theme;
        sdk.themeListeners.forEach(fn => fn(data.theme));
      }
      
      if (data.type === 'polyon:token-refresh') {
        sdk.token = data.token;
        sdk.tokenListeners.forEach(fn => fn(data.token));
      }
    });

    // 호스트에 준비 완료 알림
    window.parent.postMessage({ type: 'polyon:ready' }, '*');

    // 자동 리사이즈
    if (options?.autoResize !== false) {
      const observer = new ResizeObserver(() => {
        const height = document.documentElement.scrollHeight;
        window.parent.postMessage({ type: 'polyon:resize', height }, '*');
      });
      observer.observe(document.body);
    }

    return sdk;
  }

  // API 클라이언트
  get api(): ApiClient {
    return {
      get: (path, opts) => this._fetch('GET', path, undefined, opts),
      post: (path, body, opts) => this._fetch('POST', path, body, opts),
      put: (path, body, opts) => this._fetch('PUT', path, body, opts),
      delete: (path, opts) => this._fetch('DELETE', path, undefined, opts),
    };
  }

  private async _fetch(method: string, path: string, body?: any, opts?: RequestOptions) {
    const res = await fetch(path, {
      method,
      headers: {
        'Authorization': `Bearer ${this.token}`,
        'Content-Type': 'application/json',
        ...opts?.headers,
      },
      body: body ? JSON.stringify(body) : undefined,
    });
    if (!res.ok) throw new Error(`API ${method} ${path}: ${res.status}`);
    return res.json();
  }

  // 네비게이션 → 호스트로 전달
  navigate(path: string) {
    window.parent.postMessage({ type: 'polyon:navigate', path }, '*');
  }

  // 알림 → 호스트로 전달
  notify(notification: NotifyOptions) {
    window.parent.postMessage({ type: 'polyon:notify', ...notification }, '*');
  }
}
```

---

## 참조

- [Module UI Spec](module-ui-spec.md) — 하이브리드 A/C 아키텍처
- [PostMessage Protocol](postmessage-protocol-spec.md) — 상세 메시지 프로토콜
- [Admin API Spec](admin-api-spec.md) — 관리자 API
- [Module Spec](module-spec.md) — 모듈 기본 규격
