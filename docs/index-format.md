# 索引格式 `index.json`

CI 从 `plugins/*.json` 和各插件的 Release 生成 `index.json`,发布到 GitHub Pages:

```
https://mygo-music.github.io/plugins/index.json
https://mygo-music.github.io/plugins/icons/<id>-<sha8>.png
```

宿主只读这一个文件。应用启动时和每 6 小时刷新一次(用 `ETag` / `If-None-Match`);离线时使用上次的副本。

## 结构

```json
{
  "schema": 1,
  "generated_at": "2026-11-05T08:00:00Z",
  "plugins": [
    {
      "id": "example",
      "name": "示例来源",
      "description": "第三方示例来源:登录、推荐、搜索、歌单、电台。",
      "i18n": { "en": { "name": "Example Source", "description": "..." } },
      "icon": "icons/example-1a2b3c4d.png",
      "repo": "example-org/plugin-example",
      "tier": "community",
      "license": "Apache-2.0",
      "authors": [{ "name": "Example", "github": "example-org" }],
      "homepage": "https://github.com/example-org/plugin-example",
      "tags": ["music", "podcast"],
      "capabilities": ["search", "stream", "..."],
      "features": ["like", "..."],
      "latest": "1.2.0",
      "versions": [
        {
          "version": "1.2.0",
          "api": 1,
          "min_host": "0.2.0",
          "url": "https://github.com/example-org/plugin-example/releases/download/v1.2.0/example-1.2.0.mgp",
          "sha256": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
          "size": 4194304,
          "permissions": {
            "network": { "hosts": ["api.example.com"] },
            "media": { "hosts": ["*.cdn.example.com"] }
          },
          "published_at": "2026-11-04T12:00:00Z"
        }
      ]
    }
  ],
  "revoked": [
    { "id": "some-plugin", "versions": ["1.0.2"], "reason": "收到权利人通知", "at": "2026-11-05T00:00:00Z" }
  ]
}
```

## 字段

| 字段 | 说明 |
|---|---|
| `schema` | 索引格式版本。宿主不认识的版本 → 忽略索引,保留旧副本 |
| `plugins[].tier` | `official`(仓库属 `mygo-music` 组织)或 `community`。自动判定 |
| `plugins[].icon` | 相对索引的路径。CI 把图标复制到 Pages,文件名带内容哈希,可长期缓存 |
| `plugins[].latest` | CI 记录的**最高的、未被吊销、未暂缓的稳定版本**。宿主不直接用它:宿主在 `versions` 里按自己的规则(`api` 是否支持、`min_host`)挑选 |
| `versions[]` | 最新在前,保留最近 10 个。预发布版本不收录 |
| `versions[].url` | 直接指向 GitHub Release 资产的地址 |
| `versions[].sha256` | `.mgp` 的 sha256。**宿主必须校验** |
| `versions[].permissions` | 该版本 manifest 里的 `permissions` 原样复制,用于在不下载的情况下展示、比较 |
| `versions[].capabilities` / `features` | 同样原样复制。宿主据此计算 [权限变更差异](https://github.com/mygo-music/plugin-sdk/blob/main/docs/07-security.md) 而不必下载 |
| `versions[].settings` | 设置项的摘要:每项的 `key`、`type` 与是否 `grant_network`(不含默认值与文案) |
| `name` `icon` `authors` `repo` | 来自最新收录版本;与上一版不同时该版本暂缓(防止改名冒充) |
| `revoked[].versions` | 版本号的列表,或字符串 `"*"` 表示该插件所有版本(没有版本范围表达式) |

宿主挑选版本时:`api` 不在其支持集合、或 `min_host` 高于当前宿主的版本跳过,取剩下的最高版本。

## 生成

触发:`plugins/` 变更(合并 PR)、定时、手动。步骤:

1. 读取每个 `plugins/<id>.json`。
2. 取仓库的 Release 列表,对每个**未处理过**的非预发布版本:下载 `.mgp`,跑 `mygo-plugin check`,验证来源证明,计算 sha256。
3. **权限比较**:与上一个已收录版本比较。按**覆盖范围**比较主机(`*.example.com` 覆盖 `a.example.com`,`http://x` 比 `x` 多覆盖明文),不是比较字符串;同时比较 `local`、`capabilities`、会改动账号数据的 `features`、`grant_network`/`password` 类设置项,以及名称、图标、作者、仓库。只缩小或不变 → 收录。**有扩大 → 暂缓**:不写入索引,开一个 issue 通知维护者;只有对仓库有写权限的维护者给 issue 加 `approve-update` 标签才放行(工作流要核对加标签的人)。
4. 已收录版本的 `.mgp` 若与记录的 sha256 不符(资产被替换)→ **从新安装中撤下**该版本(不替用户卸载已装的副本),开 issue。为省 API 额度,优先用 GitHub Release 的资产 `digest` 字段或不可变 Release 来判断,而不是每小时把所有版本重新下载一遍。
5. 对任何一步失败的版本:不收录,不影响其他版本和其他插件。
6. 写出 `index.json` 与图标,推送到 `gh-pages`。

索引生成是**幂等**的:已经收录的版本的记录永远不变,除非被吊销。

## 完整性与镜像

- **不签名。** 索引没有额外的签名文件:不引入需要保管的签名私钥。完整性靠 HTTPS、每个版本的 `sha256` 与登记处 CI 对构建来源证明的验证。宿主只核对下载到的**包**与索引里的 `sha256`;**索引本身是被完全信任的**。代价与前提见插件规范的安全模型 §7.7。
- **镜像(待定)。** GitHub 在部分地区访问不稳定。宿主可配置多个索引地址与下载前缀(如 jsDelivr 镜像)。**包**的镜像可以靠索引里的 `sha256` 校验;**索引**的镜像则被完全信任,因为它自己提供 `sha256`。

## 实现注意

- 用 ETag 条件请求,避免耗尽 `GITHUB_TOKEN` 每小时每仓库 1000 次的额度。
- GitHub 会在仓库 60 天没有活动后停用定时工作流:需要保活(例如定期提交一个时间戳文件)。
- 不要用 `pull_request_target` 去跑提交者的代码或脚本;PR 检查只读取提交的 JSON 并通过 API 访问插件仓库。
- 图标由 CI 复制到 Pages,必须检查大小与格式(PNG,256×256,≤ 256 KiB)。
