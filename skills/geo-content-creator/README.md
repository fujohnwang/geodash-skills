# geo-content-creator

围绕 GeoDash 中已有产品，完成 Pillar 与支撑页的规划、起草和 MCP 提交；后续根据已有问题和反馈完善内容。建题与 variants 扩展使用独立的 geo-question-creator。Codex 和 Claude Code 共用同一份 [SKILL.md](SKILL.md)。

## 独立安装与 MCP 配置

本 skill 由独立的 geodash-skills 项目分发。GitHub 发布后，将 `YOUR_GITHUB_OWNER/geodash-skills` 替换为实际仓库，在需要使用的项目中执行：

```sh
npx skills add YOUR_GITHUB_OWNER/geodash-skills --skill geo-content-creator --agent codex claude-code
```

加 `--global` 可供所有项目使用；只用一个客户端时缩减 `--agent`。安装会携带本目录的 SKILL、README、agents 和 references，无需 GeoDash 源码或另一个 skill。需要顺序执行两者时，安装命令的 `--skill` 后同时列出 `geo-content-creator geo-question-creator`。

安装后按本目录的 [MCP 配置说明](references/mcp-setup.md) 连接服务并授权目标产品，再在新会话中调用。安装本身不配置 MCP、不授权、不写入业务数据。Skills CLI 命令依据：[官方说明](https://github.com/vercel-labs/skills#install-a-skill)。

## 快速使用

在当前项目打开客户端会话，命令后直接跟产品名称或 `product_id` 即可。这些是发送给 Agent 的指令，不是终端 shell 命令。

### Codex

```text
$geo-content-creator KeeNotes
```

也可以指定产品 ID（替换为真实 ID）：

```text
$geo-content-creator prod_实际ID
```

### Claude Code

```text
/geo-content-creator KeeNotes
```

也可以指定产品 ID（替换为真实 ID）：

```text
/geo-content-creator prod_实际ID
```

只跟产品时，默认执行一轮 **Pillar 基础内容建设**：读取产品资料和已有内容 → 规划 Pillar 与必要支撑页 → 保存内容计划 → 逐篇起草 → MCP 提交生成稿。**不自动创建主问题或衍生问题，不要求已有题库。** 文中的 FAQ、读者疑问只是写作素材，不会自动成为监测条目。

已有内容会先检查、复用，不会因再次调用就全部重建；已有问题和关联会保留。基础内容已经足够时，Agent 会说明现状，不自动转入建题或反馈分析。站点 URL 在发布时确定；未发布页面的内链关系用内容 ID、标题交接，人工发布后在 GeoDash Web 回填发布地址并核验。

产品名称重名时，Agent 会要求明确目标；未指定产品时，也会先询问。默认中文，可以在指令后指定其他语言、主题或页面范围。

## 从新产品开始

第一阶段：在 GeoDash 新建产品并录入定位、受众、实际功能与可信事实 → 授权 MCP 访问该产品 → 调用 skill 从零规划 Pillar 和支撑页 → 起草并提交 → 人工审稿、在站点发布 → 回填 published_url 并核验。新内容计划允许无问题关联，这不影响基础内容完成交付。

产品不需要已有营销网站。Agent 以产品资料为起点，关键事实不足时向产品方确认；官网只是可选的补充来源。对已有网站的产品，应先评估现有内容再决定复用或新增。KeeNotes 的本次验证从空的 GeoDash 内容库起草，但其官网已经存在，因此不代表完成了尚未建站产品的完整发布验证。

## 内容完成后建立首批问题

使用 `geo-question-creator`，从产品资料和实际正文推导监测问题，并关联能回答它们的已有页面。默认只创建主问题，不生成 variants、不改稿或启动检测：

```text
$geo-question-creator KeeNotes
```

```text
/geo-question-creator KeeNotes
```

基础稿已保存即可准备问题，无需等到发布；发布、URL 核验和反馈采样仍是后续独立步骤。题库已有内容时先复用，不重新建立一套。内容 skill 本身不再创建问题或 variants。

## 一次指令顺序执行两个 skill

在同一条消息中明确先后顺序，Agent 先保存并回读内容，再加载问题 skill，用同一 product_id 建立首批主问题、维护关联。两个 skill 都需已安装且 MCP 已获相应产品/写权限。

Codex：

```text
$geo-content-creator KeeNotes 完成 Pillar 和必要支撑页的规划、起草及 MCP 提交；回读确认后，使用 $geo-question-creator 为同一产品建立首批主问题并关联已有内容。保留现有数据，暂不生成 variants，不发布、不启动检测。
```

Claude Code：

```text
/geo-content-creator KeeNotes 完成 Pillar 和必要支撑页的规划、起草及 MCP 提交；回读确认后，继续使用 geo-question-creator skill 为同一产品建立首批主问题并关联已有内容。保留现有数据，暂不生成 variants，不发布、不启动检测。
```

Claude Code 只在消息开头使用 slash 命令，后续用自然语言指明第二个 skill。Agent 读取第二份 SKILL.md 后继续执行；这不是服务端自动编排，也不需要创建新会话。前一步提交失败或结果未知时，保留成功成果，暂停依赖它的步骤；再次执行先读远端状态，避免重建。仅调用内容 skill 时不会自动追加建题。

## 后续完善内容

取得反馈后，问题补充/variants 扩展交给问题 skill；新内容与改稿交给内容 skill。例子中的 batch_实际ID 替换为真实批次 ID：

```text
$geo-content-creator KeeNotes 根据反馈批次 batch_实际ID 的分析结果和已有问题补充、修订内容并通过 MCP 提交，复用已有页面并保留关联，不发布。
```

```text
/geo-content-creator KeeNotes 根据反馈批次 batch_实际ID 的分析结果和已有问题补充、修订内容并通过 MCP 提交，复用已有页面并保留关联，不发布。
```

没有报告或分析尚未完成时会明确说明，不把猜测写成观测效果。不指定批次时，现有 MCP 只能按产品取最近已提交批次，不保证已经分析。读取反馈需 feedback:read；两个 skill 都不执行采样或分析任务。

## 限定本轮范围

直接在产品后补充要求。以下用 Codex 语法示例；Claude Code 将开头的 `$` 换成 `/` 即可。

```text
$geo-content-creator KeeNotes 只规划并入库，暂不起草
```

```text
$geo-content-creator KeeNotes 只生成本地草稿，不写入 GeoDash
```

```text
$geo-content-creator KeeNotes 继续上次未完成的稿件并提交，保留已完成内容
```

```text
$geo-content-creator KeeNotes 用英文起草隐私主题的 Pillar，保存到 GeoDash
```

## 使用前提

- 产品已在 GeoDash 建立，具备足够的简介与可信事实；关键资料不足时，Agent 会列出缺口。
- 客户端已连接 GeoDash MCP，并通过 OAuth 授权目标产品。新增产品需要重新授权选中。
- 基础建设需要 `products:read`、`content:read`、`content:write`，当前 MCP 入口还要求 `questions:read`；无需 `questions:create` 或 `variants:create`，题库为空也可完成整轮建设。
- 内容优化读取反馈时增加 `feedback:read`；建题权限 `questions:create` 与变体权限 `variants:create` 仅供问题 skill 按范围使用。顺序执行首批建题需额外授予 `questions:create`；维护关系沿用 `content:write`。
- 当前连接的 MCP 服务必须提供产品和内容工具。安装 skill 不会自动升级服务端或启动后台任务。

Skills CLI 管理 Codex/Claude Code 的安装入口。手工复制时复制本 skill 的整个目录，保留 `references/` 和 `agents/`，不要只复制符号链接。顺序执行两个 skill 时需分别安装两者；单独内容建设不依赖问题 skill。

可写文件系统下，规划及恢复记录默认保存到 `.geo-content-creator/<product_id>/`，也可以指定输出目录。GeoDash 中的实际数据与版本始终是状态依据；跨客户端续做时先核对已提交结果，避免重复创建。
