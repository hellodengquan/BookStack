# BookStack API Token 校验、请求限流与权限判断协作流程

## 一、整体架构概览

API 请求进入系统后，依次经过以下三层校验：

```
请求 → [1. 速率限制] → [2. Token 认证] → [3. 权限判断] → 控制器处理
```

中间件注册顺序定义在 `app/Http/Kernel.php:40-46`：

```php
'api' => [
    \BookStack\Http\Middleware\ThrottleApiRequests::class,  // 1. 速率限制
    \BookStack\Http\Middleware\EncryptCookies::class,
    \BookStack\Http\Middleware\StartSessionIfCookieExists::class,
    \BookStack\Http\Middleware\ApiAuthenticate::class,       // 2. Token 认证
    \BookStack\Http\Middleware\CheckEmailConfirmed::class,   // 邮箱确认检查
],
```

---

## 二、速率限制层（ThrottleApiRequests）

### 2.1 核心实现

**文件**：`app/Http/Middleware/ThrottleApiRequests.php`

```php
class ThrottleApiRequests extends Middleware
{
    protected function resolveMaxAttempts($request, $maxAttempts): int
    {
        return (int) config('api.requests_per_minute');
    }
}
```

继承自 Laravel 的 `Illuminate\Routing\Middleware\ThrottleRequests`，重写了 `resolveMaxAttempts` 方法，从配置中读取每分钟请求数限制。

### 2.2 配置来源

**文件**：`app/Config/api.php:21`

```php
'requests_per_minute' => env('API_REQUESTS_PER_MIN', 180),
```

默认限制 180 次/分钟，可通过环境变量 `API_REQUESTS_PER_MIN` 覆盖。

### 2.3 限流粒度

在 `app/App/Providers/RouteServiceProvider.php:79-95` 中还定义了更细粒度的限流器：

```php
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

限流标识逻辑：**已认证用户按用户 ID，未认证用户按 IP 地址**。

> **注意**：虽然 `RouteServiceProvider` 定义了名为 `api` 的限流器（60次/分钟），但实际 API 路由组使用的是 `ThrottleApiRequests` 中间件，直接读取 `api.requests_per_minute` 配置（默认180次/分钟），优先级更高。

### 2.4 ThrottleApiRequests 实例化与限流 Key 解析机制

`ThrottleApiRequests` 作为 `api` 中间件组的成员被 Laravel 容器自动实例化，**无需构造参数**。它在 `Kernel.php` 的 `$middlewareGroups['api']` 数组中注册，Laravel 在匹配到 `api` 路由组时自动从容器解析并执行。

BookStack 只覆写了 `resolveMaxAttempts()` 一个方法，用于将限流窗口大小从"路由参数传入"改为"从配置文件动态读取"。父类 `ThrottleRequests` 的其余行为完整保留，**包括限流 Key 的计算逻辑**。

父类 `ThrottleRequests::handle()` 的核心调用链：

```
handle($request, $next, $maxAttempts=60, $decayMinutes=1, $prefix='')
  → resolveMaxAttempts($request, $maxAttempts)         ← BookStack 覆写此方法
  → resolveRequestSignature($request)                    ← BookStack 未覆写，沿用父类
  → hit/throttle 检查
```

**限流 Key 的解析**（`resolveRequestSignature`）沿用了 Laravel 父类的默认行为：

```php
// Laravel ThrottleRequests 父类逻辑
protected function resolveRequestSignature($request)
{
    if ($user = $request->user()) {
        return sha1($user->getAuthIdentifier());    // 已认证：sha1(user_id)
    }
    return sha1($request->ip());                     // 未认证：sha1(ip)
}
```

### 2.5 限流 Key 的 IP / 用户隔离粒度与中间件执行顺序的交互

限流中间件位于 `api` 组的第 1 位，而 `ApiAuthenticate` 位于第 4 位。这意味着**限流 Key 计算时，认证尚未执行**，`$request->user()` 取决于更前面的 `StartSessionIfCookieExists` 中间件是否已解析出用户。

`StartSessionIfCookieExists`（`app/Http/Middleware/StartSessionIfCookieExists.php`）是 BookStack 对 Laravel `StartSession` 的定制版：

```php
class StartSessionIfCookieExists extends Middleware
{
    public function handle($request, Closure $next)
    {
        $sessionCookieName = config('session.cookie');
        if ($request->cookies->has($sessionCookieName)) {
            return parent::handle($request, $next);  // Cookie 存在 → 启动 Session
        }
        return $next($request);                       // 无 Cookie → 跳过 Session
    }
}
```

因此限流 Key 的实际计算存在三种场景：

| 场景 | 请求特征 | 限流 Key | 说明 |
|------|---------|---------|------|
| Token 认证 | 无 Session Cookie，`Authorization: Token ...` | `sha1(ip)` | Session 未启动，`$request->user()` 为 null，按 IP 限流 |
| Session 认证 | 有 Session Cookie | `sha1(user_id)` | Session 已启动，`$request->user()` 可用，按用户 ID 限流 |
| 无认证 | 无 Cookie、无 Token | `sha1(ip)` | 按 IP 限流 |

**关键发现**：Token 认证的 API 请求（最常见场景），由于 Session 未启动，**限流 Key 始终基于 IP 地址**而非用户 ID。同一 IP 下多个用户的 Token 请求共享限流配额；反之，同一用户从不同 IP 发起请求则各自独立计数。

此外，`RouteServiceProvider` 中定义的 `RateLimiter::for('api')` 限流器（60次/分钟）**不会被使用**——因为 `ThrottleApiRequests` 继承自 `ThrottleRequests`，走的是 `handle()` → `resolveMaxAttempts()` 路径，而非 `RateLimiter` 命名限流器路径。这个命名限流器目前只存在于定义层面，未被任何路由或中间件引用。

### 2.6 超长查询参数与重复 retry 场景下的计数行为

**核心结论**：限流计数器只以 `sha1(user_id)` 或 `sha1(ip)` 为 bucket key，**与 URL 路径、查询参数、请求方法均无关**。这带来了几个重要行为：

#### 2.6.1 超长查询参数不影响计数

```
/api/pages?search=abc             → 同一个 bucket (sha1(ip))
/api/pages?search=xyz             → 同一个 bucket
/api/pages?filter[id][]=1&...     → 同一个 bucket (参数长度无影响)
/api/books/1                      → 同一个 bucket
/api/books/2                      → 同一个 bucket
```

无论查询参数多长、多复杂，只要来源 IP 相同（或用户相同），所有 API 请求都计入同一个计数器。这意味着：
- 攻击者无法通过"变换参数"绕过限流
- 但也无法对不同 API 路径做差异化限流
- 整个 API 系统共享同一个全局限流配额

#### 2.6.2 重复 retry 会持续累加计数

每次 HTTP 请求进入 `ThrottleRequests::handle()` 都会调用 `$limiter->hit($key)` 使计数 +1，无论请求成功或失败：

```
第 1 次请求 → hit → count=1   → 200 OK
第 2 次请求 → hit → count=2   → 401 Unauthorized (Token 错误)
第 3 次请求 → hit → count=3   → 403 Forbidden (权限不足)
第 4 次请求 → hit → count=4   → 500 Server Error
...
第 180 次 → hit → count=180   → 429 Too Many Requests
```

**重试惩罚效应**：客户端遇到 4xx/5xx 错误后重试，每次重试都会继续消耗限流配额。在高频失败重试场景下，可能快速耗尽配额导致正常请求也被拒绝。

#### 2.6.3 计数底层实现

限流计数通过 Laravel 的 `Illuminate\Cache\RateLimiter` 实现，底层依赖应用缓存驱动（`app/Config/cache.php` 配置，默认 `file` 驱动）。

计数键的实际结构（Laravel 内部）：
```
// 键名
throttle:xxxxxxxxxxxxx   ← sha1(key) 哈希后的 bucket 标识

// 值结构（缓存中存储）
[
    'key' => 'throttle:xxx',
    'time' => 1718888888,  // 窗口起始时间
    'decay' => 60,          // 衰减秒数
]
```

Laravel 的 RateLimiter 使用**固定时间窗口**算法，每次 hit 时检查距离 `time` 是否超过 `decay` 秒：
- 未超过 → 计数 +1
- 已超过 → 重置 time 为当前时间，计数归 1

这意味着在窗口边界附近可能出现"突刺"现象：窗口快结束时打满 180 次，紧接着下一个窗口开始又可以打 180 次，短时间内可能出现接近 2× 的请求量。

---

## 三、Token 认证层（ApiAuthenticate + ApiTokenGuard）

### 3.1 入口中间件：ApiAuthenticate

**文件**：`app/Http/Middleware/ApiAuthenticate.php`

支持两种认证方式，优先使用 Session 认证（便于在 UI 中探索 API），否则使用 Token 认证：

```
┌─────────────────────────────────────────────────────┐
│ ensureAuthorizedBySessionOrToken()                  │
│                                                     │
│  1. Session 已启动?                                  │
│     ├─ 是 → 检查用户是否有 AccessApi 权限             │
│     │       ├─ 无权限 → 抛出 403 ApiAuthException   │
│     │       └─ 有权限 → 仅允许 GET 请求               │
│     │                ├─ 非GET → 抛出 403 异常        │
│     │                └─ GET → 通过，继续后续流程     │
│     └─ 否 → 切换到 'api' guard，执行 Token 认证       │
│              auth()->shouldUse('api')                │
│              auth()->authenticate()                  │
└─────────────────────────────────────────────────────┘
```

关键代码片段（`app/Http/Middleware/ApiAuthenticate.php:31-64`）：

```php
protected function ensureAuthorizedBySessionOrToken(Request $request): void
{
    if (session()->isStarted()) {
        if (!$this->sessionUserHasApiAccess()) {
            throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
        }
        if ($request->method() !== 'GET') {
            throw new ApiAuthException(trans('errors.api_cookie_auth_only_get'), 403);
        }
        return;
    }

    auth()->shouldUse('api');
    auth()->authenticate();
}
```

### 3.2 Token 认证守卫：ApiTokenGuard

**文件**：`app/Api/ApiTokenGuard.php`

认证配置在 `app/Config/auth.php:50-52`：

```php
'api' => [
    'driver' => 'api-token',
],
```

完整的 Token 校验流程：

```
┌──────────────────────────────────────────────────────────────────┐
│ getAuthorisedUserFromRequest()                                   │
│                                                                  │
│  1. 读取 Authorization 请求头                                      │
│     $authToken = $request->headers->get('Authorization')         │
│                                                                  │
│  2. validateTokenHeaderValue() — 格式校验                          │
│     ├─ 空值 → "No authorization found"                           │
│     ├─ 格式错误（不含":"或不以"Token "开头）                        │
│     │     → "Bad authorization format"                           │
│     └─ 通过 → 解析 token_id:secret                               │
│                                                                  │
│  3. 数据库查询 ApiToken                                           │
│     ApiToken::where('token_id', $id)->with('user')->first()      │
│                                                                  │
│  4. validateToken() — Token 有效性校验                              │
│     ├─ Token 不存在 → "Token not found"                          │
│     ├─ Secret 哈希校验失败 → "Incorrect token secret"            │
│     ├─ Token 已过期 ($token->expires_at <= now)                  │
│     │     → "Token expired" (403)                                │
│     └─ 用户无 AccessApi 权限 → "No API permission" (403)         │
│                                                                  │
│  5. 检查邮箱确认状态                                               │
│     loginService->awaitingEmailConfirmation($user)               │
│     → 未确认邮箱 → "Email confirmation awaiting"                 │
│                                                                  │
│  6. 返回认证用户 $token->user                                      │
└──────────────────────────────────────────────────────────────────┘
```

### 3.3 Token 生成与 Hash 存储

**文件**：`app/Api/UserApiTokenController.php:37-68`

Token 在 `store()` 方法中生成，流程如下：

```
┌──────────────────────────────────────────────────────────────────┐
│ Token 生成流程 (UserApiTokenController::store)                    │
│                                                                  │
│  1. 生成原始密钥                                                  │
│     $secret = Str::random(32);      ← 32字节随机字符串(明文)      │
│     $token_id = Str::random(32);    ← 32字节随机字符串(明文)      │
│                                                                  │
│  2. Hash 存储                                                     │
│     'secret' => Hash::make($secret) ← bcrypt 哈希后存入数据库     │
│     'token_id' => Str::random(32)   ← token_id 明文存储           │
│                                                                  │
│  3. 唯一性保证                                                    │
│     while (ApiToken::where('token_id', $token->token_id)->exists())│
│         $token->token_id = Str::random(32); ← 碰撞重试           │
│                                                                  │
│  4. 保存并临时展示                                                │
│     $token->save()                                                │
│     session()->flash('api-token-secret:' . $token->id, $secret)  │
│     ← 明文 secret 仅通过 session flash 传递一次，                  │
│       在 edit 页面展示后即从 session 中 pull 掉                    │
│                                                                  │
│  5. 数据库中最终存储结构                                           │
│     token_id: 明文 32 字符                                        │
│     secret:   bcrypt 哈希值 ($2y$12$...)                         │
│     expires_at: 日期 (默认100年后)                                 │
└──────────────────────────────────────────────────────────────────┘
```

Hash 使用的算法由 Laravel 的 `hashing.php` 配置决定，默认为 `bcrypt`（rounds=10）。客户端使用时需拼出 `Authorization: Token {token_id}:{secret明文}` 请求头，其中 `token_id` 明文比对、`secret` 经过 `Hash::check()` 比对：

```php
// ApiTokenGuard::validateToken() — app/Api/ApiTokenGuard.php:120-138
if (!Hash::check($secret, $token->secret)) {         // secret 明文 vs 数据库中的 bcrypt 哈希
    throw new ApiAuthException(trans('errors.api_incorrect_token_secret'));
}
```

**安全设计要点**：
- `token_id` 明文存储，作为数据库查询索引
- `secret` 仅在创建时以明文展示一次，数据库中存 bcrypt 哈希
- 客户端须自己保管 `{token_id}:{secret}` 完整凭证
- Token 凭证格式为 `Token {token_id}:{secret}`，两部分均为 32 字符随机串

### 3.4 过期 Token 清理机制

**结论：BookStack 没有定时任务清理过期 API Token。**

过期检查仅发生在请求时校验路径：

```php
// ApiTokenGuard::validateToken() — app/Api/ApiTokenGuard.php:130-133
$now = Carbon::now();
if ($token->expires_at <= $now) {
    throw new ApiAuthException(trans('errors.api_user_token_expired'), 403);
}
```

验证方式：
1. `app/Console/Kernel.php` 的 `schedule()` 方法为空，无任何定时任务注册
2. `app/Console/Commands/` 目录下无 Token 清理命令
3. 全局搜索无 `CleanupApiTokens`、`prune` token 等相关逻辑

过期的 Token 记录会永久保留在数据库中，仅在请求时通过 `expires_at` 字段拒绝访问。默认过期时间为 100 年（`app/Api/ApiToken.php:43-46`），因此实际场景下 Token 过期并不常见。若需清理，只能通过用户在 UI 中手动删除（`UserApiTokenController::destroy`）。

### 3.5 Token scope 字段与 API 路径鉴权细分

**核心结论：BookStack 的 API Token 没有 scope 字段，API 路径的鉴权细分完全依赖用户角色权限系统。**

#### 3.5.1 Token 数据库字段全景

`api_tokens` 表只有以下字段（`database/migrations/2019_12_29_120917_add_api_auth.php`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 自增主键 |
| `user_id` | bigint | 所属用户 |
| `name` | varchar | Token 名称（用户自定义） |
| `token_id` | varchar(32) | 明文 Token ID，用于查询 |
| `secret` | varchar | bcrypt 哈希后的 secret |
| `expires_at` | timestamp | 过期时间 |
| `created_at` / `updated_at` | timestamp | 时间戳 |

**没有 `scope`、`permissions`、`endpoint` 等任何限定 Token 权限范围的字段。**

#### 3.5.2 API 鉴权细分的实际实现路径

API 的权限细分完全通过**用户角色权限系统**实现，Token 只承担身份认证的职责：

```
Token 认证 → 识别出用户 → 查询用户角色 → 角色权限决定能访问哪些 API
```

权限细分的三层机制（详见第四章）：

1. **系统级**：`Permission::AccessApi` — 能否访问 API 整体
2. **实体级**：`Permission::PageView`、`Permission::BookCreate` 等 — 对各类实体的操作权限
3. **实例级**：实体 ACL（entity_permissions 表）— 对特定实体的权限覆盖

**同一用户的所有 Token 拥有完全相同的权限**，无法为不同 Token 分配不同的 API 访问范围。

#### 3.5.3 API 路由层面的权限控制点

API 路由本身没有额外的 scope 校验中间件。权限检查发生在两个层面：

**层面 1：认证阶段的系统级检查**（`app/Api/ApiTokenGuard.php:135-137`）
```php
if (!$token->user->can(Permission::AccessApi)) {
    throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
}
```

**层面 2：控制器中的细粒度检查**（以 `PageApiController` 为例）
- `index()` / `list()` → `findVisibleBy...` → 通过 `restrictEntityQuery` 过滤可见实体
- `read()` → `findVisibleByIdOrFail` + 隐式权限检查
- `create()` → `checkOwnablePermission(Permission::PageCreate, $parent)`
- `update()` → `checkOwnablePermission(Permission::PageUpdate, $page)`
- `delete()` → `checkOwnablePermission(Permission::PageDelete, $page)`

每个 API 端点的权限由业务逻辑内嵌的 `checkOwnablePermission` / `restrictEntityQuery` 控制，而非 Token scope。

#### 3.5.4 设计影响

- **优点**：权限模型统一，Web 和 API 使用同一套角色权限体系
- **局限**：无法创建"只读 Token"、"特定范围 Token"等受限凭证
- **替代方案**：可通过创建低权限用户 + 为该用户生成 Token 的方式间接实现

### 3.6 Token 模型

**文件**：`app/Api/ApiToken.php`

```php
class ApiToken extends Model
{
    protected $fillable = ['name', 'expires_at'];
    protected $casts = ['expires_at' => 'date:Y-m-d'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

### 3.7 API Token vs User Token vs Webhook：三种 Token 类概念辨析

BookStack 代码中存在多种 "token" 概念，容易混淆。下表从**用途、存储、有效期、生成方式**四个维度进行对比：

| 维度 | API Token (Personal Access Token) | User Token (邮件/重置令牌) | Webhook |
|------|-----------------------------------|---------------------------|---------|
| **用途** | API 接口调用认证 | 邮箱确认、密码重置、邀请等一次性操作 | 出站事件通知（不是认证凭证） |
| **存储表** | `api_tokens` | `user_tokens` | `webhooks` + `webhook_tracked_events` |
| **服务类** | `ApiTokenGuard` + `UserApiTokenController` | `UserTokenService` | `DispatchWebhookJob` + `WebhookFormatter` |
| **生成方式** | `Str::random(32)` × 2 (token_id + secret) | `Str::random(24-25)` | 无 token，只配置 endpoint URL |
| **存储形式** | `token_id` 明文 + `secret` bcrypt 哈希 | `token` 明文存储 | 无 secret，直接 POST 到 URL |
| **有效期** | 默认 100 年 (可配置) | 24 小时 (`$expiryTime = 24`) | 无过期，永久有效（webhook 配置本身） |
| **认证方向** | 入站（外部调用 BookStack API） | 入站（用户点击邮件链接） | 出站（BookStack 调用外部服务） |
| **关联用户** | 有 `user_id` 外键，1:N | 有 `user_id` 外键，1:N | 有 `initiator`（触发者），但不是认证关系 |

#### 3.7.1 User Token 详解

**文件**：`app/Access/UserTokenService.php`

User Token 是用于邮件确认、密码重置等的一次性令牌，与 API Token 完全是两套独立体系：

```php
// UserTokenService — 核心逻辑
protected int $expiryTime = 24;    // 24 小时过期

protected function generateToken(): string
{
    $token = Str::random(24);
    while ($this->tokenExists($token)) {
        $token = Str::random(25);     // 碰撞后增加到 25 位
    }
    return $token;
}

protected function entryExpired(stdClass $tokenEntry): bool
{
    return Carbon::now()->subHours($this->expiryTime)
        ->gt(new Carbon($tokenEntry->created_at));
}
```

**关键差异**：
- **明文存储**：User Token 的 `token` 字段明文存储（因为需要直接查询），不像 API Token 的 secret 用 bcrypt
- **一次性语义**：使用后通常由业务逻辑删除（`deleteByUser`）
- **短有效期**：24 小时，远短于 API Token 的 100 年

#### 3.7.2 Webhook 详解

**文件**：`app/Activity/DispatchWebhookJob.php` + `app/Activity/Models/Webhook.php`

Webhook 是**出站通知机制**，不是入站认证凭证。它没有 token/secret 字段，BookStack 直接向配置的 endpoint 发送 POST 请求：

```php
// DispatchWebhookJob::handle() — 发送逻辑
$client = $http->buildClient($this->webhook->timeout, [
    'connect_timeout' => 10,
    'allow_redirects' => ['strict' => true],
]);

$response = $client->sendRequest(
    $http->jsonRequest('POST', $this->webhook->endpoint, $this->webhookData)
);
```

Webhook 配置仅包含：`name`、`endpoint`（URL）、`timeout`、`active`、`tracked_events`，**没有任何认证 token**。接收方如果需要认证，只能通过在 URL 中加查询参数或通过 `ThemeEvents::WEBHOOK_CALL_BEFORE` 主题事件自定义。

> **注意**：用户口中的 "webhook token" 在 BookStack 中并不存在。如果讨论的是"通过 webhook 调用 API"，那实际使用的是普通 API Token。

### 3.8 Guest 用户与 Admin Token 的权限隔离边界

#### 3.8.1 Guest 用户的本质

**文件**：`app/Users/Models/User.php:95-106` + `app/App/Providers/AuthServiceProvider.php:68-70`

Guest（游客）不是一种 Token 类型，而是一个**系统内置的特殊用户**：

```php
public static function getGuest(): self
{
    return app()->make('users.default');
}

public function isGuest(): bool
{
    return $this->system_name === 'public';
}
```

- `system_name = 'public'` 的用户就是 Guest 用户
- 以单例形式注册在容器中，整个请求共享
- 未登录用户的 `user()` 辅助函数返回的就是这个 Guest 用户

**Guest 用户不能创建 API Token**——因为无法登录 UI 也无法通过 API 管理 Token。换句话说，BookStack **不存在 "guest API key" 这种东西**。未携带有效 Token 的 API 请求会直接被 `ApiAuthenticate` 中间件拒绝。

#### 3.8.2 Guest 用户的权限矩阵

Guest 用户的权限来自其绑定的 **Public 角色**（`system_name = 'public'`），这是一个系统角色。权限由两个层面控制：

| 控制层 | 机制 | 说明 |
|--------|------|------|
| 应用级开关 | `app-public` 设置 | 决定整个应用是否对游客开放 |
| 角色权限 | Public 角色的 permission 列表 | 具体能看什么、做什么 |
| 实体级 ACL | `entity_permissions` 表 | 特定实体对 Public 角色的权限覆盖 |

`User::hasAppAccess()` 方法体现了这两层判断：

```php
public function hasAppAccess(): bool
{
    return !$this->isGuest() || setting('app-public');
}
```

- 非游客：永远有基础访问权
- 游客：必须 `app-public` 开启才有基础访问权

#### 3.8.3 Admin Token 与普通用户 Token 的隔离边界

Admin Token 和普通用户 Token 在**认证机制上完全相同**——都是 API Token，都走 `ApiTokenGuard`。区别仅在于**用户角色权限不同**：

```
Admin Token → 识别出 admin 用户 → admin 角色 → 拥有全部权限
普通用户 Token → 识别出普通用户 → 普通角色 → 有限权限
Guest → 无 Token → 不能调用 API
```

**隔离边界在权限系统，不在 Token 系统**：

1. **系统级**：`$user->can(Permission::SettingsManage)` 等系统权限
2. **实体级**：`page-create-all` / `page-delete-own` 等角色权限
3. **实例级**：`entity_permissions` 表的实体级 ACL 覆盖

Admin 角色的特殊地位体现在 `EntityPermissionEvaluator` 的快捷放行：

```php
// EntityPermissionEvaluator::isUserSystemAdmin()
protected function isUserSystemAdmin($userRoleIds): bool
{
    $adminRoleId = Role::getSystemRole('admin')->id;
    return in_array($adminRoleId, $userRoleIds);
}
```

如果用户是 admin 角色，`evaluateEntityForUser()` 直接返回 `true`，跳过整个 ACL 计算。

---

## 四、权限判断层

权限判断分为三个层级，层层递进：

### 4.1 系统级权限（在 Token 认证时已校验）

Token 认证阶段已校验用户是否有 `Permission::AccessApi` 权限（`app/Api/ApiTokenGuard.php:135-137`）：

```php
if (!$token->user->can(Permission::AccessApi)) {
    throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
}
```

### 4.2 路由/中间件级权限

**文件**：`app/Http/Middleware/CheckUserHasPermission.php`

通过 `can` 中间件别名注册（`app/Http/Kernel.php:56`）：

```php
'can' => \BookStack\Http\Middleware\CheckUserHasPermission::class,
```

使用方式（在路由或控制器构造函数中）：

```php
Route::get('/example', [Controller::class, 'action'])->middleware('can:access-api');
```

Permission 枚举提供了 `middleware()` 辅助方法（`app/Permissions/Permission.php:140-143`）：

```php
public function middleware(): string
{
    return 'can:' . $this->value;
}
```

### 4.3 控制器/实体级权限（最常用）

**核心方法**：`checkOwnablePermission` — 定义在 `app/Http/Controller.php:83-88`

```php
protected function checkOwnablePermission(string|Permission $permission, Model $ownable, string $redirectLocation = '/'): void
{
    if (!userCan($permission, $ownable)) {
        $this->showPermissionError($redirectLocation);
    }
}
```

**辅助函数**：`userCan` — 定义在 `app/App/helpers.php:43-53`

```php
function userCan(string|Permission $permission, ?Model $ownable = null): bool
{
    if (is_null($ownable)) {
        return user()->can($permission);
    }
    $permissions = app()->make(PermissionApplicator::class);
    return $permissions->checkOwnableUserAccess($ownable, $permission);
}
```

实体权限检查示例（`app/Entities/Controllers/PageApiController.php:72-87`）：

```php
public function create(Request $request)
{
    $this->validate($request, $this->rules['create']);

    if ($request->has('chapter_id')) {
        $parent = $this->entityQueries->chapters->findVisibleByIdOrFail(intval($request->input('chapter_id')));
    } else {
        $parent = $this->entityQueries->books->findVisibleByIdOrFail(intval($request->input('book_id')));
    }
    $this->checkOwnablePermission(Permission::PageCreate, $parent);

    $draft = $this->pageRepo->getNewDraftPage($parent);
    $this->pageRepo->publishDraft($draft, $request->only(array_keys($this->rules['create'])));

    return response()->json($draft->forJsonDisplay());
}
```

### 4.4 Permission 枚举

**文件**：`app/Permissions/Permission.php`

定义了系统中所有权限类型，主要分类：

| 分类 | 示例 |
|------|------|
| 系统权限 | `AccessApi`, `ContentExport`, `SettingsManage`, `UsersManage` |
| 实体权限 | `BookView/Create/Update/Delete`, `PageView/Create/Update/Delete` 等 |
| 非实体内容权限 | `AttachmentCreate/Update/Delete`, `ImageCreate/Update/Delete`, `CommentCreate/Update/Delete` |

权限支持 `All/Own` 后缀（如 `PageCreateAll`, `PageCreateOwn`），用于区分"创建所有"和"创建自己的"。

### 4.5 Entity 级 ACL 计算与 PermissionApplicator 协作

实体级权限判断是 BookStack 权限体系中最复杂的部分，涉及 `PermissionApplicator`、`EntityPermissionEvaluator`、`JointPermissionBuilder` 三层协作。

#### 4.5.1 整体调用链

```
Controller::checkOwnablePermission(Permission, Entity)
  → helpers::userCan(Permission, Entity)
    → PermissionApplicator::checkOwnableUserAccess(Entity, Permission)    ← 权限入口
      ├─ 角色级权限判断 (user->can('page-create-all') / 'page-create-own')
      ├─ 非实体权限快捷路径 (attachment/image/comment)
      └─ EntityPermissionEvaluator::evaluateEntityForUser(Entity, roleIds) ← 实体级 ACL
           ├─ 管理员快捷放行
           ├─ 构建实体继承链 (page → chapter → book)
           ├─ 查询 EntityPermission 表
           └─ 按 fallback / role 分类归约判定
```

#### 4.5.2 PermissionApplicator — 权限入口与角色级判断

**文件**：`app/Permissions/PermissionApplicator.php`

`checkOwnableUserAccess()` 方法是所有实体权限判断的唯一入口。它首先做**角色级权限判断**，这是独立于实体级 ACL 的前置检查：

```php
// PermissionApplicator::checkOwnableUserAccess() — 核心逻辑
$allRolePermission = $user->can($fullPermission . '-all');    // 如 page-create-all
$ownRolePermission = $user->can($fullPermission . '-own');    // 如 page-create-own
$isOwner = $user->id === $ownable->getAttribute($ownerField);
$hasRolePermission = $allRolePermission || ($isOwner && $ownRolePermission);
```

对于**非实体权限**（attachment/image/comment/restrictions），直接返回角色级判断结果，不进入实体级 ACL：

```php
$nonJointPermissions = ['restrictions', 'image', 'attachment', 'comment'];
if (in_array($explodedPermission[0], $nonJointPermissions)) {
    return $hasRolePermission;
}
```

对于**实体权限**（book/chapter/page/shelf），角色级权限只是"默认值"，如果实体上有针对性的 ACL 规则（EntityPermission），则以 ACL 规则为准：

```php
$hasApplicableEntityPermissions = $this->hasEntityPermission($ownable, $userRoleIds, $action);
return is_null($hasApplicableEntityPermissions) ? $hasRolePermission : $hasApplicableEntityPermissions;
```

- `hasEntityPermission` 返回 `null` → 该实体无针对性 ACL 规则，使用角色级权限
- `hasEntityPermission` 返回 `bool` → 有 ACL 规则，以 ACL 判定为准（可覆盖角色权限）

#### 4.5.3 EntityPermissionEvaluator — 实体级 ACL 计算

**文件**：`app/Permissions/EntityPermissionEvaluator.php`

这是实体级 ACL 的核心计算引擎。当实体上存在针对性权限设置时，通过以下步骤判定：

**第一步：构建实体继承链**

```php
// EntityPermissionEvaluator::gatherEntityChainTypeIds()
$chain = [$entity->type . ':' . $entity->id];        // 自身优先级最高

if ($entity->type === 'page' && $entity->chapter_id) {
    $chain[] = 'chapter:' . $entity->chapter_id;       // page → chapter
}
if ($entity->type === 'page' || $entity->type === 'chapter') {
    $chain[] = 'book:' . $entity->book_id;              // page/chapter → book
}

return $chain;
```

继承链的优先级：**自身 > 父级 chapter > 祖父级 book**。越靠前优先级越高。

**第二步：查询 EntityPermission 表**

`EntityPermission` 模型（`app/Permissions/Models/EntityPermission.php`）记录了"某个实体 × 某个角色"的权限设定：

| 字段 | 说明 |
|------|------|
| `entity_type` + `entity_id` | 目标实体标识 |
| `role_id` | 角色 ID（`0` 表示 fallback/默认规则） |
| `view` / `create` / `update` / `delete` | 布尔值，对应操作的允许/拒绝 |

查询时按继承链中的所有实体 ID + 用户角色 ID（含 `role_id=0` 的 fallback）批量获取：

```php
EntityPermission::query()
    ->where('entity_type', '=', $type)
    ->whereIn('entity_id', $ids)
    ->where(function ($query) use ($filterRoleIds) {
        $query->whereIn('role_id', [...$filterRoleIds, 0]);  // 含 fallback
    })
    ->get(['entity_id', 'entity_type', 'role_id', $this->action]);
```

**第三步：归约判定 — collapseAndCategorisePermissions**

将查询到的权限按 `fallback`（role_id=0）和 `role`（role_id>0）分类，并沿继承链从高优先级到低优先级遍历：

```php
foreach ($typeIdChain as $typeId) {
    foreach ($permissions as $permission) {
        $type = $roleId === 0 ? 'fallback' : 'role';
        if (!isset($permitsByType[$type][$roleId])) {
            $permitsByType[$type][$roleId] = $permission->{$this->action};
        }
    }
    if (isset($permitsByType['fallback'][0])) {
        break;    // 找到 fallback 规则即停止继承链遍历
    }
}
```

关键语义：**一旦在某层级找到 fallback 规则（role_id=0），就停止向上追溯**。这实现了"子实体设定了默认权限则不再继承父级"的逻辑。

**第四步：最终判定 — evaluatePermitsByType**

```php
// 角色级规则优先
if (count($permitsByType['role']) > 0) {
    return max($permitsByType['role'])     // 任一角色允许 → EXPLICIT_ALLOW
        ? PermissionStatus::EXPLICIT_ALLOW
        : PermissionStatus::EXPLICIT_DENY;
}
// 其次 fallback 规则
if (count($permitsByType['fallback']) > 0) {
    return $permitsByType['fallback'][0]
        ? PermissionStatus::IMPLICIT_ALLOW
        : PermissionStatus::IMPLICIT_DENY;
}
// 无任何 ACL 规则
return null;  // 回退到角色级权限
```

`PermissionStatus`（`app/Permissions/PermissionStatus.php`）定义了四级状态：

| 常量 | 值 | 含义 |
|------|---|------|
| `IMPLICIT_DENY` | 0 | 隐式拒绝（fallback 规则拒绝） |
| `IMPLICIT_ALLOW` | 1 | 隐式允许（fallback 规则允许） |
| `EXPLICIT_DENY` | 2 | 显式拒绝（角色级规则拒绝） |
| `EXPLICIT_ALLOW` | 3 | 显式允许（角色级规则允许） |

角色级规则 > fallback 规则；多角色中任一允许即判定允许。

#### 4.5.4 嵌套场景下的权限继承递归路径

实体权限继承链的深度和方向是固定的，**没有真正的递归，只有线性的 2-3 步向上追溯**。

##### 继承链完整图谱

```
Bookshelf
    │
    │  ← 不在 ACL 继承链中！通过"主动复制"机制传递权限
    ▼
  Book
    │
    │  ← 继承链第 3 级（优先级最低）
    ▼
 Chapter
    │
    │  ← 继承链第 2 级
    ▼
   Page    ← 继承链第 1 级（自身，优先级最高）
```

不同实体类型的实际继承链：

| 实体类型 | 继承链 | 长度 |
|---------|--------|------|
| Page（有 chapter） | page → chapter → book | 3 层 |
| Page（无 chapter，直接在 book 下） | page → book | 2 层 |
| Chapter | chapter → book | 2 层 |
| Book | book | 1 层 |
| Bookshelf | bookshelf | 1 层 |

##### 追溯终止条件

继承链的向上追溯不是一定走完全程，**遇到 fallback 规则（role_id=0）就停止**：

```
假设：chapter 上设置了 fallback 权限，book 上也设置了 fallback 权限

page 权限评估流程：
  1. 检查 page 自身 → 无 fallback → 继续向上
  2. 检查 chapter  → 有 fallback → 停止！不再看 book
  3. 以 chapter 的 fallback 规则为准
```

这意味着：**子实体的 fallback 规则会"屏蔽"父实体的 fallback 规则**，实现了"越具体的规则优先级越高"的语义。

##### Bookshelf 的特殊地位：主动复制而非实时继承

**Shelf 不在 ACL 实时继承链中**。评估 book/chapter/page 的权限时，永远不会向上追溯到 shelf。

Shelf → Book 的权限传递通过**主动复制**机制实现：

```php
// PermissionsUpdater::updateBookPermissionsFromShelf() — app/Entities/Tools/PermissionsUpdater.php:145-163
public function updateBookPermissionsFromShelf(Bookshelf $shelf, $checkUserPermissions = true): int
{
    $shelfPermissions = $shelf->permissions()->get([...])->toArray();
    $shelfBooks = $shelf->books()->get(['id', 'owned_by']);

    foreach ($shelfBooks as $book) {
        if ($checkUserPermissions && !userCan(Permission::RestrictionsManage, $book)) {
            continue;
        }
        $book->permissions()->delete();          // 清空 book 原有权限
        $book->permissions()->createMany($shelfPermissions);  // 写入 shelf 的权限副本
        $book->rebuildPermissions();               // 重建 joint_permissions 缓存
        $updatedBookCount++;
    }
    return $updatedBookCount;
}
```

**触发方式**（3 种）：

| 触发方式 | 代码位置 | 说明 |
|---------|---------|------|
| UI 手动点击 | `PermissionsController::copyShelfPermissionsToBooks()` | 用户在 shelf 权限页面点击"复制到书籍" |
| CLI 命令 | `CopyShelfPermissionsCommand` | `php artisan bookstack:copy-shelf-permissions` |
| 代码调用 | 直接调用 `PermissionsUpdater::updateBookPermissionsFromShelf()` | 程序内部调用 |

**重要特性**：
- 这是**一次性快照复制**，不是实时联动
- 复制后 book 的权限独立存在，后续修改 shelf 权限不会自动同步到 book
- 复制时会跳过当前用户无 `RestrictionsManage` 权限的 book
- book 下的 chapter/page 通过正常的继承链自动继承 book 的新权限（通过 `rebuildPermissions()` 触发 `JointPermissionBuilder` 更新缓存）

##### 没有更深的嵌套

BookStack 的实体层级是**扁平的固定结构**，不存在深层递归：
- Book 不能包含 Book
- Chapter 不能包含 Chapter
- Page 不能包含 Page
- Shelf 不能包含 Shelf

因此权限计算也永远是 O(1) 的固定几步查询，没有递归深度问题。

#### 4.5.5 JointPermissionBuilder — 预计算视图权限缓存

**文件**：`app/Permissions/JointPermissionBuilder.php`

`JointPermission` 表（`joint_permissions`）是**视图权限（view）的预计算缓存**，用于列表查询时的权限过滤，避免每次列表请求都实时计算 ACL。

```
┌────────────────────────────────────────────────────────────────┐
│ JointPermission 预计算流程                                      │
│                                                                │
│  JointPermissionBuilder::rebuildForAll()                       │
│    → 遍历所有 Book（含 chapters、pages）                        │
│    → 遍历所有 Bookshelf                                        │
│    → 对每个 Entity × Role 组合：                                │
│       1. admin 角色 → EXPLICIT_ALLOW                           │
│       2. MassEntityPermissionEvaluator 评估实体级 ACL           │
│          → 返回非 null → 使用该状态                             │
│       3. ACL 无结果 → 查角色权限 (book-view-all / book-view-own)│
│          → IMPLICIT_ALLOW / IMPLICIT_DENY                      │
│    → 批量插入 joint_permissions 表                              │
│                                                                │
│  触发时机：                                                     │
│    - rebuildForAll()：权限重建命令/系统更新时                    │
│    - rebuildForEntity()：实体权限变更时                         │
│    - rebuildForRole()：角色权限变更时                           │
└────────────────────────────────────────────────────────────────┘
```

`joint_permissions` 表结构：

| 字段 | 说明 |
|------|------|
| `entity_id` + `entity_type` | 实体标识（复合主键的一部分） |
| `role_id` | 角色 ID |
| `status` | PermissionStatus 值 (0-3) |
| `owner_id` | 当角色有 `-own` 权限时记录所有者 ID |

**查询过滤**使用 `PermissionApplicator::restrictEntityQuery()`：

```php
public function restrictEntityQuery(Builder $query): Builder
{
    return $query->where(function (Builder $parentQuery) {
        $parentQuery->whereHas('jointPermissions', function (Builder $permissionQuery) {
            $permissionQuery->select(['entity_id', 'entity_type'])
                ->selectRaw('max(owner_id) as owner_id')
                ->selectRaw('max(status) as status')
                ->whereIn('role_id', $this->getCurrentUserRoleIds())
                ->groupBy(['entity_type', 'entity_id'])
                ->havingRaw('(status IN (1, 3) or (owner_id = ? and status != 2))',
                    [$this->currentUser()->id]);
        });
    });
}
```

条件语义：**用户任一角色的 status 为 IMPLICIT_ALLOW(1) 或 EXPLICIT_ALLOW(3)**，或者 **owner_id 匹配当前用户且 status 不是 EXPLICIT_DENY(2)**。

#### 4.5.6 实体级 ACL 的两条路径对比

| 场景 | 使用机制 | 数据来源 | 适用范围 |
|------|---------|---------|---------|
| 单实体操作（read/update/delete） | `EntityPermissionEvaluator` 实时计算 | `entity_permissions` 表 | 精确判定单个实体的 view/create/update/delete |
| 列表/集合查询（list/search） | `JointPermission` 预计算缓存 | `joint_permissions` 表 | 仅 view 权限，用于批量过滤可见实体 |

两条路径共享同一个 `EntityPermissionEvaluator` 核心逻辑（`MassEntityPermissionEvaluator` 是其在批量场景下的优化子类），保证判定结果一致。

---

## 五、完整请求处理协作流程

### 5.1 时序图

```
客户端                   ThrottleApiRequests          ApiAuthenticate              ApiTokenGuard              Controller
   │                           │                            │                          │                         │
   │─── API 请求 ──────────────>│                            │                          │                         │
   │                           │ 1. 检查速率限制              │                          │                         │
   │                           │    (按 user_id 或 IP)       │                          │                         │
   │                           │                            │                          │                         │
   │                           │──── 超过限制 ───────────────│                          │                         │
   │<── 429 Too Many Requests ──┤                            │                          │                         │
   │                           │                            │                          │                         │
   │                           │──── 未超限 ────────────────>│                          │                         │
   │                           │                            │ 2. Session 认证?         │                         │
   │                           │                            │    ├─ 是 → 检查权限+方法  │                         │
   │                           │                            │    └─ 否 → Token 认证 ──>│                         │
   │                           │                            │                          │ 3. 解析 Token          │
   │                           │                            │                          │    ├─ 格式校验           │
   │                           │                            │                          │    ├─ 哈希校验           │
   │                           │                            │                          │    ├─ 过期校验           │
   │                           │                            │                          │    ├─ 邮箱确认检查       │
   │                           │                            │                          │    └─ AccessApi 权限    │
   │                           │                            │                          │                         │
   │                           │                            │<── 认证失败 ─────────────│                         │
   │<── 401/403 Unauthorized ───┤<───────────────────────────┤                          │                         │
   │                           │                            │                          │                         │
   │                           │                            │──── 认证通过 ────────────>│                         │
   │                           │                            │                          │                         │
   │                           │                            │                          │ 4. 控制器级权限检查     │
   │                           │                            │                          │    checkOwnablePermission│
   │                           │                            │                          │    (实体级权限)         │
   │                           │                            │                          │                         │
   │                           │                            │                          │<── 权限不足 ────────────│
   │<── 403 Forbidden ──────────┤<───────────────────────────┤<─────────────────────────┤                         │
   │                           │                            │                          │                         │
   │                           │                            │                          │──── 通过 ──────────────>│
   │                           │                            │                          │                         │ 5. 业务处理
   │<── 200 JSON Response ──────┤<───────────────────────────┤<─────────────────────────┤<────────────────────────│
```

### 5.2 API 错误响应：Token 失效 vs Rate Limit 的错误码区分

所有 API 错误响应都由 `Handler::renderApiException()` 统一格式化，结构一致：

```json
{
    "error": {
        "code": 429,
        "message": "Too Many Attempts."
    }
}
```

**核心识别方式**：通过 `error.code` 字段（即 HTTP 状态码）区分错误类型。

#### 5.2.1 各类错误的状态码与触发路径

| 错误类型 | 异常类 | HTTP 状态码 | 触发位置 | message 典型内容 |
|---------|--------|------------|---------|-----------------|
| **Token 格式错误** | `ApiAuthException` | 401 | `ApiTokenGuard::validateTokenHeaderValue()` | "No authorization token found" / "Bad authorization format" |
| **Token 不存在** | `ApiAuthException` | 401 | `ApiTokenGuard::validateToken()` | "Token not found" |
| **Token secret 错误** | `ApiAuthException` | 401 | `ApiTokenGuard::validateToken()` | "Incorrect token secret" |
| **Token 已过期** | `ApiAuthException` | **403** | `ApiTokenGuard::validateToken()` | "Token expired" |
| **用户无 API 权限** | `ApiAuthException` | **403** | `ApiTokenGuard::validateToken()` | "No API permission" |
| **邮箱未确认** | `ApiAuthException` | 401 | `ApiTokenGuard::user()` | "Email confirmation awaiting" |
| **速率限制超限** | `ThrottleRequestsException` (Laravel 内置) | **429** | `ThrottleRequests::handle()` | "Too Many Attempts." |
| **实体权限不足** | `NotifyException` | **403** | `Controller::showPermissionError()` | 权限拒绝的具体描述 |
| **数据不存在** | `ModelNotFoundException` | 404 | 模型查询 | "" (空消息) |
| **验证失败** | `ValidationException` | 422 | 表单验证 | "The given data was invalid." + `validation` 字段 |

#### 5.2.2 Token 失效的状态码差异

一个容易混淆的点：**Token 相关错误不都是 401**，部分是 403。具体规则：

```
ApiAuthException 状态码分配：
  ├─ 认证凭证问题（格式/不存在/secret错） → 401
  ├─ 邮箱未确认 → 401
  ├─ Token 已过期 → 403
  └─ 用户无 AccessApi 权限 → 403
```

代码中的体现（`app/Api/ApiTokenGuard.php`）：

```php
// 格式错误 → 401（默认）
throw new ApiAuthException(trans('errors.api_no_authorization_found'));

// Token 不存在 → 401（默认）
throw new ApiAuthException(trans('errors.api_token_not_found'));

// Token 过期 → 403
throw new ApiAuthException(trans('errors.api_user_token_expired'), 403);

// 无 API 权限 → 403
throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
```

设计逻辑：401 = "你是谁无法确认"，403 = "知道你是谁，但你不能做这件事"。

#### 5.2.3 Rate Limit 触发的响应特征

速率限制超限由 Laravel 框架的 `ThrottleRequests` 中间件抛出 `ThrottleRequestsException`，状态码固定为 **429**。

响应中额外包含限流相关的 HTTP 响应头（Laravel 自动设置）：

```
X-RateLimit-Limit: 180
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1718888888
Retry-After: 45
```

其中：
- `X-RateLimit-Limit`：每分钟配额上限
- `X-RateLimit-Remaining`：当前窗口剩余次数
- `X-RateLimit-Reset`：窗口重置的 Unix 时间戳
- `Retry-After`：建议的重试等待秒数

这些 header 通过 `$e->getHeaders()` 传入 `renderApiException`，最终原样输出到响应中。

#### 5.2.4 统一响应格式化路径

所有 API 异常的处理入口是 `Handler::render()` → `isApiRequest()` → `renderApiException()`：

```php
// Handler::renderApiException() — app/Exceptions/Handler.php:119-148
protected function renderApiException(Throwable $e): JsonResponse
{
    $code = 500;
    $headers = [];

    if ($e instanceof HttpExceptionInterface) {
        $code = $e->getStatusCode();   // 从异常中读取状态码
        $headers = $e->getHeaders();   // 从异常中读取额外 header（如限流头）
    }

    if ($e instanceof ModelNotFoundException) {
        $code = 404;
    }

    $responseData = ['error' => ['message' => $e->getMessage()]];

    if ($e instanceof ValidationException) {
        $responseData['error']['message'] = 'The given data was invalid.';
        $responseData['error']['validation'] = $e->errors();
        $code = $e->status;
    }

    $responseData['error']['code'] = $code;  // code 字段等于 HTTP 状态码

    return new JsonResponse($responseData, $code, $headers);
}
```

**关键结论**：
- `error.code` 与 HTTP 状态码始终一致
- 区分 Token 失效和 Rate Limit：**看状态码是 4xx 还是 429**
- 区分不同 Token 失效原因：看 `error.message` 文本，状态码只是粗粒度区分
- 限流错误额外带 `X-RateLimit-*` 和 `Retry-After` 响应头

### 5.3 协作关键点总结

1. **速率限制最先执行**：在任何认证逻辑之前，避免恶意请求消耗认证资源。

2. **认证方式双通道**：
   - Session 认证（Cookie）：仅允许 GET 请求，便于浏览器直接访问 API
   - Token 认证（Authorization Header）：支持所有 HTTP 方法

3. **权限分层校验**：
   - **第一层**（认证阶段）：检查 `AccessApi` 系统权限 — 确保用户能访问 API
   - **第二层**（路由阶段）：通过 `can` 中间件做粗粒度权限过滤
   - **第三层**（控制器阶段）：通过 `checkOwnablePermission` 做实体级细粒度权限校验

4. **限流标识联动**：速率限制中间件优先使用认证后的 `user()->id` 作为限流 key，未认证则回退到 IP 地址。但由于限流在认证之前执行，Token 认证场景下限流 Key 始终为 IP，而非用户 ID。

5. **异常处理**：
   - 速率限制：返回 `429 Too Many Requests`（Laravel 框架默认），带 `X-RateLimit-*` 响应头
   - Token 认证失败：`ApiAuthException` 实现 `HttpExceptionInterface`，由 `Handler::renderApiException()` 统一转换为 JSON 响应 `{'error': {'message': '...', 'code': 401/403}}`
   - 权限不足：`NotifyException` 同样实现 `HttpExceptionInterface`，转换为 `{'error': {'message': '...', 'code': 403}}`
   - 所有 API 错误响应结构统一：`{error: {code, message}}`，code 等于 HTTP 状态码

6. **Token 安全**：secret 以 bcrypt 哈希存储，明文仅创建时展示一次；无定时清理过期 Token 的机制，过期检查仅在请求时执行。

7. **实体级 ACL 双路径**：单实体操作实时计算（EntityPermissionEvaluator），列表查询使用预计算缓存（JointPermission），两条路径共享同一评估核心保证一致性。

8. **Token 无 scope**：API Token 没有 scope 字段，权限细分完全依赖用户角色权限体系，同一用户的所有 Token 权限相同。

9. **Shelf 权限主动复制**：Bookshelf 不在 ACL 实时继承链中，shelf → book 的权限通过主动复制机制传递，复制后独立存在。

10. **限流全局共享**：所有 API 路径共享同一个限流计数器，与 URL、参数、方法无关；重试会持续消耗配额。

11. **三种 Token 概念**：API Token（API 认证，长有效期）、User Token（邮件确认/重置，24 小时过期）、Webhook（出站通知，无 token），三者是完全独立的体系。

12. **Guest 无 API Token**：BookStack 不存在 "guest API key"，未认证 API 请求直接被拒绝。Admin Token 与普通 Token 的差异在权限层，不在认证层。

---

## 六、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| API Token Guard | `app/Api/ApiTokenGuard.php` |
| API Token 模型 | `app/Api/ApiToken.php` |
| Token 生成与管理控制器 | `app/Api/UserApiTokenController.php` |
| API 认证中间件 | `app/Http/Middleware/ApiAuthenticate.php` |
| API 限流中间件 | `app/Http/Middleware/ThrottleApiRequests.php` |
| Session 按需启动中间件 | `app/Http/Middleware/StartSessionIfCookieExists.php` |
| 权限检查中间件 | `app/Http/Middleware/CheckUserHasPermission.php` |
| 邮箱确认检查 | `app/Http/Middleware/CheckEmailConfirmed.php` |
| HTTP 中间件注册 | `app/Http/Kernel.php` |
| 控制器基类（权限方法） | `app/Http/Controller.php` |
| Permission 枚举 | `app/Permissions/Permission.php` |
| PermissionStatus 常量 | `app/Permissions/PermissionStatus.php` |
| 权限应用器 | `app/Permissions/PermissionApplicator.php` |
| 实体权限评估器 | `app/Permissions/EntityPermissionEvaluator.php` |
| 批量实体权限评估器 | `app/Permissions/MassEntityPermissionEvaluator.php` |
| 联合权限构建器 | `app/Permissions/JointPermissionBuilder.php` |
| 实体权限模型 | `app/Permissions/Models/EntityPermission.php` |
| 联合权限模型 | `app/Permissions/Models/JointPermission.php` |
| 轻量实体数据 | `app/Permissions/SimpleEntityData.php` |
| 权限更新工具 | `app/Entities/Tools/PermissionsUpdater.php` |
| 权限控制器（Web） | `app/Permissions/PermissionsController.php` |
| Shelf 模型 | `app/Entities/Models/Bookshelf.php` |
| API 认证异常 | `app/Exceptions/ApiAuthException.php` |
| 全局异常处理器 | `app/Exceptions/Handler.php` |
| 路由服务提供者（限流配置） | `app/App/Providers/RouteServiceProvider.php` |
| 认证配置 | `app/Config/auth.php` |
| API 配置 | `app/Config/api.php` |
| 缓存配置 | `app/Config/cache.php` |
| 辅助函数（userCan） | `app/App/helpers.php` |
| API 路由 | `routes/api.php` |
| 定时任务调度 | `app/Console/Kernel.php` |
| 数据库迁移（api_tokens） | `database/migrations/2019_12_29_120917_add_api_auth.php` |
| 复制 Shelf 权限命令 | `app/Console/Commands/CopyShelfPermissionsCommand.php` |
| MFA 限流器（参考实现） | `app/Access/Mfa/MfaVerificationLimiter.php` |
| User Token 服务（邮件/重置令牌） | `app/Access/UserTokenService.php` |
| User Token 过期异常 | `app/Exceptions/UserTokenExpiredException.php` |
| Webhook 模型 | `app/Activity/Models/Webhook.php` |
| Webhook 任务调度 | `app/Activity/DispatchWebhookJob.php` |
| Webhook 格式化器 | `app/Activity/Tools/WebhookFormatter.php` |
| Webhook 控制器（Web） | `app/Activity/Controllers/WebhookController.php` |
| 用户模型（含 Guest 逻辑） | `app/Users/Models/User.php` |
| 角色模型 | `app/Users/Models/Role.php` |
| 认证服务提供者 | `app/App/Providers/AuthServiceProvider.php` |
| NotifyException（权限拒绝） | `app/Exceptions/NotifyException.php` |
