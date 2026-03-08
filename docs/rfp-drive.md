# PolyON Drive — 기능 요구사항 정의서

## 1. 개요

Nextcloud(PHP)를 대체하는 **Rust 기반 파일 스토리지 서비스**.
PP(PolyON Platform) 인프라에 네이티브하게 통합됩니다.

### 제외 기능 (명시적)
- ❌ CalDAV / CardDAV
- ❌ Plugin / App 시스템
- ❌ App Store
- ❌ External Storage 마운트
- ❌ 영상 통화 / 채팅
- ❌ 메일

---

## 2. 인증 · 사용자 관리

### 2.1 AD DC LDAP (필수 — Native 연동)

| 항목 | 요구사항 |
|------|----------|
| 프로토콜 | LDAP v3, plain (port 389, K8s 내부) |
| 인증 방식 | LDAP Simple Bind (사용자 DN + 비밀번호) |
| 사용자 검색 | sAMAccountName, mail, userPrincipalName |
| 사용자 프로필 | displayName, mail, memberOf 자동 매핑 |
| 그룹 동기화 | AD 보안 그룹 → Drive 그룹 (공유/권한 용) |
| 계정 상태 | userAccountControl 비트로 비활성 계정 자동 차단 |
| 시스템 계정 필터 | admin, administrator, krbtgt 등 제외 |
| 자동 홈폴더 | 첫 로그인 시 사용자 루트 폴더 자동 생성 |
| 실시간 반영 | AD에서 계정 삭제/비활성 → Drive 접근 즉시 차단 |

### 2.2 Keycloak OIDC (필수 — Native 연동)

| 항목 | 요구사항 |
|------|----------|
| 프로토콜 | OpenID Connect 1.0 |
| Flow | Authorization Code + PKCE (SPA), Authorization Code (서버) |
| Discovery | `/.well-known/openid-configuration` 자동 |
| Token 검증 | JWT 서명 검증 (Keycloak JWKS) |
| 사용자 매칭 | OIDC `preferred_username` ↔ LDAP `sAMAccountName` |
| SSO | Portal/Console 로그인 상태로 Drive 자동 접근 |
| SLO | Single Logout 지원 (end_session_endpoint) |
| Refresh Token | Access Token 만료 시 자동 갱신 |

### 2.3 이중 인증 구조

```
브라우저 접근 → Keycloak OIDC (SSO)
WebDAV 접근  → LDAP Direct Bind (ID/PW)
API 접근     → JWT Bearer Token (Keycloak 발급)
```

---

## 3. 파일 관리

### 3.1 기본 파일 작업

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 업로드 | 단일 파일, 멀티파트, 청크(chunked) 업로드 | **P0** |
| 다운로드 | 단일 파일, Range 요청 (이어받기), 폴더 ZIP 다운로드 | **P0** |
| 삭제 | 파일/폴더 삭제 → 휴지통 이동 | **P0** |
| 이동 | 파일/폴더 위치 이동 | **P0** |
| 복사 | 파일/폴더 복사 | **P0** |
| 이름 변경 | 파일/폴더 이름 변경 | **P0** |
| 폴더 생성 | 중첩 폴더 구조 생성 | **P0** |

### 3.2 대용량 파일 지원

| 항목 | 요구사항 |
|------|----------|
| 최대 파일 크기 | 제한 없음 (S3 멀티파트로 처리) |
| 청크 업로드 | 5MB 단위 청크, 실패 시 해당 청크만 재전송 |
| 이어받기 | HTTP Range 헤더 지원 |
| 진행률 | 업로드/다운로드 진행률 API |
| 동시 업로드 | 여러 파일 병렬 업로드 |

### 3.3 파일 메타데이터

| 필드 | 설명 |
|------|------|
| 이름 | 파일/폴더 이름 (UTF-8) |
| 크기 | 바이트 단위 |
| MIME Type | 자동 감지 |
| 생성일 | 최초 업로드 시각 |
| 수정일 | 마지막 수정 시각 |
| ETag | 변경 감지용 해시 |
| 소유자 | sAMAccountName |
| 경로 | 전체 경로 (/ 구분) |

---

## 4. WebDAV (필수)

### 4.1 지원 메서드

| 메서드 | 설명 | RFC |
|--------|------|-----|
| `PROPFIND` | 파일/폴더 속성 조회 (Depth: 0, 1, infinity) | 4918 |
| `PROPPATCH` | 속성 수정 | 4918 |
| `GET` | 파일 다운로드 | 4918 |
| `PUT` | 파일 업로드 | 4918 |
| `MKCOL` | 폴더 생성 | 4918 |
| `DELETE` | 파일/폴더 삭제 | 4918 |
| `MOVE` | 이동 | 4918 |
| `COPY` | 복사 | 4918 |
| `HEAD` | 메타데이터 조회 | 4918 |
| `OPTIONS` | 지원 메서드 조회 | 4918 |
| `LOCK` / `UNLOCK` | 파일 잠금 (편집 충돌 방지) | 4918 |

### 4.2 WebDAV 엔드포인트

```
https://drive.{domain}/dav/files/{username}/
```

### 4.3 클라이언트 호환성 (필수)

| 클라이언트 | 검증 |
|-----------|------|
| macOS Finder | WebDAV 마운트 동작 |
| Windows 탐색기 | WebDAV 드라이브 매핑 |
| Linux (nautilus/dolphin) | WebDAV 마운트 |
| Cyberduck | 연결 및 파일 조작 |
| rclone | 마운트 및 동기화 |

### 4.4 WebDAV 인증

```
Authorization: Basic {base64(username:password)}
```
- LDAP Direct Bind로 검증
- Bearer Token도 허용 (JWT)

---

## 5. 파일 공유

### 5.1 공유 링크 (Public Sharing)

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 링크 생성 | 고유 URL 생성 (/s/{token}) | **P1** |
| 만료일 | 자동 만료 설정 | **P1** |
| 비밀번호 보호 | 접근 시 비밀번호 요구 | **P1** |
| 권한 설정 | 읽기 전용 / 읽기+쓰기 / 업로드 전용 | **P1** |
| 다운로드 횟수 제한 | N회 다운로드 후 자동 만료 | **P2** |

### 5.2 사용자/그룹 공유 (Internal Sharing)

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| AD 사용자에게 공유 | sAMAccountName으로 공유 | **P1** |
| AD 그룹에 공유 | AD 보안 그룹 단위 공유 | **P1** |
| 권한 | 읽기 / 쓰기 / 관리(reshare) | **P1** |
| 공유 해제 | 공유 취소 | **P1** |
| 공유 받은 파일 | "공유 받은 파일" 가상 폴더 | **P1** |

### 5.3 팀 폴더 (Team Folders)

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 관리자 생성 | Console에서 팀 폴더 생성 | **P2** |
| AD 그룹 바인딩 | 특정 그룹만 접근 가능 | **P2** |
| 권한 상속 | 하위 파일/폴더에 권한 전파 | **P2** |
| 할당량 | 팀 폴더별 용량 제한 | **P2** |

---

## 6. 버전 관리

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 자동 버저닝 | 파일 덮어쓰기 시 이전 버전 자동 보관 | **P2** |
| 버전 목록 | 파일별 버전 히스토리 조회 | **P2** |
| 버전 복원 | 이전 버전으로 되돌리기 | **P2** |
| 버전 다운로드 | 특정 버전 다운로드 | **P2** |
| 보관 정책 | N일 이후 자동 삭제, 최대 N개 유지 | **P3** |

---

## 7. 휴지통

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 자동 이동 | 삭제 시 즉시 삭제 아닌 휴지통 이동 | **P1** |
| 목록 조회 | 삭제된 파일 목록, 삭제일, 원래 경로 표시 | **P1** |
| 복원 | 원래 경로로 복원 | **P1** |
| 영구 삭제 | 휴지통에서 영구 삭제 | **P1** |
| 자동 비우기 | 30일(설정 가능) 후 자동 영구 삭제 | **P2** |

---

## 8. 할당량 (Quota)

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 사용자별 할당량 | 관리자가 사용자별 용량 제한 설정 | **P2** |
| 그룹별 기본 할당량 | AD 그룹별 기본 할당량 | **P2** |
| 사용량 조회 | 현재 사용량 / 할당량 API | **P1** |
| 초과 방지 | 할당량 초과 시 업로드 거부 | **P2** |

---

## 9. 검색

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 파일명 검색 | 파일/폴더 이름으로 검색 | **P1** |
| 전문 검색 | 파일 내용 검색 (PDF, TXT, DOCX) | **P2** |
| 필터 | 파일 유형, 크기, 날짜, 소유자 필터 | **P2** |
| 검색 범위 | 내 파일 / 공유 파일 / 전체 | **P2** |

OpenSearch 연동: 모듈이 자체적으로 인덱싱

---

## 10. 미리보기 · 썸네일

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 이미지 썸네일 | JPEG/PNG/GIF/WebP → 작은 썸네일 자동 생성 | **P2** |
| PDF 미리보기 | 첫 페이지 이미지 추출 | **P3** |
| 비디오 썸네일 | 첫 프레임 추출 | **P3** |
| EXIF | 이미지 EXIF 메타데이터 표시 | **P3** |

---

## 11. 활동 로그 · 감사

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 활동 로그 | 업로드/다운로드/삭제/공유/이동 기록 | **P1** |
| 사용자별 조회 | 내 활동 히스토리 | **P1** |
| 관리자 감사 | 전체 사용자 활동 조회 (Console) | **P2** |
| 파일별 히스토리 | 특정 파일의 전체 활동 이력 | **P2** |

---

## 12. 즐겨찾기 · 태그

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 즐겨찾기 | 파일/폴더 즐겨찾기 등록/해제 | **P2** |
| 태그 | 파일에 태그 추가 (개인/공유) | **P3** |
| 태그 검색 | 태그로 파일 필터링 | **P3** |

---

## 13. 파일 잠금 (Locking)

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 수동 잠금 | 편집 중 파일 잠금 | **P2** |
| WebDAV LOCK | RFC 4918 LOCK/UNLOCK | **P2** |
| 잠금 소유자 표시 | "홍길동이 편집 중" 표시 | **P2** |
| 자동 잠금 해제 | 타임아웃 후 자동 해제 | **P2** |

---

## 14. 알림

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 공유 알림 | "홍길동이 report.pdf를 공유했습니다" | **P2** |
| 변경 알림 | 공유 파일 수정 시 알림 | **P3** |
| 할당량 경고 | 80%/90% 사용 시 알림 | **P3** |

PP Core 알림 API 연동 (향후)

---

## 15. 관리자 기능 (Console API)

PP Console에서 관리하는 기능들:

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 사용자 파일 조회 | 관리자가 전체 사용자 파일 조회 | **P1** |
| 할당량 관리 | 사용자/그룹별 할당량 설정 | **P2** |
| 팀 폴더 관리 | 팀 폴더 CRUD | **P2** |
| 스토리지 통계 | 전체 사용량, 파일 유형별 통계 | **P1** |
| 활동 감사 | 전체 파일 활동 감사 로그 | **P2** |

---

## 16. 스토리지 백엔드

### RustFS (S3-compatible) — 유일한 백엔드

| 항목 | 요구사항 |
|------|----------|
| API | S3-compatible (AWS SDK) |
| 접속 | `http://polyon-rustfs:9000` |
| 인증 | Access Key / Secret Key |
| 경로 스타일 | Path-style (필수) |
| 버킷 | `drive` (모듈 전용) |
| 키 구조 | `{username}/{path}` |
| 멀티파트 | 5MB 청크, S3 Multipart Upload |
| 버저닝 | S3 Object Versioning (P2) |

### PostgreSQL — 메타데이터

| 항목 | 저장 내용 |
|------|----------|
| files | 파일/폴더 트리, 메타데이터 |
| shares | 공유 정보 |
| versions | 버전 히스토리 |
| trash | 휴지통 |
| activities | 활동 로그 |
| favorites | 즐겨찾기 |
| locks | 파일 잠금 |
| quotas | 할당량 설정 |

---

## 17. API 엔드포인트 요약

```
# 인증
POST   /api/v1/auth/login          # LDAP 로그인 → JWT
POST   /api/v1/auth/refresh         # Token 갱신
GET    /api/v1/auth/userinfo        # 현재 사용자 정보

# 파일
GET    /api/v1/files/{path}         # 파일 목록 / 메타데이터
PUT    /api/v1/files/{path}         # 업로드
GET    /api/v1/files/{path}?dl=1    # 다운로드
DELETE /api/v1/files/{path}         # 삭제 (→ 휴지통)
MOVE   /api/v1/files/{path}         # 이동
COPY   /api/v1/files/{path}         # 복사
MKCOL  /api/v1/files/{path}         # 폴더 생성

# 청크 업로드
POST   /api/v1/uploads              # 업로드 세션 시작
PUT    /api/v1/uploads/{id}/{chunk}  # 청크 전송
POST   /api/v1/uploads/{id}/finish   # 업로드 완료

# 공유
POST   /api/v1/shares               # 공유 생성
GET    /api/v1/shares               # 공유 목록
GET    /api/v1/shares/{id}          # 공유 상세
PUT    /api/v1/shares/{id}          # 공유 수정
DELETE /api/v1/shares/{id}          # 공유 삭제
GET    /api/v1/shared-with-me       # 공유 받은 파일

# 공유 링크 접근 (비인증)
GET    /s/{token}                   # 공유 링크 페이지
GET    /s/{token}/download          # 공유 파일 다운로드

# 휴지통
GET    /api/v1/trash                # 휴지통 목록
POST   /api/v1/trash/{id}/restore   # 복원
DELETE /api/v1/trash/{id}           # 영구 삭제
DELETE /api/v1/trash                # 휴지통 비우기

# 버전
GET    /api/v1/files/{path}/versions      # 버전 목록
GET    /api/v1/files/{path}/versions/{v}   # 버전 다운로드
POST   /api/v1/files/{path}/versions/{v}/restore  # 복원

# 검색
GET    /api/v1/search?q={query}     # 파일 검색

# 즐겨찾기
GET    /api/v1/favorites            # 즐겨찾기 목록
PUT    /api/v1/files/{path}/favorite      # 즐겨찾기 추가
DELETE /api/v1/files/{path}/favorite      # 즐겨찾기 해제

# 활동
GET    /api/v1/activities           # 활동 로그
GET    /api/v1/files/{path}/activities    # 파일별 활동

# 사용량
GET    /api/v1/quota                # 사용량/할당량

# WebDAV
*      /dav/files/{username}/       # WebDAV 전체 메서드

# 헬스
GET    /health                      # K8s probe
```

---

## 18. 비기능 요구사항

| 항목 | 요구사항 |
|------|----------|
| 언어 | Rust (latest stable) |
| 이미지 크기 | < 50MB (scratch/alpine) |
| 메모리 | < 128MB idle, < 512MB peak |
| 시작 시간 | < 3초 |
| 동시 사용자 | 100+ |
| 파일 크기 | 무제한 (S3 멀티파트) |
| 아키텍처 | linux/amd64 + linux/arm64 |
| 로깅 | JSON structured (tracing) |
| 설정 | 환경변수만 |
| Graceful Shutdown | SIGTERM → 진행 중 요청 완료 후 종료 |

---

## 19. Phase 계획

### Phase 1 — MVP (P0)
- 파일 CRUD (업로드/다운로드/삭제/이동/복사)
- 폴더 CRUD
- AD LDAP 인증
- Keycloak OIDC 인증
- RustFS S3 백엔드
- PostgreSQL 메타데이터
- Health endpoint
- 기본 WebDAV (PROPFIND, GET, PUT, MKCOL, DELETE, MOVE, COPY)

### Phase 2 — 공유 + 필수 기능 (P1)
- 공유 링크 (만료, 비밀번호)
- 사용자/그룹 공유
- 휴지통
- 파일명 검색
- 활동 로그
- 사용량 API
- WebDAV LOCK/UNLOCK
- 파일 잠금

### Phase 3 — 고급 기능 (P2)
- 파일 버전 관리
- 전문 검색 (OpenSearch)
- 썸네일 생성
- 팀 폴더
- 할당량 관리
- 즐겨찾기/태그
- 관리자 감사 로그
- 알림

---

## 20. 참고

- [Platform Overview](platform-overview.md) — PP 아키텍처
- [Module Spec](module-spec.md) — 모듈 개발 규격
- [Integration Guide](integration-guide.md) — AD/S3/DB/Search Rust 코드 예제
