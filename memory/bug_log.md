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

```markdown
## BUG-001: {标题}

- **现象**: {用户看到的错误表现}
- **根因**: {具体代码位置 + 数据流断裂点}
- **根因类别**: {文件IO / 渲染管线 / 状态管理 / 类型边界 / 跨平台兼容 / ...}
- **修复**: {改了什么，commit hash}
- **教训**: {一句话，可转为检查清单条目}
```

---

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

---

## 统计

| 指标               | 数值 |
| ------------------ | :--: |
| 总 BUG 数          |  6   |
| 真 BUG（已修复）   |  6   |
| DEFERRED（假 BUG） |  0   |
| 待修复             |  0   |

---

> 关联文档：`CLAUDE.md` §5.0（错题本必读）、§5.6（BUG 修复前置规则）、§5.9（代码审查易错点）
