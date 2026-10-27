# Plan: project-session-management（切换慢/删项目/删历史会话）

- 对应 intent：`docs/sdlc/project-session-management/intent.md`（2026-10-27，GATE 1 已通过）
- 定级：T2（每项一个完整功能点/bug 排查；spec 要点并入本文件前言）
- 基线：origin/dev `5f8ee0f5`，分支 `fix-project-session-management`

## Spec 要点（T2 简写，正式 spec 并入本文）

1. **切换目录**：切换后 UI 不得崩溃（`Stale read from <Show>` 必须消灭）；所选目录必须立即在
   chip/标题显示（乐观显示，数据后到再校正）；数据未就绪期间必须有可见 loading 态。
2. **project.remove**：`DELETE /api/project/:projectID`。语义 = 彻底删除：interrupt 该项目运行中
   会话 → 删 project 行（FK 级联清 worktree/session/message/inbox/instruction/blob_ref）→ 批量清
   各 session 的 durable 事件流（event/event_sequence）→ 释放孤儿 blob → 广播 `project.removed`
   （ephemeral）。`Project.global` 不可删。二次确认对话框必选。UI 入口：Settings 项目行菜单 +
   Home 项目卡片菜单。
3. **批量删会话**：`POST /api/session/removeProject { projectID }`（或 DELETE 语义，plan 定为
   `session.removeProject`）。逐会话复用现有 `Session.remove` 成熟路径（interrupt → Deleted 事件 →
   bus.remove）。UI：Settings 项目行菜单 "Delete history"（保留项目条目）+ Home archive stub 解除
   （改调 `session.update({archived:true})`）。

## 实施步骤

### 第 1 步：修复切换链路崩溃（问题 1a）

- `packages/app/src/new-session/location/dialog.tsx`（约 :516 `<Show when={props.onDrill}>`、:334-379
  `<Match>` children 的 `store.drilled!`）— 改 keyed Show / 消除非空断言，仿
  `session/route.tsx:248` 先例（注释说明原因）。
- `packages/app/src/new-session/view.tsx`（约 :110-127 `<Show when={props.project.selected()}>`）— 同上。
- `packages/app/src/home/projects/view.tsx`（约 :96-111）— 同上审查。
- `packages/app/src/session/route.tsx`（:145-178 `when={directory()}`）— 审查，必要时 keyed。
- 验证：快速连续切换目录 ≥10 次无 fatal；renderer 无 `Stale read` 崩溃。

### 第 2 步：切换乐观显示 + loading 可见（问题 1b）

- `packages/app/src/new-session/project/selector.tsx:68`：`selected()` 的回退从 `projects()[0]`
  改为优先反映 draft 目标目录（新目录未入库时显示"目标目录"临时 chip，不闪回旧项目）。
- `packages/app/src/composer/selection.ts`（:25-31 `loading()`）：确认 provider/agent 未就绪时
  composer 有 `Loader` 可见反馈（`@opencode/ui/loader`，遵守 app/AGENTS.md Loading Indicators），
  缺则补。
- i18n：仅英文源串（`packages/app/src/i18n` 现有机制）。

### 第 3 步：project.remove 全链路（问题 2）

顺序（依赖方向 Schema→Core/Protocol→Server→Client）：
1. `packages/schema/src/project.ts`：新增 `Removed = ephemeral({ type: "project.removed",
   schema: { projectID: ID } })`，`Event.Definitions = inventory(Updated, Removed)`；确认
   event manifest 聚合无需额外改动（Definitions 已被 inventory 收集）。
2. `packages/core/src/project.ts`：`Interface.remove(projectID)`：
   - 查行存在，`ID.global` 拒绝（NotFoundError/InvalidRequest）；
   - `db.select(session id).from(SessionTable).where(project_id = …)` 收集 sessionIDs；
   - 逐个 `execution.interrupt + awaitIdle`（复用 Session.remove 前置模式）；
   - 事务：删 `ProjectTable` 行（FK cascade：worktree/project_directory/session_v2/
     session_message/session_inbox/instruction_entry/instruction_state/blob_ref）；
   - 按 sessionIDs 批量删 `event`/`event_sequence`（参照 `bus.ts:706-718` 模式，分批 500）；
   - blob 孤儿清理：对每个 sessionID 调 `SessionBlob.releaseSession`（`blob.ts:429`，事务内），
     或删行后按 notExists 批量清（plan 执行时按事务边界选前者，与 session.remove 投影一致）；
   - `bus.publish(ProjectSchema.Event.Removed, { projectID })`。
3. `packages/protocol/src/groups/project.ts`：`HttpApiEndpoint.delete("project.remove",
   "/api/project/:projectID", { params, success: HttpApiSchema.NoContent,
   error: ProjectNotFoundError })`。
4. `packages/server/src/handlers/project.ts`：handler（mapError → ProjectNotFoundError）。
5. `cd packages/client && bun run generate`（不手改生成文件）。
6. 客户端事件消费：
   - `packages/client/src/solid/data.ts`（:704 `project.updated` 旁）：`project.removed` →
     从 `store.project.info` 删除该 id + invalidate 相关查询。
   - `packages/app/src/runtime/server/sync.tsx`（:240-257 `applyProjectUpdate` 旁）：事件处理移除
     服务端项目列表条目 + 清 `registry.tsx` 的 usedProjectIDs。
7. UI 入口（i18n 英文源串 + danger 确认框，复用 Home 的 DeleteDialog 模式）：
   - `packages/app/src/settings/workspaces/project-options.tsx` + `project-row.tsx`：菜单加
     "Delete project…"（描述含"删除其全部会话历史"）。
   - `packages/app/src/home/projects/view.tsx`（:659-662 菜单）：加 "Delete project…"。
   - 删除后若当前 draft/tabs 停在该项目：关闭相关 tab（参照 `notifySessionTabsRemoved` 模式）。
8. 测试：`packages/core/test/`（新增 project-remove 用例：级联删 session/事件流/blob、
   global 拒绝、404；从 packages/core 目录跑）。

### 第 4 步：批量删会话 + archive stub 解除（问题 3）

1. `packages/protocol/src/groups/session.ts`：`HttpApiEndpoint.post("session.removeProject",
   "/api/session/removeProject", { payload: { projectID }, success: HttpApiSchema.NoContent,
   error: ProjectNotFoundError })`。
2. `packages/core/src/session.ts`：`removeProject(projectID)`：按 project 列出全部 sessionID →
   逐个复用 `result.remove`（concurrency 1，天然 interrupt + 级联 + 事件流清理）。空列表 no-op。
3. `packages/server/src/handlers/session.ts`：handler。
4. `bun run generate`（与第 3 步可合并执行一次）。
5. UI：
   - `packages/app/src/settings/workspaces/project-options.tsx`：菜单加 "Delete history…"（保留项目）。
   - `packages/app/src/home/sessions/controller.tsx`（:362-363）：archive stub 改调
     `ctx.sdk.api.session.update({ sessionID, archived: true })`（协议已有，仅前端接线）。
6. 测试：`packages/core/test/session-remove.test.ts` 扩展 removeProject 用例（多会话+子会话级联）。

### 第 5 步：全量验证

- 根目录 `bun run check`（lint + typecheck 全量）。
- `cd packages/core && bun test project session-remove`（测试不得从仓库根跑）。
- `cd packages/app && bun test`（涉及组件的既有测试回归）。
- 手动验证（app/AGENTS.md Local Dev 流程）：`bun dev serve --port 4096` + app dev :4444，
  覆盖：连续切目录无崩溃、新目录 chip 乐观显示、Settings/Home 删项目（含死目录项目）、
  删历史、Home archive。
- 独立复核：task 派 general subagent 审查 diff 与 intent/spec 偏差。

## 改动文件清单（汇总）

| 层 | 文件 | 改什么 |
|---|---|---|
| app | new-session/location/dialog.tsx | keyed Show 修复 |
| app | new-session/view.tsx | keyed Show 修复 |
| app | home/projects/view.tsx | keyed 审查 + 删项目菜单项 |
| app | session/route.tsx | keyed 审查 |
| app | new-session/project/selector.tsx | selected() 乐观显示 |
| app | composer/selection.ts | loading 可见性（按需） |
| schema | src/project.ts | Removed 事件 |
| protocol | src/groups/project.ts | remove 端点 |
| protocol | src/groups/session.ts | removeProject 端点 |
| core | src/project.ts | remove 实现 |
| core | src/session.ts | removeProject 实现 |
| server | src/handlers/project.ts、session.ts | 两个 handler |
| client | generated（bun run generate） | SDK 重生成 |
| client | src/solid/data.ts | project.removed 消费 |
| app | runtime/server/sync.tsx、registry.tsx | 事件消费 + used 清理 |
| app | settings/workspaces/project-options.tsx、project-row.tsx | 删项目/删历史菜单+确认 |
| app | home/sessions/controller.tsx | archive stub 解除 + 确认框 |
| i18n | 对应语言源文件 | 英文源串 |
| test | packages/core/test/* | remove/removeProject 用例 |

## 验证方式

见第 5 步。证据要求：`bun run check` 输出、core/app 测试输出、generate diff 摘要、
独立 subagent 复核结论。

## 风险与回滚

- **破坏性删除**：project.remove 不可逆。缓解：危险按钮二次确认 + 对话框明示"包含全部会话历史"。
  回滚 = revert 提交（DB 行已删无法恢复，属功能语义本身，不额外做软删）。
- **FK cascade 依赖 schema.gen 一致性**：测试覆盖级联路径；SQLite 需确认 foreign_keys=ON
  （Database 层已启用，测试会暴露）。
- **keyed Show 行为变化**：逐个审查 narrowed accessor 用法，防止 keyed 后 children 拿到的值快照
  语义变化引发新 bug；每个改动点跑对应组件测试/手动验证。
- **Protocol 变更**：generate 忘跑会类型不匹配 —— 提交前检查 generated diff；CI 会兜底。
- **事件消费遗漏**：TUI 与 App 共用 data.ts，`project.removed` 在 data.ts 一处处理两端生效；
  需在 TUI（`bun run dev:live` 或既有 TUI 测试路径）抽查项目列表即时移除。

## 未决问题的裁决（回填 intent）

1. 修复深度：本轮做 A 崩溃修复 + B 乐观显示/loading（保守高价值）；4 槽请求队列与插件门超时
   不动（stability 优先），留待实测数据决定。
2. 批量删除：走服务端 `session.removeProject`（大历史量一次事务路径复用，避免 N 次 HTTP）。
3. 删除档位：提供"删项目（含历史）"与"仅删历史"两档；"归档"保留现有自动 sweep 语义，不加手动归档入口。
