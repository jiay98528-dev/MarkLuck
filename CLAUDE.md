# JotLuck 项目开发工作流

> 版本：v1.0 | 日期：2026-06-03 | 基于 TeachFlow 实战工作流模板裁剪
> 适用于：单人 + AI 协作的轻量开源项目

---

## ⛔ 工作区隔离

**本项目工作目录**: `D:\VibeCoding\MarkLuck`

禁止访问与 JotLuck 开发无关的系统目录和文件。所有操作限定在本工作目录内。

---

## 一、项目身份

**JotLuck** — 轻量化、本地优先、离线可用的 Markdown 笔记工具。每一条笔记就是一个 `.md` 文件，文件夹即笔记本。数据完全由用户掌控。

**核心闭环**：

```
用户打开本地文件夹 → 浏览/编辑 .md 文件 → 实时 Markdown 渲染预览
    ├── Wiki-link [[...]] 自动关联 → 反向链接面板
    ├── 全文搜索（正则 + 标签 + 日期 + 文件夹过滤）
    ├── 模板系统（{{date}} 等占位符）→ 快速创建笔记
    ├── 导出（PDF / TXT / docx / xlsx+CSV）
    └── 文字补全（幽灵文本预测 + Wiki-link/标签/路径结构化补全）
```

**一句话**: 世界上最好的笔记格式是纯文本。JotLuck 让它好用，但不锁死它。

---

## 二、技术架构（不可变）

```
┌─────────────────────────────────────────┐
│         共享前端 (Vue 3 + Vite + TS)      │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
│  │ marked   │ │highlight │ │DOMPurify│  │
│  │+wiki-link│ │  .js     │ │         │  │
│  └──────────┘ └──────────┘ └─────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
│  │minisearch│ │ docx.js  │ │ sheetjs │  │
│  └──────────┘ └──────────┘ └─────────┘  │
├─────────────────────────────────────────┤
│       Tauri v2 桥接层 (Rust)             │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
│  │文件系统IO│ │ tantivy  │ │ notify  │  │
│  │          │ │ 全文索引  │ │ 文件监控 │  │
│  └──────────┘ └──────────┘ └─────────┘  │
├─────────────────────────────────────────┤
│            部署目标                       │
│  Web PWA │ Desktop(Tauri) │ Mobile(APK) │
└─────────────────────────────────────────┘
```

### 不可变选型

| 层级          | 选型                        | 锁定原因                                                    |
| ------------- | --------------------------- | ----------------------------------------------------------- |
| 前端框架      | **Vue 3** (Composition API) | 生态统一（vue-router + Pinia 均为官方），轻量，减少决策碎片 |
| 构建工具      | **Vite**                    | Vue 官方推荐，HMR 极快                                      |
| 状态管理      | **Pinia**                   | Vue 官方，TS 友好                                           |
| 路由          | **vue-router 4**            | 官方方案                                                    |
| 类型系统      | **TypeScript** (strict)     | 强类型约束，防止运行时崩溃                                  |
| Markdown 渲染 | **marked** + 自定义扩展     | 轻量，扩展性好                                              |
| 代码高亮      | **highlight.js** (按需加载) | 成熟，语言包可控                                            |
| XSS 防护      | **DOMPurify**               | 渲染前清洗，安全底线                                        |
| Web 搜索      | **minisearch**              | 轻量全文检索引擎                                            |
| Tauri 搜索    | **tantivy**                 | Rust 原生，中文分词                                         |
| 文件监控      | **notify** (Rust crate)     | 跨平台文件事件                                              |
| 桌面/Mobile壳 | **Tauri v2**                | 全端统一 Rust 后端                                          |
| 导出 docx     | **docx.js**                 | 纯 JS                                                       |
| 导出表格      | **sheetjs**                 | 纯 JS                                                       |
| 导出 PDF      | **window.print()**          | 零依赖                                                      |

### 明确禁用的技术

| 类别 | 禁止                                                   | 原因                                         |
| ---- | ------------------------------------------------------ | -------------------------------------------- |
| 存储 | 任何数据库（SQLite/IndexedDB/LocalStorage 存笔记内容） | 文件即数据源，数据库会产生同步冲突和数据锁死 |
| 导出 | 任何 PDF 生成库（pdfkit/puppeteer）                    | 增加数 MB 体积，浏览器打印已够用             |
| 格式 | 旧版 `.doc`                                            | 仅支持 docx                                  |
| 功能 | 思维导图                                               | 超出轻量化定位                               |
| 框架 | React / Angular / Svelte                               | 已锁定 Vue 3，避免框架混乱                   |

---

## 三、当前阶段

**Phase 1: Web 先行开发** ← 当前阶段

M0-M6 已完成（总进度 ~90%），当前聚焦 M7 打磨与发布 + M6 Tauri 打包验证。

详见 `spec/milestones.md` 和 `spec/progress.md`

---

## 四、文档索引

### 首次接入必读

| #   | 文档                                        | 内容                                              | 状态 |
| --- | ------------------------------------------- | ------------------------------------------------- | :--: |
| 0   | `PRODUCT.md`                                | 品牌人格、用户画像、设计哲学、反例清单            |  ✅  |
| 1   | `CLAUDE.md`                                 | 本文件 — 项目元指令与开发工作流                   |  ✅  |
| 2   | `doc/PRD.md`                                | 产品需求规格                                      |  ✅  |
| 3   | `doc/TAD.md`                                | 技术架构设计                                      |  ✅  |
| 4   | `spec/decisions.md`                         | 架构决策记录 (ADR)                                |  ✅  |
| 5   | `spec/frontend/design-system.md`            | CSS Design Token 定义（v2.0 Paper/OKLCH ✅）      |  ✅  |
| 6   | `spec/milestones.md`                        | 里程碑拆分与进度（M0-M9）                         |  ✅  |
| 7   | `memory/bug_log.md`                         | 错题本                                            |  ✅  |
| 8   | `packages/app/src/assets/styles/tokens.css` | 实际 Design Token 定义（纸张隐喻/OKLCH/三层动效） |  ✅  |

### 代码规范

| #   | 文档                                 | 内容                                                | 状态 |
| --- | ------------------------------------ | --------------------------------------------------- | :--: |
| 9   | `doc/standards-typescript-vue.md`    | TypeScript & Vue 3 代码规范                         |  ✅  |
| 10  | `doc/standards-css.md`               | CSS & 样式规范（OKLCH/玻璃/动效）                   |  ✅  |
| 11  | `doc/standards-rust.md`              | Rust 代码规范（Tauri 后端）                         |  ✅  |
| 12  | `doc/standards-git.md`               | Git 协作规范（Conventional Commits）                |  ✅  |
| 13  | `doc/standards-theme-development.md` | Theme API v2 主题开发标准（Manifest/Slot/Host API） |  ✅  |

### 动态文档

| #   | 文档               | 内容                              | 状态 |
| --- | ------------------ | --------------------------------- | :--: |
| Δ   | `spec/progress.md` | 进度跟踪（L3 通过后更新，非冻结） |  ✅  |

> **基线冻结**：除 `spec/progress.md` 外，以上全部文档已于 **2026-06-03 冻结为 v1.0 基线**。后续修改需同步更新版本号和变更记录。

### 规格文档

| 领域      | 路径                                 | 说明                            |      状态       |
| --------- | ------------------------------------ | ------------------------------- | :-------------: |
| 产品需求  | `doc/PRD.md`                         | 功能需求、用户场景、验收标准    |       ✅        |
| 技术架构  | `doc/TAD.md`                         | 分层架构、数据流、模块职责      |       ✅        |
| 架构决策  | `spec/decisions.md`                  | 不可逆决策及理由                |       ✅        |
| 设计系统  | `spec/frontend/design-system.md`     | 色彩/字体/间距/圆角/动效 Token  |       ✅        |
| 组件规格  | `spec/frontend/components.md`        | 组件树、Props/Events/Slots 定义 |       ✅        |
| 页面规格  | `spec/frontend/pages.md`             | 路由表、页面状态、数据流        |       ✅        |
| 交互 Mock | `spec/frontend/interactions-mock.md` | Mock 数据与交互场景             | 🔜 编码阶段创建 |
| 共享类型  | `spec/types/`                        | TypeScript 实体类型、接口定义   |       ✅        |
| 文字补全  | `spec/frontend/autocomplete-spec.md` | N-gram/GhostText/基准L2/持久化  |       ✅        |
| 里程碑    | `spec/milestones.md`                 | 各阶段目标、检查点、验收清单    |       🔜        |
| 错题本    | `memory/bug_log.md`                  | BUG 记录 + 根因 + 检查清单      |       ✅        |

### 不需要的文档（明确排除）

JotLuck 无后端/API/数据库，因此以下文档类型**不需要创建**：

- `spec/api/` — 无 REST/GraphQL API
- `spec/error-codes.md` — 离线工具无错误码体系
- `spec/security.md` — 无网络通信、无用户认证
- `spec/release.md` — 简化为 README 的 Release 章节
- `spec/file-manifest.md` — 项目小时不必要，大时自动生成
- `spec/testing-standards.md` — 测试规则已整合到本文档 §5.7

---

## 五、开发工作流规则（强制）

### 5.0.0 自动推进规则

除以下情况外，**不回确认、不暂停、直接推进**：

| #   | 暂停条件       | 说明                                                                                        |
| --- | -------------- | ------------------------------------------------------------------------------------------- |
| 1   | **危险操作**   | 涉及 `git push --force`、删除数据、修改系统配置、`npm unpublish` 等不可逆操作，必须人工确认 |
| 2   | **重大阻塞**   | 依赖的外部服务不可用、权限不足、硬件限制等，需要人工介入解决的                              |
| 3   | **到达检查点** | 里程碑 L4 🔷 复审、L3.5 🔍 审计清零前，需人工确认继续                                       |
| 4   | **上下文腐败** | 执行方向迷失、陷入循环（>3次重复相似操作）、文档约束偏移、目标模糊等，主动暂停汇报          |

上述条件外的任何情况——包括但不限于安装依赖、运行测试、修复编译错误、创建文件、配置调整——均自动推进，不询问"是否继续？""要执行吗？"等确认性问题。

### 5.0.0.1 UI 控件编辑 — Skill 调用规则（强制）

**任何涉及 UI 控件编辑/新增/重设计的任务，必须先调用设计 Skill（`impeccable` 或 `impeccable:impeccable`）进行设计审查或方案输出，再动手编码。**

此规则覆盖以下场景：

| 场景            | 触发条件                                               | 调用的 Skill |
| --------------- | ------------------------------------------------------ | :----------: |
| 组件样式重构    | 修改任何 `.vue` 文件的 `<style>` 块（非 trivial 修复） | `impeccable` |
| 新增 UI 组件    | 创建新的 `.vue` 组件文件                               | `impeccable` |
| 布局调整        | 修改 grid/flexbox 布局参数或区域尺寸                   | `impeccable` |
| 设计 Token 应用 | 将硬编码颜色/间距替换为 CSS 变量                       | `impeccable` |
| 主题效果增强    | backdrop-filter / 阴影 / 光晕 / 动效调整               | `impeccable` |

**执行流程**：

```
收到 UI 编辑任务 → 调用 Skill 输出设计建议 → 评审建议 → 编码实现 → L1 验证
```

**例外**（可跳过 Skill 调用）：

- 仅修改文本内容（label/placeholder/提示语）
- 仅修复 L1 自动检测出的 stylelint 格式问题
- 仅添加/调整 `!important` 或单条 CSS 属性

### 5.0.0.4 Theme API v2 主题开发标准（强制）

任何新增/修改主题、Theme API、UX slot、`ThemeHostContext`、`ThemeDialog`、`ThemeRuntimeHost`、`ThemePackInstaller`、`ThemeRegistry`、主题 CSS 或 `.mltheme` 示例前，必须先阅读并遵守 `doc/standards-theme-development.md`。

硬约束：

- 不得新增未文档化的 `ThemeSlotId`、Host API、Manifest 字段或宿主层 `themeId` 硬编码分支。
- 如确需扩展主题 API，必须在同一变更中同步更新 `doc/standards-theme-development.md`、`packages/app/src/types/theme-pack.ts`、对应 runtime/store/installer 测试和受影响规格文档。
- 本地 `trusted-code` 是全 UX 信任模型，可接管已暴露 UX slot 和执行主题代码；但 Markdown 清洗、文件 IO、搜索索引、导出服务和系统 API 仍必须通过宿主 API 或 action 间接触发。
- 空白缓存草稿不得引入裸 `scratch` slot；必须保持完整工作区路径 `workflow-canvas` + `editor-surface`。

### 5.0 错题本必读（最高优先级）

**任何 Debug/修BUG/编码任务开始前，必须先阅读 `memory/bug_log.md`。**

同类根因的 BUG 不应重复出现。错题本末尾有检查清单，编码前逐条自问。

错题本格式：

```markdown
# JotLuck 错题本

## BUG-001: {标题}

- **现象**: {用户看到的错误表现}
- **根因**: {具体代码位置 + 数据流断裂点}
- **根因类别**: {文件IO / 渲染管线 / 状态管理 / 类型边界 / 跨平台兼容 / ...}
- **修复**: {改了什么，commit hash}
- **教训**: {一句话，可转为检查清单条目}

## 检查清单

- [ ] {条目1}
- [ ] {条目2}
      ...
```

**根因类别分类**（JotLuck 特定）：

- `文件IO` — 文件读写、路径处理、编码、权限
- `渲染管线` — marked 解析、DOMPurify 清洗、highlight.js 应用
- `状态管理` — Pinia store 数据流、组件间通信
- `类型边界` — TypeScript 类型不匹配、null/undefined 处理
- `跨平台兼容` — Web/Desktop/Mobile 行为差异
- `索引/搜索` — 索引构建、搜索查询、分词
- `导出` — docx/xlsx/PDF 生成异常
- `Wiki-link` — 链接解析、反向链接计算

### 5.1 计划先行（强制）

任何功能开发必须遵循以下流程，**禁止无计划直接写代码**：

```
需求理解 → 检查现有规格 → 规格不全则补齐 → 计划确认 → 编码
```

具体步骤：

1. **理解需求** — 明确要做什么、为什么做
2. **检查现有规格** — 查看 `doc/PRD.md`、`spec/frontend/` 是否已覆盖
3. **补齐规格** — 如果规格不全，**先写规格文档，经审核后方可编码**（见 §5.2）
4. **制定计划** — 拆分具体任务、预估影响范围
5. **确认计划** — 与用户对齐后，开始编码

### 5.2 文档先行（Document-First）（强制）

**功能开发前，若该功能的组件规格、页面规格、或类型定义尚未覆盖，必须先创建对应规格文档，审核通过后方可编码。**

禁止的行为：

- ❌ 拿着 PRD 一句话描述直接写代码
- ❌ 边写代码边"补充"规格（规格必须是先于代码的约束，而非代码的说明书）
- ❌ 规格文档中留 TODO 然后开始编码
- ❌ 说"这个简单我先写了再补文档"

规格文档的"审核通过"标准（单人项目适用版）：

- 组件 Props/Events/Slots 定义完整，类型明确
- 页面状态机（Loading / Empty / Error / Normal / Edge Cases）全部覆盖
- 数据流清晰：数据从哪来、经过哪些层、到哪去
- 与已有规格无冲突（交叉引用检查）

### 5.3 强约束开发（Constraint-Driven Development）（强制）

编码时，以下约束为**硬性要求**，违反即视为代码不合格：

| 约束                | 说明                                                            | 检查方式                                              |
| ------------------- | --------------------------------------------------------------- | ----------------------------------------------------- |
| **组件 Props 约束** | 组件 Props/Events/Slots 必须完全匹配 `components.md` 定义       | 对照 spec 逐项检查                                    |
| **样式 Token 约束** | 禁止使用 `tokens.css` / `paper.css` 未定义的色值/字号/间距/圆角 | stylelint `color-no-hex` + `function-disallowed-list` |
| **类型约束**        | 所有函数签名、Store、组件 Props 必须有完整 TypeScript 类型      | `tsc --noEmit` 零错误                                 |
| **渲染安全约束**    | 所有 Markdown 渲染输出必须经过 DOMPurify 清洗                   | 代码审查检查                                          |
| **文件路径约束**    | 所有文件操作必须使用相对路径，禁止硬编码绝对路径                | 代码审查检查                                          |
| **状态机约束**      | 每个页面/组件必须处理 Loading / Empty / Error / Normal 四种状态 | 对照 pages.md 状态定义                                |

### 5.4 前端先行 + Tauri 积木接入

```
Phase 1 (Web 先行):
  前端 UI 按 spec 开发 → Mock 文件系统（内存虚拟文件树）→ 功能完整可演示

Phase 2 (Tauri 接入):
  逐模块替换 Mock 为 Tauri IPC 调用 → 真实文件系统 → 验证行为一致
```

**Mock 文件系统要求**：

- 提供与 Tauri IPC 相同的接口签名
- 模拟文件读写延迟（50-200ms）以暴露竞态条件
- 模拟错误场景（文件不存在、权限不足、磁盘满）
- Mock 与真实实现的切换必须是**单行配置**（环境变量或 feature flag）

### 5.5 开发后自检清单（强制）

编码完成后、提交审查前，逐项确认：

- [ ] **1. 类型检查** — `tsc --noEmit` 零错误，无 `as any` 逃逸（除非有注释说明原因）
- [ ] **2. Lint 检查** — `eslint` 零警告，`prettier` 格式一致
- [ ] **3. 组件约束** — 新增/修改的组件 Props/Events 完全匹配 `components.md`
- [ ] **4. 样式约束** — 未使用 `design-system.md` 之外的色值/字号/间距
- [ ] **5. 状态覆盖** — 页面四种状态（Loading/Empty/Error/Normal）全部有对应 UI
- [ ] **6. 安全检查** — 用户输入的 Markdown 内容经过 DOMPurify 清洗后渲染
- [ ] **7. 文件操作安全** — 文件路径使用相对路径，异常情况有错误处理
- [ ] **8. Mock 一致性** — Mock 数据接口与真实接口签名一致，切换 feature flag 后行为无变化
- [ ] **9. 进度验证** — 每个标记 ✅ 的任务必须有可验证证据：(a) 自动化输出（测试 PASS / 构建产物 / lint 通过），(b) commit message 关键词必须匹配任务描述（禁止用"Parser完成"标记"Widget完成"）。进度标记不可由完成任务者自行判定——需经 L1/L2/L3 自动化闸门或独立审查确认

**通过标准**：9 项全部确认。任何一项不通过 → 修正 → 重新自检 → 全部通过后执行 L1 → L2 → L3。里程碑末额外执行 L3.5 独立审计 + L4 人工复审。

**教训 (2026-06-04)**: M1-08 (BlockWidget) 在第6项(M1 L4)复审时标记为 DEFERRED，但后续被标记为 ✅ 完成——commit message 只提了 BlockParser 和装饰器，从未实现 Widget。进度标记不可自判。此规则今后即第 9 项。

### 5.6 BUG 修复前置规则（强制）

修复任何 BUG 前，必须先完成真/假 BUG 判定。只有根因明确的真 BUG 才能执行修复。

**判定流程**：

```
复现 BUG → 定位根因（具体代码行/数据流断裂点）→ 判定真/假 → 记录根因 → 执行修复
```

**真/假 BUG 分类**：

| 类型       | 判定标准                                                    | 处理方式                              |
| ---------- | ----------------------------------------------------------- | ------------------------------------- |
| **真 BUG** | 当前里程碑交付范围内，存在数据流断裂/交互失效/规格偏离/崩溃 | 记录错题本 → 修复                     |
| **假 BUG** | 依赖的模块/功能在未来里程碑尚未实现导致的现象               | 标注 `DEFERRED` + 记录依赖项 + 不修复 |

**假 BUG 标注格式**：

```
// TODO(M2): DEFERRED — 依赖 M2 的 tantivy 索引才能正确搜索中文
```

### 5.7 五层验证体系（强制）

> 设计原则：最大化 AI 自动验证覆盖，将人工复审压缩到仅剩主观判断项。

```
L1 ⚡ 写时检查     ~10s    每次保存         tsc + eslint + prettier
L2 🧪 组件测试     ~30s    每次提交前       vitest + @vue/test-utils + 快照
L3 🔗 集成测试     ~3min   每次提交/CI      Playwright E2E + XSS套件
   ├─ L3-S1 像素对比  每次提交前       toHaveScreenshot() 自动对比
   ├─ L3-S2 AI视觉分析 里程碑末/UI重构  MCP bailian-vision analyze_image
   └─ L3-S3 手动辅助   L4人工复审时     AI 辅助截图对比
L3.5 🔍 独立审计   ~3min   每个里程碑末     Subagent 四维审计（打破信息茧房）
L4 🔷 人工复审     ~10min  每个里程碑末     手感 / 视觉 / 触控 / 文档可读性
```

---

#### L1 ⚡ 写时检查（每次保存触发）

| 检查项          | 命令                                                            | 通过标准                        |
| --------------- | --------------------------------------------------------------- | ------------------------------- |
| TypeScript 类型 | `npx vue-tsc --noEmit`                                          | 零错误                          |
| ESLint          | `npx eslint packages/app/src/ packages/renderer/src/`           | 零警告                          |
| Prettier        | `npx prettier --check packages/app/src/ packages/renderer/src/` | 格式一致                        |
| Stylelint       | `npx stylelint "packages/app/src/**/*.{vue,css}"`               | 零错误（强制 OKLCH Token 检查） |

**不通过 → 立即修复。禁止跳过 L1 直接声称"完成"。**

---

#### L2 🧪 组件测试（每次提交前触发）

| 检查项   | 命令                        | 通过标准       |
| -------- | --------------------------- | -------------- |
| 单元测试 | `npx vitest run`            | 全量 PASS      |
| 覆盖率   | `npx vitest run --coverage` | 核心模块 ≥ 80% |

**测试策略**：

- **Composables 优先测试**（纯函数，无 DOM，速度最快）— `useMarkdown`、`useSearch`、`useFileSystem` 等
- **Pinia Stores** — 用 `createTestingPinia()` 隔离测试状态变更
- **Vue 组件** — 用 `@vue/test-utils` 挂载 + `toMatchSnapshot()` 快照测试渲染输出
- **Markdown 渲染管线** — 脱离 Vue 直接测 `marked + DOMPurify` 管道（更快更精确）

---

#### L3 🔗 集成测试（CI 自动运行）

**触发条件**：L1 + L2 全部通过后，CI 自动执行。

**Playwright E2E 测试**（Web 构建）：

- 关键用户流程自动化测试
- 三引擎并行：Chromium / Firefox / WebKit
- 网络节流 + 离线模式模拟

**V-Rules（JotLuck 适配版）**：

> 来源：TeachFlow V1-V5 测试规则，裁剪适配为 JotLuck 版本
> 更新：2026-06-08 新增 V6 用户旅程完整性规则

| 规则   | 名称           | 要求                                                         | 反模式                       |
| ------ | -------------- | ------------------------------------------------------------ | ---------------------------- |
| **V1** | 交互正确性     | 验证交互后的**结果**（至少两个结果指标）                     | 仅断言 `element.isVisible()` |
| **V2** | 文件操作验证   | 文件写入后**读取验证内容**，文件删除后**确认不存在**         | 仅确认操作无报错             |
| **V3** | 跨会话持久化   | **6步**：写入 → 导航离开 → 返回 → 验证 → 刷新 → 再验证       | 单程 `goto away → goto back` |
| **V4** | 内容正确性     | 导出/渲染后的内容**必须与源 Markdown 相关**                  | 仅验证"页面不崩溃"           |
| **V5** | 按钮完整性     | 每个 `<button>` 必须**点击并验证可观测结果**                 | 仅验证按钮存在               |
| **V6** | 用户旅程完整性 | 每个核心功能必须有**≥1 个多步骤端到端测试**（≥4 步用户操作） | 仅测试孤立交互，无完整工作流 |

**V6 用户旅程标准模式**（≥4 步真实用户操作链）：

```typescript
// 示例：新建笔记 → 编辑 → 保存 → 删除 完整闭环
// Step 1: 触发新建
await page.locator('.wing-new-btn').click();
await page.locator('.tpl-card.blank-card').click();
// Step 2: 编辑内容
await page.locator('.cm-content').click();
await page.keyboard.type('# 测试笔记\n\n内容行');
// Step 3: 等待自动保存完成
await expect(page.locator('.status-saved')).toBeVisible({ timeout: 10000 });
// Step 4: 验证持久化 — 切换到其他笔记再切回
await page.locator('.wing-bookmark-dot').first().click();
await page.locator(`.wing-bookmark-dot[aria-label="测试笔记"]`).click();
const content = await getEditorContent(page);
expect(content).toContain('# 测试笔记');
// Step 5: 删除
await page.locator('[title="删除笔记"]').click();
await page.locator('.confirm-btn--danger').click();
// Step 6: 确认已删除
await expect(page.locator('.wing-bookmark-dot[aria-label="测试笔记"]')).not.toBeVisible();
```

**V6 强制覆盖清单**（每项必须有 ≥1 个测试）：

| #   | 用户旅程                            | 最少步骤                      | 状态 |
| --- | ----------------------------------- | ----------------------------- | :--: |
| 1   | 新建笔记 → 编辑 → 保存 → 删除       | 新建→键入→等保存→切回→删→确认 |  ⬜  |
| 2   | 文件抽屉 → 展开子目录 → 打开文件    | 开抽屉→点目录→点文件→验内容   |  ⬜  |
| 3   | 搜索 → 查看结果 → 点击跳转 → 编辑   | 搜索→验证结果→点击→验跳转     |  ⬜  |
| 4   | 即时渲染: 预览→点击块→编辑→ESC→预览 | 切模式→验块→点击→编辑→ESC     |  ⬜  |
| 5   | 右键菜单: 重命名/删除               | 右击→重命名→验证→右击→删除    |  ⬜  |
| 6   | 导出选项组合: 选格式→改选项→导出    | 选格式→切换选项→导出→验内容   |  ⬜  |
| 7   | 错误恢复: 保存失败 → 重试           | 注入错误→验提示→恢复→验保存   |  ⬜  |

> **能力边界说明**: 以上 7 项均可在 Web/MockFS 环境下测试，无需 Tauri 运行时。仅真实文件系统操作（系统对话框/回收站）受限于 Playwright 架构，归入 Tauri 桌面端手动测试。

**V3 多步往返标准模式**（适配文件系统）：

```typescript
// Step 1: 写入文件
await page.getByLabel('文件名').fill('test-note.md');
await page.getByLabel('内容').fill('# Hello World');
await page.getByText('保存').click();
await expect(page.getByText('保存成功')).toBeVisible();

// Step 2-3: 导航离开再返回
await page.getByText('所有笔记').click();
await page.waitForLoadState('networkidle');
await page.getByText('test-note.md').click();
await page.waitForLoadState('networkidle');

// Step 4: 第一轮验证（内存状态）
const content = await page.locator('.editor-content').textContent();
expect(content).toContain('Hello World');

// Step 5: 刷新（验证文件系统持久化）
await page.reload();
await page.waitForLoadState('networkidle');
await page.getByText('test-note.md').click();

// Step 6: 第二轮验证（文件系统持久化）
const content2 = await page.locator('.editor-content').textContent();
expect(content2).toContain('Hello World');
```

**XSS 安全测试套件**（CI 每次运行）：

- 参数化 Payload 库覆盖已知绕过向量（`<script>`、`onerror`、`javascript:` URL、HTML 实体编码、mXSS 变体）
- DOMPurify 版本锁定 + `npm audit` 阻断高危漏洞

**视觉回归测试**（三层体系）：

#### L3-S1 ⚡ 像素级自动对比（每次提交前）

- Playwright `toHaveScreenshot()` 对比关键页面截图
- 基线存储在 git 中，CI 检测差异
- 阈值配置：`maxDiffPixelRatio: 0.01`（1% 像素差异容限）

#### L3-S2 🧠 AI 视觉分析（每个里程碑末 + 每次 UI 重构后）

**前置条件**: `bailian-vision` MCP 服务器已加载（百炼视觉，支持 Qwen-VL 模型退化链）

**工具**: `analyze_image({ image_path: "绝对路径", prompt: "分析提示词" })`

**标准检查点**（定义于 `e2e/helpers/screenshot-utils.ts:STANDARD_CHECKPOINTS`）：

| #   | 检查点                | 验证内容                                                         |
| --- | --------------------- | ---------------------------------------------------------------- |
| 1   | `app-shell-initial`   | 三区布局、纸张暖色背景、书签圆点、TopBar、无横向溢出             |
| 2   | `editor-with-content` | CM6 编辑器、分栏预览、标题层级、代码高亮、状态栏、Wiki-link 样式 |
| 3   | `paper-shell`         | 羽翼布局三栏一致性、代码高亮、对比度达标                         |
| 4   | `template-dialog`     | 模态框居中+遮罩、模板卡片排列、羽翼阴影、纸张表面 Token          |
| 5   | `search-palette`      | 命令面板居中、输入框自动聚焦、placeholder 可见、面板浮起阴影     |
| 6   | `export-dialog`       | 格式卡片布局、Toggle 开关可见、选中状态反馈、按钮可点击          |

**执行流程**：

```
1. Playwright 在关键检查点自动截图 → packages/app/test-results/screenshots/
2. AI 通过 MCP analyze_image 逐张分析截图
3. AI 对比预期视觉特征，输出 PASS / FAIL / WARN 判定
4. FAIL 项记录错题本 → 修复 → 重新截图 → 重新分析
5. 全部 PASS 后，将截图归档为基线参考
```

**分析 Prompt 模板**（自动生成，定义于 `screenshot-utils.ts:buildAnalysisPrompt`）：

```
你是 JotLuck 笔记应用的视觉回归测试专家。请分析这张截图 (检查点: {name})。

预期视觉特征：{expectedVisuals}

请逐项检查并报告：
1. 布局是否正确（三区布局：左翼书签栏 / 中央编辑区 / 右翼面板）
2. 纸张主题 Token 是否正确应用（暖纸背景 + 墨色文字 + 冷蓝强调色）
3. 是否有元素重叠、遮挡、溢出或错位
4. 文字是否清晰可读，间距是否合理
5. 与预期视觉特征的偏差

输出格式：
- PASS: 说明通过的检查项
- FAIL: 说明失败项及具体位置
- 总结：整体评估 (PASS / FAIL / WARN)
```

#### L3-S3 🔄 手动截图对比（L4 人工复审辅助）

- 在关键页面手动截取全屏截图
- 使用 `analyze_image` 与基线进行 AI 辅助对比
- 关注主观审美：信息密度、留白舒适度、色彩和谐度

**依赖安全检查**：

- `npm audit --audit-level=high` — 高危漏洞阻断 CI
- `cargo audit` — Rust 依赖漏洞（M4+）
- Dependabot — 自动 PR 更新漏洞依赖

**Bundle 体积监控**：

- `size-limit` — 主 bundle 超限阻断 CI（JS ≤ 30MB gzip, CSS ≤ 30MB gzip）

**L3 最后一步 — 维护进度跟踪**：

- 所有 L3 检查通过后，**必须更新 `spec/progress.md`**
- 更新内容：已完成任务标记 ✅ + 完成日期 + Commit hash
- 重新计算并更新当前里程碑和总进度百分比
- 如 L3.5 审计已执行，补充审计摘要
- 如 L4 复审已通过，补充复审记录

**JotLuck 关键测试场景**（L2 + L3 必须覆盖）：

| 场景类别          | 测试用例                                                                    |      测试层       |  对应规则  |
| ----------------- | --------------------------------------------------------------------------- | :---------------: | :--------: |
| **Markdown 渲染** | 代码块高亮、表格、LaTeX 公式、嵌套列表、图片引用                            | L2 快照 + L3 截图 |   V1, V4   |
| **XSS 防护**      | 已知绕过向量注入 → DOMPurify 清洗后不含恶意代码                             |        L2         |     —      |
| **文件读写**      | 创建→保存→重新打开→内容一致；删除→确认文件消失                              |        L3         |   V2, V3   |
| **外部编辑**      | JotLuck 打开文件 → 外部编辑器修改 → JotLuck 检测变更并刷新                  |        L3         |     V1     |
| **Wiki-link**     | 创建 `[[其他笔记]]` → 渲染为链接 → 点击跳转 → 反向链接面板显示              |      L2 + L3      |   V1, V4   |
| **搜索**          | 中文搜索、英文搜索、正则搜索、标签过滤、日期范围过滤                        |      L2 + L3      |   V1, V4   |
| **导出**          | 导出 docx → 解析回读验证；xlsx → sheetjs 回读 cell 级对比；PDF 文本提取对比 |        L2         |   V2, V4   |
| **模板**          | 使用模板创建笔记 → `{{date}}` 正确替换 → 内容保存正确                       |      L2 + L3      |   V1, V4   |
| **跨浏览器**      | Chromium / Firefox / WebKit 三引擎并行运行相同测试                          |        L3         |     —      |
| **用户旅程(1)**   | 新建空白笔记 → 键入内容 → 等待自动保存 → 切换后切回验证 → 删除确认          |        L3         | V6, V2, V3 |
| **用户旅程(2)**   | 文件抽屉打开 → 展开子目录 → 选择文件 → 验证编辑器内容                       |        L3         |   V6, V1   |
| **用户旅程(3)**   | Ctrl+K 搜索 → 验证结果 → 点击跳转 → 编辑器内容匹配                          |        L3         | V6, V1, V4 |
| **用户旅程(4)**   | 即时渲染: 切换到live → 验证渲染块 → 点击块编辑 → ESC恢复渲染                |        L3         |   V6, V1   |
| **用户旅程(5)**   | 右键文件 → 重命名 → 验证新名称 → 右键删除 → 确认消失                        |        L3         |   V6, V2   |
| **用户旅程(6)**   | 导出对话框: 选HTML格式 → 关Wiki链接 → 导出 → 下载内容验证                   |        L3         |   V6, V4   |
| **用户旅程(7)**   | 模拟保存失败 → 验证错误提示 → 恢复 → 重试保存成功                           |        L3         |   V6, V1   |

---

#### L3.5 🔍 独立审计环（每个里程碑末，L3 通过后触发）

**目的**：打破单上下文持续开发产生的信息茧房。启动一个**全新上下文的 Subagent**，以"攻击者"视角审查本里程碑的全部代码和规格文档。

**审计维度**（四维全覆盖）：

| 维度                    | 审查内容                                                                                                                                                                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. 规格一致性**       | 组件 Props/Events 是否完全匹配 `components.md`；样式是否只用 Paper Token（`tokens.css` / `paper.css` 中定义的 CSS 变量，纯 OKLCH，无硬编码色值）；页面状态机是否完整                                                                        |
| **2. 已知易错点清单**   | 逐项对照 §5.9 的 JotLuck 特定易错点：文件编码、路径分隔符、大文件卡死、XSS 注入、Wiki-link 死链、循环引用、并发写入冲突、索引不一致、Overlay 遮挡、CSS 类名不一致、E2E 选择器过时。同时执行 L3-S2 AI 视觉分析（如 bailian-vision MCP 可用） |
| **3. 类型边界与数据流** | TypeScript 类型安全、null/undefined 处理、异常捕获完整性；渲染管线 → Pinia Store → 文件 IO 的数据流是否完整无断裂                                                                                                                           |
| **4. 跨平台兼容**       | Web (Chrome + Firefox) / Tauri Desktop / Mobile 行为差异；PWA 离线兼容；File System Access API 降级路径                                                                                                                                     |

**L3.5 不可跳过的情况**（任一条件满足即必须执行）：

| 条件                                     | 原因                                         |
| ---------------------------------------- | -------------------------------------------- |
| 里程碑包含任何 `.vue` 或 `.css` 文件变更 | 样式 Token 违规只有全新上下文能发现          |
| 里程碑新增组件 ≥ 3 个                    | 组件 Props/Events 与 spec 一致性需要独立核验 |
| 里程碑涉及 Pinia Store 或数据流变更      | 状态管理错误是最高发类别                     |
| 里程碑包含任何共享类型 (`types/`) 变更   | 类型错误会级联影响                           |

可跳过（需明确说明理由）：

- 仅配置文件变更（CI, package.json, tsconfig 等）
- 仅文档更新（`spec/`, `doc/`, README 等）

**教训 (2026-06-04)**: M2 以"规模可控"、M5 以"纯样式里程碑"为由跳过了 L3.5。结果 M2 的 10+ 组件硬编码色值完全绕过了 M5 的主题系统，直到 L4 人工复审才发现。L3.5 是防止信息茧房的唯一闸门，跳过条件必须严格。

**审计执行流程**：

```
1. L3 全量通过 → 确认 CI 绿色
2. 启动独立 Subagent（全新上下文，无对话历史）
   - 子Agent 读取：CLAUDE.md + 本里程碑所有 spec 文档 + 全部代码变更
   - 子Agent 以"攻击者/质疑者"视角逐维审查
   - 子Agent 输出结构化审计报告（发现列表 + 严重度 + 位置）
3. 主 Agent 逐条判定审计发现：
   - 真 BUG → 记录错题本 → 修复 → 重新 L1 → L2 → L3
   - 假阳性 → 注明判定理由，不修复
   - DEFERRED → 标注依赖里程碑，不修复
4. 审计报告所有条目处理完毕 → 进入 L4 人工复审
```

**审计报告格式**：

```markdown
## L3.5 审计报告 — M{n} {里程碑名称}

### 审计结果汇总

- 审查文件数：{n}
- 发现问题数：{n}（严重{n} / 一般{n} / 建议{n}）

### 发现清单

| #   | 严重度  | 维度     | 位置                        | 描述                            | 判定         |
| --- | :-----: | -------- | --------------------------- | ------------------------------- | ------------ |
| 1   | 🔴 严重 | 类型边界 | `src/stores/note.ts:42`     | saveNote 未处理磁盘满异常       | 真BUG→修复   |
| 2   | 🟡 一般 | 跨平台   | `src/utils/path.ts:15`      | 路径拼接在 Windows 上使用了 `\` | 真BUG→修复   |
| 3   | 🟢 建议 | 规格一致 | `src/components/Editor.vue` | 缺少 Empty 状态 UI              | DEFERRED(M2) |

### 审计结论

- [ ] 所有严重/一般问题已修复并重跑 L1-L3
- [ ] 建议项已标记 DEFERRED 或记录原因
- [ ] 审计通过，进入 L4
```

---

#### L4 🔷 人工复审（每个里程碑末，L3.5 通过后触发）

**这是最终质量闸门。每个里程碑 🔷 到达时，暂停开发，呼叫用户手工测试。禁止跳过。**

**里程碑将在 `spec/milestones.md` 中详细定义，届时每个里程碑的复审清单从 milestones.md 提取。**

**人工复审只关注 AI 无法自动判断的 4 件事**：

| 复审项   | 说明                                               | AI 为什么做不到    |
| -------- | -------------------------------------------------- | ------------------ |
| **手感** | 编辑器光标行为、输入延迟、拖拽流畅度、快捷键直觉性 | 主观体验，无法量化 |
| **视觉** | 色彩和谐度、信息密度、留白舒适度、渲染美观度       | 审美判断           |
| **触控** | 移动端手势、长按、触控区域大小（M4+）              | 物理操作体验       |
| **文档** | README/帮助文档的可读性和信息传达效果              | 面向人的沟通质量   |

**复审执行流程**：

```
1. 到达 🔷 检查点 → 确认 L1/L2/L3 全部通过 + L3.5 审计报告清零
2. 确认所有已知 BUG 已修复或标记 DEFERRED
3. 确认 spec 文档与代码实现一致（§5.8 文档同步检查）
4. 准备复审清单（从 milestones.md 提取验收项 + 4 项人工判断维度）
5. 通知用户开始手动测试
6. 用户测试期间，记录所有发现的问题
7. 测试结束后，逐条判定真/假 BUG（§5.6）
8. 真 BUG → 记录错题本 → 修复 → 重新 L1 → L2 → L3 → 通知用户复测
9. 假 BUG → 标注 DEFERRED + 依赖里程碑
10. 用户确认通过 → 标记里程碑完成 → 推进下一里程碑
```

**复审期间规则**：

- **禁止编辑代码**，除非是修复复审中发现的真 BUG
- 所有发现的问题必须记录（包括被认为"不是问题"的）
- BUG 修复后必须重新通过 L1 → L2 → L3 全链，确认不引入回归

**自动化验证 vs 人工复审的能力边界**：

| 自动验证（L1/L2/L3/L3.5 已覆盖） | 人工判断（L4 关注）               |
| -------------------------------- | --------------------------------- |
| TypeScript 类型正确性            | 编辑器**操作手感**是否流畅        |
| ESLint/Prettier 规范             | Markdown 渲染**视觉效果**是否美观 |
| 单元测试 + 快照测试通过          | 整体**信息密度**和**视觉舒适度**  |
| Playwright E2E 全量 PASS         | 移动端**触控体验**（M4+）         |
| XSS Payload 安全套件全量 PASS    | README / 帮助文档**可读性**       |
| 三引擎跨浏览器截图对比           | —                                 |
| Bundle 体积 ≤ 上限               | —                                 |
| 文件读写回环验证正确             | —                                 |
| 导出内容结构化对比一致           | —                                 |
| 搜索精确率/召回率达标            | —                                 |
| L3.5 四维审计清零                | —                                 |

### 5.8 文档防腐败（强制）

代码变更后，必须同步更新受影响的规格文档：

| 变更类型                                  | 需同步的文档                                                      |
| ----------------------------------------- | ----------------------------------------------------------------- |
| 新增/修改组件                             | `components.md`、`types/`                                         |
| 新增/修改页面                             | `pages.md`、`interactions-mock.md`                                |
| 样式变更                                  | `design-system.md`、实际 Token 文件（`tokens.css` / `paper.css`） |
| 主题 API / slot / manifest / runtime 变更 | `doc/standards-theme-development.md`、`components.md`、`TAD.md`   |
| 架构决策变更                              | `decisions.md`（新增 ADR）                                        |
| 依赖变更                                  | `TAD.md`                                                          |
| 功能范围变更                              | `PRD.md`、`milestones.md`                                         |

**检查方法**：每次 L4 人工复审前，执行文档同步检查：

1. 列出本次里程碑所有代码变更文件
2. 对照上表，逐一确认受影响的文档已更新
3. 如有未更新的文档，先补文档再进入 L4

### 5.9 代码审查要点

AI 编码时，额外注意以下 JotLuck 特定的易错点：

| 易错点                       | 说明                                                                                                 | 预防措施                                                                                                                               |
| ---------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **文件编码**                 | Windows 中文路径 + UTF-8 BOM 问题                                                                    | 读写文件显式指定 UTF-8，测试涵盖中文文件名                                                                                             |
| **路径分隔符**               | Windows `\` vs Unix `/`                                                                              | 统一使用 `/`，Tauri 侧处理转换                                                                                                         |
| **大文件渲染**               | >1MB 的 Markdown 卡死渲染线程                                                                        | 设置文件大小上限（建议 5MB），超大文件用分块渲染                                                                                       |
| **XSS 注入**                 | Markdown 中的 `<script>` 标签                                                                        | DOMPurify 必须在 marked 之后、DOM 插入之前执行                                                                                         |
| **Wiki-link 死链**           | 链接到不存在的笔记                                                                                   | 死链用不同样式标注，不报错崩溃                                                                                                         |
| **循环引用**                 | A → B → A 的反向链接                                                                                 | 反向链接计算检测循环，不做无限递归                                                                                                     |
| **索引重建**                 | .jotluck_index.json 与实际文件不一致                                                                 | 文件监控检测到变更 → 增量更新索引；提供"重建索引"按钮                                                                                  |
| **并发写入**                 | 外部编辑器 + JotLuck 同时保存                                                                        | 保存前检查文件 mtime，如外部已修改则提示冲突                                                                                           |
| **Overlay 遮挡**             | 抽屉/模态框选择后不关闭，`v-if` overlay 永久阻挡下层点击                                             | 改变全局 UI 状态的交互在完成使命后必须关闭；文件选择 → `showLeftDrawer = false`；模板/导出 → `showXxx = false`                         |
| **CSS 类名不一致**           | 渲染器输出类名与 CSS 选择器命名不同步（如 `wikilink` vs `wiki-link`）                                | 重构改类名时必须 grep 全局所有引用点：渲染器输出 + CSS 定义 + E2E 选择器三方一致                                                       |
| **E2E 选择器过时**           | UI 重构后按钮/文件树等 CSS 类名变更，E2E 测试选择器未同步更新                                        | 重构完成后必须 grep 所有 `.spec.ts` 中的 CSS 选择器，逐条验证在源码中仍存在                                                            |
| **Vue scoped keyframes**     | `<style scoped>` 中 `@keyframes` 名称被追加 `-data-v-xxx` hash 后缀                                  | JS 侧检查 CSS 规则时用 `startsWith(name)` 而非 `=== name`                                                                              |
| **`:key` + async onMounted** | Editor 组件 `:key="activePath"` 重建时 `onMounted(async)` 中 `view=null` 窗口期导致轮询读到空/旧内容 | 考虑预加载（父组件提前 fetch）、暴露 ready promise、或用 `v-show` + 手动重置替代 `:key`                                                |
| **CM6 decorations 异步更新** | ViewPlugin 在 `setTimeout`/`Promise` 回调中修改 `this.decorations` 不会自动触发 DOM 重绘             | 修改后必须调用 `view.dispatch({})` 触发空事务（非 `requestMeasure()`—后者只清除不设置）。CM6 只在 update cycle 中读取 decorations 字段 |
| **CM6 keymap 优先级**        | 自定义 Tab handler 注册在 `defaultKeymap`（含 `indentWithTab`）之后，Tab 被缩进逻辑先消费            | 将自定义 keymap extension 放在 `jotluckExtensions()` 之前注册。CM6 keymap 按注册顺序执行，先注册先匹配                                 |
| **模块级单例 + `:key`**      | 组件因 `:key` 重建时，模块级变量被新实例覆盖。Tab keymap 通过全局变量查找 plugin 可能匹配到错误实例  | 通过闭包参数直接传递 plugin spec，不使用模块级 `let` 变量                                                                              |

---

## 六、设计系统与主题系统

> 2026-06-27：主题系统已升级为 Theme API v2 本地 UX Theme Plugin 架构。`paper` 是默认官方主题和回退基线，不是单主题上限。主题开发边界以 `doc/standards-theme-development.md` 为准。
> `spec/frontend/design-system.md` 使用旧 hex 色值体系，**实际权威来源是以下代码文件与主题标准**：

### 代码层 Token 文件

| 文件                                               | 内容                                                           |
| -------------------------------------------------- | -------------------------------------------------------------- |
| `packages/app/src/assets/styles/tokens.css`        | 共享 Token：字体/间距/Z-Index/三层动效体系                     |
| `packages/app/src/assets/styles/themes/paper.css`  | Paper 布局：暖纸配色，纯 OKLCH 色彩                            |
| `packages/app/src/assets/styles/accessibility.css` | 无障碍：focus-ring / prefers-reduced-motion / prefers-contrast |
| `packages/app/src/assets/styles/main.css`          | 入口 + 全局 reset                                              |
| `doc/standards-theme-development.md`               | Theme API v2：Manifest、Slot、Host API、`.mltheme`、商业化接口 |

### 设计哲学

**默认隐喻**：日本和纸 + iA Writer 专注感。暖调米白底，墨色文字，单一冷调强调色。
UI 退后，内容浮现。其他主题可以在 Theme API v2 暴露的 UX slot 内实现完全不同的 Shell 级体验。

**色彩空间**：**纯 OKLCH**（禁止 hex/rgb/hsl）。Token 命名以纸张/墨水为隐喻：

- `--paper-*` — 纸面层级（bg/surface/raised）
- `--ink-*` — 墨色文字层级（primary/secondary/muted）
- `--accent` — 单一冷蓝强调色
- `--rule` — 分隔线
- `--signal-*` — 语义色（success/warning/error）

**动效**：三层体系

- Tier 1 Tactile (80-120ms): 按钮按压、hover 切换
- Tier 2 Spatial (250-400ms): 面板展开、页面切换
- Tier 3 Ambient (1.5-3s): 骨架屏 shimmer、焦点呼吸

**样式 Token 约束**：所有色值/字号/间距/圆角必须引用 `tokens.css` 或 `paper.css` 中定义的 CSS 变量。**禁止硬编码**。`stylelint` 配置了 `color-no-hex` 规则强制此约束。

---

## 七、架构决策记录 → `spec/decisions.md`

所有不可逆的架构决策必须记录，格式遵循 ADR：

```markdown
## ADR-{编号}: {标题}

- **状态**: {提议/已采纳/已废弃}
- **日期**: {YYYY-MM-DD}
- **背景**: {为什么需要做决策}
- **决策**: {选择了什么方案}
- **后果**: {正面 + 负面后果}
- **替代方案**: {考虑过但未选的方案及原因}
```

### 已确认的关键决策速查

| ADR     | 决策                                    |  状态  |
| ------- | --------------------------------------- | :----: |
| ADR-001 | Vue 3 而非 React                        | 已采纳 |
| ADR-002 | Tauri v2 而非 Electron                  | 已采纳 |
| ADR-003 | 文件架构而非数据库                      | 已采纳 |
| ADR-004 | marked 而非 markdown-it                 | 已采纳 |
| ADR-005 | PDF 用浏览器打印而非 pdfkit             | 已采纳 |
| ADR-006 | YAML frontmatter + #tag 双标签语法      | 已采纳 |
| ADR-007 | Wiki-link `[[]]` 语法支持               | 已采纳 |
| ADR-008 | Tauri v2 统一全端（桌面+移动）          | 已采纳 |
| ADR-009 | Paper 纸张隐喻默认主题（替换构成+玻璃） | 已采纳 |

---

## 八、启动命令

```bash
cd D:\VibeCoding\MarkLuck

# 开发环境
npm run dev              # Vite 开发服务器 :5173 (packages/app)
npm run tauri dev        # Tauri 桌面开发 (packages/app)

# 静态检查（路径指向 monorepo 子包）
npx vue-tsc --noEmit     # TypeScript 类型检查
npx eslint packages/app/src/ packages/renderer/src/  # ESLint
npx prettier --check packages/app/src/ packages/renderer/src/  # Prettier 格式检查
npx stylelint "packages/app/src/**/*.{vue,css}"  # Stylelint（强制 OKLCH Token 检查）

# 测试
npx vitest run           # 单元测试 (packages/app, packages/renderer)
npx playwright test      # E2E 测试 (packages/app)

# 构建
npm run build            # Vite 生产构建
npm run tauri build      # Tauri 桌面打包
```

---

## 附录 A：开发决策树

```
收到开发任务
    │
    ├─→ 读错题本 memory/bug_log.md (§5.0)
    ├─→ 读对应 spec/ 规格文档 (§5.2)
    ├─→ 规格不全? → 写规格 → 审核 → 通过后继续 (§5.2)
    │
    ▼
编码 (§5.3 约束驱动 + §5.4 前端先行)
    │
    ▼
每次保存: L1 ⚡ tsc + eslint + prettier (§5.7)
    ├─ FAIL → 修复 → 重新 L1
    └─ PASS ↓
    │
    ▼
每次提交前: L2 🧪 vitest + 快照 (§5.7)
    ├─ FAIL → 修复 → 重新 L1
    └─ PASS ↓
    │
    ▼
每次提交/CI: L3 🔗 Playwright E2E + XSS + 截图 + audit + size-limit (§5.7)
    ├─ FAIL → 修复 → 重新 L1 → L2 → L3
    └─ PASS → 更新 spec/progress.md（标记任务✅+日期+Commit）↓
    │
    ▼
§5.5 自检清单 (9项)
    ├─ FAIL → 修复 → 重新 L1
    └─ PASS ↓
    │
    ▼
是 🔷 里程碑?
    │
    NO ──→ 文档同步检查 (§5.8) → ✅ 任务完成
    │
    YES ↓
    │
L3-S2 🧠 AI 视觉分析 — 截图 + MCP analyze_image (§5.7 L3-S2)
    ├─ FAIL → 记录错题本 → 修复 → 重新截图 → 重新分析
    └─ PASS ↓
    │
L3.5 🔍 独立 Subagent 四维审计 (§5.7)
    ├─ 真BUG → 记录错题本 → 修复 → L1 → L2 → L3
    ├─ 假阳性 → 记录原因
    └─ 审计清零 ↓
    │
文档同步检查 (§5.8)
    ├─ 文档滞后 → 补文档
    └─ 同步通过 ↓
    │
L4 🔷 人工复审 (§5.7)
    ├─ 准备复审清单 (从 milestones.md 提取)
    ├─ 通知用户手动测试 (手感/视觉/触控/文档)
    ├─ 收集问题 → 真/假BUG判定 (§5.6)
    ├─ 真BUG → 记录错题本 → 修复 → L1 → L2 → L3 → L3.5 → 复测
    └─ 用户确认通过 ↓
    │
标记里程碑完成 → 推进下一里程碑
```

---

## 附录 B：目录结构

```
JotLuck/
├── CLAUDE.md                    # 本文件 — 项目元指令与开发工作流
├── README.md                    # 项目介绍（中英文）
├── PRODUCT.md                   # 产品品牌与设计哲学
├── CHANGELOG.md                 # 版本变更记录
│
├── spec/                        # 规格文档（先于代码存在）
│   ├── decisions.md             # 架构决策记录 (ADR)
│   ├── milestones.md            # 里程碑拆分与进度
│   ├── progress.md              # 进度跟踪（动态文档）
│   ├── frontend/
│   │   ├── design-system.md     # 设计 Token 定义（⚠ 待更新至 Paper/OKLCH 体系）
│   │   ├── components.md        # 组件 Props/Events/Slots 规格
│   │   ├── pages.md             # 页面路由与状态规格
│   │   └── interactions-mock.md # 交互 Mock 数据
│   └── types/
│       ├── index.ts             # 共享类型导出
│       ├── note.ts              # 笔记实体类型
│       └── search.ts            # 搜索相关类型
│
├── doc/                         # 设计文档
│   ├── PRD.md                   # 产品需求规格
│   ├── TAD.md                   # 技术架构设计
│   ├── standards-theme-development.md # Theme API v2 主题开发标准
│   └── standards-*.md           # 代码规范（TS/Vue/CSS/Rust/Git）
│
├── memory/                      # 项目记忆
│   └── bug_log.md               # 错题本（BUG + 根因 + 检查清单）
│
├── packages/                    # pnpm monorepo 工作区
│   ├── app/                     # 主应用 (Vue 3 + Vite + TS)
│   │   ├── src/
│   │   │   ├── main.ts          # 应用入口
│   │   │   ├── App.vue          # 根组件
│   │   │   ├── components/      # Vue 组件
│   │   │   │   ├── common/      # 通用 UI 组件 (Button/ContextMenu/Toast)
│   │   │   │   ├── editor/      # 编辑器组件 (MarkdownEditor/FormatBubble/TopBar...)
│   │   │   │   ├── file-tree/   # 文件树组件 (Breadcrumb...)
│   │   │   │   ├── layout/      # 布局组件 (AppShell/LeftWing/RightWing...)
│   │   │   │   ├── modals/      # 对话框组件 (Export/Share/Template/Settings...)
│   │   │   │   └── overlays/    # 浮层组件 (FileDrawer/CommandPalette/MarkdownCheatSheet...)
│   │   │   ├── pages/           # 页面组件
│   │   │   ├── stores/          # Pinia 状态管理
│   │   │   ├── services/        # 业务服务 (MockFS/Exporter/Search...)
│   │   │   ├── composables/     # Vue Composables
│   │   │   ├── utils/           # 工具函数 (CM6扩展/N-gram引擎/GhostText...)
│   │   │   ├── types/           # TypeScript 类型定义
│   │   │   └── assets/styles/   # CSS 样式
│   │   │       ├── tokens.css   # 共享 Design Token（字体/间距/Z-Index/动效）
│   │   │       ├── main.css     # 全局样式入口
│   │   │       ├── accessibility.css  # 无障碍样式
│   │   │       ├── editor.css   # 编辑器样式（含 ghost text）
│   │   │       ├── dialog.css   # 对话框/模态框样式
│   │   │       └── themes/
│   │   │           └── paper.css # Paper 主题（亮+暗，纯 OKLCH）
│   │   ├── src-tauri/           # Tauri v2 Rust 后端
│   │   │   ├── Cargo.toml
│   │   │   └── src/
│   │   │       ├── main.rs
│   │   │       ├── lib.rs
│   │   │       ├── fs_ops.rs    # 文件系统操作
│   │   │       ├── indexer.rs   # tantivy 全文索引
│   │   │       ├── file_watcher.rs  # notify 文件监控
│   │   │       ├── template.rs  # 模板引擎
│   │   │       └── path.rs      # 路径安全校验
│   │   ├── package.json
│   │   ├── vite.config.ts
│   │   └── playwright.config.ts
│   ├── renderer/                # Markdown 渲染库 (@jotluck/renderer)
│   │   └── src/
│   │       ├── index.ts         # renderMarkdown() 导出
│   │       ├── marked-extensions.ts  # Wiki-link / #tag 扩展
│   │       ├── sanitize.ts      # DOMPurify 封装
│   │       └── highlight.ts     # highlight.js 封装
│   └── vscode-ext/              # VS Code 扩展（M9 可选阶段）
│       └── src/
│           └── extension.ts
│
├── e2e/                         # E2E 测试
│   └── tests/
│       ├── 01-editor-core.spec.ts
│       ├── 02-file-operations.spec.ts
│       ├── 03-search.spec.ts
│       ├── 04-settings-panels.spec.ts
│       ├── 05-export-share.spec.ts
│       ├── 06-security.spec.ts
│       ├── 07-persistence.spec.ts
│       ├── 08-edge-cases.spec.ts
│       ├── 09-wiki-link.spec.ts
│       ├── 10-templates.spec.ts
│       ├── 11-onboarding.spec.ts
│       ├── 12-cheatsheet.spec.ts
│       ├── 13-toast.spec.ts
│       └── 14-live-preview-journey.spec.ts
│
├── .github/workflows/           # CI 配置
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.json
```

---

## 附录 C：里程碑检查清单模板

以下模板用于每个里程碑复审时生成具体的检查清单：

```markdown
## M{n} 🔷 复审清单 — {里程碑名称}

**日期**: YYYY-MM-DD
**前置条件**: L1 全部通过 | 已知 BUG 全部修复或 DEFERRED | 文档同步检查通过

### 功能验收项

- [ ] {验收项1}
- [ ] {验收项2}
- [ ] ...

### 边界情况

- [ ] 空文件夹
- [ ] 超大文件 (>5MB)
- [ ] 中文文件名/路径
- [ ] 特殊字符（emoji, 符号）
- [ ] 并发操作
- [ ] 网络断开（PWA）

### 跨平台兼容（M4+）

- [ ] Chrome/Edge (Windows)
- [ ] Firefox (Windows)
- [ ] Safari (macOS) — 如适用
- [ ] Tauri Desktop (Windows)
- [ ] Tauri Mobile (Android)

### 用户反馈

| #   | 问题描述 | 真/假BUG | 处理 |
| --- | -------- | :------: | ---- |
| 1   |          |          |      |
| 2   |          |          |      |

### 结论

- [ ] 全部通过，推进下一里程碑
- [ ] 存在问题，修复后复测
```
