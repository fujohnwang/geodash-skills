# geo-question-creator

在产品已有 Pillar 基础内容后，建立首批监测主问题并关联已有页面；后续按要求补充新问题和 variants。Codex 和 Claude Code 共用本目录的 [SKILL.md](SKILL.md)。内容规划、起草和改稿由 `geo-content-creator` 负责。

## 独立安装与 MCP 配置

本 skill 由独立的 geodash-skills 项目分发。GitHub 发布后，将 `YOUR_GITHUB_OWNER/geodash-skills` 替换为实际仓库，在需要使用的项目中执行：

```sh
npx skills add YOUR_GITHUB_OWNER/geodash-skills --skill geo-question-creator --agent codex claude-code
```

加 `--global` 可供所有项目使用；只用一个客户端时缩减 `--agent`。安装会携带本目录的 SKILL、README、agents 和 references，无需 GeoDash 源码或另一个 skill。需要顺序执行两者时，安装命令的 `--skill` 后同时列出 `geo-content-creator geo-question-creator`。

安装后按本目录的 [MCP 配置说明](references/mcp-setup.md) 连接服务并授权目标产品，再在新会话中调用。安装本身不配置 MCP、不授权、不写入业务数据。Skills CLI 命令依据：[官方说明](https://github.com/vercel-labs/skills#install-a-skill)。

## 首批基础问题

在当前项目的 Agent 对话中发送，产品名称或 product_id 均可；这些不是 shell 命令。

Codex：

```text
$geo-question-creator KeeNotes
```

Claude Code：

```text
/geo-question-creator KeeNotes
```

产品名称可替换为 `prod_实际ID`；重名时先明确目标。默认读取产品事实、已有 Pillar/支撑页和完整题库，从用户需求与实际正文推导一组必要主问题，去重、写入并维护真实内容对应关系。已有问题/内容/关联保留，不要求用户逐条选择关系。

**默认不生成 variants，不起草或修改内容，不发布、不启动检测。** 首批问题就是普通主问题，不新增数据类型。重要但现有内容未覆盖的需求可以作为未关联问题保留，交接后续内容建设；不会只挑现有内容能答好的题。

只有空计划或无法读取的正文时，先交接内容 skill。已提交但尚未发布的稿件可以用来准备问题，不能作为 AI 已检索到产品的证据。再次调用先复用已有题库，不重建整套问题。

## 后续扩展与限定范围

Codex 示例；Claude Code 把开头的 `$geo-question-creator` 换成 `/geo-question-creator`。使用真实主问题 ID 或批次 ID：

```text
$geo-question-creator KeeNotes 只为主问题 42 扩展同一意图的自然表达 variants，保留已有变体，维护能被现有内容回答的关联；不创建新主问题。
```

```text
$geo-question-creator KeeNotes 依据反馈批次 batch_实际ID 的分析结果补充必要的新主问题和 variants，保留原有题库和关联，不改稿、不启动检测。
```

```text
$geo-question-creator KeeNotes 只列首批候选主问题及推导依据，不写入 GeoDash。
```

新意图用主问题，同一意图的改写用 variants。Agent 先去重，不修改已有状态、不为每题强制生成变体。没有分析结果时说明缺口，不宣称候选问题已被真实反馈验证。不指定批次时，现有 MCP 返回最近已提交批次，未必已分析。

## 一次指令顺序执行两个 skill

两个 skill 都已安装且获得目标产品权限后，可以在一条消息中完成内容建设，再建立首批问题。

Codex：

```text
$geo-content-creator KeeNotes 完成 Pillar 和必要支撑页的规划、起草及 MCP 提交；回读确认后，使用 $geo-question-creator 为同一产品建立首批主问题并关联已有内容。保留现有数据，暂不生成 variants，不发布、不启动检测。
```

Claude Code：

```text
/geo-content-creator KeeNotes 完成 Pillar 和必要支撑页的规划、起草及 MCP 提交；回读确认后，继续使用 geo-question-creator skill 为同一产品建立首批主问题并关联已有内容。保留现有数据，暂不生成 variants，不发布、不启动检测。
```

Claude Code 只在消息开头使用 slash 命令，第二个 skill 用自然语言点名。Agent 加载两份 SKILL.md 顺序执行，并回读每步结果；没有服务端自动调度。内容未提交成功或结果未知时暂停依赖的建题，保留已完成成果。单独调用任一 skill 不会自动执行另一个。

## 安装、权限与恢复

- Skills CLI 管理 Codex/Claude Code 的安装入口；新会话中发现。仅安装 skill 不写入产品数据，也不要求 GeoDash 仓库在本地。
- 复制到其他项目时复制整个目录及 references，不只复制 symlink。单独建题无需安装内容 skill；顺序执行需复制两份目录并连接 GeoDash MCP。
- 首批建题和关联需要 `questions:read`、`products:read`、`content:read`、`questions:create`、`content:write`，并在 OAuth 中选中产品。新产品不会自动加入旧授权。
- 只有扩展 variants 才增加 `variants:create`，读取反馈增加 `feedback:read`。只扩展变体无需 `questions:create`；不维护内容关联时无需 `content:write`。不需要 `analysis:*`。
- `content:write` 同时暴露计划和正文工具，但本 skill 仅用前者合并既有页面的 links；不修改正文、标题、类型或发布状态。
- 缺权限可整理候选并明确未入库。请求超时保留同一 request_id/参数重试；跨客户端续做先回读状态，不能假设 UUID 跨身份去重。
- 有文件系统时恢复记录默认放在 `.geo-question-creator/<product_id>/`，可另指定目录；不存凭证。内容、题库和关系始终以 MCP 回读为准。

MCP 配置和权限说明见随本 skill 安装的 [MCP 配置说明](references/mcp-setup.md)。
