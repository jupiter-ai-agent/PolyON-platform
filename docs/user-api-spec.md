# PolyON Drive User API Spec — 사원용 API 규격

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 대상: 모듈 개발자 (Drive), Portal 개발자, AI 코딩 에이전트

---

## 1. 개요

이 문서는 Portal(사원 SPA)에서 Drive 모듈 백엔드를 호출하는 **사원용 REST API** 규격입니다.

Admin API(admin-api-spec.md)와 분리된 별도 API이며, 사원은 **자기 자신의 파일만** 조회/조작할 수 있습니다.

---

## 2. 인증 흐름

### 2.1 Portal iframe 모드 (운영)

```
사원 브라우저
  → Portal (Keycloak OIDC PKCE 로그인, polyon realm)
  → Portal이 JWT access_token 보유
  → Drive 메뉴 클릭 → ModuleSector(iframe) 생성
  → iframe 내 Module SDK 초기화 (polyon:ready → polyon:init)
  → SDK가 JWT 토큰 수신
  → API 호출 시 Authorization: Bearer {JWT} 자동 포함
  → Drive 백엔드가 JWT payload에서 사용자 식별
```

### 2.2 독립 모드 (개발/디버그)

```
브라우저에서 직접 Drive URL 접속
  → 로그인 폼 표시
  → Basic Auth (username:password) → LDAP 검증
  → API 호출 시 Authorization: Basic {base64} 포함
```

### 2.3 인증 헤더

| 방식 | 헤더 | 사용자 식별 |
|------|------|------------|
| **Bearer JWT** | `Authorization: Bearer {token}` | JWT payload의 `preferred_username` 또는 `sub` 클레임 |
| **Basic Auth** | `Authorization: Basic {base64(user:pass)}` | LDAP bind 검증 후 username |

모든 User API는 인증된 사용자 본인의 데이터만 반환합니다. 다른 사용자의 파일에 접근할 수 없습니다.

---

## 3. API 엔드포인트 목록

| Method | 경로 | 설명 | 섹션 |
|--------|------|------|------|
| GET | `/api/v1/files` | 루트 파일/폴더 목록 | §4 |
| GET | `/api/v1/files/{path}` | 하위 파일/폴더 목록 | §4 |
| GET | `/api/v1/trash` | 휴지통 목록 | §5 |
| POST | `/api/v1/trash/{id}/restore` | 휴지통 복원 | §5 |
| DELETE | `/api/v1/trash/{id}` | 영구 삭제 | §5 |
| DELETE | `/api/v1/trash` | 휴지통 비우기 | §5 |
| GET | `/api/v1/quota` | 내 사용량/할당량 | §6 |
| GET | `/api/v1/search?q={query}` | 파일 검색 | §7 |
| GET | `/api/v1/activities` | 내 활동 로그 | §8 |
| GET | `/api/v1/files/{path}/activities` | 특정 파일 활동 로그 | §8 |
| GET | `/api/v1/favorites` | 즐겨찾기 목록 | §9 |
| PUT | `/api/v1/files/{path}/favorite` | 즐겨찾기 추가 | §9 |
| DELETE | `/api/v1/files/{path}/favorite` | 즐겨찾기 제거 | §9 |
| GET | `/api/v1/files/{path}/versions` | 파일 버전 목록 | §10 |
| POST | `/api/v1/files/{path}/versions/{n}/restore` | 버전 복원 | §10 |
| POST | `/api/v1/shares` | 공유 생성 | §11 |
| GET | `/api/v1/shares` | 내가 공유한 목록 | §11 |
| GET | `/api/v1/shares/{id}` | 공유 상세 | §11 |
| PUT | `/api/v1/shares/{id}` | 공유 수정 | §11 |
| DELETE | `/api/v1/shares/{id}` | 공유 삭제 | §11 |
| GET | `/api/v1/shared-with-me` | 나에게 공유된 목록 | §11 |
| POST | `/api/v1/upload-url` | 업로드 WebDAV URL 발급 | §12 |
| POST | `/api/v1/download-url` | 다운로드 WebDAV URL 발급 | §12 |
| GET | `/s/{token}` | 공유 링크 정보 (비인증) | §13 |
| GET | `/s/{token}/download` | 공유 링크 다운로드 (비인증) | §13 |

---

## 4. 파일 목록 API

### `GET /api/v1/files`

루트 디렉토리의 파일/폴더 목록을 반환합니다.

### `GET /api/v1/files/{path}`

지정 경로의 하위 파일/폴더 목록을 반환합니다.

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `limit` | integer | 200 | 최대 반환 수 (1~1000) |
| `offset` | integer | 0 | 오프셋 |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "path": "documents/report.pdf",
      "name": "report.pdf",
      "is_directory": false,
      "size": 2457600,
      "updated_at": "2026-03-09T10:30:00+09:00"
    },
    {
      "path": "photos",
      "name": "photos",
      "is_directory": true,
      "size": 0,
      "updated_at": "2026-03-08T14:00:00+09:00"
    }
  ],
  "total": 15,
  "limit": 200,
  "offset": 0
}
```

**정렬:** 폴더 우선, 이름 알파벳순 (대소문자 무시)

**S3 폴백:** DB에 결과가 없으면 S3 `list_objects`로 보완합니다 (DB 동기화 지연 대비).

**경로 보안:** `../`, `//` 등 탐색 공격 경로는 `400 Bad Request` 반환.

---

## 5. 휴지통 API

### `GET /api/v1/trash`

현재 사용자의 휴지통 항목을 반환합니다.

**Query Parameters:** `limit` (기본 200), `offset` (기본 0)

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "original_path": "documents/old-report.pdf",
      "name": "old-report.pdf",
      "size": 1048576,
      "is_directory": false,
      "deleted_at": "2026-03-08T16:00:00+09:00"
    }
  ],
  "total": 3,
  "limit": 200,
  "offset": 0
}
```

### `POST /api/v1/trash/{id}/restore`

휴지통 항목을 원래 경로로 복원합니다.

**Response: `200 OK`**

```json
{ "status": "restored" }
```

**복원 동작:**
- `original_path`를 기반으로 원래 위치에 복원
- S3 오브젝트를 `.trash/` prefix에서 원래 key로 `copy_object`
- 파일 메타데이터(files 테이블) 재생성
- 활동 로그에 `restore` 기록

### `DELETE /api/v1/trash/{id}`

특정 항목을 영구 삭제합니다.

**Response: `204 No Content`**

### `DELETE /api/v1/trash`

휴지통 전체를 비웁니다 (모든 항목 영구 삭제).

**Response: `204 No Content`**

---

## 6. 할당량 API

### `GET /api/v1/quota`

현재 사용자의 스토리지 사용량과 할당량을 반환합니다.

**Response: `200 OK`**

```json
{
  "used": 5368709120,
  "limit": 10737418240
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `used` | integer (bytes) | 현재 사용량 |
| `limit` | integer (bytes) | 할당량 상한. 0이면 무제한 |

**UI 표시 예:** `5.0 GB / 10.0 GB (50%)`

---

## 7. 검색 API

### `GET /api/v1/search`

현재 사용자의 파일을 이름/경로로 검색합니다.

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `q` | string | (필수) | 검색어 |
| `limit` | integer | 100 | 최대 반환 수 (1~500) |

**Response: `200 OK`**

```json
[
  {
    "path": "documents/quarterly-report.pdf",
    "name": "quarterly-report.pdf",
    "size": 2457600,
    "is_directory": false,
    "mime_type": "application/pdf"
  }
]
```

**검색 방식:** DB `ILIKE` 패턴 매칭 (파일명 + 경로). Phase 2에서 OpenSearch FTS 전환 예정.

---

## 8. 활동 로그 API

### `GET /api/v1/activities`

현재 사용자의 활동 로그를 반환합니다.

**Query Parameters:** `limit` (기본 100), `offset` (기본 0)

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "path": "documents/report.pdf",
      "action": "upload",
      "detail": null,
      "created_at": "2026-03-09T10:30:00+09:00"
    }
  ],
  "total": 42,
  "limit": 100,
  "offset": 0
}
```

**action 값:** `upload`, `download`, `delete`, `move`, `copy`, `restore`, `mkcol`(폴더 생성)

### `GET /api/v1/files/{path}/activities`

특정 파일/폴더의 활동 로그를 반환합니다. 응답 형식은 동일합니다.

---

## 9. 즐겨찾기 API

### `GET /api/v1/favorites`

사용자의 즐겨찾기 목록을 반환합니다. 내부적으로 `files` 테이블과 INNER JOIN하여 **항상 최신 메타데이터**를 반환합니다. 원본 파일이 삭제된 경우 결과에서 자동 제외됩니다.

**응답:**

```json
[
  {
    "path": "documents/report.pdf",
    "name": "report.pdf",
    "is_directory": false,
    "size": 245760,
    "updated_at": "2026-03-09T10:30:00Z"
  }
]
```

### `PUT /api/v1/files/{path}/favorite`

파일 또는 폴더를 즐겨찾기에 추가합니다. 이미 등록된 경우 `ON CONFLICT DO NOTHING`으로 무시됩니다.

**응답:** `200 OK`

```json
{ "status": "added" }
```

### `DELETE /api/v1/files/{path}/favorite`

즐겨찾기에서 제거합니다.

**응답:** `204 No Content` (성공) / `404 Not Found` (미등록)

### 설계 원칙

- `favorites` 테이블에는 `(user_id, path)`만 저장 — 메타데이터 중복 없음
- 목록 조회 시 `files` 테이블과 INNER JOIN → 이름/크기 항상 최신
- `ON DELETE CASCADE` FK → 파일 영구 삭제 시 자동 정리
- `ON UPDATE CASCADE` FK → 파일 이동(경로 변경) 시 자동 갱신

---

## 10. 파일 버전 API

### `GET /api/v1/files/{path}/versions`

파일의 버전 목록을 반환합니다 (최대 50개).

**Response: `200 OK`**

```json
[
  {
    "version_number": 3,
    "size": 2457600,
    "created_at": "2026-03-09T10:30:00+09:00"
  },
  {
    "version_number": 2,
    "size": 2400000,
    "created_at": "2026-03-08T14:00:00+09:00"
  }
]
```

### `POST /api/v1/files/{path}/versions/{version_number}/restore`

지정 버전으로 파일을 복원합니다.

**동작:**
1. S3에서 버전 key(`{user}/.versions/{path}/v{n}`)를 현재 key로 `copy_object`
2. files 테이블 메타데이터 갱신 (size 등)

**Response: `200 OK`**

```json
{ "status": "restored" }
```

---

## 11. 공유 API

### `POST /api/v1/shares`

파일/폴더 공유를 생성합니다.

**Request Body:**

```json
{
  "path": "documents/report.pdf",
  "share_type": "link",
  "target_id": null,
  "permission": "read",
  "password": "optional-password",
  "expires_at": "2026-04-09T00:00:00Z"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `path` | string | ✅ | 공유할 파일/폴더 경로 |
| `share_type` | string | ✅ | `link` (공개 링크) 또는 `user` (특정 사용자) |
| `target_id` | string | `user`일 때 필수 | 대상 사용자 ID (sAMAccountName) |
| `permission` | string | — | `read` (기본), `write`, `manage` |
| `password` | string | — | 공유 링크 비밀번호 (argon2id 해시로 저장) |
| `expires_at` | string (RFC3339) | — | 만료 일시 |

**Response: `201 Created`**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "path": "documents/report.pdf",
  "share_type": "link",
  "token": "a1b2c3d4-e5f6-...",
  "target_id": null,
  "permission": "read",
  "url": "/s/a1b2c3d4-e5f6-..."
}
```

### `GET /api/v1/shares`

내가 생성한 공유 목록을 반환합니다.

**Response: `200 OK`**

```json
[
  {
    "id": "550e8400-...",
    "path": "documents/report.pdf",
    "share_type": "link",
    "token": "a1b2c3d4-...",
    "target_id": null,
    "permission": "read",
    "has_password": true,
    "expires_at": "2026-04-09T00:00:00Z",
    "created_at": "2026-03-09T10:30:00+09:00"
  }
]
```

### `GET /api/v1/shares/{id}`

공유 상세 정보를 반환합니다. 응답은 목록의 단일 항목과 동일합니다.

### `PUT /api/v1/shares/{id}`

공유 설정을 수정합니다.

**Request Body:**

```json
{
  "permission": "write",
  "password": "new-password",
  "expires_at": "2026-05-01T00:00:00Z"
}
```

모든 필드 선택적. `password`가 빈 문자열이면 비밀번호 제거.

**Response: `200 OK`**

```json
{ "status": "updated" }
```

### `DELETE /api/v1/shares/{id}`

공유를 삭제합니다.

**Response: `204 No Content`**

### `GET /api/v1/shared-with-me`

나에게 공유된 파일 목록을 반환합니다.

**Response: `200 OK`**

```json
[
  {
    "id": "550e8400-...",
    "owner_id": "testuser2",
    "path": "projects/design.fig",
    "permission": "read",
    "created_at": "2026-03-08T14:00:00+09:00"
  }
]
```

---

## 12. 파일 업로드/다운로드 URL API

파일 바이너리 전송은 Core API 프록시를 경유하지 않습니다.
사원이 직접 모듈의 WebDAV 엔드포인트에 업로드/다운로드합니다.

### `POST /api/v1/upload-url`

업로드용 WebDAV URL을 발급합니다.

**Request Body:**

```json
{
  "path": "documents/new-report.pdf"
}
```

**Response: `200 OK`**

```json
{
  "url": "/dav/files/testuser1/documents/new-report.pdf",
  "method": "PUT"
}
```

**클라이언트 사용:**

```javascript
const { url, method } = await sdk.api.post('/api/v1/upload-url', { path: 'docs/report.pdf' });
await fetch(url, {
  method: method,                    // "PUT"
  headers: { Authorization: `Bearer ${token}` },
  body: fileBlob,
});
```

### `POST /api/v1/download-url`

다운로드용 WebDAV URL을 발급합니다.

**Request Body:**

```json
{
  "path": "documents/report.pdf"
}
```

**Response: `200 OK`**

```json
{
  "url": "/dav/files/testuser1/documents/report.pdf",
  "method": "GET"
}
```

### WebDAV 직접 접근

URL 발급 없이 직접 WebDAV 엔드포인트를 호출할 수도 있습니다:

| 동작 | Method | 경로 |
|------|--------|------|
| 업로드 | `PUT` | `/dav/files/{username}/{path}` |
| 다운로드 | `GET` | `/dav/files/{username}/{path}` |
| 폴더 생성 | `MKCOL` | `/dav/files/{username}/{path}` |
| 삭제 | `DELETE` | `/dav/files/{username}/{path}` |
| 이동/이름변경 | `MOVE` | `/dav/files/{username}/{path}` + `Destination` 헤더 |
| 복사 | `COPY` | `/dav/files/{username}/{path}` + `Destination` 헤더 |
| 속성 조회 | `PROPFIND` | `/dav/files/{username}/{path}` |
| 잠금 | `LOCK` | `/dav/files/{username}/{path}` |
| 잠금 해제 | `UNLOCK` | `/dav/files/{username}/{path}` + `Lock-Token` 헤더 |

**인증:** WebDAV 요청에도 `Authorization: Bearer {JWT}` 또는 `Basic` 헤더 필수. 경로의 `{username}`과 인증된 사용자가 일치해야 합니다 (다른 사용자 파일 접근 차단).

---

## 13. 공유 링크 API (비인증)

공유 링크(`/s/{token}`)는 인증 없이 접근 가능합니다.
비밀번호 보호된 공유는 추가 인증이 필요합니다.

### `GET /s/{token}`

공유 링크 정보를 반환합니다.

**비밀번호 전달 방식 (택1):**
- Query: `?password=secret`
- Header: `X-Password: secret`

**Response: `200 OK`**

```json
{
  "owner_id": "testuser1",
  "path": "documents/report.pdf",
  "permission": "read",
  "expires_at": "2026-04-09T00:00:00Z",
  "download_url": "/s/a1b2c3d4-e5f6-.../download"
}
```

**에러 응답:**
- `401 Unauthorized` + `X-Requires-Password: true` — 비밀번호 필요
- `403 Forbidden` + `X-Requires-Password: true` — 비밀번호 틀림
- `404 Not Found` — 토큰 없거나 만료

### `GET /s/{token}/download`

공유 파일을 다운로드합니다.

**비밀번호:** 위와 동일하게 query 또는 헤더로 전달.

**Response: `200 OK`**

```
Content-Type: application/octet-stream
Content-Disposition: attachment; filename*=UTF-8''report.pdf

{binary data}
```

---

## 14. 공통 에러 응답

모든 API는 에러 시 일관된 JSON 형식을 반환합니다:

```json
{
  "error": "에러 메시지"
}
```

| 상태 코드 | 의미 |
|-----------|------|
| `400` | 잘못된 요청 (경로 오류, 필수 파라미터 누락) |
| `401` | 인증 필요 (Authorization 헤더 없음 또는 만료) |
| `403` | 권한 없음 (다른 사용자 파일 접근, 잘못된 비밀번호) |
| `404` | 리소스 없음 |
| `500` | 서버 내부 오류 |
| `503` | 서비스 불가 (DB 또는 S3 연결 실패) |

---

## 15. 모듈 UI 연동 가이드

### 15.1 Portal Drive UI (iframe)에서 API 호출

```javascript
// Module SDK 초기화
const sdk = PolyonSDK.init();
await sdk.ready;

// 파일 목록 조회
const files = await sdk.api.get('/api/v1/files');

// 하위 폴더 탐색
const docs = await sdk.api.get('/api/v1/files/documents');

// 파일 업로드
const { url } = await sdk.api.post('/api/v1/upload-url', { path: 'docs/new.pdf' });
await fetch(url, {
  method: 'PUT',
  headers: { Authorization: `Bearer ${sdk.auth.getToken()}` },
  body: fileBlob,
});

// 공유 생성
const share = await sdk.api.post('/api/v1/shares', {
  path: 'docs/report.pdf',
  share_type: 'link',
  permission: 'read',
});
// share.url → "/s/a1b2c3d4-..."

// 휴지통 복원
await sdk.api.post('/api/v1/trash/550e8400-.../restore');

// 할당량 표시
const quota = await sdk.api.get('/api/v1/quota');
// → { used: 5368709120, limit: 10737418240 }
```

### 15.2 Portal 페이지 ↔ API 매핑

| Portal 페이지 | 주요 API |
|--------------|---------|
| **드라이브** (파일 탐색기) | `GET /api/v1/files`, WebDAV PUT/MKCOL/DELETE |
| **공유** | `GET /api/v1/shared-with-me`, `GET /api/v1/shares` |
| **최근** | `GET /api/v1/activities?limit=50` → 최근 파일 추출 |
| **즐겨찾기** | `GET /api/v1/favorites`, `PUT/DELETE /api/v1/files/{path}/favorite` |
| **휴지통** | `GET /api/v1/trash`, `POST restore`, `DELETE` |

---

## 참조

- [Admin API Spec](admin-api-spec.md) — 관리자 API 규격
- [Module UI Spec](module-ui-spec.md) — 하이브리드 A/C 아키텍처
- [Module SDK Spec](module-sdk-spec.md) — @polyon/module-sdk
- [Module Spec](module-spec.md) — 모듈 기본 규격
- [RFP Drive](rfp-drive.md) — Drive 기능 요구사항
