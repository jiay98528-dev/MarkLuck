# JotLuck 错题本

> 版本：v1.0 | 创建日期：2026-06-03
> 同类根因的 BUG 不应重复出现。任何 Debug/修BUG/编码任务开始前，必须先阅读本文档。

---

## BUG 记录

## BUG-001: 正则搜索 `/pattern/` 无结果返回

- **现象**: 搜索框输入 `/快速/` 无任何匹配结果，输入 `快速`（不带斜杠）正常
- **根因**: `SearchEngine.ts:84-88` — `parseAdvancedQuery` 匹配正则语法后将 `text` 清空，`search()` 中 `parsed.text.trim()` 为空 → `results = []`，正则过滤在空集上执行
- **根因类别**: 索引/搜索
- **修复**: 新增 `getAllDocResults()` 方法；当仅有过滤条件（正则/标签/日期/文件夹）而无文本搜索词时，先获取全部文档再执行过滤
- **教训**: 过滤条件与全文搜索是叠加关系而非替代关系。任何过滤链的起点必须是全集或全文搜索结果，不能是空集

## BUG-002: Ctrl+Shift+F 搜索快捷键与中文输入法简繁切换冲突

- **现象**: 中文输入法激活时按 Ctrl+Shift+F 被 IME 拦截为简繁切换，搜索面板无法弹出
- **根因**: Windows 中文输入法（微软拼音/搜狗等）默认将 Ctrl+Shift+F 绑定为简繁切换热键，优先级高于网页 keydown 事件
- **根因类别**: 跨平台兼容
- **修复**: 快捷键改为 `Ctrl+Shift+P`（NotebookHome.vue:handleGlobalKeydown + 提示文案），不冲突任何主流 IME
- **教训**: 涉及 Ctrl+Shift 组合的快捷键必须避开中文 IME 已知热键（F=简繁切换, H=切换输入模式, S=全角半角等），优先用字母键 P/K/J 等无冲突键

## BUG-003: Web Share API DOCX 分享 — 未先确认浏览器能力边界就硬写 fallback

- **现象**: 分享对话框提供 DOCX 选项，系统分享报 Permission denied，三级降级逻辑复杂不可靠
- **根因**: `navigator.share({ files: [...] })` MIME 类型支持严重受限（Windows Chrome 不支持 docx），这是浏览器 API 能力边界，非代码 BUG。开发时未先查 MDN 兼容性表就实现了本不支持的功能
- **根因类别**: 跨平台兼容
- **修复**: 从分享格式中移除 DOCX，仅保留 MD 和 TXT（`text/plain` 全平台支持）
- **教训**: **实现依赖浏览器 API 的功能前，必须先查 MDN 兼容性表。遇到不支持的场景，应停止执行并告知用户，而非硬写复杂 fallback。** 这次浪费了 buildDocx + Packer.toBlob + 三级降级分享 + 两轮修复，最终回到"移除 DOCX"这一行修改

## BUG-004: 块级渲染完全缺失 — 全格式代码输出

- **现象**: 用户输入 Markdown 语法后只看到带语法高亮的源码文本，`# Heading` 显示为彩色 `# Heading` 文本而非大号标题，完全没有任何渲染预览
- **根因**: `src/utils/cm6-extensions.ts` — 只实现了 `blockDecorator`（画彩色行标记线）和 `throttledParser`（防抖重新解析），从未实现 `BlockWidget`（将 Markdown 块替换为渲染 HTML 的 CM6 Widget）。`blockParser.ts` 中所有块的 `mode` 也硬编码为 `'source'`，没有任何切换到渲染模式的路径
- **根因类别**: 渲染管线
- **修复**: 在 `NotebookHome.vue` 中新增全文档预览模式（"预览/编辑"切换按钮），使用 `@jotluck/renderer` 的 `renderMarkdown()` 渲染完整文档 HTML，通过 `v-show` 在编辑器和预览视图之间切换。预览模式下应用完整的排版样式（标题层级/代码块/表格/引用等）
- **教训**: 块级混合编辑器的 BlockWidget 是 M1 的核心交付，不能用"BlockParser + 装饰器激活"就标记完成——装饰器只是标记，渲染才是目标。如果全功能 BlockWidget 复杂度太高，至少需要全文档预览作为 fallback

## BUG-005: 页面设计原始 — 大量组件使用硬编码色值绕过主题系统

- **现象**: 页面外观与 M2 时期基本一致，主题切换后搜索面板/导出对话框/导航树等组件颜色不变。大量 `#fff`, `#eee`, `#f0f0f0`, `#999`, `#555`, `#333` 等硬编码色值
- **根因**: 多个组件（SearchPanel / ExportDialog / NavTree / NavTreeNode / SearchInput / SearchResultList / SearchResultItem / NotebookHome / AppLayout / ThemeSelector）在 scoped style 中使用裸 hex 色值或使用 `var(--clr-xxx, #硬编码)` 的 fallback 值。这些 fallback 是 bland CSS gray，完全绕过了 `construct.css` 定义的 OKLCH 色系
- **根因类别**: 状态管理（主题系统数据流断裂 — CSS 变量已定义但组件未使用）
- **修复**: 将 10+ 个组件的所有硬编码色值统一替换为 OKLCH CSS 自定义属性，fallback 值也从 `#xxx` 改为对应的 oklch() 值。涉及：SearchPanel.vue, SearchInput.vue, SearchResultList.vue, SearchResultItem.vue, ExportDialog.vue, NavTree.vue, NavTreeNode.vue, NotebookHome.vue, AppLayout.vue, ThemeSelector.vue
- **教训**: 设计系统约束（CLAUDE.md §5.3 "样式 Token 约束"）要求禁止使用 design-system.md 未定义的色值——这条规则被严重违反。CSS 变量定义后必须全局 grep 硬编码色值并替换

## BUG-006: 格式工具栏按钮溢出 — 无换行/无滚动条

- **现象**: 12 个格式按钮（含文字标签+快捷键提示）超出工具栏宽度后不可见，无法通过滚动条访问
- **根因**: `FormatToolbar.vue:67-70` — `display: flex`（默认 `nowrap`）+ `overflow-x: auto` + `scrollbar-width: none` / `::-webkit-scrollbar { display: none }`。12 个按钮需要 ~800px 宽度，中央编辑器区域约 800-1200px，在较窄窗口下溢出。滚动条被 CSS 强制隐藏，用户看不到也滚不到
- **根因类别**: 渲染管线（CSS 布局）
- **修复**: 改为 `flex-wrap: wrap` 允许按钮换行 + `max-height: 72px` + `overflow-y: auto` + `scrollbar-width: thin` + 可见滚动条
- **教训**: 隐藏滚动条的 UI 模式只有在内容确定不溢出时才安全。工具栏这种内容宽度不确定的组件，要么 wrap、要么必须保留可见的滚动指示

## 根因类别

所有 BUG 必须归类到以下根因类别之一：

| 类别           | 说明                                           |
| -------------- | ---------------------------------------------- |
| **文件IO**     | 文件读写、路径处理、编码、权限                 |
| **渲染管线**   | marked 解析、DOMPurify 清洗、highlight.js 应用 |
| **状态管理**   | Pinia store 数据流、组件间通信                 |
| **类型边界**   | TypeScript 类型不匹配、null/undefined 处理     |
| **跨平台兼容** | Web/Desktop/Mobile 行为差异                    |
| **索引/搜索**  | 索引构建、搜索查询、分词                       |
| **导出**       | docx/xlsx/PDF 生成异常                         |
| **Wiki-link**  | 链接解析、反向链接计算                         |

---

## 检查清单

编码前逐条自问，任何不满足的条目必须在计划中明确处理方案。

- [ ] 文件读写显式指定 UTF-8 编码
- [ ] 路径统一使用 `/` 分隔符
- [ ] 大文件 (>5MB) 有大小检查和降级处理
- [ ] Markdown 渲染前经过 DOMPurify 清洗
- [ ] Wiki-link 死链用不同样式标注而非崩溃
- [ ] 反向链接计算检测循环引用
- [ ] 保存前检查文件 mtime 避免并发写入冲突
- [ ] 所有异常路径有用户可见的错误提示
- [ ] CodeMirror IME composition 事件正确处理
- [ ] 语法块边界检测处理不完整/嵌套格式
- [ ] 使用浏览器 API（navigator.share / File System Access / Clipboard 等）前，先查 MDN 兼容性表和支持的 MIME/参数，遇到不支持的直接告知用户
- [ ] 大规模重构（>50 文件变更）后，检查 CLAUDE.md / spec 文档与实际代码的一致性（目录结构、Token 名称、文件路径、主题架构）
- [ ] 新增子组件 emit 时，逐层验证父组件的转发链完整性（如 LeftWing → AppShell → NotebookHome）

---

## BUG-007: 大规模 UI 重构后文档与代码严重分歧

- **现象**: 2026-06-04 `48dc95e` 提交（108 个文件，+20K/-795 行）将构成+玻璃双主题替换为纸张隐喻单主题，但 CLAUDE.md、spec/frontend/design-system.md、spec/milestones.md、spec/progress.md 均未同步更新，导致文档指向不存在的 DESIGN.md、过时的 hex Token 体系、错误的目录结构
- **根因**: 重构只改了代码，未执行 CLAUDE.md §5.8 "文档防腐败"规则。具体未更新的文档：
  - `CLAUDE.md` — §三阶段标注 Phase 0、§四引用已删除的 DESIGN.md、§六描述 hex Token、§八命令路径指向 `src/`、附录B 目录结构错误
  - `spec/frontend/design-system.md` — 仍使用 `--color-bg: #fafafa` 等 hex Token，代码已迁移至 `--paper-bg: oklch(...)` 的 OKLCH 体系
  - `spec/milestones.md` — M5 描述为"构成+玻璃"，实际交付为 Paper 主题
  - `spec/progress.md` — 当前里程碑仍标"M1"，M5 产出清单列 construct/glass.css 为核心文件
- **根因类别**: 状态管理（文档-代码一致性断裂）
- **修复**: 更新 CLAUDE.md（§三/§四/§六/§八/附录B 共 10 处编辑）+ 添加检查清单条目
- **教训**: 大规模重构的 Commit 必须包含受影响的 spec 文档更新。一个重构 = 代码 + 规格 + 元指令三者的原子提交。单独改代码不补文档，等于给未来的自己（和新 AI 会话）埋坑

## BUG-008: 左翼书签圆点无颜色且无法点击 — recentNotesList 从未初始化

- **现象**: 用户打开应用后左翼 56px 书签栏无任何圆点显示。圆点数据源为空数组。
- **根因**: `IndexService.buildFullIndex()` → `scanDirectory()` → `indexFile()` 只填充 `allDocuments` 和 `tagIndex`，从未填充 `recentNotesList`。`getRecentNotes()` 返回空数组 → `indexStore.recentNotes` 为空 → `recentNotesWithColors` 为空 → 左翼无圆点渲染。
- **根因类别**: 状态管理（数据流断裂：索引构建 → 最近笔记列表未连接）
- **修复**: 在 `buildFullIndex()` 末尾从 `allDocuments` 生成初始 `recentNotesList`（按 created 时间排序）。
- **教训**: 每个数据消费者的上游数据源必须在初始化时完整填充。索引 = {documents, tags, backlinks, recentNotes} 四个维度，缺一不可。

## BUG-009: 大纲面板点击标题无法跳转 — onNavTreeNavigate 是空函数

- **现象**: 用户在右翼大纲面板点击 H1/H2 标题，编辑器无任何反应，视口不滚动。
- **根因**: `NotebookHome.vue:329` — `onNavTreeNavigate` 函数体为 `{ /* scroll editor */ }` 空注释。CodeMirror 6 的 scrollIntoView 调用从未实现。
- **根因类别**: 渲染管线（编辑器导航功能未连接到 CM6 EditorView API）
- **修复**: 实现 `onNavTreeNavigate`：通过 `editorRef.value?.getEditorView()` 获取 CM6 view，计算目标行号对应的文档位置，调用 `view.dispatch({ selection, scrollIntoView: true })`。
- **教训**: 事件处理函数不能留空体。即使是"后续实现"的功能，也应至少打 `console.warn` 标记缺失。

## BUG-010: 设置按钮无反应 — open-settings 事件链在 AppShell 层断裂

- **现象**: 用户点击左翼底部齿轮图标 ⚙，设置对话框不弹出。
- **根因**: `LeftWing` emit `open-settings` → `AppShell` 未定义该 emit 也未在模板转发 → `NotebookHome` 未绑定。事件在 AppShell 层被丢弃。
- **根因类别**: 状态管理（组件间事件链断裂）
- **修复**: AppShell 新增 `open-settings` emit 定义 + 模板 `@open-settings="$emit('open-settings')"` + NotebookHome `@open-settings="showSettings = true"`。
- **教训**: 新增子组件 emit 时，必须逐层检查每一级父组件是否转发该事件。三层组件 (LeftWing → AppShell → NotebookHome) 中任何一层缺失都会导致功能不可用。

## BUG-011: Logo 点击无反应 — 缺少导航回首页功能

- **现象**: 用户预期点击左上角 JotLuck Logo 返回首页/清空编辑器，但无任何反应。
- **根因**: `.wing-logo` 是 `<div>` 元素，未绑定点 click 事件。
- **根因类别**: 渲染管线（交互缺失）
- **修复**: div → button + `@click="$emit('select-note', '')"` 清空当前笔记回到空状态。
- **教训**: 品牌 Logo 是用户的心智锚点，应有明确行为（首页/重置/关于三选一）。

## BUG-012: CodeMirror 默认显示行号 — 不符合纸张隐喻的极简设计

- **现象**: 编辑器左侧始终显示行号（gutter），占据视觉空间，与纸张隐喻的简洁目标冲突。
- **根因**: `cm6-extensions.ts` 使用 `lineNumbers()` 扩展。CodeMirror 默认开启行号。
- **根因类别**: 渲染管线（设计与实现不一致）
- **修复**: 移除 `lineNumbers()` 调用。行号改为仅在分栏编辑模式（左栏）显示。
- **教训**: 默认配置应与设计系统一致。"编辑时显示行号"不是默认需求。

## BUG-013: 格式工具不可见 — FormatBubble 发现性问题

- **现象**: 用户找不到格式工具（加粗/斜体等按钮），以为功能缺失。
- **根因**: Winged Editor 采用 FormatBubble（选中文字后浮现的气泡），而非固定 FormatToolbar。用户不知道需要选中文字。
- **根因类别**: 渲染管线（UX 发现性不足）
- **修复**: (1) 状态栏 "选中文字以格式化" 提示移除 `charCount > 0` 条件，添加呼吸动画（opacity pulse）; (2) 首次选中文字时显示一次性 Toast 通知 "选中文字后使用格式气泡进行加粗、斜体等操作"（localStorage 标记 `jotluck:formatBubble:hintShown`）。
- **教训**: 隐式交互（选中触发）必须有显式提示。发现性 = 主动通知 + 持久提示 + 动效。

## BUG-014: 即时格式预览 (BlockWidget) 未实现 — M1-08 遗留

- **现象**: 输入 `# Heading` 或 `**bold**` 后，编辑器仍显示带语法标记的源码文本，不自动渲染为标题/粗体。
- **根因**: M1-08 BlockWidget 从未实现，但 M7-04 Live Preview（`cm6-live-preview.ts` 1192行）已完整实现 14 种块类型的即时渲染。根因是默认视图模式为 `'split'`（分栏源码），而非 `'live'`（即时渲染）。
- **根因类别**: 渲染管线（默认配置错误）
- **修复**: (1) 默认视图模式从 `'split'` 改为 `'live'` — 用户打开笔记即看到 Markdown 即时渲染; (2) 修复 Live Preview 首次加载不渲染问题 — `constructor` 中使用 `requestAnimationFrame` 延迟 build 到下一帧。
- **教训**: 功能已实现但默认不开启 = 对用户而言功能不存在。默认值决定用户的第一印象。

## BUG-015: setTimeout 闭包中 Vue ref 动态求值导致保存写入错误文件

- **现象**: 用户快速切换笔记后，前一个笔记的编辑内容丢失，甚至写入到后一个笔记的文件中（数据损坏）
- **根因**: `NotebookHome.vue:390,225` — `setTimeout(() => debouncedSave(activePath.value, content), 600)` 中 `activePath.value` 在回调**执行时**求值而非**创建时**求值。用户切换笔记后 `activePath` 已改变，定时器触发时将旧笔记内容保存到新笔记路径
- **根因类别**: 状态管理
- **修复**: 将 `activePath.value` 快照到局部常量 `savingPath`，确保闭包捕获创建时的值。同时 `onSelectNote` 切换前 flush pending save + cancel timer
- **教训**: **Vue ref 的 `.value` 在异步回调中永远是"当前值"而非"快照值"。** setTimeout/Promise.then 中使用 ref 值必须先快照到局部常量

## BUG-016: CM6 仅监听 keyup 导致中文 IME 输入不触发内容同步

- **现象**: 用户输入中文后立即切换笔记，中文内容丢失。ASCII 输入正常。
- **根因**: `MarkdownEditor.vue:86` — 仅 `view.dom.addEventListener('keyup', syncContent)` 监听内容变更。中文 IME 通过 `insertText` (CDP Input.insertText) 输入，不触发 `keyup`，只触发 `beforeinput` → CM6 inputHandler
- **根因类别**: 渲染管线
- **修复**: 添加 `EditorView.updateListener.of((update) => { if (update.docChanged && !suppressSync) emit(...) })` — CM6 原生 update 事件覆盖所有输入方式。添加 `suppressSync` 标志切断 watch → dispatch → updateListener → emit 反馈环
- **教训**: **CM6 内容同步必须用 `updateListener`，不能用 DOM 事件。** DOM keyup/mouseup/paste 是异步、不可靠的，且遗漏 IME/合成输入/撤销重做

## BUG-017: 导出管线全为 stub — docx/xlsx/pdf/html 均未实现真实导出

- **现象**: DOCX 导出为 HTML 包装的 `.doc` 文件（非真实 docx）；XLSX 导出为 CSV；PDF/HTML 导出原始 markdown 文本而非渲染 HTML。`docx` 和 `xlsx` npm 包已安装但零引用
- **根因**: UI 全量重构时仅重做了 ExportDialog 前端对话框，`Exporter.ts` 中的 6 个导出函数从未适配真实管线
- **根因类别**: 导出
- **修复**: 600 行 Exporter.ts 重写。DOCX: `marked.lexer()` → `Document/Packer/Paragraph/TextRun`；XLSX: `extractTables()` → `write-excel-file` → `toBlob()`；PDF: `renderMarkdown()` → iframe → `window.print()`；HTML: 自包含 CSS + `renderMarkdown()`。接入 includeFrontmatter/includeWikiLinks/codeLineNumbers 选项
- **教训**: **UI 重构后必须审计所有后端服务适配状态。** 对话框只是冰山一角，真正的工作在服务层

## BUG-018: `onSplitContentUpdate` 缺少 `updateHeadings` 调用

- **现象**: 分栏模式（默认）下编辑内容后右侧大纲面板不更新，始终显示初始标题
- **根因**: `NotebookHome.vue:218` — `onSplitContentUpdate` (split 模式) 调用 `updateEditorStats` 但不调用 `updateHeadings`。而 `onContentUpdate` (live 模式) 正确调用了两者
- **根因类别**: 状态管理
- **修复**: 在 `onSplitContentUpdate` 中添加 `updateHeadings(content)`（1 行）
- **教训**: **两个 handler 之间存在功能不对称是常见陷阱。** 抽象公共 update 逻辑可防止此类遗漏

## BUG-019: `waitForSaved` 选择器错误 — 等待"未保存"而非"已保存"

- **现象**: E2E 测试在保存完成前就继续执行，导致后续断言读取旧内容
- **根因**: `test-utils.ts:115` — `.status-saved, .status-dirty` 匹配任一状态。`isDirty=true` 时 `.status-dirty` 立即可见，函数立即返回（保存尚未完成）
- **根因类别**: 类型边界（测试工具错误）
- **修复**: 改为仅 `.status-saved`，语义匹配函数名"等待已保存"
- **教训**: **CSS 选择器中的逗号是 OR 逻辑。** 函数命名必须与其实现精确一致

## BUG-020: `openNote` 用 `hasText` 匹配无文本的 bookmark dots

- **现象**: E2E 测试中 `openNote(page, '快速入门')` 无法正确打开指定笔记
- **根因**: `test-utils.ts:84` — `page.locator('.wing-bookmark-dot', { hasText: noteName })`。Bookmark dots 内部仅有 `<span class="dot-core"/>` 无文本内容（仅 aria-label），`hasText` 永远不匹配。fallback 到 `dot.first()` 始终点击第一个 dot
- **根因类别**: 类型边界
- **修复**: 改为 `page.locator(\`.wing-bookmark-dot[aria-label="${noteName}"]\`)` — 精确 aria-label 匹配
- **教训**: **`hasText` 匹配的是 rendered text content，不含 aria-label/title/data-attribute。** 纯视觉元素用属性选择器

---

## BUG-022: FileDrawer 选择文件后不自动关闭 — overlay 永久遮挡编辑器

- **现象**: 通过文件抽屉打开笔记后，`drawer-overlay`（`z-index: 500, position: fixed, inset: 0`）永久覆盖全视口。后续所有编辑器/书签点击被 overlay 拦截。E2E 测试中 format-toolbar (6个)、editor、persona 等测试因 `pointer events intercepted by drawer-overlay` 失败
- **根因**: `FileDrawer.vue:761-770` — `handleItemClick` 选择文件后仅 `emit('select-file')`，未调用 `close()`。`NotebookHome.vue:374` — `onSelectNote` 也未设置 `showLeftDrawer = false`。抽屉打开后除非用户手动点击 overlay 关闭，否则永不消失
- **根因类别**: 状态管理（组件间状态同步断裂 — drawer visible 状态在选择文件后未重置）
- **修复**: (1) `FileDrawer.vue:769` — 文件选择后添加 `close()` (2) `NotebookHome.vue:374` — `onSelectNote` 完成后设 `showLeftDrawer.value = false`
- **教训**: **任何会改变全局 UI 状态的交互（打开侧栏/抽屉/模态框），在完成其使命后必须关闭。** `v-if` 的 overlay 在 `z-index` 上高于主内容区，只要 visible 为 true 就会阻挡所有下层交互

## BUG-023: Wiki-link 分屏预览无样式 — marked-extensions 与 main.css CSS 类名不匹配

- **现象**: 分屏预览模式下 `[[...]]` 显示为普通文本（无下划虚线/颜色），实时预览和导出模式正常
- **根因**: `marked-extensions.ts:73` 输出 `class="wikilink"`（无连字符），但 `main.css:209,215` 定义 `.wiki-link`（带连字符）。`editor.css` 和 `Exporter.ts` 均使用 `wikilink` 无连字符，仅 `main.css` 使用带连字符的变体。类名不匹配 → 样式永不应用
- **根因类别**: 渲染管线（CSS-渲染器命名不一致）
- **修复**: `main.css:209,215` — `.wiki-link` → `.wikilink`，`.wiki-link--broken` → `.wikilink--dead`。对齐多数派命名（3/4 处使用 `wikilink` 无连字符）
- **教训**: **跨层命名约定必须在两个方向验证。** CSS 类名是 rendered HTML 和 stylesheet 之间的隐式契约。重构时改了一端必须改另一端，且需全局 grep 确认所有引用点的命名一致性

## BUG-024: 大规模 UI 重构后 E2E 测试选择器批量失效

- **现象**: 本轮 E2E 全量回归 28 个失败，其中 12 个因 CSS 选择器找不到目标元素。涉及 `.btn--use`, `.btn--small`, `.btn--primary`, `.node-name`, `.node-item--file`, `.wiki-link`, `.wiki-link--broken` 等 7 类过时选择器
- **根因**: 上一轮 UI 重构（Button 组件统一化 + Paper 主题迁移）将按钮类名从 BEM `btn--variant` 改为 `mk-btn--variant`，文件树类名从 `node-name/node-item--file` 改为 `tree-name/tree-item`，wiki-link 类名从 `wiki-link` 改为 `wikilink`。但 E2E 测试中的选择器**全部未同步更新**
- **根因类别**: 状态管理（代码-测试一致性断裂）
- **修复**: 批量迁移 7 类选择器，涉及 `templates.spec.ts` (`.node-name`→`.tree-name`, `.btn--use`→`.mk-btn--default`, `.btn--small`→`.save-toggle`, `.save-form-actions .btn--primary`→`.save-form-actions .mk-btn--default`), `wiki-link.spec.ts` (`.wiki-link`→`.wikilink`), `panels.spec.ts` (`.tree-name` 点击→`openNote()`), `format-toolbar.spec.ts` (`bubble-in`→`startsWith('bubble-in')`)
- **教训**: **UI 重构 = 代码 + 样式 + 测试三者的原子提交。** CLAUDE.md §5.8 文档防腐败规则应扩展：重构后必须 `grep` 所有 `.spec.ts` 文件中的 CSS 选择器，验证每个选择器在源码中仍然存在

## BUG-025: Vue scoped `<style>` 中 @keyframes 名称被 hash 后缀修改

- **现象**: `format-toolbar.spec.ts` 中 `rule.name === 'bubble-in'` 精确匹配失败，`hasBubbleInKeyframes` 返回 false
- **根因**: Vue 3 `<style scoped>` 对 `@keyframes bubble-in` 生成的 CSS 规则名称为 `bubble-in-data-v-xxxxx`（追加组件 hash 后缀）。测试用 `===` 精确匹配原始名称，永远不成立
- **根因类别**: 类型边界（CSS 框架行为假设错误）
- **修复**: `=== 'bubble-in'` → `.startsWith('bubble-in')`（1 行）
- **教训**: **Vue scoped styles 中的 @keyframes 名称会被 hash 污染。** JS 侧检查 CSS 规则时必须用 `startsWith` 而非 `===`。此行为在 Vue 3 文档中未明确说明，属于隐性知识

## BUG-026: `setSearchHistory` 测试辅助函数 newest-first 语义错误

- **现象**: `search.spec.ts:625` — `expect(history[0]).toBe('查询15')` 失败，实际 `history[0]` 为 `'查询1'`
- **根因**: `setSearchHistory` 用 `value.slice(0, max)`（取前 10 条，保留原始顺序），但 `addToHistory` 的真实行为是每次 `unshift` + `slice(0,10)`（最新插入排在第一位）。15 次连续 addToHistory 后结果应为 `value.slice(-10).reverse()`（最后 10 条，newest first），而非 `value.slice(0, 10)`（前 10 条，oldest first）
- **根因类别**: 类型边界（测试 Mock 行为与真实行为不一致）
- **修复**: 测试中预计算正确状态：`const simulated = allQueries.slice(-10).reverse()` 传给 `setSearchHistory`
- **教训**: **测试 Mock 函数必须精确模拟真实函数的行为语义。** `setSearchHistory` 的"直接写 localStorage"和 `addToHistory` 的"unshift + cap"是两种不同的数据写入路径，需在测试数据中体现差异

## BUG-027: 笔记切换后内容丢失 — 三个独立缺陷叠加

- **现象**: 编辑笔记 → 保存完成 → 切换到另一篇笔记 → 切回原笔记，编辑器显示原始内容（编辑的标记丢失）。手动验证 T2 测试稳定复现，editor.spec.ts V2 和 persona.spec.ts E1 均失败
- **根因**: 经 4 轮诊断（D1→D2→D3→D4），定位到三个独立缺陷叠加：

  **缺陷 1 — `waitForSaved` 选择器错误（BUG-019 修正从未落地）**
  - `test-utils.ts:138` — `.status-saved, .status-dirty` 为 OR 逻辑，`isDirty=true` 时 `.status-dirty` 立即可见 → 函数立即返回 → 保存未完成测试就切笔记
  - **修复**: 改为仅 `.status-saved`（2026-06-10, T2 step3 验证通过 ✅）

  **缺陷 2 — `onUnmounted` 残留函数污染新组件**
  - `MarkdownEditor.vue:148-149` — 旧组件 `onUnmounted` 删除了 `__jotluck_getEditorContent` 但**未删除** `__jotluck_getEditorView` 和 `__jotluck_editorInitValue`
  - 新组件 setup 重新注册后，时序窗口期内旧函数残留引用了已销毁的 view 对象 → `getEditorView()` 返回非null但 `getEditorContent()` 返回空串
  - **修复**: `onUnmounted` 增加删除 `__jotluck_getEditorView`, `__jotluck_editorInitValue`, `__jotluck_modelOverwrites`（2026-06-10 ✅）

  **缺陷 3 — Vue 3 `:key` patch 顺序错误：新 setup 先于旧 unmount**
  - `MarkdownEditor.vue:61,149` — `__jotluck_getEditorContent/View` 在 `<script setup>` 顶层注册，旧组件的 `onUnmounted` 删除它们。Vue 3 `:key` 补丁顺序为 **新 setup → 旧 unmount → 新 mount**。旧 unmount 删除了新 setup 刚注册的函数，导致 `view=null` 对外不可见
  - **证据**: 生命周期日志显示 `setup(id=j70ame, t=3875) → unmounting(id=i0qy85, t=3876) → mounted(id=j70ame, t=3882)` — setup 在 unmount 之前
  - **修复**: 将 `__jotluck_getEditorContent/View` 注册从 `setup` 移到 `onMounted`（在旧 unmount 之后执行）（2026-06-10 ✅ T2 PASS）

- **根因类别**: 状态管理（异步挂载竞态 × `:key` 重建 × updateListener 反馈环 × DOM 清理不完整 — 四重窗口期叠加）
- **诊断数据**（D2 2026-06-10）:
  ```
  MockFS:        hasMarker=true, len=437   ✅ localStorage 持久化正常
  EditorView:    initValueLen=437          ✅ props.modelValue 传入正确
  CM6 DOM:       cmContentLen=377          ❌ 60字符丢失（= 标记长度）
  modelOverwrite: count=0                  ❌ watch未触发，排除Vue路径
  editorViewExists: false                  ❌ 函数被清理，旧view废弃
  ```
- **教训**:
  1. **`waitForSaved` 的修复必须实际写代码，不能只记录错题本就标记完成。** BUG-019 记录了修复但 `test-utils.ts` 从未改过
  2. **`onUnmounted` 中注册的全局对象（window hooks）必须逐条删除，不能漏项。** 遗漏的 `__jotluck_getEditorView` 返回已销毁的 view 导致调用方拿到垃圾数据
  3. **Vue 3 `:key` 补丁的生命周期顺序是 新 setup → 旧 unmount → 新 mount。** setup 注册的全局函数会被旧 unmount 删除。全局注册应放在 onMounted 中
  4. **`async onMounted` 把 EditorView 创建延迟到 HTTP fetch 之后，造成 view=null 窗口期。** 异步初始化应改为 fire-and-forget，不阻塞 view 创建

## BUG-021: 搜索全文匹配失效 — `preloadContent()` 已实现但从未调用

- **现象**: Ctrl+K 搜索仅有标题匹配结果，内容匹配（如搜索正文中的 "纸张隐喻"）永远返回空。E2E Journey 3 搜索 "快速入门" 返回 0 条结果
- **根因**: `SearchEngine.ts:22` — `buildIndex()` 用 `content: ''` 硬编码空内容存储所有文档。`preloadContent()` (line 26) 正确实现内容加载，但**整个代码库中无任何调用**。连锁影响：
  - `SearchEngine.ts:122-129` — `toResult()` 返回 `{ snippet, matchType, relevanceScore, positions }` 而非类型声明的 `{ matches: SearchMatch[], score }`。一旦结果出现，`CommandPalette.vue:67` 访问 `result.matches.length` 将崩溃
  - `IndexService.ts:137-139` — `updateDocument()` 仅刷新元数据，不调用 `engine.updateDocument()`
  - `SearchEngine.ts:58` — `updated ?? created` 引用了不存在的 `updated` 字段
- **根因类别**: 渲染管线（搜索子系统）
- **修复**: (1) `IndexService.buildFullIndex()` 添加 `await engine.preloadContent(documents, readFile)`；(2) `toResult()` 重写为符合 `SearchResult` 类型；(3) `updateDocument()` 添加 `engine.updateDocument()` 调用；(4) `DocumentEntry`/`DateRange` 类型对齐运行时形状；(5) wiki-link 提取从 `buildFullIndex` 的二次文件读取移入 `indexFile()`（消除重复 I/O）
- **教训**: **实现了但未调用的代码等于不存在。** 两阶段初始化（buildIndex + preloadContent）是合理的分离模式，但任一阶段遗漏都导致系统静默降级（内容匹配失效但标题匹配仍工作，掩盖了缺陷）

---

## BUG-028: MarkdownPredictor 服务层单元测试完全缺失

- **现象**: `packages/app/src/services/MarkdownPredictor.ts`（493 行核心预测服务）没有任何单元测试覆盖。`vitest run` 无 MarkdownPredictor 相关测试运行。NGramEngine 有 24 个单元测试，但上层融合服务为零。
- **根因**: M7-13/14/15 于 2026-06-09 完成时仅覆盖了纯算法层（NGramEngine），服务层被遗漏。深层根因链：
  1. `autocomplete-spec.md` Phase 1 验收标准仅要求 "NGramEngine 单元测试全部通过"，未覆盖 MarkdownPredictor
  2. `progress.md` M7-15 任务描述为 "NGramEngine 单元测试 24/24"，交付范围定义不全面
  3. 服务层测试需要 mock localStorage/fetch/IndexStore，心理上比纯函数测试更难写
- **根因类别**: 状态管理（规格-交付一致性断裂）
- **修复**: 创建 `packages/app/src/services/__tests__/MarkdownPredictor.test.ts`，72 个测试用例覆盖 6 大类：
  1. 纯函数工具（extractContext / getLineAt / isInFencedCode / isInFrontmatter / detectOpenFormat）— 20 个
  2. 语法上下文检测（detectSyntaxContext / isDisabledContext）— 14 个
  3. 结构化预测（Wiki-link / Tag / File Path / Format Closure）— 10 个
  4. getGhostText() 融合决策 — 5 个
  5. 持久化与淘汰（save/load/closeDocument/forceEliminate）— 8 个
  6. 边界与错误处理（constructor/loadBaseline/initialize/ingestExcerpts）— 9 个
  7. acceptCompletion/rejectCompletion/scanOpenedDocument — 4 个
  8. L1/L2 优先级 — 2 个
- **教训**:
  1. **规格验收标准必须覆盖完整的模块清单。** autocomplete-spec.md 列出了 7 个文件，验收标准只覆盖了 NGramEngine。每个核心模块都应出现在验收清单中
  2. **里程碑任务描述应明确区分工具层和服务层。** "NGramEngine 单元测试"不应等同于"文字补全功能已测试"
  3. **服务层 mock 模式（vi.stubGlobal for localStorage/fetch, 接口注入 for IndexData）是通用测试基础设施，应提取为测试工具函数**

---

## BUG-029: Ghost Text 不渲染 — 四层独立缺陷叠加

- **现象**: 用户输入 `**` 等未闭合格式标记后，编辑器无 ghost text 显示。E2E 测试中 `.cm-ghost-text` 元素数为 0，`getGhostText()` 虽然返回正确结果但 widget 不渲染到 DOM。
- **根因**: 四层独立缺陷叠加，每条单独即可阻断 ghost text：

  **Defect 1 — `injectStructuredKnowledge` 门禁过宽** (`MarkdownPredictor.ts:230`)
  - `predictFormatClosure` 是完全不需要 `indexData` 的纯函数，但被外层 `if (!this.indexData) return null` 拦截
  - 修复: `markdown-format` 类型移到 `indexData` 检查之前

  **Defect 2 — `setIndexData` 仅调用一次** (`NotebookHome.vue:557-579`)
  - 只在 `onMounted` 中调用，编辑器因 `:key` 重建后新 predictor 的 `indexData=null`
  - 修复: 提取 `connectPredictor()` 函数 + `watch(activePath)` 在每次编辑器重建后重连

  **Defect 3 — L2 从不持久化** (`MarkdownEditor.vue:163-179`)
  - `onUnmounted` 缺少 `predictor.closeDocument()`，L1 不合并到 L2，L2 不写 localStorage
  - 修复: 在 `view.destroy()` 后添加 `predictor.closeDocument()`

  **Defect 4 — CM6 decorations 不触发重绘** (`cm6-ghost-text.ts:114-122`)
  - `doPredict` 在 `setTimeout` 回调中修改 `this.decorations`，但 CodeMirror 6 只在 view update 周期中读取 decorations 字段。`setTimeout` 中修改不会自动触发重绘
  - 修复: 设置 decorations 后添加 `view.dispatch({})` 触发空事务
  - 对比: `clearGhost()` 中有 `view.requestMeasure()`，`doPredict` 缺少等价调用

  **Defect 5 — 全局 plugin 引用竞态** (`cm6-ghost-text.ts:220-224`)
  - `ghostTextPluginInstance` 是模块级变量，每次编辑器重建时被覆盖。Tab/Escape keymap 通过该变量查找 plugin，可能匹配到错误的实例
  - 修复: plugin spec 直接作为参数传给 `ghostTextKeymap()`，不再使用全局变量

  **Defect 6 — keymap 优先级倒置** (`MarkdownEditor.vue:73-77`)
  - ghost text Tab handler 注册在 `defaultKeymap`（含 `indentWithTab`）之后，Tab 先被缩进逻辑消费
  - 修复: `autocompleteCompartment` 移到 `...jotluckExtensions()` 之前

- **根因类别**: 渲染管线（CM6 ViewPlugin 生命周期）+ 状态管理（predictor/indexData 生命周期）
- **诊断过程**:
  1. 单元测试确认 `getGhostText()` 返回正确 (confidence 0.85)
  2. E2E debug 日志确认 `doPredict` 被调用
  3. 发现 `.cm-ghost-text` 元素数为 0 但 `lastPrediction` 有值 → 定位到 Defect 4
  4. 修复 Defect 4 后 ghost text 出现但 Tab 不工作 → 定位到 Defect 5 + 6
  5. 修复 Defect 2+3 后 L2 持久化确认 (`l2Size: 14319`)
- **教训**:
  1. **CM6 ViewPlugin 在 `setTimeout` 中修改 decorations 必须调用 `view.dispatch({})` 触发重绘。** `requestMeasure()` 不足够（`clearGhost` 用它因为它只清除，`doPredict` 设置新 decorations 需要完整 update cycle）
  2. **CM6 keymap 优先级由注册顺序决定，先注册先执行。** 自定义 Tab handler 必须在 `defaultKeymap` 之前
  3. **模块级单例变量在组件重建场景下不可靠。** 编辑器 `:key` 重建时，全局引用会被覆盖。应使用闭包参数传递
  4. **格式闭合预测不应依赖 indexData。** `predictFormatClosure` 是纯格式匹配，应该独立于结构化数据源
  5. **Vue 组件销毁时应持久化状态。** `onUnmounted` 中漏掉 `closeDocument()` 导致跨会话学习数据全部丢失

## 统计

| 指标             | 数值 |
| ---------------- | :--: |
| 总 BUG 数        |  41  |
| 真 BUG（已修复） | 46\* |
| 真 BUG（待修复） |  0   |

> \*BUG-029 含 6 个独立缺陷，BUG-027 含 3 个独立缺陷，BUG-032 含 3 个独立缺陷。全部 BUG 已修复完毕。
> | DEFERRED（假 BUG） | 0 |

| 指标 | 数值 |
| ---- | :--: |

## 检查清单

- [ ] 两阶段初始化（build + load）：第二阶段的 `preloadContent` 是否已调用？
- [ ] `toResult` 返回形状是否与 `SearchResult` 类型完全匹配（`matches` + `score`）？
- [ ] 增量更新路径（`updateDocument`）是否同步更新了搜索引擎？
- [ ] 类型定义的字段是否与运行时代码实际写入的字段一致？
- [ ] 是否有"实现了但从未调用"的方法？grep 检查 `.ts` 文件
- [ ] Ghost text 可见时 Tab 接受补全，不可见时回退缩进，不能误触发
- [ ] 中文 IME composition 期间不触发 ghost text 预测
- [ ] 代码块/frontmatter 区域内不显示 ghost text
- [ ] localStorage L2 缓存超限时末位淘汰正确触发
- [ ] 冷启动时基准 L2 加载失败（fetch 404）有降级处理，不阻断编辑器启动
- [ ] **UI 重构后必须 grep 所有 `.spec.ts` 中的 CSS 选择器，验证每个选择器在源码中仍然存在**（教训：BUG-024）
- [ ] **改变全局 UI 状态的交互（侧栏/抽屉/模态框）在完成使命后必须关闭，避免 `v-if` overlay 永久遮挡**（教训：BUG-022）
- [ ] **跨层 CSS 类名约定必须双向验证 — 渲染器输出 ↔ Stylesheet 定义 ↔ E2E 选择器三方一致**（教训：BUG-023）
- [ ] **Vue scoped styles 中 `@keyframes` 名称会被 hash 污染，JS 侧检查时用 `startsWith` 而非 `===`**（教训：BUG-025）
- [ ] **E2E 测试 Mock 辅助函数必须精确模拟真实函数的行为语义（如 newest-first 排序）**（教训：BUG-026）
- [ ] **`async onMounted` + `:key` 重建组合会导致 `view = null` 窗口期。且 Vue 3 `:key` 补丁顺序为新 setup → 旧 unmount → 新 mount，setup 注册的全局函数会被旧 unmount 删除。全局注册应放在 onMounted 中**（教训：BUG-027）
- [ ] **规格验收标准必须覆盖完整的模块清单，不能只测试最底层工具而遗漏上层服务。里程碑任务描述应明确区分工具层和服务层测试**（教训：BUG-028）
- [ ] **CM6 ViewPlugin 的 `setTimeout` 回调中修改 decorations 后必须调用 `view.dispatch({})`（非 `requestMeasure()`）。** 框架只在 update cycle 中读取 decorations 字段，异步回调中修改不会自动触发 DOM 更新（教训：BUG-029 Defect 4）
- [ ] **CM6 keymap 的 Tab handler 优先级必须高于 `defaultKeymap`（`indentWithTab`）。** 将自定义 keymap extension 放在 `jotluckExtensions()` 之前注册（教训：BUG-029 Defect 6）
- [ ] **模块级单例变量在组件 `:key` 重建场景下不可靠。** 全局引用会被新实例覆盖。应使用闭包参数传递 plugin spec（教训：BUG-029 Defect 5）
- [ ] **Vue 组件 `onUnmounted` 中必须持久化有状态服务。** 漏掉 `closeDocument()` 导致跨会话数据全部丢失（教训：BUG-029 Defect 3）
- [ ] **降低 N-gram 阈值会引入噪声预测。短上下文仅适用于结构化预测（`[[`/`#`/路径/格式闭合），N-gram 必须保持足够上下文长度**（教训：BUG-030）
- [ ] **CM6 `view.contentDOM` 才是 IME 事件源头。监听 `compositionstart/end` 必须挂载在 `contentDOM` 上，而非外层 `view.dom`**（教训：BUG-032）
- [ ] **Playwright 无法模拟 IME composition 事件链。IME 相关 BUG 必须纳入 L4 人工验证清单**（教训：BUG-032）
- [ ] **Vue 3 `provide/inject` 方向为父→子。子组件 provide 的值父组件无法 inject。跨层级通信应使用模块级单例状态**（教训：BUG-033）
- [ ] **`toBeGreaterThanOrEqual(0)` 是 E2E 最常见假通过模式——`.count()` 永不为负数。禁止在 E2E 中使用此模式**（教训：BUG-034）
- [ ] **CM6 ViewPlugin 的 `destroy()` 必须清理所有在构造函数中注册的外部资源（DOM 监听器/rAF/定时器）。框架不自动管理这些**（教训：BUG-035）
- [ ] **localStorage 序列化格式变更必须保持向后兼容。读取逻辑应同时处理新旧两种格式，写入统一为新格式**（教训：BUG-040）
- [ ] **模态对话框的关闭方式必须包含三要素：遮罩点击 + Escape 键 + 关闭按钮。新增对话框时对照 ShareDialog 模板检查**（教训：BUG-041）
- [ ] **IndexService 的增量操作方法（updateDocument/removeDocument）必须同时更新 allDocuments、tagIndex、wikiOutgoing/Incoming、recentNotesList、SearchEngine 五个维度**（教训：BUG-042）
- [ ] **UI 文本变更后必须 grep 所有 E2E 测试中的 `hasText` / `filter({ hasText })` 选择器**（教训：BUG-044）
- [ ] **始终渲染（无 v-if）但动态内容的 fixed 容器需要 min-height/min-width 保证布局稳定性，否则 Playwright toBeVisible() 会因 0x0 尺寸失败**（教训：BUG-045）

---

## BUG-040: 主题 localStorage 存储格式与测试期望不一致

- **现象**: 旧外观持久化测试断言本地存储格式错误，字符串与对象格式不一致。
- **根因**: 历史外观状态序列化格式与测试预期不一致。
- **根因类别**: 状态管理
- **修复**: (1) `apply()` 改为直接存储 `colorScheme.value` 字符串；(2) `init()` 兼容读取两种格式（先直接字符串检查，再 JSON 解析兜底）
- **教训**: localStorage 序列化格式是隐式 API 契约。测试断言的格式即是消费者期望的格式。存储格式变更时必须同步更新读取逻辑并保持向后兼容

## BUG-041: Escape 键不关闭设置/导出对话框

- **现象**: E2E 测试按 Escape 键后 SettingsDialog 和 ExportDialog 不关闭，`modal-overlay` 仍可见
- **根因**: `SettingsDialog.vue:3` 和 `ExportDialog.vue:3` 仅在 `div.modal-overlay` 上绑定了 `@click.self`，缺少 `@keydown.escape` 处理器
- **根因类别**: 渲染管线（交互缺失）
- **修复**: 两文件各加 `@keydown.escape="close"` / `@keydown.escape="cancel"`（参考已正确实现的 ShareDialog 和 TemplateDialog）
- **教训**: 模态对话框的关闭方式矩阵为 {遮罩点击, Escape 键, 关闭按钮} 三项，缺一不可。新增对话框时必须对照已有模板检查

## BUG-042: 创建笔记后书签圆点不更新

- **现象**: 新建/模板创建笔记后，左翼 56px 书签栏不出现新笔记的彩色圆点，E2E 切换笔记测试超时
- **根因**: `IndexService.updateDocument()` 和 `removeDocument()` 更新了 allDocuments/tagIndex/搜索引擎，但从未更新 `this.recentNotesList`。`getRecentNotes()` 始终返回 `buildFullIndex()` 时的静态快照
- **根因类别**: 状态管理（数据流断裂：索引增量更新 → 最近笔记列表未连接）。与 BUG-008 同模式
- **修复**: `updateDocument()` 中 filter+unshift+sort+slice 将新路径插入头部（限 20 条）；`removeDocument()` 中 filter 移除对应条目
- **教训**: 索引的五个维度（documents, tags, backlinks, recentNotes, searchEngine）在增量操作中必须全部更新。`buildFullIndex` 是初始化的全面填充，增量方法也必须全面——任何维度的遗漏都会导致视图层拿到过期数据

## BUG-043: 书签 active 状态缺失

- **现象**: 点击新创建的笔记后，对应书签圆点无 active ring 动画，`.wing-bookmark-dot.active` 选择器匹配不到
- **根因**: BUG-042 的直接后果 — 没有圆点 DOM 元素可应用 active 类。activePath 传递链（NotebookHome→AppShell→LeftWing）和比较逻辑本身正确
- **根因类别**: 状态管理（级联失效）
- **修复**: BUG-042 修复后自动解决（无独立代码变更）
- **教训**: 当 BUG 表现为"某个 UI 状态不更新"时，应先检查数据源是否可到达，再检查 CSS 类名/条件。数据源缺失的 BUG（如 BUG-042）会产生连锁表象 BUG

## BUG-044: 右侧面板区段折叠 E2E 测试选择器文本不匹配

- **现象**: 测试 14/18/19/20 点击区段标题后断言 `section-body` 可见性失败，超时
- **根因**: E2E 测试使用 `hasText: '目录'` 和 `hasText: '反向链接'`，模板实际文本为 `大纲` 和 `反链`。`filter({ hasText })` 子串匹配无法匹配这些语义相近但文字不同的词
- **根因类别**: 类型边界（测试-UI 文本不一致）
- **修复**: 4 处选择器文本对齐模板：`'目录'`→`'大纲'`，`'反向链接'`→`'反链'`
- **教训**: UI 文本是测试的隐式 API。组件重构时改变 section label 文本后，必须 grep 所有 E2E 测试中的对应 `hasText` 选择器

## BUG-045: Toast 空容器 Playwright toBeVisible() 失败

- **现象**: E2E 测试 13-toast 用例 01/03 断言 `.toast-container` 和 `.toast-stack` 可见失败，返回 hidden
- **根因**: `.toast-container` 和 `.toast-stack` 无 toast 时始终渲染（无 v-if），但 CSS 无 min-height/min-width，导致 0×0 尺寸。Playwright `toBeVisible()` 要求元素有非零边界框
- **根因类别**: 渲染管线（CSS 布局边界情况）
- **修复**: `.toast-container` 添加 `min-height: 1px; min-width: 1px`
- **教训**: 始终渲染（无 v-if）但动态内容的 `position: fixed` 容器需要 min-height/min-width 保证布局稳定性。空容器对 Playwright 是不可见的

## BUG-030: Ghost Text 格式闭合预测级联

- **现象**: 输入 `**` 后出现 `***************`（15+ 个星号）的幽灵文本级联
- **根因**: `effectiveN = min(ctx.length, n)` 降级暴露 2-gram 噪声 + `view.dispatch({})` → `schedulePredict` 反馈环
- **根因类别**: 渲染管线
- **修复**: ① N-gram 恢复 `n≥4` 阈值，② `if (result.text === this.currentGhostText) return;` 防重复

## BUG-031: 中文输入无法触发幽灵文本

- **现象**: 输入 6-7 个中文字符后零触发
- **根因**: 基线 L2 中文 N-gram 覆盖率不足。字符级 4-gram 匹配不到基线条目
- **根因类别**: 索引/搜索
- **修复**: 同 BUG-030。中文预测目前依赖 L1 积累，长期需中文语料重训基线

## BUG-032: IME 组合输入三连 BUG（双行/光标跳/吞字）

- **现象**: ① 中文输入法临时字符双行渲染，② 确认后光标跳到行首，③ 标点需多次输入
- **根因**: `livePreview.update()` 在 `compositionupdate` 期间执行 `Decoration.replace`
- **根因类别**: 渲染管线
- **修复**: IME 期间 `isComposing` flag + `tr.isUserEvent('input.type.compose')` 二层防护 + `contentDOM` 监听

## BUG-033: Toast 通知永远不显示

- **现象**: `toast.show()` 零效果、零错误
- **根因**: `provide/inject` 方向错误——`ToastContainer`（子）provide，`NotebookHome`（父）inject
- **根因类别**: 状态管理
- **修复**: 改为模块级 `ref<ToastItem[]>([])` 单例状态

## BUG-034: E2E 测试假通过

- **现象**: E2E 全绿但用户手动测试发现大量 BUG
- **根因**: 弱断言（`>=0`/`||`）/ 选择器错误（`.cm-rendered-block` 等不存在）/ Playwright IME 盲区
- **根因类别**: 类型边界
- **修复**: 8 文件弱断言清零 + 80+ 选择器交叉验证 + 中文 IME 专项测试 + `storageState`

## BUG-035: cm6-live-preview 事件监听器泄漏

- **现象**: 编辑器重建后旧监听器未释放
- **根因**: `change`/`click` 匿名监听器 + rAF 在 `destroy()` 中未清理
- **根因类别**: 渲染管线
- **修复**: 实例字段存储引用 + `destroy()` 完整清理 + `destroyed` flag

## BUG-036: IME 组合输入期间 ghost/live preview 仍会抢占 DOM 更新

- **现象**: 中文输入时概率性串行、焦点漂移；输入法候选占位文字出现在下方其他行；提交后文字丢失、焦点偏移到文档底部；ghost text 出现在下方行。
- **根因**: `cm6-live-preview.ts` 虽有 composition guard，但初始化 rAF 和 `compositionend` 后立即重建 `Decoration.replace`，可能早于 IME 最后一笔 `input.type.compose` 提交；`cm6-ghost-text.ts` 只依赖本地 `isComposing`，未检查 `EditorView.composing` / `compositionStarted` / compose transaction，且 `compositionend` 后立即预测，widget `side: 0` 容易贴到候选文本前后边界。
- **根因类别**: 渲染管线
- **修复**: IME 开始时清空 live preview 替换装饰与 ghost text；IME 活跃时同时检查 DOM flag、`EditorView.composing`、`compositionStarted`、`input.type.compose`；composition 结束后延迟 80ms 再重建装饰/预测；ghost widget 改为 `side: 1`；销毁时清理 composition 延迟计时器。
- **教训**: IME 防护不能只拦 `update()` 主路径，所有异步入口（rAF、setTimeout、compositionend 回调）都必须复用同一 IME active 判定，并且 ViewPlugin `update()` 内不得嵌套 `dispatch()`。

## BUG-037: Markdown 格式补全把 `**` 训练成星号级联并困住行尾焦点

- **现象**: 输入 `**` 设置粗体或其他格式时，自动补全把多行当成同一格式判断范围；手动换行后难以脱离该范围；方向键焦点被困在特定多行文本里；接受 `**` 补全后出现 `**************` 等星号级联建议。
- **根因**: `predictFormatClosure()` 对 `**` 返回 `粗体**` 这类占位文本，并且 `acceptGhost()` 将结构化格式闭合补全当作普通 N-gram 样本写入 L1/L2；`acceptGhost()` 中 `view.dispatch()` 会同步触发 `update()`，若不先快照预测来源，结构化来源会被清空后误学习；`cm6-live-preview.ts` 用 `cursor < block.to` 判断当前行焦点，光标在行尾时当前行被替换成 widget，导致换行/方向键边界体验异常。
- **根因类别**: 渲染管线
- **修复**: 格式闭合只补闭合标记本身（`**文字` → `**`），裸 `**` 不显示占位建议；`PredictionResult` 增加 `source/syntaxType`，结构化补全不回写 N-gram；`acceptGhost()` 在 dispatch 前快照预测来源；live preview 行焦点判断改为包含行尾（`cursor <= block.to`）。
- **教训**: 结构化编辑辅助不等于用户语料，不能直接喂给 N-gram；任何依赖插件状态的逻辑都要注意 CM6 `dispatch()` 的同步 update 副作用；行尾是编辑态的一部分，焦点范围判断必须包含 `block.to`。

## BUG-038: Windows 中文输入法换行后焦点错位与光标跳跃

- **现象**: (1) 第一行输入确认后按 Enter 换到第二行，再次启动输入法时临时字符打断 IME 运行，字符自动追加到上一行，需连续换行两次才能避免；(2) 选取文章中部的行进行输入时，选中文字后输入聚焦框自动跳到文章末尾或随机中间行，文字输入到错误位置，光标聚焦位置也错误。
- **根因**: 三重因素：(a) CM6 在 Chrome 126+ 上启用了 EditContext API，该 API 在 Windows 中文 IME 下存在光标坐标报告错误，导致候选窗口和合成结果错位；(b) `cm6-ghost-text.ts` 的 `domEventHandlers.keydown` 中 Tab/Escape 拦截未检查 IME 合成状态，IME 候选窗关闭键被误拦截；(c) `MarkdownEditor.vue` 中 Escape keymap 未检查 IME 状态，live preview 的 `unpinFocusedBlock` 在合成期间被错误触发。
- **根因类别**: 渲染管线 / 跨平台兼容
- **修复**: (1) 在 `MarkdownEditor.vue` 的 `onMounted` 中，`new EditorView` 之前设置 `EditorView.EDIT_CONTEXT = false`（通过 `as any` 绕过 TS 类型检查，因 6.43.x 未导出该静态属性）；(2) `cm6-ghost-text.ts` 的 `domEventHandlers.keydown` 中增加 `view.composing || view.compositionStarted` 守卫，IME 合成期间返回 false 不拦截 Tab/Escape；(3) `MarkdownEditor.vue` 的 Escape keymap 增加 `view.composing || view.compositionStarted` 守卫；(4) `@codemirror/view` 从 6.43.0 升级到 6.43.1。
- **教训**: CM6 的 EditContext API 在 Windows 中文 IME 下尚未稳定，发行前必须显式禁用；所有自定义 keymap 和 domEventHandlers 的键盘拦截必须在处理逻辑前检查 `view.composing` / `view.compositionStarted`，不得假设 IME 不会触发这些按键。

## BUG-039: Live Preview 残留 IME 焦点漂移与空白页视图切换缺失

- **现象**: 修复 BUG-038 后，中文 IME 仍可能在换行后把组合占位字符贴回上一行；在示例文章中部点击 rendered block 后输入，焦点会跳到文档尾部或其他块；默认空白编辑窗口无法切换即时渲染和分栏显示。
- **根因**: `cm6-live-preview.ts` 的 `isImeActive()` 只检查插件本地 `isComposing` 和 `input.type.compose` transaction，漏掉 CM6 自身的 `view.composing` / `view.compositionStarted`；rendered block 的普通点击仍依赖 `Decoration.replace` widget 的默认 DOM→文档位置映射，点击块内文本时可能映射到错误位置；空白页视图切换按钮被 `v-if="activePath"` 隐藏，且 `watch(activePath)` 在空白文档和视图切换重建编辑器时不会重连 predictor。
- **根因类别**: 渲染管线 / 状态管理
- **修复**: `cm6-live-preview.ts` 统一将 `view.composing` / `view.compositionStarted` 纳入 IME active 判定；普通点击 `.cm-live-block` 时通过 `data-block-key` 显式定位到源码 block 起点并聚焦；光标所在空行的相邻块不做 `Decoration.replace`，避免换行后 IME 锚点回落上一行；`toggleBlockRender()` / `unpinFocusedBlock()` 的焦点判断包含行尾；`cm6-ghost-text.ts` 在接受 ghost 前先清空插件状态和 pending timer，避免 `dispatch()` 同步触发旧建议重入；`NotebookHome.vue` 移除视图切换按钮的 `activePath` 条件，并改为监听 `[activePath, viewMode]` 重连 predictor。
- **教训**: IME 防护必须覆盖 CM6 状态、DOM composition 事件、transaction 注解和所有异步重建入口；`Decoration.replace` widget 的 DOM 定位不能用于编辑入口，用户点击 rendered block 时必须显式映射到文档位置；空白文档也是编辑器状态，不能用 `activePath` 作为视图能力开关。

## BUG-048: IME 在标题换行及格式编辑后发生输入回跳、吞字符和渲染块漂移

- **现象**: 标题后按 Enter 并立即使用中文输入法时，首个临时字符概率性回跳到标题行末并取消输入，中文标点被吞一到两次，Markdown 符号可能不隐藏；中文输入法介入格式编辑后，文本还会概率性闪烁并失焦，随后连续按 Backspace 时内容复制到文末或随机下方行。
- **根因**: 标题后插入换行的同一次更新中，`cm6-live-preview.ts` 立即用 `Decoration.replace` 将上一行替换成 Widget，Chrome 在紧随其后的 `compositionstart` 前丢失新空行 DOM 输入锚点；IME 活跃时停止重建 `DecorationSet`，但文档变化后没有用 `update.changes` 映射旧装饰位置，导致 Widget 持有旧坐标并漂移；`compositionend` 延迟重建与紧随其后的标点或 Backspace 同步重建并行；`cm6-ghost-text.ts` 同时保留 composition 延迟预测和普通输入防抖预测，可能生成两次 stale ghost widget。
- **根因类别**: 渲染管线 / 跨平台兼容
- **修复**: 光标位于格式块下一空行时，该相邻块保持源码状态，不执行 `Decoration.replace`；当下一行出现实际文本、光标离开或 composition 结束后再恢复即时渲染，从状态上消除 IME 建立锚点前的竞态，不依赖延迟时间。IME 活跃和冷却窗口内的文档变化统一执行 `decorations.map(update.changes)`；冷却窗口内的标点、Backspace 和 Delete 只续期同一个重建任务；ghost text 在任意文档或选区变化时同时取消普通防抖与 composition 延迟预测，再创建唯一任务。
- **教训**: 输入法防护必须从 `compositionstart` 前一个可能改变 DOM 锚点的事务开始，计时器只能降低复现概率，不能作为正确性约束；暂停 Decoration 重建不等于可以保留原坐标，任何 `docChanged` 都必须 map 现有 DecorationSet；composition 冷却任务必须具备去重和续期语义，不能与后续输入并行修改 Widget DOM。

## BUG-049: 静态 IME 相邻空行守卫导致格式符号持续可见

- **现象**: 即时模式中，光标停在格式化块的下一空行时，上一行的 `#`、`**` 等标记持续可见；必须输入正文、移动光标或再换行一次才恢复渲染。
- **根因**: `touchesEmptyCursorLine` 为保证 Windows IME 能读取上一行源码上下文，禁止相邻块使用 `Decoration.replace`，但直接退回完整源码造成视觉状态与即时预览不一致。移除守卫、时间窗口、`contentEditable=false` 均无法同时满足 IME 正确性和视觉一致性。
- **根因类别**: 渲染管线 / 跨平台兼容
- **修复**: 对相邻空行上方的 `heading` 和 `paragraph` 使用局部源码保留式装饰：源码文本节点继续留在 DOM，CodeMirror Markdown 语法树提供标题字号、粗体、斜体等排版；`HeaderMark`、`EmphasisMark`、`CodeMark`、`LinkMark` 等定界符通过 `Decoration.mark` 折叠为零字号。其他位置仍使用完整 HTML Widget，复杂结构块继续保留安全守卫。
- **教训**: IME 兼容的核心契约是源码文本上下文必须留在编辑 DOM；视觉隐藏不等于删除文本节点。局部混合渲染可以把兼容性成本限制在危险边界，无需在全量 Widget 与全量源码之间二选一。

## BUG-050: 选区格式操作重复包裹 Markdown 定界符

- **现象**: 已加粗文字再次点击加粗不会取消格式，而会从 `**文字**` 叠加为 `****文字****`；选区只包含正文时也无法识别外围定界符。
- **根因**: `NotebookHome.onBubbleFormat()` 只执行 `${open}${selected}${close}` 字符串拼接，没有解析选区内部或相邻位置的既有定界符，也没有在事务后恢复正文选区和编辑器焦点。
- **根因类别**: 渲染管线
- **修复**: 新增纯函数格式命令，统一处理“选区包含定界符”和“定界符位于选区外围”两种情况；相同格式再次执行时解包，清除格式递归移除常见行内定界符及标题/引用前缀，事务后恢复正文选区与焦点。
- **教训**: Markdown 格式按钮必须是幂等切换命令，不能是无条件字符串包裹；格式事务必须同时定义 changes、selection 和 focus 三个结果。

## BUG-051: Word 式固定格式栏在 UI 重构后缺失

- **现象**: 用户只能选中文字后发现浮动气泡，无法像常见文字处理器一样随时选择正文、标题和引用样式，增加发现与记忆成本。
- **根因**: `migration-map.md` 保留了 FormatToolbar 规格，但纸张主题 UI 重构只实现 FormatBubble，固定工具栏组件和页面接入丢失。
- **根因类别**: 渲染管线 / 状态管理
- **修复**: 新增固定 `FormatToolbar`，提供正文、标题 1/2/3、引用预设及常用行内格式、链接、清除格式；与 FormatBubble 共用同一格式命令管线。
- **教训**: UI 重构不能把“有替代入口”视为功能等价；固定入口解决发现性，浮动入口解决操作距离，两者服务不同用户任务。

## BUG-052: IME 中间值回传覆盖后续中文标点

- **现象**: 中文输入法下首次输入标点可能被吞，需要再次按键才能写入；格式操作后更容易出现。
- **根因**: （1）`MarkdownEditor` 将每个 composition 中间文档值 emit 给父组件，父组件回传为 `modelValue` 时仅用”是否等于当前文档”判断是否同步；同时 `EditorView.updateListener` 与 DOM `keyup/mouseup/paste` 监听器重复执行同步，快速输入中旧中间值可能晚于下一次标点事务返回并全量覆盖文档。格式按钮点击后未恢复焦点进一步放大输入取消概率。（2）**残留根因（本次修复）**：`FormatToolbar` 所有按钮均使用 `@mousedown.prevent` 阻止焦点转移，导致 IME 保持活跃时点击格式按钮 → `applyPendingFormat` 无 `view.composing` 检查 → `view.dispatch()` 在 composition 进行中直接插入格式标记字符 → 破坏 CM6 的 composition 事务 → 首个输入字符被吞。
- **根因类别**: 状态管理 / 跨平台兼容
- **修复**: （第一轮）记录 EditorView 内部发出的最近文档值，watch 收到自身回声时直接忽略；移除重复的 DOM 同步监听器，以 `EditorView.updateListener` 作为唯一内容同步入口；格式事务完成后显式恢复 EditorView 焦点和正文选区。（第二轮）`applyPendingFormat` 入口增加 `view.composing || view.compositionStarted` 守卫；IME 活跃时将格式变更推迟到 `compositionend` 后执行（`setTimeout(0)` 确保 CM6 完全退出 composition 状态再 dispatch）。
- **教训**: 双向绑定的编辑器必须区分内部回声与真实外部更新；IME 期间不能只靠字符串”当前是否相等”判断同步方向；`@mousedown.prevent` 的副作用是编辑器保持焦点 → IME 保持活跃 → 任何 document dispatch 都会破坏 composition 事务。

## BUG-053: 分栏预览首行被遮挡、逐行产生额外段距且源码区带渲染样式

- **现象**: 分栏右侧不显示第一行，每一行之间出现异常大间距；左侧源码中的标题等内容仍显示为渲染态字号。
- **根因**: `renderLineByLine()` 将每一行分别交给 marked，普通行全部生成独立 `<p>`；固定格式栏覆盖了分栏内容顶部；源码编辑器虽关闭 live preview，仍加载语义化标题高亮样式。
- **根因类别**: 渲染管线 / 布局
- **修复**: 分栏右侧改为完整文档单次 `renderMarkdown()`；分栏内容为固定栏预留顶部空间；源码编辑器增加 `sourceOnly` 高亮模式，取消标题放大、粗体等渲染态排版。
- **教训**: Markdown 块结构只能在完整文档上下文中解析；关闭 Widget 预览不等于关闭语义化排版。

## BUG-054: 固定格式栏错误复用选区后格式化逻辑

- **现象**: 固定 Word 式格式栏必须先选择文字才能生效，与格式气泡行为重复，无法先选格式再输入。
- **根因**: 固定栏和浮动气泡共用 `formatSelection`，页面没有维护待输入格式状态，编辑器也没有处理预插入定界符及 Enter 结束状态。
- **根因类别**: 状态管理 / 交互
- **修复**: 固定栏改为 `pendingFormat` 状态机：先选择行内或段落格式，再在光标处输入；非 composition 的 Enter 结束预选并恢复正文，IME 候选确认 Enter 不拦截。浮动气泡继续负责选区后格式化。
- **教训**: 固定预设栏和选区气泡是两类不同命令模型，不能只复用视觉控件而忽略输入状态机。

## BUG-055: 全角 Markdown 格式标记无法识别

- **现象**: 中文输入法输出 `＃`、`＊`、`＞` 等全角字符时，渲染器和即时预览不识别格式，视觉状态频繁切换。
- **根因**: marked 输入及 `parseLiveBlocks()` 仅识别 ASCII Markdown 定界符，未在解析边界归一化全角等价字符。
- **根因类别**: 渲染管线 / 跨平台兼容
- **修复**: 新增全角 Markdown 语法归一化函数，并在完整渲染及即时块解析入口统一调用；只归一化解析输入，不改写用户保存的原始文本。
- **教训**: 输入法兼容不只涉及 composition 事件，也包括语法字符集合；归一化应位于解析边界，避免污染文件内容。

---

## BUG-046: E2E 持久化测试 — 书签 aria-label 在内容替换后变化

- **现象**: 3 个持久化测试（V3 刷新后保持、V3 切换切回、V6 多次刷新）在 `page.reload()` 后 `switchToNote(原标题)` 超时 30s，找不到匹配的书签圆点
- **根因**: `extractTitle()` 只匹配 `# H1` 标题行。`typeInEditor` 使用 Ctrl+A+Backspace 清除全部内容（含原标题 H1），然后输入 `## H2` 或 `### H3` 测试标记。`refreshDocument → updateDocument → extractTitle()` 找不到 H1 → 回退到文件名作为 title。书签圆点 `aria-label` 从原标题（如"子文件夹笔记"）变为文件名（如"笔记A"）。测试在刷新前保存的 `firstLabel` 在刷新后不再有效。
- **根因类别**: 类型边界（测试与 `extractTitle` 行为假设不一致）
- **修复**: 3 个测试改用 `# H1` 标记 + 在 `waitForAutoSave` 后重新读取 active dot 的 `aria-label` 用于刷新后查找（而非使用 `typeInEditor` 前的 label）
- **教训**: **当测试用 `typeInEditor`（Ctrl+A+Backspace 全量替换）时，必须考虑内容替换对元数据（标题）的影响。** `extractTitle` 只读 H1 的约定需要通过测试策略体现：要么用 H1 标记，要么保存后的 label 重新读取

## BUG-047: FileDrawer 删除文件后 overlay 未关闭 — 测试阻塞

- **现象**: V6 删除重建测试中，通过 FileDrawer 右键删除文件后，drawer-overlay 仍覆盖全视口。后续点击 `.wing-new-btn` 被 overlay 拦截，测试超时 90s
- **根因**: `FileDrawer.handleContextMenuDelete()` 调用 `closeContextMenu()` 后仅 emit `delete-file`，未调用 `close()`。文件删除后 drawer 仍保持打开状态。测试层面，`@keydown.escape` 绑定在 `.drawer-overlay` 上但该元素缺少 `tabindex`，键盘事件无法到达（与 BUG-041 同模式）。`@click.self="close"` 需要点击在 overlay 自身（非子元素）上才能触发
- **根因类别**: 状态管理（组件状态未在操作完成后重置）+ 渲染管线（键盘事件可达性）
- **修复**: （测试层）在验证删除完成后，通过点击 overlay 右侧区域（`position: { x: 600, y: 300 }`，drawer panel 宽度 ~300px 之外）触发 `@click.self="close"` 关闭 drawer。**建议软件层修复**: `handleContextMenuDelete` 末尾调用 `close()`，或在 overlay 添加 `tabindex="-1"` 使 Escape 键可关闭
- **教训**: **`@keydown` 绑定在无 `tabindex` 的 `<div>` 上不可靠。** 需要通过键盘关闭的 overlay 必须有 `tabindex="-1"` + 打开时 `focus()`（与 BUG-041 相同根因）。另外，FileDrawer 中任何破坏性操作（删除/重命名）完成后都应考虑关闭 drawer

---

> 关联文档：`CLAUDE.md` §5.0（错题本必读）、§5.6（BUG 修复前置规则）、§5.9（代码审查易错点）

---

## BUG-062: 删除后书签残留（recentNotes/index/removeDocument）— 左侧孤儿书签

- **现象**: GUI 删除 `GUI-mqqksjaw-综合旅程.md` 后，文件树中条目消失，但左侧书签仍残留 `.wing-bookmark-dot[aria-label="GUI-mqqksjaw-综合旅程"]`，点击后停留在无标题空态且无错误提示
- **根因**: `IndexService.removeDocument()` 与增量索引清理链未统一更新 `allDocuments/tagIndex/wikiIncoming/wikiOutgoing/recentNotesList/searchIndex` 五维索引，`indexStore.recentNotes` 与文件树脱节；UI 侧按文件树隐藏/展示，未阻断索引层孤儿入口
- **根因类别**: 状态管理 / 文件IO / 索引/搜索
- **修复**: Codex M-R2-F1 收口落地统一一致性修复：`IndexService` 新增 `clearIndexesForPaths` 并在 `removeDocument` 中清理 `allDocuments/tagIndex/wikiOutgoing/wikiIncoming/recentNotesList/SearchEngine`；`synchronizeFromFileTree` 直接以文件树白名单过滤 `allDocuments/recentNotesList/tagIndex/wikiOutgoing/wikiIncoming` 已知路径，避免只从 `allDocuments` 推导 stale；`index` store 同步刷新 tags/recent/documentCount；`NotebookHome` 删除和读取异常按归一化路径清理 active note 全量状态并刷新文件树。E2E 新增 `J2b: 删除新建笔记后书签与文件树无残留`，覆盖新建-保存-删除-刷新后文件树与左侧书签均无残留。
- **验收**: `vitest` 145/145 PASS；`16-user-journeys.spec.ts` Chromium/Firefox 155/155 全量矩阵中的 J2b 均 PASS；内置浏览器 GUI 复验 `gui-final-mqqpzx9i.md` 删除后 `treeAfterDelete=0`、`dotAfterDelete=0`、刷新后 `treeAfterReload=0`、`dotAfterReload=0`。
- **教训**: 索引更新必须保证五维状态闭环，左侧书签仅是索引症状；任何删除必须同时清掉索引与搜索映射，避免点击到残留的孤儿入口

## BUG-056: 图片上传写入链路未真正按图片二进制闭环

- **现象**: Ctrl+V 粘贴图片后 Markdown 会插入 `./assets/img_*.png`，但桌面端实际写入的是 data URL 文本；子目录笔记仍插入 `./assets/...`，从该笔记相对解析会指向错误位置；上传后文件抽屉不刷新，看不到新 `assets` 文件。
- **根因**: `useImageUpload` 使用 `readAsDataURL()` 后直接调用文本 `fs.writeFile()`；`IFileSystemService`/Tauri IPC 缺少二进制读写契约；图片引用路径未基于当前笔记目录计算；上传完成未触发文件树刷新。
- **根因类别**: 文件IO / 跨平台兼容
- **修复**: 增加 `writeBinary/readBinary` 文件系统接口与 Tauri `write_binary_file/read_binary_file` 命令；上传时剥离 data URL 前缀后写 base64；按当前笔记目录生成 `./assets/...` 或 `../assets/...`；上传成功后刷新文件树。
- **教训**: 文件名和 Markdown 引用出现不代表二进制资产闭环成立，图片上传必须同时验证写入字节、相对路径和文件树可见性。

## BUG-057: Live Preview Wiki-link 存在性状态不随文件树/索引刷新

- **现象**: 即时预览中 `[[不存在笔记]]` 和 `[[快速入门]]` 视觉状态相同，缺少 dead link 标识；新建/删除笔记后 Wiki-link 状态不会稳定刷新。
- **根因**: renderer 的 Wiki-link 扩展没有接收 `wikiLinkExists` 判定函数；live preview 扩展也没有把 NotebookHome 的文件树/索引状态传入渲染管线，且刷新后没有触发编辑器插件重建。
- **根因类别**: Wiki-link / 状态管理 / 渲染管线
- **修复**: `renderMarkdown` 支持 `wikiLinkExists` 选项并通过 renderer 扩展写入 `wikilink--dead`；`MarkdownEditor`/`cm6-live-preview` 接收存在性回调和 revision；文件树或索引更新后递增 revision 强制重建预览。
- **教训**: 渲染器状态不能隐式读取应用层 store，跨包扩展点必须显式传参，并在外部数据变化时有可观测的重建信号。

## BUG-058: 即时预览任务 checkbox 点击不写回 Markdown 源码

- **现象**: GUI 中点击 `- [ ] first task` 的可视 checkbox 后，按钮状态和复制出的 Markdown 仍是 `- [ ]`，任务未完成状态没有持久化。
- **根因**: 任务 checkbox 位于 CM6 `Decoration.replace` widget 内，原逻辑依赖 click/change 冒泡或 widget 内回调；CodeMirror 的 pointer/mouse 处理可能先移动焦点并替换 DOM，导致后续事件拿不到稳定 widget 或被拦截。
- **根因类别**: 渲染管线 / 状态管理
- **修复**: 在编辑器根节点 `pointerdown` 捕获阶段拦截 `.cm-task-toggle`，在 CM6 改写 DOM 前通过 `data-block-from` 定位源码行并直接 dispatch `- [ ]`/`- [x]` 替换；widget 内事件只负责阻止焦点抢占。
- **教训**: CM6 widget 内的交互不要依赖普通 click 冒泡完成状态写回，涉及源码变更的控件应在捕获阶段先完成文档事务。

## BUG-059: TemplateDialog 自定义模板卡片嵌套 button

- **现象**: Vite 在运行时提示 `<button> cannot be child of <button>`；自定义模板卡片本身是按钮，内部删除模板也是按钮。
- **根因**: 模板选择和模板删除两个交互被放进同一个 button 层级，违反 HTML 交互内容嵌套规则，可能影响可访问性与后续事件行为。
- **根因类别**: 渲染管线 / 可访问性
- **修复**: 自定义模板卡片改为 `role="button"` + `tabindex="0"` 的非 button 容器，保留 Enter/Space 键盘选择；内部删除仍使用独立 `<button>`。
- **教训**: 卡片可点击不等于必须使用 button 包裹全部内容；当卡片内部还有独立操作按钮时，应拆分语义角色，避免交互元素嵌套。

## BUG-060: Firefox 下标题换行后中文 IME 提交无法恢复 live preview

- **现象**: 在即时模式标题行按 Enter 后输入中文标点，Firefox 中内容实际保存为 `# 标题\n中。`，但标题行没有恢复为 `.cm-live-block[data-block-type="heading"]`，测试持续看到源码态。
- **根因**: live preview 为避免 IME 组合输入破坏光标，组合期间跳过 decoration rebuild；但 Firefox 的 composition/transaction 状态在提交后仍可能让重建被清掉或延后不足，导致标题上方源码态没有在中文提交后恢复。
- **根因类别**: 渲染管线 / 跨浏览器兼容
- **修复**: 对 composition transaction 做显式判定；组合结束且当前行已有提交文本时延迟重建 decorations，空行时保留上一块源码上下文，确保中文提交后标题恢复渲染且光标仍停留在新行。
- **教训**: IME 场景不能只按 Chromium 行为判断，至少要覆盖“标题 Enter → 中文输入 → 标点提交 → live preview 恢复”的跨浏览器闭环。

## 检查清单增补

- [ ] 图片上传必须验证三件事：写入内容不是 data URL 文本、Markdown 相对路径从当前笔记出发正确、文件树能看到 assets 文件。
- [ ] 跨包 renderer 扩展需要应用状态时，必须显式传入选项并提供外部状态变化触发重建的 revision/key。
- [ ] CM6 `Decoration.replace` widget 内的可点击状态控件，应优先在捕获阶段完成文档写回，避免 DOM 被 CM6 先替换。
- [ ] 可点击卡片内若存在删除/更多等独立动作，不允许使用 button 嵌套 button。
- [ ] live preview 的 IME 验证必须覆盖 Firefox，尤其是标题/列表等源码保留块后紧跟中文组合输入。

## BUG-061: 清理默认训练元数据时复用 `trainedPaths` 对象导致状态泄漏

- **现象**: DeepSeek 清理提交将 `loadTrainingMeta()` 的空状态返回值简化为 `{ ...DEFAULT_TRAINING_META }`。训练文件后如果清空 `localStorage` 或遇到损坏/旧版本 meta，再次读取训练元数据仍可能带出之前训练过的路径。
- **根因**: `DEFAULT_TRAINING_META.trainedPaths` 是引用类型。浅拷贝只复制顶层对象，`trainFile()` 会原地写入 `meta.trainedPaths[path]`，从而污染默认对象。
- **根因类别**: 状态管理 / 类型边界
- **修复**: 新增 `createDefaultTrainingMeta()`，所有默认返回路径都重新创建空 `trainedPaths`；补充回归测试覆盖“训练 → 清空 localStorage → 再读取应为空”的场景。
- **教训**: 包含对象/数组的默认状态不能用浅拷贝作为可变运行态返回值；返回前必须深拷贝可变字段，或提供工厂函数。

## 检查清单增补

- [ ] 默认状态对象包含 `Record` / `Array` / `Map` 等可变字段时，必须通过工厂函数创建运行态副本，禁止直接浅拷贝后写入。

## BUG-063: M-R3 小屏浮层与导出开关可访问性未形成发布级闭环

- **现象**: M-R3 响应式审计发现通用 modal/命令面板依赖固定宽度（480/520/560px），在 360px 视口下需要显式 `max-width` 和移动端高度约束才能证明不会横向溢出；导出对话框的三个选项开关仅支持鼠标点击，缺少 `role="switch"`、`aria-checked`、焦点环和 Space/Enter 键盘切换。
- **根因**: 早期 E2E 只覆盖桌面 happy path 与可见性，未用 360/768/1280/1440 多视口验证核心浮层；`SettingsDialog` 已补齐 switch 语义，但 `ExportDialog` 的同类 toggle 没有同步治理。
- **根因类别**: 渲染管线 / 可访问性 / 跨平台兼容
- **修复**: `dialog.css` 为通用 `.modal-card` 增加视口约束和 480px 以下移动端规则；`CommandPalette.vue` 为命令面板增加小屏 `max-width/max-height` 与 footer 换行；`ExportDialog.vue` 为三个导出选项补齐 switch 语义、键盘切换和 focus ring；新增 `17-responsive-a11y-visual.spec.ts` 覆盖 360x740、768x1024、1280x800、1440x900 下 app shell、设置、搜索、模板、导出、文件抽屉与 switch 键盘行为。
- **验收**: 新增 M-R3 规格 Chromium/Firefox 均 PASS；Chromium 全量 E2E 161/161 PASS；Firefox 全量 E2E 161/161 PASS；`typecheck`、ESLint、stylelint、Prettier 触达文件、Vitest 145/145、build 均通过。
- **教训**: 发布前 UI 验收不能只看桌面宽屏可见性；任何自绘 toggle 必须按同一语义标准补齐 role/state/keyboard/focus，且核心浮层必须在 360px 视口下有自动化断言。

## 检查清单增补

- [ ] 新增或修改浮层时，必须覆盖 360px、768px、1280px、1440px 视口下无横向溢出和 Escape 退出。
- [ ] 自绘 toggle/switch 控件必须具备 `role="switch"`、`aria-checked`、键盘切换和可见 focus ring。

## BUG-064: Tauri 真实文件系统无法处理前端 `/note.md` 路径

- **现象**: M-R4 审计发现前端统一使用 `/note.md`、`/folder/a.md` 作为笔记本内路径；Rust `resolve_safe_path()` 直接把该字符串交给 `Path::new()`，在 Tauri 真实 FS 下会被解释为 OS 绝对路径，导致列目录、读取、写入、重命名等命令无法稳定落在 notebook root 内。
- **根因**: Web MockFS 的路径契约是“以 `/` 表示笔记本根”，Rust 层未做相同归一化；`root.join(Path::new("/x.md"))` 不等价于 `root/x.md`，而是尝试逃出 root。既有 Rust 测试只验证相对路径，未覆盖前端真实传入的 notebook-root marker。
- **根因类别**: 文件IO / 跨平台兼容 / 类型边界
- **修复**: `resolve_safe_path()` 将前导 `/` 视为 notebook-root marker 并剥离，再做绝对路径和 `..` 逃逸检查；新增 Rust 单元测试覆盖 `/notes/test.md`、`/`、真实绝对路径拒绝。`fs_ops.rs` 增加可测试内部函数，并在真实临时目录中验证文本写读、二进制写读、重命名、列目录和路径逃逸拒绝。
- **验收**: `cargo fmt --check` PASS；`cargo test` 6/6 PASS；`pnpm.cmd --filter @jotluck/app tauri:build:debug` PASS。
- **教训**: Web/MockFS 的路径语义必须在 Tauri/Rust 边界显式归一化；任何跨端文件接口测试都必须使用真实前端会传入的路径形态，而不是只测 Rust 自然相对路径。

## BUG-065: Tauri v2 旧式 `plugins.fs.scope` 配置导致桌面端启动即 panic

- **现象**: `pnpm.cmd --filter @jotluck/app tauri:dev` 启动到 `target\debug\JotLuck.exe` 后直接 panic：`PluginInitialization("fs", "Error deserializing 'plugins.fs' ... unknown field scope, expected requireLiteralLeadingDot")`。
- **根因**: `tauri.conf.json` 仍保留旧版 `plugins.fs.scope.allow/deny` 配置；当前 `@tauri-apps/plugin-fs` v2.5.x 已不接受该字段。权限已经在 `capabilities/default.json` 中配置，旧 scope 块既无必要又会阻断运行时初始化。
- **根因类别**: 跨平台兼容 / 类型边界
- **修复**: 删除 `tauri.conf.json` 中过时的 `plugins.fs.scope` 块，保留 shell 插件配置；Tauri 文件访问继续由 capabilities 和自定义 IPC 控制。
- **验收**: 修复前 `tauri:dev` 复现 panic；修复后 `tauri:dev` 启动到 `[app_lib][INFO] JotLuck Tauri backend initialized`；最终 `tauri:build:debug` PASS 并生成 `JotLuck_0.1.0_x64-setup.exe`。
- **教训**: Tauri 插件升级后，能 build 不代表运行时配置有效；桌面端发布闸门必须包含 `tauri:dev` 实际启动，不能只跑 bundle build。

## 检查清单增补

- [ ] Tauri/Rust 文件命令必须覆盖前端 notebook-root 路径形态：`/`、`/note.md`、`/dir/note.md`。
- [ ] Tauri 插件配置变更后必须同时跑 `tauri:dev` 和 `tauri:build:debug`，防止 build 通过但运行时插件初始化失败。

## BUG-066: 启动后台版本检查未尊重自动检查开关

- **现象**: M-R5 网络隐私审计发现，应用初始化 15 秒后会无条件调用 GitHub release API；即使用户未开启或已关闭 `jotluck:version:autoCheck`，仍可能产生启动联网。
- **根因**: `useVersionCheck` 内部的 `checkForUpdates()` 会读取自动检查设置，但 `NotebookHome.vue` 挂载后的后台定时器直接调用 `checkNow()`，绕过了设置门控。
- **根因类别**: 跨平台兼容 / 状态管理 / 安全隐私
- **修复**: 在 `NotebookHome.vue` 增加 `VERSION_AUTO_CHECK_KEY` 与 `shouldRunBackgroundVersionCheck()`，后台延迟检查前先读取本地开关；新增 E2E 拦截 `https://api.github.com/**`，验证自动检查关闭时启动 16 秒内请求数为 0。
- **教训**: 所有后台网络行为必须在调用点和服务点都受用户设置门控；测试要用路由拦截证明“没有请求”，不能只验证 UI 设置存在。

## BUG-067: XLSX 导出依赖 `xlsx` 存在高危漏洞且 npm 无可用修复版

- **现象**: `pnpm audit --audit-level high` 报告 `xlsx@0.18.5` 存在 Prototype Pollution 与 ReDoS 高危漏洞；npm registry 最新仍为 0.18.5，审计建议的 0.19.3/0.20.2 不可直接通过 npm 升级获得。
- **根因**: ADR-009 锁定 SheetJS `xlsx` 作为运行时导出依赖，但发布安全闸门未验证“存在可安装的 patched release”。
- **根因类别**: 导出 / 安全隐私 / 依赖管理
- **修复**: 移除根目录和 app 的 `xlsx` 依赖，改用 `write-excel-file@4.1.1` 生成 XLSX Blob；保留 Markdown 表格提取与多 sheet 导出行为；新增 E2E 验证 XLSX 下载文件大小与 ZIP 包头 `PK`；同步 ADR/PRD/TAD/progress 中的导出依赖描述。
- **教训**: 对无可用修复版本的高危运行时依赖，发布收口阶段应优先替换而不是压低审计等级；导出格式替换必须有下载文件层面的可观测断言。

## 检查清单增补

- [ ] 后台网络请求必须同时验证“开启时可用”和“关闭时不发请求”，关闭态应通过网络拦截计数证明。
- [ ] 发布依赖审计中若 patched version 不可从当前 registry 安装，必须替换依赖或记录明确发布阻断，不能仅升级到 latest。
- [ ] 导出库替换后，E2E 必须读取下载文件并验证格式特征或内容相关性。

## BUG-068: Windows Playwright webServer 使用 `pnpm` 导致最终 E2E 启动不稳定

- **现象**: M-R7 最终 E2E 在 Windows 环境下由 Playwright `webServer` 拉起 dev server 时可能超时，手动命令可运行但测试入口不稳定。
- **根因**: `packages/app/playwright.config.ts` 使用 `pnpm --filter @jotluck/app dev`，Windows 下非 shell 场景对 shim 解析不如 `pnpm.cmd` 稳定；30s 超时对 Vite 冷启动和依赖扫描偏紧。
- **根因类别**: 跨平台兼容 / 测试基础设施
- **修复**: webServer 命令改为 `pnpm.cmd --filter @jotluck/app dev`，timeout 提升到 60s。
- **教训**: 发布级 E2E 配置必须使用目标平台稳定可执行入口；Windows 下优先显式 `.cmd`，避免 shell shim 假设。

## BUG-069: NotebookHome 主 chunk 过大且 Tauri event 静态/动态导入混用

- **现象**: M-R7 前生产构建存在 `NotebookHome` chunk 超过 500kB 的 Vite warning，并提示 `@tauri-apps/api/event.js` 同时被静态和动态导入。
- **根因**: `NotebookHome.vue` 静态引入多个重型对话框和导出路径，导出库被卷入页面主 chunk；同一 Tauri event 模块在页面中动态导入，在 Tauri IPC 层静态导入。
- **根因类别**: 性能体积 / 构建管线
- **修复**: 页面弹层改为 `defineAsyncComponent`；Tauri event `listen` 改为静态导入；Vite manualChunks 拆分 CodeMirror、导出库、Markdown、Vue/Pinia、Tauri API。
- **教训**: 发布前不能把构建 warning 全部当作噪声；当 warning 指向真实首屏体积或重复导入路径时，应在最终 RC 前收口。

## BUG-070: Playwright HTML report 作为跟踪文件污染发布候选补丁

- **现象**: M-R7 运行 E2E 后 `e2e/report/index.html` 反复出现在工作区 diff 中，并让广义 `prettier --check e2e` 扫到生成物。
- **根因**: Playwright HTML report 曾被纳入 git 跟踪，且 `.gitignore` 未覆盖 `e2e/report/`。
- **根因类别**: 测试基础设施 / 发布卫生
- **修复**: 删除 tracked `e2e/report/index.html`，并在 `.gitignore` 增加 `e2e/report/`。
- **教训**: 发布补丁必须把测试证据和测试生成物分开；HTML report、截图、视频、trace、`test-results/` 不应作为源码合入。

## 检查清单增补

- [ ] Windows 发布级 Playwright 配置应使用 `pnpm.cmd`，并给 dev server 冷启动留出足够 timeout。
- [ ] 生产构建 warning 若涉及首屏 chunk 或同模块静态/动态导入混用，必须在 RC 前解释、修复或记录为明确非阻断项。
- [ ] E2E/测试报告生成目录必须在 `.gitignore` 中，已跟踪生成物必须从源码补丁中移除。

## BUG-071: 桌面安装版双击 Markdown 文件后打开空白编辑页

- **现象**: 在 Windows 桌面双击 `.md` 文件会启动 JotLuck，但应用只显示无标题空白编辑页，目标文件内容没有被读取，文件抽屉也可能停留在“未打开笔记本”。
- **根因**: Tauri 启动参数只以裸绝对路径字符串保存并发送给前端；前端把它当作笔记本内部路径使用，没有先打开目标文件的父目录作为 notebook root。启动期事件还可能早于前端 listener 注册，导致待打开文件丢失；已有运行实例也没有单实例路由接收后续双击文件。
- **根因类别**: 文件IO / 跨平台兼容 / 状态管理
- **修复**: Tauri 将启动参数结构化为 `{ absolutePath, notebookRoot, relativePath }`；前端启动时先消费 `get_opened_file`，再监听 `opened-file` 事件；收到文件后先 `openNotebookAt(notebookRoot)`、重建索引，再选中并读取 `relativePath`。补入 `tauri-plugin-single-instance`，让已运行实例也能接收后续文件打开事件。
- **验收**: `cargo test` 覆盖 Markdown 参数解析和非 Markdown 忽略；Chromium/Firefox 自动化通过；安装版 GUI 风险复核已验证真实 `.mdx` 启动会打开父目录并显示目标文件内容。
- **教训**: 桌面文件关联不能只传一个字符串。跨进程/跨平台入口必须把“系统绝对路径”和“应用内部相对路径”拆开，并提供启动前缓存 + 运行时事件两条消费路径。

## BUG-072: 桌面首次启动示例笔记本与默认文档缺失

- **现象**: 安装版启动后顶部仍显示示例笔记本语义，但文件抽屉显示“未打开笔记本”；预设的新手引导和格式示例文档不可见。
- **根因**: Web MockFS 有内存种子数据，但 Tauri 桌面端正常启动时没有真实 notebook root，也没有在用户本地目录创建示例笔记本。MockFS 历史种子还存在乱码和版本缓存，导致 Web/E2E 与安装版真实体验分叉。
- **根因类别**: 文件IO / 状态管理 / 跨平台兼容
- **修复**: 新增 Rust `open_sample_notebook`，在 `%LOCALAPPDATA%/JotLuck/示例笔记本` 下按缺失写入 `快速入门.md`、`格式示例.md`、`项目规划.md`；前端桌面启动时无最近笔记本则打开示例笔记本。MockFS 种子重写为 UTF-8，提升 storage version，确保 Web 测试和桌面默认体验一致。
- **验收**: MockFS 单测覆盖默认文档；E2E Wiki-link、文件抽屉和用户旅程在 Chromium/Firefox 全量通过。
- **教训**: “默认文档”不能只存在于 Web mock。任何首次体验核心内容都必须在真实桌面文件系统中有等价种子路径，并且 root 状态只能有一个来源。

## BUG-073: 中文 IME 输入期间 Live Preview 异步重建导致格式渲染和行号错位

- **现象**: 中文输入法下输入 `# 标题` 和多行正文时，Live Preview/Split Preview 可能出现源码 placeholder 残留、标题渲染错位、行号/块映射不稳定。
- **根因**: `cm6-live-preview` 在 composition 期间虽然有主流程 guard，但异步 `setTimeout`/rAF 入口仍可能在 IME 最后一笔提交前重建 `Decoration.replace`，导致装饰状态基于旧 selection/doc 被写回。
- **根因类别**: 渲染管线 / 跨平台兼容
- **修复**: composition 期间只映射已有 decorations，不重建替换装饰；composition end 后等待一个 macrotask + 一帧，再基于最新 `view.state.doc` 和 selection 重建，并统一检查 `isComposing`、`view.composing`、`view.compositionStarted`。
- **验收**: `14-live-preview-journey.spec.ts` Chromium 12/12 PASS；Firefox 全量中 IME 专项和 Live Preview 全部 PASS。
- **教训**: IME 防护必须覆盖所有异步入口，不能只看 ViewPlugin `update()` 主路径；composition end 后也要给浏览器一次提交落盘窗口。

## BUG-074: Tab 在补全和控件焦点导航之间产生竞态

- **现象**: 用户按 Tab 时，编辑器幽灵文本补全和按钮/弹窗/设置项焦点导航互相抢占；有时控件导航被应用补全吞掉，有时可见 ghost text 因焦点丢失无法接受。
- **根因**: ghost text 的 Tab handler 只检查是否存在 `currentGhostText`，没有验证当前焦点是否仍在 CodeMirror 编辑区、selection 是否绑定当前光标、是否处于 IME composition。失焦后旧 ghost text 也没有及时清理。
- **根因类别**: 渲染管线 / 可访问性 / 状态管理
- **修复**: 新增 `canAcceptGhost(view)`，只有编辑区持焦、光标为空选区、有可见 ghost text、且非 IME 时才消费 Tab；blur 清空 ghost 和 pending timers，focus 重新预测；DOM-level Tab handler 再确认事件目标在编辑区内。
- **验收**: 新增 E2E 覆盖“设置弹窗中 Tab 保留原生焦点导航，回到编辑器后 Tab 接受 ghost text”；Chromium/Firefox 自动补全专项和全量 E2E PASS。
- **教训**: 键盘可访问性和编辑器快捷键必须按焦点域分层，不能用全局状态抢 Tab。

## BUG-075: 欢迎页默认 Markdown 应用设置制造 no-op 成功假象

- **现象**: 欢迎页选择“默认开启 Markdown 格式”后没有实际设置系统默认应用，用户仍需在 Windows 系统设置中手动选择 JotLuck。
- **根因**: Windows 不允许普通应用静默强制成为 `.md` 默认应用；原逻辑把“用户点击过按钮”持久化成近似成功态，产品文案误导用户。
- **根因类别**: 跨平台兼容 / 状态管理
- **修复**: 文案改为说明“安装器已注册 JotLuck 为可选打开程序，系统默认应用需用户手动选择”；按钮改为打开 `ms-settings:defaultapps`，失败时展示手动路径说明；只持久化“已查看/已尝试设置”，不持久化“已成功设为默认”。
- **验收**: 欢迎页 E2E 继续 PASS；安装版 GUI 风险复核已确认文案不会误导用户“已自动设为默认”，并提供系统设置/人工路径。
- **教训**: 操作系统级能力不可达时，产品必须给出真实状态和下一步路径，禁止用本地 flag 伪装成功。

## BUG-076: Windows 桌面图标有效内容面积过小

- **现象**: 安装后桌面快捷方式图标在 Windows 桌面上明显偏小，和常见应用图标比例不一致。
- **根因**: 早期抠图/导出保留了过大的透明边距，512 图标 alpha bbox 约 374x376，真实图形只占画布约 73%。
- **根因类别**: 跨平台兼容 / 发布资产
- **修复**: 基于用户回传的干净图标重新裁切并放大 Windows icon master，重新生成 Tauri app icons 和文件关联 icon；最终 `icon.png` 的 512 alpha bbox 达到 475x477，超过 455px 目标。
- **验收**: 静态 bbox 检查通过；新图标已应用到 Tauri app icon、file icon 和安装包资源，安装版风险复核未再发现桌面图标尺寸阻断。
- **教训**: 图标验收不能只看源图，应检查目标平台最终 `.ico`/PNG 多尺寸中的 alpha bbox，有效内容面积需要量化阈值。

## 检查清单增补

- [ ] 桌面文件关联必须同时验证冷启动和已运行实例两种入口，且启动参数应包含绝对路径、notebook root 与应用内部相对路径。
- [ ] 首次体验默认内容必须在 Web MockFS 和 Tauri 真实 FS 中同时存在，不能只依赖 mock 种子。
- [ ] IME 修复必须覆盖 Chromium 与 Firefox，并验证 Live Preview 标题、空行、普通中文正文三类块映射。
- [ ] Tab 快捷键只能在编辑器持焦且存在当前光标 ghost text 时消费；弹窗、工具栏、设置页必须保留原生焦点导航。
- [ ] 系统级设置按钮不得持久化虚假的成功态；不可自动完成时必须打开系统设置或展示人工路径。
- [ ] 发布图标必须检查最终多尺寸资产的 alpha bbox，Windows 512 图标有效宽高不应小于 455px。

## BUG-077: 文件管理器与系统文件关联的支持格式白名单分裂

- **现象**: 安装版 GUI 复测时发现软件没有形成系统层级和软件内一致的“支持格式”管理。文件抽屉可能展示图片、PDF、备份文件等不可编辑项；`.mdx` 在前端抽屉被当作 Markdown，但 Windows 注册、启动参数、搜索索引和后台监听并未完整支持。
- **根因**: 支持格式被散落在多处硬编码：`fs_ops.rs` 的真实 FS 列表允许图片和 `.txt`，`FileDrawer.vue` 对所有非目录都可触发打开，`IndexService.ts` 只索引 `.md`，`lib.rs`/`tauri.conf.json`/`hooks.nsh` 只处理 `.md/.markdown`，`file_watcher.rs` 与后台训练也有独立白名单。
- **根因类别**: 文件IO / 跨平台兼容 / 状态管理
- **修复**: 新增 `note-files.ts` 统一前端可编辑笔记格式：`.md/.markdown/.mdx/.txt`；文件抽屉只展示目录和支持格式并保留重命名原扩展名；主页选择、新建、重命名、Wiki-link、索引同步全部接入同一规则；MockFS 与 Tauri FS 列表保持一致；Tauri 外部启动参数最终收敛为只接受 `.md/.markdown/.mdx`，`.txt` 仅作为应用内可打开格式；NSIS/Windows 只注册 Markdown 家族 `.md/.markdown/.mdx`，不抢占 `.txt`；Tantivy、文件监听、后台训练同步 `.mdx`。
- **验证**: `typecheck` PASS；`eslint` PASS；`stylelint` PASS；`prettier --check` PASS（仅支持解析的文件）；`vitest` 156/156 PASS；`cargo fmt --check` PASS；`cargo check` PASS；`cargo test` 11/11 PASS；`build` PASS；Chromium 全量 E2E 167/167 PASS（最终表格微调后 J1c 定向复跑 PASS）；Firefox `16-user-journeys` 10/10 PASS；安装版 GUI 风险复核已验证外部 Markdown 单文件只读/编辑、`.txt` 应用内打开、文件抽屉过滤和系统关联边界。
- **教训**: 支持格式是跨层契约，不能散落在 UI、索引、真实 FS、MockFS、启动参数和安装器脚本中。每新增或调整扩展名，必须同时检查“展示、打开、索引、监听、训练、系统注册”六个入口。

## 检查清单增补

- [ ] 文件格式白名单变更必须同步检查：FileDrawer 展示、NotebookHome 打开/新建/重命名、IndexService、MockFS、Tauri FS、启动参数、文件监听、训练服务、NSIS/系统注册。
- [ ] 文件抽屉不得展示不可作为笔记打开的普通资产文件；`assets/` 仍用于图片存储，但默认不作为用户笔记入口暴露。

## BUG-078: 外部 Markdown 文件启动误把父目录当笔记本并触发全局扫描

- **现象**: 在真实 Windows 环境中双击桌面或下载目录中的 `.md` 文件会先进入空白/等待态，随后把父目录当作笔记本递归加载，导致 Desktop/Downloads/用户目录中的大量文本文件、标签和最近笔记污染，严重时卡死。
- **根因**: 早期桌面文件关联修复把外部启动参数转换为 `{ absolutePath, notebookRoot, relativePath }` 后直接调用 `openNotebookAt(parent)`，混淆了“打开一个外部文件”和“用户显式打开一个笔记本文件夹”两种产品语义。前端启动流程随后继续执行文件树加载、索引初始化、标签/搜索/后台训练和最近笔记本写入。
- **根因类别**: 文件IO / 状态管理 / 索引/搜索 / 跨平台兼容
- **修复**: 外部 `.md/.markdown/.mdx` 启动改为 `external-readonly` 单文件会话；默认全屏只读渲染，不显示欢迎页、左翼、右翼、文件抽屉、标签和搜索；点击“启用编辑”确认后进入 `external-edit`，只通过绝对路径读写当前文件，不扫描父目录，不写最近笔记本，不启动索引或后台训练。`.txt` 仅作为应用内文件格式，不作为系统级外部启动格式。
- **验证**: E2E 覆盖 mock opened-file 后不显示 WelcomePage/LeftWing/RightWing/FileDrawer；安装版 GUI 通过 Windows 文件关联启动真实桌面 `.md`，确认进入只读页、启用编辑后保存同一文件、磁盘回读成功，且没有父目录扫描。
- **教训**: 系统文件关联入口必须先判定“单文件会话”还是“笔记本会话”。普通文件双击不应隐式升级为打开所在目录，更不能启动全库索引、标签和训练。

## BUG-079: 即时模式 Markdown 表格列宽和对齐渲染塌陷

- **现象**: Live Preview/即时模式中普通 Markdown 表格被渲染成错位文本，中文表头如“维度 评分 说明”会粘在一起，数字列如 `85` 会贴到下一列正文前，阅读体验接近不可用。
- **根因**: 旧实现按单行伪造表格块，依赖 `display: table-row`/单元格替换，无法在 CodeMirror 不跨行替换的约束下共享列宽；后续 CSS `data-table-column-count` fallback 又覆盖了解析器计算出的 `--ml-table-template`。同时内联 `text-align` 容易被清洗/覆盖，表头继承右对齐后进一步造成中文标题粘连。
- **根因类别**: 渲染管线 / 前端样式 / IME 交互边界
- **修复**: 表格解析按组计算列数、列宽模板、分隔行和对齐；每行用 CSS Grid 渲染同一组列模板，分隔行隐藏，单元格使用 `ml-table-cell--align-*` 类控制对齐；表头强制左对齐并增加列间 padding。只读外部文件模式走完整 Markdown renderer，输出真实 `<table>`。
- **验证**: `cm6-live-preview` 单元测试覆盖中文表头、对齐和无 `.ml-td`；J1c E2E 定向通过；安装版 GUI 在单文件编辑模式中验证中文表格视觉不再串列。
- **教训**: CodeMirror 内的“富文本式表格”不能按行独立决定列宽。只要视觉上是一个表格，就必须有表格组级别的列模型。

## BUG-080: 冷启动期间全局快捷键监听注册过晚导致 Ctrl+K/Ctrl+Shift+P 失效

- **现象**: 应用冷启动或外部文件打开期间，用户按 `Ctrl+K`/`Ctrl+Shift+P` 可能没有打开搜索/命令面板，表现为快捷键被吞掉或等待初始化完成后才可用。
- **根因**: `NotebookHome` 在 `onMounted()` 内先 `await initNotebook()`，再注册全局 keydown listener。真实文件系统初始化、外部文件读取或目录扫描较慢时，用户早期输入发生在 listener 存在之前；同时按键匹配大小写不完全稳健。
- **根因类别**: 状态管理 / 交互竞态
- **修复**: 全局快捷键 listener 提前到挂载开始处注册，使用 capture 阶段和小写 key 匹配；外部单文件会话在初始化早期 return，避免不必要的 notebook 初始化阻塞。
- **验证**: 搜索专项 E2E 通过；Chromium 全量 E2E 167/167 通过；安装版 GUI 外部文件只读/编辑路径未再出现快捷键初始化阻塞。
- **教训**: 用户可见的全局快捷键不能排在异步初始化之后。初始化越慢，越要先挂交互入口，再按当前状态决定是否响应。

## 检查清单追加

- [ ] 桌面文件关联入口必须区分单文件会话和笔记本会话；除非用户显式选择文件夹，否则不得扫描父目录。
- [ ] 外部单文件会话不得调用索引、标签、搜索、后台训练、最近笔记本写入或文件抽屉递归加载。
- [ ] `.txt` 只能作为应用内可打开格式，不得被系统文件关联或命令行外部启动入口接受。
- [ ] Live Preview 表格必须用表格组级列模型验证，不能只断言单行 HTML 可见。
- [ ] 冷启动前 1 秒内的全局快捷键也必须可响应或明确按当前页面状态忽略，listener 不得注册在长异步初始化之后。

## BUG-081: 外部只读切编辑后完整控件未恢复且文件抽屉可能停留在视口外

- **现象**: 外部 Markdown 进入只读页后，点击“启用编辑”只能进入临时单文件编辑面板，TopBar、格式工具栏、状态栏、导出/分享/设置等完整控件没有按 AppShell 形态恢复；随后打开文件抽屉时，抽屉语义上存在但可能因横向 transform 处于视口外，用户无法点击同目录文件。
- **根因**: 外部文件会话只有 `external-readonly/external-edit` 的临时壳，未把“完整 AppShell 外观”和“单文件数据域”拆开；文件抽屉依赖滑入 transform 动画，发布收口中新的外部编辑切换路径暴露出抽屉可能保留 offscreen transform 的不稳定状态。
- **根因类别**: 状态管理 / 前端样式 / 可访问性
- **修复**: 外部会话扩展为 `external-readonly`、`external-edit-shell`、`external-folder-indexed`；只读页改为正式 reader layout；启用编辑后恢复 AppShell、TopBar、FormatToolbar、StatusBar、导出、分享、设置和主题切换，但默认只读写当前外部文件；文件抽屉按需列出父目录支持格式，未点击文件不进入左侧彩色圆点；抽屉横向 transform 动画移除，保留遮罩淡入，避免 offscreen 卡住。
- **验证**: `typecheck` PASS；`prettier --check` PASS；`vitest` 156/156 PASS；`cargo fmt --check` PASS；`cargo check` PASS；`cargo test` 11/11 PASS；Chromium `16-user-journeys` 10/10 PASS；Firefox `16-user-journeys` 10/10 PASS；内置浏览器 GUI 抽样确认文件抽屉 `x=0`、宽 `320px`、`transform: none`，设置页“扫描根目录文本文件”开关可见。
- **教训**: 外部文件编辑可以恢复完整软件控件，但数据域仍必须保持单文件边界。动画不能成为交互可达性的前提，发布阶段对抽屉/弹层这类基础导航控件要优先保证稳定可点击。

## 检查清单追加

- [ ] 外部文件“编辑态完整控件恢复”不等于打开父目录笔记本；AppShell 外观和数据域必须分离验证。
- [ ] 单文件会话左侧彩色圆点只显示当前文件和用户本会话实际打开过的文件，不能加入扫描得到但未点击的文件。
- [ ] 抽屉、弹层、确认框的入场动画不得让可交互元素长期停留在视口外；E2E 需要验证实际点击，不只验证 DOM 可见。

## BUG-082: 卸载后 `.md` 文件图标仍残留 JotLuck 图标

- **现象**: 本机卸载 JotLuck 后，`.md` 文件的关联图标仍然显示为 JotLuck 文件图标；注册表中 `.md/.markdown/.mdx` 扩展名仍可能指向旧 `Markdown` ProgID，且 Explorer `OpenWithList/OpenWithProgids` 中可能残留 `JotLuck.exe` 或 `Markdown`。
- **根因**: Windows NSIS 安装器生成的文件关联使用通用 `Markdown` 作为 ProgID；自定义 hook 又把 `Software\Classes\Markdown\DefaultIcon` 改为 `$INSTDIR\file-icon.ico`。卸载时 Tauri 的 `APP_UNASSOCIATE` 只按安装时备份恢复扩展名默认值并删除 ProgID，未覆盖旧版本 hook 写入的通用 `Markdown` 图标/打开命令和 Explorer 用户级残留。
- **根因类别**: 跨平台兼容 / 发布资产 / 文件IO
- **修复**: 本机先执行静默卸载并清理用户级注册表残留；安装器配置把文件关联 ProgID 改为 `JotLuck.Markdown`，避免继续污染通用 `Markdown` 类；`installer-assets/hooks.nsh` 增加 `NSIS_HOOK_POSTUNINSTALL`，卸载时清理 `JotLuck.Markdown`、扩展名备份值、Explorer `OpenWithProgids/OpenWithList`，并在确认旧 `Markdown` 类由 JotLuck 安装目录拥有时兼容删除旧残留。
- **验证**: 本机卸载后 `D:\JotLuck` 不存在，卸载项为 0，`HKCU:\Software\Classes\Markdown` 与 `HKCU:\Software\Classes\Applications\JotLuck.exe` 不存在；新 v0.15.0 安装器静默安装后 `.md` 指向 `JotLuck.Markdown`。模拟旧包留下的 `Markdown` 默认关联、DefaultIcon、open command 与 `OpenWithList` 后，再用新 v0.15.0 静默安装/卸载，`JotLuck.Markdown`、旧 `Markdown`、卸载项、安装目录均不存在，`.md/.markdown/.mdx` 默认值清空。
- **教训**: Windows 文件关联必须使用应用专属 ProgID，卸载 hook 要清理安装器自定义写入的所有注册表路径；验证不能只看安装目录是否删除，还必须查扩展名默认值、ProgID、OpenWithList/OpenWithProgids 和 shell 图标缓存。

## 检查清单追加

- [ ] Windows 文件关联不得使用通用 `Markdown` ProgID；应使用应用专属 ProgID，并确保卸载时撤销扩展名默认值、ProgID、OpenWithProgids 和应用打开列表残留。
- [ ] 发布安装包验收必须包含“安装 → 查看 `.md` 图标/打开方式 → 卸载 → 重新查看图标/打开方式”的闭环，不能只验证安装成功。

## BUG-083: 正则搜索复用 global RegExp 导致连续候选文档漏匹配

- **现象**: 多篇笔记内容都匹配同一个正则时，搜索结果可能隔一篇漏一篇；表现取决于上一次 `RegExp.test()` 后的 `lastIndex`。
- **根因**: `SearchEngine.search()` 默认用 `gi` flags 构造正则，并在 `candidates.filter()` 中复用同一个 `RegExp` 实例。带 `g` 的正则是有状态对象，跨文档调用 `test()` 会继承上一轮 `lastIndex`。
- **根因类别**: 索引/搜索
- **修复**: `SearchQuery` 增加可选 `regexFlags` 类型契约；搜索实现编译正则时剥离 `g` flag，保留大小写等其他 flags；新增连续候选文档回归测试。
- **验证**: `SearchEngine.test.ts` 定向通过；`vitest run` 188/188 PASS；coverage PASS。
- **教训**: 过滤链中的正则匹配不得复用带 `g`/`y` 等状态型 flags 的 `RegExp.test()`，除非每次调用前显式重置 `lastIndex`。

## BUG-084: Tauri 文本保存固定 `.md.tmp` 临时文件名存在碰撞风险

- **现象**: 同目录同 stem 但不同扩展名的笔记（如 `same.md` 与 `same.txt`）保存时会落到同一个临时文件路径，快速保存或并发保存存在互相覆盖/rename 错目标风险。
- **根因**: `fs_ops.rs::write_file_at()` 使用 `target.with_extension("md.tmp")` 生成临时路径，扩展名被固定替换为 `md.tmp`，导致不同原始扩展名收敛到相同临时文件名。
- **根因类别**: 文件IO / 跨平台兼容
- **修复**: 新增同目录唯一临时文件 helper，文件名包含原始文件名、进程 id、时间戳和自增计数；保存时写入唯一 temp 后替换目标文件；Windows 使用 `MoveFileExW(REPLACE_EXISTING | WRITE_THROUGH)` 保持覆盖已有文件的保存语义；外部单文件写入也接入同一原子写 helper。
- **验证**: Rust `fs_ops` 新增 temp path 唯一性和覆盖已有文件测试；`cargo test --manifest-path packages/app/src-tauri/Cargo.toml --lib` 13/13 PASS；`cargo check` PASS。
- **教训**: 临时文件名必须保留目标文件身份并具备每次写入唯一性；跨平台保存还要显式验证“目标已存在”的覆盖语义，不能假设 `rename` 在所有平台一致。

## 检查清单追加

- [ ] 搜索过滤链中复用 `RegExp` 时，不得保留 `g`/`y` 等会推进 `lastIndex` 的状态型 flags。
- [ ] 文本保存的临时文件必须与目标文件同目录且每次写入唯一；同 stem 不同扩展名和目标已存在两种路径都要有测试。

## BUG-085: Tauri safe path check allowed symlink/junction escape

- **现象**: notebook root 内如果存在指向外部目录的 symlink/junction，`linked/file.md` 这类路径可能通过旧的 parent lexical check，被当作 root 内安全路径。
- **根因**: `resolve_safe_path()` 先用 `parent.starts_with(root)` 做字符串/路径前缀判断，未对目标或最近已存在祖先执行 canonicalize 后再校验 root 边界。
- **根因类别**: 文件IO / 跨平台兼容 / 安全边界
- **修复**: `is_safe_path()` 改为 canonical root + canonical target/nearest existing ancestor 双路径校验；新增 symlink/junction escape 单测。
- **教训**: 文件安全边界不得只用 lexical path 判断；写入不存在目标时也必须校验最近已存在祖先的 canonical 路径。

## BUG-086: Web MockFS 默认持久化到 localStorage 偏离“文件即数据源”

- **现象**: Web 预览默认把笔记内容写入 `localStorage`，普通浏览器会话会悄悄变成持久化笔记存储。
- **根因**: MockFS 最初为 E2E 刷新验证而默认启用 localStorage，但没有区分 E2E 持久化和普通 Web preview。
- **根因类别**: 文件IO / 状态管理 / 架构偏移
- **修复**: `MockFSService` 增加 `{ persist }` 选项，默认内存态；`NotebookHome` 仅在 `mode=e2e` 或显式 `VITE_JotLuck_MOCKFS_PERSIST=1` 时启用持久化。
- **教训**: 测试便利开关必须显式隔离，不能改变普通产品路径的数据所有权语义。

## BUG-087: Theme/CSP/Tauri capability boundary too wide for trusted-code themes

- **现象**: Tauri 开启 global API 且 CSP 为空；默认 capability 还开放未使用的 fs 插件权限；外部 theme.css 可声明全局选择器污染宿主 UI。
- **根因**: Theme API v2 接受 trusted-code 本地信任模型，但宿主边界没有同步最小化：runtime 检测依赖 `window.__TAURI__`，能力文件保留了早期 fs 插件授权，CSS 导入只校验包结构不校验作用域。
- **根因类别**: 安全边界 / 主题系统 / 跨平台兼容
- **修复**: 使用 `@tauri-apps/api/core.isTauri()` 代替 global Tauri 检测；关闭 `withGlobalTauri`、设置 CSP、移除 fs 插件 capability；安装主题包时拒绝未包含 `[data-theme-id="..."]` 的 DOM 选择器。
- **教训**: trusted-code 可以是本地高级能力，但宿主仍应最小暴露 Tauri API、插件权限和 CSS 作用域。

## BUG-088: 文件树、右栏拖拽柄和欢迎页开关缺少键盘/触控语义

- **现象**: FileDrawer treeitem 不可 Tab 聚焦，目录/文件无法通过标准树键盘操作；RightWing resize handle 只支持 mouse；WelcomePage 自绘开关是 span + click，缺少 switch 语义。
- **根因**: UI 控件早期以鼠标 happy path 为主，未把 tree/separator/switch 的原生语义、focus ring、pointer/touch 输入作为验收条件。
- **根因类别**: 可访问性 / 前端交互 / 跨平台兼容
- **修复**: FileDrawer 增加 roving tabindex 与 Arrow/Home/End/Enter/Space 行为；RightWing handle 改为 `role="separator"`、pointer 事件和键盘调宽；WelcomePage toggle 改为 button switch，触控目标提升到 44px。
- **教训**: 自绘控件必须先补齐 role/state/keyboard/focus/pointer，再谈视觉打磨；鼠标路径通过不等于发布级交互闭环。

## BUG-089: 外部 MD 启用编辑过早显示完整工作区导致首轮编辑丢失

- **现象**: 外部 Markdown 只读预览点击“启用编辑”后，完整工作区会先出现，但当前文件尚未完成 `activePath` 选中；用户或 E2E 立即输入后，内容会被后续 `onSelectNote()` 读取原文件覆盖，文件抽屉/搜索/保存链路表现为未进入普通笔记本。
- **根因**: `openExternalParentAsNotebook()` 先把 `externalSessionMode` 置为 `none` 并清空外部会话，再异步打开父目录、写入 MockFS、构建索引和选中文件。UI 状态切换早于数据状态机完成，造成 AppShell 在 `activePath=''` 的中间态可交互。E2E bridge 还可能被过渡残留编辑器污染，放大了竞态。
- **根因类别**: 状态管理 / 文件IO / E2E 基础设施
- **修复**: 外部只读页在晋升期间保持 `readonly + loading`，等待父目录打开、MockFS hydration、索引初始化和 `onSelectNote(target.relativePath)` 全部完成后，才切到普通工作区并清空外部会话；`MarkdownEditor` E2E bridge 在 focus/mousedown 时重新注册当前实例，避免隐藏旧编辑器污染测试通道；外部文件 E2E 改为验证父目录笔记本、同目录文件、MockFS 落盘和切回读闭环。
- **验证**: `pnpm typecheck` PASS；`pnpm lint` PASS；`pnpm --filter @jotluck/app lint:style` PASS；`pnpm test` 272/272 PASS；Chromium 定向 E2E `16-user-journeys.spec.ts -g "外部文件"` PASS。
- **教训**: 页面外壳切换必须滞后于关键数据状态完成；任何“先显示再补 activePath”的中间态都会让用户输入和异步读文件产生覆盖竞态。E2E bridge 必须绑定当前可见编辑器实例，不能默认全局单例永远新鲜。

## 检查清单追加

- [ ] 外部只读文件晋升为父目录笔记本时，必须先完成 notebook root、文件树、索引和 `activePath` 选中，再显示可编辑 AppShell。
- [ ] 涉及 `<Transition>`/主题 slot 的编辑器 E2E，不得默认使用全局旧 bridge；测试桥应在当前可见编辑器 focus/mousedown 时刷新。

## BUG-090: E2E 默认读取测试桥导致编辑器内容断言可能假绿

- **现象**: E2E `getEditorContent()` 优先读取全局 bridge 内容，测试可能在可见编辑器未正确渲染或旧编辑器残留时仍通过；JotLuck 改名后大小写不同的 storage key 也可能导致状态清理不完整。
- **根因**: 测试 helper 把诊断桥作为默认内容源，绕过了用户真实可见 DOM；storage 清理只匹配旧前缀。
- **根因类别**: E2E 基础设施 / 状态管理
- **修复**: `getEditorContent()` 默认读取可见 `.cm-content` DOM；新增 `getEditorContentFromBridge()` 仅供源码精确断言显式调用；E2E 状态清理同时识别 `jotluck` / `JotLuck`，base URL 环境变量统一为 `JOTLUCK_E2E_BASE_URL`。
- **验证**: Chromium 定向 E2E `07-persistence + 15-autocomplete-journey` 9/9 PASS；`pnpm test` 277/277 PASS。
- **教训**: E2E 默认 helper 必须模拟用户可见结果；测试桥只能用于明确的诊断或源码精确断言，不能作为默认通过路径。

## BUG-091: 外部文件与索引命令信任前端传入 root 造成文件边界过宽

- **现象**: Tauri `build_index` / `update_index_document` 接收前端 rootPath，外部单文件读写缺少 Rust 侧会话授权状态；收紧后又暴露出“另存为新文件”因目标不存在被拒绝的问题。
- **根因**: 文件边界由前端参数表达，Rust 没有统一以 `NotebookRoot` 和会话级 `ExternalAccessGrants` 为真源；写入路径复用读取解析，错误要求目标文件预先存在。
- **根因类别**: 文件IO / 跨平台兼容 / 安全边界
- **修复**: 索引命令只从 `NotebookRoot` 读取当前笔记本根；外部文件读写改为 Rust 签发的会话级 opaque grant，按读/写/目录枚举/watch 能力校验相对路径，支持外部文件授权后由后端提升为父目录笔记本；写入解析允许在已授权父目录下创建新 `.md/.markdown/.mdx/.txt` 文件。
- **验证**: `cargo check` PASS；`cargo test` 35/35 PASS；Tauri 外部文件读写、目录提升和 watcher 均不接受 Renderer 绝对路径作为授权凭据。
- **教训**: 桌面端文件命令的边界必须在 Rust 状态中闭合，不能由前端每次传 root 决定；读路径和写路径解析规则不同，另存为新文件必须单独测试。

## BUG-092: 文字补全与自定义模板把正文派生数据藏进 localStorage

- **现象**: 补全 ngram、训练 meta 和 accepted lexicon 使用全局 key，跨笔记本可能串数据；自定义模板把当前笔记全文写入 localStorage，脱离“文件即数据源”的产品边界。
- **根因**: 早期持久化为了功能速度直接使用全局 localStorage，没有按 notebook root 隔离，也没有把用户自定义模板纳入笔记本文件域。
- **根因类别**: 状态管理 / 文件IO / 产品边界
- **修复**: 补全学习数据改为 `jotluck:scope:<notebook-root-hash>:...`，首次加载迁移并清理旧全局 key；训练 meta 同步按 scope 读写；自定义模板迁移到当前笔记本 `/.jotluck/templates/` 文件目录，临时草稿和外部单文件禁用“保存当前为模板”。
- **验证**: `TemplateEngine.test.ts` 覆盖文件模板保存/迁移/受控删除；`MarkdownPredictor` 与 `CompletionTrainingService` 单测更新后通过；coverage 总体 94.6%。
- **教训**: “本地优先”不等于“任何本地存储都可以藏正文”。正文或正文派生的长期数据必须有明确 scope；用户模板应成为用户文件域的一部分。

## BUG-093: CSV 导出未防护公式注入且 RC gate 可被旧报告误放行

- **现象**: CSV 表格单元格或无表格整文单格以 `= + - @ tab CR` 开头时，表格软件可能按公式执行；RC gate 只检查报告标记和 PASS，未重新计算安装包 hash，旧 L4 报告可能对应旧安装包。
- **根因**: CSV 转义只处理逗号、引号和换行，未做 spreadsheet formula injection 防护；发布闸门没有把安装包路径、版本、SHA256 与当前产物绑定。
- **根因类别**: 导出 / 发布工程 / 安全边界
- **修复**: CSV 所有单元格统一对危险前缀加单引号后再做 RFC 风格转义；RC gate 要求 `L4-APP-VERSION`、`L4-INSTALLER-PATH`、`L4-INSTALLER-SHA256`，并重新计算当前安装包 hash、校验报告新鲜度；CI 增加 Rust check/test/audit 和 Windows Tauri build。
- **验证**: `Exporter.test.ts` 覆盖表格与整文单格 CSV 注入；`pnpm audit --audit-level high` PASS；release gate `--help` 和模板路径输出可用，安装包 SHA256 已计算。
- **教训**: CSV 是可执行表格输入，不是纯文本；发布闸门必须绑定真实产物身份，不能只信报告里写了 PASS。

## 检查清单追加

- [ ] E2E 默认内容 helper 只读可见 DOM；需要源码精确内容时必须显式调用 bridge-only helper。
- [ ] Tauri 索引命令不得接受前端 rootPath，必须使用 Rust 侧 `NotebookRoot`。
- [ ] 外部文件读写必须先登记父目录；写入新文件必须校验已存在父目录的 canonical 边界。
- [ ] 补全、训练和用户模板等正文派生数据必须按 notebook scope 或文件域持久化，不得使用全局 localStorage key 长期保存。
- [ ] CSV 导出所有单元格必须经过公式注入防护，包括无表格时的整文单格。
- [ ] RC gate 必须校验安装包路径、版本、SHA256 和 L4 报告时间，不得接受旧报告或手写 PASS。
- [ ] P2 release hardening must keep app identity, Tauri capabilities, native watcher lifecycle, and Markdown renderer boundary tests synchronized before packaging.

## BUG-094: P2 发布质量项缺少资源生命周期与发布身份防回归约束

- **现象**: 发布元信息散落在设置页和更新检查；Tauri capability 仍保留无作用域 `shell:allow-open`；Rust 文件 watcher 通过泄漏 watcher 保持运行；Markdown 边界修复缺少系统性回归测试。
- **根因**: P0/P1 修复聚焦阻断项，P2 发布质量项没有同步形成单一配置源、静态权限测试、watcher stop/replace 状态机和 renderer/live-preview 双链路测试。
- **根因类别**: 发布工程 / 跨平台兼容 / 渲染管线 / 状态管理
- **修复**: 新增 `app-meta` 单一配置源；移除无作用域 shell 权限并补静态测试；Rust watcher 改为可停止单例并接入前端 `unwatchAll()`；补 Setext、表格、列表、fenced code、裸 JSON-like 块回归测试；ThemeDialog 补安全区与稳定滚动槽。
- **教训**: 发布质量项不能只靠人工清单；凡是“版本身份、权限、系统资源、渲染边界”都必须落到自动化测试或集中配置源中。

## BUG-095: 未定义 spacing custom property 令 Halo 控件整条布局声明失效

- **现象**: Halo Canvas 的搜索按钮文字贴边、导航条目没有有效间距、命令坞与格式工具栏缺少内边距；同时中央 Atelier 画布失去应有的 inset 与浮层层级，主题视觉退化为浅色圆角卡片。
- **根因**: `tokens.css` 只定义了部分 4px 间距档位，但宿主与主题引用了 `--space-2/6/10/14/18/22/28/36/80`，以及孤立的 `--space-3/9`。CSS custom property 未定义时，包含它的整个 `padding`、`gap` 或 `inset` 声明在计算期失效；全局 reset 使部分属性视觉上回退为零。旧 Halo 又以裸主题控件替代宿主控制层，放大了这一系统缺口。
- **根因类别**: 设计系统 / 渲染管线 / 主题系统
- **修复**: 建立“4px 布局节奏 + 2px 控件微调”完整 spacing 合同，归一 `--space-3/9` 为 `--space-4/8`；新增 `lint:tokens` 扫描 CSS、Vue 与主题 CSS 字符串；为默认宿主控件添加稳定 `data-theme-part` 样式部件，Halo 改为保留宿主动作/图标/格式工具栏，仅以 scoped 玻璃 frame 重建环境、chrome 与安静正文画布。
- **验证**: token lint 必须通过；单测覆盖未知 token 与 wrapper 保留的 action/格式/语义；E2E 断言搜索 padding、导航 gap、命令坞与工具栏 inset、正文画布阴影和玻璃降级，并固定真实运行时视觉基线。
- **教训**: 设计 token 是可执行合同，不是命名建议。任何新 custom property 都必须同时有定义、静态检查和运行时 computed-style 断言；主题不能为了视觉重建已经由宿主正确提供的语义控件。

## 检查清单追加

- [ ] 新增或引用 `--space-*` 前，先确认其定义存在并运行 `pnpm.cmd lint:tokens`；不得使用 `--space-3`、`--space-9`。
- [ ] 主题视觉验收必须检查关键控件的 computed padding/gap/inset 与正文阴影，不能只看 DOM 存在或单张探针截图。
- [ ] 默认 slot 主题 wrapper 必须保留宿主 action、图标、ARIA、格式工具栏和编辑器状态机；若仅需材质与层级，不得重建裸控件。

## BUG-096: 离线补全模型的文档训练、实例生命周期和发布基线缺少完整性边界

- **现象**: 同一笔记每次自动保存都会重复放大整篇正文权重，编辑/删除无法撤销旧模式；切换编辑器会重复解析约 6MB L3，训练器仍绑定旧 Predictor；损坏或污染的 HTTP 200 baseline 会被静默接受；部分失焦路径还能把 ghost text 当作 Tab 接受。
- **根因**: L1、正文派生模型和用户反馈共用聚合式 L2 写入；`ingestDocument()` 忽略 path，训练 meta 只记状态不记可撤销贡献；Predictor 由 keyed 编辑器拥有而训练器由页面长期持有；模型没有 manifest/hash/schema/最小条目校验；Markdown 边界、Unicode N-gram、Provider 去重和 ghost 焦点域分别实现，缺少统一契约。
- **根因类别**: 状态管理 / 索引与模型完整性 / 渲染管线 / 发布工程
- **修复**: 按 `autocomplete-spec.md` v1.3 实施工作区级 Predictor、按文件可替换 N2、scoped Personal L2、应用级只读 L3、模型 manifest 与 fail-closed 训练管线；统一 Markdown context、Unicode N-gram、Resolver 文本去重和 Tab-only 接受规则。
- **教训**: 离线学习模型也必须具备数据身份、撤销语义、作用域、完整性校验和独立评测；扩大语料之前先证明每一条计数来自哪里、何时失效、由谁加载。

## 检查清单追加

- [ ] 自动保存训练必须以文件路径替换贡献，重复保存、修改、删除和重命名都要有可逆测试。
- [ ] keyed 编辑器不得拥有大模型生命周期；切换 20 篇笔记只能加载/解析一次 L3。
- [ ] baseline 的空/HTML/截断/hash/schema/条目数异常必须回退，训练闸门失败不得覆盖发布资产。
- [ ] ghost text 只有编辑器持焦且非 IME 的无修饰键 Tab 可以接受；任何 blur 只能清除。

## BUG-097: 连续输入被误记为拒绝且 E2E 在选笔记竞态中写入旧编辑器

- **现象**: 同一段重复文本的 LineEcho 在单测稳定命中，但黑盒逐字输入时会随机消失；部分失败用例的可见编辑器已被初始化队列切回另一篇笔记，测试桥内容与待测文本完全不同。
- **根因**: Ghost 插件在用户继续键入且没有采用当前候选时调用 `rejectCompletion()`，把隐式覆盖错误地当成 Escape 拒绝；达到两次后相同候选被 Resolver 静默。黑盒脚本同时只等待编辑器 DOM 可见，没有等待初始 `activePath` 选中队列完成，后到的 `onSelectNote()` 会重挂编辑器并覆盖刚输入的文本。
- **根因类别**: 状态管理 / 补全反馈 / E2E 基础设施
- **修复**: 删除继续输入路径的隐式拒绝，只允许明确 Tab/Escape 写学习反馈；黑盒和正式 holdout 在输入前等待文件清单、显式选择稳定路径并确认 `activePath`，再切换源码视图。新增继续输入不接受也不拒绝的 CM6 单测和全域 LineEcho 参数化回归。
- **验证**: 单元测试 32 files / 376 tests 通过；补全 E2E 24/24 通过；黑盒 11/11 通过；真实写作 25/68，触发率与可用率 36.8%，误触发率 0，mixed 0，p90 105ms。
- **教训**: “没有接受”不等于“明确拒绝”；个人学习只能消费有清晰用户意图的事件。E2E 的“编辑器可见”也不等于数据状态稳定，涉及 keyed remount 时必须同时确认 active identity。

## 检查清单追加

- [ ] 继续输入、blur、窗口失焦、弹窗切换只清除 ghost，不得写 accepted/rejected 信号。
- [ ] keyed 编辑器 E2E 输入前必须确认 `activePath` 与当前测试目标一致，不能只等待 `.cm-content` 可见。

## BUG-098: Halo 将环境、控件与正文拉平为同一张透明圆角卡

- **现象**: 多层全屏彩雾使亮色 Halo 显得沉闷；导航、命令坞、工具栏和正文画布缺失材质差异，字体、圆角和控件宽度不匹配；768px 的右翼在压缩画布，360px 的两翼和多行命令坞抢占首屏。
- **根因**: Halo 同时叠加了主题背景、app shell 和 workflow 的大面积渐变，再将外壳、工具组和正文使用同质的高透明白色、圆角和阴影。另外，主题以 `width: 100% !important` 覆盖 `RightWing` 的内联宽度，破坏了宿主窄轨与桌面 resize 的属性。
- **根因类别**: 设计系统 / 主题系统 / 响应式布局
- **修复**: 环境收束为单层低色度银雾，棱镜光只作为 glass chrome 的局部伪层；建立环境、轻浮 chrome、浅内凹工具组、主正文阴影的材质高度；将几何统一为 10/14/20/24px。`RightWing` 以 CSS custom property 作为内联 width 的响应式回退，桌面仍由宿主面板宽度控制，721–900px 则使用可访问的本地 56px/264px 检查器窄轨。
- **验证**: 单测断言 `inspector-rail-toggle` 的 ARIA 状态；E2E 断言 1280 宿主右翼 resize、768 窄轨展开、360 正文优先、命令坞高度和水平工具栏。视觉基线必须在真实运行时重新编录，最后再做 GUI 旅程验收。
- **教训**: “液态玻璃”不是给所有层增加透明和圆角；它是一套有明确光学、阴影、空间预算和字体尺寸信号的材质高度系统。

## 检查清单追加

- [ ] 官方主题的彩色折射必须局部、低频且仅存在于 chrome；不得用多层全屏彩雾代替材质层次。
- [ ] 主题不得以 `!important` 接管宿主桌面尺寸所有权；如需响应式变体，优先使用 CSS custom property 作为宿主内联样式的可控回退。
- [ ] 视觉回归必须覆盖 1280、768、360 的材质、空间和正文优先级；不得仅断言卡片可见或阴影非 `none`。

## BUG-099: Personal L2、Notebook N2 与公共 L3 合并过早导致学习和跨文档规律失真

- **现象**: Personal L2 与笔记本正文模型在进入 Provider 前已合并，无法独立归因或让学习信号改变同层胜者；单篇文档先裁剪再聚合会丢掉“分别在两篇文档各出现一次”的真实跨文档 transition；Provider 只暴露 top-1，使 Resolver 看不到可被拒绝/接受信号重排的备选项。
- **根因**: 运行时把数据来源当成一张计数表，而不是具有不同生命周期和可信度的独立候选层；文档支持数只存在于聚合结果之后，训练入口没有保存每文件可撤销贡献与 support map；N-gram API 只返回单一预测。
- **根因类别**: 状态管理 / 补全模型 / 统计口径
- **修复**: L1、Personal L2、Notebook N2、L3 分表进入 `provideMany`，各层返回确定性 top-k；Notebook 保存每文件贡献和文档支持表，先聚合再按至少两篇文档支持过滤，编辑/删除/重命名可撤销；Resolver 先按规范化文本跨 Provider 去重，再应用拒绝和学习信号；short2/short3 继续独立存储和查询。
- **验证**: 单测覆盖同文件幂等、编辑撤销、删除/重命名、跨文档单例 transition、四层候选、top-2、拒绝抑制与学习重排；最终 `vitest` 33 files / 410 tests PASS，补全 E2E 24/24 PASS。
- **教训**: 离线模型的“计数相加”不是无损操作。来源、文档支持数、可撤销身份和候选集合必须保留到 Resolver 做完最终决策以后。

## BUG-100: legacy 基线、剪枝后置信度与评测时序可伪造模型可发布性

- **现象**: 旧 v3 基线没有词级分区、训练输入身份和绑定 holdout 的发布证据；剪枝后只看剩余分支会把单例样本表示成接近满置信度；固定探针、后台训练和人工 seed 可互相污染，报告随异步时序从 36 分跳到 99 分；中文句尾弱英文功能词（如 `他笑着说 the`）还可能错误切到英文候选。
- **根因**: manifest 只验证文件存在，没有区分 runtime/release 资格；训练器未保存剪枝掉的概率质量，也未把质量报告绑定到模型、输入和 holdout 哈希；诊断 E2E 与真实后台训练共享同一 Predictor；mixed 语言路由只取最近 token，不判断该 token 是否足以成为技术语言锚点。
- **根因类别**: 发布工程 / 评测泄漏 / 补全模型 / 语言门控
- **修复**: 新增 sectioned v4 字符/英文词模型、`countScale`、隐藏 `other mass`、异步分块解析、SHA-256/字节/条目/schema 校验和 `runtimeEligible`/`qualityGatePassed`/`releaseEligible`；训练器实行全局 6MiB 预算蒸馏、原始与残余重复率、许可证/来源/语言/holdout 重叠闸门及原子候选输出；旧 v3 manifest 明确不可运行、不可发布，RC gate 给出 legacy 重建阻断；质量 E2E 关闭后台训练并只消费显式 seed；mixed 中文段落中的英文功能词不再单独触发英文路由。
- **验证**: verified baseline governance 29 tests PASS；E2E `19-autocomplete-quality` 隔离后稳定为场景分 99、负样本误触发 0、mixed 0；真实写作 25/68，触发率/可用率 36.8%、误触发 0、mixed 0、p90 101ms；正式 holdout 0/32，因此候选未发布，RC gate 正确阻断 legacy 公共资产；生产/E2E build PASS。
- **教训**: 模型质量必须绑定“哪份输入生成哪份模型、用哪份未见数据验收”。文件能解析、体积够小或固定探针分高，都不能替代独立 holdout 与可审计 manifest。

## 检查清单追加

- [ ] Personal L2、Notebook N2 与 L3 不得在 Provider 前合并；每层至少保留可归因的 top-k 候选到 Resolver。
- [ ] 文档支持阈值必须在聚合所有文件贡献后计算，不能对单文件先剪枝再假装统计跨文档规律。
- [ ] compact 模型剪枝必须保留未输出分支的概率质量；置信度需要按 manifest `countScale` 校准，单例不得变成满置信度。
- [ ] 正式模型 manifest 必须绑定模型哈希、训练输入哈希和独立 holdout 证据；legacy 或 `runtimeEligible:false` 资产不得静默加载。
- [ ] 质量 E2E 的人工 seed 不得与真实后台训练并行；诊断语料、开发探针和正式 holdout 必须隔离。
- [ ] mixed 技术写作只能由有信息量的局部技术词切换语言；`the/a/to/of` 等功能词不能单独改变中文段落的模型路由。

## BUG-101: Halo 玻璃层共用高乳白透明度导致磨砂工程塑料感

- **现象**: Halo Canvas 已有模糊、圆角和浅色高光，但背景仍显灰闷；左右翼、顶栏、命令坞和按钮像同一种乳白塑料，缺少液态玻璃的透光、厚度、接触阴影与边缘色散。检查器空状态的斜体和等权分隔线进一步放大了粗糙感。
- **根因**: 所有 chrome 共用 `--halo-glass-live: ... / 0.76`、近白 `--halo-glass-edge` 与同一 `--halo-glass-shadow`；环境场色差过小，`backdrop-filter` 没有可感知的底层光色可采样。普通图标按钮又使用透明边框和单层内高光，工具栏只呈现平面填充，材质高度没有通过上下缘明暗、接触阴影和局部折射建立。
- **根因类别**: 设计系统 / 主题系统 / 视觉合成
- **修复**: 将环境改为清透日光银蓝基底并加入克制暖光过渡；为左右翼、顶栏/状态栏、命令坞分别建立不同透明度与阴影高度；以双向内缘、接触阴影、局部蓝紫/珊瑚色散和半透明控制层表现玻璃厚度。检查器文本改用正常字形与更轻分隔层级，保留宿主控件、Theme API v2 和降级路径。
- **教训**: `backdrop-filter` 不是玻璃质感本身。只有背景存在可采样光色、前景透明度足够、上下缘明暗不对称且各高度阴影不同，模糊才会被读成玻璃；所有层共用乳白填充只会得到塑料。

## 检查清单追加

- [ ] 玻璃主题必须分别验证环境色差、前景 alpha、双向内缘、接触阴影和局部折射；不得以单一 `blur()` 或统一乳白色替代材质高度。
- [ ] 顶栏、侧翼、工具坞、内凹工具组和正文画布不得共用完全相同的背景与阴影组合。

## BUG-102: V2 异步检索缺少请求身份、资源上限与双端一致性

- **现象**: 工作区短语检索接入异步 Worker/Tauri 后，旧请求可能在切换文档或工作区后迟到；连续输入只取消主线程 Promise，Worker 仍会排队执行旧查询；Web/Rust 索引没有共同硬预算，损坏响应和 Unicode 兼容形式还可能产生平台差异；空段落会错误继承上一段的短语候选。
- **根因**: 原同步 Predictor facade 没有不可变请求快照、epoch/deadline/Abort 契约和候选批次边界；Worker 协议只有 execute/response，没有 latest-only cancel；工作区索引的正文贡献、entry/surface 分配和 Unicode 规范化分别在 TS/Rust 独立实现，缺少统一预算与 fail-closed 校验。
- **根因类别**: 补全模型 / 状态管理 / 跨平台兼容 / 资源生命周期
- **修复**: 新增最多 8 条候选的 `CandidateBatch`、`CompletionEngineRouter`、反馈 token 与异步 ghost 请求；所有迟到结果校验 scope/epoch/document/cursor/focus，Worker 使用有序队列和 latest-only cancel；Web/Rust 统一 2,000 文档、512KiB 单文档、16MiB 总输入、20,000 单文档 entries、300,000 总 entries 的净值预算，并统一 NFKC、跨文档支持、损坏响应校验和超限保留旧贡献；普通检索禁止跨空段落。
- **验证**: App 单测 41 files / 504 tests 通过；Rust 30/30 通过；真实浏览器 Worker 命中 `hybrid-retrieval-zh/en`，种子场景分 100、误触发 0、mixed 0、p90 ≤105ms；真实写作触发率/可用率 36.8%、误触发 0、p90 110ms。正式独立 holdout 仍证明公共 legacy L3 不具备发布资格，manifest 正确 fail closed。
- **教训**: 异步补全后端首先是生命周期和资源边界问题，其次才是模型问题。任何可插拔排名或检索引擎都必须只能消费有界候选快照，并在取消、超时、切换、损坏和预算超限时无条件退回确定性免费路径。

## 检查清单追加

- [ ] 异步补全请求必须同时绑定 scope、engine/retrieval epoch、document version、cursor、deadline 和 AbortSignal；迟到结果不得更新 ghost 或学习归因。
- [ ] Worker 查询必须 latest-only，取消要清理 pending/listener；dispose 不得等待可能永久悬挂的 mutation chain。
- [ ] Web/Rust 工作区索引必须共享 Unicode 规范化、文档支持阈值和硬资源预算；新 surface 也必须计入 entry 预算。
- [ ] 空段落、heading、table、code、frontmatter 不得触发普通工作区短语检索；结构化补全继续独立旁路。

## BUG-103: 三态视图循环混用当前状态与目标动作并落入播放图标

- **现象**: 在只读渲染模式中，工具区显示“编辑”按钮却配有播放三角图标；即时与分栏模式又显示当前模式名称。格式工具栏在只读渲染中仍然可见，用户无法判断按钮表示当前状态还是点击后的动作。
- **根因**: `NotebookHome.vue` 的 `view-toggle` 在 `live/split` 使用“当前状态”文案，在 `read` 使用“下一动作”文案，并把“是否等于主题默认模式”写入二态 `aria-pressed`；`ShellActionButton.vue` 没有 `view-toggle` 专属图标，使它落入播放三角 fallback；`EditorControlStrip` 不感知只读渲染状态，始终渲染格式工具栏。
- **根因类别**: 交互语义 / 无障碍 / 主题系统
- **修复**: 三态按钮统一表达下一动作（分栏、只读、返回编辑），使用中性的视图布局图标并移除错误的 pressed 状态；宿主将当前 `viewMode` 传给默认控制条，在 `read` 模式保留视图动作但隐藏格式工具栏。Halo 继续只包裹宿主控件，不新增 Theme API 或 action。
- **教训**: 循环动作不能同时充当状态指示器；图标、可见文案、ARIA 与点击结果必须表达同一件事。只读模式不得展示看似可用的编辑控件。

## 检查清单追加

- [ ] 多状态循环按钮必须统一表达目标动作，不能使用二态 `aria-pressed` 表示“是否等于默认值”。
- [ ] 只读渲染必须隐藏或禁用编辑专属控件，并以 E2E 验证实际布局状态而非依赖按钮文本推断模式。

## BUG-104: 补全质量诊断被误作发布证据且 Hybrid 故障会阻塞或永久熔断

- **现象**: seeded/固定探针通过时容易被表述为模型质量通过，但独立写作仍无有效命中；Hybrid mutation 积压会拖住查询，Worker/CSP 不可用时可能在主线程建索引，单次瞬态后端错误还会永久关闭工作区检索。
- **根因**: 评测报告没有区分治理、运行时安全与模型质量，也未把 V1/V2、holdout 和评测器身份绑定到同一证据链；检索服务把 query 串在 mutation chain 后，缺少 revision 快照、有限重建和当前工作区文件回放契约。
- **根因类别**: 评测泄漏 / 发布工程 / 状态管理 / 补全模型
- **修复**: 新增冻结 V1 评测身份、workspace-conditioned 200-checkpoint holdout、Oracle@8/完整 mixed/双 p90/归因报告与 RC 证据哈希；查询改读最近原子提交快照，Worker 不可用时禁用 Hybrid，首次故障从工作区文件幂等重建、重建期间 mutation 延后提交，第二次才熔断。24MiB 项目自有合成池只产出隔离候选，未获独立证据不得发布。
- **教训**: “测试通过”必须说明通过的是治理、运行时还是模型质量；异步索引的读取可用性必须建立在原子快照上，恢复必须从事实源回放而不是延续损坏内存状态。

## 检查清单追加

- [ ] seeded 场景、固定探针和 holdout 安全检查不得被写成模型质量通过；RC 必须核对模型、输入、holdout、评测器、质量证据和学习曲线哈希。
- [ ] V1/V2 对照必须使用同一冻结 holdout，并分别报告 Top-1、Oracle@8、usable、false trigger、完整 mixed、全请求/可见 p90 和来源归因。
- [ ] Hybrid query 不得等待全部 mutation；Worker 不可用时禁止主线程同步索引，首次重建期间到达的 mutation 不得丢失。
- [ ] 合成训练正文与候选必须保持可再生且 Git 忽略；任何档位未通过独立质量闸门都不得覆盖正式 public 资产或启动 V2.1。

## BUG-105: 合成语料输出依赖调用目录且冻结 V1 提交后身份漂移

- **现象**: 从 package 级命令调用 24MiB synthetic 生成器时可能在 `packages/app/scripts/` 下产生第二份训练池；提交钩子格式化冻结 V1 runner 后，快照 manifest 仍保存旧 runner/tree SHA，导致离线评测身份校验失败。
- **根因**: 生成器以 `process.cwd()` 作为默认仓库根目录，而不是以生成器模块位置定位；冻结快照的 manifest 在 lint-staged 改写 runner 后没有按最终提交字节重新计算聚合身份。
- **根因类别**: 发布工程 / 评测证据 / 路径边界
- **修复**: synthetic 生成器默认从 `import.meta.url` 推导仓库根目录，同时保留测试用 `workspaceRoot` 注入；按格式化后的 runner 最终字节更新 `runnerSha256`、快照 `treeSha256` 和运行时固定身份常量。RC gate 继续以 code 10 拒绝未获资格的公共 legacy 模型。
- **验证**: 单测覆盖生成器默认根目录、冻结 V1 完整性与 fb46b1e golden、生产包隔离，并明确断言当前 `--autocomplete-only` 因 web-local v4 未获资格返回 code 10。
- **教训**: 可复现训练入口不能继承调用者 CWD；任何由内容哈希标识的冻结证据都必须以提交钩子处理后的最终字节为准。

## 检查清单追加

- [ ] 训练/评测脚本的默认工作区必须由模块位置或显式参数确定，不得依赖调用者 CWD。
- [ ] 冻结快照进入提交钩子后必须重新验证 runner、逐文件和聚合 tree SHA；格式化通过不能替代身份校验。

## BUG-106: 公共 v4 丢失英文词边界且同质扩池掩盖结构性不可达

- **现象**: 24MiB 合成池扩充后 public L3 的 context hit 固定在 17.5%、usable 始终为 0；正式 validation 的英文正例没有任何可见触发，总触发率随扩池反而降到 5.5%。旧诊断数据曾被误读成公共模型质量，造成“已经接近发布”的错误预期。
- **根因**: 英文字符路径只保留词内字母并用字母正则截断 continuation，光标处于完整单词末尾时丢失前导空格和标点；词级表只在正文已以空格结尾时查询。75 个英文正例全部处在这一断路上。与此同时，六模板合成池重复同一局部 N-gram，扩池只增加计数而不增加模式覆盖；在英文命中为 0 时，即使 75 个中文正例全部触发，200 个 checkpoint 的理论上限仍只有 37.5%。
- **根因类别**: 补全模型 / 训练分布 / 评测口径 / 发布工程
- **修复**: 停止调参公共 N-gram v4，保留个人与 Notebook N-gram，公共 L3 改为固定短语库上的微型 Transformer 与显式 abstain；英文短语保留前导边界和完整单词，中文短语设置有效字符下限。训练改用 30MiB 分组干净池、固定六次矩阵和 Oracle@32 前置停损；重新冻结多参考 cold/workspace validation/final，并以绝对可用率、分语言指标和真实运行时证据决定发布。
- **教训**: 训练量不能修复表示空间里不存在的边界。扩池前必须先证明目标 continuation 可被模型格式表达，并把理论覆盖上限、Oracle 与绝对分母写入闸门。

## 检查清单追加

- [ ] 公共英文模型必须把词间空格、标点和完整词边界作为一等特征；不得只在“用户已输入空格”后查询词级 continuation。
- [ ] 扩大语料前必须计算各语言/类别在当前表示下的理论触发上限；结构性不可达时先改模型，不得继续堆同质数据。
- [ ] Oracle 候选能力未达到前不得训练或调节发布阈值；validation 可选型，final 只能在候选完全冻结后消费一次。

## BUG-107: V2R 短语库与 WASM 构建在训练前仍存在不可达和仓库作用域陷阱

- **现象**: 初版 V2R 会优先收录较长 continuation，并只从生成器显式 slot 采样中文边界；即使 Transformer 训练成功，冻结 checkpoint 的可接受短前缀也可能根本不在 8,192–16,384 短语库中。精简 ONNX Runtime 在链接末尾又会因仓库根 `type: module` 把其 CommonJS 后处理脚本当作 ESM；直接依赖 stock runtime 还会把约 13MiB WASM 打进应用。
- **根因**: 训练管线把“分类器能否学会”放在“输出空间是否包含答案”之前，没有按 checkpoint 聚合嵌套完整词/短语变体并计算短语库 Oracle 上界；构建脚本也没有隔离第三方工具链的 Node package scope，Vite Worker 未固定外置 WASM 条件。
- **根因类别**: 补全模型 / 发布工程 / 构建边界 / 评测口径
- **修复**: 短语提取改为英文完整词边界的嵌套变体与中文 3–12 字短语，中文训练文档增加确定性内部光标采样；按同一光标聚合变体并把缺失变体只记录为 bank coverage，禁止生成 `bank-miss` abstain。正式矩阵在 CPU 训练前对 cold/workspace validation 分别执行总体 ≥70%、中英文各 ≥65% 的表示能力预检。ORT 固定到 v1.27.0 commit，在忽略的工具链根写入带哈希的 CommonJS scope，关闭未使用的 ML/contrib/generation 算子，并由 Vite 外置加载已验证的 reduced WASM；实测构建产物为 3,695,459 字节。
- **教训**: 神经分类器不能选中标签空间里不存在的答案；可复现第三方构建也必须隔离宿主仓库的模块语义，并把兼容措施纳入证据哈希。

## 检查清单追加

- [ ] 固定短语库训练前必须在独立 validation 上计算表示能力 Oracle；失败时停止 CPU 训练，不能用 loss 或内部集分数替代。
- [ ] 同一光标的所有合法短语变体必须共同参与 bank coverage；不得因一个长变体缺失而把已有短变体误标为 abstain。
- [ ] 第三方构建脚本的 ESM/CommonJS 作用域必须显式隔离并绑定哈希；运行时不得回退到未验证的 stock WASM。

## BUG-108: V2R 单标签评测误判合法前缀且训练输入证据无法闭环

- **现象**: 同一光标存在多个合法完整词/短语前缀时，训练数据只保存一个主标签；模型输出另一个合法前缀仍被 Top-1/Oracle@32 记错。与此同时，v5 verifier 把 training-data report 的 `reportSha256` 与 corpus `inputTreeSha256` 直接比较，但两者定义不同且没有绑定 generator report，导致真实 v5 证据永远无法满足校验。
- **根因**: 短语库表示能力按多参考判定，但训练样本、损失和内部评测仍沿用单类分类语义，训练/评测合同不一致；manifest 只保存输入树值，没有把能从 selection 文档 ID+内容哈希推导该值的 generator report 纳入 evidence closure。
- **根因类别**: 补全模型 / 评测口径 / 发布工程 / 证据完整性
- **修复**: training-data 升级为 v2，每个光标保存确定性主标签和最多 32 个合法标签，训练损失最大化合法集合总概率，Top-1/Oracle@32 按任一合法前缀判定；候选资格新增 development 阈值校准和 internal selection 质量硬门槛。v5 manifest 新增 generator/bundle 绑定，verifier 从 selection 重算 input tree，并交叉校验 generator、training-data、training/quantization report 与模型资产。
- **教训**: 多参考只存在于评测器而不存在于训练标签，会同时浪费学习信号并制造假阴性；内容寻址证据必须能从更上游事实独立重算，不能让两个不同语义的 SHA 字段碰巧相等。

## 检查清单追加

- [ ] 固定短语分类器的训练损失、Top-1 和 Oracle 必须使用同一组合法完整前缀；主标签只能用于确定性权重，不能成为唯一正确答案。
- [ ] manifest 的训练输入树必须从 selection 的逐文档 ID+内容哈希重算，并绑定 generator 与 training-data report；禁止比较名称相似但语义不同的 SHA 字段。

## BUG-109: V2R 候选发布资格与生产链评测形成循环依赖

- **现象**: Worker 正确要求 `runtimeEligible/qualityGatePassed/releaseEligible` 全部为真才加载 public manifest，但 validation 又要求候选通过生产 Router/Worker 实测后才能取得 `qualityGatePassed/releaseEligible`。现有脚本没有隔离候选入口，只能选择提前伪造资格或根本无法评测。
- **根因**: 生产 manifest 与候选评测 manifest 共用同一加载资格合同，没有把“内部训练闸门已过、允许盲测”与“正式 final 已过、允许发布”建模为互斥状态；Playwright 默认还会重新执行普通 E2E build，覆盖任何临时评测产物。
- **根因类别**: 发布工程 / 评测证据 / 补全模型 / 构建边界
- **修复**: 新增显式 `autocomplete-v2r-evaluation` Vite mode 和不可覆盖的候选构建器。候选必须先通过内部质量、训练输入、量化、精简运行时与语料绑定，manifest 固定为 `evaluationOnly:true`、`runtimeEligible:true`、`qualityGatePassed:false`、`releaseEligible:false`；只写 `dist/autocomplete-v2r-evaluation`，普通 production/E2E mode 忽略候选 URL，Worker 默认拒绝该 manifest。评测时只预览已经冻结的 evaluation dist，避免 Playwright 重建覆盖；bundle 体积证据首次生成后不可被不同内容改写。
- **教训**: “允许评测”和“允许发布”是两种不同能力。候选通道必须在构建期显式、运行时可审计、生产包不可达，不能靠临时篡改同一组发布布尔值打破循环依赖。

## 检查清单追加

- [ ] 未发布模型只能由显式 evaluation-only 构建加载；普通 production/E2E mode 必须忽略候选 manifest 覆盖。
- [ ] evaluation manifest 必须保持质量/发布资格为 false，且候选构建不得写入 `packages/app/public`。
- [ ] 候选评测前必须交叉验证内部质量、训练输入、量化和 reduced runtime；体积报告不得被不同内容覆盖。

## BUG-110: V2R 项目生成器的局部模板占比合格但文档骨架仍高度同质

- **现象**: 初版 30MiB 项目自有短笔记池按模板 ID 统计时占比很低，但大量文档仍复用相同段落顺序、固定标签和局部三元组；短语库中的少数结构短语由过高比例文档共同贡献，继续扩池只会放大伪模式。
- **根因**: 治理只计算逐模板 ID 与全局 5-gram 频率，没有对“一个短语覆盖多少篇不同文档”和逐文档 trigram 集中度设闸；六类骨架通过参数替换制造了表面差异，却没有增加真实写作结构的熵。
- **根因类别**: 补全模型 / 训练分布 / 语料治理
- **修复**: 项目生成器升级到 v3.1，以 43 组基础/组合 frame、16 种结构、扩展语义槽和 6 种 Markdown 布局生成五类中英文短笔记；新增逐文档 trigram ≤8%、结构性短语文档覆盖率 ≤10% 的硬闸，普通单词覆盖率只作透明诊断。30MiB 选择报告固定 generator version、seed、selection/input tree SHA 与各集中度指标。
- **教训**: 模板 ID 多不等于语料多样。训练治理必须从模型真正看到的局部模式和跨文档支持分布衡量集中度，不能只检查生成器参数组合数。

## BUG-111: 短语库缺失被误标为 abstain，拒答训练与 false-trigger 指标同时失真

- **现象**: 8,192 短语诊断集中约 8 万个 abstain 样本来自 `bank-miss`，真正的文档末尾静默只有约 600 个；降低 abstain 类权重能提高触发率，却把误触发推高，恢复权重后触发率又骤降，指标无法代表用户是否需要补全。
- **根因**: 管线把“真实 continuation 不在固定输出空间”解释为“用户希望静默”。这是表示覆盖失败，不是拒答监督；同时对 abstain 使用 0.25 的特殊类权重，进一步人为倾向触发。旧 training-data v2 缓存没有契约字段阻止这类样本被复用。
- **根因类别**: 补全模型 / 标签语义 / 评测口径 / 缓存迁移
- **修复**: training-data 升级为 silence-safe v3 并使旧缓存强制失效；每篇文档只在文档末尾生成一个 abstain，`bankMiss` 必须为 0、`documentEnd` 必须等于 abstain 总数，表示缺口只保留在 `bankCoverage`。abstain 类权重固定为 1，训练器、evaluation bundle、发布器和独立 verifier 交叉校验该语义；192-byte 输入同步固定为 48 个有序 4-byte patch 并纳入训练/运行时证据。
- **教训**: 输出空间没有答案与用户不需要答案是两种相反事实。任何拒答模型都必须从真实静默位置学习，表示失败只能驱动扩充候选空间，不能反向训练成拒答。

## 检查清单追加

- [ ] 生成语料治理必须检查逐文档 n-gram 与结构性短语跨文档集中度；参数/模板 ID 数量不能替代模型视角的多样性。
- [ ] bank miss 只能计入候选表示覆盖率，不得生成 abstain 标签或计入 false-trigger 静默分母。
- [ ] abstain 样本必须能追溯到真实静默位置；训练报告须绑定原因计数与固定类权重，旧语义缓存必须通过 schema 升级失效。

## BUG-112: Evaluation fixture 用对象型来源表掩盖真实数组不可解析

- **现象**: 30MiB 物化器正确把 `source-registry.json` 写成来源对象数组，但 evaluation bundle 构建器通过 `requireRecord` 只接受普通对象；单测 fixture 又写成 `{ schema, sources }` 对象，导致测试全部通过而任何真实候选都会在读取来源证据时失败。
- **根因**: selection、独立 source registry 和 generator report 只有 SHA 字段层面的松散绑定，没有共享可执行的 JSON 形态与 generator version/seed 契约；fixture 没有复用真实物化器输出形状。
- **根因类别**: 发布工程 / 证据完整性 / 测试替身
- **修复**: evaluation builder 新增严格数组解析，fixture 改用真实数组形态；构建器、publisher 与独立 verifier 同时校验 selection sources canonical hash、project-owned MIT/外部 CC0、冻结 generator v3.1 version/seed，并从 selection 文档 ID+内容 SHA 重算 input tree。
- **教训**: 内容哈希只能证明“某些字节没变”，不能证明两份证据表达同一事实。发布 fixture 必须保持生产文件的顶层形态，关键身份还要跨文件语义校验。

## 检查清单追加

- [ ] 证据 fixture 必须与生产物化器输出保持相同的数组/对象顶层形态，不得为测试方便另造 schema。
- [ ] selection sources、source registry、generator report 的 version/seed/license 与 input tree 必须跨文件重算校验；只比较各自 64 位 SHA 不足以放行。

## BUG-113: RC Gate 只验证旧 v4，真实 v5 发布后仍会永久 code 10

- **现象**: V2R publisher 能原子安装 web-local/release 两份 v5 manifest 和神经资产，但 `release-rc-gate.mjs` 的模型列表仍硬编码两份 `baseline-ngram.*` v4；即使全部 v5 final 证据通过，RC 仍会先因旧 v4 未获资格而拒绝。
- **根因**: 新的 `verify-autocomplete-v2r-evidence.mjs` 已实现完整验证，却没有接入最终 RC 路由；旧 gate 不区分“尚未安装 v5”和“v5 已开始原子迁移”两种状态。
- **根因类别**: 发布工程 / 证据路由 / 迁移边界
- **修复**: RC gate 检测到任一 v5 public manifest 后，强制调用 V2R 独立 verifier 并要求两个 profile 同时存在、等价且证据闭环；没有任何 v5 时才走旧 v4 fail-closed 分支。当前无 v5 资产的仓库继续稳定返回 code 10，未来部分安装也不能回退 v4 绕过。
- **教训**: 实现新发布器不等于打通发布链；最终 gate 必须有明确的版本路由和部分迁移失败语义，否则新证据永远不可达或可被旧路径绕过。

## 检查清单追加

- [ ] 新模型格式的 publisher、独立 verifier 和最终 RC gate 必须在同一变更中接通；至少测试“无新格式、部分安装、双 profile 完整安装”三种状态。
- [ ] 检测到任一新格式 manifest 后不得回退旧模型证据；部分原子迁移必须 fail closed。

## BUG-114: V2R 候选能构建但真实评测桥、WebView smoke 与 CI 路由未闭环

- **现象**: `autocomplete-v2r-evaluation` 能生成隔离候选包，但该 mode 不创建生产 Router 诊断桥，Playwright 无法取得 L3 候选与真实延迟；Tauri smoke 又固定读取 v4 manifest，即使 v5 候选通过 final 也无法生成 verifier 要求的 Worker 推理回执。独立 v5 verifier 没有 CLI，普通 CI 也不会检查部分安装。
- **根因**: 候选构建、浏览器质量评测、真实 WebView2 和最终发布分别实现，却没有共享一个编译期 evaluation 身份与全资产哈希；旧 smoke/CI 入口仍假设公共模型只有 v4。
- **根因类别**: 发布工程 / 评测证据 / 构建边界 / 补全模型
- **修复**: evaluation mode 专属启用诊断桥，正常 production mode 继续编译期关闭；Tauri smoke 增加互斥 V2R 模式，逐项验证 ONNX、短语库、metadata、reduced WASM，并要求真实 Worker 推理、离线重载和 WebView2。新增不可覆盖的单 holdout runner、evaluator tree 生成器、post-final Tauri 编排、v5 verifier CI/RC CLI 与独立 Windows RC workflow；候选始终保持质量/发布标志为 false，只有 smoke 后的原子 publisher 可安装两份正式 profile。
- **教训**: “存在构建器、评测器和发布器”不等于发布链闭环。未发布候选必须在同一个不可伪造的 evaluation 身份下贯穿 Web、Tauri 和证据哈希，同时保证正常生产包不可达。

## BUG-115: Resolver 的字符上限会把神经候选第一个英文长词切成残词

- **现象**: 公共短语库正确保存 ` configuration` 等带前导空格的完整单词，但默认 12 字符 ghost 上限会输出 ` configurati`；当截断片段中只有开头空格时，旧逻辑找不到 `boundary > 0`，残词直接进入信息量与最终可用率评测。
- **根因**: Resolver 只在已截断文本中寻找最后一个普通空格，没有先判断切点是否位于英文单词内部，也没有处理“前导空格 + 第一个长词”的零个完整词情况。
- **根因类别**: 补全模型 / 英文词边界 / Resolver
- **修复**: 按 Unicode code point 截断后检查切点两侧；只有切在英文词内部时才回退到前一个完整词边界，若第一个完整词本身超过上限则拒绝该候选。新增神经 L3 回归测试，验证前导空格保留且任何可见候选不以残词结尾。
- **教训**: 模型输出空间保证完整词并不代表 UI 截断后仍完整；长度限制、去重、门控和 ghost 渲染的每一层都必须维持词边界不变量。

## BUG-116: Workspace holdout 只引用 support ID，不能证明模式真的有两篇独立支持

- **现象**: workspace v3 checkpoint 只要填写两个存在的 `supportDocumentIds` 就能通过，即使两篇文档与 `patternId` 无关或彼此近重复；E2E 随后把这些无关文档注入 Notebook/Hybrid，workspace-conditioned 结论失去含义。
- **根因**: support 文档 schema 没有声明其支持的抽象模式，validator 仅检查 ID、语言和与目标的精确不等，未检查 support—support、support—target 整篇近重复。
- **根因类别**: 评测泄漏 / holdout 治理 / 补全模型
- **修复**: workspace support 文档必须列出唯一且合法的 `patternIds`；每个 complete checkpoint 引用的至少两篇文档都必须声明同一模式。冻结校验新增 support—support 与 support—target 整篇近重复为 0，并把两项计数写入 audit。
- **教训**: “引用了两篇文档”只是关系存在，不是学习信号存在。workspace-conditioned 证据必须把模式归属和文档独立性都变成可执行约束。

## 检查清单追加

- [ ] evaluation-only 模型必须在 production Router、真实 Tauri WebView 和最终 verifier 中保持同一候选/资产身份；正常 production build 不得暴露诊断桥或候选 URL。
- [ ] 英文候选经过任何字符上限后都必须重新验证完整词边界；第一个词超限时拒绝，不得输出残词。
- [ ] workspace holdout 的每个模式必须由至少两篇显式声明该模式且整篇不近重复的支持文档提供；只检查两个 support ID 不足以放行。

## BUG-117: 训练前语料治理提前读取 final，盲测只剩名义隔离

- **现象**: 正式 RC 工作流虽然在候选冻结后才下载 final artifact，但 30MiB 物化器默认要求并读取 cold/workspace final，用于训练—holdout 重叠治理；只要正式训练能启动，训练进程同权限范围内就已经能看到 final 明文。
- **根因**: “训练前重叠治理”和“独立 final 盲测”被合并为同一个四 holdout 输入，没有把 validation 重叠审计与候选冻结后的 final 重叠审计拆成两个时序不同的证据。
- **根因类别**: 评测泄漏 / 语料治理 / 发布证据 / 补全模型
- **修复**: 物化器只接受两套 validation，并写入 `validation-only-before-candidate-freeze-v1` 范围；RC 先按独立保管方提供的 final SHA 创建不可复用 ref，再下载明文。聚合阶段从冻结 selection 重读并逐文件校验训练正文，生成 `final-overlap-audit.json`，其输入树、两套 final 树、精确/近似重叠和报告 SHA 由质量报告、消费回执、publisher、v5 manifest 与独立 verifier 交叉绑定。
- **教训**: 盲测是数据可见性的时序约束，不是文件名或声明。训练阶段不得为“提前证明无重叠”而打开 final；final 重叠审计必须发生在候选不可变之后，失败也必须消耗数据集。

## 检查清单追加

- [ ] 训练前语料治理只能读取 validation；final 必须在候选身份不可变且消费 ref 创建后才解封。
- [ ] train–final 重叠审计必须绑定冻结 selection、逐文件内容 SHA、两套 final 数据树和 evaluator 源码树；审计失败不得回到同一 final 上继续训练。

## BUG-118: 过滤 bank miss 后把条件 Oracle 误称为候选能力

- **现象**: 8,192 档完整训练在 internal selection 上报告 Oracle@32 63.28%，但同一短语库在真实写作诊断集上只有 10.5% 绝对表示率；扩大到 16,384 也只有 13%。两个数字看似矛盾，容易诱导继续长训练。
- **根因**: `training-data.ts::writeSplitSamples` 只把 `selectTrainingTarget` 命中的正例写进 JSONL，bank miss 仅进入旁路 `bankCoverage`；Python 的 `metrics()` 又只以 JSONL 行数为分母，并把结果命名为 `candidateCapabilityPassed`。因此 63.28% 实际是“短语已存在时的条件排序 Oracle”，不是开放写作候选上限。
- **根因类别**: 评测口径 / 补全模型 / 候选表示 / 发布证据
- **修复**: 新增 release-ineligible 的真实写作短语表示诊断，完整运行 8,192/12,288/16,384 三档并记录逐语言上限；Python 明示 `conditionalOnRepresentablePositiveSamples` 与 `conditionalRankingOraclePassed`，固定短语架构停止期间强制 `candidateCapabilityPassed:false`。新增 architecture-stop 记录，由长训练入口、publisher 和 v5 verifier 共同拒绝同一路线继续执行或发布。
- **教训**: 只在“模型有答案”的样本上计算 Oracle 会把表示失败从分母里消失。候选生成架构必须先在未过滤机会集上证明 coverage，再讨论排序精度；条件指标不得命名为绝对候选能力。

## 检查清单追加

- [ ] Oracle/usable 的分母必须包含候选空间无法表示的正例；若训练损失无法编码 bank miss，至少要单独报告并绑定未过滤 representation preflight。
- [ ] 架构停止记录必须同时阻断训练、发布和 RC verifier；只在计划文档写“停止”不足以防止旧命令误跑。

## BUG-119: 已停止模型仍默认进入生产依赖图且同一 v4 字节保留双 profile

- **现象**: 普通生产构建虽然没有可运行的公共 L3，却仍默认构造 `public-phrase-transformer-v1` Worker，并把 ONNX Runtime 代码打入应用；与此同时，`baseline-ngram.web-local.compact.*` 与 `baseline-ngram.v1.compact.*` 的模型字节和 SHA 完全相同，训练、publisher 与 RC 仍把它们当成两个公共模型来源。
- **根因**: “模型资格 fail-closed”只约束了 manifest 加载，没有同步撤销默认 factory、运行时依赖和构建配置；发布协议又把部署 profile 误建模为两份物理模型，而不是一个 canonical 资产。架构停止、生产依赖图和模型注册表没有形成同一个状态转换。
- **根因类别**: 补全模型 / 生命周期 / 构建边界 / 发布工程
- **修复**: `CompletionPublicEngine` 收口为唯一、模型无关且默认未绑定的插槽；移除 V2R Worker、ONNX adapter、默认 factory、专用测试与 `onnxruntime-web` 依赖。公共目录只保留 `baseline-ngram.web-local.compact.*`，训练配置、publisher、证据 verifier 和 RC 只认该 canonical profile；冻结 V1 和 V2R stop 证据继续隔离在 `scripts/`。
- **教训**: fail-closed 不等于零成本，也不等于唯一真相源。停止一个模型时必须同时清理默认构造、依赖、构建产物、重复资产和发布脚本；模型版本只能占用一个公共插槽和一个 canonical 路径。

## 检查清单追加

- [ ] 每次停止或替换公共模型时，必须同时检查 factory、Worker/WASM/原生依赖、构建配置、public 资产、训练输出、publisher、verifier 与 RC 路由。
- [ ] 公共 L3 只能有一个 canonical 模型路径；环境/profile 差异不得通过复制相同模型字节实现，评测快照不得进入生产依赖图。

## BUG-120: 公共引擎把整篇正文和可写候选权限交给异步后端

- **现象**: 通用 Public L3 插槽虽已收口为单实例，但请求仍携带光标前整篇文档，异步引擎可直接返回包含 `from/providerId/source/sourceLayer/priority` 的完整候选；统计式 V2S 接入后会产生不必要的跨线程正文复制，并让损坏 Worker 响应越过宿主插入与排名边界。
- **根因**: `CompletionPublicEngine` 从已停止 V2R 继承了“模型即可信 Provider”的宽接口，没有把模型所需的短上下文、原始打分输出和宿主拥有的编辑权限分开；Router 只负责 deadline/epoch，没有强制候选数量、语言、mixed、长度和字段所有权。
- **根因类别**: 补全模型 / 隐私边界 / 类型边界 / 生命周期
- **修复**: Public V2S 协议固定宿主按完整 code point 只传最后 256 个 UTF-8 字节；Worker 仅返回原始文本、语言与分数，Router 校验后统一盖章插入位置、来源、层级、provider 和优先级。Worker/CSP/资产失败时关闭公共 L3，禁止主线程同步推理。
- **教训**: 模型候选是非可信数据，不是编辑器命令；最小上下文和最小返回权限应由宿主协议强制，而不能依赖具体 Worker 自律。

## 检查清单追加

- [ ] 公共模型请求不得复制整篇正文；截断必须按 UTF-8 字节预算且不能切断 Unicode code point。
- [ ] 异步生成器不得控制 `from/sourceLayer/priority/learnable`；Router 必须验证候选数量、ID、置信度、单行、语言、mixed、长度、epoch 与 deadline。
- [ ] 新公共架构必须先在未过滤机会集证明绝对 Oracle，再训练 Gate；bank miss 不得再次消失于分母或伪装成 abstain。

## BUG-121: MKN 淘汰按阶数排序且沿用完整模型回退权重

- **现象**: 首轮 3MiB V2S 候选的两个语言分区都只保留 5 阶记录，2–4 阶回退实际为空；改为信息量排序后虽然恢复多阶记录，但被剪枝上下文仍沿用完整模型的 backoff，局部概率总质量可下降到约 25%，导致不同上下文的分数、Gate 特征和触发阈值不可比较。
- **根因**: 训练器把“阶数越高”误当作压缩效用，并用平铺记录字节估算替代真实 Trie 成本；选择子集后又直接复制剪枝前的 `lambda`，没有按保留的 local mass 重算 `1 - ΣP(local)`。训练侧用逐层 Q16 舍入，运行时却使用浮点递推，边界同分仍可能换序。
- **根因类别**: 补全模型 / 概率归一化 / 序列化 / 评测可信度
- **修复**: 淘汰改为确定性相对熵/真实 Trie 摊销字节排序，预算不足时继续扫描可容纳记录；每个嵌套预算按保留 local mass 重算 backoff。MKN 二进制升级为压缩 TypedArray Trie v2，运行时按同一 Q16 公式逐层递推，并在测试中验证多阶保留、预算嵌套、归一化和训练—运行时容器往返。
- **教训**: 语言模型压缩不能只验证“文件没超预算”；必须同时验证各阶覆盖、剪枝后概率质量、编码器/运行时位级语义和真实候选指标。

## 检查清单追加

- [ ] MKN/Trie 淘汰报告必须列出每语言、每阶记录数；任一计划中的回退阶次意外归零都应阻断候选评测。
- [ ] 删除 continuation 后必须重算该上下文的 backoff，并验证 `Σlocal + lambda = 1`（量化容差内）。
- [ ] 训练侧与运行时的定点概率递推必须共享 golden，不能用“数学近似相同”替代位级一致。

## BUG-122: V2S 发布验证器相信自报候选与治理汇总

- **现象**: 候选描述符只要提交格式正确的 200 条 observation、低延迟数组和通过布尔值，即使这些候选并非绑定 `.bin` 的真实输出，也可能得到 `runtimeEligible/qualityGatePassed/releaseEligible=true`；selection verifier 同时只检查 upstream 存在 `sources` 数组，没有逐文档证明来源、许可证和正文身份。
- **根因**: 证据链只重算“报告内部的算术”，没有用绑定模型和冻结 `target.text/cursorOffset` 重放候选，也没有从 upstream source/document 清单重建治理事实。模型、输入、输出和发布结论之间只有字符串 SHA 关联，没有执行语义闭环。
- **根因类别**: 发布证据 / 评测可信度 / 语料治理 / 补全模型
- **修复**: 新增候选专用确定性 replay runner，经真实 `MarkdownPredictor`、Resolver 与绑定 V2S 二进制重算 Public-off、未门控 Public、门控 Public 和 Combined Top-1；verifier 必须逐 checkpoint 比对重放结果。selection/governance 改为逐文档回溯批准 source、许可证、content root、语言、类别、字节和 SHA，并从原始条目重算汇总。任何缺失重放、来源越界或自报结果不一致继续 fail-closed。
- **教训**: 哈希只能证明“文件没变”，不能证明“文件里的结论是真的”。质量证据必须从绑定输入和绑定可执行模型重新产生；治理报告也必须由原始来源清单重算，不能验证自己的声明。

## 检查清单追加

- [ ] 正式质量 verifier 必须用绑定模型、冻结正文和光标重放候选；只重算 observation JSON 指标不构成模型质量证据。
- [ ] selection 中每篇文档都必须能回溯到批准 upstream document/source，并复核许可证、content root、语言、类别、字节与内容 SHA。
- [ ] 运行时延迟和 WebView 证据必须由 RC 当前执行链重新产生并绑定模型/commit；预填的低延迟数组或通过布尔值不得直接放行。

## BUG-123: 双语物理分区却按总体分数选择单一 tokenizer

- **现象**: 固定矩阵按总体 Oracle 选择 Unigram 后，最大 24MiB/5.5MiB 候选的英文 Oracle@8 仅 19/100；首档 BPE 的英文 Oracle@8 已达到 32/100，而中文表现恰好相反。继续扩大同一个全局 tokenizer 只会在两种语言之间交换损失。
- **根因**: V2S 二进制和 MKN Trie 已按中英文物理分区，但训练矩阵仍把 tokenizer 当成全局枚举并按总体平均分选胜者，错误地把两个独立表示问题重新耦合。
- **根因类别**: 补全模型 / 双语建模 / 训练方法 / 评测口径
- **修复**: tokenizer 选择改为逐语言比较；只允许一次预注册的方法修正，将同一 selection、语料前缀、阶数、量化和预算约束下的英文 BPE 分区与中文 Unigram 分区合成一份 canonical 候选。合成器校验输入树、训练字节、预算、Gate 和量化参数一致，仍只写候选目录并保持 fail-closed。
- **教训**: 物理分区必须延伸到模型选择和停止判断；总体平均不能掩盖任一语言的表示退化。

## 检查清单追加

- [ ] 双语模型的 tokenizer、Oracle 和停止结论必须逐语言计算；不得仅按总体平均选择同一表示方法。
- [ ] 组合语言分区前必须证明它们来自同一冻结 selection、相同语料前缀、训练预算和序列化语义；组合不是额外超参搜索。

## BUG-124: 唯一模型源码审计递归遍历 30MiB 生成缓存

- **现象**: Public V2S 全量训练生成约十万篇缓存文档后，`autocomplete-single-model-source.test.ts` 的唯一 publisher 审计在全量 Vitest 中超过 5 秒；单测逻辑本身没有发现多 publisher。
- **根因**: `walkFiles(scripts)` 先递归枚举 `scripts/corpus/_web-cache` 的全部生成文件，之后才按 `.ts/.mjs` 扩展名过滤。缓存目录并非源码审计范围，却承担了绝大多数遍历成本。
- **根因类别**: 补全模型 / 测试基础设施 / 缓存边界
- **修复**: 源码树遍历在目录层直接跳过 `_web-cache`，继续扫描全部受版本控制的 scripts 源码；不提高测试超时，也不把缓存产物计入唯一真相源证据。
- **教训**: 源码闭包审计必须在递归入口排除生成缓存，不能先枚举整个语料池再做文件扩展名过滤。

## 检查清单追加

- [ ] 任何递归源码/依赖扫描都必须在目录层排除 `_web-cache`、构建物和报告缓存；过滤应发生在遍历前而非遍历后。

## BUG-125: 架构停止后默认 factory 仍把 V2S Worker 打入生产包

- **现象**: canonical V2S 资产不存在且 architecture stop 已生效，但 production build 仍生成约 22.05KB 的 `public-v2s.worker` chunk；运行时会先尝试构造默认 factory，再因 manifest 缺失返回空。
- **根因**: 停止合同只阻断训练、publisher 和 RC，`MarkdownPredictor` 仍静态导入 `createCanonicalPublicV2sEngine`。Vite 因静态依赖保留整条 Worker 构造链，导致停止态仍占用生产依赖图。
- **根因类别**: 补全模型 / 生命周期 / 构建边界 / 唯一真相源
- **修复**: 生产 Predictor 只安装构造时显式注入的 `CompletionPublicEngine`，不再自动导入 V2S factory；V2S engine/factory/Worker 源码保留给单元测试和隔离评测，生产 bundle 不可达。
- **教训**: architecture stop 必须同步撤销默认构造入口；“manifest 不存在所以不会运行”不能替代生产依赖图清理。

## 检查清单追加

- [ ] 公共模型进入 architecture stop 后必须检查生产 bundle 是否仍包含其 factory、Worker、WASM 或推理依赖；无运行资格的实现不得依赖仅靠 404/空 manifest 关闭。

## BUG-126: 连续学习 E2E 把文件重挂载与不可启动的冷态接受混进 Personal L2 评测

- **现象**: `22-autocomplete-continuous-learning.spec.ts` 清空正文后可能因自动保存与 keyed editor 重挂载切回另一篇笔记；即使规避焦点问题，固定冷态候选全部错误，Tab 接受数为 0，却仍要求“重复接受后提升”。
- **根因**: 用例同时承担文件生命周期、ghost 交互和 Personal L2 学习三种职责；它依赖键盘清空当前真实笔记，并假定冷启动必然先给出可接受候选。Public L3 停止后，这个启动前提不再成立。
- **根因类别**: 补全模型 / E2E 隔离 / 评测口径 / 生命周期
- **修复**: Personal L2 用例改为通过生产 Predictor 的浏览器 bridge 比较冷态诊断与显式接受历史后的学习态诊断；接受历史仍调用产品 `acceptCompletion(..., learn:true)` 路径。真实 Tab、Escape、Shift+Tab 与 blur 契约继续由 `15-autocomplete-journey.spec.ts` 独立覆盖，不再修改笔记正文或触发自动保存。
- **教训**: 学习评测必须先提供真实可接受事件；冷态没有正确候选时，不能用“零次接受”证明接受学习失败。文件保存与编辑器重挂载也不应成为模型学习基准的隐含前置条件。

## 检查清单追加

- [ ] Personal L2 学习基准必须显式区分冷态候选能力、接受事件注入和学习后排序，不得假定失格 Public 模型能提供首次正确候选。
- [ ] 补全质量 E2E 不应通过清空真实笔记制造训练轮次；Tab/失焦交互与 Predictor 学习指标应由职责单一的用例分别验证。

## BUG-127: Architecture stop 在 CLI 与 CI 证据边界发生语义分叉

- **现象**: 停止态执行 `autocomplete-v2s train/repack-gate` 时，CLI 会先读取不存在的 gate samples 并报 ENOENT；同时 CI/RC 的 JavaScript verifier 只验证 stop 身份字段和自哈希，重算自哈希后的 lifecycle/gates/Oracle 篡改仍可能被报告为 fail-closed stop。
- **根因**: 底层 trainer/repack 有统一 stop guard，但 CLI 在进入底层前自行读取输入；训练侧 TypeScript 与发布侧 JavaScript 又维护了强度不同的 stop 校验合同。
- **根因类别**: 补全模型 / 生命周期 / 发布证据 / 类型边界
- **修复**: CLI 的 train 与 repack 分支在读取任何 gate input 前调用 `assertV2SArchitectureActive`；JS verifier补齐 gates、Oracle 算术、frontier、bindings 和 lifecycle 校验，并增加 CLI 缺失输入及“语义篡改后重算自哈希”回归。
- **教训**: 停止合同必须覆盖最外层入口并在任何输入访问前执行；训练、CI 和 RC 对同一停止记录必须接受与拒绝同一组语义状态。

## BUG-128: 视觉基线平台漂移与自报式 RC 证据形成假闸门

- **现象**: Linux CI 运行 Halo Canvas 像素测试却只有 Win32 基线；通用 RC gate 仅相信 machine-evidence v1 自报的命令名和 exitCode，还要求受版本管理 manifest 内的 commit 等于包含该 manifest 的 HEAD，既可伪造又无法形成合法固定点。
- **根因**: 像素证据没有明确平台所有权；RC 把同一状态文件同时当 required cases、执行结果和 PASS 结论，并错误地让证据 commit 自引用自身 SHA。
- **根因类别**: CI / 发布证据 / 视觉回归 / 状态管理
- **修复**: Linux 三浏览器 E2E 排除 `@windows-visual`，新增 Windows Chromium job 实际执行 Win32 基线；修正 Playwright artifact 路径。永久删除 machine-evidence v1 的 PASS 路径，通用 RC 在候选 commit、只读原始报告、结构化二次转录和证据 commit 的 v2 协议实现前明确 fail-closed。
- **教训**: 平台专属像素基线必须由对应平台真实执行；证据不能自报 PASS，也不能在内容中绑定包含自身的 commit。无法闭环时应阻断，而不是保留看似严格的伪闸门。

## 检查清单追加

- [ ] Architecture stop 必须从 CLI 到 trainer/publisher/RC 全链路在输入读取前执行，并用重算自哈希的语义篡改测试验证各实现一致。
- [ ] 像素基线必须声明唯一执行平台；其他平台应由 workflow 排除并由专属 job 提供实际 PASS，不能依赖缺失基线或 skip。
- [ ] RC 证据必须区分候选 commit 与证据 commit；原始执行产物和二次转录必须独立绑定，状态 JSON 不得同时自定义 cases 与 PASS。
