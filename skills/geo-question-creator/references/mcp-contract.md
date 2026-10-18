# 问题工作流 MCP 契约

本契约依据 GeoDash 服务端实现核对；安装本 skill 无需持有服务端源码。运行时以当前连接实际 MCP schema 为准，工具短名可能带客户端前缀。缺少工具时核对服务版本和 OAuth 权限，不臆造接口；连接方法见 [MCP 配置说明](mcp-setup.md)。

## 读取与权限

| 工具 | 输入 | 返回与用途 | Scope |
| --- | --- | --- | --- |
| list_products | {} | {items}，仅已授权产品，无分页 | products:read |
| get_product | product_id | 产品资料、facts、域名与 revision | products:read |
| list_content_pages | product_id；可选 limit 1–100、cursor 字符串、status | {items,next_cursor}，内容与生命周期 | content:read |
| get_content_page | content_page_id | 完整正文、draft_ref、links、版本、发布/核验信息 | content:read |
| list_questions | 本 skill 必传 product_id；可选 limit 1–100、cursor 整数、status | {items,next_cursor}，每项含 variants | questions:read |
| get_question | question_id 正整数 | 主问题及 variants，核对归属与去重 | questions:read |
| get_geo_feedback | batch_id 或 product_id；可选 run_id | 指定批次或产品最近已提交批次的报告 | feedback:read |

有 next_cursor 的列表读到其严格等于 null；不自行计算 cursor。去重时不要只取 active 子集。问题/variant ID 是数字，产品/内容 ID 是字符串。

MCP 所有请求需要 questions:read。首批建题加 questions:create；关联页面加 content:write；扩展变体加 variants:create。反馈无可用批次时返回 REPORT_NOT_AVAILABLE，最近已提交批次可能尚未分析。新产品需在 OAuth 中选中，不自动加入旧授权。

正常调用检查 structuredContent、isError 和 error.code；客户端只提供文本时解析 JSON。HTTP 200 不代表业务成功。缺权限时保留未提交项，不调用 Web API 或索取 K_TOKEN 绕过 OAuth。

## 写入工具

### create_question

- 必填 request_id（UUID）、question（非空、最多 2000 字符）；本 skill 还必须传 product_id。
- 可选 intent=generic|branded、topic（最多 200）、language（最多 32）、status=active|inactive。默认 topic 为空、language=zh-CN、status=active。
- 返回完整 Question，含数字 id 和 variants。不按文本自动去重，写入前先读题库。

### add_question_variants

- 必填 request_id、question_id、variants（1–100 项，每项 {question,source?}）。question 非空、最多 2000 字符；source 最多 64。
- 顶层 source 可选，MCP 缺省 agent。每个主问题最多 1000 个 variants。
- 无 product_id 参数，先读 get_question 核对归属。返回更新后的完整 Question。
- 现有 MCP 不提供问题/variant 的修改、删除、启停工具；不通过新建重复记录模拟这些操作。

### upsert_content_plan：本 skill 仅维护已有页面的关联

- 必传 request_id、product_id、content_page_id、当前正整数 revision、原 title、原 page_type、合并后的完整 links。
- title 非空且最多 200；page_type 为 pillar/feature/use_case/comparison/faq/trust/article。
- links 最多 100 项 {question_key,role}，key 为 q:<主问题ID> 或 v:<variantID>，role 为 primary/supporting。必须属于同一产品，不允许重复 key。
- **这是完整替换，省略 links 会变成 []。** 先重新读取页面，保留所有旧 links 和角色，只合并确有依据的新关系。没有变更不调用。不得原样提交整个 ContentPage，schema 是 strict。
- 仅更新关系不会生成正文新版本。保留 title/page_type，避免修改已核验标题而触发重新核验；回读确认正文、引用、版本、发布/核验状态和 URL 未变。
- 不传 content_body、draft_ref、published_url 或 content_status。本 skill 不创建页面，不调用 submit_generated_content。

## 参数示例

以下只演示形状。产品、问题、页面 ID 和 revision 都替换为真实读取/返回值，每个新操作生成自己的 UUID；不要复用示例 UUID。

1. Pillar 已提交并读取后，创建首批主问题中的一项：

```json
{
  "request_id": "91f4b7f2-a71f-4b87-91a1-392b254a4729",
  "product_id": "prod_example",
  "question": "如何选择适合记录零散灵感的个人笔记工具？",
  "intent": "generic",
  "topic": "灵感记录",
  "language": "zh-CN",
  "status": "active"
}
```

2. 假设返回 question_id=42，cp_example 当前 revision=3，原有 q:7/supporting 关系。实际正文能回答新问题时，保留原标题、类型和关系：

```json
{
  "request_id": "a7325cde-82ea-4aa7-9347-a44a866d8c24",
  "product_id": "prod_example",
  "content_page_id": "cp_example",
  "revision": 3,
  "title": "个人灵感记录指南",
  "page_type": "pillar",
  "links": [
    {"question_key": "q:7", "role": "supporting"},
    {"question_key": "q:42", "role": "primary"}
  ]
}
```

3. **仅在后续明确要求 variants 时**，先核对主问题归属和已有变体，再提交同一意图的改写：

```json
{
  "request_id": "241b40fa-a850-4f44-aeb3-03eea8c6899b",
  "question_id": 42,
  "source": "agent",
  "variants": [{"question": "记下零散想法时，应该怎么挑选个人笔记软件？"}]
}
```

从返回的 variants 获取真实 ID，确有覆盖时才追加相应 v: 关系；主问题与已有关系继续保留。关联失败时问题可能已成功创建，恢复应读回并只补关联，不能重建问题。

## 重试与冲突

| 情况 | 处理 |
| --- | --- |
| 超时、断线、INTERNAL_ERROR，结果未知 | 同工具、原参数、原 request_id、同一 OAuth 客户端重试，最多 2 次；仍未知则记录并暂停该项 |
| IDEMPOTENCY_CONFLICT | 核对原始请求，不能换 UUID 掩盖不确定结果 |
| CONFLICT / REVISION_REQUIRED | 重读页面并比较，保留远端最新内容和关系；新的有意合并用新 UUID 与最新 revision，反复冲突则暂停 |
| 明确参数校验失败 | 确认旧请求未成功后修正，用新 UUID 提交 |
| PRODUCT_FORBIDDEN / 缺工具或 scope | 请求补充产品授权/权限；保留候选，不绕过 OAuth |
| NOT_FOUND / 归属不符 | 重新确认真实目标，不转写其他产品 |

去重按认证身份隔离，Codex 与 Claude Code 不一定共用身份；跨客户端不能凭相同 UUID 保证去重。恢复记录保存完整原参数、UUID、回执、实体 ID 与未完成项，不保存凭证，也不替代远端状态。
