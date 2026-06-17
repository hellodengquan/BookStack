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

#### 7.3.4 一个特别危险的边角案例

**文件**: `app/Access/RegistrationService.php:55-57`

```php
$user = User::query()
    ->where('external_auth_id', '=', $externalId)
    ->first();
```

如果：
1. 某个用户通过 Social 注册，`users.external_auth_id` 被写入空字符串 `""`
2. 另一个外部 Provider（比如 LDAP）配置错误，导致某些用户的 `uid` 也返回空字符串

那么两个完全无关的用户会因为 `WHERE external_auth_id = ''` 命中同一条记录，**后登录的人会直接以第一个注册用户的身份进入系统**。这是一个潜在的身份接管风险，配置时需要确保所有外部 Provider 的 external_id 字段都不能为空。

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
| 空 external_auth_id 跨 Provider | ⚠️ | 所有产生空 external_id 的用户会被错误合并为同一个账号 | `RegistrationService::findOrRegister:55-57` |
