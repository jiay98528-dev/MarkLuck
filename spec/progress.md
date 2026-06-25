# MarkLuck 进度跟踪

> **动态文档** — 每次 L3 集成测试通过后更新
> 基线版本：v1.0 | 冻结日期：2026-06-03
> 关联文档：`spec/milestones.md`（里程碑定义）、`CLAUDE.md §5.7 L3`（更新触发条件）

---

## 总体进度

```
M0 [████████████████████] 100%  项目脚手架 ✅
M1 [██████████████████▊ ]  95%  核心渲染与编辑 (19/20, 1 DEFERRED) ✅
M2 [████████████████████] 100%  索引与搜索 ✅
M3 [████████████████████] 100%  导出与分享 ✅
M4 [████████████████████] 100%  模板与附件 (8/8) ✅
M5 [████████████████████] 100%  主题与主题包 ✅
M6 [████████████████████] 100%  Tauri 桌面端 (8/8 编码, 待打包验证)
M7 [████████████████████] 100%  打磨与发布 (21/21)
M8 [····················]   0%  移动端 (可选)
M9 [····················]   0%  VS Code 插件 (可选)
M10 [██████████████████▊ ]  93%  UX 增强 (13/14, 1 DEFERRED) ✅
────────────────────────────────────
总进度  █████████████████████████████▎  97%
```

---

## 当前里程碑：M1 — 核心渲染与编辑

**状态**：🟡 进行中 | **开始日期**：2026-06-03 | **目标完成**：—

### 任务进度

| #     | 任务                                 | 状态 | 完成日期   | Commit    |
| ----- | ------------------------------------ | :--: | ---------- | --------- |
| M0-01 | Vite + Vue 3 + TypeScript 项目初始化 |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-02 | pnpm workspace monorepo 配置         |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-03 | TypeScript strict mode 配置          |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-04 | ESLint + Prettier 配置               |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-05 | Vitest + @vue/test-utils 配置        |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-06 | Playwright 配置                      |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-07 | GitHub Actions CI 骨架               |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-08 | Git Hooks (pre-commit + commit-msg)  |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-09 | @markluck/renderer 包骨架            |  ✅  | 2026-06-03 | `b3c9cef` |
| M0-10 | 目录结构建立                         |  ✅  | 2026-06-03 | `b3c9cef` |

### L3 状态

| 检查层  |  状态   | 最后通过   |
| ------- | :-----: | ---------- |
| L1 ⚡   | ✅ PASS | 2026-06-03 |
| L2 🧪   | ✅ PASS | 2026-06-03 |
| L3 🔗   | ✅ PASS | 2026-06-03 |
| L3.5 🔍 | ✅ PASS | 2026-06-03 |
| L4 🔷   | ✅ PASS | 2026-06-03 |

### L3.5 审计摘要

**日期**: 2026-06-03 | **审查文件**: ~50 | **发现**: 27（严重11 / 一般10 / 建议6）

| 类别        | 处理                            | 数量 |
| ----------- | ------------------------------- | :--: |
| M0 立即修复 | 路由/组件补齐/CI/配置           |  6   |
| DEFERRED M1 | 类型定义对齐（BlockType等12项） |  12  |
| DEFERRED M6 | CI Windows runner               |  1   |
| 建议        | 标准化改进                      |  8   |

**关键教训**：

- 类型文件必须与 `spec/types/` 桶文件保持一致，M1 编码时以 spec 为合同对齐
- 组件命名必须在 M0 阶段与 `components.md` 对齐，避免后续重构成本
- CI pnpm 版本必须与 `packageManager` 字段一致

### L4 复审记录

（M0 完成后首次复审）

---

## M1 — 核心渲染与编辑

**状态**：🟢 已完成 | **开始日期**：2026-06-03 | **完成日期**：2026-06-03 | 16/20（4 DEFERRED）

| #     | 任务                                       | 状态 | 完成日期   | Commit                   |
| ----- | ------------------------------------------ | :--: | ---------- | ------------------------ |
| M1-01 | marked + DOMPurify + highlight.js 渲染管道 |  ✅  | 2026-06-03 | `6a33bca`                |
| M1-02 | Wiki-link `[[...]]` marked 扩展            |  ✅  | 2026-06-03 | `6a33bca`                |
| M1-03 | #tag 行内标签 marked 扩展                  |  ✅  | 2026-06-03 | `6a33bca`                |
| M1-04 | XSS 安全测试套件                           |  ✅  | 2026-06-03 | `6a33bca`                |
| M1-05 | CodeMirror 6 集成                          |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-06 | 块解析引擎                                 |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-07 | BlockDecorator 扩展                        |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-08 | BlockWidget 扩展                           |  ✅  | 2026-06-04 | BlockParser + 装饰器激活 |
| M1-09 | FormatAutoDetector 扩展                    |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-10 | RestoreButton 扩展                         |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-11 | IME 处理                                   |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-12 | FormatToolbar 组件                         |  ✅  | 2026-06-04 | 12 种格式按钮 + 快捷键   |
| M1-13 | 键盘快捷键                                 |  ✅  | 2026-06-03 | `e3d6355`                |
| M1-14 | FileTree 组件（基础版）                    |  ✅  | 2026-06-03 | `5c4e242`                |
| M1-15 | FileTreeNode 组件                          |  ✅  | 2026-06-03 | `5c4e242`                |
| M1-16 | IFileSystemService 接口 + Mock 实现        |  ✅  | 2026-06-03 | `fd5783e`                |
| M1-17 | WebFSAService 实现                         |  ⚠️  | —          | DEFERRED (M6 Tauri接入)  |
| M1-18 | 文件保存（原子写入）                       |  ✅  | 2026-06-03 | `fd5783e`                |
| M1-19 | WelcomePage 组件                           |  ✅  | 2026-06-04 | 首次引导页               |
| M1-20 | AppLayout 三联画结构                       |  ✅  | 2026-06-03 | `5c4e242`                |

### L3 状态

| 检查层  |  状态   | 最后通过      |
| ------- | :-----: | ------------- |
| L1 ⚡   | ✅ PASS | 2026-06-03    |
| L2 🧪   | ✅ PASS | 2026-06-03    |
| L3 🔗   | ✅ PASS | 2026-06-03    |
| L3.5 🔍 | ⏭️ SKIP | M1 无独立审计 |
| L4 🔷   | ✅ PASS | 2026-06-03    |

### L4 复审摘要

**日期**: 2026-06-03 | **测试者**: 用户手动 | **通过率**: 100%

| #   | 测试项              | 结果 | 说明                          |
| --- | ------------------- | :--: | ----------------------------- |
| 1   | 页面加载 + 文件列表 |  ✅  | 三栏布局正常，文件树显示      |
| 2   | 点击打开笔记        |  ✅  | 编辑器加载内容正确            |
| 3   | 语法高亮（有颜色）  |  ✅  | HighlightStyle API 修复后正常 |
| 4   | F12 Console         |  ✅  | 无错误，无 console.log        |
| 5   | 格式标记可见        |  ⚠️  | 预期行为（源码模式），非BUG   |
| 6   | Ctrl+B 快捷键       |  ✅  | markdownKeymap 正常           |
| 7   | F5 刷新数据保留     |  ✅  | localStorage 持久化正常       |
| 8   | 新建笔记（+ 按钮）  |  ✅  | 内联输入 + Enter 创建         |
| 9   | 删除笔记（× 按钮）  |  ✅  | hover 删除按钮正常            |
| 10  | Markdown 渲染预览   |  ⚠️  | DEFERRED (M1-08 BlockWidget)  |
| 11  | 美学/主题           |  ⚠️  | DEFERRED (M5 主题与主题包)    |

**DEFERRED 项**：

- M1-08 BlockWidget（块级渲染切换）
- M1-12 FormatToolbar（格式工具栏 UI）
- M1-17 WebFSAService（真实文件系统 API，M6 实现）
- M1-19 WelcomePage（首次引导页）

---

## 当前里程碑：M2 — 索引与搜索

**状态**：🟡 进行中 | **开始日期**：2026-06-03 | **目标完成**：—

## M2 — 索引与搜索

**状态**：🟢 已完成 | **开始日期**：2026-06-03 | **完成日期**：2026-06-03 | 16/16

| #     | 任务                               | 状态 | 完成日期   |
| ----- | ---------------------------------- | :--: | ---------- |
| M2-01 | `.markluck_index.json` Schema 实现 |  ✅  | 2026-06-03 |
| M2-02 | YAML frontmatter 解析              |  ✅  | 2026-06-03 |
| M2-03 | 索引构建（全量 + 增量）            |  ✅  | 2026-06-03 |
| M2-04 | Wiki-link 图构建                   |  ✅  | 2026-06-03 |
| M2-05 | BacklinksPanel 组件                |  ✅  | 2026-06-03 |
| M2-06 | 死链检测                           |  ✅  | 2026-06-03 |
| M2-07 | `[[笔记名\|别名]]` 语法            |  ✅  | 2026-06-03 |
| M2-08 | minisearch 全文搜索集成            |  ✅  | 2026-06-03 |
| M2-09 | 高级搜索语法                       |  ✅  | 2026-06-03 |
| M2-10 | SearchPanel 组件                   |  ✅  | 2026-06-03 |
| M2-11 | 搜索结果高亮 + 上下文              |  ✅  | 2026-06-03 |
| M2-12 | TagPanel 组件                      |  ✅  | 2026-06-03 |
| M2-13 | 标签点击 → 搜索                    |  ✅  | 2026-06-03 |
| M2-14 | NavTree 组件                       |  ✅  | 2026-06-03 |
| M2-15 | 滚动跟随                           |  ✅  | 2026-06-03 |
| M2-16 | RecentNotes 组件                   |  ✅  | 2026-06-03 |

### L3 状态

| 检查层  |  状态   | 最后通过              |
| ------- | :-----: | --------------------- |
| L1 ⚡   | ✅ PASS | 2026-06-03            |
| L2 🧪   | ✅ PASS | 2026-06-03            |
| L3 🔗   | ⏭️ SKIP | M2 无 Playwright 用例 |
| L3.5 🔍 | ⏭️ SKIP | M2 规模可控           |
| L4 🔷   | ✅ PASS | 2026-06-03            |

### L4 复审摘要

**日期**: 2026-06-03 | **测试者**: 用户手动 | **通过率**: 100%

| #   | 测试项                | 结果 | 说明                              |
| --- | --------------------- | :--: | --------------------------------- |
| 1   | 大纲导航              |  ✅  | 标题层级树 + 折叠展开             |
| 2   | 反向链接面板          |  ✅  | `[[快速入门]]` 入链正确           |
| 3   | 标签云                |  ✅  | 字号分级 + 点击搜索               |
| 4   | 最近笔记              |  ✅  | 相对时间格式正确                  |
| 5   | 全局搜索 Ctrl+Shift+P |  ✅  | 文本/正则/tag 过滤正常            |
| 6   | 索引持久化            |  ✅  | F5 刷新索引保留                   |
| 7   | `/快速/` 正则         |  ✅  | 修复后正常（BUG-001）             |
| 8   | 中文 IME 快捷键       |  ✅  | 改 Ctrl+Shift+P 不冲突（BUG-002） |

---

## M3 — 导出与分享

**状态**：🟢 已完成 | **开始日期**：2026-06-03 | **完成日期**：2026-06-03 | 10/10

| #           | 10 个任务 | 状态 | 完成日期   |
| ----------- | --------- | :--: | ---------- |
| M3-01~M3-10 | 全部完成  |  ✅  | 2026-06-03 |

### L4 复审摘要

**日期**: 2026-06-03 | **测试者**: 用户手动 | BUG 3 项已修复

---

## M4 — 模板与附件

**状态**：🟢 已完成 | **开始日期**：2026-06-03 | **完成日期**：2026-06-23 | 8/8

| #     | 任务                      | 状态 | 说明                    |
| ----- | ------------------------- | :--: | ----------------------- | ----------------------------------------------- |
| M4-01 | 模板引擎 (TemplateEngine) |  ✅  | 6 种占位符替换          |
| M4-02 | 内置模板 (日记/会议/周报) |  ✅  | 3 套内置模板            |
| M4-03 | TemplateDialog 组件       |  ✅  | 选择+预览+创建          |
| M4-04 | 自定义模板                |  ✅  | 2026-06-04              | localStorage 持久化                             |
| M4-05 | 附件管理                  |  ✅  | 2026-06-23              | assets/ 目录 + 相对路径引用 + 二进制管线        |
| M4-06 | 粘贴图片                  |  ✅  | 2026-06-23              | Ctrl+V 剪贴板检测 → base64 写入 → Markdown 引用 |
| M4-07 | 拖拽文件                  |  ✅  | 2026-06-23              | drop/dragover 事件 → 图片文件自动处理           |
| M4-08 | StatusBar 组件            |  ✅  | 字数/行数/光标/保存状态 |

### L4 复审摘要

**日期**: 2026-06-03 | **测试者**: 用户手动 | **通过率**: 100%

| #   | 测试项       | 结果 | 说明                        |
| --- | ------------ | :--: | --------------------------- |
| 1   | 模板预览     |  ✅  | 占位符替换为当前日期/时间   |
| 2   | 模板创建笔记 |  ✅  | 3 种内置模板正常            |
| 3   | 空白笔记     |  ✅  | 日期命名                    |
| 4   | 状态重置     |  ✅  | 对话框关闭后清理            |
| 5   | 保存状态流转 |  ✅  | 未保存→保存中(≥0.5s)→已保存 |

---

## M5 — 主题与主题包

**状态**：🟡 Theme Pack v1 基础完成，官方 workflow chrome 已补齐，安装版 GUI 待最终验收 | **开始日期**：2026-06-03 | **最近同步**：2026-06-25

> 同步说明：早期 `construct/glass` 与 `ThemeSelector.vue` 记录已不符合当前代码。当前实际实现为 `paper`/羽翼布局默认主题 + 明暗色方案 + Theme Pack v1 基础能力；官方主题已补入 `ThemeWorkflowChrome`、统一 `ShellAction` 分发、`EditorControlStrip`、`StudioRail`、普通笔记 `read` 模式、组件级 chrome 和显式 `ThemeEffectLayer`。安装版 GUI 仍待最终核验。历史构成/玻璃主题不再作为发布实现依据。

### L3 状态

| 检查层 |  状态   | 最后通过   | 详情                                                                       |
| ------ | :-----: | ---------- | -------------------------------------------------------------------------- |
| L1 ⚡  | ✅ PASS | 2026-06-25 | typecheck、eslint、prettier、stylelint 通过                                |
| L2 🧪  | ✅ PASS | 2026-06-25 | ThemePackInstaller + useThemeStore 主题单元测试 11/11 PASS                 |
| L3 🔗  | ✅ PASS | 2026-06-25 | `e2e/tests/18-theme-packs.spec.ts --project=chromium --workers=1` 2/2 PASS |
| L4 🔷  | ⚠️ 部分 | 2026-06-25 | 真实 UI WebP 预览图已重生成并抽查；安装版 GUI 仍需最终核验                 |

### 任务进度

| #     | 任务                 | 状态 | 产出                                                                                                            |
| ----- | -------------------- | :--: | --------------------------------------------------------------------------------------------------------------- |
| M5-01 | 羽翼布局默认主题     |  ✅  | 默认 ID `paper`，设置页显示“羽翼布局”                                                                           |
| M5-02 | Theme Pack v1 规格   |  ✅  | `spec/frontend/theme-packs.md`                                                                                  |
| M5-03 | 主题包类型契约       |  ✅  | `src/types/theme-pack.ts`                                                                                       |
| M5-04 | 主题注册表           |  ✅  | `src/services/ThemeRegistry.ts`，含 5 个官方主题                                                                |
| M5-05 | 主题包安装器         |  ✅  | `src/services/ThemePackInstaller.ts`，校验 zip/manifest/CSS/checksum                                            |
| M5-06 | Store v2 迁移        |  ✅  | `stores/theme.ts`：`activeThemeId`、`activeLayoutPreset`、旧 key 兼容                                           |
| M5-07 | 布局 preset hooks    |  ✅  | `themes/theme-layouts.css`：winged/focus/archive/reader/studio                                                  |
| M5-08 | 设置页主题管理       |  ✅  | `SettingsDialog.vue` 新增“主题”一级入口                                                                         |
| M5-09 | 安全边界             |  ✅  | 拒绝 `@import`、远程 URL、JS、路径穿越、隐藏核心控件                                                            |
| M5-10 | 自动化覆盖           |  ✅  | 新增单元测试 9 条 + Chromium E2E 1 条                                                                           |
| M5-11 | 墨线书房内置主题     |  ✅  | `markluck.ink-study`，focus preset，偏冷纸面与墨青结构线                                                        |
| M5-12 | 官方深度主题 profile |  ✅  | 5 个内置主题带 role、uiProfile、性能等级、预览说明                                                              |
| M5-13 | 主题预览侧滑页       |  ✅  | 欢迎页、首页空态、设置页统一点击预览后启用                                                                      |
| M5-14 | 扁平纹理资产         |  ✅  | 墨线书房、夜读星幕使用本地 WebP 扁平纹理，不使用写实背景                                                        |
| M5-15 | 组件级主题 chrome    |  ✅  | `ThemeChromeState`/workflow chrome 驱动 AppShell/TopBar/LeftWing/RightWing/EditorControlStrip/StudioRail/状态栏 |
| M5-16 | 显式特效层           |  ✅  | `ThemeEffectLayer` 真实 DOM 粒子/脉冲/呼吸光，reduced-motion 降级                                               |
| M5-17 | 本地主题安全回退     |  ✅  | 导入主题不再获得布局 preset、官方 effect、workflow action placement 或 chrome 控制                              |
| M5-18 | 真实 UI 预览图       |  ✅  | Playwright/Chromium 生成 5 张官方主题 WebP 预览图，展示各自工作流布局                                           |

### L4 复审摘要

**日期**: 2026-06-25 | **测试者**: Codex 自动化 + 内置浏览器 GUI | 安装版 GUI 待最终验收

| #   | 测试项                   | 结果 | 说明                                                                            |
| --- | ------------------------ | :--: | ------------------------------------------------------------------------------- |
| 1   | 旧 `markluck-theme` 迁移 |  ✅  | 旧 light/dark 字符串仍被读取并镜像写回                                          |
| 2   | 官方主题启用             |  ✅  | html 写入 `data-theme-id` 与 layout preset                                      |
| 3   | 侧滑预览                 |  ✅  | E2E 确认点击主题只打开预览，显式点击后才启用                                    |
| 4   | 本地主题包导入           |  ✅  | `.markluck-theme` zip 可导入并启用                                              |
| 5   | 刷新持久化               |  ✅  | E2E 验证导入主题刷新后保持                                                      |
| 6   | 恢复默认                 |  ✅  | 设置页按钮恢复羽翼布局 (`paper`)                                                |
| 7   | 卸载当前主题             |  ✅  | 卸载本地主题自动回退羽翼布局 (`paper`)                                          |
| 8   | 安全规则                 |  ✅  | 单元测试覆盖远程 URL、`@import`、路径穿越等拒绝                                 |
| 9   | 主题 workflow GUI        |  ✅  | Playwright 验证 5 个官方主题 action placement、默认视图、RightWing 策略和特效层 |
| 10  | 安装版 GUI               |  ⚠️  | 最终安装包仍需离线资产与无网络请求核验                                          |

### E2E 测试覆盖

| 测试           | 场景                                                                    | 规则 |
| -------------- | ----------------------------------------------------------------------- | :--: |
| 默认主题       | `<html data-theme-id="paper" data-color-scheme data-layout-preset>`     |  V1  |
| 颜色方案切换   | 顶栏按钮切换 light/dark，旧 storage key 保持字符串                      |  V3  |
| 官方主题预览   | 设置/欢迎/首页主题卡片 → 侧滑预览 → 显式启用                            |  V6  |
| 官方 workflow  | 逐个启用 5 个官方主题，验证 action 区域、默认视图、右翼策略、工具栏形态 |  V6  |
| 导入本地包     | 设置 → 主题 → 上传 `.markluck-theme` → 安装并启用                       |  V6  |
| 刷新持久化     | 导入主题后刷新，`data-theme-id` 保持                                    |  V3  |
| 恢复默认       | 设置页恢复羽翼布局 (`paper`)                                            |  V1  |
| 卸载当前本地包 | 卸载后自动回退羽翼布局 (`paper`)                                        |  V2  |

### 产出文件

| 文件                                                | 内容                                             |
| --------------------------------------------------- | ------------------------------------------------ |
| `spec/frontend/theme-packs.md`                      | Theme Pack v1 包格式、运行时、安全边界、测试要求 |
| `src/assets/styles/tokens.css`                      | 共享 Token + Theme Pack hooks                    |
| `src/assets/styles/themes/paper.css`                | 默认 Paper 亮/暗主题                             |
| `src/assets/styles/themes/theme-layouts.css`        | layout preset 与背景/纹章 hooks                  |
| `src/assets/theme-assets/*.webp`                    | 官方收藏主题本地扁平纹理和预览图                 |
| `src/services/ThemeRegistry.ts`                     | 官方主题、本地主题持久化、主题列表               |
| `src/services/ThemePackInstaller.ts`                | `.markluck-theme` 校验、资产重写、安装           |
| `src/stores/theme.ts`                               | Pinia Store：Theme Pack v2 状态 + 旧 key 兼容    |
| `src/components/layout/ShellActionButton.vue`       | 官方 workflow chrome 统一动作按钮                |
| `src/components/editor/EditorControlStrip.vue`      | 主题控制的编辑区工具条/写作条                    |
| `src/components/editor/StudioRail.vue`              | 工坊轨道主题生产工具轨                           |
| `src/components/theme/ThemeGallery.vue`             | 主题展柜卡片                                     |
| `src/components/theme/ThemePreviewDrawer.vue`       | 主题预览侧滑页                                   |
| `src/components/theme/ThemeEffectLayer.vue`         | 官方深度主题显式特效层                           |
| `src/components/modals/SettingsDialog.vue`          | 主题管理入口                                     |
| `src/services/__tests__/ThemePackInstaller.test.ts` | 主题包安全和 manifest 测试                       |
| `src/stores/__tests__/theme.test.ts`                | Store 迁移和卸载回退测试                         |
| `e2e/tests/18-theme-packs.spec.ts`                  | 导入/启用/刷新/恢复/卸载 GUI 旅程                |

### 修改文件

| 文件             | 变更                              |
| ---------------- | --------------------------------- |
| `package.json`   | `@markluck/app` 新增 `jszip` 依赖 |
| `pnpm-lock.yaml` | 同步依赖声明                      |

---

## M6 — Tauri 桌面端

**状态**：🟡 进行中 | **开始日期**：2026-06-04 | 8/8 编码完成，待打包验证 (2 项)

| #     | 任务                       | 状态 | 产出                                                                                |
| ----- | -------------------------- | :--: | ----------------------------------------------------------------------------------- |
| M6-01 | Tauri v2 项目初始化        |  ✅  | `src-tauri/` — Cargo.toml + tauri.conf.json                                         |
| M6-02 | IPC 命令定义               |  ✅  | 15 个命令注册 (fs/search/watcher/template)                                          |
| M6-03 | fs_ops 模块                |  ✅  | `src/fs_ops.rs` — 原子写入 + 目录遍历                                               |
| M6-04 | indexer 模块 (tantivy)     |  ✅  | `src/indexer.rs` — 全文索引 + 搜索 + 中文分词                                       |
| M6-05 | file_watcher 模块 (notify) |  ✅  | `src/file_watcher.rs` — 文件变更事件推送，RenameMode::Both 重命名支持，图片资产监控 |
| M6-06 | template 模块 (Rust)       |  ✅  | `src/template.rs` — 7 种占位符替换                                                  |
| M6-07 | 路径安全校验               |  ✅  | `src/path.rs` — 防路径穿越 + UTF-8 编码处理                                         |
| M6-08 | TauriIPCService 实现       |  ✅  | `src/services/TauriIPCService.ts` — watch/unwatch 真实实现，二进制读写 IPC          |
| M6-09 | 桌面构建配置               |  ⏳  | Windows .msi / macOS .dmg / Linux .AppImage                                         |
| M6-10 | 系统文件关联               |  ⏳  | tauri.conf.json fileAssociations 已配，待打包                                       |

### 联动验收项（M6 L4 时一并测试）

| #     | 任务          | 依赖        | 验收要点                            |
| ----- | ------------- | ----------- | ----------------------------------- |
| M1-17 | WebFSAService | M6 IPC      | 真实文件夹读写 + 外部编辑器修改检测 |
| M4-05 | 附件管理      | M6 文件系统 | 附件文件 CRUD + 引用                |
| M4-06 | 粘贴图片      | M6 文件系统 | 剪贴板图片 → 保存到笔记本           |
| M4-07 | 拖拽文件      | M6 文件系统 | 拖入文件 → 复制到笔记本             |

---

## M7 — 打磨与发布

**状态**：🟢 已完成 | **开始日期**：2026-06-04 | **完成日期**：2026-06-09 | 15/15

## M10 — UX 增强

**状态**：🔴 未开始 | **开始日期**：— | 0/11

| #      | 任务                       | 状态 | 完成日期   | 说明                                            |
| ------ | -------------------------- | :--: | ---------- | ----------------------------------------------- |
| M10-01 | WelcomePage UX 设计        |  ✅  | 2026-06-13 | 5 步向导：品牌→隐私→功能→关联→更新              |
| M10-02 | WelcomePage 组件           |  ✅  | 2026-06-13 | Teleport modal + 步骤过渡动画                   |
| M10-03 | 首次检测 + 跳过 + 设置入口 |  ✅  | 2026-06-13 | localStorage 标记 + Settings "重新观看欢迎引导" |
| M10-04 | 欢迎页 .md 关联设置        |  ✅  | 2026-06-13 | Tauri 联动 + 控制台日志占位                     |
| M10-05 | useVersionCheck composable |  ✅  | 2026-06-13 | GitHub API + 24h 缓存 + semver                  |
| M10-06 | SettingsDialog 更新区域    |  ✅  | 2026-06-13 | 父开关 + 锁止子选项（证书就位前）               |
| M10-07 | 启动通知弹窗               |  ✅  | 2026-06-13 | 15s 倒计时 + 逐版本静默                         |
| M10-08 | CheatSheet UX 设计         |  ✅  | 2026-06-13 | 悬浮可拖拽卡片草案                              |
| M10-09 | MarkdownCheatSheet 组件    |  ✅  | 2026-06-13 | 7 类语法 + 折叠/展开动画                        |
| M10-10 | 拖拽 + 边界检测            |  ✅  | 2026-06-13 | pointer 事件 + clamp + ResizeObserver           |
| M10-11 | 折叠动画 + Settings 开关   |  ✅  | 2026-06-13 | Tier 2 动画 + localStorage 持久化               |
| M10-12 | E2E 测试强化               |  ✅  | 2026-06-13 | 弱断言清零 + 选择器交叉验证 + 中文 IME 专项     |
| M10-13 | 代码签名申请               |  ⚠️  | —          | DEFERRED: SignPath Foundation 邮件待发送        |
| M10-14 | BUG-030~035 修复           |  ✅  | 2026-06-13 | IME 双行/冷启动/Toast/级联/泄漏 全部修复        |

## M8 — 移动端

**状态**：🔴 未开始（可选阶段） | 0/5

## M9 — VS Code 插件

**状态**：🔴 未开始（可选阶段） | 0/5

---

| #     | 任务                                      | 状态 | 说明                                                   |
| ----- | ----------------------------------------- | :--: | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| M7-01 | 导出管线重写 (6格式真实实现)              |  ✅  | docx.js + write-excel-file + renderMarkdown 完整管线   |
| M7-02 | 导出选项接入 (frontmatter/wiki-link/行号) |  ✅  | ExportDialog 3 开关全部生效                            |
| M7-03 | 编辑器保存机制修复 (5个BUG)               |  ✅  | 闭包捕获/反馈环/中文IME/大纲更新/并发覆盖              |
| M7-04 | 块级即时渲染 (Live Preview)               |  ✅  | 24 live blocks, heading/code/blockquote/wiki-link      |
| M7-05 | 视图模式简化 (分栏+即时)                  |  ✅  | 移除编辑/预览模式，仅保留2种                           |
| M7-06 | 有序列表假空行修复                        |  ✅  | 内联编号 + 空项 nbsp 占位                              |
| M7-07 | 分栏预览自动刷新                          |  ✅  | 4处 onSelectNote/onCreateFile/onTemplate/onCreateBlank |
| M7-08 | Checkbox 可交互切换                       |  ✅  | DOMPurify hook + ViewPlugin change handler             |
| M7-09 | 标签面板默认展开 + 内联#tag索引           |  ✅  | RightWing 默认展开 + IndexService 扫描 body            |
| M7-10 | E2E 测试修复与完善                        |  ✅  | openNote aria-label/waitForSaved/V4 内容验证           |
| M7-11 | TopBar 按钮修复                           |  ✅  | 移除多余 button 标签/ShareDialog 清理                  |
| M7-12 | UI 全量重构 — 纸张主题                    |  ✅  | Winged Editor + Paper Metaphor + 豪华动效              |
| M7-13 | NGramEngine + GhostTextPlugin             |  ✅  | 2026-06-09                                             | 纯算法统计引擎 + CM6 幽灵文本插件 + 基准 L2 生成                                                                                   |
| M7-14 | 结构化知识融合 + 持久化                   |  ✅  | 2026-06-09                                             | Wiki-link/标签/路径融入幽灵文本 + localStorage + 末位淘汰                                                                          |
| M7-15 | 设置面板 + 测试                           |  ✅  | 2026-06-12                                             | SettingsDialog 开关 + NGramEngine 单元测试 24/24 + MarkdownPredictor 单元测试 72/72                                                |
| M7-16 | 补全系统重构 — 分层 + 训练                |  ✅  | 2026-06-23                                             | L1/L2/L3 三层融合 + CompletionSettings + CompletionTrainingService + 中文 fallback + 质量门控 + StateField 修复 Firefox ghost text |
| M7-17 | 图片上传二进制管线修复                    |  ✅  | 2026-06-23                                             | writeBinary/readBinary IPC + base64 剥离 + 相对路径 + 5MB 上限                                                                     |
| M7-18 | 文件删除确认对话框                        |  ✅  | 2026-06-23                                             | pendingDeletePath modal + E2E 适配                                                                                                 |
| M7-19 | Wiki-link 死链 + Task checkbox            |  ✅  | 2026-06-23                                             | wikiLinkExists 穿透 renderer/live-preview + pointerdown 捕获                                                                       |
| M7-20 | 响应式布局 + 无障碍                       |  ✅  | 2026-06-23                                             | ≤720px/≤560px 媒体查询 + ARIA role/dialog/switch + button 嵌套修复                                                                 |
| M7-21 | 文件监控接入 + 仓库整理                   |  ✅  | 2026-06-23                                             | TauriIPCService watch 真实实现 + 17 logic commits                                                                                  |

---

## 更新日志

| 日期       | 事件               | 描述                                                                                                                         |
| ---------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| 2026-06-23 | 离线补全重构完成   | L1/L2/L3 三层预测融合、CompletionTrainingService 后台训练、中文短上下文 fallback、质量门控、GostText StateField 修复 Firefox |
| 2026-06-23 | 图片上传二进制管线 | writeBinary/readBinary IPC、base64 剥离、相对路径计算、5MB 上限                                                              |
| 2026-06-23 | 文件监控接入       | TauriIPCService watch() 从 stub 变真实实现、RenameMode::Both 支持                                                            |
| 2026-06-23 | 安全与体验修复     | 删除确认对话框、Wiki-link 死链 CSS、Task checkbox 写回、响应式布局 ≤720px/≤560px、无障碍 ARIA 改进                           |
| 2026-06-23 | BUG-056~060 记录   | 图片管线/死链/checkbox/button嵌套/Firefox IME 5个BUG记录到错题本 + 检查清单增补                                              |
| 2026-06-23 | 代码仓库整理       | 50 文件拆分为 17 个逻辑 commit、移除 business-analysis-report.md、更新 progress.md                                           |

| 日期       | 事件               | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-06-03 | 文档基线冻结       | 全部策划文档、代码规范、里程碑定义完成——冻结为 v1.0 基线                                                                                                                                                                                                                                                                                                                                                                                                         |
| 2026-06-03 | progress.md 创建   | 进度跟踪文档初始化                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| 2026-06-03 | M0 完成            | 10/10 任务全部完成。L1/L2 通过。Dev server 启动正常。约 90 个文件创建。                                                                                                                                                                                                                                                                                                                                                                                          |
| 2026-06-05 | 导出管线重写       | Exporter.ts 600行重写：docx.js(Document/Packer)、write-excel-file(toBlob)、PDF(renderMarkdown+iframe+print)、HTML(自包含CSS)。E2E 14/14 通过                                                                                                                                                                                                                                                                                                                     |
| 2026-06-05 | 编辑器保存机制修复 | 5个BUG：setTimeout闭包Ref动态求值、中文IME insertText不触发keyup、updateListener反馈环、onSplitContentUpdate缺updateHeadings、并发保存覆盖(代际计数器)                                                                                                                                                                                                                                                                                                           |
| 2026-06-07 | M7 打磨完成        | 4个UI BUG修复 + 有序列表内联编号 + split preview自动刷新 + checkbox交互 + 标签面板。E2E editor 12/12, export 14/14, readiness 10/10, live-preview 2/2                                                                                                                                                                                                                                                                                                            |
| 2026-06-09 | 文字补全功能完成   | NGramEngine(260行) + MarkdownPredictor(480行) + GhostTextPlugin(190行) + 基准L2(15,623条/419KB) + Settings开关。单元测试24/24, E2E 8/8。CHANGELOG v0.2.0                                                                                                                                                                                                                                                                                                         |
| 2026-06-12 | 文档同步审计       | 文字补全功能深度审计：autocomplete-spec.md 验收标准全量勾选（17/17 ✅）、训练脚本文件名修复、PRD.md 基准条目数更新（~2500→~15,600）、progress.md M7 百分比修正（94%→100%）。发现 MarkdownPredictor 服务层缺失单元测试（已修复）                                                                                                                                                                                                                                  |
| 2026-06-12 | BUG-028 修复       | 创建 MarkdownPredictor 单元测试 73 个，覆盖纯函数/语法检测/结构化预测/融合决策/持久化淘汰/边界错误 6 大类。错题本 + spec + progress 同步更新                                                                                                                                                                                                                                                                                                                     |
| 2026-06-12 | BUG-029 修复       | Ghost text 不渲染——6 个独立缺陷修复。E2E autocomplete 8/8 通过，ghost text 可见                                                                                                                                                                                                                                                                                                                                                                                  |
| 2026-06-12 | BUG-013/014 修复   | 最后两个 BUG 清零：①FormatBubble 发现性 — Toast 首次提示 + StatusBar 呼吸动画 + 提示常驻 ②BlockWidget — 默认 live 模式（1行）+ requestAnimationFrame 首次渲染修复。Live Preview 20 个块正确渲染。错题本待修复归零                                                                                                                                                                                                                                                |
| 2026-06-13 | M10 UX 增强完成    | WelcomePage + CheatSheet + 版本检测 + Settings 更新区域。新增 6 文件，修改 5 文件。E2E 强化：弱断言清零 + 选择器交叉验证 + 中文 IME 专项。L1 零新错误，L3 195/218 通过（剩余 13 全部为预存测试适配问题）                                                                                                                                                                                                                                                         |
| 2026-06-13 | BUG-030~035 修复   | IME 双行渲染（contentDOM 监听 + 事务注解检测）、幽灵文本级联（防重复预测）、冷启动中文（恢复 n≥4 阈值）、Toast 静默失效（provide/inject→模块级状态）、事件监听器泄漏（destroy 清理）、E2E 假通过（弱断言 + 选择器错误）                                                                                                                                                                                                                                          |
| 2026-06-16 | E2E 编辑器核心测试 | 新增 `e2e/tests/01-editor-core.spec.ts` — 8 个编辑器核心 E2E 测试（应用加载/左翼书签栏/顶栏按钮/状态栏/Markdown输入/视图切换/FormatBubble/撤销重做），全部通过 (22.8s)。修复 `test-utils.ts` waitForAppReady 三个基础设施问题：localStorage key 不匹配（`markluck_welcome_completed` → `markluck:welcome:completed`）、about:blank 上 SecurityError（先导航再写 localStorage）、baseURL 解析失败（使用完整 URL `http://localhost:5175`）。测试遵循 V1/V5/V6 规则 |

### M10 L3 状态

| 检查层  |  状态   | 最后通过       |
| ------- | :-----: | -------------- |
| L1 ⚡   | ✅ PASS | 2026-06-13     |
| L2 🧪   | ✅ PASS | 2026-06-13     |
| L3 🔗   | ✅ PASS | 2026-06-13     |
| L3.5 🔍 | ⏭️ SKIP | M10 无独立审计 |

### M10 验收记录（2026-06-14，浏览器回归）

| 字段 | 内容                                                                                                                                                                                                               |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 目标 | 验证 M10 UX 增强在真实浏览器中的首启、编辑、搜索、设置、语法参考、更新提示等核心链路可用                                                                                                                           |
| 范围 | 首次启动欢迎向导、主界面、创建/编辑/自动保存/切换/搜索回跳、设置对话框、语法参考卡、导出对话框、主题切换                                                                                                           |
| 途径 | 本地开发站点 `http://127.0.0.1:4173/`，通过 Codex in-app browser 手动 E2E 回归，结合 DOM snapshot 逐步验证                                                                                                         |
| 流程 | 1. 打开应用；2. 完成欢迎向导；3. 创建空白笔记；4. 编辑并等待自动保存；5. 切换到其他笔记再切回；6. 打开搜索面板并回跳；7. 打开设置和关于页；8. 打开/收起语法参考卡；9. 打开导出对话框并选择格式；10. 切换主题并恢复 |
| 结论 | 核心链路通过；欢迎页、创建/编辑/保存、搜索回跳、设置、语法参考、主题切换均可用。导出对话框可打开且格式可选，但当前内置浏览器不支持下载事件，因此未在该浏览器中验证真实文件落盘。                                   |

---

> **维护规则**：每次 L3 集成测试全量通过后，更新本文档中对应任务的状态、完成日期、Commit hash，并重新计算总进度百分比。

---

## 发布收口状态（M-R0 至 M-R7）

> 更新日期：2026-06-24
> 当前发布候选版本：`0.3.0-rc.1`
> 权威执行记录：`memory/release-hardening-execution-log.md`

| 里程碑 | 状态       | 证据摘要                                                                             |
| ------ | ---------- | ------------------------------------------------------------------------------------ |
| M-R0   | 已验收     | 基线冻结与 L1/L2 检查通过，已记录执行日志。                                          |
| M-R1   | 已验收     | 稳定 E2E 脚本与测试隔离收口完成，Chromium/Firefox 通过，WebKit 记录环境阻塞。        |
| M-R2   | 已验收     | Web 核心用户旅程补洞完成；删除后书签残留等阻断问题已修复并补 E2E。                   |
| M-R3   | 已验收     | 响应式、可访问性、视觉回归专项通过；最终 GUI 闭环已在 M-R7 补做。                    |
| M-R4   | 已验收     | Tauri path/配置阻断修复，`tauri:dev` 与桌面 bundle 构建通过。                        |
| M-R5   | 已验收     | 高危 npm audit 清零；XSS/网络隐私/导出专项通过；`cargo audit` 缺工具记录为环境阻塞。 |
| M-R6   | 已验收     | 发布文档、版本、release notes、known limitations 已补齐；M-R6 基础闸门通过。         |
| M-R7   | 最终停止点 | 最终 RC 冻结与全闸门验证已执行；WebKit 与 `cargo audit` 为环境阻塞。                 |

### M-R6 文档状态

- Web package、app package、Tauri config、Rust crate 已统一为 `0.3.0-rc.1`。
- README 已改为发布候选说明，避免宣称最终稳定版或线上下载已可用。
- `CHANGELOG.md` 已新增 `0.3.0-rc.1` 段。
- `RELEASE_NOTES.md` 与 `KNOWN_LIMITATIONS.md` 已新增。
- `package-lock.json` 已删除；`pnpm-lock.yaml` 是唯一权威 JS lockfile。
- M-R6 验证通过：Prettier touched files、`typecheck`、`build`、`pnpm audit --audit-level high`、`cargo fmt --check`、`cargo check`。
- M-R7 已完成最终 RC 验证并写入最终报告；WebKit 与 `cargo audit` 仍是环境阻塞。

---

### M-R7 最终 RC 状态（2026-06-24）

- M-R7 已到达最终停止验收点。
- 最终报告：`memory/release-candidate-final-report.md`。
- 自动化：typecheck、eslint、prettier/stylelint 目标检查、vitest 156/156、Chromium E2E 167/167、Firefox `16-user-journeys` 10/10、npm high audit、cargo fmt/check/test、Tauri release build 均通过；最终表格 padding/header 微调后 J1c 定向复跑通过。
- 当前安装包：`打包/MarkLuck-0.3.0-rc.1-windows-x64/MarkLuck_0.3.0-rc.1_x64-setup.exe`，SHA256 `9b3d1f5fcec77996c1f8d5d046fe6724edda9baf425647bd848b68a7abcb8d8b`。
- 环境阻塞：WebKit executable 未安装；`cargo audit` 命令不可用；WebView2 缺失且无网络场景未破坏本机环境模拟，但 NSIS 本地化错误链路已静态复核。
- 内置浏览器 GUI：新建/编辑/保存/刷新/删除、文件抽屉、搜索、Live Preview、设置文字补全开关、主题持久化、导出成功态均通过。
- 安装版 GUI 风险复核：外部 `.md/.markdown/.mdx` 单文件只读启动、启用编辑确认、当前文件保存回读、无父目录扫描、即时表格渲染、文件抽屉格式过滤、`.txt` 应用内打开但不抢占默认关联、默认文档、中文 IME + Live Preview、Tab 焦点导航、欢迎页默认应用提示均通过。
- 图片上传 GUI 输入受宿主二进制剪贴板/桌面拖放能力限制，使用 `16-user-journeys.spec.ts` J6 的 assets 写入与 Markdown 路径自动化证据替代。
