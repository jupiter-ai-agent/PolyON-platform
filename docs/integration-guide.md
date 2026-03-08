# PolyON Integration Guide

PP 인프라 서비스에 실제로 연동하는 방법입니다.

## 1. AD DC LDAP 인증 (Pattern A — 권장)

### 개요

모듈이 Samba AD DC에 직접 LDAP bind하여 사용자를 인증합니다.
Keycloak 경유 없이 가장 단순하고 안정적인 방식입니다.

### 연결 정보

```
Host:     ${POLYON_DC_HOST}          (기본: polyon-dc)
Port:     ${POLYON_DC_PORT}          (기본: 389)
Base DN:  ${POLYON_DC_BASE_DN}       (예: DC=company,DC=com)
Bind DN:  ${POLYON_DC_ADMIN_DN}      (예: CN=Administrator,CN=Users,DC=company,DC=com)
Password: ${POLYON_DC_ADMIN_PASSWORD}
TLS:      No (K8s 내부 네트워크)
```

### Rust 예제 (ldap3 crate)

```rust
use ldap3::{LdapConnAsync, Scope, SearchEntry};

async fn authenticate(username: &str, password: &str) -> Result<bool, Box<dyn std::error::Error>> {
    let dc_host = std::env::var("POLYON_DC_HOST").unwrap_or("polyon-dc".into());
    let dc_port = std::env::var("POLYON_DC_PORT").unwrap_or("389".into());
    let base_dn = std::env::var("POLYON_DC_BASE_DN")?;

    let (conn, mut ldap) = LdapConnAsync::new(
        &format!("ldap://{}:{}", dc_host, dc_port)
    ).await?;
    ldap3::drive!(conn);

    // 1. 서비스 계정으로 바인드하여 사용자 DN 검색
    let admin_dn = std::env::var("POLYON_DC_ADMIN_DN")?;
    let admin_pw = std::env::var("POLYON_DC_ADMIN_PASSWORD")?;
    ldap.simple_bind(&admin_dn, &admin_pw).await?.success()?;

    // 2. sAMAccountName으로 사용자 검색
    let filter = format!(
        "(&(objectClass=user)(!(objectClass=computer))\
         (!(userAccountControl:1.2.840.113556.1.4.803:=2))\
         (sAMAccountName={}))",
        username
    );
    let (rs, _) = ldap.search(&base_dn, Scope::Subtree, &filter, vec!["dn"]).await?.success()?;

    if rs.is_empty() {
        return Ok(false); // 사용자 없음
    }

    let user_dn = SearchEntry::construct(rs.into_iter().next().unwrap()).dn;

    // 3. 사용자 DN으로 바인드 (비밀번호 검증)
    let result = ldap.simple_bind(&user_dn, password).await?;
    Ok(result.rc == 0)
}
```

### 사용자 프로필 조회

```rust
async fn get_user_profile(username: &str) -> Result<UserProfile, Error> {
    // ... bind with service account ...

    let filter = format!(
        "(&(objectClass=user)(sAMAccountName={}))", username
    );
    let attrs = vec![
        "sAMAccountName",    // 사용자 ID
        "displayName",       // 표시 이름
        "mail",              // 이메일
        "memberOf",          // 그룹 목록
        "userPrincipalName", // UPN
    ];

    let (rs, _) = ldap.search(&base_dn, Scope::Subtree, &filter, attrs)
        .await?.success()?;

    // ... parse result ...
}
```

### 시스템 계정 필터링

다음 계정은 일반 사용자 목록에서 제외합니다:

```rust
const SYSTEM_ACCOUNTS: &[&str] = &[
    "admin", "administrator", "system-bot", "system",
    "guest", "polyon", "advisor", "krbtgt",
];
```

## 2. RustFS (S3 Object Storage)

### 개요

모든 파일/오브젝트는 RustFS에 저장합니다.
RustFS는 S3 호환 API를 제공하므로 **AWS SDK**를 그대로 사용할 수 있습니다.

### 연결 정보

```
Endpoint:   http://${POLYON_RUSTFS_HOST}:9000  (기본: polyon-rustfs)
Access Key: ${POLYON_RUSTFS_ACCESS_KEY}
Secret Key: ${POLYON_RUSTFS_SECRET_KEY}
Region:     us-east-1 (기본값, 실제 리전 무관)
SSL:        No (K8s 내부)
Path Style: Yes (virtual-hosted 아님)
```

### Rust 예제 (aws-sdk-s3)

```rust
use aws_sdk_s3::{Client, Config, config::Credentials, config::Region};
use aws_sdk_s3::primitives::ByteStream;

async fn create_s3_client() -> Client {
    let endpoint = format!(
        "http://{}:9000",
        std::env::var("POLYON_RUSTFS_HOST").unwrap_or("polyon-rustfs".into())
    );

    let creds = Credentials::new(
        &std::env::var("POLYON_RUSTFS_ACCESS_KEY").unwrap(),
        &std::env::var("POLYON_RUSTFS_SECRET_KEY").unwrap(),
        None, None, "polyon",
    );

    let config = Config::builder()
        .endpoint_url(endpoint)
        .region(Region::new("us-east-1"))
        .credentials_provider(creds)
        .force_path_style(true)  // 필수!
        .build();

    Client::from_conf(config)
}

// 파일 업로드
async fn upload_file(client: &Client, bucket: &str, key: &str, data: Vec<u8>) {
    client.put_object()
        .bucket(bucket)
        .key(key)
        .body(ByteStream::from(data))
        .send()
        .await
        .expect("upload failed");
}

// 파일 다운로드
async fn download_file(client: &Client, bucket: &str, key: &str) -> Vec<u8> {
    let resp = client.get_object()
        .bucket(bucket)
        .key(key)
        .send()
        .await
        .expect("download failed");

    resp.body.collect().await.unwrap().to_vec()
}
```

### 버킷 규칙

- 모듈별 전용 버킷: `{moduleId}` (예: `drive`)
- 버킷 내 사용자별 prefix: `{username}/` (예: `drive/testuser1/documents/`)
- 공유 파일: `__shared__/` prefix
- 버킷은 모듈 설치 시 Core가 자동 생성

## 3. PostgreSQL

### 연결 정보

```
Host:     ${DB_HOST}      (기본: polyon-db)
Port:     ${DB_PORT}      (기본: 5432)
Database: ${DB_NAME}      (모듈별 자동 생성)
User:     ${DB_USER}      (모듈별 자동 생성)
Password: ${DB_PASSWORD}  (자동 생성)
SSL:      disable
```

### Rust 예제 (sqlx)

```rust
use sqlx::postgres::PgPoolOptions;

async fn create_pool() -> sqlx::PgPool {
    let url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| {
            format!("postgres://{}:{}@{}:{}/{}?sslmode=disable",
                std::env::var("DB_USER").unwrap(),
                std::env::var("DB_PASSWORD").unwrap(),
                std::env::var("DB_HOST").unwrap_or("polyon-db".into()),
                std::env::var("DB_PORT").unwrap_or("5432".into()),
                std::env::var("DB_NAME").unwrap(),
            )
        });

    PgPoolOptions::new()
        .max_connections(10)
        .connect(&url)
        .await
        .expect("Failed to connect to database")
}
```

### 마이그레이션

모듈이 자체 마이그레이션을 관리합니다.
`sqlx migrate`, `refinery`, `barrel` 등 Rust 마이그레이션 도구 사용 권장.

## 4. OpenSearch (Full-text Search)

### 연결 정보

```
URL:  http://${POLYON_SEARCH_HOST}:9200  (기본: polyon-search)
Auth: None (K8s 내부)
API:  Elasticsearch 7.x compatible
```

### Rust 예제 (opensearch-rs 또는 elasticsearch-rs)

```rust
use opensearch::OpenSearch;
use opensearch::http::transport::Transport;

async fn create_search_client() -> OpenSearch {
    let url = format!(
        "http://{}:9200",
        std::env::var("POLYON_SEARCH_HOST").unwrap_or("polyon-search".into())
    );
    let transport = Transport::single_node(&url).unwrap();
    OpenSearch::new(transport)
}
```

### 인덱스 네이밍

```
{moduleId}_{resource}

예:
  drive_files       ← 파일 메타데이터
  drive_contents    ← 파일 내용 (전문검색)
```

## 5. Health Check

모듈은 HTTP health endpoint를 제공해야 합니다:

```
GET /health

Response 200:
{
  "status": "ok",
  "version": "1.0.0",
  "uptime": 3600
}

Response 503:
{
  "status": "unhealthy",
  "error": "database connection failed"
}
```

K8s Probe 설정:
- **Readiness:** 서비스가 트래픽을 받을 준비가 되었는지
- **Liveness:** 서비스가 살아있는지 (실패 시 재시작)

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```
