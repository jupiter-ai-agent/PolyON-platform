# PolyON Module UI Spec — 모듈 UI 통합 규격 v2

> 작성: Jupiter (AI 팀장) | 2026-03-09 v2 (하이브리드 A/C 구조)
> 대상: 모듈 개발자, Console/Portal 개발자, AI 코딩 에이전트

---

## 1. 개요

PP(PolyON Platform) 모듈의 UI는 **하이브리드 아키텍처**로 통합됩니다.

- **Foundation 페이지 (A 방식):** Console/Portal 코드에 직접 구현. Carbon Design 100% 보장
- **모듈 페이지 (C 방식):** 모듈이 자기완결적 UI 번들을 포함. Console/Portal은 iframe 섹터를 제공

이 문서는 두 방식의 구조, 메뉴 통합, iframe 렌더링, Module SDK를 정의합니다.

---

## 2. UI 아키텍처 — 하이브리드 A/C

```
┌─────────────────────────────────────────────────────────────┐
│                    Console (관리자 SPA)                      │
│                                                             │
│  ┌── Foundation (A 내장) ──┐  ┌── Module Sector (C) ──────┐ │
│  │  대시보드               │  │  ┌─────────────────────┐  │ │
│  │  사용자 관리            │  │  │  Drive Admin UI     │  │ │
│  │  시스템 정보            │  │  │  (iframe)           │  │ │
│  │  인증 설정              │  │  │  자기완결적 번들    │  │ │
│  │  (Carbon Design 직접)  │  │  └─────────────────────┘  │ │
│  └─────────────────────────┘  └───────────────────────────┘ │
│                                                             │
│  메뉴: module.yaml JSON → 동적 SideNav                      │
│  통신: @polyon/module-sdk (postMessage 프로토콜)             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Portal (사원 SPA)                         │
│                                                             │
│  ┌── Foundation (A 내장) ──┐  ┌── Module Sector (C) ──────┐ │
│  │  대시보드               │  │  ┌─────────────────────┐  │ │
│  │  프로필                 │  │  │  Drive Files UI     │  │ │
│  │  알림                   │  │  │  (iframe)           │  │ │
│  │  (Carbon Design 직접)  │  │  │  Google Drive 스타일 │  │ │
│  └─────────────────────────┘  │  └─────────────────────┘  │ │
│                               └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 Foundation vs Module 분류

| 분류 | 대상 | UI 방식 | 설명 |
|------|------|---------|------|
| **Foundation** | Core, Console 자체 페이지 | A (내장) | Carbon Design 직접 구현. Console/Portal 코드에 포함 |
| **Module** | Drive, Chat, Wiki, Mail관리 등 | C (Micro-Frontend) | 자기완결적 UI 번들. iframe으로 렌더링 |

**Foundation은 플랫폼 존재에 필수적인 요소만 해당합니다:**
- PostgreSQL, Redis, OpenSearch, RustFS, Traefik (인프라)
- Samba DC, Keycloak, Stalwart Mail (인증/메일)
- Core, Console, Portal (플랫폼 엔진)

**Drive, Chat, Wiki 등 업무 모듈은 모두 Module입니다.** Foundation이 아닙니다.

### 2.2 왜 하이브리드인가

| 방식 | Foundation | Module | 판단 |
|------|-----------|--------|------|
| A만 (전부 내장) | ✅ | ❌ Console 수정 없이 모듈 추가 불가 | 확장성 없음 |
| C만 (전부 iframe) | ❌ 플랫폼 핵심도 격리 | ✅ | 통합성 저하 |
| **A + C 하이브리드** | ✅ 깊은 통합 | ✅ 독립 확장 | **채택** |

### 2.3 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **메뉴는 동적 주입** | module.yaml의 JSON 선언으로 SideNav에 자동 등록 |
| **Foundation = 내장** | Console/Portal 코드에 Carbon Design으로 직접 구현 |
| **Module = 자기완결** | 모듈 이미지 안에 UI 번들 포함, iframe으로 렌더링 |
| **Carbon 권장, 강제 아님** | 모듈 UI에 Carbon Design 추천. 통일성 유지하되 별도 L/F 허용 |
| **의존성 자기완결** | 모듈 UI 번들은 필요한 라이브러리를 자체 포함. Console과 버전 충돌 차단 |
| **Core API 프록시** | 메타데이터 API는 Core 경유. 파일 바이너리는 WebDAV 직접 |

---

## 3. module.yaml UI 선언

### 3.1 메뉴 선언 스키마

```yaml
apiVersion: polyon.io/v1
kind: Module

metadata:
  id: drive
  name: PP Drive
  version: "0.1.0"
  category: engine
  description: "파일 스토리지 서비스"
  icon: Document           # @carbon/icons 이름 (32px variant, 메뉴 아이콘용)

spec:
  engine: drive
  # ... (기존 resources, database, ingress, ldap 등)

  # ── Console Admin UI 선언 ──
  console:
    menuGroup: services    # 메뉴 그룹 (§3.2 참조)
    # 모듈 자체 Admin UI 엔드포인트 (iframe src)
    # 생략 시 기본값: http://polyon-{id}:{port}/admin
    adminPath: /admin
    pages:
      - id: overview
        title: "개요"
        icon: Dashboard
        path: overview       # iframe URL에 해시/경로로 전달
        default: true        # 메뉴 클릭 시 기본 페이지
      - id: storage
        title: "스토리지"
        icon: DataBase
        path: storage
      - id: quota
        title: "할당량"
        icon: Meter
        path: quota
      - id: activity
        title: "활동 로그"
        icon: Activity
        path: activity
      - id: settings
        title: "설정"
        icon: Settings
        path: settings

  # ── Portal 사원 UI 선언 ──
  portal:
    menuGroup: apps
    # 모듈 자체 사원 UI 엔드포인트 (iframe src)
    # 생략 시 기본값: http://polyon-{id}:{port}/
    userPath: /
    pages:
      - id: files
        title: "드라이브"
        icon: Folder
        path: ""             # 루트
        default: true
      - id: shared
        title: "공유"
        icon: Share
        path: shared
      - id: recent
        title: "최근"
        icon: RecentlyViewed
        path: recent
      - id: trash
        title: "휴지통"
        icon: TrashCan
        path: trash
```

### 3.2 Console 메뉴 그룹

Console 좌측 SideNav 구조:

```
──────────────────────
  시스템                  ← group: system (Foundation, A 내장)
    대시보드
    사용자 관리
    시스템 정보 (모듈 설치)
──────────────────────
  서비스                  ← group: services (모듈, C iframe)
    📁 드라이브           ← module: drive
    💬 채팅               ← module: chat
    📝 위키               ← module: wiki
──────────────────────
  인프라                  ← group: infra (Foundation, A 내장)
    인증 (Keycloak)
    메일 (Stalwart)
    스토리지 (RustFS)
──────────────────────
```

| 그룹 ID | 표시 이름 | UI 방식 | 대상 |
|---------|-----------|---------|------|
| `system` | 시스템 | A (내장) | Foundation 고정 메뉴 |
| `services` | 서비스 | C (iframe) | 설치된 모듈 (동적) |
| `infra` | 인프라 | A (내장) | Foundation 고정 메뉴 |

### 3.3 Portal 메뉴 그룹

```
──────────────────────
  홈                     ← 고정 (A 내장)
    대시보드
    알림
──────────────────────
  앱                     ← group: apps (모듈, C iframe)
    📁 드라이브
    💬 채팅
    📝 위키
──────────────────────
  도구                   ← group: tools (모듈, C iframe)
    🤖 AI 어시스턴트
──────────────────────
```

### 3.4 아이콘 규격

- module.yaml의 `icon` 필드: `@carbon/icons-react` 컴포넌트 이름 (32px variant)
- emoji 절대 금지 (Console/Portal SideNav에서 Carbon 아이콘으로 렌더링)
- 예: `Document`, `Folder`, `Chat`, `Dashboard`, `Settings`

Console/Portal 아이콘 렌더링:
```tsx
import * as CarbonIcons from '@carbon/icons-react';

function ModuleIcon({ name, size = 20 }: { name: string; size?: number }) {
  const Icon = (CarbonIcons as any)[name];
  if (!Icon) return <CarbonIcons.Application size={size} />;
  return <Icon size={size} />;
}
```

---

## 4. Module Sector — iframe 렌더링

### 4.1 Console의 모듈 렌더링 흐름

```
1. 사용자가 SideNav에서 "드라이브 > 개요" 클릭
2. Console 라우터가 /modules/drive/overview 매칭
3. ModuleSector 컴포넌트가 iframe 렌더링:
   - src: "http://polyon-drive:8080/admin#/overview"
   - (Ingress 경유: "https://{domain}/modules/drive/admin#/overview")
4. iframe 내 Module SDK가 Console과 postMessage로 통신
```

### 4.2 ModuleSector 컴포넌트

Console/Portal이 제공하는 범용 iframe 렌더러:

```tsx
// console/src/components/modules/ModuleSector.tsx
import { useEffect, useRef, useState } from 'react';
import { Loading, InlineNotification } from '@carbon/react';

interface ModuleSectorProps {
  moduleId: string;         // 'drive'
  basePath: string;         // '/admin' (console) or '/' (portal)
  pagePath: string;         // 'overview', 'storage' 등
  token: string;            // JWT (postMessage로 전달)
  theme: 'light' | 'dark';
}

export default function ModuleSector({
  moduleId, basePath, pagePath, token, theme
}: ModuleSectorProps) {
  const iframeRef = useRef<HTMLIFrameElement>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  // iframe src 구성
  const src = `/modules/${moduleId}/proxy${basePath}#/${pagePath}`;

  useEffect(() => {
    const handleMessage = (event: MessageEvent) => {
      if (event.data?.type === 'polyon:ready') {
        // 모듈 SDK 초기화 완료 → 인증 토큰 + 테마 전달
        iframeRef.current?.contentWindow?.postMessage(
          { type: 'polyon:init', token, theme, moduleId },
          '*'
        );
        setLoading(false);
      }
      if (event.data?.type === 'polyon:navigate') {
        // 모듈이 크로스 네비게이션 요청
        window.location.hash = event.data.path;
      }
      if (event.data?.type === 'polyon:resize') {
        // iframe 높이 자동 조절
        if (iframeRef.current) {
          iframeRef.current.style.height = `${event.data.height}px`;
        }
      }
    };

    window.addEventListener('message', handleMessage);
    return () => window.removeEventListener('message', handleMessage);
  }, [token, theme, moduleId]);

  return (
    <div style={{ position: 'relative', width: '100%', minHeight: '400px' }}>
      {loading && <Loading withOverlay />}
      {error && (
        <InlineNotification
          kind="error"
          title="모듈 로드 실패"
          subtitle={error}
        />
      )}
      <iframe
        ref={iframeRef}
        src={src}
        style={{
          width: '100%',
          height: '100%',
          minHeight: 'calc(100vh - 120px)',
          border: 'none',
        }}
        sandbox="allow-scripts allow-same-origin allow-forms allow-popups"
        onLoad={() => setLoading(false)}
        onError={() => setError('모듈 UI를 로드할 수 없습니다')}
      />
    </div>
  );
}
```

### 4.3 Console 라우팅

```tsx
// console/src/App.tsx (라우터 발췌)
import ModuleSector from '@/components/modules/ModuleSector';
import ModuleGuard from '@/components/modules/ModuleGuard';

function ModuleRoute() {
  const { moduleId, '*': pagePath } = useParams();
  const { token, theme } = useAuth();
  const { data: module } = useModule(moduleId);

  return (
    <ModuleGuard moduleId={moduleId!}>
      <ModuleSector
        moduleId={moduleId!}
        basePath={module?.console?.adminPath || '/admin'}
        pagePath={pagePath || ''}
        token={token}
        theme={theme}
      />
    </ModuleGuard>
  );
}

// 라우터
<Routes>
  {/* Foundation (A 내장) */}
  <Route path="/dashboard" element={<Dashboard />} />
  <Route path="/users" element={<Users />} />
  <Route path="/modules" element={<ModuleStore />} />

  {/* Module (C iframe) — 범용 라우트 */}
  <Route path="/modules/:moduleId/*" element={<ModuleRoute />} />
</Routes>
```

### 4.4 ModuleGuard 패턴

미설치 모듈 접근 시 "설치 안내" UI 표시:

```tsx
function ModuleGuard({ moduleId, children }) {
  const { data: module, isLoading } = useModule(moduleId);

  if (isLoading) return <Loading />;
  if (!module || module.status !== 'active') {
    return (
      <ModuleNotInstalled
        moduleId={moduleId}
        onInstall={() => navigate('/modules')}
      />
    );
  }
  return children;
}
```

### 4.5 Ingress를 통한 iframe 프록시

Console/Portal에서 iframe src가 직접 K8s ClusterIP에 접근할 수 없으므로,
Traefik Ingress로 프록시합니다:

```yaml
# Console Ingress에 모듈 프록시 규칙 추가
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: polyon-module-proxy
  namespace: polyon
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
    - host: console.cmars.com
      http:
        paths:
          # 모듈별 프록시 경로 (설치 시 자동 생성)
          - path: /modules/drive/proxy
            pathType: Prefix
            backend:
              service:
                name: polyon-drive
                port:
                  number: 8080
          - path: /modules/chat/proxy
            pathType: Prefix
            backend:
              service:
                name: polyon-chat
                port:
                  number: 8080
```

결과적으로 iframe src 경로:
```
Console iframe → https://console.cmars.com/modules/drive/proxy/admin#/overview
                 → Ingress → polyon-drive:8080/admin#/overview
```

---

## 5. 동적 메뉴 로딩

### 5.1 Core API로 설치된 모듈 조회

```
GET /api/v1/modules?status=active

Response:
[
  {
    "id": "drive",
    "name": "PP Drive",
    "version": "0.1.0",
    "icon": "Document",
    "status": "active",
    "console": {
      "menuGroup": "services",
      "adminPath": "/admin",
      "pages": [
        { "id": "overview", "title": "개요", "icon": "Dashboard", "path": "overview", "default": true },
        { "id": "storage", "title": "스토리지", "icon": "DataBase", "path": "storage" },
        ...
      ]
    },
    "portal": {
      "menuGroup": "apps",
      "userPath": "/",
      "pages": [...]
    }
  }
]
```

### 5.2 SideNav 동적 렌더링

```tsx
function Sidebar() {
  const { data: modules } = useInstalledModules();

  // 그룹별 모듈 분류
  const serviceModules = modules?.filter(m => m.console?.menuGroup === 'services') || [];

  return (
    <SideNav>
      {/* Foundation 고정 (A 내장) */}
      <SideNavMenu title="시스템">
        <SideNavMenuItem href="/dashboard">대시보드</SideNavMenuItem>
        <SideNavMenuItem href="/users">사용자 관리</SideNavMenuItem>
        <SideNavMenuItem href="/modules">시스템 정보</SideNavMenuItem>
      </SideNavMenu>

      {/* 모듈 동적 (C iframe) */}
      <SideNavMenu title="서비스">
        {serviceModules.map(m => {
          const defaultPage = m.console.pages.find(p => p.default);
          return (
            <SideNavMenu
              key={m.id}
              title={m.name}
              renderIcon={() => <ModuleIcon name={m.icon} />}
            >
              {m.console.pages.map(page => (
                <SideNavMenuItem
                  key={page.id}
                  href={`/modules/${m.id}/${page.path}`}
                >
                  {page.title}
                </SideNavMenuItem>
              ))}
            </SideNavMenu>
          );
        })}
      </SideNavMenu>

      {/* Foundation 고정 (A 내장) */}
      <SideNavMenu title="인프라">
        <SideNavMenuItem href="/infra/auth">인증</SideNavMenuItem>
        <SideNavMenuItem href="/infra/mail">메일</SideNavMenuItem>
        <SideNavMenuItem href="/infra/storage">스토리지</SideNavMenuItem>
      </SideNavMenu>
    </SideNav>
  );
}
```

---

## 6. 모듈 UI 번들 구조

### 6.1 모듈 이미지 내 UI 위치

```
모듈 Docker 이미지:
├── /polyon-module/
│   └── module.yaml               # 모듈 매니페스트
├── /app/                         # 모듈 백엔드 바이너리
├── /web/                         # 사원용 UI (portal pages)
│   ├── index.html
│   └── static/
│       ├── app.js                # 자기완결적 JS 번들
│       └── app.css
└── /admin/                       # 관리자용 UI (console pages)
    ├── index.html
    └── static/
        ├── admin.js              # 자기완결적 JS 번들
        └── admin.css
```

### 6.2 모듈 UI HTML 구조

```html
<!-- /admin/index.html (모듈 Admin UI) -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>PP Drive Admin</title>
  <link rel="stylesheet" href="static/admin.css" />
</head>
<body>
  <div id="root"></div>
  <!-- Module SDK (postMessage 통신) -->
  <script src="https://cdn.polyon.io/module-sdk/v1/sdk.js"></script>
  <!-- 또는 npm 번들: import { PolyonSDK } from '@polyon/module-sdk' -->
  <script src="static/admin.js"></script>
</body>
</html>
```

### 6.3 자기완결적 번들 요구사항

모듈 UI 번들은 외부 CDN이나 Console 내부 라이브러리에 의존하지 않습니다:

| 항목 | 규칙 |
|------|------|
| **프레임워크** | 자유 (React, Vue, Svelte, Vanilla JS 등) |
| **디자인 시스템** | Carbon Design 권장, 다른 L/F 허용 |
| **의존성** | 번들에 자체 포함 (vendor chunk). Console과 공유 없음 |
| **Module SDK** | `@polyon/module-sdk` 필수 포함 — Console과 통신용 |
| **라우팅** | Hash routing 권장 (`#/overview`, `#/settings`) — iframe 내부 |
| **API 호출** | Module SDK를 통해 인증 토큰 획득 → 모듈 자체 API 호출 |

---

## 7. 모듈 UI 개발 가이드 (Module 개발자용)

### 7.1 React + Carbon Design 예시 (권장)

```tsx
// admin/src/App.tsx (Drive Admin UI)
import { useEffect, useState } from 'react';
import { PolyonSDK } from '@polyon/module-sdk';
import {
  Theme,
  Content,
  DataTable,
  TableContainer,
  Table,
  TableHead,
  TableRow,
  TableHeader,
  TableBody,
  TableCell,
  ProgressBar,
} from '@carbon/react';
import { HashRouter, Routes, Route } from 'react-router-dom';

const sdk = PolyonSDK.init();

function App() {
  const [theme, setTheme] = useState<'white' | 'g100'>('white');

  useEffect(() => {
    sdk.theme.onChange((t) => setTheme(t === 'dark' ? 'g100' : 'white'));
  }, []);

  return (
    <Theme theme={theme}>
      <Content>
        <HashRouter>
          <Routes>
            <Route path="/overview" element={<Overview />} />
            <Route path="/storage" element={<Storage />} />
            <Route path="/quota" element={<Quota />} />
            <Route path="/activity" element={<Activity />} />
            <Route path="/settings" element={<Settings />} />
            <Route path="*" element={<Overview />} />
          </Routes>
        </HashRouter>
      </Content>
    </Theme>
  );
}

function Overview() {
  const [status, setStatus] = useState(null);

  useEffect(() => {
    sdk.api.get('/api/v1/admin/status').then(setStatus);
  }, []);

  // Carbon DataTable로 렌더링...
}
```

### 7.2 API 호출 패턴

모듈 UI는 **자기 자신의 백엔드 API를 직접 호출**합니다.
(Console → Core 프록시 → 모듈 경로가 아님. iframe 자체가 이미 모듈 서비스로 라우팅됨)

```tsx
// Module SDK를 통해 인증된 API 호출
const status = await sdk.api.get('/api/v1/admin/status');
const stats = await sdk.api.get('/api/v1/admin/stats');

// 파일 업/다운로드 URL
const { url } = await sdk.api.post('/api/v1/upload-url', { path: 'docs/report.pdf' });
```

Module SDK가 Console에서 전달받은 JWT 토큰을 자동으로 Authorization 헤더에 포함합니다.

---

## 8. Console/Portal Foundation 페이지 (A 내장)

Foundation 페이지는 기존 방식 그대로 Console/Portal 코드에 직접 구현합니다.

### 8.1 Console Foundation 페이지

```
console/src/pages/
├── system/
│   ├── Dashboard.tsx        # 시스템 대시보드 (전체 현황)
│   ├── Users.tsx            # 사용자 관리 (AD DC CRUD)
│   └── ModuleStore.tsx      # 모듈 설치/삭제
├── infra/
│   ├── Auth.tsx             # Keycloak 관리
│   ├── Mail.tsx             # Stalwart Mail 관리
│   └── Storage.tsx          # RustFS 관리
```

### 8.2 Portal Foundation 페이지

```
portal/src/pages/
├── home/
│   ├── Dashboard.tsx        # 사원 홈 대시보드
│   ├── Notifications.tsx    # 알림 센터
│   └── Profile.tsx          # 내 프로필
```

---

## 9. 모듈 개발 체크리스트

### 9.1 백엔드 (모듈 개발자 책임)

- [ ] Admin API 구현 (`/api/v1/admin/*`) — admin-api-spec.md 참조
- [ ] User API 구현 (`/api/v1/*`) — module-spec.md 참조
- [ ] Health endpoint (`GET /health`)
- [ ] module.yaml `console`, `portal` 섹션 선언
- [ ] module.yaml `icon` 필드에 `@carbon/icons` 이름 지정

### 9.2 Admin UI (모듈 개발자 책임)

- [ ] `/admin/` 경로에 SPA 서빙 (Hash routing)
- [ ] `@polyon/module-sdk` 통합 — 인증, 테마, 네비게이션
- [ ] module.yaml `console.pages`에 선언된 모든 페이지 구현
- [ ] Carbon Design 권장 (통일성), 다른 프레임워크 허용
- [ ] 의존성 자기완결적 번들 (외부 CDN 의존 금지)

### 9.3 사원 UI (모듈 개발자 책임)

- [ ] `/` 경로에 SPA 서빙 (Hash routing)
- [ ] `@polyon/module-sdk` 통합
- [ ] module.yaml `portal.pages`에 선언된 모든 페이지 구현
- [ ] 자기완결적 번들

### 9.4 Console/Portal (플랫폼 개발자 책임)

- [ ] `ModuleSector` 컴포넌트 구현 (iframe 렌더링)
- [ ] `ModuleGuard` 컴포넌트 구현 (미설치 안내)
- [ ] 동적 메뉴 로딩 (`GET /api/v1/modules`)
- [ ] postMessage 프로토콜 구현 (module-sdk-spec.md 참조)
- [ ] 모듈별 Ingress 프록시 자동 생성

---

## 10. FAQ

### Q: 모듈 추가 시 Console/Portal 코드를 수정해야 하는가?
**A:** 아닙니다. module.yaml 메뉴 선언 + 자기완결적 UI 번들이면 Console/Portal은 자동으로 메뉴를 표시하고 iframe으로 렌더링합니다. 코드 수정 없이 모듈 설치만으로 UI가 확장됩니다.

### Q: 모듈 UI에서 Carbon Design을 반드시 써야 하는가?
**A:** 권장이지 필수가 아닙니다. Carbon을 사용하면 Console과의 시각적 통일성이 보장되어 사용자 경험이 좋습니다. 하지만 모듈 특성에 따라 별도 L/F도 허용합니다 (예: Drive는 Google Drive 스타일).

### Q: 모듈 삭제 시 UI는?
**A:** Console/Portal은 `GET /api/v1/modules`에서 삭제된 모듈을 더 이상 반환하지 않으므로, 메뉴에서 자동으로 사라집니다. 직접 URL 접근 시 ModuleGuard가 "설치 안내" 페이지를 표시합니다.

### Q: iframe 보안은?
**A:** `sandbox="allow-scripts allow-same-origin allow-forms allow-popups"` 적용. 동일 도메인이므로 쿠키/토큰 공유 가능. Module SDK의 postMessage 프로토콜로 안전한 통신.

### Q: Admin 페이지와 사원 페이지의 L/F가 달라도 되는가?
**A:** 네. Admin과 사원은 완전히 다른 세계입니다. Admin은 관리/감시 목적, 사원은 업무 사용 목적이므로 각각의 L/F에 최적화하는 것이 맞습니다.

---

## 참조

- [Module Spec](module-spec.md) — 모듈 기본 규격
- [Module SDK Spec](module-sdk-spec.md) — @polyon/module-sdk 인터페이스
- [PostMessage Protocol](postmessage-protocol-spec.md) — Console ↔ Module 통신
- [Admin API Spec](admin-api-spec.md) — 관리자 API 규격
- [Module Lifecycle Spec](module-lifecycle-spec.md) — 설치/삭제 자동화
- [RFP Drive](rfp-drive.md) — Drive 모듈 기능 요구사항
