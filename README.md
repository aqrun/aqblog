# Aqblog Rest Server

> 原仓库：ddimaria/rust-actix-example

Rust 语言实现的 Actix 2.0 REST 服务.

## 动机

Actix Web 是一个 Rust 语言实现的用于创建网站应用的快速、强大的框架。
相较于其它语言框架，本项目更符合人体工程抽象学并且最大化利用了 Actix 的性能。

## 功能

- Actix 2.x HTTP Server
- 多数据库支持 (CockroachDB, Postgres, MySQL, Sqlite)
- JWT 支持
- 异步缓存API
- 公开安全的静态文件服务
- 非阻塞 Diesel 操作
- 组织良好的可扩展的文件系统
- .env 本地开发配置文件
- 集成式应用状态 （Application State）
- 懒加载静态配置结构体
- 内置健康检查(包括 cargo 版本信息)
- 配置好的 TDD 监听器
- 自定义错误和 HTTP （json）数据验证
- 安全的 Argon2i 密码加密
- CORS 支持
- 单元测试和集成测试
- Test 覆盖报告
- 容器化运行的 Dockerfile 文件
- TravisCI 集成

## 主要依赖包

- `Argon2i`: 密码加密
- `actix-cors`: CORS 支持
- `actix-identity`: 用户登录认证
- `actix-redis` and `redis-async`: 异步缓存层
- `actix-web`: Actix Web Server
- `derive_more`: Error 错误格式化
- `diesel`: ORM 数据库操作
- `dotenv`: 配置参数加载器 (.env)
- `envy`: 将环境变量解构为配置参数结构体
- `jsonwebtoken`: JWT 编码/解码
- `kcov`: 测试覆盖分析
- `listenfd`: 监听文件系统修改
- `rayon`: 并行执行
- `r2d2`: 数据库连接池
- `validator`: JSON数据验证

## 安装

复制并进入项目文件夹：

```shell
git clone https://github.com/aqrun/aqblog.git
cd aqblog
```

复制示例 .env 文件

```shell
cp .env.example .env
```

**注意:** 需要根据你自己的环境修改 .env 中的值, 特别注意 salt 和其它一些键.

设置完 .env 中的 `DATABASE` 后，还需要把 `Cargo.toml` 中的 `features` 区域的 `default` 值和 .env的 `DATABASE` 统一：

```toml
[features]
cockroach = []
mysql = []
postgres = []
sqlite = []
default = ["postgres"]
```

_注意:_ `default` 值只支持一种数据库。

接下来需要安装 Diesel 命令行工具：

```shell
cargo install diesel_cli
```

安装出现任何错误可以查看： http://diesel.rs/guides/getting-started/

安装后就可以通过 Diesel 命令行数据库执行迁移操作：

```shell
diesel migration run
```

## 启动服务

启动服务运行：

```shell
cargo run
```

## 自动更新加载Autoreloading

可以增加启动参数让应用监控代码改动：

```shell
systemfd --no-pid -s http::3000 -- cargo watch -x run
```

## 测试

集成测试都在 `/src/tests` 目录，里面有很多工具函数简化测试操作。
例如你想测试 `GET /api/v1/user` 路由：

```rust
  use crate::tests::helpers::tests::assert_get;

  #[test]
  async fn test_get_users() {
      assert_get("/api/v1/user").await;
  }
```

可以使用 Actix 测试服务，发送请求并断言为成功的响应：

`assert!(response.status().is_success());`

同样的，要测试一个 POST 路由如下：

```rust
use crate::handlers::user::CreateUserRequest;
use crate::tests::helpers::tests::assert_post;

#[test]
async fn test_create_user() {
    let params = CreateUserRequest {
        first_name: "Satoshi".into(),
        last_name: "Nakamoto".into(),
        email: "satoshi@nakamotoinstitute.org".into(),
    };
    assert_post("/api/v1/user", params).await;
}
```

### 运行测试用例

要执行所有的测试用例运行：

```shell
cargo test
```

### Test Covearage

I created a repo on DockerHub that I'll update with each Rust version
(starting at 1.37), whose tags will match the Rust version.

In the root of the project:

```shell
docker run -it --rm --security-opt seccomp=unconfined --volume "${PWD}":/volume --workdir /volume ddimaria/rust-kcov:1.37 --exclude-pattern=/.cargo,/usr/lib,/src/main.rs,src/server.rs
```

_note: coverage takes a long time to run (up to 30 mins)._

You can view the HTML output of the report at `target/cov/index.html`

## Docker

构建应用的 Docker 镜像：

```shell
docker build -t rust_actix_example .
```

镜像制作完成，可以在 3000 端口运行一个容器：

```shell
docker run -it --rm --env-file=.env.docker -p 3000:3000 --name rust_actix_example rust_actix_example
```

## 公开静态文件服务

静态文件是在 `/static` 目录。
目录列表功能已关闭。
支持 Index 文件（`index.html`）。


如:

```shell
curl -X GET http://127.0.0.1:3000/test.html
```

## Secure Static Files

只提供已认证用户访问的安全文件放在 `/static-secure` 目录。
这些文件引用自 `/secure` 根目录

Example:

```shell
curl -X GET http://127.0.0.1:3000/secure/test.html
```

## 应用状态（Application State）

自动给服务添加了一个共享可变的应用状态，要在一个处理中读取这些数据，只需要简单的添加 `data: AppState<_, String>` 函数签名即可。

### 帮助函数

#### get\<T\>(data: AppState\<T\>, key: &str) -> Option\<T\>

通过键名 key 获取对应应用状态的一个拷贝。

例如:

```rust
use create::state::get;

pub async fn handle(data: AppState<'_, String>) -> impl Responder {
  let key = "SOME_KEY";
  let value = get(data, key);
  assert_eq!(value, Some("123".to_string()));
}
```

#### set\<T\>(data: AppState\<T\>, key: &str, value: T) -> Option\<T\>

插入或更新应用状态

Example:

```rust
use create::state::set;

pub async fn handle(data: AppState<'_, String>) -> impl Responder {
  let key = "SOME_KEY";
  let value = set(data, key, "123".into());
  assert_eq!(value, None)); // if this is an insert
  assert_eq!(value, Some("123".to_string())); // if this is an update
}
```

#### delete\<T\>(data: AppState\<T\>, key: &str) -> Option\<T\>

删除指定应用状态

Example:

```rust
use create::state::get;

pub async fn handle(data: AppState<'_, String>) -> impl Responder {
  let key = "SOME_KEY";
  let value = delete(data, key);
  assert_eq!(value, None);
}
```

## 缓存系统

如果设置了 `REDIS_URL` 环境变量，会自动添加异步 redis 支持
处理中获取这种数据只需要添加 `cache: Cache` 函数签名。

### 帮助函数

#### get(cache: Cache, key: &str) -> Result<String, ApiError>

通过 key 获取缓存数据的拷贝：

Example:

```rust
use crate::cache::{get, Cache};

pub async fn handle(cache: Cache) -> impl Responder {
  let key = "SOME_KEY";
  let value = get(cache, key).await?;
  assert_eq!(value, "123");
}
```

#### set(cache: Cache, key: &str, value: &str) -> Result<String, ApiError>

添加或更新缓存数据。

Example:

```rust
use crate::cache::{set, Cache};

pub async fn handle(cache: Cache) -> impl Responder {
  let key = "SOME_KEY";
  set(cache, key, "123").await?;
}
```

#### delete(cache: Cache, key: &str) -> Result<String, ApiError>

删除缓存数据。

Example:

```rust
use crate::cache::{delete, Cache};

pub async fn handle(cache: Cache) -> impl Responder {
  let key = "SOME_KEY";
  delete(cache, key).await?;
}
```

## 非阻塞 Diesel 数据库操作

当通过 Diesel 访问数据库时，操作会阻塞服务主线程。
这种阻塞可以通过运行处理程序内线程池中的阻塞代码来减轻。

例如:

```rust
pub async fn get_user(
    user_id: Path<Uuid>,
    pool: Data<PoolType>,
) -> Result<Json<UserResponse>, ApiError> {
    let user = block(move || find(&pool, *user_id)).await?;
    respond_json(user)
}
```

为保证 API 的统一性，阻塞错误会被自动转化为 ApiErrors

```rust
impl From<BlockingError<ApiError>> for ApiError {
    fn from(error: BlockingError<ApiError>) -> ApiError {
        match error {
            BlockingError::Error(api_error) => api_error,
            BlockingError::Canceled => ApiError::BlockingError("Thread blocking error".into()),
        }
    }
}
```

## 已添加的接口

### Healthcheck

Determine if the system is healthy.

`GET /health`

#### Response

```json
{
  "status": "ok",
  "version": "0.1.0"
}
```

Example:

```shell
curl -X GET http://127.0.0.1:3000/health
```

### Login 登录

`POST /api/v1/auth/login`

#### Request

| Param    | Type   | Description              | Required | Validations           |
| -------- | ------ | ------------------------ | :------: | --------------------- |
| email    | String | The user's email address |   yes    | valid email address   |
| password | String | The user's password      |   yes    | at least 6 characters |

```json
{
  "email": "torvalds@transmeta.com",
  "password": "123456"
}
```

#### Response

Header

```json
HTTP/1.1 200 OK
content-length: 118
content-type: application/json
set-cookie: auth=COOKIE_VALUE_HERE; HttpOnly; Path=/; Max-Age=1200
date: Tue, 15 Oct 2019 02:04:54 GMT
```

Json Body

```json
{
  "id": "0c419802-d1ef-47d6-b8fa-c886a23d61a7",
  "first_name": "Linus",
  "last_name": "Torvalds",
  "email": "torvalds@transmeta.com"
}
```

**When sending subsequent requests, create a header variable `cookie` with the value `auth=COOKIE_VALUE_HERE`**

### Logout 登出

`GET /api/v1/auth/logout`

#### Response

`200 OK`

Example:

```shell
curl -X GET http://127.0.0.1:3000/api/v1/auth/logout
```

### Get All Users 获取所有用户

`GET /api/v1/user`

#### Response

```json
[
  {
    "id": "a421a56e-8652-4da6-90ee-59dfebb9d1b4",
    "first_name": "Satoshi",
    "last_name": "Nakamoto",
    "email": "satoshi@nakamotoinstitute.org"
  },
  {
    "id": "c63d285b-7794-4419-bfb7-86d7bb3ff17d",
    "first_name": "Barbara",
    "last_name": "Liskov",
    "email": "bliskov@substitution.org"
  }
]
```

Example:

```shell
curl -X GET http://127.0.0.1:3000/api/v1/user
```

### Get a User 获取一个用户

`GET /api/v1/user/{id}`

#### Request

| 字段 | 类型 | 描述   |
| ----- | ---- | ------------- |
| id    | Uuid | The user's id |

#### Response

```json
{
  "id": "a421a56e-8652-4da6-90ee-59dfebb9d1b4",
  "first_name": "Satoshi",
  "last_name": "Nakamoto",
  "email": "satoshi@nakamotoinstitute.org"
}
```

Example:

```shell
curl -X GET http://127.0.0.1:3000/api/v1/user/a421a56e-8652-4da6-90ee-59dfebb9d1b4
```

#### Response - Not Found

`404 Not Found`

```json
{
  "errors": ["User c63d285b-7794-4419-bfb7-86d7bb3ff17a not found"]
}
```

### Create a User 创建一个用户

`POST /api/v1/user`

#### Request

| 字段      | 类型   | 描述              | 必须 | 验证           |
| ---------- | ------ | ------------------------ | :------: | --------------------- |
| first_name | String | The user's first name    |   yes    | at least 3 characters |
| last_name  | String | The user's last name     |   yes    | at least 3 characters |
| email      | String | The user's email address |   yes    | valid email address   |

```json
{
  "first_name": "Linus",
  "last_name": "Torvalds",
  "email": "torvalds@transmeta.com"
}
```

#### Response

```json
{
  "id": "0c419802-d1ef-47d6-b8fa-c886a23d61a7",
  "first_name": "Linus",
  "last_name": "Torvalds",
  "email": "torvalds@transmeta.com"
}
```

Example:

```shell
curl -X POST \
  http://127.0.0.1:3000/api/v1/user \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Linus",
    "last_name": "Torvalds",
    "email": "torvalds@transmeta.com"
}'
```

#### Response - 数据验证错误 Validation Errors

`422 Unprocessable Entity`

```json
{
  "errors": [
    "first_name is required and must be at least 3 characters",
    "last_name is required and must be at least 3 characters",
    "email must be a valid email"
  ]
}
```

### Update a User 更新一个用户

`PUT /api/v1/{id}`

#### Request

Path

| Param | Type | Description   |
| ----- | ---- | ------------- |
| id    | Uuid | The user's id |

Body

| Param      | Type   | Description              | Required | Validations           |
| ---------- | ------ | ------------------------ | :------: | --------------------- |
| first_name | String | The user's first name    |   yes    | at least 3 characters |
| last_name  | String | The user's last name     |   yes    | at least 3 characters |
| email      | String | The user's email address |   yes    | valid email address   |

```json
{
  "first_name": "Linus",
  "last_name": "Torvalds",
  "email": "torvalds@transmeta.com"
}
```

#### Response

```json
{
  "id": "0c419802-d1ef-47d6-b8fa-c886a23d61a7",
  "first_name": "Linus",
  "last_name": "Torvalds",
  "email": "torvalds@transmeta.com"
}
```

Example:

```shell
curl -X PUT \
  http://127.0.0.1:3000/api/v1/user/0c419802-d1ef-47d6-b8fa-c886a23d61a7 \
  -H 'Content-Type: application/json' \
  -d '{
    "first_name": "Linus",
    "last_name": "Torvalds",
    "email": "torvalds@transmeta.com"
}'
```

#### Response - Validation Errors

`422 Unprocessable Entity`

```json
{
  "errors": [
    "first_name is required and must be at least 3 characters",
    "last_name is required and must be at least 3 characters",
    "email must be a valid email"
  ]
}
```

#### Response - Not Found

`404 Not Found`

```json
{
  "errors": ["User 0c419802-d1ef-47d6-b8fa-c886a23d61a7 not found"]
}
```

### Delete a User 删除一个用户

`DELETE /api/v1/user/{id}`

#### Request

| Param | Type | Description   |
| ----- | ---- | ------------- |
| id    | Uuid | The user's id |

#### Response

```json
{
  "id": "a421a56e-8652-4da6-90ee-59dfebb9d1b4",
  "first_name": "Satoshi",
  "last_name": "Nakamoto",
  "email": "satoshi@nakamotoinstitute.org"
}
```

#### Response

`200 OK`

Example:

```shell
curl -X DELETE http://127.0.0.1:3000/api/v1/user/a421a56e-8652-4da6-90ee-59dfebb9d1b4
```

#### Response - Not Found

`404 Not Found`

```json
{
  "errors": ["User c63d285b-7794-4419-bfb7-86d7bb3ff17a not found"]
}
```

## 许可证

本项目许可:

- MIT license (LICENSE-MIT or http://opensource.org/licenses/MIT)
