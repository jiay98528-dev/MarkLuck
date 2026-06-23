# MarkLuck 发布打磨执行日志

> 创建日期：2026-06-23
> 关联文档：`spec/release-hardening-deepseek-plan.md`

---

## M-R0：基线冻结与执行环境确认

- **执行时间**: 2026-06-23 15:50 - 15:58
- **修改文件**: 无（仅创建本日志文件 `memory/release-hardening-execution-log.md`）
- **必需文档已阅读**: AGENTS.md ✅ | memory/bug_log.md ✅ | spec/release-hardening-deepseek-plan.md ✅

### 执行命令和结果

| #   | 命令                                                          |  结果   | 摘要                                   |
| --- | ------------------------------------------------------------- | :-----: | -------------------------------------- |
| 1   | `git status --short --branch`                                 |   ✅    | main 分支，领先 origin/main 27 commits |
| 2   | `git log --oneline -10`                                       |   ✅    | 最新 commit: `7ea193e chore: simplify` |
| 3   | `pnpm.cmd --filter @markluck/app typecheck`                   | ✅ PASS | vue-tsc --noEmit 零错误                |
| 4   | `pnpm.cmd exec eslint packages/app/src packages/renderer/src` | ✅ PASS | 零警告                                 |
| 5   | `pnpm.cmd --filter @markluck/app lint:style`                  | ✅ PASS | Stylelint 零错误                       |
| 6   | `pnpm.cmd --filter @markluck/app exec vitest run`             | ✅ PASS | 5 test files, 145 tests all passed     |
| 7   | `pnpm.cmd --filter @markluck/app build`                       | ✅ PASS | 构建成功 (4.71s)                       |

### 构建产物

| 文件                                    | 大小            | Gzip          |
| --------------------------------------- | --------------- | ------------- |
| `dist/index.html`                       | 0.41 kB         | 0.28 kB       |
| `dist/assets/index-eo2DjQhC.css`        | 30.83 kB        | 6.06 kB       |
| `dist/assets/NotebookHome-BJpbbQpQ.css` | 65.53 kB        | 9.49 kB       |
| `dist/assets/index-rR13BdQi.js`         | 116.66 kB       | 46.50 kB      |
| `dist/assets/NotebookHome-BTyWgBph.js`  | **1,468.52 kB** | **477.41 kB** |

### 构建 Warning

1. **Tauri API 动态/静态导入冲突**: `@tauri-apps/api/event.js` 被 `NotebookHome.vue` 动态导入，同时被 `TauriIPCService.ts` 静态导入。不影响功能。
2. **NotebookHome chunk 超 500 kB**: 1,468.52 kB (gzip 477.41 kB)。已知问题，`spec/release-hardening-deepseek-plan.md` §0 已记录，在 M-R5 中评估是否低风险懒加载优化。

### 未提交改动

| 文件                                                                    | 变更内容                                                                                              |      判定      |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | :------------: |
| `memory/bug_log.md`                                                     | BUG-061 记录 + 检查清单增补 (+12 行)                                                                  | 文档补充，无害 |
| `packages/app/src/services/CompletionTrainingService.ts`                | `createDefaultTrainingMeta()` 工厂函数，修复 `DEFAULT_TRAINING_META.trainedPaths` 引用泄漏 (+7/-3 行) |  真 BUG 修复   |
| `packages/app/src/services/__tests__/CompletionTrainingService.test.ts` | 回归测试：清空 localStorage 后 trainedPaths 为空 (+13 行)                                             |    测试补充    |

这些改动是 BUG-061 的修复（默认状态对象浅拷贝导致 `trainedPaths` 引用泄漏），应在 M-R0 后、M-R1 前提交。

### 未跟踪文件

| 文件                                      | 说明                                         |
| ----------------------------------------- | -------------------------------------------- |
| `spec/release-hardening-deepseek-plan.md` | 发布打磨计划规格文档（新增，应纳入版本管理） |

### 基线判定

- ✅ **所有 L1 检查通过** (typecheck/eslint/prettier/stylelint)
- ✅ **L2 全量通过** (vitest 145/145)
- ✅ **Web build 通过** (含已知 chunk size warning)
- ⚠️ 3 个未提交改动 (BUG-061 修复 + 测试) + 1 个未跟踪文件 (发布打磨计划)
- ⚠️ Playwright E2E 全量回归未在本里程碑运行（属于 M-R1 范围）

### 结论

**基线可继续。** 当前工作区干净（仅有 BUG-061 修复待提交和发布计划文档待跟踪），所有自动化检查绿色，构建成功。M-R0 后应先提交未保存改动，再进入 M-R1。

---

M-R0 已到达停止验收点，等待 Codex 验收。不得继续执行下一里程碑。
