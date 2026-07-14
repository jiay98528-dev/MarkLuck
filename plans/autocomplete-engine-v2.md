# 离线补全引擎 V2 执行计划与 V2.1 扩展方向

> 日期：2026-07-14
> 状态：V2 工作区运行时完成；V2R/V2S 公共模型实验均已停止；公共 L3 默认未绑定；不再继续扩大语料或训练，V2.1 未解锁
> 范围：仅文字补全引擎、模型与对应规格；不涉及主题、UI 重设计或品牌重命名
> 训练操作：统一见 `doc/autocomplete-model-training.md`；本文只保留架构演进与实验结论

## 1. 目标与边界

V2 用混合检索替代“继续扩大公共 N-gram 语料”作为主要提升路线。在不生成新文本的前提下，把结构化规则、当前文档复用、工作区短语检索、英文词前缀和现有分层 N-gram 汇合为最多 8 条可审计候选，再由统一排名与质量门控输出唯一 ghost text。

V2 必须独立、免费且完整可用。V2.1 可在未来以付费插件提供微型 Transformer 语义重排，但不能成为 V2 的运行依赖，也不能削弱免费路径。

本轮明确不做：

- 不新增或抓取 30MiB 语料；现有训练管线和 6MiB 运行时资产上限保持不变。
- 不做开放式文本生成，不让模型创造候选池之外的文本。
- 不提供候选菜单，仍只显示一条 ghost text。
- 不允许插件执行任意 JavaScript、WASM 或原生代码。
- 不修改主题、主题运行时、主题文档或任何品牌名称。

## 2. V2 运行时架构

```text
Markdown context snapshot
  ├─ structured providers（确定性旁路）
  └─ free V2 candidate sources
       ├─ current-document reuse / recent phrase
       ├─ notebook phrase retrieval
       ├─ lexicon / English word prefix
       └─ Personal L2 / Notebook N2 / optional public L3 (currently unbound)
            ↓ normalize + language/Markdown/quality gates
       CandidateBatch(top 8) + deterministic fallback
            ↓ common lightweight ranker
       CompletionResolver
            ↓
       one ghost text
```

结构化 Wiki-link、标签、文件路径、格式闭合和列表延续不进入语义重排，始终由公共 Resolver 以最高层级处理。普通候选只允许在 paragraph/list/quote 行尾触发，heading/table/code/frontmatter 继续静默。

### 2.1 候选批次契约

每次请求必须携带不可变快照：

- `requestId`、`epoch`、`workspaceScope`
- `docVersion`、UTF-16 `cursorPos`
- 截断后的上下文和语言/Markdown block 信息
- `deadlineAt` 与 `AbortSignal`

`CandidateBatch` 最多保留 8 条经过规范化、去重和硬门控的普通候选，同时保存无需异步后端即可得到的确定性 V2 fallback。候选必须保留 `text`、`from`、`providerId`、`sourceLayer`、`confidence` 和可学习属性；后端或未来插件不得改写候选文本与插入位置。

### 2.2 双后端

- Web/PWA：专用 Web Worker 持有工作区短语索引，后台执行替换、删除、重命名和查询。
- Tauri：Rust 应用状态持有等价内存索引，通过固定 IPC 命令执行相同协议；不扫描工作区之外的路径，不持久化正文派生索引。
- 两端使用同一 TypeScript 候选归一化、轻量排名、Resolver 和质量门控，保证行为可对照。
- 后端不可用、超时或返回损坏数据时立即使用确定性 V2 fallback；编辑器不得因此阻塞。

### 2.3 生命周期与交互

- 工作区切换递增 engine epoch、取消旧请求并清空旧 scope；文档贡献按路径替换，重复保存幂等。
- 每次输入/选区变化取消上一请求。异步结果只有在 epoch、docVersion、cursor 和当前焦点都仍匹配时才可显示。
- 默认 deadline 为桌面 80ms、Web 110ms；超过 deadline 使用免费同步 fallback。已经显示的 ghost 绝不被迟到结果替换。
- 只有无修饰键 `Tab` 接受；`Shift+Tab` 保持原生行为；`Escape` 明确拒绝；blur、窗口失焦、弹窗切换和继续输入只清除。

### 2.4 工作区索引资源预算

Web Worker 与 Rust 后端必须采用相同的 fail-closed 默认预算：最多 2,000 篇文档、单文档输入 512KiB、总输入 16MiB、单文档 20,000 个贡献条目、总计 300,000 个贡献条目。文档贡献只保存内容 fingerprint、输入字节数、条目数和可逆聚合，不保留整篇正文。

替换文档时按“撤销旧贡献后的净值”核算预算，并先在临时贡献中完成扫描；任一预算超限必须拒绝本次替换并原样保留旧贡献。诊断只暴露计数、预算和拒绝次数，不暴露正文或路径内容。

## 3. V2 实施阶段

1. 定义 `CandidateBatch`、异步后端协议、请求 epoch/deadline/cancel 与统一候选排名入口。
2. 把现有 Provider 输出扩展为可复用 top-k，保留同步 `getGhostText()` 兼容 facade。
3. 实现可测试的纯 TypeScript 短语检索核心和 Web Worker adapter，包含 latest-only 查询取消、异常清理和硬资源预算。
4. 实现等价 Rust 内存检索状态与 Tauri adapter；统一 Unicode 规范化、资源预算与协议，失败自动退回本地免费候选。
5. 将 ghost text 调用改为可取消异步请求，验证迟到结果、销毁、IME 和焦点域。
6. 使用独立 holdout 比较 V1 与 V2 的 oracle@8、top-1、可用率、误触发率和延迟；只有质量闸门通过才切换默认引擎。

V2 验收维持：真实写作触发率 35%–42%、可用率至少 35%、误触发率不超过 3%、mixed 为 0、可见预测 p90 不超过 140ms；单条 ghost、无候选菜单。

V2 的公共检索资产（字符/词级基线及未来只读短语包合计）以 ≤4MiB 为产品目标，6MiB 为不可突破的兼容硬上限；工作区内存索引和 V2.1 可选插件分别核算。质量不足时不得为了体积目标发布缩水模型。

## 4. V2.1：付费微型 Transformer 重排插件

V2.1 的定位是“Semantic Completion”增值包：只消费 V2 已通过硬门控的 top-8 普通候选和短上下文，返回候选索引及分数。首版不生成新文本。

```text
free V2 CandidateBatch
  ├─ timeout / unavailable ───────────────→ common Resolver
  └─ entitled + ready → SemanticReranker → common Resolver
```

### 4.1 插件格式与信任边界

`.mlcompletion` 是签名的数据包，而不是代码插件。manifest 至少包含：

- `schemaVersion`、`id`、`version`、`engineApiVersion`
- `runtime: micro-transformer-reranker-v1`
- `capabilities: ["rerank"]`、语言和最小应用版本
- 模型、tokenizer 的文件名、字节数与 SHA-256
- 产品/授权标识和发布签名

宿主只加载白名单架构和算子。新架构需要应用更新；包内不得携带任意 JS、WASM、sidecar 或系统权限。安装必须先校验签名、hash、API 和授权，再在隔离 Worker 中预热，成功后原子切换；失败保留旧版本并可回滚。

### 4.2 推荐模型与运行时

- 首选 2 层、`d_model=128`、4 heads、FFN 256、约 4096 词表、64–96 token 上下文的 cross-encoder reranker。
- 目标约 0.8–1.5M 参数；INT8 权重约 1–2MiB，模型、tokenizer 和 manifest 合计约 1–3MiB。
- 首版 Web 与 Tauri WebView 共用固定 ONNX/WASM Worker 宿主以支持真正热切换；若桌面延迟证据不达标，再增加使用相同 ONNX 权重和协议的 Rust 原生 adapter。
- 不使用运行时下载的 Tauri sidecar 作为首版插件机制。

### 4.3 产品、隐私与放行门槛

- 推理只接收截断上下文和 top-8 候选，不接收工作区路径、文件系统能力或网络权限。
- 授权到期、更新或切换不能在一次按键处理中途替换引擎；安全边界后生效。
- 离线权重无法承诺不可提取。商业价值依赖签名授权、便捷安装、持续更新与支持，而不是不可破解的本地加密。
- 只有当免费 V2 的 `Oracle@8 - Top1 >= 8pp` 时，付费语义重排才有足够机会空间。
- 相比 V2，V2.1 的可用率至少提升 3 个百分点或机会集条件可用率提升 8 个百分点；静默负例误触发恶化不超过 0.2 个百分点，mixed 保持 0。
- 插件推理 p90 目标：桌面 ≤50ms、Web ≤80ms；超时 fallback 率 ≤5%；签名插件包连同精简运行时目标 ≤15MiB。

## 5. 架构适配结论

V2 的分层 top-k、统一 Resolver、单 ghost、离线门控和 Web/Desktop 双后端方向与 V2.1 兼容，无需改换模型路线。唯一必须在 V2 阶段补齐的是候选批次后的可取消异步扩展缝；它同时是 Web Worker/Tauri 后端的正常需求，不是为付费功能引入的额外耦合。

因此 V2.1 不进入 Provider 列表，不复用 Theme API，也不影响 L1/N2/Personal L2/L3 的训练和持久化。免费 V2 完成并取得 oracle@8 证据后，再决定是否启动 V2.1 模型训练。

## 6. 2026-07-11 执行记录

V2 运行时已按本计划实现：CandidateBatch/Router 异步接缝、稳定 Predictor 生命周期、Web Worker 与 Tauri/Rust 双后端、按路径可逆贡献、latest-only 取消、deadline/epoch/feedback token、损坏响应降级、共同 NFKC 和共同硬预算均已落地。生产构建中的独立 Worker chunk 为 12.71KiB。

验证结果：

- App typecheck、ESLint、目标格式检查通过；Vitest 41 files / 502 tests 通过。
- Rust 全量 30/30 通过；Web/Rust 预算、兼容 Unicode、Markdown 排除和协议测试通过。
- Chromium 补全 E2E 24/24 通过，另有正式 holdout 1/1 通过；真实 Web Worker 最终候选可归因为 `hybrid-retrieval-zh/en:notebook`。
- 种子 notebook 场景分 100，误触发 0、mixed 0、场景 p90 ≤105ms；真实连续写作触发率/可用率 36.8%、误触发 0、mixed 0、p90 110ms。

发布闸门仍保持关闭：独立公共 holdout 的 200 个机会点仅触发 1.5%、可用率 0，虽然误触发 1.5%、mixed 0、p90 62ms，但当前 legacy L3 manifest 正确保持 `runtimeEligible:false`、`qualityGatePassed:false`、`releaseEligible:false`。因此本轮完成的是 V2 引擎与工作区检索能力，不宣称公共基线已达到发布质量；也不启动 V2.1 训练，需先取得独立 oracle@8 与桌面 WebView smoke 证据。

## 7. P1/P2 可信度与可靠性收口

- 评测证据拆分为治理正确、运行时安全、模型质量三类。固定探针和 seeded 场景只保留诊断身份，不得写入发布分数。
- `workspace-conditioned-holdout.json` 降级为 `releaseEvidence:false` 的诊断历史；正式 `workspace-conditioned-final-v2.json` 覆盖 50 篇目标、200 个 checkpoint、中英文各 100 个且 silence 50 个。final 新增结构模板与共享长片段闸门，并以 holdout SHA 写一次性 receipt；失败后更换模型也不能复用。
- V1 评测快照固定到 `fb46b1e`，包含独立子进程、压缩旧模型、逐文件 SHA、行为 golden 与聚合树 SHA。CI 在生产 build 后扫描真实 Vite `dist`，保证快照、observer 和旧模型 SHA 不进入应用包。
- 评测器分别报告 context hit、Top-1、Oracle@8、usable、false trigger、完整候选 mixed、全请求/可见预测 p90、Provider/sourceLayer 和拒绝原因。真实运行时报告强制绑定候选模型 SHA、完整 request 数与非零 visible 样本；零样本的 p90=0 不具备证据资格。
- Worker/CSP 不可用时关闭 Hybrid Retrieval，禁止在主线程同步建工作区索引。查询读取最近一次原子提交的 revision 快照，不等待 mutation backlog；健康诊断暴露后端类型、ready/warming/degraded/disabled、revision、积压、重建次数、构建耗时、估算内存和长任务数。
- 每个工作区每次会话只允许一次后端重建。重建由训练服务从当前工作区文件幂等回放；重建期间到达的 mutation 等待原子替换后再提交，第二次故障熔断并退回确定性免费路径。
- 24MiB 扩池仅使用固定 seed 的五个项目自有合成来源。正文和七档候选位于 Git 忽略目录；selection manifest 绑定实际文档 ID、片段 ID 与内容 SHA，小档必须是大档的真实严格前缀。七档候选始终 `releaseEligible:false`；RC 先生成候选，只对确定性合格档临时安装并通过生产 Router 测量，然后恢复 fail-closed public 资产并重新选档。
- final 通过后仍需由独立 post-final publisher 重读 selected candidate、receipt、质量/运行时/V1 对照/学习曲线和 WebView 证据，才可原子替换唯一 canonical `web-local` profile；`selectedTier:null`、缺失证据或任一 SHA 不一致时必须零写入。
- 发布证据分别保存 final holdout 与 final 报告；evaluator identity 的文件集合由发布代码固定并覆盖 Web/Worker/编辑器、E2E runtime evaluator、Rust 检索与共用 golden fixture。RC/publisher 必须读取实际源码重算 tree，并将冻结 V1 与固定 `fb46b1e` tree/model 身份复核，identity 文件不能自行缩小校验范围。

V2.1 继续保持关闭。只有免费 V2 已为 `releaseEligible:true`，且 workspace-conditioned holdout 的中英文分项都支持 `Oracle@8 - Top1 >= 8pp`，才允许进入微型 Transformer 训练。

七档项目自有合成 dry-run 已执行：五个来源、成员嵌套、治理、5.7/6MiB 预算全部通过；实际训练字节从 104,705 增至 25,160,955，模型从 353,097B 增至 840,413B。但 cold validation 的 usable 全部为 0，Top-1/Oracle@8 最高仅 1.33%，24MiB-cap 为 0.67%，触发率从 7% 降至 5.5%。因此 `selectedTier:null`，不运行候选 runtime/final/publisher，不替换 public，也不解锁 V2.1。另经新治理器复核，当前冻结 final-v2 的 support 结构模板占比为 25%，超过 10% 上限；即使 validation 日后选出候选，该 final 版本也必须先 fail-closed，不能作为手工独立证据。候选目录固定为 `scripts/corpus/_web-cache/autocomplete-candidates/learning-curve-v2/`。

## 8. 公共 L3 V2R 全量重构（2026-07-12）

### 8.1 重构原因与保留边界

24MiB 学习曲线已经证明旧 v4 不是容量不足：context hit 固定在 17.5%，usable 始终为 0。正式 validation 的 75 个英文正例都位于完整单词边界，后缀以空格或标点开头；旧字符模型会剥离这些边界，英文可见触发为 0。即使所有中文正例全部命中，总触发也只有 75/200，即 37.5%，结构上不可能达到新门槛。

因此停止调参公共 N-gram v4，替换其公共 L3 模型、训练器、加载器和证据。结构化 Provider、L1、Personal L2、Notebook/Hybrid、Resolver、ghost text 交互以及个人/Notebook N-gram 全部保留。旧 v4 public 资产在 V2R 放行前继续 fail-closed，放行后移出生产加载路径；冻结 V1 仍只用于评测。

### 8.2 固定实施阶段

1. 定义 `CompletionPublicEngine`、v5 manifest、Worker 协议、epoch/deadline/Abort 契约和损坏资产降级测试。
2. 建立项目自有生成器 v3、许可明确的可选 CC0 导入器、30MiB 分组语料治理和固定短语库构建器。
3. 为三档短语库先运行独立 validation 表示能力预检。总体 <70% 或任一语言 <65% 时跳过对应两个 seed，不消耗 CPU 训练，也不读取 final。
4. 只对通过预检的 8,192/96/2、12,288/128/2、16,384/128/3 档位执行每档两个 seed、最多 8 epochs、patience 2 的可复现训练。
5. 只在 development 校准 abstain/触发阈值，选择满足内部门槛的最小模型并冻结 ONNX、短语库和阈值。
6. 依次消费 cold final v3 与 workspace final v3。任一失败则该 final 版本作废，保留 RC code 10；全部通过才原子发布唯一 canonical profile。

当前管线状态：30MiB 池已由生成器 v3.1 与固定 SHA 的 Tatoeba English CC0 子集物化，并通过容量、来源、类别、重复、模板、高频 5-gram 和逐文档 trigram 集中度治理；Common Voice 未取得批准快照，因此其份额由项目自有生成器补足。训练输入固定为 silence-safe v3：bank miss 只计覆盖缺口，文档末尾才是 abstain；192-byte 上下文以 48×4-byte patch 输入。用于验证构建链的 reduced ORT v1.27.0 WASM 为 3,695,459B，未使用 stock 运行时。新的四套人工冻结 v3 holdout 尚未提交，所以正式矩阵必须在表示能力预检入口 fail closed，不得把旧 v2 诊断集或 smoke 模型升级为发布证据。

### 8.3 质量与资源闸门

- 每套 final 固定 200 checkpoints：触发 120–130，绝对可用至少 120；中文和英文分别 ≥55%，任一类别 ≥50%；50 个 silence 最多误触发 1，mixed 为 0。
- 每个正例在训练前冻结 3–5 条人工参考。中文候选至少 3 个有效汉字；英文至少一个完整单词且总字母数 ≥5。不得用纯空格、标点或低信息虚词刷命中。
- 公共资产合计 ≤6MiB，包含精简 Worker/WASM 的应用静态增量 ≤12MiB；模型工作不得在主线程产生 >50ms 长任务；全请求与可见 ghost p90 均 ≤140ms。
- 训练来源、许可、隐私、重复、训练—holdout 重叠和证据哈希任一失败均阻止发布。final 不可重跑，RC 不得只相信 manifest 布尔值。

### 8.4 与 V2.1 的关系

V2R 是免费的公共候选生成器，不是付费 V2.1 Ranker。它只能从固定短语库生成候选，并继续经过现有硬门控与 Resolver。V2.1 的数据型热插拔接口保留，但本轮不实现；其解锁条件将在 V2R 发布后重新依据同集 Oracle 差距、分语言收益和真实运行时证据计算。

### 8.5 2026-07-13 架构停止结论

固定矩阵的训练数据已完整生成，但不再启动 12,288/16,384 的长训练。原因不是 INT8、WASM、Worker 或模型体积：8,192 档 INT8 仅 1,225,507B，量化 Top-1/Oracle 一致率分别为 98.62%/99.79%。真正阻碍是固定输出空间无法覆盖真实写作 continuation。

- 8,192/12,288/16,384 短语库在生成池 internal split 的 bank coverage 分别为 74.54%/78.56%/80.97%。
- 同三档在已观察、仅作诊断的 `formal-independent-writing-v2` 上，单参考绝对表示率分别为 10.5%/10.5%/13%；16,384 档中文 6%、英文 20%。该诊断不是发布证据，但足以用于终止明显无效的架构搜索。
- 8,192/96/2 完整八轮训练在经过 bank-hit 过滤的 internal selection 上仍只有 38.16% 绝对可用率、63.28% Oracle@32；`candidateEligible:false`。这里的 Oracle 是“已可表示正例上的条件指标”，不能再称为真实写作候选上限。
- 从 8,192 扩到 12,288 没有增加任何真实诊断命中，扩到 16,384 只增加 5/200。继续增加同类短语或重调分类阈值不能填补中文开放组合空间，也不能把 13% 提升到 70%。

因此 `public-phrase-transformer-v1` 的“固定短语库 + 全库分类头”路线被标记为 architecture-blocked。训练入口、publisher 与 v5 verifier 在 `scripts/corpus/autocomplete-v2r-architecture-stop.json` 存在时全部拒绝长训练或发布；其 Worker/ONNX 默认运行时已从生产源码与依赖图移除。当前只保留一份 canonical v4 public 资产且继续 fail-closed，RC 保持 code 10。若以后重启公共神经补全，必须以新的 ADR 定义开放词表或可组合输出空间，并重新建立未观察 v3 holdout；新实现只能替换唯一公共 L3 插槽，不能删除 stop 记录后继续同一矩阵。V2.1 也不因本次实验自动解锁。

## 9. Public V2S 实施方向（2026-07-13）

Public V2S 以 `public-v2s-mkn-v1` 替换失败的固定短语输出空间：中英文独立 Subword Modified Kneser-Ney/Trie 负责可组合候选生成，逻辑回归或 16-hidden INT8 Gate 负责排序与拒答。它不恢复 ONNX、Transformer、Tiny GRU 或通用推理运行时。

实施顺序固定为：先加固公共 Worker 的 256-byte 上下文与原始候选信任边界；再生成 V2S selection、双 tokenizer 和四档 MKN 资产；中英文分区分别从同一 3MiB BPE/Unigram 对照中选择胜者，最多允许一次按语言组合校正；Oracle 通过后才训练两种 Gate；随后依次运行全新 cold/workspace validation，并在唯一候选冻结后一次性消费两套 final。训练器和评测器不得写 public，唯一 publisher 最后原子安装一份 v6 manifest 和一份内容寻址二进制。

发布硬门槛恢复为每套 200 checkpoint 触发 70–84、绝对可用至少 70、50 个 silence 最多误触发 1、mixed 0、双 p90 ≤140ms，并要求合流结果不低于同集 Public-off B0。Public Oracle@8/32 absolute 分别低于 40%/45% 时判定 architecture blocked；Oracle 足够但固定 Gate 加一次 hard-negative 修订仍失败时判定 method blocked。任一停止状态都保持 RC code 10，禁止继续同根因循环训练。

### 9.1 2026-07-13 有界训练结论

固定矩阵与唯一一次逐语言方法修正已执行完毕。最大 `BPE-en + Unigram-zh` 候选使用 24MiB train 前缀，单一二进制 5,735,917B，解析约 21.47ms、生成 p90 约 8.97ms、mixed 0；但在 200 checkpoint development 预检上 Oracle@8 只有 74（37%），Oracle@32 只有 80（40%），英文 Oracle@8 为 31%。考虑到训练规模并非单调提升，逐语言取全部固定档位最好值的理论前沿也只有 Oracle@8 75（37.5%）和 Oracle@32 81（40.5%）；英文 Oracle@8 最好值为 32%，只能刚好达到分语言门槛，总体两项仍低于 40%/45%。

因此 `public-v2s-mkn-v1` 已记录为 `architecture-blocked`。该 development 结果明确标记 `releaseEvidence:false`，不能外推为正式真实写作质量，也不能证明所有本地离线补全技术不可行；它只证明当前 30MiB 语料、≤6MiB 资产、4096-token 双语 Subword MKN/Trie 方案无法形成足够候选空间。按停止合同，不训练 G0/G1、不读取四套 final、不写 public、不解锁 V2.1，RC 继续 code 10。

## 10. 训练工作收口（2026-07-14）

本计划到此停止继续训练。v4、V2R 和 V2S 的实现与证据保留用于复核，但不再作为并列操作手册；语料治理、候选训练、评测口径、证据链、发布顺序和未来重启条件统一收口到 `doc/autocomplete-model-training.md`。

后续维护默认只验证 architecture stop、生产依赖隔离和确定性免费路径。任何新公共模型研究都必须新立 ADR、engine ID、manifest schema 和未观察 holdout，不能删除现有 stop 后继续同一矩阵。
