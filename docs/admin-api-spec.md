# PolyON Admin API Spec — 모듈 관리자 API 규격

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 대상: 모듈 백엔드 개발자, Core 개발자, AI 코딩 에이전트

---

## 1. 개요

모듈이 Console(관리자)에 관리 기능을 제공하기 위해 구현해야 하는 **Admin API 규격**입니다.

```
Console UI → Core API (프록시) → 모듈 Admin API
```

### 1.1 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Core 프록시 필수** | Console → 모듈 직접 호출 금지. Core가 인증/권한 검증 후 프록시 |
| **표준 엔드포인트** | 모든 모듈이 동일한 패턴의 Admin API를 제공 |
| **Admin ≠ User** | Admin API는 `/api/v1/admin/*` 경로, User API는 `/api/v1/*` |
| **내부 인증** | 모듈 간 통신은 Core가 발급하는 내부 JWT 사용 |

---

## 2. 인증 흐름

### 2.1 Console → Core → 모듈

```
1. 관리자가 Console에 Keycloak OIDC로 로그인 (admin realm)
2. Console이 Core API 호출: Bearer {keycloak-jwt}
3. Core가 JWT 검증:
   - Keycloak JWKS로 서명 확인
   - admin realm 소속 확인
   - 관리자 역할(role) 확인
4. Core가 모듈에 프록시:
   - 새로운 내부 JWT 발급 (또는 원본 전달)
   - X-Polyon-Admin: true 헤더 추가
   - X-Polyon-User: {username} 헤더 추가
```

### 2.2 모듈 측 인증 처리

모듈은 내부 요청을 아래 헤더로 판별합니다:

```
X-Polyon-Admin: true          # 관리자 요청 여부
X-Polyon-User: admin          # 요청자 username
Authorization: Bearer {jwt}    # Core가 전달한 JWT (선택적 검증)
```

**MVP 단계:** 모듈은 `X-Polyon-Admin: true` 헤더가 있으면 Admin API 접근을 허용합니다.
K8s 네트워크 정책으로 외부에서 모듈 직접 접근을 차단하므로 안전합니다.

**향후:** 모듈이 JWT를 자체 검증하여 더 세밀한 권한 제어.

### 2.3 Portal → Core → 모듈 (일반 사용자)

```
1. 사원이 Portal에 Keycloak OIDC로 로그인 (polyon realm)
2. Portal이 Core API 호출: Bearer {keycloak-jwt}
3. Core가 JWT 검증: polyon realm 소속 확인
4. Core가 모듈에 프록시:
   - X-Polyon-Admin: false
   - X-Polyon-User: {username}
```

모듈은 `X-Polyon-Admin`이 `false`이면 Admin API 접근을 거부합니다 (403).

---

## 3. Core 프록시 API

### 3.1 프록시 라우팅 규칙

```
Console/Portal 요청:
  GET  /api/v1/modules/{moduleId}/proxy/{path}
  POST /api/v1/modules/{moduleId}/proxy/{path}
  PUT  /api/v1/modules/{moduleId}/proxy/{path}
  DELETE /api/v1/modules/{moduleId}/proxy/{path}

Core가 변환:
  → GET  http://polyon-{moduleId}:{port}/api/v1/{path}
  → POST http://polyon-{moduleId}:{port}/api/v1/{path}
  → PUT  http://polyon-{moduleId}:{port}/api/v1/{path}
  → DELETE http://polyon-{moduleId}:{port}/api/v1/{path}
```

**예시:**
```
Console:  GET /api/v1/modules/drive/proxy/admin/quota
Core →:   GET http://polyon-drive:8080/api/v1/admin/quota
          + X-Polyon-Admin: true
          + X-Polyon-User: admin

Portal:   GET /api/v1/modules/drive/proxy/files
Core →:   GET http://polyon-drive:8080/api/v1/files
          + X-Polyon-Admin: false
          + X-Polyon-User: testuser1
```

### 3.2 Core 프록시 Go 구현 가이드

```go
// core/internal/api/module_proxy.go

func ModuleProxyHandler(w http.ResponseWriter, r *http.Request) {
    moduleId := chi.URLParam(r, "moduleId")
    proxyPath := chi.URLParam(r, "*")

    // 1. 모듈 활성 상태 확인
    module, err := store.GetActiveModule(moduleId)
    if err != nil {
        http.Error(w, "Module not found or inactive", 404)
        return
    }

    // 2. 인증 정보 추출 (미들웨어에서 처리됨)
    claims := r.Context().Value("claims").(*JWTClaims)
    isAdmin := claims.Realm == "admin"

    // 3. Admin API 접근 권한 확인
    if strings.HasPrefix(proxyPath, "admin/") && !isAdmin {
        http.Error(w, "Admin access required", 403)
        return
    }

    // 4. 프록시 대상 URL 구성
    port := module.Manifest.Spec.Resources.Ports[0].ContainerPort
    targetURL := fmt.Sprintf("http://polyon-%s:%d/api/v1/%s", moduleId, port, proxyPath)

    // 5. 프록시 요청 생성
    proxyReq, _ := http.NewRequest(r.Method, targetURL, r.Body)
    proxyReq.Header = r.Header.Clone()
    proxyReq.Header.Set("X-Polyon-Admin", strconv.FormatBool(isAdmin))
    proxyReq.Header.Set("X-Polyon-User", claims.Username)

    // 6. 프록시 실행
    resp, err := httpClient.Do(proxyReq)
    if err != nil {
        http.Error(w, "Module proxy error", 502)
        return
    }
    defer resp.Body.Close()

    // 7. 응답 전달
    copyHeaders(w.Header(), resp.Header)
    w.WriteHeader(resp.StatusCode)
    io.Copy(w, resp.Body)
}
```

---

## 4. 모듈 Admin API 표준 엔드포인트

모든 모듈이 구현해야 하는 Admin API입니다.

### 4.1 필수 (MUST)

```
GET  /api/v1/admin/status         # 모듈 상태 정보
```

### 4.2 권장 (SHOULD — 해당 기능이 있는 경우)

```
# 통계
GET  /api/v1/admin/stats          # 전체 통계

# 할당량 관리
GET  /api/v1/admin/quota          # 전체 사용자 할당량 목록
GET  /api/v1/admin/quota/:userId  # 특정 사용자 할당량
PUT  /api/v1/admin/quota/:userId  # 할당량 설정

# 활동 감사
GET  /api/v1/admin/activities     # 전체 활동 로그 (관리자 관점)

# 설정
GET  /api/v1/admin/settings       # 모듈 설정 조회
PUT  /api/v1/admin/settings       # 모듈 설정 변경

# 사용자 관리
GET  /api/v1/admin/users          # 모듈 내 사용자 목록/상태
GET  /api/v1/admin/users/:userId  # 특정 사용자 상세
```

---

## 5. Drive 모듈 Admin API 상세

Drive 모듈이 구현해야 할 구체적인 Admin API입니다.

### 5.1 모듈 상태 (필수)

```
GET /api/v1/admin/status

Response 200:
{
  "module_id": "drive",
  "version": "0.1.0",
  "status": "healthy",
  "uptime_secs": 86400,
  "connections": {
    "database": "connected",
    "s3": "connected",
    "ldap": "connected"
  },
  "stats": {
    "total_users": 15,
    "total_files": 1234,
    "total_size_bytes": 5368709120,
    "s3_bucket": "drive"
  }
}
```

Console "드라이브 > 개요" 페이지에서 사용.

### 5.2 스토리지 통계

```
GET /api/v1/admin/stats

Response 200:
{
  "total_size_bytes": 5368709120,
  "total_files": 1234,
  "total_folders": 89,
  "total_users": 15,
  "file_type_distribution": [
    { "mime_type": "application/pdf", "count": 120, "size_bytes": 1073741824 },
    { "mime_type": "image/jpeg", "count": 450, "size_bytes": 2147483648 },
    { "mime_type": "application/zip", "count": 30, "size_bytes": 536870912 }
  ],
  "user_usage": [
    { "user_id": "testuser1", "display_name": "홍길동", "file_count": 120, "size_bytes": 1073741824 },
    { "user_id": "testuser2", "display_name": "김철수", "file_count": 85, "size_bytes": 536870912 }
  ]
}
```

Console "드라이브 > 스토리지" 페이지에서 사용.

### 5.3 할당량 관리

```
# 전체 사용자 할당량 목록
GET /api/v1/admin/quota?page=1&limit=50

Response 200:
{
  "items": [
    {
      "user_id": "testuser1",
      "display_name": "홍길동",
      "used_bytes": 1073741824,
      "limit_bytes": 5368709120,     # 0 = 무제한
      "usage_percent": 20.0
    }
  ],
  "total": 15,
  "page": 1,
  "limit": 50
}

# 특정 사용자 할당량 조회
GET /api/v1/admin/quota/testuser1

Response 200:
{
  "user_id": "testuser1",
  "display_name": "홍길동",
  "used_bytes": 1073741824,
  "limit_bytes": 5368709120,
  "file_count": 120,
  "folder_count": 15
}

# 할당량 설정
PUT /api/v1/admin/quota/testuser1
Content-Type: application/json

{
  "limit_bytes": 10737418240    # 10GB, 0 = 무제한
}

Response 200:
{ "user_id": "testuser1", "limit_bytes": 10737418240 }
```

Console "드라이브 > 할당량" 페이지에서 사용.

### 5.4 활동 감사 로그

```
GET /api/v1/admin/activities?page=1&limit=100&user_id=testuser1&action=upload&from=2026-03-01&to=2026-03-09

Response 200:
{
  "items": [
    {
      "id": "uuid",
      "user_id": "testuser1",
      "display_name": "홍길동",
      "action": "upload",
      "path": "documents/report.pdf",
      "detail": null,
      "size_bytes": 1048576,
      "created_at": "2026-03-09T08:30:00Z"
    }
  ],
  "total": 456,
  "page": 1,
  "limit": 100
}
```

**필터 파라미터:**
| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `page` | int | 페이지 (기본 1) |
| `limit` | int | 건수 (기본 100, 최대 500) |
| `user_id` | string | 사용자 필터 |
| `action` | string | 액션 필터 (upload, download, delete, move, copy, share, restore) |
| `from` | date | 시작일 (YYYY-MM-DD) |
| `to` | date | 종료일 (YYYY-MM-DD) |

Console "드라이브 > 활동 로그" 페이지에서 사용.

### 5.5 설정

```
GET /api/v1/admin/settings

Response 200:
{
  "default_quota_bytes": 5368709120,    # 신규 사용자 기본 할당량
  "trash_retention_days": 30,           # 휴지통 자동 비우기 (일)
  "max_upload_bytes": 10737418240,      # 단일 파일 최대 크기
  "version_max_count": 50,              # 파일당 최대 버전 수
  "allowed_extensions": [],             # 빈 배열 = 제한 없음
  "blocked_extensions": [".exe", ".bat", ".cmd"]
}

PUT /api/v1/admin/settings
Content-Type: application/json

{
  "default_quota_bytes": 10737418240,
  "trash_retention_days": 60
}

Response 200:
{ "updated": true }
```

Console "드라이브 > 설정" 페이지에서 사용.

### 5.6 사용자 관리

```
GET /api/v1/admin/users?page=1&limit=50&sort=size_desc

Response 200:
{
  "items": [
    {
      "user_id": "testuser1",
      "display_name": "홍길동",
      "file_count": 120,
      "folder_count": 15,
      "used_bytes": 1073741824,
      "limit_bytes": 5368709120,
      "last_activity_at": "2026-03-09T08:30:00Z",
      "home_folder_exists": true
    }
  ],
  "total": 15
}

# 특정 사용자 파일 조회 (관리자 감사용)
GET /api/v1/admin/users/testuser1/files?path=documents

Response 200:
{
  "items": [
    {
      "path": "documents/report.pdf",
      "name": "report.pdf",
      "size": 1048576,
      "is_directory": false,
      "updated_at": "2026-03-09T08:30:00Z"
    }
  ]
}
```

---

## 6. Drive 모듈 User API 정리

Portal에서 사원이 사용하는 API (기존 API 재정리):

### 6.1 파일 관리

```
# 파일/폴더 목록
GET    /api/v1/files
GET    /api/v1/files/{path}

# 검색
GET    /api/v1/search?q={query}&limit=100

# 할당량 (본인)
GET    /api/v1/quota

# 활동 (본인)
GET    /api/v1/activities
GET    /api/v1/files/{path}/activities

# 버전
GET    /api/v1/files/{path}/versions
POST   /api/v1/files/{path}/versions/{n}/restore

# 공유
POST   /api/v1/shares
GET    /api/v1/shares
GET    /api/v1/shares/{id}
PUT    /api/v1/shares/{id}
DELETE /api/v1/shares/{id}
GET    /api/v1/shared-with-me

# 휴지통
GET    /api/v1/trash
POST   /api/v1/trash/{id}/restore
DELETE /api/v1/trash/{id}
DELETE /api/v1/trash
```

### 6.2 파일 업로드/다운로드 (직접 접근)

파일 바이너리 전송은 Core 프록시를 경유하지 않습니다:

```
# Portal이 Core에 업로드/다운로드 URL 요청
GET /api/v1/modules/drive/proxy/upload-url?path={path}
Response: { "url": "https://drive.cmars.com/dav/files/{username}/{path}", "method": "PUT" }

GET /api/v1/modules/drive/proxy/download-url?path={path}
Response: { "url": "https://drive.cmars.com/dav/files/{username}/{path}", "method": "GET" }

# 브라우저가 WebDAV URL에 직접 업로드/다운로드
PUT  https://drive.cmars.com/dav/files/{username}/{path}
GET  https://drive.cmars.com/dav/files/{username}/{path}
Authorization: Basic {ldap-credentials}
```

> **대안 (MVP 단순화):** Core 프록시를 통해 파일 바이너리도 전달. 성능은 떨어지지만 인증이 단순해짐. 대용량 파일이 많아지면 직접 접근으로 전환.

---

## 7. 모듈 Admin API 인증 미들웨어 (Rust 구현 가이드)

### 7.1 Axum 미들웨어

```rust
use axum::http::{HeaderMap, StatusCode};
use axum::response::IntoResponse;

/// Admin 요청에서 사용자 정보 추출
pub struct AdminUser {
    pub username: String,
    pub is_admin: bool,
}

/// X-Polyon-Admin / X-Polyon-User 헤더로 Admin 인증
pub fn extract_admin_user(headers: &HeaderMap) -> Result<AdminUser, (StatusCode, &'static str)> {
    let is_admin = headers
        .get("X-Polyon-Admin")
        .and_then(|v| v.to_str().ok())
        .map(|v| v == "true")
        .unwrap_or(false);

    let username = headers
        .get("X-Polyon-User")
        .and_then(|v| v.to_str().ok())
        .map(String::from)
        .unwrap_or_default();

    if username.is_empty() {
        return Err((StatusCode::UNAUTHORIZED, "Missing X-Polyon-User header"));
    }

    Ok(AdminUser { username, is_admin })
}

/// Admin 전용 라우트 가드
pub fn require_admin(headers: &HeaderMap) -> Result<AdminUser, (StatusCode, &'static str)> {
    let user = extract_admin_user(headers)?;
    if !user.is_admin {
        return Err((StatusCode::FORBIDDEN, "Admin access required"));
    }
    Ok(user)
}
```

### 7.2 라우터 구성

```rust
// 모듈 main.rs 또는 api/mod.rs

let app = Router::new()
    // User API (기존)
    .merge(api::files::router())
    .merge(api::trash::router())
    .merge(api::shares::router())
    // ...

    // Admin API (신규)
    .merge(api::admin::router())    // /api/v1/admin/*

    // Health
    .route("/health", get(health::handler));
```

```rust
// api/admin/mod.rs
pub fn router() -> Router<AppState> {
    Router::new()
        .route("/api/v1/admin/status", get(status_handler))
        .route("/api/v1/admin/stats", get(stats_handler))
        .route("/api/v1/admin/quota", get(list_quota_handler))
        .route("/api/v1/admin/quota/:user_id", get(get_quota_handler).put(set_quota_handler))
        .route("/api/v1/admin/activities", get(admin_activities_handler))
        .route("/api/v1/admin/settings", get(get_settings_handler).put(update_settings_handler))
        .route("/api/v1/admin/users", get(list_users_handler))
        .route("/api/v1/admin/users/:user_id/files", get(admin_user_files_handler))
}
```

---

## 8. 응답 형식 표준

### 8.1 성공 응답

```json
{
  "items": [...],        // 목록 API
  "total": 100,          // 전체 건수
  "page": 1,             // 현재 페이지
  "limit": 50            // 페이지당 건수
}
```

또는 단건:
```json
{
  "user_id": "testuser1",
  "limit_bytes": 5368709120
}
```

### 8.2 에러 응답

```json
{
  "error": "NOT_FOUND",
  "message": "User not found",
  "detail": "user_id: testuser999"
}
```

HTTP 상태 코드:
| 코드 | 의미 |
|------|------|
| 200 | 성공 |
| 201 | 생성 성공 |
| 400 | 잘못된 요청 (파라미터 오류) |
| 401 | 인증 필요 |
| 403 | 권한 없음 (Admin 아닌데 Admin API 접근) |
| 404 | 리소스 없음 |
| 409 | 충돌 (의존성 미충족 등) |
| 500 | 서버 내부 오류 |
| 507 | 할당량 초과 |

---

## 9. 모듈 Admin API 구현 체크리스트

### Drive 모듈

- [ ] `GET /api/v1/admin/status` — 모듈 상태/연결/통계
- [ ] `GET /api/v1/admin/stats` — 스토리지 통계 (파일 유형별, 사용자별)
- [ ] `GET /api/v1/admin/quota` — 전체 할당량 목록 (페이지네이션)
- [ ] `GET /api/v1/admin/quota/:userId` — 사용자 할당량 상세
- [ ] `PUT /api/v1/admin/quota/:userId` — 할당량 설정
- [ ] `GET /api/v1/admin/activities` — 감사 로그 (필터/페이지네이션)
- [ ] `GET /api/v1/admin/settings` — 설정 조회
- [ ] `PUT /api/v1/admin/settings` — 설정 변경
- [ ] `GET /api/v1/admin/users` — 사용자 목록/사용량
- [ ] `GET /api/v1/admin/users/:userId/files` — 관리자 파일 조회
- [ ] `X-Polyon-Admin` 헤더 검사 미들웨어
- [ ] `settings` DB 테이블 + 마이그레이션
- [ ] Admin API Rust 테스트

---

## 참조

- [Module Spec](module-spec.md) — module.yaml 규격
- [Module UI Spec](module-ui-spec.md) — Console/Portal UI 통합
- [Module Lifecycle Spec](module-lifecycle-spec.md) — 설치/삭제 자동화
- [Integration Guide](integration-guide.md) — 인프라 연동
- [RFP Drive](rfp-drive.md) — Drive 기능 요구사항
