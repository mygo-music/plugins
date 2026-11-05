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
      "id": "netease",
      "name": "网易云音乐",
      "description": "第三方网易云音乐来源:扫码登录、推荐、搜索、歌单、电台、播客。",
      "i18n": { "en": { "name": "NetEase Cloud Music", "description": "..." } },
      "icon": "icons/netease-1a2b3c4d.png",
      "repo": "mygo-music/plugin-netease",
      "tier": "official",
      "license": "AGPL-3.0-only",
      "authors": [{ "name": "MyGo Music", "github": "mygo-music" }],
      "homepage": "https://github.com/mygo-music/plugin-netease",
      "tags": ["music", "podcast"],
      "capabilities": ["search", "stream", "..."],
      "features": ["like", "..."],
      "latest": "1.2.0",
      "versions": [
        {
          "version": "1.2.0",
          "api": 1,
          "min_host": "0.2.0",
          "url": "https://github.com/mygo-music/plugin-netease/releases/download/v1.2.0/netease-1.2.0.mgp",
          "sha256": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
          "size": 4194304,
          "permissions": {
            "network": { "hosts": ["music.163.com"] },
            "media": { "hosts": ["*.music.126.net"] }
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
| `plugins[].latest` | 宿主应推荐的版本:**最高的、`api` 被宿主支持的、未被吊销的、未暂缓的稳定版本** |
| `versions[]` | 最新在前,保留最近 10 个。预发布版本不收录 |
| `versions[].url` | 直接指向 GitHub Release 资产的地址 |
| `versions[].sha256` | `.mgp` 的 sha256。**宿主必须校验** |
| `versions[].permissions` | 该版本 manifest 里的 `permissions` 原样复制,用于在不下载的情况下展示、比较 |
| `revoked[].versions` | 版本列表,或字符串 `"*"` 表示该插件所有版本 |

宿主挑选版本时:`api` 不在其支持集合、或 `min_host` 高于当前宿主的版本跳过,取剩下的最高版本。

## 生成

触发:`plugins/` 变更(合并 PR)、每小时一次、手动。步骤:

1. 读取每个 `plugins/<id>.json`。
2. 取仓库的 Release 列表,对每个**未处理过**的非预发布版本:下载 `.mgp`,跑 `mygo-plugin check`,验证来源证明,计算 sha256。
3. **权限比较**:与上一个已收录版本比较 `permissions` 和 `capabilities`。只缩小或不变 → 收录。**扩大 → 暂缓**:不写入索引,开一个 issue 通知维护者;维护者给 issue 加 `approve-update` 标签后,下一轮生成时放行。
4. 已收录版本的 `.mgp` 若与记录的 sha256 不符(资产被替换)→ 自动吊销该版本并开 issue。
5. 对任何一步失败的版本:不收录,不影响其他版本和其他插件。
6. 写出 `index.json` 与图标,推送到 `gh-pages`。

索引生成是**幂等**的:已经收录的版本的记录永远不变,除非被吊销。

## 签名与镜像(待定,S5)

- **签名。** 建议 `index.json.sig`(ed25519),宿主内置公钥验证,防止 Pages / 传输被篡改。私钥放在 CI 的 secret 中(与应用更新器的私钥分开)。要不要在 v1 做,待评审。
- **镜像。** GitHub 在部分地区访问不稳定。宿主可配置多个索引地址与下载前缀(如 jsDelivr 镜像)。`sha256` 保证无论从哪里下载都能校验。
