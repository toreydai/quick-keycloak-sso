# 配置项清单（填空模板）

部署过程中会积累一批分散在 CloudFormation 参数、Keycloak 配置、AWS 控制台表单里的值，互相之间有依赖关系（改一处要同步改另一处）。这份清单帮你在部署时把这些值记全，本身不含任何真实值——**不要把这份文件填完真实值之后直接提交到 git 仓库**，密码/密钥类的值应该走密码管理工具，其余的值如果要留档建议新建一个不进版本库的本地副本。

## 登录入口（部署完确认这几个入口都各自记清楚了）

| 用途 | 入口 | 账号体系 |
|---|---|---|
| Quick 网页版日常登录（SSO 用户） | `https://quicksight.aws.amazon.com/sn/account/<QUICK_ACCOUNT_NAME>/start` | Keycloak `quick` realm 用户 |
| Quick Desktop 客户端登录 | 桌面 App 内 "Continue with SSO" | 同上，前提是该浏览器已登录过网页版 |
| AWS 账户级管理（SSO 设置、Extension access） | `.../start?enable-sso=0` | **AWS IAM 原生账号**，不是 SSO 账号 |
| Keycloak 管理台 | `https://<KEYCLOAK_DOMAIN>` | Keycloak `master` realm 管理员 |

## CloudFormation

| 参数 | 你的值 | 备注 |
|---|---|---|
| VpcId | | |
| PublicSubnet1 / PublicSubnet2 | | 需要 2 个不同 AZ |
| InstanceType | | 模板默认 `t4g.medium` |
| AmiId | | 见 [deployment.md](deployment.md) AMI 那段，不建议改回动态解析写法 |
| KeyPairName | | 需要能 SSH 排障时才必填 |
| KeycloakDomain | | |
| ImportedCertificateArn（仅当你按 [deployment.md](deployment.md) 前置章节自己改过模板、加了这个参数） | | 模板默认不带这个参数，见 deployment.md |
| 栈名 / Stack Name | | |

## Keycloak

| 项 | 你的值 |
|---|---|
| Realm | `quick`（默认） |
| 正式管理员用户名 | |
| Bootstrap `.env` 里的 `KEYCLOAK_QUICK_ADMIN_USERNAME`/`EMAIL` | |
| `QUICK_ACCOUNT_NAME` | |
| `AWS_ACCOUNT_ID` | |
| `MAPPER_SESSION_DURATION_VALUE`（秒） | 默认 43200（12 小时） |
| passwordPolicy（是否已加固，见 deployment.md） | |
| bruteForceProtected | |

## IAM SAML Provider

| 项 | 值 |
|---|---|
| Name | **必须是 `keycloak`**，不能改 |
| ARN | `arn:aws:iam::<账户ID>:saml-provider/keycloak` |
| Metadata 来源 | `https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/saml/descriptor`（走公网路径拿，不要走实例内网直连，见 deployment.md） |

## Amazon Quick 账户

| 项 | 你的值 |
|---|---|
| Account Name | |
| Edition | |
| Authentication Method | |
| 原生 IAM 管理员账号的 Quick 内 Email | 如果要跟某个 SSO 用户共用邮箱，记得改成占位邮箱，见 troubleshooting.md #3 |

## Quick SSO 设置（控制台填的四个字段之一，见 README 步骤 5）

| 字段 | 值 |
|---|---|
| IdP URL | `https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/saml/clients/aws` |
| IdP 重定向 URL 参数 | `RelayState` |

## Quick Desktop 扩展访问（README 步骤 6）

| 字段 | 值 |
|---|---|
| Issuer URL | `https://<KEYCLOAK_DOMAIN>/realms/quick` |
| Authorization Endpoint | `https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/openid-connect/auth` |
| Token Endpoint | `https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/openid-connect/token` |
| JWKS URI | `https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/openid-connect/certs` |
| Client ID | `amazon-quick-desktop`（默认） |

## DNS

| 项 | 值 |
|---|---|
| CNAME / A 记录 | `<KEYCLOAK_DOMAIN>` → ALB DNS Name |
| DNS 托管方 | |
