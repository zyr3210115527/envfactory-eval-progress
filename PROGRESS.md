# 进度总览

## 进行中：base 换成 MiniCPM5-2.6B SFT step3000（08-06 15:45 起）

模型路径 `/user/zhouyiran/experiments/beijing/projects/2372-lhz-minicpm5-sft/662880/model/MiniCPM5-2.6B_job662880_step0003000`，服务名 `minicpm5-2.6b-sft-step3000`。

这是一次**干净的对照**：新 checkpoint 与 step0599 的 chat template 逐字节相同，架构也相同（Llama 42 层 / hidden 2048 / 上下文 131072），所以部署参数、工具解析器、以及此前修好的那些坑全部沿用，差异只在权重。

沿用的修复项（都已确认在位）：上下文 131072、解析器补丁（router 日志有 `minicpm4_xml detect_and_parse patched`）、router `--disable-circuit-breaker`、Toolathlon `--max-tokens 16384 --context-window 114688`、MCPMark 动态 `max_tokens`、postgres 的 `mcp<2` 依赖钉版。

上线前烟雾测试：工具调用 **10/10 解析成功、零泄漏**，思考块正常。启动后 router 前 143 次请求**全部 200、零 400**。

Toolathlon（`runs/sft3000`）与 MCPMark（`results/sft3000`）已同时启动全量。

**早期观察（各判分十几题时）**——两类失败都指向模型侧，不是环境问题：

1. **不自主行动，转而反问用户**。`academic-pdf-report` 探查几步后撞到 `Security violation`，随即输出「你想要哪个会议？需要哪些字段？」等一串问题并停止调用工具，回合直接结束，启动后 11 秒就判了分。
2. **工具名 / 参数名不合 schema**，于是调用被检测器丢弃（表现为「泄漏」）。已抓到三种：参数名多写斜杠 `<param name="/dataset_id">`；工具名拼错 `scolar_search_arxiv`（应为 `scholar`）；给 `filesystem_ls` 传了 schema 里不存在的 `tail`。

下面的总分是**上一个 checkpoint（step0599）**的收官结果，保留作对照。

## step0599 总分（08-06 收官，全集已覆盖）

| 评测 | 任务数 | 通过 | 通过率 |
| --- | --- | --- | --- |
| Toolathlon | 108 | 1 | 0.9% |
| MCPMark | 127 | 2 | 1.6% |
| **合计** | **235** | **3** | **1.28%** |

通过的三题：Toolathlon 的 `train-ticket-plan`，MCPMark 的 `file_context__file_merging` 和 `web_search__birth_of_arvinxu`。

**Toolathlon 全集 108 题的覆盖来自两个独立来源，结论一致：**

- 我这轮（`ref1`）跑到 100 题，通过 1 题（`train-ticket-plan`）；
- 模型负责人 liuhezi 今天跑满 **108 题 × 3 轮 = 324 次尝试**，`pass@1 = 0.31%`、`pass@3 = 0.93%`，**唯一通过的也是 `train-ticket-plan`**（3 轮里过 1 次）。

我缺的 8 题全部依赖 Notion（`notion-find-job`、`notion-hr`、`notion-movies`、`notion-personal-website`、`experiments-recordings`、`oil-price`、`quantitative-financial-analysis`、`task-tracker`），在 liuhezi 那轮里**三轮全部失败**，所以不存在被漏掉的得分点。

> 这 8 题我本地跑不了的原因值得记录：任务初始化要按**硬编码的页面 ID**（`7ded87a0-0805-8287-b4b6-0142fe545458`）从团队 Notion 空间复制模板页。08-06 用浏览器重新授权后，可选的工作空间只有个人那一个（`yiran81955@gmail.com`），模板页不在里面，于是 `duplicate_child_page` 找不到页面、preprocess 直接失败，模型一次请求都没发出（`agent_llm_requests: 0`）。要在本地跑通，需要用持有模板页的账号授权，或把模板页共享给该账号。另外这些任务必须串行——refresh token 一次性、编排器无锁，并行会触发令牌重用把整个授权吊销。

**MCPMark 已跑满团队标准集**（127 题，与基线 508 = 127×k4 逐题对齐）。仓库里另有 `insforge` 和 `supabase` 各 31 题，`.mcp_env` 无凭证、基线也未跑，不计入。

参照系：同一套 Toolathlon 上 kimi-k3 的 pass@1 是 **73.46%**，今天现场复现 4/4；MCPMark 上 `nanbeige4.2-3b` 是 5–8%。

**最后更新：2026-08-06 15:35（北京时间）**

## 给训练侧：行为缺陷清单（基于 55–75 个逐题对照）

拿 k3 通过、我们失败的**同一批任务**做轨迹对照。为避免「被解析泄漏截断」拉低数字，下表是**剔除泄漏中断后**的 55 个任务；不剔除时结论一致，只是更极端。

| 指标 | k3（通过） | step0599（失败） |
| --- | --- | --- |
| 工具调用数 中位数 | 21 | 11 |
| **写操作次数 中位数** | **5** | **1** |
| 首次写入前的探查次数 | 4 | **6** |
| 收尾前回读验证的比例 | 53% | **24%** |
| **没写任何东西就宣称完成** | 2% | **27%** |
| 报错后仍继续行动 | 100% | 91% |
| 用过并行调用的任务 | 99% | **19%** |

**1. 最主要的缺陷是探完不动手。** 探查次数我们反而更多（6 vs 4），但写操作中位数只有 1 次，而 k3 是 5 次。**27% 的失败任务从头到尾没调用过任何写类工具，却宣称完成了。** 这不是「不会查」，是「查完不执行」。

> 这一条更正了 08-06 上午基于单个任务给出的判断。当时我看 `academic-warning` 里模型没查表，就说分水岭是「先探查元数据」；放到 75 个任务上统计，探查根本不是短板。

**2. 宣称完成前不回读验证。** k3 有 53% 的任务在最后一次写入后还会读回自己的产出确认，我们只有 24%。这与「零写入却宣称完成」是同一个根子：缺少对「我到底做了什么」的核对。

**3. 几乎不用并行工具调用。** k3 有 31.5% 的消息一次发多个调用、99% 的任务用过；我们是 8.7% 和 19%。框架已开 `parallel_tool_calls`，这是能力没被用起来，直接影响能在回合预算内做完多少事。

**4. 报错后放弃（次要）。** 91% vs 100%，差距不大但存在。

## 最终结果（08-06 下午）：Toolathlon 1/100，MCPMark 2/127

两个评测都跑完了。

| 评测 | 判分 | 通过 | 参照系 |
| --- | --- | --- | --- |
| Toolathlon `ref1` | 100 | **1** | 今天同机同码：kimi-k3 **4/4**，grok-4.5 **3/5** |
| MCPMark `clean2` | 127 | **2**（1.6%） | nanbeige4.2-3b：8% 和 5%；liuhezi 跑 step0599：1% |

**Toolathlon 修复前后几乎没有差别**：ref1 的失败分布是 70% 结果不达标 / 23% 泄漏中断 / 4% 跑满回合 / 2% 零工具调用，clean1 是 68% / 27% / 2% / 2%，工具调用中位数 10 vs 9。也就是说 `enable_thinking` 和 `context_window` 那两个配置错误是真错，但对分数几乎没有影响——这与 liuhezi 用正确配置跑出 0/98 完全一致。

**MCPMark 的绝对分数本来就低**，参照系是一个 3B 小模型能拿 5–8%，我们大约是它的五分之一。

MCPMark 分服务看，有一整块是环境坏的：

| 服务 | 任务 | 通过 | 硬报错 |
| --- | --- | --- | --- |
| filesystem | 30 | 1 | 2 |
| notion | 28 | 0 | 1 |
| github | 23 | 0 | 2 |
| playwright_webarena | 21 | 0 | 1 |
| **postgres** | **21** | **0** | **21（全挂）** |
| playwright | 4 | 1 | 0 |

**postgres 那 21 个任务模型一次都没碰到题。** 原因是 `postgres-mcp==0.3.0` 的 `mcp` 依赖没钉版本，装到了 **mcp 2.0.0**，而 2.0 移除了 `mcp.server.fastmcp`——postgres_mcp 在 import 阶段就崩，表现为 `Agent execution failed: Connection closed`。已修（钉 `mcp<2`，并绕开镜像里那个按 mcp 2.0 解析过的 pipx 缓存 venv）。

**重跑结果（`pgfix1`）：21 题全部有效判分，0 硬报错（修复前 21/21 全挂），0 通过。** 所以 MCPMark 的分母现在是干净的 127 题、2 通过（1.6%）。

postgres 的失败原因是上午那份行为清单的独立复现——全是「要求的东西根本没建出来」：

```
❌ FAIL: transfer_parts function does not exist
❌ FAIL: The 'theme_analyst' role was not created.
❌ FAIL: RLS is not enabled on table 'lego_sets'.
❌ No index found on payment.customer_id column
❌ relation "persons" does not exist
```

两套完全不同的评测、完全不同的工具集，指向同一个缺陷：**探完不动手**。

另一个保留项：router 端仍有 3% 的请求返回 400（5806 个 200 对 180 个 400），来自 mcpmark 的超长对话，属于真实的上下文耗尽。

> 监控脚本的一个读数错误已修：mcpmark 把判定放在 `execution_result.success`，我的脚本只读顶层 `success`，因此整整一天把 mcpmark 报成 0 通过。实际是 2 通过。

## 08-06 中午：又查出两个我方配置错误，但它们不解释 0 分

有人质疑「一个都不对有点离谱」，于是我把工具链本身当作嫌疑对象重查了一遍。结论分三部分。

### 已排除：环境和评分器都是好的

- **环境完好。** 我把今天任务环境喂给模型的**前几次只读探查响应**和 7 月 30 日 kimi-k3 基线那轮逐字对比：7 条可比对的响应**全部一致，0 条不同**——同一个 BigQuery 数据集、同一份学生成绩 CSV、同一份 `ref.bib`、同一封邮件搜索结果、同一份 memory graph。只读探查的结果模型无法影响，所以这直接证明环境种子数据没有腐坏。
- **评分器代码同一个版本。** 两个仓库 git HEAD 都是 `a3c20a1b`，我的工作区除了几个图标文件没有任何改动。
- **这套 harness 出得了分。** 同一份代码上 kimi-k3 的 pass@1 是 **73.46%**（108 题，三轮，81/105 和 77/106）。

### 查出来的两个我方配置错误（真错，已修）

我用的编排器是**未打补丁的原始版本**，而基线实验用的是打过补丁的。补丁里有两件我完全没有的事：

1. **没有发送 `chat_template_kwargs.enable_thinking=true`。** 参考版注释写明「部分自建 endpoint（如 cctl/sglang）必须带这个，否则会拒答或空 content」。我那份代码里根本不存在发送这个参数的代码路径。
2. **没有设置 `context_window`。** 自建模型不在 `API_MAPPINGS` 里，`get_context_window()` 会兜底返回 **1000000**，于是框架的主动截断**永不触发**，只能等服务端报超长，然后走 `_reset_context_and_history()` **把整段对话历史清空重来**。我的日志里确认发生过：`You requested a total of 135158 tokens: 118774 tokens from the input messages`。也就是说模型做到一半的工作被反复抹掉——这很可能就是「探查几步就宣称完成」这个表象的来源之一。

两项都已修（换用参考版编排器 + `--context-window 114688`），三条轴第一次同时干净的那轮正在跑（exp `ref1`）。

### 但它们不是 0 分的原因

模型负责人 liuhezi 在 08-05 跑的三轮，配置是**正确的**——`context_window=114688` 和 `enable_thinking=true` 都有——结果**仍然是 0/98**，失败分布和我这轮几乎逐项吻合（68.4% vs 68% 结果不达标，29.6% vs 27% 泄漏）。所以我的配置错误是真错，但换成正确配置并不改变结果。

### 判别实验的结论：harness 是好的，0 分是模型的真实表现

今天在**同一台机器、同一套代码、同样这 5 个任务**上跑了两个外部模型作对照（经 LLM Center，沙箱可直连）：

| 模型 | 今天的结果 |
| --- | --- |
| kimi-k3 | **4/4 全通过** |
| grok-4.5 | **3/5 通过** |
| step0599 | **0/5** |

k3 那个 73% 不是陈旧数据——它今天把 `ab-testing`、`academic-warning`、`apply-phd-email`、`cooking-guidance` 全做对了。所以环境、评分器、MCP 工具链都验证可用，**「一个都不对」不是工具链的问题**。

### k3 做对 vs 我们做错：同一个任务的逐步对照

`academic-warning`（要求对比历史成绩找出下滑超 45% 的学生，写 `bad_student.csv` 并写告警日志）：

**k3（通过，45+ 轮实际工作）**

1. 先查元数据再写查询：`INFORMATION_SCHEMA.TABLES` → `INFORMATION_SCHEMA.COLUMNS`，拿到真实表名 `scores_2501…2507` 和列结构；
2. 一轮里**并行**发 2–3 个调用，不是一次一个；
3. **被拒后立刻换法**：建 bucket 被拒就跳过；写日志名 `exam_log` 被拒，立刻改用完整桶名 `exam_log-f11cc696bba4`，成功；
4. **收尾前回读验证**：重新读 `bad_student.csv`、读日志确认写进去了。

**step0599（未通过，6 次调用）**

1. 读了 CSV、列了 dataset、看了 dataset info、列了目录；
2. **一次表都没查**——连表名都没试对，历史成绩从未取到；
3. **没有调用过任何写文件工具**；
4. 直接 `local_claim_done`，收尾语称「已处理数据并记录了告警」。

grok 在这题上栽的是同一个坑：猜表名 `academic_warning."2501"`，报语法错后放弃。k3 与它们的分水岭就是**先查元数据**和**报错后换一条路**。

对训练侧最直接的三条：**动手前先探查 schema/元数据**；**报错不等于终止，要换一条路重试**；**宣称完成前必须回读自己产出的东西**。

> 一条自我更正：我一度拿「轨迹里 `reasoning_content` 为 0 条」当作思考没开的证据，这个证据是错的——k3 基线那轮同样是 0 条，说明这个字段根本不写进 traj_log。真正站得住的证据是代码层面的 diff。

## 08-06 上午：修掉两个我方 bug，Toolathlon 和 MCPMark 同时在跑

机器 `673016`（2×H800，上下文 131072）。今天修掉两个**都会让评测数据作废**的问题，现在两个评测都在干净的服务上重跑（Toolathlon exp `clean1`、MCPMark exp `clean2`）。

**bug 1：SGLang 工具调用解析器的未初始化变量。** `minicpm4_xml_detector.py` 的兜底正则路径里有一行 `has_invalid_param = has_invalid_param`，这个变量只在主路径（XML 解析）的分支里初始化。模型输出的参数值含裸 `&`（shell 命令的 `&&`）时 XML 解析先抛异常，兜底路径一进门就 `NameError` 自爆被吞掉，整块 `<function>` 原样漏进正文，工具不执行、任务终止。**只咬每条消息的第一个坏块**，所以泄漏看起来毫无规律。修复是补一行初始化，用 `PYTHONPATH` 注入 `sitecustomize.py`，不动共享 conda 环境。在线探针泄漏率 **4/6 → 0/10**。详见 [FINDINGS.md](FINDINGS.md)。

> 注：08-06 早上我曾把泄漏改判为「模型输出非法 XML」，那次改判用错了方法（拿解析后的 content 当模型原始输出）。定位到具体代码行后已推翻，昨天的判断才是对的。

**bug 2：`max_tokens` 配错导致全量 400，并且雪崩。** 我给 Toolathlon 传了 `--max-tokens 131072`，等于把整个上下文窗口都要求留给生成，于是**每一个**请求都被 SGLang 以 400 拒绝。更糟的是 sglang router 把 4xx 客户端错误计入 worker 失败并打开熔断，之后连 `hi` 都返回 503——**两个评测同时被一个参数错误拖死**。修复三件事：Toolathlon 改 `--max-tokens 16384`；router 换成 `--disable-circuit-breaker` 重起；MCPMark 的硬编码 `max_tokens=32768` 改为按剩余窗口动态夹取（它的对话会涨到 11 万 token，32768 必然超窗）。

## 结果：Toolathlon 100/108 判分，0 通过；MCPMark 76 判分，0 通过

**核心数字：100 个任务里有 70 个主动调用了 `local_claim_done` 宣称完成，60% 的任务以「task has been completed」之类的话收尾，而实际通过率是 0。模型宣称成功的准确率为零。**

失败原因分布（100 个已判分任务）：

| 占比 | 原因 |
| --- | --- |
| 68% | 工具跑通但结果不达标 |
| 27% | 工具调用泄漏后中断 |
| 2% | 从未成功调用任何工具 |
| 2% | 跑满回合数仍未达标 |

工具调用次数中位数 9 次，70 个任务出现过工具报错。**MCPMark 那 76 个任务零泄漏，同样 0 通过**——这条旁证说明解析已经不是瓶颈。

### 那 27% 的泄漏，91% 是模型侧

34 个泄漏块逐块核对（分布在 27 个任务）：

| 数量 | 归属 | 说明 |
| --- | --- | --- |
| 17 | 模型 | 参数键和 schema 不符：`k8s_kubectl_get` 漏掉必需的 `namespace`；`terminal_run_command` 凭空多出一个 `url`；`google_sheet_list_sheets` 一个参数都不给 |
| 12 | 模型 | 工具名本轮从未成功过，疑似幻觉 |
| 3 | 模型 | CDATA 之后漏写 `</param>` |
| 2 | **未解释** | `local_claim_done` 零参数调用被拒。同样的字符串在 44 个任务里成功 45 次，只在 4 个任务里失败 |

那 2 块占泄漏的 6%、占任务的 2%，且都发生在模型本来就要收工的时刻，不改变任务结果。34 条泄漏里 19 条位于轨迹末尾。

### 最有代表性的一例

`academic-warning`：6 次工具调用**全部成功、零报错**，读了 CSV、列了 BigQuery 数据集、看了 dataset 信息、列了目录，然后直接 `local_claim_done`，声称「已处理数据并记录了告警」——**但它从未执行过一次查询去取历史成绩，也从未调用任何写文件工具**，评分器找不到要求的 `bad_student.csv`。

无解析失败、无 400、上下文远未吃紧、所有工具调用都成功。这是真实的行为缺陷。

## 给训练侧的建议

按证据强度排序：

1. **宣称完成前先自检产物。** 这是最大的一块。模型在 70% 的任务里主动 `claim_done`，准确率 0。它需要在收工前回头验证任务要求的产物是否真的存在（列目录、读回文件、查一次结果），而不是凭「我讲过要做」就宣称做完。
2. **遇到工具报错要重试或换路径。** 70 个任务出现过工具报错，普遍模式是报错之后不重试、直接收尾。
3. **严格按 schema 填参数。** 17 例漏填必需参数或臆造参数名。值得在训练数据里强化「调用前对照 schema 检查必需字段」。
4. **抑制工具名幻觉。** 12 例调用了不存在的工具。
5. **CDATA 场景可靠闭合 `</param>`。** 只有 3 例，优先级最低。

前两条是行为层面的，影响远大于后三条格式层面的。

## 历史：128k 已验证可用，服务在 2 卡上重新起好了

新机器 `668307`（**2×H800**，不是 8 卡）。改用 2 卡是有依据的：昨天 8 个副本的日志显示，解码批次并发为 1 的采样有 60,927 次、为 2 的只有 4,530 次，**全场峰值只到 4**——8 张卡里至少 6 张在陪跑，瓶颈在 E2B 沙箱不在 GPU。单卡 KV 容量 144 万 token，装得下 11 条满长度的 128k 对话，2 卡应付 workers=6 绰绰有余。

大海捞针验证**13 个探针全过**：4k / 32k / 64k / 96k / 120k 五个长度，每个长度在 10%、50%、90% 三个深度都准确取回了埋点数字，120k 的预填充只要 4-6 秒。**至此可以确认：模型 128k 是真的，昨天的 64k 是我部署时传错参数。**

新的公网地址：`http://modelbest-svc-bj.modelbest.co/beijing/job/668307/expose/v1`（连续 3 次探测 200，两个评测的配置都已指过来）。端点自检 4 项全过，工具调用能正确解析成 `tool_calls`。

## 一句话状态

第一批结果出来了：Toolathlon 两轮各 108 个任务、MCPMark filesystem 30 个任务，**全部 0 分**。但这批数据**不能直接当结论用**——里面有两个我这边造成的污染源：工具调用解析故障（毁掉 36% 的任务）和上下文只开了 64k（模型其实是 128k，影响另外 27%）。剔除这两者后仍有 51 个干净任务，模型在这些任务上还是 0 分。完整分析见 [FINDINGS.md](FINDINGS.md)，实时状态见 [STATUS.md](STATUS.md)。

## 服务信息

| 项 | 值 |
| --- | --- |
| checkpoint | `/user/zhouyiran/projects/envfactory_v3_hf/step0599` |
| 对外模型名 | `envfactory-v3-step0599` |
| devspace | `664834`（8×H800） |
| 推理框架 | SGLang，8 个单卡副本 + router |
| 网关端口 | `8790` |
| 公网地址 | `http://modelbest-svc-bj.modelbest.co/beijing/job/664834/expose/v1` |
| 日志目录 | `/user/zhouyiran/logs/envfactory-v3-step0599_20260804_101844` |

模型是 2.4B 的 Llama 结构，42 层，GQA 2 个 KV head。工具调用是 MiniCPM 风格的 XML，所以 SGLang 起的时候带了 `--tool-call-parser minicpm4_xml --reasoning-parser qwen3`。

**上下文长度（已更正）**：模型实际是 **128k**。第一次部署时 `config.json` 里写的是 `max_position_embeddings: 65536`，我照这个配了 64k，昨天 8 个副本的日志确认全部 `context_len=65536`——**这堵墙是配置错误，不是模型限制**。现在 config 已 patch 成 131072（`patch_mpe.py`，备份在 `.bak_mpe`）。部署脚本改用 `--context-length 131072`，并保留 `SGLANG_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN=1`——这个字段被还原过一次，留着这个环境变量以防它再变回去时服务起不来。

## 已完成

- **探查 checkpoint**：确认了架构、tokenizer、chat template，定下部署参数。
- **部署模型服务**：8 个副本分别占 GPU 0-7（端口 8000-8007），router 挂在 8790。从启动到网关就绪约 70 秒。
- **打开端口暴露**：这一步原本卡住了——`expose.enabled` 是 `False`，公网地址一直 404。后来发现 `cctl devspace expose 664834 --port 8790` 可以直接开，不用去控制台点。开完之后公网连续 6 次探测都是 200。
- **端点自检全部通过**（本地 8790 和公网地址各跑一遍，结果一致）：
  - 模型正确出现在 `/v1/models`
  - 普通对话 `finish_reason` 是 `stop`，没有泄漏 `<|im_end|>`
  - 工具调用被正确解析成 `tool_calls` 对象，函数名和 JSON 参数都对
  - 带工具结果的第二轮能正常收尾

一个无关紧要的观察：模型自我介绍时说自己是「Google 训练的大语言模型」，是训练数据里的痕迹，不影响评测。

## 进行中

**Toolathlon 全量（`full1`）** — 108 个任务 × 3 轮，workers=6。18:45 启动。
**MCPMark filesystem（`smoke_fs`）** — 30 题，k=1。

实时进度看 [STATUS.md](STATUS.md)。

### 冒烟阶段踩的坑（都已解决）

1. **venv 是空的**。共享盘上原来那个 venv 里什么都没有，连 pip 都没有，之前那次安装没落盘。重建在 `/user/zhouyiran/.venv-eval`。
2. **e2b 版本不能用最新的**。直接装 2.37 会拒绝这里的 API key——腾讯自建版的 key 是 `ark_` 开头，新版 SDK 强制校验 `e2b_` 前缀并直接抛 `AuthenticationException`。对齐成 liuhezi 在用的 `e2b==2.12.1` + `e2b-code-interpreter==2.6.0` 后正常。
3. **`--max-tokens` 不能照抄样例**。样例里的 65536 是给上下文更大的模型用的；当时服务只开了 64k（**这本身就是我配错的，见上文**），给补全预留 32768 之后，输入涨到 34k 就会被端点以 400 拒绝。同时 `--max-steps` 给小了（30），框架「上下文超长就重置」的兜底还没来得及生效步数就用完了。改成 `--max-tokens 8192 --max-steps 100` 之后，任务能跑到评分环节。

### 冒烟结果

改对参数后（`smoke3`）两个任务都跑完并进入评分，**0/2 通过**，失败原因是真实的能力问题而不是环境问题：

- `cooking-guidance`：模型输出的 JSON 不合法，评分器解析时抛 `JSONDecodeError`。
- `add-bibtex`：内容对不上（作者字段值不匹配）。

MCPMark filesystem 前几题也都是实打实的失败：该生成的报告文件没生成、该归档的文件没动、金额算错。

一个值得注意的现象：模型的思考链很长且反复推翻自己，同一个判断会绕好几圈。`add-bibtex` 中途两次撞到当时设的 64k 上限，靠框架重置才继续下去。这既吃 step 预算也吃分——不过这堵墙是配置错误造成的，开到 128k 后应该会缓解。

## 已解决：MCPMark 的 `github_state/`

你从 Google Drive 下的那个包已经就位。补充说明：

- Drive 那份只有 3 个仓库快照（`anthropics-claude-code`、`hiyouga-EasyR1`、`openai-harmony`），但任务实际引用 6 个模板。
- 缺的 3 个（`codecrafters-io-build-your-own-x`、`missing-semester-missing-semester`、`zjwu0522-mcpmark-cicd`）代码里有 CDN 兜底地址。CDN 在 devspace 上被白名单代理挡了，但从你电脑能访问，已经下好补齐。
- 现在 `/user/zhouyiran/projects/mcpmark/github_state/` 下 6 个模板齐全，每个都有 `issues.json`、`pulls.json`、`meta.json`、`repo/`。
- `GITHUB_TOKENS` 和 `GITHUB_EVAL_ORG` 之前就填好了，所以 GitHub 那组任务现在可以跑。

另外注意文档里写的 `unzip github_state.zip -d ./github_state` 会多套一层目录，因为压缩包本身已经带了 `github_state/`，要解到项目根。

## 待办 / 风险

- **等 8 卡机器**：`664834` 被 `IdleAutoRelease` 回收，`667838` 已 Killed，现在等 **`668042`**（8×H800，HIGH 优先级，同一个 snapshot 镜像）。到位后按新配置（128k 上下文）重新部署，先跑 `verify_long_context.py` 验证长上下文真能用，再复现并修解析器，然后重跑评测。
- Toolathlon 全量跑完后出总分；MCPMark 等 filesystem 跑完再铺开到全部服务。
- devspace 有生命周期，长时间评测要留意别在中途被回收。
- 自动状态更新依赖你电脑上的 Teleport 证书，证书过期后 `STATUS.md` 会明确写"连不上"，那就说明需要在电脑上重新登录一次。

## 配置改动记录

两个评测的模型地址都已指向新服务：

- `Toolathlon/scripts/e2b.env` → `TOOLATHLON_OPENAI_BASE_URL` 等四项
- `mcpmark/.mcp_env` → `OPENAI_BASE_URL`、`OPENAI_API_BASE`、`MCPMARK_MODEL`
