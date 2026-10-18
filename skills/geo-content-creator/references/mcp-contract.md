# 内容工作流 MCP 契约

本契约依据 GeoDash 服务端实现核对；安装本 skill 无需持有服务端源码。运行时以当前连接实际 MCP schema 为准，工具短名可能带客户端前缀。缺少工具时核对服务版本和 OAuth 权限，不臆造接口；连接方法见 [MCP 配置说明](mcp-setup.md)。

## 读取与权限

| 工具 | 输入 | 输出和用途 | Scope |
| --- | --- | --- | --- |
| `list_products` | `{}` | `{items: Product[]}`，仅当前授权产品，无分页参数 | products:read |
| `get_product` | `product_id: string` | Product，含 facts、aliases、official_domain、allowed_domains、revision | products:read |
| `list_questions` | `product_id: string`；可选 limit 1–100、cursor 整数、status | `{items, next_cursor}`；每项含 variants，cursor 为数字或 null | questions:read |
| `get_question` | `question_id: positive integer` | 主问题及 variants | questions:read |
| `list_content_pages` | `product_id: string`；可选 limit 1–100、cursor 字符串、status | `{items, next_cursor}`；cursor 为字符串或 null | content:read |
| `get_content_page` | `content_page_id: string` | 完整页面，含正文、问题关联、revision、content_version 和生命周期 | content:read |
| `get_geo_feedback` | batch_id 或 product_id；可选 run_id | 指定批次或产品最近已提交批次的报告，含原始回答、分析与证据 | feedback:read |

提供 `next_cursor` 的读列表循环使用该值，直到严格等于 null；不要自行计算下一页。此工作流在 `list_questions` 中总是传 product_id。问题 ID 和 variant ID 是数字，产品和内容 ID 是字符串，不能混用。

MCP 依赖 OAuth；所有会话还要求 questions:read。基础建设只需产品/内容读取和内容写入，不需要题库非空或问题写权限；默认不调用问题创建工具。内容优化时按本轮范围读取已有题库并维护对应关系，不调用问题创建或 variants 工具。get_geo_feedback 无可用批次时返回 REPORT_NOT_AVAILABLE，已提交批次也可能尚未分析，不能把它视为已有结论。新建产品需要重新授权选中；只换 scope 并不自动包含新产品。缺工具可能是 scope 不足或服务端版本不同。不得索取 `K_TOKEN`、读取浏览器 token 或改用 `/web/api/v1` 绕过 OAuth。

调用成功通常同时提供 `structuredContent` 和 JSON 文本 content；使用客户端提供的结构化结果，若只暴露文本则解析 JSON。检查 `isError` 与 `{error:{code,message}}`，不能仅凭 HTTP 200 判断成功。

## 写入参数

### upsert_content_plan

- 必填：`request_id`、`product_id`、`title`（非空，最多 200 字符）、`page_type`。
- page_type：pillar / feature / use_case / comparison / faq / trust / article。
- `links`：最多 100 项 `{question_key, role}`，缺省 []。基础新页可省略或传 []，不需要先建问题。question_key 为 `q:<主问题数字ID>` 或 `v:<variant数字ID>`；role 为 primary / supporting（缺省 supporting）。不能重复关联同一个 key，必须属于当前产品。
- 更新时还要传 `content_page_id` 和正整数 `revision`；创建时两者都省略。Scope：content:write。
- 返回 ContentPage。新建状态 planned、正文版本 0；更新计划不等于产生新稿。修改已核验页面标题会使其退回 published、需要重新核验。
- **这是完整计划替换，不是 PATCH**：更新时从最新页面取出 title、page_type、links，再合并本次有意修改，否则省略的关联会恢复缺省值。不得原样提交整个 ContentPage（schema 是 strict）。
- 不支持 `outline`、`parent_id`、`children`、`slug`、`seo_title`、`meta_description`、`published_url`、`content_body` 等额外参数。

### submit_generated_content

- 必填：`request_id`、`content_page_id`、当前正整数 `revision`。
- `content_body`：最多 100000 字符；`draft_ref`：null 或不含凭据/自定义端口的 HTTPS URL（最多 2048 字符）。至少一项非空。不支持 file://、本地路径或虚构 URL。
- Scope：content:write。返回更新后的 ContentPage：content_version + 1、revision + 1、content_status=generated、核验信息清除。
- 这是完整新稿版本；传正文但不传 draft_ref 时旧引用会被清空，反之正文会变为空。保留哪些内容要显式决定。
- 不接收 product_id、content_status 或 published_url。用户需在 Web 回填/核验；旧 published_url 可能仍在返回值中，不能视为新稿发布证据。

## 基础建设：无需先创建问题

以下仅演示参数形状。所有 product_id、内容 ID、问题 ID 与 revision 必须替换为当前服务真实读取/返回的值；每个新操作生成自己的 UUID，禁止直接复用示例 UUID。

1. `upsert_content_plan` 创建 Pillar 计划，不要求问题 ID：

```json
{
  "request_id": "2f5af763-65a8-48b2-ae7c-faa4b8eaee1b",
  "product_id": "prod_example",
  "title": "个人笔记与隐私设置指南",
  "page_type": "pillar",
  "links": []
}
```

2. 假设计划返回 id=cp_example、revision=1，重新读取确认未变后用 `submit_generated_content` 提交稿件：

```json
{
  "request_id": "87f26416-acad-4e0c-b865-fbe7c7ca7c49",
  "content_page_id": "cp_example",
  "revision": 1,
  "content_body": "# 个人笔记与隐私设置指南\n\n选择工具时，先确定哪些内容需要保存、谁需要访问，以及希望如何导出。再根据候选产品的官方说明核对设置。\n\n## 核对访问权限\n\n分别检查个人笔记与共享笔记的可见范围，并用非所有者账户验证。",
  "draft_ref": null
}
```

上面的短正文仅演示参数，不是完整内容交付示范。成功后用 `get_content_page` 核对正文、版本、generated 状态和 links=[]；空关联不影响本轮基础内容交付，题库仍可为空。

## 内容优化：使用已有问题关联

本 skill 不创建问题或 variants；这些由 geo-question-creator 负责。以下只演示为本轮内容使用已有问题的参数。

假设已通过 list_questions/get_question 核对目标产品的问题 id=42，重新读取 cp_example，确认当前 revision=2、links=[]，用 `upsert_content_plan` 给既有内容补充关系：

```json
{
  "request_id": "d90af77d-9af0-40b5-adcc-92f2856fe44d",
  "product_id": "prod_example",
  "content_page_id": "cp_example",
  "revision": 2,
  "title": "个人笔记与隐私设置指南",
  "page_type": "pillar",
  "links": [{"question_key": "q:42", "role": "primary"}]
}
```

此示例因为旧 links 为空才只传一条；实际有关系时需与最新完整 links 合并去重，保留已有角色，不重新建页或清空关联。仅补充关系不产生正文版本。需要改稿时，再用返回的最新 revision 提交有实质变化的正文。需要新增问题或 variants 时交接 geo-question-creator，不在内容 skill 中入库。

## 重试与冲突

| 情况 | 行动 |
| --- | --- |
| 超时、连接中断、INTERNAL_ERROR，提交结果未知 | 保留同一工具、原始全部参数（含原 revision）、原 request_id 和同一 OAuth 客户端，至多重试 2 次；仍无法确认则记录为待核对，继续独立项 |
| IDEMPOTENCY_CONFLICT | 该 UUID 已对应其他操作或参数；停止这项写入，核对恢复记录，不能换 UUID 掩盖不确定的结果 |
| CONFLICT / REVISION_REQUIRED | 重新读取页面，比较远端改动；确定如何合并后，新的有意写入使用新 UUID 和最新 revision。反复冲突说明有并发编辑，暂停该项 |
| VALIDATION_ERROR 等明确参数错误 | 按当前 schema 修正参数；已确认旧请求没有成功后，新操作使用新 UUID |
| PRODUCT_FORBIDDEN / OAuth scope 不足 / 工具缺失 | 请求补充对应产品或权限；保留未提交稿件，不能绕过权限 |
| NOT_FOUND 或产品/问题归属不匹配 | 重新确认真实目标与 ID；不改写其他产品的数据以绕过错误 |

请求去重按认证身份隔离。Codex 与 Claude Code 可能是不同 OAuth 客户端，同一 UUID 在跨客户端或 REST/MCP 间不保证去重。跨客户端续做以远端数据与本地回执核对，尤其不要盲目重放未知结果的创建操作。

本地恢复记录应保存原参数而不只是 UUID/哈希，才能精确重试。可保存为 worklog.json，包含 product_id、本轮范围和 operations（tool、request_id、arguments、outcome、返回 ID/revision/version），以及 remaining；稿件可独立保存。远端数据优先，记录中不含任何认证凭证。
