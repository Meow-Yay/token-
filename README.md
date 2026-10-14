[Uploading README.md…]()
# tokenskip-context

一个把 **TokenSkip 的压缩规则**从 CoT 生成搬到 agent 上下文管理的 skill。可选装、可卸载、三档强度。

> **为什么放在「夏鹤明」文件夹** — TokenSkip 的第一作者是 Heming Xia（夏鹤明，PolyU NLP Group），本 skill 是对他那篇论文的规则移植，与 `PolyU两位老师研究方向调研报告.docx` 同源。

---

## 来源（已核实）

**TokenSkip: Controllable Chain-of-Thought Compression in LLMs**
Heming Xia, Chak Tou Leong, Wenjie Wang, Yongqi Li, Wenjie Li

- arXiv: [2502.12067](https://arxiv.org/abs/2502.12067)（v3，2025-09-16 camera-ready）
- EMNLP 2025 Long Paper；ACL Anthology `2025.emnlp-main.165`
- 代码：<https://github.com/hemingkx/TokenSkip>

注意最后作者 Wenjie Li —— 与调研报告里另一位研究者 Jiashuo Wang 的导师是同一人，两人同属 PolyU NLP Group。

---

## 搬运了什么

TokenSkip 压缩 CoT 时的那行 API 调用：

```python
llm_lingua.compress_prompt(cot_output, rate=compression_ratio,
                           force_tokens=['Step', ':'],
                           force_reserve_digit=True,
                           drop_consecutive=True)
```

其中 `force_tokens` / `force_reserve_digit` 的意思是：**有些单元不能剪**。翻译成 agent 上下文规则就是六类必须逐字保留：

1. 带行号的文件路径（`lib/hooks.ts:347`）
2. 函数 / 类 / 类型签名
3. 错误原文与堆栈行
4. 精确数值（版本号、阈值、配置键、大小、耗时）
5. 决策**及其理由**（"选 X 不选 Y，因为 Z"）
6. 用户意图（带 ref 的短引用；没有 ref 就改写，不要伪造引文）

可丢弃：叙述与进度播报、探索日志与死路（只留一行教训）、重复读取、冗长命令输出、已知状态后的 `git status` / `ls`。

另有「摄入纪律」一节——真实开销通常在读不在写：先 grep 后 read、大文件带 `offset`/`limit`、已转写的图不重读、溢出的抓取结果去 grep 而非重读、批量阅读外派 subagent 只回收结论。

---

## 安装 / 卸载

```bash
node install.mjs              # 装
node install.mjs --status     # 查
node install.mjs --uninstall  # 卸
```

- 技能源目录 = 本文件夹（`SKILL.md` + `agents/openai.yaml`）
- 目标插件根 = `<DSH_HOME>/plugins/dsh-codex-skills`，可用环境变量 `DSH_CODEX_SKILLS_PLUGIN` 覆盖
- 安装会复制 bundle 并在插件的 `catalog.json` 里插入条目；插件的 watcher 监听 `catalog.json`，**无需重启**即生效

---

## 选装的三层实现

| 层 | 机制 | 说明 |
|---|---|---|
| 不自动触发 | `agents/openai.yaml` → `policy.allow_implicit_invocation: false` | 只在点名调用时才跑。对照：`token-smart`、`photographers-eye` 等都是 `true`（会自己跳出来） |
| 可整体装卸 | `install.mjs --uninstall` | 一条命令移除，源文件保留 |
| 档位可选 | `light` / `balanced` / `deep` | 不指定默认 `balanced`，skill 内写明「不得擅自升档」 |

---

## 诚实边界

- **这是提示词侧规则，不改权重。** TokenSkip 的 40%（Qwen2.5-14B-Instruct 在 GSM8K 上从 313 → 181 token，掉点 <0.4%）是 **LoRA 微调**跑出来的，提示词做不到。skill 里白纸黑字写了「不建立任何省 token 百分比」，不要拿它当承诺。
- **skill 不会自动生效。** `allow_implicit_invocation: false` 意味着不点它就不跑。若要每回合强制生效，需另配 AGENTS.md 常驻指令——那是另一件事。
- **交付物、引用、验证不得裁剪。** 压缩的对象是叙述，不是结果。

---

## 文件

```
tokenskip-context/
├── SKILL.md          # skill 正文（改这里即生效，无需重装）
├── agents/openai.yaml # 选装开关 allow_implicit_invocation
├── install.mjs       # 便携安装/卸载脚本
└── README.md         # 本文件
```

SKILL.md 的正文由 `lib/index.js` 直接读磁盘加载，所以改正文不需要重新安装；但**新增 skill 必须写 catalog.json**（插件的文件监听器只响应 `catalog.json`）。
