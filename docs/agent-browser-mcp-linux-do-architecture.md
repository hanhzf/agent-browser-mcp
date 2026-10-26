# 基于 Chrome 的网站自动化框架

![基于 Chrome 的网站自动化架构](agent-browser-mcp-linux-do.svg)

这张图描述最终运行状态。系统只有三个主要运行主体；新增网站时增加业务 Adapter / Job，不复制 MCP Server 或 Chrome 扩展。

## 三个运行主体

| 主体 | 负责内容 | 复用范围 |
| --- | --- | --- |
| 业务调度程序 | 理解网站规则，决定何时执行、操作哪个对象、如何校验结果，并保存业务状态与日志 | 一套共享调度框架；每个网站增加一个 Adapter / Job |
| Agent Browser MCP Server | 提供通用浏览器工具、线程会话、Profile 路由；WebSocket Hub 是它的内部模块 | 所有网站和 Chrome Profile 共用 |
| Chrome Agent Bridge | 运行在指定 Chrome Profile 中，接收命令，并在任务专属标签组内执行页面动作 | 所有网站共用；每个 Profile 安装一份扩展实例 |

任务专属标签组是扩展在 Chrome 中创建的隔离工作区，不是第四个独立服务。

## 新增网站

以 Linux.do、Expense Boss 和后续网站为例，新增接入只需要：

1. 在业务调度程序中增加该网站的 Adapter / Job，定义页面规则、账号校验、执行节奏和业务结果。
2. Adapter 调用同一套 MCP 工具，不直接连接 Chrome。
3. MCP Server 通过内置 WebSocket Hub，把命令路由到指定 Profile 的扩展实例。
4. 扩展在对应任务标签组中完成点击、填写、滚动、读取或截图。

一个调度进程可以承载多个网站。只有账号生命周期、故障隔离或发布节奏需要独立管理时，才把某个 Adapter 单独启动为进程。

## 边界

- **业务层**：网站知识、调度策略、随机节奏、账号校验、日志和指标。
- **浏览器桥接层**：MCP 协议、Profile 路由、WebSocket 连接和通用浏览器动作。
- **浏览器执行层**：Chrome Profile 身份、任务标签组、页面动作和重连恢复。
- **隔离规则**：Profile 由 `deviceId` 固定，任务工作区由 `threadTitle` 固定；任务不能访问用户标签或其他任务组。

Linux.do 是第一个业务 Adapter，也是当前已完成真实 FEB Profile 联调的验证场景。验证证据见 [FEB Profile 前后端联调记录](feb-profile-verification-2026-10-26.md)。

可交互版本：[agent-browser-mcp-linux-do.html](agent-browser-mcp-linux-do.html)。它支持查看源码来源、聚焦调用链、切换主题和重新导出 SVG。
