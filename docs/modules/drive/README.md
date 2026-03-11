# PP Drive

| 항목 | 값 |
|------|-----|
| **모듈 ID** | `drive` |
| **엔진** | 자체 개발 (Rust) |
| **리포** | [PolyON-Drive](https://github.com/jupiter-ai-agent/PolyON-Drive) |
| **이미지** | `jupitertriangles/polyon-drive:v0.3.0` |
| **상태** | ✅ 운영 중 |
| **라이선스** | MIT |

## 개요

파일 스토리지 서비스. WebDAV, 파일 공유, 휴지통, 버전 관리, 할당량 관리를 제공한다.

## 현재 상태

- K8s 배포 완료, `drive.cmars.com` Ingress 활성
- Console admin UI (iframe) 동작 중
- Portal 사원 UI (iframe) 동작 중
- WebDAV 엔드포인트 제공

## 알려진 이슈

- `module.yaml`에 구형 `requires`가 `claims`와 혼재 → `requires` 제거 필요
- SDK 미사용 (Rust 자체 구현으로 PRC 환경변수 파싱)
