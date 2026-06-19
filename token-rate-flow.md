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

### 3.5 Token 模型

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

#### 4.5.4 JointPermissionBuilder — 预计算视图权限缓存

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

#### 4.5.5 实体级 ACL 的两条路径对比

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

### 5.2 协作关键点总结

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
   - 速率限制：返回 `429 Too Many Requests`（Laravel 框架默认）
   - Token 认证失败：`ApiAuthException` 实现 `HttpExceptionInterface`，由 `Handler::renderApiException()` 统一转换为 JSON 响应 `{'error': {'message': '...', 'code': 401/403}}`
   - 权限不足：`NotifyException` 同样实现 `HttpExceptionInterface`，转换为 `{'error': {'message': '...', 'code': 403}}`

6. **Token 安全**：secret 以 bcrypt 哈希存储，明文仅创建时展示一次；无定时清理过期 Token 的机制，过期检查仅在请求时执行。

7. **实体级 ACL 双路径**：单实体操作实时计算（EntityPermissionEvaluator），列表查询使用预计算缓存（JointPermission），两条路径共享同一评估核心保证一致性。

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
| API 认证异常 | `app/Exceptions/ApiAuthException.php` |
| 全局异常处理器 | `app/Exceptions/Handler.php` |
| 路由服务提供者（限流配置） | `app/App/Providers/RouteServiceProvider.php` |
| 认证配置 | `app/Config/auth.php` |
| API 配置 | `app/Config/api.php` |
| 辅助函数（userCan） | `app/App/helpers.php` |
| API 路由 | `routes/api.php` |
| 定时任务调度 | `app/Console/Kernel.php` |
