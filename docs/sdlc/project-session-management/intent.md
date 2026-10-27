# Intent: project-session-management（项目/会话管理三连修复）

- 提出人 / 日期：xcx1993 / 2026-10-27
- 基线：origin/dev `5f8ee0f5`（v2.1.4），分支 `fix-project-session-management`
- 用户实际运行版本：OpenCode Desktop 1.19.4（win32，日志 `opencode-debug-20261027T061201.zip`）

## 问题 1：切换工作目录很久才生效（bug 排查）

**现象**：桌面版切换 project/目录后，UI 明明已操作却长时间显示旧状态；期间 renderer 反复报
`ResizeObserver loop completed with undelivered notifications`（14:02:25 ~ 14:09:53 多次），
最终 14:11:38 fatal renderer error：SolidJS `Stale read from <Show>`（堆栈 refetch → runUserEffects → completeUpdates）。

**调查结论**（代码级证据，详见探索报告）：
- 首要嫌疑：新目录 per-directory instance 首次 boot + 插件激活门。
  `packages/server/src/location.ts:36-47` 对 provider/model/agent 端点设 5s 超时
  （`awaitPlugins` → `Plugin.awaitActivation`）；客户端 `packages/app/src/providers/catalog/providers.ts:18-21`
  `syncWhileBooting` 以 1~5s 间隔重试至多 12 次 → 最坏几十秒"切了不动"。
- 放大器：客户端全局仅 4 并发槽（`packages/app/src/runtime/server/request-queue.ts:6-16`），
  切换触发 1+12 路定位同步 + bootstrap 全量 refetch + worktree 慢通道（git 扫描占 2 槽）互相排队。
- 显示层：项目 chip 的 `selected()` 在数据未到位时回退 `projects()[0]`（`packages/app/src/new-session/selector.tsx:68`），
  造成"已切换但显示未变"。
- 崩溃：切换引发 keyed remount + 事件风暴重排，某处非 keyed `<Show>`/`<Match>` 的 narrowed
  accessor 在条件翻转后被调用 → renderer 崩溃（代码库已在 `session/route.tsx:248` 修过同类问题，注释为证）。
- ResizeObserver 告警是 remount/重排的症状，非独立根因。

**目标**：切换目录后 UI 快速（秒级）进入 loading 态并尽快完成数据就绪；彻底消除切换路径上的
`Stale read from <Show>` 崩溃。

## 问题 2：无法删除已不存在/已迁移的项目

**现象**：项目列表残留死条目，用户无任何清理手段（设置页/Home 菜单都只有 hide/编辑，无删除）。

**调查结论**：
- 项目列表 = SQLite `project` 表持久化行（`packages/core/src/project.ts:238-247`），非目录扫描。
- dev 已有"死目录自动归档" sweep（`packages/core/src/project.ts:443-487`，每小时，
  需 1h 无活动；1.19.4 无此功能）。归档 ≠ 删除，行永久留存。
- **`project.remove` 在 schema/protocol/core/server/client/UI 六层全部缺失**（全仓零命中）。
- 非 git 项目目录迁移 = 新 ID，旧条目只能等 sweep 归档——正是痛点核心。

**目标**：提供显式删除项目的能力（含其全部会话历史）：Settings 项目菜单 + Home 项目菜单
入口，服务端事务删除 + 事件广播，所有客户端即时移除条目。

## 问题 3：历史会话记录无法删除

**现象**：用户无法删除历史会话。

**调查结论**：
- dev（2.1.4）单会话删除已全链路可用：TUI（2026-07-06 `ctrl+d`）、App Home（2026-08-24 菜单+确认框）、
  CLI `opencode session delete`（2026-09-07）。用户 1.19.4 部分缺这些。
- 真正缺失：**批量删除**（清空某项目/全部历史）；Home 归档按钮是 stub
  （`home/sessions/controller.tsx:362-363` TODO）。

**目标**：补齐批量删除能力（至少按项目清空；可含"全部历史"），并解除 Home 归档 stub。

## 范围内 / 范围外

**范围内**：
1. 目录切换响应链路的延迟与崩溃修复（app + 必要时 server 侧配合）。
2. `project.remove` 全链路（schema→protocol→core→server→client generate→UI）+ 删除入口 UI。
3. 会话批量删除（服务端批量端点或前端循环，plan 阶段定）+ Home 归档 stub 解除。

**范围外**：
- TUI 专属的项目切换器（TUI 无此链路）。
- 数据保留期（retention）策略。
- 1.19.4 的回移植；只在 dev 上修复。

## 约束

- 遵守仓库 AGENTS.md：Protocol/HttpApi 变更后必须 `bun run generate`（packages/client），禁止手改生成文件；
  运行时依赖方向 Schema→Core/Protocol→Server；测试不得从仓库根目录跑；`bun run check` 为全量校验。
- session-ui 包内不得硬编码英文文案（i18n key）。
- 删除项目属破坏性操作：必须二次确认；删除语义 = 项目行 + 级联 worktree/session（FK 已 cascade）
  + 显式清理 event/event_sequence 流与 blob 孤儿 + interrupt 运行中会话。

## 未决问题（讨论结论回填）

1. 问题 1 修复深度：只修崩溃 + loading 态体验（保守），还是连 4 槽队列/插件门超时一起动（激进）？
   → 待 plan 阶段给出分层方案由人选择。
2. 批量删除走前端循环还是服务端批量端点？→ 待 plan 阶段定。
3. project.remove 是否需要保留"仅归档（可恢复）"与"彻底删除"两档？→ 待讨论。
