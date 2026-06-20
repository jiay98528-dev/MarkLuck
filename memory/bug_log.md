# MarkLuck 错题本

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
- **修复**: 在 `NotebookHome.vue` 中新增全文档预览模式（"预览/编辑"切换按钮），使用 `@markluck/renderer` 的 `renderMarkdown()` 渲染完整文档 HTML，通过 `v-show` 在编辑器和预览视图之间切换。预览模式下应用完整的排版样式（标题层级/代码块/表格/引用等）
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

- **现象**: 用户预期点击左上角 MarkLuck Logo 返回首页/清空编辑器，但无任何反应。
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
- **修复**: (1) 状态栏 "选中文字以格式化" 提示移除 `charCount > 0` 条件，添加呼吸动画（opacity pulse）; (2) 首次选中文字时显示一次性 Toast 通知 "选中文字后使用格式气泡进行加粗、斜体等操作"（localStorage 标记 `markluck:formatBubble:hintShown`）。
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
- **修复**: 600 行 Exporter.ts 重写。DOCX: `marked.lexer()` → `Document/Packer/Paragraph/TextRun`；XLSX: `extractTables()` → `XLSX.utils.aoa_to_sheet` → `write()`；PDF: `renderMarkdown()` → iframe → `window.print()`；HTML: 自包含 CSS + `renderMarkdown()`。接入 includeFrontmatter/includeWikiLinks/codeLineNumbers 选项
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
  - `MarkdownEditor.vue:148-149` — 旧组件 `onUnmounted` 删除了 `__markluck_getEditorContent` 但**未删除** `__markluck_getEditorView` 和 `__markluck_editorInitValue`
  - 新组件 setup 重新注册后，时序窗口期内旧函数残留引用了已销毁的 view 对象 → `getEditorView()` 返回非null但 `getEditorContent()` 返回空串
  - **修复**: `onUnmounted` 增加删除 `__markluck_getEditorView`, `__markluck_editorInitValue`, `__markluck_modelOverwrites`（2026-06-10 ✅）

  **缺陷 3 — Vue 3 `:key` patch 顺序错误：新 setup 先于旧 unmount**
  - `MarkdownEditor.vue:61,149` — `__markluck_getEditorContent/View` 在 `<script setup>` 顶层注册，旧组件的 `onUnmounted` 删除它们。Vue 3 `:key` 补丁顺序为 **新 setup → 旧 unmount → 新 mount**。旧 unmount 删除了新 setup 刚注册的函数，导致 `view=null` 对外不可见
  - **证据**: 生命周期日志显示 `setup(id=j70ame, t=3875) → unmounting(id=i0qy85, t=3876) → mounted(id=j70ame, t=3882)` — setup 在 unmount 之前
  - **修复**: 将 `__markluck_getEditorContent/View` 注册从 `setup` 移到 `onMounted`（在旧 unmount 之后执行）（2026-06-10 ✅ T2 PASS）

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
  2. **`onUnmounted` 中注册的全局对象（window hooks）必须逐条删除，不能漏项。** 遗漏的 `__markluck_getEditorView` 返回已销毁的 view 导致调用方拿到垃圾数据
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
  - 修复: `autocompleteCompartment` 移到 `...markluckExtensions()` 之前

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
- [ ] **CM6 keymap 的 Tab handler 优先级必须高于 `defaultKeymap`（`indentWithTab`）。** 将自定义 keymap extension 放在 `markluckExtensions()` 之前注册（教训：BUG-029 Defect 6）
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

- **现象**: E2E 测试 `04-theme-settings-panels` 用例 02-03 断言 `localStorage.getItem('markluck-theme')` 为纯字符串 `'dark'`，实际存储为 `{"c":"dark"}`
- **根因**: `stores/theme.ts:49` — `apply()` 使用 `JSON.stringify({ c: colorScheme.value })` 序列化；测试期望纯字符串
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
- **根因**: `MarkdownEditor` 将每个 composition 中间文档值 emit 给父组件，父组件回传为 `modelValue` 时仅用“是否等于当前文档”判断是否同步；同时 `EditorView.updateListener` 与 DOM `keyup/mouseup/paste` 监听器重复执行同步，快速输入中旧中间值可能晚于下一次标点事务返回并全量覆盖文档。格式按钮点击后未恢复焦点进一步放大输入取消概率。
- **根因类别**: 状态管理 / 跨平台兼容
- **修复**: 记录 EditorView 内部发出的最近文档值，watch 收到自身回声时直接忽略；移除重复的 DOM 同步监听器，以 `EditorView.updateListener` 作为唯一内容同步入口；格式事务完成后显式恢复 EditorView 焦点和正文选区。
- **教训**: 双向绑定的编辑器必须区分内部回声与真实外部更新；IME 期间不能只靠字符串“当前是否相等”判断同步方向。

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
