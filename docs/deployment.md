# 部署指南（详细版）

[README.md](../README.md) 的六步清单是操作顺序；这份文档补上每一步实际执行时会遇到的分支决策、命令示例、以及一次真实部署踩过的坑（对应的排错条目见 [troubleshooting.md](troubleshooting.md)）。按 README 的步骤编号组织，跳步骤看就行，不用整篇通读。

## 前置：TLS 证书——当前模板只支持 DNS 验证签发新证书

`template.yaml` 的证书来源写死是 `AWS::CertificateManager::Certificate` + `ValidationMethod: DNS`（模板第 127 行附近），**没有导入已有证书的参数**——如果公司已经有一张覆盖目标域名的证书（比如统一采购的通配符证书），当前版本没有跳过 DNS 验证、直接复用现成证书的开关，只能：

1. 走模板默认的 DNS 验证签发一张新证书（部署过程中在 ACM 控制台完成域名验证记录），或
2. 自己改模板：把 `Certificate` 这个资源块删掉，新增一个 `ImportedCertificateArn` 参数，把 `HttpsListener.Properties.Certificates[0].CertificateArn` 从 `!Ref Certificate` 改成 `!Ref ImportedCertificateArn`，部署前先 `aws acm import-certificate` 把已有证书导入 ACM 拿到 ARN。这是一次真实改造验证过可行的方案，但**没有反向合并回这个模板**，用之前自己动手改，不要照抄下面这段当成模板已有的能力用：

```bash
# 证书文件如果是多段 PEM（叶子证书 + 中间证书链拼在一起），先拆开：
# 第 1 段是叶子证书，其余段是证书链
aws acm import-certificate \
  --certificate fileb://leaf.pem \
  --private-key fileb://your-domain.key \
  --certificate-chain fileb://chain.pem
```

判断私钥和证书是否匹配可以用 `openssl x509 -modulus -in cert.pem | openssl md5` 和 `openssl rsa -modulus -in key.pem | openssl md5` 对比输出是否一致。

## 步骤 1：部署 Keycloak 基础设施

`aws cloudformation deploy` 或控制台都行。这一步部署完之后，Keycloak 本身能跑起来了，但 realm/client/组这些业务配置还没有——那是步骤 3 Bootstrap 做的事。

**已知问题（已在 `template.yaml` 修复，如果你的 fork 版本比较旧，检查一下）**：
- EC2 `UserData` 里的 `git clone` 地址应固定到受维护仓库的 release tag 或 commit，不能拉个人 fork 的浮动默认分支；否则自举代码不可复现，也可能因为仓库不可访问导致实例启动失败（`cloud-init status` 显示 `error`，Docker 容器起不来）。
- `ImageId` 如果还是内联的 `!Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/...}}'` 写法（不是走 `AmiId` 参数），以后任何一次 `cloudformation deploy`（哪怕跟 EC2 毫无关系，比如只改了 Lambda 代码）都有概率因为 AWS 发布了新 AMI 而把这台跑着 Keycloak+Postgres 的实例整个替换重建——**这意味着数据丢失**。这是真实发生过的事故，不是理论风险。

如果你是从零部署这个模板（不是接手别人部署好的实例），这两个坑上游最新版应该已经修了，正常走就行。

## 步骤 2：配置 Keycloak 管理员

用 `KeycloakAdminUsername` 和 CloudFormation 自动生成到 Secrets Manager 的临时密码登录，创建正式管理员，然后处理掉临时账号（改密码或禁用）。

可以用下面的命令取回初始密码：

```bash
aws secretsmanager get-secret-value \
  --secret-id <KeycloakAdminSecret ARN> \
  --query SecretString --output text
```

不要把这个临时密码写进工单、聊天记录或本地长期配置文件。

## 步骤 3：运行 Bootstrap

`bootstrap/main.py` 用 `python-keycloak` 库直接调 Keycloak Admin REST API，**不调用任何 AWS API**，所以哪怕 Amazon Quick 账户还没开通也不影响这一步执行。

跑之前把 `.env` 填完整（模板见 [`bootstrap/.env.example`](../bootstrap/.env.example)），然后：

```bash
cd /opt/keycloak/bootstrap   # 或者你本地 clone 出来的对应目录
chmod +x bootstrap.sh
docker compose run --rm bootstrap
```

（如果当前用户不在 `docker` 组，前面加 `sudo`。）

跑完应该看到一串 `[OK]`：Realm、Quick 管理员用户、SAML Client、6 个 Client Role（三档 Pro + 三档标准）、4 个 Mapper、6 个 Group、OIDC Client。重复跑是幂等的——已存在的资源会显示 `[SKIP]`，不会报错中断。

## 步骤 4：创建 SAML Identity Provider

拉 SAML Metadata 时有个容易踩的坑：**不要直接在 EC2 实例内网 `curl http://localhost:8080/...` 去拿**——本机直连没有走 ALB 的转发头（`X-Forwarded-*`），Keycloak 解析出来的 `entityID` 会带上内部端口/协议（比如 `http://<domain>:8080` 而不是 `https://<domain>`），拿这份 metadata 去注册 SAML Provider，真实用户登录时会被引导去一个打不通的地址。

正确做法是走完整的公网路径（ALB → HTTPS → Keycloak）拿 metadata：

```bash
curl -s "https://<KEYCLOAK_DOMAIN>/realms/quick/protocol/saml/descriptor" -o saml-metadata.xml
# entityID 应该长这样：https://<KEYCLOAK_DOMAIN>/realms/quick
grep entityID saml-metadata.xml

aws iam create-saml-provider --name keycloak --saml-metadata-document file://saml-metadata.xml
```

**Provider 名称必须是 `keycloak`**（不能改成别的）——模板里 IAM Role 的信任策略写死了这个名字，改了名字信任关系就断了。

如果 DNS 还没生效、暂时没法走公网域名，可以用 `curl --resolve <domain>:443:<ALB的IP>` 强制把域名解析到 ALB 走一遍完整的 HTTPS 路径，同样能拿到正确的 entityID，等 DNS 生效了没有必须重新做这一步（前提是 metadata 内容本身没变）。

## 步骤 5：Quick 控制台 SSO 设置 + 步骤 6：Desktop 扩展访问

这两步都要**用 AWS IAM 原生管理员账号**登录（`?enable-sso=0` 那条入口），不能用刚建好的 SSO 联邦账号——SSO 联邦身份拿到的 IAM Role 权限是收窄过的，做不了账户级配置操作（详见 [troubleshooting.md](troubleshooting.md) 第 7 条）。

步骤 6 的 "Extension access" 和 "Extensions" 是两个独立入口：先在 Manage account → Permissions → Extension access 填参数保存，再去 Quick 主界面的 **Connections → Extensions**（不在 Manage account 侧边栏里）基于刚才的配置创建出实际的 extension——漏了第二步会报"尚未为此账户配置 Quick Desktop 的企业登录"（见 [troubleshooting.md](troubleshooting.md) 第 5 条）。

配完之后建议先用一个专门的测试账号（不是生产要用的真实用户）走一遍完整链路验证：Web 端登录 + Desktop 端登录都成功，再开始批量给真实用户建号。验证完把这个测试账号删掉，不要留着当长期账号用。

## 批量建号（步骤 3 之后，随时可以做）

两种方式，按适用场景选：

- **手动 `kcadm.sh` 逐个建**：适合个位数账号，见 Keycloak 官方 `kcadm.sh` 文档，注意事项见下方。
- **自己写批量脚本**：适合几十上百人规模、需要离线生成密码表格发放的场景，用 Admin REST API 的 `POST /admin/realms/quick/partialImport` 一次提交所有待建用户，比对每个人循环调用建号/加组/设密码三个接口快得多。

不管走哪种方式，以下几条是通用的：

- **`requiredActions` 要显式清空成 `[]`**：不清的话新用户可能残留 `VERIFY_PROFILE` 等默认必做项，首次登录会被要求补资料（见 [troubleshooting.md](troubleshooting.md) 第 8 条，其实不影响后续使用，只是多一步）。
- **批量建号前先核对邮箱唯一性**：`aws quicksight list-users` 看清单里的邮箱有没有跟已有 Quick 用户（尤其是原生 IAM 账号、Bootstrap 建的初始管理员）重复，避免撞上邮箱同步冲突（见 [troubleshooting.md](troubleshooting.md) 第 3 条）。
- **如果自己生成随机密码写进 Excel/CSV 表格**：密码首字符如果是 `= + - @` 之一，会被 Excel/WPS 当成公式解析，密码在表格里会"消失"或显示成公式报错，不是文本。生成密码时排除这几个首字符，如果用 `openpyxl` 写表格，写入后再检查一遍单元格 `data_type` 是不是被误判成了 `'f'`（公式），是的话强制拉回字符串类型。
- **考虑显式声明 `realmRoles: ["default-roles-quick"]`**：正常情况下 Keycloak 建号会自动赋这个默认角色，但走某些接口路径（尤其是 `partialImport`）偶发漏赋，导致该用户 Desktop 客户端登录报 "Offline tokens not allowed"（网页端不受影响，见 [troubleshooting.md](troubleshooting.md) 第 10 条）。显式声明不增加接口调用次数，是免费的双保险。

## 建议：加固密码策略

`quick` realm 刚建出来时密码策略是空的（无长度/复杂度要求），暴力破解防护也是关的。生产环境建议至少加上：

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "https://<KEYCLOAK_DOMAIN>/admin/realms/quick" \
  -d '{
    "passwordPolicy": "length(10) and upperCase(1) and lowerCase(1) and digits(1) and specialChars(1) and notUsername and notEmail",
    "bruteForceProtected": true,
    "failureFactor": 5
  }'
```

改之前最好先 `GET /admin/realms/quick` 把现有配置读出来，只改要改的字段再 PUT 回去（realm 配置是整体替换，漏带字段等于把没提到的配置清空）。

注意 `resetPasswordAllowed` 默认是 `false`——登录页不会有"忘记密码"邮件找回入口，真忘了密码的用户只能找管理员在后台重置，这是设计如此，不是 bug，提前告诉用户群体这一点能省很多支持工单。

## 部署完成后要盯的两件事

1. **`ROLE_SUBSCRIPTION_MAP`（QuickSubscriptionAssignFunction）**：这个 Lambda 负责用户首次登录时自动分配 QuickSight 订阅角色。如果你后续给某个角色对应的 IAM Role 加了自注册权限之外的逻辑，或者怀疑某类用户订阅没有自动生效，先去 CloudWatch 查这个函数的 Invocations 指标，确认 EventBridge 规则真的在触发它（而不是规则配置错了 `detail-type` 导致规则形同虚设，静默失效）。
2. **`data/postgres`（docker-compose 部署方式下）** 或对应的 EBS 卷：这是 Keycloak 的持久化存储，任何会导致 EC2 实例被替换/删除的操作（`cloudformation deploy`、`docker compose down -v`）都要先确认不会波及这个卷/数据库，尤其是已经有真实用户数据之后。
