# M1 开发会话 — 操作与数据流记录

> 开始时间：2026-06-03
> Dev Server：http://localhost:5174/ (L4 复审)
> 当前阶段：M1 核心渲染与编辑 → L4 🔷 人工复审

---

## 数据流（M1 运行时）

```
浏览器 HTTP GET /
  → Vite Dev Server :5174
  → index.html → main.ts
    → App.vue → <router-view />
      → NotebookHome.vue (onMounted)
        → new MockFSService(50ms delay)
          → initNotebook()
            → fs.listDirectory('/')
              → DirEntry[] [{name: '欢迎.md', ...}, {name: '快速入门.md', ...}]
        → AppLayout (三联画布局)
          ├── #left-sidebar
          │     └── FileTree (files=2, loading=false)
          │           └── FileTreeNode (📄 欢迎.md)
          │           └── FileTreeNode (📄 快速入门.md)
          │                 ↓ 用户点击
          │                 emit('select', '/欢迎.md')
          │                   → onSelectFile('/欢迎.md')
          │                     → fs.readFile('/欢迎.md')
          │                       → currentContent = '# 欢迎使用 MarkLuck...'
          └── #editor
                └── MarkdownEditor (:model-value=currentContent)
                      └── CodeMirror 6 EditorView
                            ├── markdown() 语法高亮
                            ├── blockDecorator (蓝色/绿色标记)
                            ├── imeHandler (中文输入追踪)
                            ├── throttledParser (150ms防抖)
                            └── markdownKeymap (Ctrl+B/I/K)
                      ↓ 用户编辑
                      emit('update:modelValue', newContent)
                        → onContentUpdate(newContent)
                          → fs.writeFile(path, content)  [自动保存]
```

---

## L4 🔷 复审清单

### 测试步骤与预期结果

| #   | 操作               | URL/位置                 | 预期结果                                                                              | 数据流验证点                                                                    |
| --- | ------------------ | ------------------------ | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| 1   | 打开首页           | `http://localhost:5174/` | 左侧显示文件列表（📄 欢迎.md, 📄 快速入门.md），中间显示 "选择左侧一条笔记开始编辑"   | NotebookHome onMounted → MockFSService.listDirectory → files ref → FileTree渲染 |
| 2   | 点击 "欢迎.md"     | 左侧文件树               | 中间编辑区显示 CodeMirror 6 编辑器，内容为 "# 欢迎使用 MarkLuck..."                   | onSelectFile → fs.readFile → MarkdownEditor :model-value                        |
| 3   | 检查 CM6 编辑器    | 中间编辑区               | 代码编辑器有行号、Markdown 语法高亮、块标记（左侧蓝色竖线）                           | CodeMirror 6 EditorView + extensions                                            |
| 4   | F12 Console        | DevTools                 | 无红色错误，无 console.log 输出                                                       | ESLint no-console 规则生效                                                      |
| 5   | 输入 Markdown 语法 | 编辑器                   | `**bold**` → 语法高亮；`# heading` → 标题高亮                                         | markdown() extension                                                            |
| 6   | Ctrl+B 快捷键      | 编辑器                   | 选中文字 → 按 Ctrl+B → 自动包裹 `**text**`                                            | markdownKeymap                                                                  |
| 7   | 修改内容后刷新     | F5                       | 重新打开同一文件，内容为刷新前最后编辑的内容                                          | MockFSService.writeFile (自动保存)                                              |
| 8   | 布局验证           | 桌面浏览器               | 三栏布局：左侧 ~260px 文件树 + 中间编辑器 + 右侧 ~240px 导航面板（显示"导航面板 M2"） | AppLayout CSS Grid                                                              |
| 9   | 空状态             | Mock 无文件时            | 显示 "空文件夹 — 创建你的第一条笔记开始吧"                                            | FileTree Empty 状态                                                             |
| 10  | 错误状态           | Mock 抛出异常            | 显示错误信息 + 重试按钮                                                               | FileTree Error 状态                                                             |

---

## 复审结果记录

| #   | 结果 | 备注 |
| --- | :--: | ---- |
| 1   |  ⬜  |      |
| 2   |  ⬜  |      |
| 3   |  ⬜  |      |
| 4   |  ⬜  |      |
| 5   |  ⬜  |      |
| 6   |  ⬜  |      |
| 7   |  ⬜  |      |
| 8   |  ⬜  |      |
| 9   |  ⬜  |      |
| 10  |  ⬜  |      |
