# 离线文字补全改造记录

日期：2026-06-23

## 背景

本轮目标是修复离线文字补全“出现概率低、预测不准、设置不实时生效”的体验问题，同时保持 MarkLuck 的产品约束：完全离线、单一 ghost text、不弹出候选菜单。

## 已完成改动

### 1. 设置状态统一

- 新增 `packages/app/src/services/CompletionSettings.ts`。
- 新增 `CompletionSettings` 单一状态源：
  - `enabled`
  - `aggressiveness: 'balanced'`
  - `backgroundTraining`
  - `maxSuggestionLength`
  - `minConfidence`
  - `showDebugStats`
- 兼容旧 key：`markluck:autocomplete:enabled`。
- 使用自定义事件同步设置变化，避免设置页只写 localStorage、编辑器不响应的问题。

### 2. 预测器重构

- 重写 `packages/app/src/services/MarkdownPredictor.ts` 的融合逻辑。
- 保留并优先返回结构化补全：
  - Wiki-link
  - tag
  - file-path
  - Markdown 格式闭合
  - Markdown 结构建议
- 普通文本改为三层来源：
  - L1：当前文档内存模型
  - L2：当前笔记本本地训练模型
  - L3：出厂 baseline 模型
- 增加中文短上下文 fallback，例如“这是 / 为了 / 用户 / 项目 / 今天”。
- 增加质量门控：
  - 过滤中英文上下文错配
  - 过滤重复字符级联
  - 过滤 Markdown 噪声
  - 标点后避免不合理普通文本建议
- 接受补全后写入 L1/L2，并立即持久化。
- 拒绝或继续输入不匹配时对对应预测降权。

### 3. N-gram 引擎改进

- 更新 `packages/app/src/utils/ngram-engine.ts`。
- 预测置信度从“预测越长越高”的近似值，改为首步置信度与平均置信度组合。
- 保留旧序列化格式兼容。

### 4. 后台训练服务

- 新增 `packages/app/src/services/CompletionTrainingService.ts`。
- 支持低优先级训练当前笔记本文件：
  - 仅处理 `.md/.markdown/.txt`
  - 跳过 `assets/`
  - 跳过超大文件
  - 跳过 frontmatter、代码块、行内代码和图片引用
- 写入训练元数据：`markluck:autocomplete:trainingMeta`。
- 元数据包含：
  - version
  - status
  - trainedPaths
  - fileCount
  - updatedAt
  - lastError
- 保存、新建、重命名、删除后做增量训练状态更新。

### 5. 编辑器与 ghost text 插件

- 更新 `packages/app/src/components/editor/MarkdownEditor.vue`：
  - 接收 `completionSettings`
  - 设置变化时实时 reconfigure CodeMirror autocomplete compartment
  - 初始化预测器时使用统一设置
- 更新 `packages/app/src/utils/cm6-ghost-text.ts`：
  - balanced 防抖改为 120ms
  - Tab 接受后记录命中
  - Escape 或继续输入不匹配时记录拒绝/降权
  - Firefox 下改用 StateField 驱动 decorations，修复异步预测后 ghost text 不渲染的问题
  - 继续保留 compositionstart/compositionend 的 IME 安全防护

### 6. 设置页补齐

- 更新 `packages/app/src/components/modals/SettingsDialog.vue`。
- “文字补全”页新增：
  - 启用 ghost text 开关
  - 后台训练开关
  - 训练状态
  - 已训练文件数
  - 上次训练时间
  - “仅处理当前笔记本的本地 Markdown / 文本文件，不上传内容”说明
- 开关实时影响当前编辑器，无需刷新。

### 7. NotebookHome 接线

- 更新 `packages/app/src/pages/NotebookHome.vue`。
- 将 `completionSettings` 传入 split/live 两个 MarkdownEditor。
- 订阅补全设置和训练元数据变化。
- 索引初始化、文件保存、新建、重命名、删除后触发后台训练或元数据更新。

### 8. Baseline 语料和报告

- 更新 `scripts/corpus/corpus.config.json`：
  - 降低小说语料权重
  - 提高技术写作、Markdown 结构、项目文档权重
  - 新增真实笔记风格语料源 `scripts/corpus/note-patterns-zh/`
- 新增 `scripts/corpus/note-patterns-zh/common-notes.md`。
- 更新 `scripts/train-baseline.ts`：
  - 训练报告输出固定 probe 集结果
- 重新生成：
  - `packages/app/public/baseline-ngram.v1.compact.txt`
  - `scripts/corpus/training-report.json`
- 固定 probe 集输出 9/9：
  - `这是 -> 一个`
  - `为了 -> 更好`
  - `用户 -> 可以`
  - `项目 -> 进度`
  - `今天 -> 的`
  - `-  -> [ ] `
  - `[[ -> 结构化 wiki-link`
  - `# -> 标题`
  - `**粗 -> **`

### 9. 测试覆盖

- 更新 `packages/app/src/services/__tests__/MarkdownPredictor.test.ts`：
  - 中文短上下文 fallback
  - 跨语言过滤
  - 设置禁用
  - L3 baseline 分层行为
- 新增 `packages/app/src/services/__tests__/CompletionTrainingService.test.ts`：
  - 代码块/行内代码清理
  - 文件过滤与训练 meta 写入
  - 删除路径后 meta 更新
- 新增 `e2e/tests/15-autocomplete-journey.spec.ts`：
  - 输入常见短句或结构化上下文 -> ghost text 出现 -> Tab 接受 -> 保存 -> 刷新后仍可预测
  - 设置里关闭补全 -> 当前编辑器 ghost text 立即消失

## GUI 实测结果

使用内置浏览器在 `http://localhost:5173/` 执行真实 GUI 操作：

- 输入“这是”后出现 ghost text。
- 按 Tab 后 ghost text 被接受，编辑器内容更新。
- 打开设置 -> 文字补全页：
  - 启用开关可见
  - 后台训练状态可见
  - 已训练文件数可见
  - 上次训练时间可见
  - 本地处理说明可见
- 关闭补全开关后，当前编辑器重新输入“这是”不再出现 ghost text。
- 测试结束后已恢复补全开关为开启。

## 自动化验证结果

已通过：

- `pnpm --filter @markluck/app typecheck`
- `npx eslint packages/app/src packages/renderer/src`
- `npx prettier --check` 本次修改文件集合
- `pnpm --filter @markluck/app exec vitest run`
  - 5 个测试文件通过
  - 144 个测试通过
- `pnpm --filter @markluck/app build`
- `npx playwright test e2e/tests/15-autocomplete-journey.spec.ts --browser=chromium`
  - 2/2 通过
- `npx playwright test e2e/tests/15-autocomplete-journey.spec.ts --browser=firefox`
  - 2/2 通过

说明：

- Firefox 自动化里中文输入会走不同的 composition 路径；E2E 使用结构化补全 probe 验证 Firefox 的 ghost text 插件渲染、Tab 接受和设置实时关闭。
- Chromium 和内置浏览器 GUI 已覆盖中文短上下文 ghost text。

## 当前未完成项

1. `stylelint` 未执行成功。
   - `pnpm exec stylelint "packages/app/src/**/*.{vue,css}"` 失败：本地没有 `stylelint` 命令。
   - `npx stylelint` 会拉临时包，但缺少项目扩展配置 `stylelint-config-standard`。
   - 处理方式：补齐本地 stylelint 依赖后重跑。

2. WebKit 未验证。
   - 本轮完成 Chromium + Firefox。
   - 如发布闸门要求三引擎，需要安装/配置 WebKit 浏览器后补跑。

3. 未执行全量历史 E2E。
   - 本轮针对补全新增用例已通过 Chromium + Firefox。
   - 全量 E2E 数量较多，建议发布前另起一次完整回归。

4. 训练报告 source coverage 仍按原始上下文数量统计。
   - 小说语料权重已降到 0.4，但报告里的 coverage 显示仍基于原始条目数，因此看起来小说占比仍高。
   - 后续可把报告增加 `weightedEntries`，让权重后的分布更直观。

5. Background training 的真实文件系统压力测试未单独覆盖。
   - Web/MockFS 与单元测试已覆盖核心逻辑。
   - Tauri 真实大笔记本场景建议补一次手动或桌面 E2E：大量文件、mtime 变化、删除/重命名、超大文件跳过。

## 重点风险与后续建议

- IME 相关逻辑仍是高风险区。当前策略保留 compositionstart/compositionend 防护，同时避免 Firefox 自动化输入长期抑制 ghost text。后续如果改动 CM6 或输入法逻辑，应回归中文 IME、Firefox、Tab/Escape。
- L2 训练现在会自动持久化，若用户笔记本非常大，需要继续观察 localStorage 容量与淘汰策略。
- 建议下一轮补一个训练状态的可视化回归截图，确保设置页在窄屏和暗色主题下仍不溢出。
