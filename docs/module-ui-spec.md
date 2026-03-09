# PolyON Module UI Spec — 모듈 UI 통합 규격

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 대상: 모듈 개발자, Console/Portal 개발자, AI 코딩 에이전트

---

## 1. 개요

PP(PolyON Platform) 모듈은 **백엔드 엔진**과 **프론트엔드 UI**로 구성됩니다.

- **백엔드:** 독립 컨테이너 (Rust, Go 등). module-spec.md 참조
- **프론트엔드:** Console(관리자)과 Portal(사원)에 통합되는 UI

이 문서는 **모듈 UI가 Console/Portal에 통합되는 방식**을 정의합니다.

---

## 2. UI 아키텍처

```
┌──────────────────────────────────────────────────┐
│              Console (관리자 SPA)                 │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │ Foundation │ │ Drive Admin│ │ Chat Admin │   │
│  │  Pages     │ │  Pages     │ │  Pages     │   │
│  └────────────┘ └──────┬─────┘ └──────┬─────┘   │
│                        │              │          │
│              ┌─────────┴──────────────┘          │
│              │    Core API (프록시)               │
│              └─────────┬──────────────┐          │
│                        │              │          │
│                  polyon-drive    polyon-chat      │
│                  (Admin API)    (Admin API)       │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│               Portal (사원 SPA)                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │  Dashboard │ │ Drive UI   │ │  Chat UI   │   │
│  │            │ │ (파일탐색) │ │            │   │
│  └────────────┘ └──────┬─────┘ └──────┬─────┘   │
│                        │              │          │
│              ┌─────────┴──────────────┘          │
│              │    Core API (프록시)               │
│              └─────────┬──────────────┐          │
│                        │              │          │
│                  polyon-drive    polyon-chat      │
│                  (User API)     (User API)       │
└──────────────────────────────────────────────────┘
```

### 2.1 기본 원칙

| 원칙 | 설명 |
|------|------|
| **Console/Portal이 UI 주체** | 모듈은 API만 제공. UI 페이지는 Console/Portal React 코드로 구현 |
| **Core API 프록시** | Console/Portal → Core API → 모듈 백엔드. 직접 호출 금지 |
| **Carbon Design 필수** | Console/Portal 내 모든 UI는 `@carbon/react` 컴포넌트 사용 |
| **동적 메뉴** | 모듈 설치/삭제 시 메뉴가 자동으로 추가/제거 |
| **module.yaml 선언적** | 모듈이 필요한 메뉴/페이지를 manifest에 선언 |

### 2.2 왜 Console/Portal 내장 방식인가

| 대안 | 문제점 |
|------|--------|
| iframe | UX 단절, 스타일 불일치, 인증 전달 복잡 |
| Module Federation | 빌드 의존성 지옥, React/Carbon 버전 충돌 |
| 모듈 자체 SPA | 네비게이션 단절, SSO 쿠키 관리 복잡 |
| **Console/Portal 내장** | ✅ 일관된 UX, 인증 공유, Carbon 강제, 유지보수 용이 |

**결론:** 모듈은 **Admin API + User API**만 제공하고, UI 코드는 **Console/Portal 프로젝트 내부**에 모듈별 디렉토리로 구현합니다.

---

## 3. module.yaml UI 선언

### 3.1 확장된 module.yaml 스키마

```yaml
apiVersion: polyon.io/v1
kind: Module

metadata:
  id: drive
  name: PP Drive
  version: "0.1.0"
  category: engine
  description: "파일 스토리지 서비스"
  icon: Document           # @carbon/icons 이름 (32px variant)

spec:
  engine: drive

  # ... (기존 resources, database, ingress, ldap 등)

  # ── UI 선언 (신규) ──

  console:                 # Admin Console 페이지 선언
    menuGroup: services    # 메뉴 그룹 (아래 §3.2 참조)
    pages:
      - id: overview
        title: "개요"
        icon: Dashboard
        route: /modules/drive
        default: true      # 모듈 메뉴 클릭 시 기본 페이지
      - id: storage
        title: "스토리지"
        icon: DataBase
        route: /modules/drive/storage
      - id: quota
        title: "할당량"
        icon: Meter
        route: /modules/drive/quota
      - id: activity
        title: "활동 로그"
        icon: Activity
        route: /modules/drive/activity
      - id: settings
        title: "설정"
        icon: Settings
        route: /modules/drive/settings

  portal:                  # Portal 사용자 페이지 선언
    menuGroup: apps
    pages:
      - id: files
        title: "드라이브"
        icon: Folder
        route: /drive
        default: true
      - id: shared
        title: "공유"
        icon: Share
        route: /drive/shared
      - id: recent
        title: "최근"
        icon: RecentlyViewed
        route: /drive/recent
      - id: trash
        title: "휴지통"
        icon: TrashCan
        route: /drive/trash
```

### 3.2 메뉴 그룹 (Console)

Console 좌측 사이드바 메뉴 구조:

```
──────────────────────
  시스템                   ← group: system (Foundation 전용)
    대시보드
    사용자 관리
    시스템 정보 (모듈 설치)
──────────────────────
  서비스                   ← group: services (설치된 모듈)
    📁 드라이브            ← module: drive
    💬 채팅                ← module: chat
    📝 위키                ← module: wiki
──────────────────────
  인프라                   ← group: infra (Foundation 전용)
    인증 (Keycloak)
    메일 (Stalwart)
    스토리지 (RustFS)
──────────────────────
```

| 그룹 ID | 표시 이름 | 대상 |
|---------|-----------|------|
| `system` | 시스템 | Foundation 고정 메뉴 |
| `services` | 서비스 | 설치된 모듈 (동적) |
| `infra` | 인프라 | Foundation 고정 메뉴 |

### 3.3 메뉴 그룹 (Portal)

Portal 좌측 사이드바:

```
──────────────────────
  홈                      ← 고정
    대시보드
    알림
──────────────────────
  앱                      ← group: apps (설치된 모듈)
    📁 드라이브
    💬 채팅
    📝 위키
──────────────────────
  도구                    ← group: tools (설치된 모듈)
    🤖 AI 어시스턴트
──────────────────────
```

### 3.4 아이콘 규격

- **반드시 `@carbon/icons-react` 컴포넌트 이름 사용**
- emoji 절대 금지
- `module.yaml`의 `icon` 필드: 32px variant 기준 컴포넌트 이름
- 예: `Document`, `Folder`, `Chat`, `Dashboard`, `Settings`

Console/Portal에서 아이콘 렌더링:
```tsx
import * as CarbonIcons from '@carbon/icons-react';

function ModuleIcon({ name, size = 20 }: { name: string; size?: number }) {
  const Icon = (CarbonIcons as any)[name];
  if (!Icon) return <CarbonIcons.Application size={size} />;
  return <Icon size={size} />;
}
```

---

## 4. Console Admin 페이지 구현 규격

### 4.1 디렉토리 구조 (Console 프로젝트 내)

```
console/src/
├── pages/
│   ├── system/           # Foundation 고정 페이지
│   │   ├── Dashboard/
│   │   ├── Users/
│   │   └── ModuleStore/  # 모듈 설치/삭제 (시스템 정보)
│   │
│   └── modules/          # 모듈별 Admin 페이지
│       ├── drive/        # ← Drive 모듈 Admin
│       │   ├── index.tsx           # 라우터
│       │   ├── DriveOverview.tsx   # 개요 (상태, 통계)
│       │   ├── DriveStorage.tsx    # 스토리지 관리
│       │   ├── DriveQuota.tsx      # 할당량 관리
│       │   ├── DriveActivity.tsx   # 활동 로그
│       │   └── DriveSettings.tsx   # 설정
│       │
│       ├── chat/         # ← Chat 모듈 Admin
│       │   └── ...
│       └── wiki/         # ← Wiki 모듈 Admin
│           └── ...
│
├── components/
│   └── modules/          # 모듈 공통 컴포넌트
│       ├── ModuleGuard.tsx      # 미설치 모듈 접근 차단
│       ├── ModulePageLayout.tsx # 모듈 페이지 공통 레이아웃
│       └── ModuleStatusCard.tsx # 모듈 상태 카드
│
└── hooks/
    └── useModuleApi.ts   # Core API → 모듈 프록시 호출 hook
```

### 4.2 모듈 Admin 페이지 라우팅

```tsx
// console/src/pages/modules/drive/index.tsx
import { Routes, Route } from 'react-router-dom';
import ModuleGuard from '@/components/modules/ModuleGuard';
import DriveOverview from './DriveOverview';
import DriveStorage from './DriveStorage';
import DriveQuota from './DriveQuota';
import DriveActivity from './DriveActivity';
import DriveSettings from './DriveSettings';

export default function DriveAdmin() {
  return (
    <ModuleGuard moduleId="drive">
      <Routes>
        <Route index element={<DriveOverview />} />
        <Route path="storage" element={<DriveStorage />} />
        <Route path="quota" element={<DriveQuota />} />
        <Route path="activity" element={<DriveActivity />} />
        <Route path="settings" element={<DriveSettings />} />
      </Routes>
    </ModuleGuard>
  );
}
```

### 4.3 동적 메뉴 로딩

Console은 시작 시 Core API에서 설치된 모듈 목록 + UI 선언을 가져옵니다:

```tsx
// Console 초기화 시
const { data: modules } = useQuery('installedModules', () =>
  coreApi.get('/api/v1/modules?status=active')
);

// 사이드바 렌더링
function Sidebar() {
  return (
    <SideNav>
      {/* Foundation 고정 메뉴 */}
      <SideNavMenu title="시스템">
        <SideNavMenuItem href="/dashboard">대시보드</SideNavMenuItem>
        <SideNavMenuItem href="/users">사용자 관리</SideNavMenuItem>
        <SideNavMenuItem href="/modules">시스템 정보</SideNavMenuItem>
      </SideNavMenu>

      {/* 설치된 모듈 동적 메뉴 */}
      <SideNavMenu title="서비스">
        {modules
          ?.filter(m => m.console?.pages?.length > 0)
          .map(m => (
            <SideNavMenuItem
              key={m.id}
              href={m.console.pages.find(p => p.default)?.route}
              renderIcon={() => <ModuleIcon name={m.icon} />}
            >
              {m.name}
            </SideNavMenuItem>
          ))
        }
      </SideNavMenu>
    </SideNav>
  );
}
```

### 4.4 ModuleGuard 패턴

미설치 모듈 페이지 접근 시:

```tsx
// components/modules/ModuleGuard.tsx
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

### 4.5 Core API 프록시 호출

Console/Portal은 모듈 백엔드를 **직접 호출하지 않습니다.** Core API를 통해 프록시합니다:

```
Console UI → Core API → 모듈 백엔드
POST /api/v1/modules/drive/proxy/admin/quota
  → Core가 Bearer JWT 검증 + 관리자 권한 확인
  → Core가 http://polyon-drive:8080/api/v1/admin/quota 호출
  → 응답 전달
```

```tsx
// hooks/useModuleApi.ts
function useModuleApi(moduleId: string) {
  const { token } = useAuth();

  return {
    get: (path: string) =>
      fetch(`/api/v1/modules/${moduleId}/proxy${path}`, {
        headers: { Authorization: `Bearer ${token}` },
      }),
    post: (path: string, body: any) =>
      fetch(`/api/v1/modules/${moduleId}/proxy${path}`, {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(body),
      }),
  };
}
```

---

## 5. Portal 사용자 페이지 구현 규격

### 5.1 디렉토리 구조 (Portal 프로젝트 내)

```
portal/src/
├── pages/
│   ├── home/             # 고정 페이지
│   │   └── Dashboard/
│   │
│   └── modules/          # 모듈별 사용자 페이지
│       ├── drive/
│       │   ├── index.tsx
│       │   ├── DriveFiles.tsx       # 파일 탐색기
│       │   ├── DriveShared.tsx      # 공유받은 파일
│       │   ├── DriveRecent.tsx      # 최근 파일
│       │   └── DriveTrash.tsx       # 휴지통
│       │
│       ├── chat/
│       │   └── ...
│       └── wiki/
│           └── ...
│
├── components/
│   └── drive/            # Drive 전용 컴포넌트
│       ├── FileList.tsx
│       ├── FileUpload.tsx
│       ├── FolderTree.tsx
│       ├── ShareDialog.tsx
│       └── QuotaBar.tsx
```

### 5.2 Portal Drive UI — Carbon Design 필수 컴포넌트

| UI 요소 | Carbon 컴포넌트 |
|---------|-----------------|
| 파일 목록 | `DataTable`, `TableRow`, `TableCell` |
| 툴바 | `ButtonSet`, `Button`, `OverflowMenu` |
| 업로드 | `FileUploader`, `FileUploaderDropContainer` |
| 검색 | `Search` |
| 경로 (Breadcrumb) | `Breadcrumb`, `BreadcrumbItem` |
| 폴더 트리 | `TreeView`, `TreeNode` |
| 공유 다이얼로그 | `Modal`, `TextInput`, `DatePicker` |
| 할당량 바 | `ProgressBar` |
| 알림/토스트 | `InlineNotification`, `ToastNotification` |
| 아이콘 | `@carbon/icons-react` — `Folder`, `Document`, `Upload`, `Download`, `TrashCan`, `Share`, `Search` |
| 로딩 | `DataTableSkeleton`, `SkeletonText` |
| 빈 상태 | `Tile` + 안내 메시지 |

### 5.3 Portal 인증 흐름

```
사원 브라우저 → Portal (Keycloak OIDC PKCE 로그인)
  → Access Token 획득
  → Core API 호출 시 Bearer JWT 첨부
  → Core가 JWT 검증 + 사용자 권한 확인
  → Core가 모듈 백엔드 호출
```

Portal의 모듈 API 호출도 Console과 동일하게 Core 프록시:
```
GET /api/v1/modules/drive/proxy/files
GET /api/v1/modules/drive/proxy/files/{path}
PUT /api/v1/modules/drive/proxy/files/{path}
DELETE /api/v1/modules/drive/proxy/files/{path}
GET /api/v1/modules/drive/proxy/trash
GET /api/v1/modules/drive/proxy/shared-with-me
GET /api/v1/modules/drive/proxy/quota
GET /api/v1/modules/drive/proxy/search?q=...
```

---

## 6. 모듈 개발 체크리스트

### 6.1 백엔드 (모듈 개발자 책임)

- [ ] Admin API 구현 (`/api/v1/admin/*`) — admin-api-spec.md 참조
- [ ] User API 구현 (`/api/v1/*`) — 기존 module-spec.md 참조
- [ ] Health endpoint (`GET /health`)
- [ ] module.yaml에 `console`, `portal` 섹션 선언
- [ ] module.yaml `icon` 필드에 `@carbon/icons` 이름 지정

### 6.2 Console Admin 페이지 (Console 개발자 책임)

- [ ] `console/src/pages/modules/{moduleId}/` 디렉토리 생성
- [ ] module.yaml `console.pages`에 선언된 모든 페이지 구현
- [ ] `ModuleGuard`로 미설치 상태 처리
- [ ] `useModuleApi` hook으로 Core 프록시 경유 API 호출
- [ ] Carbon Design 컴포넌트만 사용
- [ ] `@carbon/icons-react`만 사용 (emoji 금지)

### 6.3 Portal 사용자 페이지 (Portal 개발자 책임)

- [ ] `portal/src/pages/modules/{moduleId}/` 디렉토리 생성
- [ ] module.yaml `portal.pages`에 선언된 모든 페이지 구현
- [ ] Keycloak OIDC JWT로 Core API 호출
- [ ] Carbon Design 컴포넌트만 사용

---

## 7. FAQ

### Q: 모듈이 자체 웹 UI를 가져도 되는가?
**A:** 개발/디버그 용도로만 허용. 실제 사용자에게는 Portal UI를 통해 노출합니다. 모듈 자체 UI에 외부 Ingress를 연결하지 않습니다.

### Q: 새 모듈 추가 시 Console/Portal 코드를 수정해야 하는가?
**A:** 네. Console/Portal 프로젝트에 모듈별 페이지 코드를 추가하고 빌드해야 합니다. 이는 **UI 품질 보장**(Carbon Design 강제)과 **보안**(Core 프록시 강제)을 위한 의도적 설계입니다.

### Q: 모듈 삭제 시 UI는?
**A:** ModuleGuard가 미설치 상태를 감지하여 "설치 안내" 페이지를 표시합니다. Console/Portal 코드에서 모듈 페이지를 물리적으로 삭제할 필요는 없습니다.

### Q: Portal에서 Drive 파일 업로드/다운로드는 Core를 경유하는가?
**A:** 파일 바이너리 전송은 **Core 프록시를 경유하지 않습니다.** Core에서 presigned URL 또는 직접 WebDAV URL을 받아 브라우저가 모듈에 직접 업로드/다운로드합니다. 메타데이터 API만 Core 프록시를 경유합니다. 자세한 내용은 admin-api-spec.md §5 참조.

---

## 참조

- [Module Spec](module-spec.md) — 모듈 기본 규격
- [Admin API Spec](admin-api-spec.md) — 관리자 API 규격
- [Module Lifecycle Spec](module-lifecycle-spec.md) — 설치/삭제 자동화
- [RFP Drive](rfp-drive.md) — Drive 기능 요구사항
