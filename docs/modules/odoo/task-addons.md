# PP Odoo Addon 개발 작업 지시서

> 대상: Cursor / Sonnet / 코딩 에이전트
> 작성: Jupiter (AI 팀장) | 2026-03-10
> 소스 위치: 별도 리포 또는 Odoo 이미지 빌드 내 `addons-custom/`

---

## 목표

Odoo 19를 PP(PolyON Platform)의 ERP 모듈로 통합하기 위한 **커스텀 addon 3개**를 구현한다.

**핵심 원칙:**
- Odoo core 소스 수정 금지 — addon만으로 구현
- res.users 테이블은 유지하되 AD의 읽기 전용 미러로 활용
- Keycloak SSO가 유일한 로그인 경로
- 파일은 RustFS(S3)에 저장
- PP Console/Portal에서 iframe으로 표시

---

## 참조 문서

아래 문서를 반드시 읽고 작업할 것:

| 문서 | 위치 | 핵심 내용 |
|------|------|---------|
| **PP Odoo 설계서** | `docs/rfp-odoo.md` | addon 구조, 사용자 잠금, 인증 흐름, S3 연동 |
| **PRC Provider Reference** | `docs/prc-provider-reference.md` | 환경변수 매핑 (auth, database, objectStorage, smtp) |
| **Module Spec** | `docs/module-spec.md` | module.yaml 규격, PRC claim 선언 |

---

## 환경 정보

### Odoo 버전
- **Odoo 19.0** (`(19, 0, 0, 'final', 0, '')`)
- Python 3.12+
- PostgreSQL 18

### K8s 환경
- Namespace: `polyon`
- Pod: `polyon-odoo-*`
- Service: `polyon-odoo:8069`
- Ingress: `odoo.cmars.com` → `polyon-odoo:8069`
- Secret: `polyon-module-odoo` (PRC가 자동 생성)

### Keycloak 환경
- Realm: `polyon`
- SSO 도메인: `sso.cmars.com` (외부) / `polyon-auth:8080` (내부)
- Client ID: `odoo` (PRC auth가 자동 생성)
- Client Type: `confidential`

### PRC 환경변수 (Pod에 자동 주입됨)

```bash
# Auth (Keycloak OIDC)
OIDC_ISSUER=https://sso.cmars.com/realms/polyon
OIDC_CLIENT_ID=odoo
OIDC_CLIENT_SECRET=8Y57hxiUCuHsB04FT0Gxf4GYbdpL0KkL
OIDC_AUTH_ENDPOINT=https://sso.cmars.com/realms/polyon/protocol/openid-connect/auth
OIDC_TOKEN_ENDPOINT=http://polyon-auth:8080/realms/polyon/protocol/openid-connect/token
OIDC_JWKS_URI=http://polyon-auth:8080/realms/polyon/protocol/openid-connect/certs

# Database
DB_HOST=polyon-db
DB_PORT=5432
DB_NAME=polyon_odoo
DB_USER=mod_odoo
DB_PASSWORD=(자동생성)

# S3 (RustFS)
AWS_HOST=http://polyon-rustfs:9000
AWS_BUCKET_NAME=odoo
AWS_ACCESS_KEY_ID=(자동생성)
AWS_SECRET_ACCESS_KEY=(자동생성)

# SMTP (Stalwart)
SMTP_HOST=polyon-mail
SMTP_PORT=587
SMTP_USER=(자동생성)
SMTP_PASSWORD=(자동생성)
```

### DB 사용자 (현재)
- uid=1: `__system__` (active=true)
- uid=2: `admin` (active=true, login=admin)
- uid=3: `public` (inactive)
- uid=4: `portaltemplate` (inactive)

---

## 구현 대상: addon 3개

### Addon 1: `polyon_oidc`

**역할:** Keycloak SSO 인증 + Odoo 내 사용자 직접 관리 차단

#### 파일 구조

```
polyon_oidc/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── res_users.py          # res.users 잠금 (create/write/unlink override)
│   └── res_users_auth.py     # 비밀번호 로그인 차단 (_check_credentials)
├── controllers/
│   ├── __init__.py
│   └── oidc.py               # OIDC callback (필요 시)
├── data/
│   └── oauth_provider.xml    # auth.oauth.provider 레코드 (또는 post_init_hook)
└── views/
    ├── hide_menus.xml         # 사용자 관리 메뉴 숨김
    └── login_template.xml     # 로그인 페이지: SSO 버튼만 표시
```

#### __manifest__.py

```python
{
    'name': 'PolyON OIDC',
    'version': '19.0.1.0.0',
    'category': 'Authentication',
    'summary': 'Keycloak SSO 인증 + 사용자 관리 잠금',
    'depends': ['auth_oauth', 'web'],
    'author': 'Triangle.s',
    'license': 'LGPL-3',
    'data': [
        'views/hide_menus.xml',
        'views/login_template.xml',
    ],
    'post_init_hook': 'post_init_hook',
    'auto_install': False,
}
```

#### 핵심 구현 사항

**1. post_init_hook — OAuth Provider 자동 등록**

```python
# __init__.py
import os
def post_init_hook(env):
    """addon 설치 시 Keycloak OAuth Provider 자동 등록."""
    Provider = env['auth.oauth.provider'].sudo()
    
    # 이미 등록되어 있으면 업데이트
    existing = Provider.search([('client_id', '=', os.environ.get('OIDC_CLIENT_ID', 'odoo'))], limit=1)
    
    vals = {
        'name': 'PolyON SSO',
        'flow': 'id_token_code',
        'client_id': os.environ.get('OIDC_CLIENT_ID', 'odoo'),
        'auth_endpoint': os.environ.get('OIDC_AUTH_ENDPOINT', ''),
        'validation_endpoint': os.environ.get('OIDC_TOKEN_ENDPOINT', ''),
        'scope': 'openid profile email',
        'enabled': True,
        'body': 'PolyON SSO 로그인',
        'css_class': 'fa fa-fw fa-sign-in',
    }
    
    if existing:
        existing.write(vals)
    else:
        Provider.create(vals)
```

**2. res.users 잠금**

- `create()` — `context.get('polyon_sync')` 없으면 UserError
- `write()` — `name, login, email, password, active, oauth_uid` 변경 시 polyon_sync 없으면 UserError
- `unlink()` — polyon_sync 없으면 UserError

**3. _check_credentials override**

⚠️ **Odoo 19 주의:** `_check_credentials`의 시그니처가 변경됨

```python
# Odoo 19에서의 시그니처:
def _check_credentials(self, credential, env):
    # credential = {'type': 'password', 'password': '...'} (dict)
    # Odoo 18 이하: def _check_credentials(self, password, env)
```

`credential['type'] == 'password'`이면 AccessDenied 발생시켜 비밀번호 로그인 차단.
`auth_oauth` 모듈은 자체 인증 경로를 사용하므로 `_check_credentials`를 호출하지 않음 → SSO는 정상 동작.

**4. 로그인 페이지 커스텀**

QWeb 템플릿 상속으로:
- 비밀번호 입력 필드 숨김
- 로그인 버튼 숨김
- "PolyON SSO 로그인" 버튼만 표시

**5. 메뉴 숨김**

XML로 `active=False` 처리:
- `base.menu_users` (Settings → Users & Companies)
- 기타 사용자 관련 메뉴

---

### Addon 2: `polyon_s3`

**역할:** ir.attachment 파일 저장을 로컬 filestore에서 S3(RustFS)로 리다이렉트

#### 파일 구조

```
polyon_s3/
├── __init__.py
├── __manifest__.py
└── models/
    ├── __init__.py
    └── ir_attachment.py       # _file_write, _file_read, _file_delete override
```

#### 핵심 구현 사항

- `_file_write()` → S3 `PutObject`
- `_file_read()` → S3 `GetObject` (fallback: 로컬 filestore)
- `_file_delete()` → S3 `DeleteObject`
- S3 credentials: 환경변수 `AWS_HOST`, `AWS_BUCKET_NAME`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- `post_init_hook`에서 `ir.config_parameter`에 S3 설정 저장

**의존성:** `boto3` — Dockerfile에 `pip install boto3` 추가 필요

---

### Addon 3: `polyon_iframe`

**역할:** PP Console/Portal에서 iframe으로 Odoo를 표시할 수 있도록 보안 헤더 조정

#### 파일 구조

```
polyon_iframe/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── ir_http.py             # 응답 헤더 조정
└── controllers/
    ├── __init__.py
    └── main.py                # /polyon/ready 엔드포인트
```

#### 핵심 구현 사항

**1. ir.http 상속 — 응답 헤더 조정**

```python
class Http(models.AbstractModel):
    _inherit = 'ir.http'

    @classmethod
    def _post_dispatch(cls, response):
        response = super()._post_dispatch(response)
        if hasattr(response, 'headers'):
            base_domain = os.environ.get('POLYON_DOMAIN', 'cmars.com')
            response.headers['Content-Security-Policy'] = (
                f"frame-ancestors 'self' https://console.{base_domain} "
                f"https://portal.{base_domain}"
            )
            response.headers.pop('X-Frame-Options', None)
        return response
```

⚠️ **Odoo 19 주의:** `_post_dispatch`가 있는지 확인. 없으면 `_pre_dispatch` 또는 middleware 방식 사용.

**2. /polyon/ready — iframe 로드 확인**

```python
@http.route('/polyon/ready', type='json', auth='none', cors='*')
def ready(self):
    return {'status': 'ok', 'module': 'odoo'}
```

**3. SameSite Cookie 패치**

iframe에서 Odoo 세션 쿠키가 전달되려면 `SameSite=None; Secure` 필요.
Odoo의 session cookie 설정을 override하거나, Werkzeug response hook에서 변경.

---

## ⚠️ Odoo 19 호환 주의사항 (필수 숙지)

이전 작업에서 발견된 문제들. **반드시 주의:**

| # | 문제 | 상세 |
|---|------|------|
| 1 | **`groups_id` → `group_ids`** | Odoo 19에서 res.users의 그룹 필드명 변경됨 |
| 2 | **`_check_credentials` 시그니처 변경** | Odoo 19: `credential` dict (`{'type': 'password', ...}`), Odoo 18 이하: `password` string |
| 3 | **`session.authenticate()` 변경** | Odoo 19: `credential` dict 필요, `type` key 필수 |
| 4 | **`from odoo.http import Root` 불가** | Odoo 19에서 `Root` 클래스 제거/변경. monkeypatch 시 다른 방식 사용 |
| 5 | **`sudo(False)` 문제** | `auth="none"` 라우트에서 ORM의 `sudo(False)` 호출 시 `env.user`가 빈 recordset → ValueError |
| 6 | **`ir_attachment._check_contents`** | `self.env['ir.ui.view'].sudo(False).has_access('write')` 내부에서 env.user 누락 문제 |
| 7 | **post_init_hook 선언** | `__manifest__.py`에 선언하면 `__init__.py`에 반드시 해당 함수 존재해야 함. 없으면 AttributeError |
| 8 | **broken module state** | addon 로딩 실패 시 DB에 `to install` 잔류 → 서버 재시작해도 계속 실패. `UPDATE ir_module_module SET state='uninstalled'` 필요 |

---

## 테스트 방법

### 로컬 빌드 & 배포

```bash
# 1. addon 파일을 Odoo 이미지의 /opt/odoo/addons-custom/에 마운트 또는 COPY
# 2. Odoo 이미지 빌드
docker build -t jupitertriangles/polyon-odoo:v0.3.0 .

# 3. K8s 배포
docker push jupitertriangles/polyon-odoo:v0.3.0
kubectl set image deploy/polyon-odoo odoo=jupitertriangles/polyon-odoo:v0.3.0 -n polyon

# 4. addon 설치
kubectl exec -it deploy/polyon-odoo -n polyon -- \
  odoo -d polyon_odoo -i polyon_oidc,polyon_s3,polyon_iframe --stop-after-init

# 5. 검증
# - https://odoo.cmars.com/web/login → SSO 버튼만 보이는지
# - Settings → Users 메뉴 없는지
# - 파일 업로드 → RustFS S3에 저장되는지
```

### 검증 체크리스트

- [ ] `polyon_oidc` 설치 성공 (broken module state 없음)
- [ ] /web/login 페이지에 "PolyON SSO 로그인" 버튼만 표시
- [ ] SSO 로그인 → Odoo 세션 생성 → ERP 화면 진입
- [ ] Settings → Users 메뉴 없음
- [ ] Odoo 내에서 직접 사용자 생성 시도 → UserError
- [ ] `polyon_s3` 설치 성공
- [ ] 파일 업로드 → S3 버킷에 저장 확인
- [ ] `polyon_iframe` 설치 성공
- [ ] Console iframe에서 Odoo 로드 성공 (X-Frame-Options 없음)
- [ ] /polyon/ready → `{"status": "ok"}`

---

## 산출물

```
addons-custom/
├── polyon_oidc/
│   ├── __init__.py
│   ├── __manifest__.py
│   ├── models/res_users.py
│   ├── models/res_users_auth.py
│   ├── controllers/oidc.py (필요시)
│   ├── data/oauth_provider.xml (또는 post_init_hook)
│   ├── views/hide_menus.xml
│   └── views/login_template.xml
├── polyon_s3/
│   ├── __init__.py
│   ├── __manifest__.py
│   └── models/ir_attachment.py
└── polyon_iframe/
    ├── __init__.py
    ├── __manifest__.py
    ├── models/ir_http.py
    └── controllers/main.py
```
