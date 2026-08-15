# GitHub Actions 自动签名与 OTA 配置指南

在 Fork 仓库中用自己的 Apple 签名资产构建 Feather，发布 IPA，并提供一个不会变的 OTA 安装入口。

稳定入口（签名仍会过期，入口 URL 不变）：

```text
https://<owner>.github.io/<repository>/ios/latest/install.html
```

定时任务每 30 分钟查询上游 `claration/Feather` 的 `main`。同一个 source SHA 不会重复消耗 macOS runner。

## 1. 需要准备什么

- 付费 Apple Developer 账号
- 带私钥的 Distribution 证书（导出为带密码的 `.p12`）
- 包含目标 iPhone UDID 的 Ad Hoc provisioning profile
- profile 的 App ID 必须和构建 Bundle ID 一致（默认 `thewonderofyou.Feather`）

## 2. 配置仓库

1. Fork 本仓库。
2. **Settings → Actions → General**：Workflow permissions 选 **Read and write permissions**。
3. **Settings → Pages**：Build and deployment Source 选 **GitHub Actions**。

## 3. 写入 Secrets 和 Variables

不要把 `.p12`、密码或 profile 提交到 Git。

### 推荐：本地脚本生成后上传

在执行脚本后，可选择手动添加Secrets/Variables  
也可以使用自动生成的依赖gh的上传脚本

关于更新链接，默认使用 `--ota-base-url 'https://<owner>.github.io/<repo>'`  
也可以手动指定域名

```bash
./Resources/Scripts/generate.github.action.inputs.sh \
  --p12 /path/to/certificate.p12 \
  --p12-password 'p12-password' \
  --mobileprovision /path/to/profile.mobileprovision
```

然后：

```bash
export GITHUB_REPOSITORY='<owner>/<fork-repository>'
/path/to/generated/apply-with-gh.sh
```

生成目录含敏感信息，上传后立刻删除。

### 手动填写

**Settings → Secrets and variables → Actions**

#### Secrets

| 名称 | 必需 | 内容 |
| --- | --- | --- |
| `IOS_CERT_P12_BASE64` | 是 | `.p12` 的 Base64（`base64 -i certificate.p12 \| pbcopy`） |
| `IOS_CERT_PASSWORD` | 是 | 导出 `.p12` 时的密码 |
| `IOS_PROVISIONING_PROFILE_BASE64` | 是 | `.mobileprovision` 的 Base64 |
| `IOS_KEYCHAIN_PASSWORD` | 否 | runner 临时 keychain 密码 |
| `IOS_TEAM_ID` | 否 | 不填则从 profile 读取 |

#### Variables

| 名称 | 必需 | 默认/示例 | 内容 |
| --- | --- | --- | --- |
| `IOS_BUNDLE_ID` | 建议 | `thewonderofyou.Feather` | 必须与 profile 的 App ID 匹配 |
| `IOS_EXPORT_METHOD` | 建议 | `ad-hoc` | OTA 用 `ad-hoc`；开发测试可用 `development` |
| `IOS_SIGNING_IDENTITY` | 否 | `Apple Distribution` | 覆盖签名身份名称 |
| `IOS_OTA_BASE_URL` | 否 | `https://owner.github.io/repo` | 自定义 Pages 域名 |

`app-store` IPA 不能走这套 Ad Hoc OTA。`enterprise` 仅限企业账号。

## 4. 运行

1. 打开 Actions → **Upstream Signed iOS Build** → **Run workflow**。
2. 默认同步上游 `claration/Feather` 的 `main`。也可以改成构建本 Fork。
3. 首次建议先手动跑通，再依赖 30 分钟定时任务。

工作流不会把上游代码 merge 进 Fork 分支。Fork 只保存工作流和配置；源码按完整 SHA checkout。

## 5. 安装

构建成功后，用 iPhone Safari 打开：

```text
https://<owner>.github.io/<repo>/ios/latest/install.html
```

点击安装。如系统提示，到 **设置 → 通用 → VPN 与设备管理** 信任对应开发者。

| 地址 | 作用 |
| --- | --- |
| `releases/download/<tag>/Feather-<sha>.ipa` | 某一次构建的 IPA |
| `ios/<sha>/install.html` | 指定版本安装页 |
| `ios/latest/install.html` | 始终指向最新构建 |

## 6. 常见问题

| 现象 | 优先检查 |
| --- | --- |
| `No signing identity` | `.p12` 是否含私钥、密码是否正确、`IOS_SIGNING_IDENTITY` 是否匹配 |
| profile 与 Bundle ID 不匹配 | `IOS_BUNDLE_ID` 必须等于 profile 的 App ID |
| IPA 能下但装不上 | profile 是否 Ad Hoc、UDID 是否登记、证书是否有效 |
| `latest` 页面 404 | Pages Source 是否为 GitHub Actions，以及 `pages: write` 权限 |
| 没有新构建 | 该 SHA 的 Release 是否已存在；看 Actions Summary 的 Source Detection |
| 上游代码变了但工作流行为没变 | 预期行为：跟随的是上游源码，不是上游的 workflow 文件 |

## 7. 安全注意

- 不要把 `.p12`、密码写进仓库、Issue、Release 或日志。
- 上游 `main` 会用你的证书签名。高安全场景应只构建审核过的 tag，或给签名 job 加 Environment 审批。
- 不要对不可信 PR 跑带签名 Secrets 的 job。
