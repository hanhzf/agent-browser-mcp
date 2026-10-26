# Agent Browser MCP 与 Linux.do 自动化工作流

![Agent Browser MCP 与 Linux.do 自动化架构](agent-browser-mcp-linux-do.svg)

这张图只描述最终运行状态。WebSocket Hub 是 Node.js MCP Server 的内部模块，不作为独立服务部署。

## 端到端流程

1. Python `reader.py` 按时间窗和随机规则选择话题，并发出 MCP 工具调用。
2. Node.js Agent Browser MCP Server 维护线程和 Profile 状态；内置 WebSocket Hub 把命令发送给 FEB 扩展实例。
3. Chrome Agent Bridge 把 `threadTitle` 映射为唯一任务标签组，只允许命令访问该组标签页。
4. 扩展在后台完成页面读取、截图、滚动或交互，并将结果原路返回给 `reader.py`。
5. `reader.py` 保存事件日志和指标缓存，并继续下一次调度。

## 边界

| 边界 | 负责内容 | 不负责内容 |
| --- | --- | --- |
| Python `reader.py` | 时间窗、候选话题、随机节奏、账号与站点规则、日志、低频指标查询 | Chrome Profile 选择、标签隔离、页面动作实现 |
| Node.js MCP Server | MCP 工具协议、线程会话状态、显式 Profile 选择、内置 WebSocket Hub | 具体网站的访问策略与业务判断 |
| Chrome Agent Bridge | Profile 身份、任务标签组、Chrome API/CDP 动作、重连与标签组恢复 | 调度计划和 Linux.do 业务规则 |
| 任务专属标签组 | 当前任务可读写的页面集合 | 用户标签页、其他任务的标签组 |

## 身份与隔离

- **Profile 边界**：扩展发送持久 `deviceId` 和 Profile 邮箱；多个 Profile 同时连接时，Hub 不根据连接顺序猜测，调用方必须显式选择。
- **线程边界**：每个工具调用携带 `threadTitle`；扩展将它映射到一个标签组，并在分派前覆盖调用方提供的分组信息。
- **用户标签边界**：不属于任务组的标签页不会出现在任务可见列表中，也不能作为动作目标。
- **恢复边界**：扩展重连或 Chrome 重启后，标签组注册表按任务身份重新绑定；关闭标签组时先解除分组再关闭标签，避免留下同步的保存组。

## 语言与部署结论

- Chrome Agent Bridge 保持 JavaScript，因为它直接使用 Chrome 扩展 API。
- Agent Browser MCP Server 继续使用现成的 Node.js 实现；WebSocket Hub 已包含在其中。
- Linux.do 业务调度保持 Python；只需把浏览器驱动调用收敛到 MCP 接口。
- Rust 可以重写整个本机后端，但需要同时重写成熟的 MCP Server 和 Python 调度逻辑，目前没有性能或部署收益支撑这项成本。

最终只需要维护三个本机组件：Python 调度器、Node.js MCP Server、Chrome 扩展。没有独立的 WebSocket Hub 部署项。

这张图表示最终架构；当前验证进度见 [FEB Profile 前后端联调记录](feb-profile-verification-2026-10-26.md)。

可交互版本：[agent-browser-mcp-linux-do.html](agent-browser-mcp-linux-do.html)。该版本可以查看节点来源、追踪主路径、切换主题并重新导出 SVG。
