# MyGo 音乐 · 插件市场

> **状态:草案。** 收录标准在评审,CI 与索引生成尚未实现。

这是 MyGo 音乐的**插件登记处**。这里登记的插件,会出现在应用内的「插件市场」。

- 想**写插件** → [`mygo-music/plugin-sdk`](https://github.com/mygo-music/plugin-sdk)
- 想**提交插件到市场** → 往下读
- 想**不进市场、直接分享给别人** → 不用提交。把你的仓库地址发给用户,在应用里选「从 URL 安装」即可

## 提交一个插件

你只需要添加**一个文件**。

1. 用 SDK 写好插件,并在你自己的仓库发布一个 Release(SDK 模板自带的工作流会构建 `<id>-<version>.mgp`、生成构建来源证明并上传)。
2. Fork 本仓库,新建 `plugins/<id>.json`:

   ```json
   {
     "$schema": "https://mygo-music.github.io/plugins/schema/plugin.schema.json",
     "id": "my-source",
     "repo": "your-name/plugin-my-source"
   }
   ```

   `id` 必须与插件 `manifest.json` 里的 `id` 一致,也是文件名。**其他信息(名称、说明、图标、权限、版本)都从你的 Release 里读取,不必重复填写。**
3. 发起 Pull Request。机器人会在几分钟内给出检查结果(下表),按提示改到全绿。
4. 维护者人工评审(权限是否最小、是否符合[收录政策](CONTRIBUTING.md#收录政策))。通过后合并。
5. 合并后几分钟内,插件出现在市场。之后你每发一个新版本,索引会自动跟进,**不需要再提 PR**。

### 机器人检查什么

| 检查 | 说明 |
|---|---|
| 条目格式 | 符合 [`schema/plugin.schema.json`](schema/plugin.schema.json);文件名 = `id` |
| 仓库 | 公开、未归档、有许可证 |
| Release | 最新 Release 的标签是 `v<version>`,且带 `<id>-<version>.mgp` |
| 包 | 结构合法;`mygo-plugin check` 通过(与你本地跑的同一个) |
| 一致性 | `manifest.id` = 条目的 `id`;`manifest.repository` = 条目的 `repo` |
| 构建来源 | `.mgp` 带 GitHub 构建来源证明,且由 `repo` 的工作流构建 |
| 提交者 | PR 作者是 `repo` 的所有者,或列在 `manifest.authors[].github` 里 |
| 重名与仿冒 | `id` 唯一;与已有插件或知名服务过于相似的名字转人工 |
| 权限 | 过宽的主机通配、`network.local` 转人工 |

### 之后:更新、权限变化、下架

- **发新版本**:自动收录。
- **新版本扩大了权限**(新增主机、`local`、新能力):该版本先**暂缓**,由维护者评审后放行。旧版本继续可用。
- **下架**:删除你的 `plugins/<id>.json`(PR 即可)。已安装的用户不受影响,但不再收到更新。
- **紧急撤销某个版本**:见 [CONTRIBUTING](CONTRIBUTING.md#吊销与下架)。

## 层级

| 层级 | 条件 | 标记 |
|---|---|---|
| 官方 | 仓库属于 `mygo-music` 组织 | 「官方」 |
| 社区 | 其他 | 「社区」 |

层级由仓库所有者**自动判定**,不可在条目里自填。

## 文档

- [收录政策与评审标准](CONTRIBUTING.md)
- [索引格式 `index.json`](docs/index-format.md)
- [插件规范](https://github.com/mygo-music/plugin-sdk/tree/main/docs)

## 许可证

[Apache-2.0](LICENSE)。登记的插件各自遵循其仓库的许可证。
