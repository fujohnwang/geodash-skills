# 连接 GeoDash MCP

安装 skill 与连接 MCP 是两件事。需要一个可访问的 GeoDash 服务、已有产品，以及管理员对客户端和目标产品的 OAuth 授权。本 skill 不要求安装 GeoDash 源码，不向用户索取 K_TOKEN 或 Cloudflare 凭证。

下面使用 `https://geo.afoo.me/mcp`。连接自己的部署时，将 URL 替换为管理员提供的 MCP 入口。不要使用 `/web/api/v1`、public API token 或模型 API Key 代替 OAuth。

## 按任务选择权限

| 任务 | OAuth scopes |
| --- | --- |
| Pillar 内容规划、起草、提交 | `questions:read products:read content:read content:write` |
| 首批主问题与内容关联，或顺序执行两个 skill | 上述权限加 `questions:create` |
| 扩展 variants | 加 `variants:create`；只扩展已有题目时不需要 `questions:create` |
| 依据已有反馈优化 | 加 `feedback:read` |

所有请求都需要 `questions:read`。只做 variants 且不维护关联时无需 `content:write`。两个 skill 均不需要 `analysis:*`。工具列表会按权限筛选，连接成功不代表已获得所有工具。

## Codex

首次添加服务：

```sh
codex mcp add geodash --url https://geo.afoo.me/mcp
```

按范围请求权限。以下用于内容建设和首批建题，不含 variants 或反馈读取：

```sh
codex mcp login geodash --scopes questions:read,products:read,content:read,content:write,questions:create
```

只用内容 skill 时去掉 `questions:create`；要扩展 variants 或读取报告时再加入对应 scope。已有同名连接时保留配置，按需要更新 URL/权限并重新授权，不重复添加或覆盖其他配置。

## Claude Code

首次添加支持内容建设与首批建题的连接：

```sh
claude mcp add-json --scope user geodash '{"type":"http","url":"https://geo.afoo.me/mcp","oauth":{"scopes":"questions:read products:read content:read content:write questions:create"}}'
claude mcp login geodash
```

`oauth.scopes` 是空格分隔的字符串，按任务删减或添加权限。`--scope user` 表示 MCP 配置的保存范围，与 OAuth scope 不同。已有同名连接时只修改其所需字段，保留其他配置；也可在 Claude Code 的 `/mcp` 中认证或重新连接。若所用版本没有 `mcp login`，使用 `/mcp` 的认证入口。

## 授权与验证

1. 在授权页由管理员登录，选择真实目标产品，确认本轮所需权限。新产品不会自动加入旧授权，需要重新授权选中。
2. 让 Agent 列出当前 GeoDash 工具，再用 `list_products` 和 `get_product` 验证目标可读。只读验证不创建计划或问题。
3. 内容 skill 需要 `list_content_pages`、`get_content_page`、`upsert_content_plan`、`submit_generated_content`；问题 skill 需要问题读取、按本轮范围提供 `create_question` / `add_question_variants`，维护关联还需要 `upsert_content_plan`。
4. 缺少工具时核对授权 scope、产品范围和服务版本，必要时重新连接；不能用另一入口绕过权限。调用前以实际 MCP schema 为准，具体参数见同目录的 [MCP 契约](mcp-contract.md)。

连接或安装过程不自动执行两个 skill；内容起草、建题与提交仍需用户在 Agent 对话中明确安排。恢复日志放在当前工作项目时，可将 `.geo-content-creator/` 和 `.geo-question-creator/` 加入该项目的 `.gitignore`，日志中不保存凭证。
