# 金字塔感知检索协议 · 验证流程说明(Validation Pipeline)

> 这份文档回答一个问题:**"你怎么知道它有效?"**
> 完整证据链从线上 agent 轨迹出发,经缺陷诊断、协议干预、评测集构建、离线消融,最后回到线上 bench 复核,并对每一步的效度威胁做诚实清算。
>
> 配套文档:`PYRAMID_RETRIEVAL_SPEC.md`(协议设计与硬门槛)、`eval/`(评测资产与全部原始结果)。
> fork 分支:`cjmp-pyramid-retrieval`(提交 7f46d93)。

## 0. 一页结论

| 证据层 | 结论 | 关键数字 | 强度 |
|---|---|---|---|
| 引擎层(离线) | 单元化救召回,卡片救负载,两杠杆各自可归因 | recall(真实查询)28.1%→81.3%;离线负载中位 6758→672 字符 | 强(消融可复现) |
| 单点层(在线) | 新协议被 agent 正常使用,漏斗成立,负载有界 | 在线单次 search 中位 14.2K→3.4K(-76%),上限 34.8K→4.8K | 中 |
| 端到端层(bench) | 检索不是总 token 的主杠杆 | judge 83.9(臂2 为 84.4,门槛 82.4 过);token 双门槛(-30%/-50%)未达 | 弱/否定性 |
| 方法论层 | 六步流水线可复用于任何"文档→MCP 检索"改造 | 全程显式标注测量通道,防止跨通道数字误比 | — |

**元结论**:总 token ≈ 轮数 × 上下文(92% 是缓存读,检索注入仅占总额 ~0.26%)。检索优化的收益真实但被"轮数×上下文"稀释;要降总 token,合同治理(冻结规则/批修规则)优先级高于检索优化。这条否定性结论同样是本验证流程的产出。

## 1. 验证流程总览

```
① 轨迹考古        ② 缺陷诊断         ③ 协议干预
  线上 agent 如何查 → 旧引擎为什么差  →  单元化 + 摘要卡片(两个可消融的杠杆)
                                              │
⑥ 线上复核+效度检查 ◄──── ⑤ 离线对比+消融 ◄──── ④ 评测集构建
  (bench 7 任务 +     (5 组配置,归因     (114 条,real/syn 分列,
   在线负载解剖)        到每个杠杆)        三重命中判定)
```

方法论纪律:**每一个数字都显式标注测量通道**——在线轨迹(`bench-run/opencode.stdout.jsonl` 的 tool_use 事件)、离线 CLI JSON、bench metadata、轨迹 step-finish 重算——四个通道不可互相冒充。本次实验曾因 metadata 在 continuation 场景漏统计(最多低报 8 倍)而作废过一版 token 结论,通道纪律由此而来。

## 2. 第①步 轨迹考古:线上 agent 到底怎么查

**输入**:bench 各任务工程的 `bench-run/opencode.stdout.jsonl`(opencode 轨迹)。
**方法**:提取全部 `tool_use` 事件中 `part.tool` 为 `cjmp-docs_search_docs` / `cjmp-docs_read_section` 的 `state.input`(查询)与 `state.output`(返回)。

**产出(优化臂,7 个 ui-interaction 任务,Android)**:

| 任务 | search 次数 | read_section 次数 |
|---|---|---|
| calculator | 7 | 6 |
| entry-threshold-task | 3 | 7 |
| flashcard-quiz | 6 | 6 |
| meituan-prototype | 12 | 22 |
| menu-interaction | 7 | 4 |
| rider-plan-hub | 4 | 6 |
| shopping-cart | 6 | 13 |
| **合计** | **45** | **64** |

**发现 1——查询语言形态**:**45/45 条 search 查询全部是英文 API 标识符关键词包,0 条含中文**——尽管任务语境、AGENTS.md、文档正文全部是中文。典型形态:

```
"Grid GridItem layout columns rows"
"PanGesture onActionEnd onActionUpdate GestureEvent offsetX offsetY"
"padding margin border EdgeMargin EdgePadding object literal syntax"
```

含义:agent 检索时用的是 API 面标识符(仓颉/CJ-UI 标识符本就是拉丁字母,且 CJ-UI 刻意对齐 ArkUI 命名)。这对度量选型有决定性影响——生产环境 GLM proxy(LiteLLM)只有 chat 模型、没有 embedding,检索实际是纯 FTS/BM25;标识符关键词包恰好是 BM25 最强的查询形态(无分词歧义、token 精确匹配)。**评测集必须复刻这个分布,而不是想象中的"中文自然语言提问"**(见第⑤步效度检查)。

**发现 2——检索频率**:每任务 3~12 次 search,不是高频动作。单任务检索注入总量级在数万字符——负载优化空间在"单次单价",不在"次数"。

**产出(臂2,旧引擎基线,同通道)**:34 次检索,单次返回负载中位 **14.2K 字符**,P90 23K,最大 34.8K;返回内容常为整个 L2 case 源文件全文。这是后文一切"-76%"比较的在线基线。

## 3. 第②步 缺陷诊断:旧引擎为什么差

两个结构性缺陷(详见 SPEC §1):

1. **索引粒度**:金字塔被索引成 3MB 单页 bundle,1732 个 chunk 共享一个 URL。search 的 MarkdownAssemblyStrategy 返回"命中簇+父块+1 前兄弟+3 子块+2 后兄弟"拼装全文——一次查询拖回整个 case 文件,且无法按单元去重。
2. **返回形态**:只有全文一档,没有"卡片→小节→全文"的渐进披露,agent 每次检索都为 5% 的信息付 100% 的负载。

离线复核(第⑤步的 baseline 组):recall(总)63.2%,**真实查询仅 28.1%**——引擎层确实烂,不只是贵。

## 4. 第③步 协议干预:两个可分别消融的杠杆

| 杠杆 | 改动 | 作用对象 |
|---|---|---|
| **A. 检索单元化** | 金字塔 → 265 个独立检索单元(1 个 L1 索引 + 10 个 L2 case 文件 + 254 个 L3 api 文件),每单元独立 URL,front-matter 携带 `layer/topic/api_names/keywords`(`build_corpus.py`) | 召回 |
| **B. 摘要卡片 + 渐进披露** | `search_docs` 新增 `detail: "cards"`:每单元返回一张消化卡(`## API 摘要` 标题锚定切片,无摘要则 700 字符摘录兜底),检索单元去重 + 2× 超采样防同单元多 chunk 浪费槽位;新增 `read_section(library, path, section?)` 按锚点拉单小节或整单元 | 负载 |

三级检索阶梯(与金字塔 L1→L2→L3 同构):

```
search_docs(detail=cards) ──► 摘要卡片 ~0.5-0.7K 字符/单元
    │ 90% 场景到此为止
    ▼ 点名要
read_section(..., section) ──► 单个小节全文 ~1-3K 字符
    │ 需要整单元上下文时
    ▼
read_section(...)           ──► 整个检索单元全文
```

设计原则:**每个 token 都被点名要,没有顺带塞进来的。**

## 5. 第④步 评测集构建

`eval/eval_set.jsonl`,**114 条 = 32 real + 82 syn**:

- **real(32 条)**:手写,刻意复刻第①步观测到的线上关键词包形态(如 `"Grid GridItem layout grid columns rows"`)——不是想象的中文问句;
- **syn(82 条)**:从单元 front-matter 的 `api_names`/topic 自动拼接(如 `"information-display-loadingprogress information-display-progress enableLoading"`)。syn 自带答案 token,天然乐观,**必须与 real 分列统计**;
- **中文查询 8 条**:作为自然语言形态的底线覆盖(见第⑦步效度威胁)。

**golden 与命中判定**(三重规则,新旧 store 公平,`run_eval.py:67`):结果 URL 含 golden 文件路径 / 内容含金字塔 case 示例代码预埋的 `.id("case-<anchor>")` 埋点 / 含 golden 小节标题文本。

**指标**:recall@k(总 + 按 src 分列);payload = Σ(结果 content + url) 字符数,即"一次 search 往上下文灌多少字"。

**诚实声明**:baseline 按当时默认 limit=3 跑,fork 定稿为 cards@5,非严格同 limit;但 fork 槽位更多仍便宜 10 倍,方向上对结论更有利。

## 6. 第⑤步 离线对比 + 消融(五组)

| # | 配置 | 干预 | recall 总 | recall(real) | payload 中位 | payload P90 |
|---|---|---|---|---|---|---|
| 1 | baseline_old_store(旧 store+旧 search,limit3) | 无 | 63.2% | **28.1%** | 6758 | 13420 |
| 2 | ablation_unitized(仅单元化,全文返回) | A | 93.9% | 78.1% | 5429 | 10663 |
| 3 | fork_full(单元化,全文返回) | A | 91.2% | 68.8% | 5420 | 10348 |
| 4 | fork_cards@3 | A+B | 89.5% | 62.5% | **405** | 448 |
| 5 | **fork_cards@5(定稿,+2×超采样)** | A+B | **94.7%** | **81.3%** | **672** | 742 |

**归因读法**:

- **召回的功劳归单元化**:28.1%→78.1%(组1→组2),只做了杠杆 A。旧 store 一个 URL 占满 5 个槽位,单元化后同查询能命中多个不同单元。
- **负载的功劳归卡片**:5429→672(组2→组5,-88%),只差杠杆 B。
- **卡片在 limit3 会回撤召回**(real 62.5%):压缩后区分度下降,同单元去重+limit5+2×超采样把它拉回 81.3%——这就是定稿 cards@5 的原因。
- 组2 vs 组3(93.9% vs 91.2%):fork store 含 5 库,跨库竞争轻微拉低 cj-ui-docs 的命中,代价可接受。

**口径注意**:此处的 672 是**离线 CLI JSON 通道**(卡片字符串之和),与在线 MCP 格式化文本不是同一通道——偏差解剖见下一步。

## 7. 第⑥步 线上 bench 复核

**设置**:7 个 ui-interaction 任务,Android 模拟器(Pixel_8a),执行合同 v3,judge=Qwen3.7-Plus,单 seed。

### 7.1 引擎行为在线表现(轨迹通道)

| 指标 | 臂2(旧引擎) | 优化臂(新协议) | 变化 |
|---|---|---|---|
| search 单次负载中位 | 14.2K 字符 | **3404** | **-76%** |
| search P90 / 最大 | 23K / 34.8K | 4335 / **4784** | 上限有界 |
| read_section 单次中位 | —(无此工具) | 1118 | — |
| read_section 均值 / 最大 | — | 4343 / 34592 | 整单元深读按需发生(第三级阶梯) |
| campaign 检索总注入 | — | 423.7K 字符(≈0.12M token) | 占 46.16M 总消耗 **~0.26%** |

### 7.2 离线 672 vs 在线 3404 的偏差解剖(重要)

离线预测的 672 字符/次,在线实测 3404。差额来自三个可指认的来源,不是召回或协议失效:

1. **MCP 文本格式化开销**:每张卡带 `[n] unitPath · layer · topic` 卡头 + `full content: read_section(...)` 指针页脚(~100 字符/卡);
2. **L1 索引卡**:几乎所有查询都会命中 `index.md`,它没有 `## API 摘要`,回退到 700 字符 TOC 摘录——这张胖卡平均贡献 ~1/3 负载;
3. **通道差**:离线算卡片字符串,在线算完整工具响应文本。

已知瑕疵(待修,不影响结论):L1 卡的摘录把 store 侧 front-matter 头带进了卡片,渲染成 `---\nundefined\n---` 开头(语料文件本身干净,是 store chunk 的 front-matter 泄漏)。修法:L1 索引也应生成摘要式消化块,且摘录应跳过 front-matter。

### 7.3 端到端结果

| 维度 | 优化臂 | 臂2 基线 | 门槛(Q12) | 判定 |
|---|---|---|---|---|
| judge 均分 | **83.9** | 84.4 | ≥82.4 | ✅(差 0.5,单 seed 内噪声) |
| total token 均值 | 6.59M(轨迹精确口径) | 4.10M | ≤2.87M / ≤2.05M | ❌ |

任务级:fresh token 4/7 任务较臂2 降 35%~52%;**meituan 反例 +81%**(528 轮,为 calculator 的 3.5 倍;read 负载 503K 字符)——迭代密集型任务的轮数是比检索大一个量级的变量,这正是元结论的直接证据。

## 8. 效度威胁清单(诚实清算)

1. **查询语言分布依赖模型行为**:45/45 英文关键词包是 GLM-5.2 的当前行为;换更"爱说人话"的模型,中文自然语言查询对 BM25 分词是未测区(评测集 8 条中文仅是底线,防这个风险应扩到 ~1/3 再跑一轮);
2. **离线评测无 agent 在环**:测引擎质量,不测"agent 会不会用好"(query 构造、何时停手、read_section 跟进时机);
3. **单 seed**:7 任务端到端方差 ±28%,83.9 vs 84.4 不作强结论;离线 114 条则样本足够;
4. **消融 limit 不齐**:baseline=3 vs 定稿=5(已声明,方向有利);
5. **metadata token 陷阱**:bench metadata 在 continuation 场景系统性漏统计(最多低报 8 倍),全部 token 结论以轨迹 step-finish 事件逐轮重算为准;
6. **judge 抖动**:shopping-cart 首评失败、重评 90,同一产物两次评分差 >40 分——judge 分数只看 campaign 均值,不看单点。

## 9. 结论

1. **引擎层(强)**:两个杠杆各自可归因、消融可复现——单元化把真实查询召回从 28.1% 拉到 78.1%,卡片把负载压掉 ~90%(离线口径)/ 76%(在线口径),上限从 34.8K 压到 4.8K(有界性对 agent 上下文预算比均值更重要)。
2. **单点层(中)**:新协议在线被 agent 正常使用——search→read_section 漏斗成立(45→64 次,深读按需发生),无滥用、无空转。
3. **端到端层(否定性)**:检索负载仅占总消耗 ~0.26%,检索优化**不是**总 token 的主杠杆;总 token ≈ 轮数 × 上下文,合同治理(一次全过即冻结、批修再建)和轮数治理才是。Q12 的 token 门槛未达,判定如实记为未通过。
4. **方法论层**:六步流水线(轨迹考古→诊断→干预→评测集→消融→线上复核+效度清算)本身是可移植资产——任何"自有文档 → MCP 检索"的改造都可以照此验证,且每步显式标注测量通道、每条结论标注证据强度。

## 10. 复现指南

```bash
cd /Users/user/Desktop/project/docs-mcp-server

# ① 语料:金字塔 → 265 检索单元(带 front-matter/digest)
python3 docs-cjmp/build_corpus.py

# ② store:5 库正式 store(cj-ui-docs 1304 chunks)
node dist/index.js scrape docs-cjmp/corpus-scrape.yaml

# ③ 离线评测(五组中的定稿组;换 --store/--limit/--detail 复现其余组)
node dist/index.js search cj-ui-docs "<query>" --store-path <store> --detail cards --limit 5 --output json   # 单查询
python3 docs-cjmp/eval/run_eval.py --store <store> --detail cards --limit 5 --out docs-cjmp/eval/rerun.jsonl  # 全量

# ④ 轨迹考古:线上查询分布/负载(对任意 bench 任务工程)
python3 - <<'EOF'
import json, glob
for f in glob.glob('<task>/workspace/<project>/bench-run/opencode.stdout.jsonl'):
    for line in open(f):
        if '"cjmp-docs_' not in line: continue
        ev = json.loads(line)
        if ev.get('type') == 'tool_use':
            p = ev['part']; print(p['tool'], p['state']['input'], len(p['state'].get('output','')))
EOF
```

—— 验证资产索引:`PYRAMID_RETRIEVAL_SPEC.md`(设计)、`build_corpus.py` + `corpus-scrape.yaml`(杠杆 A)、`src/tools/SearchTool.ts` + `src/tools/ReadSectionTool.ts`(杠杆 B)、`eval/eval_set.jsonl`(评测集)、`eval/*.summary.json`(五组原始结果)。
