# GeoDash Skills

为 [GeoDash](https://geo.afoo.me/web/products) 提供两个可独立安装的 Agent Skills，兼容 Codex 和 Claude Code。通过已授权的 GeoDash MCP 读写数据，不依赖 GeoDash 源码、本机目录或特定模型。

| Skill | 默认任务 | 后续任务 |
| --- | --- | --- |
| [geo-content-creator](skills/geo-content-creator/README.md) | 规划、起草 Pillar 和支撑页，通过 MCP 提交；不建题 | 根据已有问题和反馈完善稿件 |
| [geo-question-creator](skills/geo-question-creator/README.md) | 基于产品事实和已提交内容建立首批主问题，维护内容关联；默认不生成 variants | 按要求扩展新意图的问题或同意图的 variants |

## 安装

使用 [Skills CLI](https://github.com/vercel-labs/skills)。命令是 `npx skills add`；从 GitHub 读取 SKILL.md 及其支持文件，无需将本项目发布到 npm。

本地项目尚未推送 GitHub。以下 `YOUR_GITHUB_OWNER/geodash-skills` 是仓库占位符，发布后替换为实际的 GitHub 用户/组织和仓库名。在准备使用 skill 的项目目录执行：

```sh
# 查看两个 skill，不安装
npx skills add YOUR_GITHUB_OWNER/geodash-skills --list

# 只安装内容 skill，同时供 Codex 和 Claude Code 使用
npx skills add YOUR_GITHUB_OWNER/geodash-skills --skill geo-content-creator --agent codex claude-code

# 只安装问题 skill
npx skills add YOUR_GITHUB_OWNER/geodash-skills --skill geo-question-creator --agent codex claude-code

# 同时安装两者
npx skills add YOUR_GITHUB_OWNER/geodash-skills --skill geo-content-creator geo-question-creator --agent codex claude-code
```

默认安装到当前项目；需要所有项目都能使用时加 `--global`。只用一个客户端时，`--agent` 后只保留 `codex` 或 `claude-code`。重新打开客户端会话以发现新 skill。安装 skill 不会自动配置 MCP、授权产品或写入业务数据。

GitHub 发布前，可从另一个项目目录使用本地 checkout：

```sh
npx skills add ../geodash-skills --list
npx skills add ../geodash-skills --skill geo-content-creator geo-question-creator --agent codex claude-code
```

如果两个项目不在同一父目录，将 `../geodash-skills` 换成本地 checkout 的实际路径。下载或复制时保留整个 skill 目录，包括 `references/` 和 `agents/`。

## 连接 GeoDash MCP

需要访问已部署的 GeoDash 服务，并由产品管理员通过 OAuth 授权目标产品。仅安装 skill 不代表拥有该服务的账号或权限。

- 内容建设最小权限：`questions:read`、`products:read`、`content:read`、`content:write`。
- 首批建题及关联：再加 `questions:create`。
- 扩展 variants 时加 `variants:create`；读取反馈报告时加 `feedback:read`。
- 不需要 `analysis:*`、Cloudflare 凭证、`K_TOKEN` 或模型 API Key。

每个安装包都包含独立的 [MCP 配置说明](skills/geo-content-creator/references/mcp-setup.md)，单独安装任意一个也可以完成配置。示例服务为 `https://geo.afoo.me/mcp`；使用其他部署时，替换为产品管理员提供的 MCP URL。服务端实际 schema 和 OAuth 权限是调用依据。

## 工作流与调用方式

1. 在 GeoDash 新建产品并维护定位、可信事实及来源。
2. 使用 `geo-content-creator` 规划、起草、提交 Pillar 与支撑页；无需先建问题库。
3. 使用 `geo-question-creator` 基于可读取的稿件建立首批主问题，按实际覆盖维护内容关联；默认不生成 variants。
4. 人工发布内容、回填实际 URL 并核验；另行创建 Feedback Batch、采样，交给外部分析 Agent 执行分析。
5. 依据反馈由问题 skill 扩展题库、内容 skill 完善稿件，再发布、复测。

未发布但已提交的稿件可以用于准备问题，不能作为已被 AI 检索到的证据。这是协作流程，不是服务端自动调度；两个 skill 均保留已有成果和关联，不自动发布、采样或分析。

在 Agent 对话中发送，`产品名` 可替换为 `product_id`；这些不是 shell 命令：

| 任务 | Codex | Claude Code |
| --- | --- | --- |
| 内容建设 | `$geo-content-creator 产品名` | `/geo-content-creator 产品名` |
| 首批问题 | `$geo-question-creator 产品名` | `/geo-question-creator 产品名` |

### 一条消息顺序执行两个 skill

两个 skill 均已安装、MCP 已授权后，Codex：

```text
$geo-content-creator 产品名 完成 Pillar 和必要支撑页的规划、起草及 MCP 提交；回读确认后，使用 $geo-question-creator 为同一产品建立首批主问题并关联已有内容。保留现有数据，暂不生成 variants，不发布、不启动检测。
```

Claude Code：

```text
/geo-content-creator 产品名 完成 Pillar 和必要支撑页的规划、起草及 MCP 提交；回读确认后，继续使用 geo-question-creator skill 为同一产品建立首批主问题并关联已有内容。保留现有数据，暂不生成 variants，不发布、不启动检测。
```

Claude Code 只在消息开头使用 slash 命令，第二个 skill 用自然语言点名。Agent 顺序读取两份 SKILL、交接同一产品的实际回执；前一步失败或结果未知时，保留成功成果，暂停依赖步骤。单独调用一个 skill 不会自动执行另一个。

## 仓库结构与维护

```text
geodash-skills/
├── README.md
├── implementation_note.md
└── skills/
    ├── geo-content-creator/
    │   ├── SKILL.md
    │   ├── README.md
    │   ├── agents/openai.yaml
    │   └── references/
    └── geo-question-creator/
        ├── SKILL.md
        ├── README.md
        ├── agents/openai.yaml
        └── references/
```

每个 skill 的所有相对引用都留在自己的目录内，单独安装后不依赖另一个 skill 或仓库顶层文档。两份 `mcp-setup.md` 随各自安装包分发，维护连接说明时同步更新。业务契约参考 GeoDash 实现，执行时始终发现实际工具 schema；服务端不提供的接口不在 skill 中假设存在。

本项目没有构建、数据库或部署步骤。发布到 GitHub 后，用户直接使用上面的安装命令；本地维护可先运行 `npx skills add . --list` 检查发现结果，再在临时项目分别安装验证。
