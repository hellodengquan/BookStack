# BookStack 多 Provider 认证体系全链路梳理

BookStack 同时支持 **OIDC**、**SAML2**、**LDAP** 三种外部身份方案以及标准（standard）本地认证。整体架构围绕 Laravel Guard 体系构建，通过 `AUTH_METHOD` 环境变量选定唯一主认证方式，再辅以 Socialite 社交登录作为附加入口。下面按代码调用顺序逐层拆解。

---

## 1. 认证入口与 Provider 分流

### 1.1 全局分流锚点：`AUTH_METHOD`

**文件**: `app/Config/auth.php:14`

```php
'method' => env('AUTH_METHOD', 'standard'),
```

`AUTH_METHOD` 取值 `standard | ldap | saml2 | oidc`，它同时决定了：

| 作用点 | 代码位置 | 效果 |
|--------|----------|------|
| 默认 Guard | `auth.defaults.guard` = `AUTH_METHOD` | `auth()` 无参调用时使用的 Guard |
| 中间件 `guard:xxx` | `app/Http/Middleware/CheckGuard.php:20` | 校验当前 `auth.method` 是否在允许列表中，不在则 403 |
| 登录页自动跳转 | `LoginService::shouldAutoInitiate()` | 当 `auth.auto_initiate=true` 且无社交驱动时，oidc/saml2 自动跳转 |
| 登录表单字段 | `LoginController::username()` | standard → `email`，其余 → `username` |

### 1.2 路由层分流

**文件**: `routes/web.php:330-363`

四类认证入口拥有各自独立路由：

```
┌─────────────────────────────────────────────────────────────────┐
│ 标准 + LDAP                                                     │
│   GET  /login           → LoginController@getLogin              │
│   POST /login           → LoginController@login                 │
│   POST /logout          → LoginController@logout                │
│   中间件: guard:standard,ldap (login)                            │
│           guard:standard,ldap,oidc (logout)                     │
├─────────────────────────────────────────────────────────────────┤
│ Social（Google/GitHub/Azure 等）                                 │
│   GET  /login/service/{socialDriver}       → SocialController@login    │
│   GET  /login/service/{socialDriver}/callback → SocialController@callback│
│   POST /login/service/{socialDriver}/detach → SocialController@detach   │
│   GET  /register/service/{socialDriver}     → SocialController@register │
├─────────────────────────────────────────────────────────────────┤
│ SAML2                                                           │
│   POST /saml2/login     → Saml2Controller@login                 │
│   POST /saml2/logout    → Saml2Controller@logout                │
│   GET  /saml2/metadata  → Saml2Controller@metadata              │
│   POST /saml2/acs       → Saml2Controller@startAcs (免CSRF/Session)│
│   GET  /saml2/acs       → Saml2Controller@processAcs            │
│   GET  /saml2/sls       → Saml2Controller@sls                   │
│   中间件: guard:saml2                                            │
├─────────────────────────────────────────────────────────────────┤
│ OIDC                                                            │
│   POST /oidc/login      → OidcController@login                  │
│   GET  /oidc/callback   → OidcController@callback               │
│   POST /oidc/logout     → OidcController@logout                 │
│   中间件: guard:oidc                                             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Guard 注册与驱动映射

**文件**: `app/Config/auth.php:33-53` + `app/App/Providers/AuthServiceProvider.php:17-55`

```
AUTH_METHOD  →  Guard 名称  →  Guard 驱动              →  User Provider
────────────────────────────────────────────────────────────────────────
standard     →  standard    →  session (Laravel内置)     →  users (Eloquent)
ldap         →  ldap        →  ldap-session             →  external
saml2        →  saml2       →  async-external-session   →  external
oidc         →  oidc        →  async-external-session   →  external
```

- **`ldap-session`** → `LdapSessionGuard`：同步式，在 `attempt()` 中完成 LDAP 绑定验证 + 本地用户查找/创建 + 组同步
- **`async-external-session`** → `AsyncExternalBaseSessionGuard`：异步式，`attempt()` / `validate()` 永远返回 `false`，所有认证逻辑在 Controller → Service 层手动完成，Guard 只负责 session 读写
- **`external` Provider** → `ExternalBaseUserProvider`：通过 `external_auth_id` 字段查找用户，不做密码校验

### 1.4 CheckGuard 中间件

**文件**: `app/Http/Middleware/CheckGuard.php`

```php
public function handle($request, Closure $next, ...$allowedGuards)
{
    $activeGuard = config('auth.method');
    if (!in_array($activeGuard, $allowedGuards)) {
        session()->flash('error', trans('errors.permission'));
        return redirect('/');
    }
    return $next($request);
}
```

效果：当 `AUTH_METHOD=oidc` 时，用户无法访问 `/login`（POST），因为 `LoginController@login` 限制 `guard:standard,ldap`；同理 `AUTH_METHOD=standard` 时 `/saml2/login` 和 `/oidc/login` 均不可达。**这是一种互斥式分流**——同一时刻只能有一个主认证 Provider 生效。

### 1.5 自动跳转逻辑

**文件**: `app/Access/LoginService.php:211-222`

```php
public function shouldAutoInitiate(): bool
{
    $autoRedirect = config('auth.auto_initiate');
    if (!$autoRedirect) return false;
    $socialDrivers = $this->socialDriverManager->getActive();
    $authMethod = config('auth.method');
    return count($socialDrivers) === 0 && in_array($authMethod, ['oidc', 'saml2']);
}
```

条件：`AUTH_AUTO_INITIATE=true` + 无活跃社交驱动 + 主认证为 oidc 或 saml2 → 登录页直接渲染 `auth.login-initiate` 视图并自动发起外部认证流程。

---

## 2. 外部账号到本地用户的映射

所有外部认证的核心问题：**外部身份信息如何对应到 `users` 表中的一行？**

### 2.1 统一锚点：`external_auth_id`

**文件**: `app/Users/Models/User.php:42`

`users` 表有一个 `external_auth_id` 字符串字段，LDAP / SAML2 / OIDC 三者均通过此字段关联外部身份：

| Provider | `external_auth_id` 来源 | 配置键 | 默认值 |
|----------|------------------------|--------|--------|
| LDAP | `id_attribute` 对应的 LDAP 属性值，缺省为用户 DN | `services.ldap.id_attribute` | `dn` |
| SAML2 | `external_id_attribute` 对应的 SAML 属性值，缺省为 NameID | `saml2.external_id_attribute` | `null` → NameID |
| OIDC | `external_id_claim` 对应的 ID Token 声明值 | `oidc.external_id_claim` | `sub` |

### 2.2 LDAP 映射链路

**文件**: `app/Access/Guards/LdapSessionGuard.php:62-99`

```
用户输入 username + password
  → LdapSessionGuard::attempt()
    → LdapService::getUserDetails(username)
      → LDAP 搜索 → 返回 {uid, name, dn, email}
    → ExternalBaseUserProvider::retrieveByCredentials(['external_auth_id' => uid])
      → SELECT * FROM users WHERE external_auth_id = ?
    → 若用户不存在 → createNewFromLdapAndCreds()
      → RegistrationService::registerUser([... 'external_auth_id' => uid])
    → 若邮箱为空 → 抛出 LoginAttemptEmailNeededException（要求补填 email）
```

关键细节：
- `uid` 优先取 `id_attribute`（如 `objectguid`），取不到则回退到 `dn`
- 新用户密码设为 `Str::random(32)`，不可用于本地登录
- 已有用户通过 `external_auth_id` 精确匹配，**不检查邮箱**

### 2.3 SAML2 映射链路

**文件**: `app/Access/Saml2Service.php:343-381`

```
IdP ACS Response
  → Saml2Controller::startAcs() → 缓存 SAMLResponse → redirect → processAcs()
  → Saml2Service::processAcsResponse(requestId, samlResponse)
    → Onelogin Toolkit 解析 → getNameId() + getAttributes()
    → getUserDetails(samlID, attrs)
      → external_id = getExternalId(attrs, samlID)
        → 若 external_id_attribute 配置了 → 取该属性值
        → 否则 → 用 NameID（samlID）
      → email = getSamlResponseAttribute(attrs, email_attribute)
      → name = getUserDisplayName(attrs, external_id)
    → RegistrationService::findOrRegister(name, email, external_id)
      → SELECT * FROM users WHERE external_auth_id = ?
      → 不存在 → registerUser()
```

关键细节：
- `samlID`（NameID）始终保留但 `external_auth_id` 可被 `external_id_attribute` 覆盖
- 邮箱为空时直接抛出 `SamlException`，不会像 LDAP 那样允许补充
- `findOrRegister` 是 SAML2/OIDC 共用的方法，先查 `external_auth_id`，再创建

### 2.4 OIDC 映射链路

**文件**: `app/Access/Oidc/OidcService.php:176-242`

```
Authorization Code 回调
  → OidcController::callback() → state 校验（3分钟过期）
  → OidcService::processAuthorizeResponse(code)
    → OAuth2 Token Exchange → OidcAccessToken
    → processAccessTokenCallback(accessToken, settings)
      → 解析 ID Token → OidcIdToken::validate(clientId)
      → getUserDetailsFromToken(idToken, accessToken, settings)
        → OidcUserDetails::populate(idToken, external_id_claim, display_name_claims, groups_claim)
        → 若信息不完整且 userinfo_endpoint 存在 → 再调 userinfo 端点补充
      → RegistrationService::findOrRegister(name, email, externalId)
```

关键细节：
- `externalId` 来自 `sub` 声明（默认），可通过 `OIDC_EXTERNAL_ID_CLAIM` 自定义
- 支持通过 Theme 事件 `OIDC_ID_TOKEN_PRE_VALIDATE` 在验证前修改 claims
- ID Token 文本存入 session（`oidc_id_token`），登出时用于 RP-initiated logout
- 已登录用户再访问回调会抛 `OidcException`，防止重复登录

### 2.5 Social（Socialite）映射链路

**文件**: `app/Access/SocialAuthService.php:90-140`

Social 登录（Google/GitHub/Azure 等）使用**完全不同的映射机制**：

```
回调 → SocialAuthService::handleLoginCallback(driver, socialUser)
  → SocialAccount::where('driver_id', socialUser.getId()).first()
  → 若 socialAccount 存在且用户未登录 → loginService->login(socialAccount->user)
  → 若用户已登录且无 socialAccount → 创建 SocialAccount 关联到当前用户
  → 若无 socialAccount 且未登录 → 抛 SocialSignInAccountNotUsed
    → 若 auto_register 开启 → socialRegisterCallback()
      → registerUser(userData, socialAccount, emailVerified)
```

与 LDAP/SAML2/OIDC 的关键区别：
- **不使用 `external_auth_id`**，而是通过 `social_accounts` 表（`driver` + `driver_id`）做关联
- 社交登录与主认证方式**可以并存**，不受 `AUTH_METHOD` 互斥限制
- `SocialAccount` 是独立的 Eloquent Model，通过 `user_id` 外键关联到 `users` 表

### 2.6 映射机制对比总结

| 维度 | LDAP | SAML2 | OIDC | Social |
|------|------|-------|------|--------|
| 关联字段 | `users.external_auth_id` | `users.external_auth_id` | `users.external_auth_id` | `social_accounts.driver_id` |
| 查找方式 | `WHERE external_auth_id = uid` | `WHERE external_auth_id = externalId` | `WHERE external_auth_id = sub` | `WHERE driver_id = socialId` |
| 新用户创建 | `registerUser()` | `findOrRegister()` | `findOrRegister()` | `registerUser() + SocialAccount` |
| 邮箱缺失时 | 抛异常，允许补充 | 直接拒绝 | 直接拒绝 | 取邮箱 @ 前缀 |
| 与 AUTH_METHOD 互斥 | 是 | 是 | 是 | 否，可附加 |

---

## 3. 组与角色的同步

所有外部 Provider 的组同步最终汇聚到 **`GroupSyncService::syncUserWithFoundGroups()`**。

**文件**: `app/Access/GroupSyncService.php`

### 3.1 统一同步入口

```php
public function syncUserWithFoundGroups(User $user, array $userGroups, bool $detachExisting): void
{
    $groupsAsRoles = $this->matchGroupsToSystemsRoles($userGroups);
    if ($detachExisting) {
        $user->roles()->sync($groupsAsRoles);
        $user->attachDefaultRole();
    } else {
        $user->roles()->syncWithoutDetaching($groupsAsRoles);
    }
}
```

### 3.2 外部组名 → 本地角色的匹配规则

**文件**: `app/Access/GroupSyncService.php:15-24`

```
外部组名规范化: "Admin Users" → "admin-users"（小写 + 空格换连字符）
                    ↓
匹配逻辑: 对每个 Role
  ① 若 role.external_auth_id 非空
     → 解析为逗号分隔列表（支持 \, 转义）
     → 任一值匹配规范化后的组名 → 命中
  ② 若 role.external_auth_id 为空
     → 规范化 role.display_name
     → 与规范化后的组名做 in_array 比较
```

举例：

| 外部组名 | Role display_name | Role external_auth_id | 是否匹配 |
|---------|-------------------|-----------------------|---------|
| `Admin Users` | `Admin Users` | `null` | ✅ (admin-users == admin-users) |
| `Admin Users` | `Viewer` | `admin-users` | ✅ (external_auth_id 命中) |
| `Admin, VIP` | `Admin` | `null` | ❌ (admin,-vip ≠ admin) |
| `Admin, VIP` | `Some Role` | `admin\, vip` | ✅ (external_auth_id 包含 "admin, vip") |

### 3.3 各 Provider 的组提取方式

#### LDAP

**文件**: `app/Access/LdapService.php:339-461`

```
LdapService::getUserGroups(username)
  → getUserWithAttributes(username, [group_attribute])
  → extractGroupsFromSearchResponseEntry() → 取 group_attribute 多值
  → getGroupsRecursive() → 递归查找父组
    → getParentsOfGroup(groupDN) → LDAP read 操作
  → extractGroupNamesFromLdapGroupDns() → 从 DN 提取第一个 RDN 值
```

特点：
- 支持**递归组解析**（嵌套组结构）
- 从 DN 中提取组名（`CN=Admins,OU=Groups,DC=example` → `Admins`）
- 配置项：`LDAP_GROUP_ATTRIBUTE`，`LDAP_USER_TO_GROUPS`，`LDAP_REMOVE_FROM_GROUPS`

#### SAML2

**文件**: `app/Access/Saml2Service.php:290-300`

```php
public function getUserGroups(array $samlAttributes): array
{
    $groupsAttr = $this->config['group_attribute'];
    $userGroups = $samlAttributes[$groupsAttr] ?? null;
    if (!is_array($userGroups)) $userGroups = [];
    return $userGroups;
}
```

特点：
- 直接从 SAML Assertion 属性中取组名数组
- 配置项：`SAML2_GROUP_ATTRIBUTE`，`SAML2_USER_TO_GROUPS`，`SAML2_REMOVE_FROM_GROUPS`

#### OIDC

**文件**: `app/Access/Oidc/OidcUserDetails.php:62-76`

```php
protected static function getUserGroups(string $groupsClaim, ProvidesClaims $claims): ?array
{
    if (empty($groupsClaim)) return null;
    $groupsList = Arr::get($claims->getAllClaims(), $groupsClaim);
    if (!is_array($groupsList)) return null;
    return array_values(array_filter($groupsList, fn($val) => is_string($val)));
}
```

特点：
- 从 ID Token 或 Userinfo 响应的 claims 中按路径取组数组
- 只接受字符串元素，过滤掉数字等类型
- 配置项：`OIDC_GROUPS_CLAIM`，`OIDC_USER_TO_GROUPS`，`OIDC_REMOVE_FROM_GROUPS`

### 3.4 组同步行为对比

| 维度 | LDAP | SAML2 | OIDC |
|------|------|-------|------|
| 触发位置 | `LdapSessionGuard::attempt()` | `Saml2Service::processLoginCallback()` | `OidcService::processAccessTokenCallback()` |
| 触发条件 | `shouldSyncGroups()` = `enabled && user_to_groups !== false` | `user_to_groups !== false` | `user_to_groups !== false` |
| 组来源 | LDAP 搜索 + 递归 | SAML Assertion 属性 | ID Token / Userinfo claims |
| detachExisting | `LDAP_REMOVE_FROM_GROUPS` | `SAML2_REMOVE_FROM_GROUPS` | `OIDC_REMOVE_FROM_GROUPS` |
| 递归支持 | ✅ 递归父组 | ❌ | ❌ |
| Social 有组同步? | ❌ 无 | — | — |

### 3.5 detachExisting 的影响

- `detachExisting = false`（默认）：`syncWithoutDetaching` → 只添加新角色，不删除已有角色
- `detachExisting = true`：`sync` → 完全替换角色集合为外部组对应的角色，再追加默认角色

---

## 4. 各 Provider 回调与登出链路差异

### 4.1 登录回调链路

#### LDAP（同步式）

```
POST /login {username, password}
  → LoginController::login()
    → LoginService::attempt(credentials, 'ldap')
      → auth()->attempt() → LdapSessionGuard::attempt()
        → LdapService::getUserDetails()
        → ExternalBaseUserProvider::retrieveByCredentials()
        → LdapService::validateUserCredentials() (LDAP Bind)
        → 若新用户 → createNewFromLdapAndCreds()
        → LdapService::syncGroups()
        → LdapService::saveAndAttachAvatar()
        → Guard::login(user)
      → auth()->logout() ← 退出 Guard 级别 session
      → LoginService::login(user, 'ldap') ← 统一登录流程
        → MFA / 邮箱确认检查
        → auth()->login(user)
        → 管理员跨 Guard 同步登录
```

#### SAML2（异步式，两步回调）

```
POST /saml2/login
  → Saml2Controller::login()
    → Saml2Service::login() → Onelogin Auth::login()
    → session flash saml2_request_id
    → redirect → IdP

IdP → POST /saml2/acs (无 Session/Cookie 上下文)
  → Saml2Controller::startAcs()
    → 缓存 SAMLResponse 到 cache (10分钟)
    → redirect → /saml2/acs?id=xxx

GET /saml2/acs?id=xxx (有 Session 上下文)
  → Saml2Controller::processAcs()
    → cache pull → decrypt → 获取 SAMLResponse
    → Saml2Service::processAcsResponse(requestId, samlResponse)
      → Onelogin processResponse → 验签
      → getNameId() + getAttributes()
      → processLoginCallback()
        → getUserDetails()
        → findOrRegister()
        → GroupSyncService::syncUserWithFoundGroups()
        → LoginService::login(user, 'saml2')
```

**两步回调原因**：IdP POST 回调时浏览器不带当前会话 Cookie（SameSite 策略），所以先缓存 SAMLResponse，再通过 302 重定向带 Session 处理。

#### OIDC（异步式，单步回调）

```
POST /oidc/login
  → OidcController::login()
    → OidcService::login() → OAuth2 getAuthorizationUrl()
    → session: oidc_state, oidc_pkce_code
    → redirect → Authorization Server

Authorization Server → GET /oidc/callback?code=xxx&state=yyy
  → OidcController::callback()
    → 校验 state（3分钟过期）
    → OidcService::processAuthorizeResponse(code)
      → Token Exchange (code → access_token + id_token)
      → OidcIdToken::validate()
      → getUserDetailsFromToken() → 可能补充 userinfo
      → findOrRegister()
      → GroupSyncService::syncUserWithFoundGroups()
      → LoginService::login(user, 'oidc')
```

#### Social（异步式，单步回调）

```
GET /login/service/{driver}
  → SocialController::login()
    → session: social-callback = 'login'
    → Socialite redirect → OAuth Provider

OAuth Provider → GET /login/service/{driver}/callback
  → SocialController::callback()
    → session pull social-callback
    → SocialAuthService::getSocialUser() → Socialite user()
    → handleLoginCallback(driver, socialUser)
      → SocialAccount 查找
      → 若已关联 → loginService->login()
      → 若未关联且 auto_register → socialRegisterCallback()
        → registerUser() + loginService->login()
```

### 4.2 登出链路

#### 通用入口

```
POST /logout
  → LoginController::logout()
    → LoginService::logout()
      → auth()->logout()          ← 清除所有 Guard session
      → session()->invalidate()   ← 销毁整个 session
      → session()->regenerateToken()
      → 若 shouldAutoInitiate() → redirect '/login?prevent_auto_init=true'
      → 否则 → redirect '/'
```

#### SAML2 登出（SP-Initiated Single Logout）

```
POST /saml2/logout
  → Saml2Controller::logout()
    → Saml2Service::logout(user)
      → 先调用 LoginService::logout() 清除本地 session
      → Onelogin Auth::logout(returnUrl, [], user->email, sessionIndex)
        → 若 IdP 支持 SLO → 返回 IdP 登出 URL
        → 若不支持 (SAML_SINGLE_LOGOUT_NOT_SUPPORTED) → 返回本地登出 URL
      → session flash saml2_logout_request_id
    → redirect → IdP SLO URL 或本地 URL

IdP → GET /saml2/sls
  → Saml2Controller::sls()
    → Saml2Service::processSlsResponse(requestId)
      → Onelogin processSLO() → 验签
      → 返回 IdP 指定的 redirect 或本地默认
```

关键：SAML2 登出先清本地 session，再跳 IdP；如果 IdP 不支持 SLO 则直接返回本地 URL。

#### OIDC 登出（RP-Initiated Logout）

```
POST /oidc/logout
  → OidcController::logout()
    → OidcService::logout()
      → session pull oidc_id_token
      → LoginService::logout() → 清除本地 session
      → 若 endSessionEndpoint 存在:
        → 拼接: {endSessionEndpoint}?id_token_hint={id_token}&post_logout_redirect_uri={本地登出URL}
        → 返回此 URL
      → 若不存在 → 返回本地登出 URL
    → redirect → OIDC End Session URL 或本地 URL
```

配置 `OIDC_END_SESSION_ENDPOINT`：
- `false`（默认）：强制禁用 RP-initiated logout
- `true`：从 discovery 获取
- 字符串：直接使用

#### LDAP 登出

LDAP 没有外部登出协议。直接走通用入口 `LoginService::logout()`，清除本地 session 即可。

#### Social 登出

Social 登录没有外部登出协议。同样走通用入口 `LoginService::logout()`。

### 4.3 管理员跨 Guard 登录

**文件**: `app/Access/LoginService.php:54-59`

```php
if ($user->can(Permission::UsersManage) && $user->can(Permission::UserRolesManage)) {
    $guards = ['standard', 'ldap', 'saml2', 'oidc'];
    foreach ($guards as $guard) {
        auth($guard)->login($user);
    }
}
```

当管理员用户登录时，同时在所有 Guard 的 session 中写入登录态，确保管理员能访问受任何 Guard 保护的端点（如 SAML2/OIDC 的登出路由中间件 `guard:saml2`/`guard:oidc`）。

### 4.4 回调与登出对比总结

| 维度 | LDAP | SAML2 | OIDC | Social |
|------|------|-------|------|--------|
| 回调方式 | 同步（表单提交） | 异步两步（POST→缓存→GET处理） | 异步单步（GET回调） | 异步单步（GET回调） |
| State/Request 校验 | 无 | `saml2_request_id` (session flash) | `oidc_state` (session, 3分钟TTL) | `social-callback` (session) |
| 外部登出协议 | 无 | SAML SLO (SP-initiated) | OIDC RP-Initiated Logout | 无 |
| 登出是否先清本地 | N/A | ✅ 先 `LoginService::logout()` | ✅ 先 `LoginService::logout()` | N/A |
| Session 特殊存储 | 无 | `saml2_session_index` | `oidc_id_token`, `oidc_pkce_code` | `social-callback` |
| CSRF 豁免 | 否 | ✅ `/saml2/acs` 豁免 CSRF + Session | 否 | 否 |

---

## 5. 整体调用时序图

```
                           ┌──────────────────┐
                           │  AUTH_METHOD      │
                           │  (env 变量)       │
                           └────────┬─────────┘
                                    │
              ┌─────────────────────┼──────────────────────┐
              │                     │                      │
         ┌────▼────┐          ┌─────▼─────┐         ┌──────▼─────┐
         │ standard│          │   ldap    │         │saml2 / oidc│
         │  Guard  │          │  Guard    │         │   Guard    │
         │(session)│          │(ldap-     │         │(async-     │
         │         │          │ session)  │         │ external-  │
         │         │          │           │         │ session)   │
         └────┬────┘          └─────┬─────┘         └──────┬─────┘
              │                     │                      │
         POST /login          POST /login           POST /saml2/login
         (email+pass)         (username+pass)       POST /oidc/login
              │                     │                      │
              ▼                     ▼                      ▼
         Eloquent             LdapService            Service::login()
         UserProvider         ::getUserDetails()      → redirect to IdP
              │                     │                      │
              │                     ▼                      ▼
              │              ExternalBase           IdP 回调
              │              UserProvider           /saml2/acs
              │              (external_auth_id)     /oidc/callback
              │                     │                      │
              ▼                     ▼                      ▼
         LoginService          LoginService          RegistrationService
         ::attempt()           ::attempt()           ::findOrRegister()
              │                     │                      │
              ▼                     ▼                      ▼
         ┌──────────────────────────────────────────────────────┐
         │              LoginService::login()                   │
         │  ┌─────────────────────────────────────────┐        │
         │  │ 1. MFA 检查                              │        │
         │  │ 2. 邮箱确认检查                          │        │
         │  │ 3. auth()->login()                       │        │
         │  │ 4. 管理员跨 Guard 同步                    │        │
         │  │ 5. Activity 日志 + Theme 事件派发          │        │
         │  └─────────────────────────────────────────┘        │
         └──────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  GroupSyncService │  ← LDAP/SAML2/OIDC 均调用
                    │  ::syncUserWith   │
                    │  FoundGroups()    │
                    └──────────────────┘
```

---

## 6. 关键代码索引

| 模块 | 文件路径 |
|------|---------|
| 认证配置 | `app/Config/auth.php` |
| OIDC 配置 | `app/Config/oidc.php` |
| SAML2 配置 | `app/Config/saml2.php` |
| LDAP 配置 | `config/services.php` 中的 `ldap` 键 |
| Guard 注册 | `app/App/Providers/AuthServiceProvider.php` |
| Guard 中间件 | `app/Http/Middleware/CheckGuard.php` |
| 登录控制器 | `app/Access/Controllers/LoginController.php` |
| OIDC 控制器 | `app/Access/Controllers/OidcController.php` |
| SAML2 控制器 | `app/Access/Controllers/Saml2Controller.php` |
| Social 控制器 | `app/Access/Controllers/SocialController.php` |
| 登录服务 | `app/Access/LoginService.php` |
| OIDC 服务 | `app/Access/Oidc/OidcService.php` |
| SAML2 服务 | `app/Access/Saml2Service.php` |
| LDAP 服务 | `app/Access/LdapService.php` |
| 组同步服务 | `app/Access/GroupSyncService.php` |
| 注册服务 | `app/Access/RegistrationService.php` |
| Social 认证服务 | `app/Access/SocialAuthService.php` |
| 外部用户 Provider | `app/Access/ExternalBaseUserProvider.php` |
| LDAP Guard | `app/Access/Guards/LdapSessionGuard.php` |
| 异步外部 Guard | `app/Access/Guards/AsyncExternalBaseSessionGuard.php` |
| Guard 基类 | `app/Access/Guards/ExternalBaseSessionGuard.php` |
| SocialAccount 模型 | `app/Access/SocialAccount.php` |
| OIDC 用户详情 | `app/Access/Oidc/OidcUserDetails.php` |
| 路由定义 | `routes/web.php` |

---

## 7. 跨 Provider 冲突分析

由于 BookStack 同时承载 OIDC、SAML2、LDAP 和 Social 四套身份体系，存在多类跨 Provider 的冲突场景。下面按代码路径逐一还原。

### 7.1 场景一：同一邮箱或用户名从不同 Provider 登录

#### 7.1.1 核心分支代码

**`RegistrationService::findOrRegister()`** (`app/Access/RegistrationService.php:53-71`)

```php
public function findOrRegister(string $name, string $email, string $externalId): User
{
    $user = User::query()
        ->where('external_auth_id', '=', $externalId)
        ->first();          // ← 只按 external_auth_id 查，不查 email

    if (is_null($user)) {
        $userData = [
            'name'             => $name,
            'email'            => $email,
            'password'         => Str::random(32),
            'external_auth_id' => $externalId,
        ];
        $user = $this->registerUser($userData, null, false);
    }
    return $user;
}
```

**`RegistrationService::registerUser()`** (`app/Access/RegistrationService.php:78-125`)

```php
public function registerUser(array $userData, ?SocialAccount $socialAccount = null, ...): User
{
    $userEmail = $userData['email'];
    ...
    // Ensure the user does not already exist
    $alreadyUser = !is_null($this->userRepo->getByEmail($userEmail));
    if ($alreadyUser) {
        throw new UserRegistrationException(
            trans('errors.error_user_exists_different_creds', ['email' => $userEmail]),
            '/login'
        );
    }
    ...
}
```

**LDAP 分支** (`app/Access/Guards/LdapSessionGuard.php:78-127`)

```php
if (is_null($user)) {
    $user = $this->createNewFromLdapAndCreds($userDetails, $credentials);
    // createNewFromLdapAndCreds 内部同样调用 $this->registrationService->registerUser()
    // registerUser 里同样会触发 email 唯一校验
}
```

#### 7.1.2 冲突矩阵

设用户的邮箱为 `alice@example.com`。下表展示不同组合下的行为：

| 已存在用户的来源 | 新登录尝试的 Provider | external_auth_id 是否相同 | 结果 |
|-----------------|----------------------|--------------------------|------|
| OIDC (ext_id=`alice_oidc`) | SAML2 (ext_id=`alice_saml`) | ❌ 不同 | `findOrRegister` 查 `external_auth_id=alice_saml` 未果 → `registerUser` → 查到 email 已存在 → **抛异常拒绝** |
| OIDC (ext_id=`alice_oidc`) | LDAP (ext_id=`uid=alice`) | ❌ 不同 | `LdapSessionGuard::attempt` 查 external_auth_id 未果 → `createNewFromLdapAndCreds` → `registerUser` → email 已存在 → **抛异常拒绝** |
| SAML2 (ext_id=`alice@example.com`) | OIDC (ext_id=`alice@example.com`) | ✅ 相同（双方都用邮箱作 external_id） | `findOrRegister` 按 external_auth_id 命中已有用户 → **复用同一记录**，不会检查 email |
| 标准本地注册 (email=`alice@example.com`, external_auth_id=`""`) | OIDC (ext_id=`alice_oidc`) | ❌ 不同 | OIDC `findOrRegister` 查 external_auth_id 未果 → `registerUser` → email 已存在 → **抛异常拒绝** |

#### 7.1.3 冲突结论

- **默认策略：拒绝创建，不自动合并**。只要两个 Provider 产生的 `external_auth_id` 不同，即使用户名/邮箱完全一致，也会在 `registerUser` 中被 email 唯一校验拦下。
- **只有一种例外**：如果管理员把两个 Provider 的 `external_id_claim` / `external_id_attribute` / `id_attribute` 都配置为同一个值（比如都是邮箱），使得两边产生的 `external_auth_id` 完全相同，则 `findOrRegister` 会命中已有记录并复用。
- **LDAP 的额外特例**：如果 LDAP 目录没有返回 email，`LoginAttemptEmailNeededException` 会先抛出来让用户补填 email；而如果补填的 email 已被其他 Provider 占用，同样会被 `registerUser` 的 email 校验拦住。

### 7.2 场景二：同一用户在不同 Provider 获得不同角色

#### 7.2.1 生效前提

由于 `AUTH_METHOD` 互斥（详见 1.2 路由层分流和 `CheckGuard` 中间件），**同一时间只能激活一个外部 Provider**。因此"同一用户同时从 OIDC 和 LDAP 登录"在正常部署中不会发生。冲突只会在以下场景出现：

1. 管理员修改 `AUTH_METHOD` 环境变量（例如从 `oidc` 切到 `ldap`），用户先后用两个 Provider 登录
2. 用户已经通过 OIDC 创建了账号，管理员又配置 Social 登录并开启 `auto_register`，用户再通过 Social 同邮箱尝试注册
3. 管理员同时配置 Social + 外部主 Provider（Social 不互斥），两边返回不同组名

#### 7.2.2 组同步的最终决定权代码

**`GroupSyncService::syncUserWithFoundGroups()`** (`app/Access/GroupSyncService.php:73-85`)

```php
public function syncUserWithFoundGroups(User $user, array $userGroups, bool $detachExisting): void
{
    $groupsAsRoles = $this->matchGroupsToSystemsRoles($userGroups);

    if ($detachExisting) {
        $user->roles()->sync($groupsAsRoles);     // ← 完全覆盖
        $user->attachDefaultRole();                // ← 再补默认角色
    } else {
        $user->roles()->syncWithoutDetaching($groupsAsRoles);  // ← 只加不减
    }
}
```

这段代码是**所有 Provider 共享的最终落库路径**，没有任何 Provider 级别的分支。

#### 7.2.3 `detachExisting` 配置决定冲突结果

每个 Provider 独立配置自己的 `remove_from_groups`（LDAP 的 `LDAP_REMOVE_FROM_GROUPS`、SAML2 的 `SAML2_REMOVE_FROM_GROUPS`、OIDC 的 `OIDC_REMOVE_FROM_GROUPS`）。冲突时谁生效取决于 **最后一次成功登录使用的是哪个 Provider，以及该 Provider 的 `detachExisting` 开关**。

| 场景 | detachExisting | 最终角色 |
|------|---------------|---------|
| OIDC 返回 `[Admin]` 登录 → LDAP 返回 `[Viewer]` 登录，两边均为 `true` | 每次都覆盖 | 最后一次（LDAP）同步的 `[Viewer + 默认角色]`，Admin 被删除 |
| OIDC 返回 `[Admin]` 登录 (detach=true) → LDAP 返回 `[Viewer]` 登录 (detach=false) | OIDC 覆盖，LDAP 叠加 | `[Admin + Viewer]` |
| OIDC 返回 `[Admin]` 登录 (detach=false) → LDAP 返回 `[Viewer]` 登录 (detach=true) | OIDC 叠加，LDAP 覆盖 | `[Viewer + 默认角色]`，Admin 被删除 |
| 两边 detach 均为 false | 始终叠加 | `[Admin + Viewer + 其他手动分配的角色]` |

#### 7.2.4 冲突结论

- **最终角色完全由 `GroupSyncService::syncUserWithFoundGroups` 的两个参数决定**：`$userGroups`（当前 Provider 返回的组列表）和 `$detachExisting`（当前 Provider 的 `remove_from_groups` 配置）。
- **哪个 Provider 最后完成登录，哪个 Provider 的配置就生效**。之前登录过的 Provider 的组同步结果不会保留任何"优先级"。
- **Social 登录不参与组同步**——Social 注册只调 `attachDefaultRole()`，不调用 `syncUserWithFoundGroups`，所以 Social 永远不会影响用户的角色集合（除非 Social 登录前用户已有角色，detach=true 模式下另一个 Provider 会覆盖它们）。

### 7.3 场景三：Social 账号体系与 `external_auth_id` 体系的冲突

#### 7.3.1 两套身份关联方式

```
OIDC / SAML2 / LDAP         Social (Socialite)
───────────────────         ──────────────────
users.external_auth_id      social_accounts.driver
                            social_accounts.driver_id
```

- `external_auth_id` 是 `users` 表的单列字符串，**一个用户只能存一个值**
- Social 账号在独立 `social_accounts` 表里，**一个用户可以绑定多个 Social 账号**（一行一条 `driver + driver_id`）

#### 7.3.2 Social 注册/登录时的冲突代码

**Social 注册校验** (`app/Access/SocialAuthService.php:56-70`)

```php
public function handleRegistrationCallback(string $socialDriver, SocialUser $socialUser): SocialUser
{
    if (SocialAccount::query()->where('driver_id', '=', $socialUser->getId())->exists()) {
        throw new UserRegistrationException(...);
    }
    // 同样检查 email 唯一性
    if (User::query()->where('email', '=', $socialUser->getEmail())->exists()) {
        throw new UserRegistrationException(
            trans('errors.error_user_exists_different_creds', ['email' => $email]),
            '/login'
        );
    }
    return $socialUser;
}
```

**Social 已登录状态下绑定** (`app/Access/SocialAuthService.php:111-117`)

```php
// When a user is logged in but the social account does not exist,
// Create the social account and attach it to the user & redirect to the profile page.
if ($isLoggedIn && $socialAccount === null) {
    $account = $this->newSocialAccount($socialDriver, $socialUser);
    $currentUser->socialAccounts()->save($account);
    ...
}
```

#### 7.3.3 冲突矩阵

| 已有账号 | 新操作 | 结果 |
|---------|--------|------|
| OIDC 用户 (email=`a@x.com`, ext_id=`a_oidc`) | 用同邮箱的 Google 账号走 Social 注册 | `handleRegistrationCallback` 查 email → 已存在 → **抛异常拒绝** |
| OIDC 用户（已登录） | 绑定 GitHub Social 账号 | 走 `$isLoggedIn && $socialAccount === null` 分支 → **成功绑定**，不创建新用户 |
| Social 用户（Google 注册，email=`a@x.com`，external_auth_id=`""`） | 同邮箱走 OIDC 登录 | `findOrRegister` 查 external_auth_id 未果 → `registerUser` → email 校验 → **抛异常拒绝** |
| Social 用户（Google 注册，external_auth_id=`""`） | 同 external_auth_id=`""` 的第二个 OIDC 用户尝试登录 | `findOrRegister` 查 `external_auth_id=""` → **命中第一个 Social 用户**！如果 OIDC 返回的 email 不同，不会检查，直接复用该记录 |

#### 7.3.4 空 `external_auth_id` 身份接管路径深度分析

这是一个需要严肃对待的安全边界。下面从三个维度逐层核对代码。

##### 7.3.4.1 哪些 Provider 配置错误时 external_id 会落空？各 Provider 有无前置校验？

**OIDC — 风险最高**

**文件**: `app/Access/Oidc/OidcUserDetails.php:40` + `app/Access/Oidc/OidcService.php:221-225`

```php
// OidcUserDetails::populate()
$this->externalId = $claims->getClaim($idClaim) ?? $this->externalId;
```

- `$idClaim` 取自 `OIDC_EXTERNAL_ID_CLAIM`（默认 `sub`）
- 如果管理员把 `OIDC_EXTERNAL_ID_CLAIM` 配成了一个**不存在**的 claim，`getClaim()` 返回 `null`，`??` 运算符保留原值（初始为 `null`）
- 若同时配置了 userinfo 端点且组同步开启，`isFullyPopulated()` 会触发 userinfo 补充；但如果组同步关闭或 userinfo 也没有该 claim，`externalId` 仍为 `null`
- **关键漏洞点**：`findOrRegister(string $externalId)` 参数类型声明为 `string`，传入 `null` 时 PHP 非严格模式下**隐式转换为空字符串 `''`**

```php
// OidcService::processAccessTokenCallback() — 注意没有 externalId 非空校验
if (empty($userDetails->email)) {
    throw new OidcException(trans('errors.oidc_no_email_address'));
}
if (empty($userDetails->name)) {
    $userDetails->name = $userDetails->externalId;
}
// ↓ 直接传入，无 empty 校验
$user = $this->registrationService->findOrRegister(
    $userDetails->name,
    $userDetails->email,
    $userDetails->externalId   // 可能为 null → 隐式转成 ''
);
```

**OIDC 有无前置校验？** 没有。`processAccessTokenCallback` 只校验了 `email` 和 `name`，**完全没有校验 `externalId` 是否为空**。只要 ID Token 验证通过（签名有效、过期时间正常等），就直接传入 `findOrRegister`。

---

**SAML2 — 风险较低**

**文件**: `app/Access/Saml2Service.php:256-264` + `app/Access/Saml2Service.php:360-372`

```php
protected function getExternalId(array $samlAttributes, string $defaultValue)
{
    $userNameAttr = $this->config['external_id_attribute'];
    if ($userNameAttr === null) {
        return $defaultValue;   // ← 默认为 NameID，SAML 规范下不会空
    }
    return $this->getSamlResponseAttribute($samlAttributes, $userNameAttr, $defaultValue);
}
```

- 如果不配置 `SAML2_EXTERNAL_ID_ATTRIBUTE`，external_id 始终取 NameID，不会为空
- 如果配置了但属性**不存在**，`getSamlResponseAttribute` 会回退到默认值 NameID，也不会为空
- 只有一种场景会产生空值：配置的属性**存在但值就是空字符串**（如 `['']`），`simplifyValue` 会返回 `''`
- SAML2 同样没有校验 `external_id` 非空的代码

**SAML2 有无前置校验？** 没有。`processLoginCallback` 只校验了 `email`，没有校验 `external_id`。

---

**LDAP — 风险较低**

**文件**: `app/Access/LdapService.php:121` + `app/Access/LdapService.php:144-162`

```php
// getUserDetails() 中
'uid' => $this->getUserResponseProperty($user, $idAttr, $user['dn']),
```

- 默认 `id_attribute = dn`，而 DN 在 LDAP 搜索结果中一定存在，不会为空
- 如果配置了自定义 `id_attribute` 但属性**不存在**，`getUserResponseProperty` 回退到默认值 `$user['dn']`，也不会为空
- 只有当配置的属性**存在但值就是空字符串**时，uid 才会是空字符串
- LDAP Guard 也没有校验 uid 非空的代码

**LDAP 有无前置校验？** 没有。`LdapSessionGuard::attempt()` 直接把 `uid` 传给 `retrieveByCredentials`，不做空值检查。

---

**标准注册 / Social 注册 — 始终产生空值**

**文件**: `app/Users/UserRepo.php:66`

```php
$user->external_auth_id = $data['external_auth_id'] ?? '';
```

- 标准注册（`RegisterController@postRegister`）和 Social 注册（`socialRegisterCallback`）都不传 `external_auth_id`
- 因此所有本地注册用户和 Social 用户的 `external_auth_id` 都是空字符串 `''`
- 这意味着**几乎每个 BookStack 实例都至少有一个 `external_auth_id = ''` 的用户**（初始 admin 用户也是空值，见迁移 `2016_01_11_210908_add_external_auth_to_users` 直接加列不带 default，旧数据全部变 `''`）

##### 7.3.4.2 攻击者触发接管的前置条件、路径与最高权限

**触发前提**

| 条件 | 说明 |
|------|------|
| 系统中存在 `external_auth_id = ''` 的用户 | 几乎必然满足（初始 admin、标准注册用户、Social 用户均为空） |
| 外部 Provider 的 external_id 能被控制为空字符串 | OIDC 最易触发（配错 `OIDC_EXTERNAL_ID_CLAIM`）；SAML2/LDAP 需 IdP 侧返回空值 |
| 该外部 Provider 处于激活状态 | `AUTH_METHOD` 设为对应值 |

**攻击路径（以 OIDC 为例）**

```
1. 管理员将 AUTH_METHOD 设为 oidc
2. 管理员配置 OIDC_EXTERNAL_ID_CLAIM=my_custom_claim
   （该 claim 在 IdP 侧不存在或对某些用户为空）
3. 攻击者拥有一个合法的 OIDC 账号（能正常完成认证流程）
   但该账号的 my_custom_claim 值为空字符串 / 不存在
4. 攻击者访问 /oidc/login → 跳转 IdP → 登录 → 回调 /oidc/callback
5. OidcService::processAccessTokenCallback()
   → ID Token 验签通过（签名合法，是真用户）
   → externalId = getClaim('my_custom_claim') → null
   → 无空值校验，直接传给 findOrRegister
   → PHP 类型隐式转换：null → ''
6. findOrRegister('')
   → WHERE external_auth_id = '' → first()
   → 命中 id 最小的那个空 external_auth_id 用户
   → 通常是 admin 用户（id=1）
7. 直接以该用户身份登录 → 身份接管成功
```

**可接管的最高权限**

取决于 `external_auth_id = ''` 的用户中 id 最小的那个是谁：
- **默认部署下，id=1 是初始 admin 用户**，拥有全部权限（用户管理、角色管理、系统设置等）
- 如果管理员后来修改过 admin 的 `external_auth_id`，那接管的就是最早注册的普通用户
- 接管后，组同步（如果开启）还会根据攻击者 OIDC 账号的组重新设置角色，可能扩大权限

**补充：Social 路径不会触发此问题**

Social 登录通过 `social_accounts` 表关联，不走 `findOrRegister` + `external_auth_id` 路径，因此不会命中空值接管。

##### 7.3.4.3 兜底机制核查：DB 唯一约束与应用层校验

**数据库层面 — 无兜底**

**文件**: `database/migrations/2016_01_11_210908_add_external_auth_to_users.php:15`

```php
$table->string('external_auth_id')->index();
```

- 只有普通索引（`index`），**没有唯一约束（`unique`）**
- 允许多个用户拥有完全相同的 `external_auth_id`（包括空字符串）
- 数据库层面不提供任何防止重复的保护

**应用层 — 不校验空值，也不校验唯一性**

- `findOrRegister`：只查 `external_auth_id`，找到就返回，不检查是否有多个匹配（`first()` 只取第一条）
- `registerUser`：只校验 email 唯一性，**完全不校验 `external_auth_id` 唯一性**，也不校验是否为空
- 三个 Provider 的登录回调（OIDC/SAML2/LDAP）均未在调用 `findOrRegister` 前检查 external_id 是否为空

**结论：零兜底**

数据库层和应用层都没有针对空 `external_auth_id` 的防护，也没有对 `external_auth_id` 做全局唯一性约束。一旦空值进入 `findOrRegister`，就会直接命中第一个空值用户并完成登录。

##### 7.3.4.4 接管后的横向提权路径

如果被接管的账号本身不是 admin，攻击者是否还能进一步提升权限？

**直接路径：用户管理功能**

**文件**: `app/Users/Controllers/UserController.php:35-224`

```
/user/create  → checkPermission(Permission::UsersManage)
/user/{id}/edit → checkPermission(Permission::UsersManage)
/user/{id}/update → checkPermission(Permission::UsersManage)
```

- `UserController@update` 接收 `roles` 数组，通过 `UserRepo::update → setUserRoles → $user->roles()->sync($roles)` 直接修改角色
- 如果被接管账号本身不具备 `UsersManage` 权限，以上路径全部被 `checkPermission` 拦截
- 如果被接管账号本身就是 admin（默认接管的 id=1），则可直接通过页面操作：新建带 admin 角色的用户、给自己或其他用户追加 admin 角色

**间接路径：组同步改写角色**

**文件**: `app/Access/GroupSyncService.php:73-85`

如果管理员同时开启了组同步（如 `OIDC_USER_TO_GROUPS=true` + `OIDC_REMOVE_FROM_GROUPS=true`）：

1. 攻击者用空值 OIDC 账号登录 → 被接管为某普通用户 A
2. 如果 IdP 侧给攻击者的账号配置了 admin 对应的组名
3. 每次登录都会跑 `syncUserWithFoundGroups(user, ['Admin'], true)` → `roles()->sync([adminRoleId])` → 覆盖为 admin
4. 但这需要被接管的用户 A 已经存在，且**是用户 A 的后续登录触发了组同步**，攻击者无法主动给其他用户同步

结论：**组同步不会让攻击者横向提权到其他账号**，但如果被接管的账号恰好开启了 `remove_from_groups=true` 的组同步，且攻击者 IdP 账号含有 admin 映射组，则可以在后续登录中**自我提升**为 admin。

**API Token 路径**

**文件**: `app/Users/Controllers/UserApiController.php`

同理，创建 API Token 需要 `Permission::ApiTokensManage` 权限，admin 用户可以直接创建长期有效的 API Token，供后续脚本化操作使用。

##### 7.3.4.5 Admin 关键路径上的防护核查：IP 限制 / MFA / 审批拦截

**IP 限制 — 完全没有**

**文件**: `app/Config/app.php:97` + 全站搜索 `ip_whitelist`/`allowed_ips`

```php
'ip_address_precision' => env('IP_ADDRESS_PRECISION', 4),
```

- `ip_address_precision` 只是控制**日志存储时 IP 的掩码精度**（0=全隐，4=全存），不是访问控制
- `UserController`、`RoleController`、`SettingController` 等管理控制器没有任何 IP 白名单中间件
- 结论：**没有任何 IP 层面的访问限制**，攻击者从任意 IP 均可操作管理功能

**MFA 二步验证 — 只挡登录入口，不挡后续管理操作**

**文件**: `app/Access/LoginService.php:36-60` + `app/Http/Kernel.php`

```php
// LoginService::login() 中
if ($this->awaitingEmailConfirmation($user) || $this->needsMfaVerification($user)) {
    $this->setLastLoginAttemptedForUser($user, $method, $remember);
    throw new StoppedAuthenticationException($user, $this);
}
```

```php
// Http/Kernel.php — 路由中间件
'auth' => \BookStack\Http\Middleware\Authenticate::class,
'mfa-setup' => \BookStack\Http\Middleware\AuthenticatedOrPendingMfa::class,
```

- MFA 校验**只在 `LoginService::login()` 时执行一次**
- `AuthenticatedOrPendingMfa` 中间件只放行 MFA 设置流程相关页面（mfa-setup），不保护管理页面
- 一旦通过 login 流程（MFA 也通过），后续访问 `/settings/users/{id}/update` 等敏感端点只检查 `auth()` 登录态和 `Permission::UsersManage`，**不再二次校验 MFA**
- 对身份接管场景的影响：如果被接管的 admin 账号**没有配置 MFA**，攻击者一次登录即可永久畅通；如果 admin **配置了 MFA**，攻击会被 MFA 校验拦下
- 但注意：MFA 是按**被接管的用户**来检查的，不是按攻击者。如果被接管的 admin 没有 MFA，就直接通过。

**审批拦截 — 完全没有**

代码中不存在任何需要二次审批的操作机制。用户角色变更、系统设置修改等敏感操作一经提交立即落库，没有审批流，没有审批人通知，也没有"待审批状态"。

**Demo 模式保护**

**文件**: `UserController.php:145, 202, 218`

```php
$this->preventAccessInDemoMode();
```

`update / destroy / resetMfa` 等路径有 Demo 模式保护（环境变量 `APP_ENV=demo` 时禁止），但生产环境 `APP_ENV=production` 时此保护不生效。

##### 7.3.4.6 Audit Log 字段核查：能否事后追责到攻击者？

**Activity 表结构**

**文件**: `database/migrations/2015_08_16_142133_create_activities_table.php` + 后续迁移

activities 表包含的关键字段：

| 字段 | 类型 | 内容 |
|------|------|------|
| `id` | int | 主键 |
| `type` | string | 事件类型，如 `auth_login`、`user_update` |
| `user_id` | int | 当前登录用户 ID |
| `ip` | string(45) | 来源 IP，受 `IP_ADDRESS_PRECISION` 控制精度（默认 4 = 完整 IP） |
| `detail` | text | 事件详情（`logDescriptor()` 或自定义字符串） |
| `loggable_id` | int | 关联实体 ID（可选） |
| `loggable_type` | string | 关联实体类型（可选） |
| `created_at` | datetime | 事件时间 |

**登录事件记录**

**文件**: `app/Access/LoginService.php:50`

```php
Activity::add(ActivityType::AUTH_LOGIN, "{$method}; {$user->logDescriptor()}");
```

`detail` 字段内容示例：`"oidc; (1) Admin"`

- `method`：登录方式（`standard` / `ldap` / `saml2` / `oidc` / `google` 等 Social driver 名）
- `logDescriptor()`：`({$user->id}) {$user->name}` —— **记录的是被接管用户的 ID 和 name，不是外部身份 ID**

**敏感操作事件**

| 操作 | Activity Type | 记录 detail |
|------|--------------|------------|
| 新建用户 | `USER_CREATE` | `(user_id) User Name` |
| 修改用户 | `USER_UPDATE` | `(user_id) User Name` |
| 删除用户 | `USER_DELETE` | `(user_id) User Name` |
| 重置 MFA | `USER_MFA_RESET` | `(user_id) User Name` |
| 新建角色 | `ROLE_CREATE` | `(role_id) Role Display Name` |
| 修改角色 | `ROLE_UPDATE` | `(role_id) Role Display Name` |
| 修改系统设置 | `SETTINGS_UPDATE` | 空字符串 |
| 登录 | `AUTH_LOGIN` | `方法; (userId) 用户名` |
| 注册 | `AUTH_REGISTER` | `(userId) 用户名` |

**追责能力评估**

| 追责维度 | 是否能从 log 中还原 | 说明 |
|---------|-------------------|------|
| 操作发生时间 | ✅ | `created_at` 精确记录 |
| 操作类型 | ✅ | `type` 完整枚举 |
| 操作人（本地用户 ID） | ✅ | `user_id` 记录的是当前登录态用户 ID，即**被接管的本地用户** |
| 操作人真实身份（OIDC sub / SAML NameID） | ❌ | **没有任何地方记录外部身份的原始 ID**。接管时外部 ID 为空字符串，`detail` 里只存本地用户 ID 和方法名 |
| 来源 IP | ⚠️ 部分 | 依赖 `IP_ADDRESS_PRECISION` 配置，默认 4 可完整记录；如果设为 0/1/2/3 则 IP 被掩码，无法精确溯源 |
| 被修改对象 | ✅ | `loggable_id + loggable_type` 关联实体，或 `detail` 中的 logDescriptor |
| 操作前后的变化（Diff） | ❌ | `USER_UPDATE` 只记录一条 "update" 事件，**不记录改了哪些字段、原值是什么**（比如角色从 Viewer 改成 Admin 没有日志痕迹） |
| 具体改了哪个用户的角色 | ❌ | `USER_UPDATE` 的 detail 是被修改用户的 logDescriptor，但**不区分改了 name / email / roles / external_auth_id 中的哪一项** |

**最关键的追责缺陷**：

1. **外部身份 ID 完全不入库**——即使是正常登录，外部 `sub` / NameID / DN 也只用于 `findOrRegister` 的查找，不会写入任何日志。身份接管时 external_id 是空字符串，日志里根本区分不了"正常 admin 登录"和"攻击者用空 external_id 登录成 admin"。
2. **USER_UPDATE 无 Diff**——攻击者把自己的角色从普通用户改为 admin，日志里只有一条 `user_update`，不记录 roles 变化；事后审计需要额外对比 `activities.created_at` 时间点前后的 `role_user` 表快照才能发现。
3. **无法区分攻击者和真实用户**——攻击者使用被接管账号操作时，`user_id` 就是被接管用户的 ID，IP 也可能重合（如同一企业网络出口），没有独立的身份指纹。

#### 7.3.5 冲突结论

- **Social 和外部主 Provider 不会自动合并身份**，靠 email 校验互相拦住注册；只有用户先通过一种方式登录后，再在个人设置里手动绑定 Social 账号才能建立关联。
- **`social_accounts` 表和 `users.external_auth_id` 完全互不感知**，没有任何代码把两边做交叉同步。
- **空 `external_auth_id` 是高危值**，可能导致不同 Provider 的用户被错误合并。

### 7.4 冲突总览

| 冲突类型 | 是否会自动合并 | 实际行为 | 相关代码 |
|---------|---------------|---------|---------|
| 不同 Provider + 同邮箱 + 不同 external_id | ❌ | email 校验抛异常，拒绝创建 | `RegistrationService::registerUser:87-90` |
| 不同 Provider + 同 external_id | ✅ | 直接复用已有 `users` 记录 | `RegistrationService::findOrRegister:55-57` |
| 不同 Provider 给同一用户不同角色 | — | 取**最后一次登录**的 Provider 的组同步结果，是否覆盖由该 Provider 的 `remove_from_groups` 决定 | `GroupSyncService::syncUserWithFoundGroups:79-84` |
| Social 与外部主 Provider 同邮箱 | ❌ | Social 注册被 email 校验拦截；外部 Provider 注册也被 email 校验拦截 | `SocialAuthService::handleRegistrationCallback:63-67` |
| Social 已登录 + 绑定外部 Provider | ✅ | 把 Social 账号挂到当前已登录用户上，不新建 | `SocialAuthService::handleLoginCallback:111-117` |
| 空 external_auth_id 身份接管 | ⚠️ 高危 | OIDC 配错 external_id_claim 时，externalId 为 null → PHP 隐式转空字符串 → `findOrRegister('')` 命中第一个空值用户（通常是 admin）；DB 无 unique 约束，应用层无空值校验，零兜底 | `RegistrationService::findOrRegister:55-57` + `OidcService:221-225` |
