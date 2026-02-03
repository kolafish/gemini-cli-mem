# Context 组织与管理（core）

本文梳理 `packages/core` 中上下文（context）的组织方式、来源与加载流程，便于二次开发时快速定位改动点。

## 1. 主要组件
- `ContextManager`：集中加载和缓存上下文内容，提供 `refresh()` 和 JIT 加载能力。`packages/core/src/services/contextManager.ts`
- `memoryDiscovery`：负责发现并读取 `GEMINI.md` 族文件（含导入），并拼接成可注入的指令块。`packages/core/src/utils/memoryDiscovery.ts`
- `WorkspaceContext`：管理工作区根目录集合，并提供路径是否在工作区内的校验。`packages/core/src/utils/workspaceContext.ts`
- `environmentContext`：生成“运行环境上下文”并构造初始历史。`packages/core/src/utils/environmentContext.ts`
- `chatCompressionService`：在历史过长时压缩上下文，截断大体量工具输出。`packages/core/src/services/chatCompressionService.ts`

## 2. Context 的来源（按层次）
1. **全局记忆**：用户主目录 `~/.gemini/` 下的 `GEMINI.md`（支持多个文件名变体）。
2. **工作区记忆**：从工作区根向上查找 `GEMINI.md`，并在工作区内向下 BFS 搜索子目录；受“信任文件夹”开关与过滤规则影响。
3. **扩展记忆**：启用的扩展可提供额外 `contextFiles`，与工作区记忆一起合并。
4. **环境上下文**：日期、系统、临时目录、文件夹结构等环境信息由 `environmentContext` 生成。
5. **JIT（按需）记忆**：访问特定路径时，按需向上加载该路径到最近“可信根目录”的 `GEMINI.md`，避免重复加载。

> `memoryDiscovery` 会读取文件内容并处理导入（见 `docs/core/memport.md`），最终用 `--- Context from: path ---` 包裹各段内容后拼接。

## 3. 加载流程（简化）
1. **启动/刷新**：`ContextManager.refresh()` 会清空已加载路径，然后加载全局记忆与环境记忆，并触发 `CoreEvent.MemoryChanged`。
2. **环境记忆拼接**：环境记忆会额外拼接 MCP 指令（如果启用 MCP）。
3. **初始历史**：`getInitialChatHistory()` 将环境上下文作为首条“用户消息”注入对话历史。
4. **按需加载**：当工具或流程需要时，可调用 `ContextManager.discoverContext()` 进行 JIT 记忆补充。

## 4. 工作区与信任边界
`WorkspaceContext` 负责维护一个或多个工作区根目录：
- 新增/更新目录时会触发回调，便于其他模块刷新上下文。
- `isPathWithinWorkspace()` 用于限制工具访问与上下文发现的范围。
- 目录解析会做真实路径校验，避免路径欺骗。

## 5. 历史压缩与上下文窗口
当对话历史接近模型 token 上限时，`chatCompressionService` 会：
- 选择可压缩的历史分割点。
- 优先保留最近轮次。
- 对超大工具输出进行截断并保存到临时文件。

这一步不会改变 `GEMINI.md` 记忆内容，但会影响“当前对话上下文”对模型的可见性。

## 6. 常见扩展点
- **调整 `GEMINI.md` 搜索范围**：`memoryDiscovery` 的上/下行查找策略与过滤规则。
- **增加新的上下文来源**：在 `ContextManager` 中拼接新的指令块，或扩展 `environmentContext`。
- **JIT 策略优化**：根据访问路径、工具类型或安全策略调整触发条件。
