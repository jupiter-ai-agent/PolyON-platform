# PP Canvas

| 항목 | 값 |
|------|-----|
| **모듈 ID** | `canvas` |
| **엔진** | AFFiNE |
| **리포** | [PolyON-Canvas](https://github.com/jupiter-ai-agent/PolyON-Canvas) |
| **이미지** | 미정 (AFFiNE stable 기반) |
| **상태** | ⛔ 미배포 (stub) |
| **라이선스** | MIT |

## 개요

지식베이스·화이트보드·문서 통합 플랫폼. AFFiNE을 PRC로 통합 예정.

## 현재 상태

- module.yaml, Dockerfile, entrypoint.sh만 존재
- 실제 빌드/배포 안 됨
- AFFiNE 소스 빌드 시 `docker-clean.mjs` 이슈 있음 (MEMORY.md 참조)

## TODO

- [ ] AFFiNE 소스 빌드 환경 구성
- [ ] OIDC SSO 연동
- [ ] S3 스토리지 연동
- [ ] Console/Portal iframe 통합
