# Agent Browser MCP 与 Linux.do 自动化工作流

![Agent Browser MCP 与 Linux.do 自动化架构](agent-browser-mcp-linux-do.svg)

这张图描述 FEB Chrome Profile 上已经验证的浏览器桥接链路，以及现有 Linux.do 调度器迁移到 MCP 时的边界。实线是当前 Agent Browser MCP 已实现并通过真实 Profile 测试的调用链；虚线是保留业务策略、后续替换驱动接口的迁移部分。

## 端到端流程

1. Linux.do 调度与站点适配器按照现有规则决定何时运行、选择哪些话题，以及何时查询 Connect 指标。
2. 迁移后，适配器向 Agent Browser MCP 发出工具调用，并在同一任务中持续使用固定 `threadTitle`。
3. MCP 服务维护该线程的当前标签页和已选择 Profile，通过本机 WebSocket Hub 将命令路由到固定的 FEB `deviceId`。
4. FEB Profile 中的 Chrome Agent Bridge 扩展把 `threadTitle` 解析为唯一任务标签组，只允许命令访问该组的标签页。
5. 扩展通过 Chrome API、CDP 和页面脚本在后台读取、截图、滚动或交互；任务标签页保持 `active=false`。
6. 页面结果返回调度器；调度器继续负责事件日志、指标缓存、随机批次和运行时间窗。

## 边界

| 边界 | 负责内容 | 不负责内容 |
| --- | --- | --- |
| Linux.do 调度与适配器 | 时间窗、候选话题、随机节奏、账号与站点规则、日志、低频指标查询 | Chrome Profile 选择、标签隔离、页面动作实现 |
| Agent Browser MCP | MCP 工具协议、线程会话状态、显式 Profile 选择、Hub/Relay 路由 | 具体网站的访问策略与业务判断 |
| Chrome Agent Bridge | Profile 身份、任务标签组、Chrome API/CDP 动作、重连与标签组恢复 | 调度计划和 Linux.do 业务规则 |
| 任务专属标签组 | 当前任务可读写的页面集合 | 用户标签页、其他任务的标签组 |

## 身份与隔离

- **Profile 边界**：扩展发送持久 `deviceId` 和 Profile 邮箱；多个 Profile 同时连接时，Hub 不根据连接顺序猜测，调用方必须显式选择。
- **线程边界**：每个工具调用携带 `threadTitle`；扩展将它映射到一个标签组，并在分派前覆盖调用方提供的分组信息。
- **用户标签边界**：不属于任务组的标签页不会出现在任务可见列表中，也不能作为动作目标。
- **恢复边界**：扩展重连或 Chrome 重启后，标签组注册表按任务身份重新绑定；关闭标签组时先解除分组再关闭标签，避免留下同步的保存组。

## 当前状态

2026-10-26 的真实 FEB Profile 验证已经覆盖 Profile 选择、linux.do 登录态、后台读取、滚动、截图、线程隔离、扩展重连、冻结页面恢复、任务清理和不抢占前台。完整结果见 [FEB Profile 前后端联调记录](feb-profile-verification-2026-10-26.md)。

现有 `Linux do 自动刷帖子/reader.py` 仍使用 Agent Browser CLI/CDP。本图把它到 MCP 的连接标为待迁移，因此不能据此认为旧后台进程已经切换，也不能把页面访问等同于 Connect 指标已经入账。

可交互版本：[agent-browser-mcp-linux-do.html](agent-browser-mcp-linux-do.html)。该版本可以查看节点来源、追踪主路径、切换主题并重新导出 SVG。
