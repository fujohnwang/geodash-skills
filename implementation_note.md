# 实施说明

- 两个 skill 统一维护在本项目的 skills/ 下，可分别安装；不包含 GeoDash 服务源码、凭证或业务数据。未新增服务端接口，默认行为和写入边界保持不变。
- 为保证单独安装，每个 skill 自带一份 mcp-setup.md；连接方式调整时需同步两份。所有相对文档引用均留在各自安装包内。
- GitHub 仓库尚未创建或推送，安装示例保留 YOUR_GITHUB_OWNER 占位符；无需发布 npm 包。这里只初始化独立 Git 仓库，不创建提交。
- 验证使用 Skills CLI 1.7.2：发现两个 skill；在四个隔离项目验证仅内容、仅问题、双 skill、双 skill copy 模式，Codex/Claude Code 入口、文件逐项一致和相对链接完整性均通过。官方 skill 格式校验通过；未调用业务 MCP 或写入产品数据。
- 正式路径验证：GeoDash 的四个本机入口已切到同级项目，Codex skills/list 和 Claude Code 初始化均发现两个 skill；32 个相对文档链接有效。原始文件备份位于 /tmp/geodash-skills-original-20261018-7i8i84ms。
