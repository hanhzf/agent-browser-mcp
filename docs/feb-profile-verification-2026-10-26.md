# FEB Profile 前后端联调记录

日期：2026-10-26  
目标：验证 `Chrome Agent Bridge` 扩展与 Agent Browser MCP 服务能否在 FEB Chrome Profile 中复用 `linux.do` 登录态，并在不抢占前台、不接触用户标签页的条件下完成页面读取和后台操作。

## 验证环境

- 个人仓库：`git@github.com:hanhzf/agent-browser-mcp.git`
- 上游仓库：`https://github.com/cmsflash/agent-browser-mcp.git`
- 基线提交：`c9d2907544c5a04316a47c06fb3ad2c3ca7556bc`
- 扩展：Chrome Agent Bridge `1.2.0`
- MCP 服务：`server/index.mjs`，运行时版本 `1.2.1`
- Chrome Profile：`Profile 3` / `feb.hanhzf` / `feb.hanhzf@gmail.com`
- linux.do 用户：`frankwell`

## 结果

### 隔离环境测试

| 测试 | 结果 | 通过数 |
| --- | --- | ---: |
| `test/e2e.mjs` | 通过 | 126 |
| `test/thread-isolation.mjs` | 通过 | 26 |
| `test/no-blank-tab.mjs` | 通过 | 18 |
| `test/window-reuse.mjs` | 通过 | 23 |
| `test/freeze-recovery.mjs` | 通过 | 1 组场景 |
| `test/background-guarantee.mjs` | 通过 | 1 组场景 |

后台保证测试的 Chrome 前台采样为 `0`，Agent 标签页处于活动状态的次数为 `0`，原有用户标签页在测试后仍存在。

### FEB 真实 Profile 测试

| 测试 | 结果 | 关键覆盖 |
| --- | --- | --- |
| `test/smoke-real.mjs` | 通过 | Profile 选择、首次连接、后台标签页、页面读取、截图、可信点击、过期 ref 拒绝、主世界 JavaScript、清理 |
| `test/reconnect-real.mjs` | 通过 | 扩展断线重连 |
| `test/thread-isolation-real.mjs` | 通过，25 项 | 两个线程的标签组隔离与权限边界 |
| `test/tools-real.mjs` | 通过 | 页面读取、输入与诊断工具 |
| `test/frontmost-real.mjs` | 通过 | 9 个动作、4 次前台采样，Chrome 抢占前台为 0 |

`test/smoke-real.mjs` 原来假设 example.com 的正文始终包含 `Example Domain`。该站正文已改为多语言版本，导致测试误报。断言已改为同时检查稳定的页面标题 `Example Domain` 和链接文本 `Learn more`。

### linux.do 只读小样

在 `feb.hanhzf@gmail.com` Profile 中创建一个独立任务标签组，打开 `https://linux.do/latest` 并执行：

1. 读取页面标题和正文，确认页面是 `LINUX DO - 新的理想型社区`；
2. 从页面内容确认当前用户为 `frankwell`；
3. 确认话题列表可读；
4. 在后台标签页滚动到 `scrollY=480`；
5. 在滚动前后检查任务标签页均为 `active=false`；
6. 保存后台截图并删除本次任务标签组。

证据截图：[linuxdo-feb-readonly-poc.jpg](evidence/linuxdo-feb-readonly-poc.jpg)

## 当前结论

Agent Browser MCP 的前端扩展与后端服务已经能够：

- 识别并显式选择 FEB Profile；
- 复用该 Profile 已有的 linux.do 登录态；
- 将每个任务限制在以 `threadTitle` 标识的独立 Chrome 标签组；
- 在背景标签页执行读取、截图、滚动和交互，不把 Chrome 拉到前台；
- 在标签页被关闭、扩展重连和浏览器重启后恢复任务身份；
- 清理任务标签组，同时保留用户自己的标签页。

现有 `Linux do 自动刷帖子/reader.py` 仍通过独立的 Agent Browser CLI/CDP 连接运行，尚未迁移到本仓库的 MCP 接口。因此，本次完成的是通用浏览器桥接能力与真实登录态验证，不代表旧调度器已经切换，也不代表 Connect 的浏览统计一定入账。

## 后续迁移边界

迁移时保留旧调度器已有的业务策略：08:15–22:00 时间窗、每批 5–10 个话题、批次 240–360 秒、滚动停顿 8–12 秒、指标查询间隔 1–2 小时、账号核验、日志和访问限制时停止。只替换浏览器驱动层：由 Agent Browser CLI/CDP 调用改为带固定 `threadTitle` 和显式 FEB `deviceId` 的 MCP 工具调用。

JAN Profile 只保留框架支持；本次没有启用或访问 JAN 账号。
