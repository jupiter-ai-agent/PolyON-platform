# PP Channel

| 항목 | 값 |
|------|-----|
| **모듈 ID** | `mattermost` (배포 시 `chat`) |
| **엔진** | Mattermost Team Edition |
| **리포** | [PolyON-Channel](https://github.com/jupiter-ai-agent/PolyON-Channel) |
| **이미지** | `jupitertriangles/polyon-chat:v0.2.0` |
| **상태** | ✅ 운영 중 |
| **라이선스** | MIT (래퍼), Mattermost AGPL |

## 개요

협업 채팅·워크플로우 플랫폼. Mattermost를 PRC로 통합.

## 현재 상태

- K8s 배포 완료
- PRC claims 프로비저닝 (database, objectStorage, smtp, auth, directory)
- Keycloak SSO + LDAP 듀얼 패턴
- 소스 빌드 (공식 이미지 arm64 미지원)

## 알려진 이슈

| 이슈 | 우선순위 |
|------|---------|
| module.yaml 이미지 `latest` → 버전 고정 필요 | 🟡 |
| LDAP Sync Enterprise 전용 → Core 커스텀 sync | 🟡 |
| module ID `mattermost` ≠ 배포명 `chat` | 🟡 |
