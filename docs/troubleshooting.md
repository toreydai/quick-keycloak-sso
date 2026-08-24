# 登录排错手册

给管理员/支持人员用的排错参考，覆盖的是这套 Keycloak + Amazon Quick SSO 架构里**结构性会出现**的报错——不是某一次部署偶发的问题，是只要这套 SAML/OIDC 联邦身份链路搭起来，任何一次真实部署都可能踩到的坑。来源：一次真实生产部署，从搭建到给几十个用户批量开号，全部错误信息都实际复现过、定位到根因、解决过，见 [deployment.md](deployment.md) 的对应记录。

## 目录

1. Your request included an invalid SAML response.
2. 我们无法验证您的登录凭证。请重试。
3. Failed to authorize user: we found more than 1 user for your email.
4. 注销不了 / 一直被自动跳转
5. 尚未为此账户配置 Quick Desktop 的企业登录
6. User info request failed (HTTP 401)
7. not authorized to perform: quicksight:xxx
8. 首次登录卡在补资料页面
9. Desktop 客户端登录卡住不动
10. "Token exchange failed: Offline tokens not allowed for the user or client"

---

## 1. "Your request included an invalid SAML response."

**出现场景**：浏览器跳转到 `signin.aws.amazon.com/saml` 时报错。

**常见原因，按可能性排序排查**：

| 原因 | 怎么确认 | 解决办法 |
|---|---|---|
| 用户不在任何 `quick-*` 组里 | Keycloak 管理台 → Users → 该用户 → Groups，看是否为空 | 把用户加入对应角色的组（`quick-admin`/`quick-admin-pro`/`quick-author`/...） |
| 浏览器残留了别的账号的登录会话 | 换个账号测试却还是报错/或者显示的是别人的身份 | 换无痕窗口重试，或清掉 Keycloak 域名和 `.amazon.com` 两个域名的 cookie |
| 断言已过期（间隔太久才提交） | 一般不会遇到，除非中间卡了很久 | 重新走一遍登录流程 |

**为什么用户名不在组里就报这个错**：Keycloak 的 `saml-role-list-mapper` 只会把用户所在组绑定的角色塞进 SAML 断言的 `Role` 属性，没有组 = 断言里没有 Role 属性 = AWS 判定断言无效，报错信息不会明说"缺 Role"，只会笼统报"invalid"。

---

## 2. "我们无法验证您的登录凭证。请重试。"

**出现场景**：网址是 `signin.aws` 开头（不是 Keycloak 域名），也就是 **AWS 原生登录页**，不是 Keycloak 登录页。

**根本原因**：用了不该在这个页面用的账号——比如拿 Keycloak 账号的用户名密码，去填 AWS 原生的 IAM 登录表单。这两套账号体系完全独立，AWS 原生登录页根本不认识 Keycloak 账号。

**怎么排查**：先看当时访问的完整网址是不是带了 `?enable-sso=0`——这个参数会跳过 SSO，直接给出 AWS 原生登录页。

**解决办法**：
- 如果本来就想用 SSO（Keycloak 账号）登录：改用不带 `?enable-sso=0` 的地址 `https://quicksight.aws.amazon.com/sn/account/<QUICK_ACCOUNT_NAME>/start`
- 如果本来就想用 AWS IAM 账号登录：确认填的是 IAM 用户名密码，不是 Keycloak 密码

---

## 3. "Failed to authorize user: we found more than 1 user for your email."

**出现场景**：登录到最后一步，Quick 报邮箱冲突。

**根本原因**：Amazon Quick 开了"邮箱同步"功能（README 步骤 5 的"联合身份用户的电子邮件同步"）后，会强制要求每个邮箱只能对应一个 Quick 用户。如果同一个邮箱同时被两个 Quick 用户占用（比如一个原生 IAM 用户 + 一个新建的 SSO 联邦身份），登录会被直接拒绝。

**怎么排查**（管理员操作）：

```bash
aws quicksight list-users --aws-account-id <账户ID> --namespace default \
  --query 'UserList[].[UserName,Email,Role,IdentityType]' --output table
```

看有没有两行 Email 一样。

**解决办法**：
1. 确定哪个用户该保留这个邮箱（通常保留 SSO 联邦身份，删掉重复的那个）
2. `aws quicksight delete-user --user-name "<多余的那个用户名>"`
3. 如果冲突方是某个原生 IAM 账号（比如共用的初始管理员账号），可以把它的 Email 改成占位邮箱腾出来：`aws quicksight update-user --user-name <用户名> --email <新邮箱> --role <角色>`

**预防**：批量建号前先用上面的 `list-users` 命令核对清单里的邮箱有没有跟已有用户重复，尤其是 Bootstrap 建号时用过的、以及部署时创建的原生 IAM 账号——这两类账号最容易跟后续 SSO 用户撞邮箱。

---

## 4. 注销不了 / 一直被自动跳转

**出现场景**：点注销后立刻又被带回登录页，或者反复刷新都停不下来。

**根本原因**：Quick 开了"服务提供商发起的 SSO"（README 步骤 5），只要访问默认地址，Quick 就会自动重定向去 Keycloak 认证。如果这时候认证失败（比如账号有问题），注销→自动重定向→再次失败会形成循环。

**解决办法**：用带 `?enable-sso=0` 的地址跳过自动重定向，走到正常的登录/注销页面：

```
https://quicksight.aws.amazon.com/sn/account/<QUICK_ACCOUNT_NAME>/start?enable-sso=0
```

---

## 5. "尚未为此账户配置 Quick Desktop 的企业登录。请联系您的 IT 团队以获得更多支持。"

**出现场景**：Quick Desktop 客户端点 "Continue with SSO"。

**根本原因**：README 步骤 6 里"添加扩展访问"和"添加扩展"是**两个独立的步骤**，很容易漏第二步——只配了 Extension access 参数，没有基于它在 Extensions 页面创建出实际的 extension。

**解决办法**（管理员操作，需要 AWS IAM 原生管理员账号，见下条为什么不能用 SSO 账号）：
1. Manage account → Permissions → Extension access，确认已经填了 Client ID + 四个 OIDC 端点并保存
2. 找 **Connections → Extensions**（注意：不在 Manage account 的侧边栏里，在 Quick 主界面自己的导航菜单里，容易找不到）
3. 基于第 1 步的 Extension access 创建 extension

---

## 6. "User info request failed (HTTP 401)"

**出现场景**：Extension access 和 Extension 都配置完了，Desktop 登录还是失败。

**根本原因**：Extension access 里填的 OIDC 端点 URL 有误，最常见是 **Issuer URL** 拼错或者结尾多/少了斜杠。

**解决办法**：逐字核对四个端点（直接从 README 步骤 6 复制粘贴，不要手打）：

```
Issuer URL:                https://<KEYCLOAK_DOMAIN>/realms/quick
Authorization Endpoint:    https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/openid-connect/auth
Token Endpoint:             https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/openid-connect/token
JWKS URI:                   https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/openid-connect/certs
```

Issuer URL 要求跟 Keycloak 实际签发 token 里的 `iss` 字段完全一致，多一个字符都不行。

---

## 7. "not authorized to perform: quicksight:xxx"

**出现场景**：用 SSO 账号登录后，想去 Manage account 改 SSO 设置、Extension access 等账户级配置，报 IAM 权限错误。

**根本原因**：SSO 联邦登录拿到的 IAM Role（`QuickAdminRole`/`QuickAdminProRole` 等）权限是收窄过的，只够自己注册成 Quick 应用内的 Admin 角色，不含管理账户级设置需要的 IAM 权限。**这是模板刻意的权限设计，不是 bug**——就算在 Quick 应用里显示是"Admin"，也不代表有 AWS 账户管理权限。

**解决办法**：账户级配置（SSO 设置、Extension access、其他账户管理操作）必须用 **AWS IAM 原生管理员账号**登录（`?enable-sso=0` 那条路），SSO 账号做不了。

---

## 8. 首次登录卡在补资料页面

**出现场景**：用临时密码登录、改完新密码后，又被要求填姓名等资料才能继续。

**根本原因**：建号时没有提供 firstName/lastName，触发了 Keycloak 的默认必做项（required action）。

**解决办法**：跟着页面提示填完姓名信息即可继续，不影响后续正常使用。管理员建号时如果提前把姓名字段填好、并显式清空 `requiredActions`，可以避免用户遇到这一步。

---

## 9. Desktop 客户端登录卡住不动

**出现场景**：点 "Continue with SSO" 后一直转圈，没反应也不报错。

**根本原因**：Desktop 的 SSO 依赖浏览器里**当前活跃**的 Amazon Quick 网页会话来判断用哪个身份。如果浏览器里没有登录过 Quick 网页版，或者会话已经过期，Desktop 拿不到身份信息。

**解决办法**：先用浏览器登录一次网页版（`https://quicksight.aws.amazon.com/sn/account/<QUICK_ACCOUNT_NAME>/start`），保持该浏览器窗口打开、不要退出登录，再回 Desktop 客户端重试。

---

## 10. "Token exchange failed: Offline tokens not allowed for the user or client"

**出现场景**：Desktop 客户端点 "Continue with SSO" 报这个错，但**同一账号网页版能正常登录**。

**根本原因**：该用户在 `quick` realm 里缺少 `default-roles-quick` 这个默认复合角色（里面包含 `offline_access`）。Desktop 走 OIDC 登录会请求 `offline_access` scope，用户没有这个角色 Keycloak 直接拒绝签发 token；网页版走 SAML，不涉及 `offline_access`，所以网页端不受影响——这是判断"是不是这个问题"的关键分界点：**网页正常 + Desktop 报这条错 = 基本可以确诊**。

`bootstrap/main.py` 用 `create_user()` API 直接建号，正常情况下 Keycloak 会自动给新用户赋 `default-roles-quick`；但如果用户是通过别的路径（比如 Admin REST API 的 `partialImport` 批量导入接口）建的，某些 Keycloak 版本/场景下这一步偶发漏赋，实际发生过。

**怎么排查**（管理员操作，需要 master realm 管理员 token）：

```bash
# 查某个用户的 realm role mappings，看有没有 default-roles-quick
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://<KEYCLOAK_DOMAIN>/admin/realms/quick/users/<用户ID>/role-mappings/realm"
```

如果返回是空数组 `[]`（没有任何 realm role，包括 `default-roles-quick`），就是这个问题。

**解决办法**：

```bash
# id 是 quick realm 里 default-roles-quick 角色的固定 ID，先查一遍：
# GET /admin/realms/quick/roles/default-roles-quick
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "https://<KEYCLOAK_DOMAIN>/admin/realms/quick/users/<用户ID>/role-mappings/realm" \
  -d '[{"id":"<default-roles-quick 的 id>","name":"default-roles-quick"}]'
```

**预防**：如果你自己写批量建号脚本（不是用 `bootstrap/main.py`），建号 payload 里显式声明 `"realmRoles": ["default-roles-quick"]`，不要依赖 Keycloak 自动赋——多一步声明的开销可以忽略，能双保险防住这个偶发问题。

---

## 通用排查思路

遇到没收录的新报错，按这个顺序缩小范围：

1. **确认走的是哪个入口**：Keycloak 登录页、AWS 原生登录页（`signin.aws`）、还是 Quick 应用本身（`quicksight.aws.amazon.com`）——错误往往是"入口用错了"
2. **确认用的是哪个账号**：SSO 联邦身份 vs AWS IAM 原生账号，两者权限和用途完全不同，不能混用
3. **换无痕窗口重试一次**：排除浏览器残留会话的干扰
4. **管理员用 `aws quicksight list-users` 核对状态**：邮箱是否唯一、用户是否已建、Role 是否符合预期
