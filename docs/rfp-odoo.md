# PP Odoo 커스텀 설계서

> Odoo 19를 PP의 ERP/HR/회계 모듈로 통합하기 위한 커스텀 addon 설계
> 원칙: **Odoo core 수정 금지, addon 레벨 커스텀만, res.users는 AD의 읽기 전용 미러**

---

## 1. 설계 원칙

| 원칙 | 설명 |
|------|------|
| **Odoo는 모른다** | Odoo는 PP 위에서 돌아간다는 걸 모름. 평소처럼 동작 |
| **사용자는 모른다** | 사용자는 ERP가 Odoo라는 걸 모름. PP의 기능일 뿐 |
| **Core가 중재한다** | AD ↔ Odoo 동기화, OAuth Provider 등록, 권한 매핑 — 전부 Core |
| **Odoo core 수정 금지** | 모든 커스텀은 addon으로. 업그레이드 호환성 유지 |
| **res.users 유지** | 삭제/교체 아님. AD 데이터의 읽기 전용 미러로 활용 |

---

## 2. 커스텀 addon 구성

```
addons-custom/
├── polyon_oidc/          # 1. OIDC 인증 + 사용자 잠금
├── polyon_s3/            # 2. 파일 스토리지 → RustFS(S3)
└── polyon_iframe/        # 3. iframe 통합 (X-Frame-Options, SameSite)
```

### 2.1 polyon_oidc — OIDC 인증 + 사용자 잠금

**역할:** Keycloak SSO를 유일한 로그인 경로로 만들고, Odoo 내 사용자 직접 관리를 차단

#### 의존성

```python
# __manifest__.py
{
    'name': 'PolyON OIDC',
    'version': '19.0.1.0.0',
    'depends': ['auth_oauth', 'web'],
    'author': 'Triangle.s',
    'auto_install': False,
    'data': [
        'data/oauth_provider.xml',
        'views/hide_menus.xml',
        'views/login_template.xml',
    ],
}
```

#### 2.1.1 OAuth Provider 자동 등록

```xml
<!-- data/oauth_provider.xml -->
<odoo noupdate="1">
  <record id="polyon_sso_provider" model="auth.oauth.provider">
    <field name="name">PolyON SSO</field>
    <field name="flow">id_token_code</field>
    <field name="client_id" eval="os.environ.get('OIDC_CLIENT_ID', 'odoo')" />
    <field name="auth_endpoint" eval="os.environ.get('OIDC_AUTH_ENDPOINT', '')" />
    <field name="validation_endpoint" eval="os.environ.get('OIDC_TOKEN_ENDPOINT', '')" />
    <field name="data_endpoint" eval="os.environ.get('OIDC_USERINFO_ENDPOINT', '')" />
    <field name="scope">openid profile email</field>
    <field name="enabled" eval="True" />
    <field name="body">PolyON SSO 로그인</field>
    <field name="css_class">fa fa-fw fa-sign-in</field>
  </record>
</odoo>
```

> ⚠️ **환경변수 방식이 Odoo XML에서 안 되면** → Core가 JSON-RPC로 설치 후 자동 등록

#### 2.1.2 res.users 잠금 (AD 미러 보호)

```python
# models/res_users.py
from odoo import models, api
from odoo.exceptions import UserError

class Users(models.Model):
    _inherit = 'res.users'

    @api.model_create_multi
    def create(self, vals_list):
        """PP Core(polyon_sync context)만 사용자 생성 허용."""
        if not self.env.context.get('polyon_sync'):
            raise UserError(
                '사용자 생성은 PolyON Console에서만 가능합니다.\n'
                'Settings → Users에서 사용자를 추가할 수 없습니다.'
            )
        return super().create(vals_list)

    def write(self, vals):
        """보호 필드 변경은 PP Core만 허용."""
        protected_fields = {'name', 'login', 'email', 'password', 'active', 'oauth_uid'}
        if protected_fields & set(vals.keys()):
            if not self.env.context.get('polyon_sync'):
                raise UserError(
                    '사용자 정보는 PolyON Console에서 변경하세요.\n'
                    'AD 계정 정보가 자동 동기화됩니다.'
                )
        return super().write(vals)

    def unlink(self):
        """사용자 삭제 차단 — 비활성화만 허용."""
        if not self.env.context.get('polyon_sync'):
            raise UserError('사용자 삭제는 PolyON Console에서만 가능합니다.')
        return super().unlink()
```

#### 2.1.3 메뉴 숨김

```xml
<!-- views/hide_menus.xml -->
<odoo>
  <!-- Settings → Users & Companies 숨김 -->
  <record id="base.menu_users" model="ir.ui.menu">
    <field name="active" eval="False" />
  </record>

  <!-- Settings → Technical → Database Structure 숨김 -->
  <record id="base.menu_ir_property" model="ir.ui.menu">
    <field name="active" eval="False" />
  </record>

  <!-- /web/database/manager 접근 차단은 config로 처리 -->
  <!-- list_db = false (이미 적용됨) -->
</odoo>
```

#### 2.1.4 로그인 페이지 커스텀

```xml
<!-- views/login_template.xml -->
<odoo>
  <template id="polyon_login" inherit_id="web.login" priority="99">
    <!-- 비밀번호 입력 필드 숨김 — SSO 버튼만 표시 -->
    <xpath expr="//div[hasclass('field-login')]" position="replace">
      <div class="text-center mb-3">
        <h4>PolyON 계정으로 로그인</h4>
      </div>
    </xpath>
    <xpath expr="//div[hasclass('field-password')]" position="replace" />
    <xpath expr="//button[@type='submit']" position="replace" />
    <xpath expr="//div[hasclass('o_login_auth')]" position="attributes">
      <attribute name="class">o_login_auth text-center</attribute>
    </xpath>
  </template>
</odoo>
```

→ 결과: Odoo 로그인 페이지에 **"PolyON SSO 로그인" 버튼만** 표시. 비밀번호 입력 불가.

#### 2.1.5 비밀번호 로그인 차단

```python
# models/res_users_auth.py
from odoo import models
from odoo.exceptions import AccessDenied

class Users(models.Model):
    _inherit = 'res.users'

    def _check_credentials(self, credential, env):
        """비밀번호 로그인 차단 — OAuth만 허용."""
        cred_type = credential.get('type', 'password') if isinstance(credential, dict) else 'password'
        if cred_type == 'password':
            raise AccessDenied('비밀번호 로그인이 비활성화되었습니다. PolyON SSO를 사용하세요.')
        return super()._check_credentials(credential, env)
```

---

### 2.2 polyon_s3 — 파일 스토리지 → RustFS

**역할:** `ir.attachment`의 파일 저장을 로컬 filestore에서 S3(RustFS)로 리다이렉트

```python
# models/ir_attachment.py
import boto3
from odoo import models, api

class IrAttachment(models.Model):
    _inherit = 'ir.attachment'

    def _get_s3_client(self):
        return boto3.client(
            's3',
            endpoint_url=self.env['ir.config_parameter'].sudo().get_param('polyon.s3.endpoint'),
            aws_access_key_id=self.env['ir.config_parameter'].sudo().get_param('polyon.s3.access_key'),
            aws_secret_access_key=self.env['ir.config_parameter'].sudo().get_param('polyon.s3.secret_key'),
        )

    @api.model
    def _file_write(self, bin_data, checksum):
        """파일을 S3에 저장."""
        s3 = self._get_s3_client()
        bucket = self.env['ir.config_parameter'].sudo().get_param('polyon.s3.bucket', 'odoo')
        key = f"filestore/{checksum[:2]}/{checksum}"
        s3.put_object(Bucket=bucket, Key=key, Body=bin_data)
        return checksum

    @api.model
    def _file_read(self, fname):
        """파일을 S3에서 읽기."""
        s3 = self._get_s3_client()
        bucket = self.env['ir.config_parameter'].sudo().get_param('polyon.s3.bucket', 'odoo')
        key = f"filestore/{fname[:2]}/{fname}"
        try:
            response = s3.get_object(Bucket=bucket, Key=key)
            return response['Body'].read()
        except s3.exceptions.NoSuchKey:
            # fallback to local filestore (마이그레이션 기간)
            return super()._file_read(fname)

    @api.model
    def _file_delete(self, fname):
        """파일을 S3에서 삭제."""
        s3 = self._get_s3_client()
        bucket = self.env['ir.config_parameter'].sudo().get_param('polyon.s3.bucket', 'odoo')
        key = f"filestore/{fname[:2]}/{fname}"
        try:
            s3.delete_object(Bucket=bucket, Key=key)
        except Exception:
            pass  # 이미 없으면 무시
```

**S3 설정 자동 주입 (PRC):**

```python
# hooks.py — addon 설치 시 PRC 환경변수를 ir.config_parameter에 저장
import os
def post_init_hook(env):
    ICP = env['ir.config_parameter'].sudo()
    ICP.set_param('polyon.s3.endpoint', os.environ.get('S3_ENDPOINT', ''))
    ICP.set_param('polyon.s3.bucket', os.environ.get('S3_BUCKET', 'odoo'))
    ICP.set_param('polyon.s3.access_key', os.environ.get('S3_ACCESS_KEY', ''))
    ICP.set_param('polyon.s3.secret_key', os.environ.get('S3_SECRET_KEY', ''))
```

---

### 2.3 polyon_iframe — iframe 통합

**역할:** PP Console/Portal에서 iframe으로 Odoo를 표시할 수 있도록 보안 헤더 조정

```python
# controllers/main.py
from odoo import http

class PolyonIframe(http.Controller):

    @http.route('/polyon/ready', type='json', auth='none', cors='*')
    def ready(self):
        """PP Console이 Odoo iframe 로드 완료를 확인하는 엔드포인트."""
        return {'status': 'ok', 'module': 'odoo'}
```

```python
# models/http.py — 응답 헤더 조정
from odoo import models
from odoo.http import Response

class Http(models.AbstractModel):
    _inherit = 'ir.http'

    @classmethod
    def _post_dispatch(cls, response):
        response = super()._post_dispatch(response)
        if isinstance(response, Response):
            # iframe 허용 (PP 도메인만)
            base_domain = os.environ.get('POLYON_DOMAIN', 'cmars.com')
            response.headers['Content-Security-Policy'] = (
                f"frame-ancestors 'self' https://console.{base_domain} "
                f"https://portal.{base_domain}"
            )
            # X-Frame-Options 제거 (CSP가 대체)
            response.headers.pop('X-Frame-Options', None)
            # SameSite=None for cross-origin iframe
            # (Odoo 세션 쿠키가 iframe에서도 전달되려면 필요)
        return response
```

---

## 3. Core 측 자동화

### 3.1 PRC auth Provider 확장

모듈 설치 시 PRC가 자동으로:

```
1. Keycloak client 생성 (clientId: odoo, confidential)
2. client_secret → K8s Secret 주입
3. Odoo JSON-RPC → auth.oauth.provider 레코드 생성
4. Odoo JSON-RPC → ir.config_parameter에 S3 설정 주입
```

### 3.2 사용자 동기화 (OdooSync)

Core에 `OdooSync` 컴포넌트 추가:

| 이벤트 | Core 동작 |
|--------|----------|
| AD 사용자 생성 | → Odoo `res.users` 생성 (oauth 전용, 비밀번호 없음) |
| AD 사용자 정보 변경 | → Odoo `res.users` name/email 업데이트 |
| AD 사용자 비활성화 | → Odoo `res.users` active=False |
| AD 그룹 변경 | → Odoo `res.groups` 매핑 (Phase 2) |

**동기화 방식:** Odoo JSON-RPC API (`/jsonrpc`)

```
POST /jsonrpc
{
  "jsonrpc": "2.0",
  "method": "call",
  "params": {
    "service": "object",
    "method": "execute_kw",
    "args": ["polyon_odoo", 2, "admin_api_key",
             "res.users", "create",
             [{"name": "홍길동", "login": "gildong", "email": "gildong@cmars.com",
               "oauth_provider_id": 1, "oauth_uid": "keycloak-sub-uuid"}],
             {"context": {"polyon_sync": true}}]
  }
}
```

`polyon_sync: true` context가 있어야 `polyon_oidc` addon의 잠금을 통과합니다.

### 3.3 인증 방식

Core → Odoo API 호출 시 인증:

| 방식 | 설명 |
|------|------|
| **API Key** (추천) | Odoo 19 내장 `res.users.apikeys` — 설치 시 자동 생성 |
| JSON-RPC + session | 기존 방식, 단 세션 관리 필요 |

PRC가 Odoo 설치 시 `__system__` (uid=1) 또는 `admin` (uid=2)의 API Key를 자동 생성하여 K8s Secret에 저장.

---

## 4. 사용자 경험 (최종 결과)

### 관리자 (Console)

```
Console → 사용자 추가 → AD 계정 생성
                      → 자동: Keycloak 동기화
                      → 자동: Odoo res.users 생성
                      → 자동: Mattermost 유저 생성
                      → 자동: 메일 계정 생성
```

### 사원 (Portal)

```
Portal → ERP 메뉴 클릭 → Odoo iframe 로드
                        → SSO 자동 로그인 (KC 세션)
                        → Odoo ERP 기능 사용
                        → PP Portal로 돌아가기
```

- 별도 로그인 없음
- Odoo UI가 보이지만 "Odoo"라는 브랜딩 없음
- 사용자 메뉴/설정 메뉴 없음 — PP Console에서 관리

### Odoo 내부

```
Odoo → res.users 정상 → ORM 정상
     → auth_oauth 정상 → 세션 정상
     → ir.attachment → S3 → 파일 정상
     → 모든 ERP 모듈 → 정상 동작
     → 커뮤니티 모듈 설치 → 가능 (addon 충돌 없음)
```

---

## 5. 구현 우선순위

| 순서 | 작업 | addon | 난이도 |
|------|------|-------|--------|
| 1 | `polyon_oidc` — auth_oauth 연동 + 로그인 페이지 | 신규 | 🟡 |
| 2 | `polyon_oidc` — res.users 잠금 + 메뉴 숨김 | 신규 | 🟢 |
| 3 | `polyon_iframe` — CSP/X-Frame-Options | 수정 | 🟢 |
| 4 | Core `OdooSync` — AD → res.users 동기화 | Core | 🟡 |
| 5 | `polyon_s3` — ir.attachment S3 리다이렉트 | 신규 | 🟡 |
| 6 | Core PRC — OAuth Provider 자동 등록 | Core | 🟡 |
| 7 | AD 그룹 → Odoo 그룹 매핑 | Core | 🔴 Phase 2 |

---

## 6. 검증 시나리오

| # | 시나리오 | 기대 결과 |
|---|---------|----------|
| 1 | Console에서 사용자 추가 | AD + Odoo에 동시 생성 |
| 2 | Portal → ERP 클릭 | SSO 자동 로그인, Odoo UI 표시 |
| 3 | Odoo Settings 접근 | Users 메뉴 없음 |
| 4 | Odoo /web/login 직접 접근 | SSO 버튼만 표시, 비밀번호 입력 불가 |
| 5 | Odoo에서 파일 업로드 | RustFS S3에 저장됨 |
| 6 | AD에서 사용자 비활성화 | Odoo res.users active=False |
| 7 | Odoo 커뮤니티 모듈 설치 | 정상 동작 (addon 충돌 없음) |
| 8 | Odoo 버전 업그레이드 (19→20) | addon만 수정, core 변경 없음 |
