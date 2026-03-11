# PP Drive — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims

```yaml
claims:
  - type: database
    config:
      name: drive
  - type: objectStorage
    config:
      bucket: drive
  - type: auth
    config:
      clientId: drive
```

- ~~`requires`~~ 제거 필수 → `claims`만 사용
- `directory` claim 선택 (LDAP bind 인증 시)

## 2. 인증 (제1원칙)

| 경로 | 방식 | 비고 |
|------|------|------|
| 웹 UI | Keycloak OIDC Bearer | Console/Portal iframe에서 토큰 전달 |
| WebDAV | LDAP bind (Basic Auth) | WebDAV 클라이언트 호환 (예외 승인됨) |
| API | OIDC Bearer 또는 Basic Auth | 듀얼 패턴 |

- OIDC JWT 검증: `OIDC_JWKS_URI` (내부 K8s URL)
- LDAP bind: `LDAP_URL`, `LDAP_BASE_DN`

## 3. 스토리지 (제7원칙)

- 모든 사용자 파일은 RustFS(S3) 버킷 `drive`에 저장
- 메타데이터(파일 정보, 공유, 휴지통)는 PostgreSQL
- 로컬 PVC 파일 저장 금지

## 4. UI 통합 (제4원칙)

### Console (관리자)

| 페이지 | 경로 | 기능 |
|--------|------|------|
| 개요 | `#/overview` | 대시보드 (전체 용량, 사용자 수) |
| 스토리지 | `#/storage` | 버킷/파일 관리 |
| 할당량 | `#/quota` | 사용자별 할당량 설정 |
| 활동 로그 | `#/activity` | 파일 변경 이력 |
| 설정 | `#/settings` | 시스템 설정 |

### Portal (사원)

| 페이지 | 경로 | 기능 |
|--------|------|------|
| 드라이브 | `#/` | 파일 탐색기 |
| 공유 | `#/shared` | 공유된 파일 |
| 최근 | `#/recent` | 최근 파일 |
| 즐겨찾기 | `#/favorites` | 즐겨찾기 |
| 휴지통 | `#/trash` | 휴지통 |

- PostMessage 프로토콜 구현 (`polyon:ready` → `polyon:init`)
- Carbon Design은 iframe 내부 자체 UI

## 5. API

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/health` | GET | K8s probe |
| `/api/files` | CRUD | 파일 관리 REST API |
| `/api/shares` | CRUD | 공유 관리 |
| `/api/trash` | CRUD | 휴지통 관리 |
| `/api/quota` | GET/PUT | 할당량 조회/설정 |
| `/webdav/` | WebDAV | WebDAV 엔드포인트 |

## 6. 환경변수

PRC가 주입하는 환경변수:

| 변수 | PRC Claim | 용도 |
|------|-----------|------|
| `DATABASE_URL` | database.url | PostgreSQL 연결 |
| `S3_ENDPOINT` | objectStorage.endpoint | RustFS API |
| `S3_BUCKET` | objectStorage.bucket | 버킷명 |
| `S3_ACCESS_KEY` | objectStorage.accessKey | 인증 |
| `S3_SECRET_KEY` | objectStorage.secretKey | 인증 |
| `OIDC_ISSUER` | auth.issuer | OIDC 발급자 |
| `OIDC_JWKS_URI` | auth.jwksUri | JWT 검증 |
| `LDAP_URL` | directory.url | LDAP 서버 |
| `LDAP_BASE_DN` | directory.baseDN | 검색 base |

## 7. 빌드/배포

```bash
# 빌드 (Mac Mini에서만)
cd PolyON-Drive
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-drive:v{VERSION} --push .

# 배포
kubectl set image deploy/polyon-drive drive=jupitertriangles/polyon-drive:v{VERSION} -n polyon
```

- 이미지 태그: `v{SEMVER}` (latest 금지)
- 멀티플랫폼: `linux/amd64` + `linux/arm64`
