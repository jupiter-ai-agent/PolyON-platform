# PP Canvas (AFFiNE) — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims

```yaml
claims:
  - type: database
    config:
      name: canvas
  - type: objectStorage
    config:
      bucket: canvas
  - type: smtp
    config:
      domain: canvas
  - type: auth
    config:
      clientId: canvas
```

- Redis는 Foundation 공용 사용 (별도 claim 불필요)

## 2. 인증 (제1원칙)

- Keycloak OIDC SSO 강제 (AFFiNE 자체 LDAP 미지원)
- AD/LDAP는 Keycloak User Federation으로만 사용
- AFFiNE Admin UI 또는 ENV로 OIDC 설정 주입

## 3. 스토리지 (제7원칙)

- `AFFINE_STORAGE_PROVIDER: aws-s3` → RustFS
- 모든 문서/미디어/블롭 S3 저장
- 로컬 저장 금지

## 4. UI 통합 (제4원칙)

- AFFiNE 자체 UI → iframe (ModuleSector)
- Console: 관리자 설정
- Portal: 사원 문서/화이트보드

## 5. 환경변수

| 변수 | PRC Claim | 용도 |
|------|-----------|------|
| `DATABASE_URL` | database.url | PostgreSQL |
| `REDIS_SERVER_HOST/PORT` | (Foundation 공용) | Redis |
| `AFFINE_STORAGE_*` | objectStorage.* | RustFS |
| `MAILER_*` | smtp.* | 메일 |
| `OIDC_*` | auth.* | Keycloak |

## 6. 빌드 주의사항

- AFFiNE 소스 빌드 필수 (커스터마이징 필요)
- `docker-clean.mjs`의 `pruneServerNative()`가 `linux-x64-gnu.node` 삭제 → 복원 단계 필요
- arm64 빌드 시 추가 네이티브 모듈 확인 필요

## 7. 빌드/배포

```bash
cd PolyON-Canvas
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-canvas:v{VERSION} --push .

kubectl set image deploy/polyon-canvas canvas=jupitertriangles/polyon-canvas:v{VERSION} -n polyon
```
