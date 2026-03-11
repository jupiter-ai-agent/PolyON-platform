# PP Channel (Mattermost) — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims

```yaml
claims:
  - type: database
    config:
      name: channel
  - type: objectStorage
    config:
      bucket: channel
  - type: directory
    config:
      ou: channel
  - type: smtp
    config:
      domain: channel
  - type: auth
    config:
      clientId: channel
```

## 2. 인증 (제1원칙)

### 듀얼 패턴 — SSO + LDAP 병행

| 경로 | 방식 | 비고 |
|------|------|------|
| 웹 UI | Keycloak OIDC SSO | 기본 로그인 |
| 웹 UI | LDAP (AD) 직접 | 보조 (동일 AD 계정) |
| API | Bearer Token | SSO 토큰 |

- `MM_OAUTHSETTINGS_*` → Keycloak OIDC
- `MM_LDAPSETTINGS_*` → AD LDAP 직접 bind
- LDAP Sync: Enterprise 전용 → Core 커스텀 sync 구현 필요
- `IdAttribute`: `sAMAccountName` (objectGUID 바이너리 깨짐 주의)

### 사용자 생성

- SSO 첫 로그인 시 JIT 생성
- Core MattermostSync로 AD 사용자 일괄 동기화
- LDAP auth 사용자 생성 시 `password` 필드 보내면 안 됨 (`auth_data`만)

## 3. 스토리지 (제7원칙)

- `MM_FILESETTINGS_DRIVERNAME: amazons3` → RustFS
- 모든 첨부파일/프로필 사진 S3 저장
- `MM_FILESETTINGS_AMAZONS3SSL: false` (내부 네트워크)

## 4. UI 통합 (제4원칙)

- Mattermost 자체 UI → iframe (ModuleSector)
- Console: 관리자 System Console 접근
- Portal: 사원 채팅 인터페이스
- PostMessage 또는 `skipHandshake`

## 5. 환경변수

| 변수 패턴 | PRC Claim | 용도 |
|-----------|-----------|------|
| `MM_SQLSETTINGS_DATASOURCE` | database | PostgreSQL |
| `MM_FILESETTINGS_AMAZONS3*` | objectStorage | RustFS |
| `MM_LDAPSETTINGS_*` | directory | AD LDAP |
| `MM_EMAILSETTINGS_*` | smtp | 메일 |
| `MM_OAUTHSETTINGS_*` | auth | Keycloak OIDC |

## 6. 빌드 주의사항

- Mattermost 공식 이미지 arm64 없음 → 소스 빌드 필수
- webapp만 공식 이미지에서 추출, 서버는 Go 소스 빌드
- `go.work`에 go 1.25 필요 (Mattermost v11.5.1)
- GitLab: `gitlab.triangles.co.kr/cmars/polyon-chat.git`

## 7. 빌드/배포

```bash
cd PolyON-Channel
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-chat:v{VERSION} --push .

kubectl set image deploy/polyon-mattermost mattermost=jupitertriangles/polyon-chat:v{VERSION} -n polyon
```
