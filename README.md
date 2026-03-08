# PolyON Platform

**PolyON**은 기업 인프라 통합 플랫폼입니다.

Active Directory 기반 계정 통합, S3 오브젝트 스토리지, SSO 인증을 제공하며,  
모든 업무 서비스(메일, 채팅, 드라이브, 위키 등)를 하나의 플랫폼 위에서 운영합니다.

## 문서

| 문서 | 설명 |
|------|------|
| [Platform Overview](docs/platform-overview.md) | PolyON 아키텍처와 설계 원칙 |
| [Module Spec](docs/module-spec.md) | 모듈 개발 스펙 — PP 위에서 동작하는 서비스 규격 |
| [Integration Guide](docs/integration-guide.md) | 인프라 서비스 연동 가이드 (AD, S3, DB, Search) |
| [Drive Module RFP](docs/rfp-drive.md) | Drive(파일 스토리지) 모듈 요구사항 정의서 |

## 핵심 원칙

1. **계정 통합** — AD DC가 유일한 계정 원천
2. **유기적 통합** — 계정 추가 시 모든 서비스 자동 프로비저닝
3. **통제와 감사** — 정책 기반 접근 제어, 감사 로그
4. **제품 품질 UI** — IBM Carbon Design System
5. **단일 관문** — Traefik Ingress가 유일한 외부 진입점
6. **인프라 단일 책임** — Operator가 인프라 관장
7. **오브젝트 스토리지 단일화** — 모든 모듈의 파일은 RustFS(S3)

## 런타임 환경

- **Orchestration:** Kubernetes
- **Ingress:** Traefik v3
- **Database:** PostgreSQL 18
- **Cache:** Redis 8
- **Object Storage:** RustFS (S3-compatible, MinIO fork)
- **Search:** OpenSearch 3.x
- **Identity:** Samba AD DC + Keycloak OIDC
- **TLS:** Self-signed wildcard (production: Let's Encrypt)

## 라이선스

Platform documentation: MIT  
Individual modules may have their own licenses.

---

**Triangle.s** — https://triangles.co.kr
