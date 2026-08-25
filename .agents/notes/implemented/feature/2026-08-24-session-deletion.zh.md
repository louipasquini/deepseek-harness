# Agent Note: 永久删除会话与“归档会话”视图

Status: implemented

[English](2026-08-24-session-deletion.md) | 中文

## Problem

GUI 只能隐藏会话（归档），却无法真正移除：归档会保留会话日志和工作区记账槽位，而且没有任何界面能列出已归档的会话或将其恢复。删除会话（包括持久化日志）在整个技术栈中都不存在：`SessionPersistence` 严格只追加，工作区注册表没有任何破坏性会话操作，Host 没有删除 RPC，浏览器侧边栏也没有归档会话视图。用户积累的归档会话既无法回收历史，也无法释放其占用的磁盘空间。

## Decision

### Host：持久化删除是显式的管理操作

`SessionPersistence.delete(id)` 是只追加契约唯一被允许的例外，且只由工作区注册表的 `deleteSession` 调用，绝不进入写路径。未知 id 返回 `false`，移除持久化产物时返回 `true`；已创建但从未追加的惰性 id 会清空协调器状态，使该 id 可以被重新创建。协调器拒绝仍绑定在存活 SessionStore 上的身份，等待该身份的进行中 retirement（`session/disposed` 的最终 drain）收敛，丢弃按会话维护的状态和保留的 preparation，然后调用后端新增的 `deleteStored` 钩子。后端实现物理移除：JSONL 删除日志文件及其会话目录（尽力而为的清理，删除可重试），SQLite 删除会话行及其级联的事件行。

`ctx.workspaceRegistry.deleteSession(id)` 从每个工作区记录中移除该会话的记账槽位，将其从归档集合中移除，并调用 `SessionPersistence.delete`。仍绑定在存活 SessionStore 上的会话会以 `WorkspaceLiveSessionError` 拒绝——调用方须先停止存活 agent。被删除的身份同时会离开注册表的 header 索引，防止之后对同一 id 的归档针对一个幽灵成功。`ctx.workspaceRegistry.unarchiveSession(id)` 是归档集合的逆操作：将会话恢复到所有分组表面，对集合外的 id 幂等。

Host 网关（`workspace.deleteSession` RPC）在注册表执行前，通过新增的 `AgentRegistry.stop(sessionId)` 停止存活 agent。`stop` 是持有方 `AgentHandle` 的管理性对应物：注册表保留 `create`/`resume` 返回的每个 handle 的拆除能力（按会话 id 键控，detach 时丢弃），使显式的身份生命周期无需知道创建方消费者即可停止 agent。停止会触发 `session/disposed`，排空持久化写缓冲并通过既有的 `host/session-removed` 帧广播；域写入通过既有的 `domain/changed` 投影发出 `host/workspace-changed` 和 `host/archived-sessions-changed`，因此所有已连接的标签页无需任何新帧类型即可收敛。

### 客户端与 GUI

`ctx.workspaces` 新增 `unarchiveSession` 和 `deleteSession`；工作区列表状态本就有 `archivedSessionIds`，sessions manager 也已在 `host/session-removed` 时移除会话，因此没有新增客户端状态。侧边栏视图菜单新增第三种模式 **归档会话**（`groupBy: 'workspace' | 'flat' | 'archived'`，与同一查看 store 一起持久化）：归档会话按所属工作区（或“未分组”）分组渲染，始终展开，并隐藏排序控件（归档顺序即 Host 追加顺序）。每个归档行的菜单包含 **恢复会话**（与归档相同、无需对话框的集合翻转）和 **删除会话**（危险操作）。删除会打开浏览器自持的确认对话框——与工作区删除相同的模式——并保持打开，直到 sessions 投影已提交移除（`host/session-removed`），避免陈旧帧泄漏到下一次操作。

## Alternatives considered

**通过归档集合中的持久化墓碑做软删除。** 已拒绝：持久留在工作区域中的墓碑仍会把会话日志留在磁盘上，而需求正是擦除历史并回收空间。归档集合仍只是显示层；删除是工作区注册表的破坏性身份操作，也是持久化服务的显式移除。

**由 sessions 域拥有一个新的 `session.delete` RPC。** 已拒绝：删除同时涉及工作区记账、归档成员资格和持久化日志，这正是工作区域的身份表面——也是归档放在那里的原因。新增第二个网关方法会把一次用户操作拆到两个域。

**通过新建的 loop 服务停止存活 agent。** 已拒绝，改用 `AgentRegistry.stop`：网关本就依赖 `ctx.agents`，注册表无论如何都会收到每个 `create`/`resume` handle，而 loop 服务会为网关增加一条新依赖边，却不带来额外覆盖（配置声明的 agent 重启后由 loop 重新创建，也不是 GUI 可删除的行）。

**拒绝删除存活会话。** 已拒绝：用户正在查看的会话恰恰是最可能被删除的；网关先停止 agent，注册表与持久化的守卫作为纵深防御保留。

## Consequences

**GUI 现在可以永久销毁历史。** 删除会不可逆地擦除持久化日志；确认对话框明确说明这一点，并保持打开直到已提交投影落地。没有撤销，被删除的会话无法恢复，也无法再次归档。

**SessionPersistence 不再严格只追加。** Service Definition 和两个第一方后端都把 `delete`/`deleteStored` 文档化为显式的管理例外；第三方后端必须实现这个新增的必需钩子，才能继续通过 `PersistenceBackend` 接口编译。

**删除存活会话会在所有标签页收敛。** 一个 RPC 停止 agent（排空其写缓冲）、移除记账与归档成员资格、擦除日志，并复用既有的 `session-removed`/`workspace-changed`/`archived-sessions-changed` 帧——没有新线帧，没有新客户端状态机。

**归档会话现在是一等表面。** 归档视图展示 Archive 隐藏的内容，Restore 反转 Archive，Delete 是终态步骤——补全了 GUI 此前止步于“仅隐藏”的归档生命周期。
