# LLM 在 CNC 全流程中的应用分析

> 最后更新：2025-05-15

---

## 全景图

```
CAD ────────── CAM ────────── G-code ──────── 控制器 ──────── 加工
  │              │               │               │              │
  │ ①自然语言    │ ③参数推荐      │ ⑤直接生成     │              │ ⑦对话式
  │   生成设计    │ ④刀具路径      │   G-code      │ ⑥(传统ML    │   故障诊断
  │ ②DFM检查    │   策略选择      │               │   更适合)    │ ⑧预测维护
  │              │               │               │              │ ⑨质量诊断
```

---

## 1. CAD 阶段 — "告诉 AI 你想要什么"

### ① 自然语言生成设计（Text-to-CAD）

2025 年最火的 AI + 制造方向之一。[Zoo.dev 的 Zookeeper](https://zoo.dev/zookeeper) 已提供开源 text-to-CAD；[arXiv Text-to-CadQuery 论文](https://arxiv.org/abs/2505.06507) 用 LLM 直接生成 CadQuery（Python 参数化建模代码）。

实际体验分三个层次：

| 层次 | 输入 | 输出 | 成熟度 |
|------|------|------|--------|
| L1 简单形状 | "一个 50×30×5mm 的矩形，中间有个直径 10mm 的孔" | 参数化 CAD 模型 | **可用**，LLM 擅长这类结构化描述 |
| L2 组合特征 | "一个手机支架，可以横放和竖放，厚度不超过 5mm" | 需要多轮对话 + 约束求解 | **部分可用**，复杂几何容易出错 |
| L3 复杂设计 | "帮我设计一个空气动力学外壳" | 曲面、自由形状 | **不可用**，超出当前 LLM 能力 |

**关键洞察：** LLM 不直接"画图"，而是**生成 CAD 脚本**（CadQuery/OpenSCAD 代码），然后由 CAD 引擎渲染。这比直接生成几何数据可靠得多。

```
用户："做一个直径 30mm、厚 3mm 的齿轮，17 齿，模数 1.5"
  ↓
LLM → 生成 CadQuery Python 代码:
  ↓
import cadquery as cq
gear = (cq.Workplane("XY")
    .circle(15)        # 齿顶圆
    .extrude(3)        # 厚度
    .polarArray(radius=13.5, startAngle=0, angle=360, count=17)
    .circle(1.5)       # 齿根
    .cutThruAll())
  ↓
CAD 引擎执行 → 生成 STEP/STL 文件
```

### ② DFM 检查（Design for Manufacturing）

LLM 可以审查设计是否可加工：

- "这个内角没有圆角，铣刀过不去" → 自动添加 fillet
- "这个壁厚 0.3mm，切削时会振断" → 建议加厚
- "这个深宽比 8:1，需要加长刀具" → 建议修改设计或更换刀具
- "这个特征需要 5 轴才能加工，你的机器是 3 轴" → 标注不可加工区域

这本质是**用 LLM 的推理能力做设计规则检查**，把工程师的经验编码为 prompt + 知识库。

---

## 2. CAM 阶段 — "最核心的 AI 应用场景"

这是 LLM **最有价值但也最被低估**的应用场景。

### ③ 切削参数智能推荐

当前痛点：用户需要同时理解材料力学、刀具几何、切削动力学才能正确设置参数。

LLM 方案：**自然语言对话 + 知识库 RAG**

```
用户："我想用 6mm 平底刀切 6061 铝合金"
  ↓
AI: "推荐参数如下：
     主轴转速: 10,000 RPM
     进给率: 600 mm/min (单刃切屑负载 0.06mm)
     最大切深: 1.5mm (不超过刀具直径的 25%)
     步距: 3mm (刀具直径的 50%)

     注意：你的主轴最高转速是多少？如果低于 10000，
     需要相应降低进给率以保持切屑负载。"
```

参考 [CloudNC 的实践](https://www.cloudnc.com/news-room/new-cutting-parameters-ai-solution)，他们用 AI 自动生成不同刀具路径的优化切削参数 — 不同刀路段可能需要不同参数。

技术实现路径：

```
┌─────────────────────────────────────┐
│           RAG 知识库                 │
│  ┌───────────────────────────────┐  │
│  │ 材料数据库（300+ 材料切削性能） │  │
│  │ 刀具数据库（形状/涂层/几何参数）│  │
│  │ 经验公式库（Taylor、Kienzle）  │  │
│  │ 社区加工记录（实际参数+结果）   │  │
│  └───────────────────────────────┘  │
│                 ↓                    │
│  LLM (推理 + 整合 + 自然语言解释)    │
│                 ↓                    │
│  参数推荐 + 理由说明 + 风险提示       │
└─────────────────────────────────────┘
```

### ④ 刀具路径策略选择

LLM 可以分析几何特征，推荐最适合的加工策略：

```
用户上传一个 STEP 文件（一个有凹槽和凸台的零件）
  ↓
AI 分析几何特征:
  - 识别到 3 个 pocket（凹槽），深度 5mm
  - 识别到 2 个 boss（凸台），需要轮廓加工
  - 识别到 4 个 hole（孔），直径 4mm
  - 整体尺寸: 80×60×12mm
  ↓
AI 推荐策略:
  1. Face: 平面铣顶面 (Ø6 平底刀, 一步到位)
  2. Pocket × 3: 自适应粗加工 + 精加工 (Ø4 平底刀)
  3. Profile: 轮廓切外形 (Ø6 平底刀)
  4. Drill: 钻 4 孔 (Ø4 钻头)

  总预估时间: 12 分钟
```

[DELMIA/Dassault](https://blog.3ds.com/brands/delmia/revolutionizing-machining-operations-with-artificial-intelligence/) 已在做 AI 驱动的刀具路径提案系统；[MDPI 论文](https://www.mdpi.com/2075-1702/14/1/89) 用 CNN 直接从 CAD 模型生成刀具路径。

---

## 3. G-code 阶段 — "LLM 能直接生成，但有风险"

### ⑤ LLM 直接生成 G-code

[ResearchGate 研究](https://www.researchgate.net/publication/397562415) 比较了 ChatGPT-3.5 vs 4o 生成 G-code 的能力；[i4valley](https://i4valley.com/resources/leveraging-llm-for-complex-g-code-generation-in-cnc-machinery/) 也在探索这个方向。

#### 可行性分析

| 场景 | LLM 能力 | 风险 |
|------|----------|------|
| 简单 2D 轮廓 | **可行** — G-code 结构简单、模式固定 | 低 |
| 挖槽 + 分层 | **部分可行** — 需要理解切深分层逻辑 | 中 |
| 3D 曲面 | **不可行** — 数据量大，LLM 无法精确计算数万行坐标 | 高 |
| 多刀切换 | **部分可行** — 需要理解 T/M 指令 | 中 |
| 复杂操作（螺纹、攻丝） | **高风险** — 一个参数错误就报废 | 很高 |

**核心问题：G-code 是安全关键的。** 一行错误就可能导致撞机、断刀、甚至人身伤害。LLM 的幻觉（hallucination）在这个场景下是**不可接受**的。

#### 更安全的做法：LLM 不直接生成 G-code

```
LLM 生成高层加工意图（Machining Intent）
  ↓
确定性 CAM 引擎编译为 G-code
  ↓
LLM 审查 G-code（检查明显的错误模式）
```

例如：

```json
{
  "operations": [
    {
      "type": "pocket",
      "geometry": "pocket_1",
      "tool": "flat_endmill_6mm",
      "strategy": "adaptive",
      "depth_per_pass": 1.5,
      "finish_passes": 1
    },
    {
      "type": "drill",
      "geometry": "hole_pattern",
      "tool": "drill_4mm",
      "depth": 12,
      "peck_depth": 3
    }
  ]
}
```

CAM 引擎负责**确定性计算**（路径规划、碰撞检测），LLM 负责**高层决策**（选什么策略、用什么刀）。

---

## 4. 控制器阶段 — "LLM 不太适合，传统 ML 更好"

### ⑥ 为什么不适合 LLM

控制器需要**硬实时**（1ms 级别）的运动控制 — 插补计算、脉冲生成、加减速规划。这些任务：
- 对延迟零容忍（LLM 推理至少 100ms+）
- 需要数学确定性（LLM 有随机性）
- 需要可证明的安全保证（LLM 无法提供）

**传统 ML/DL 更适合的方向：**

| 应用 | 方法 | 说明 |
|------|------|------|
| 颤振检测 | LSTM / 时序分析 | 分析主轴振动信号，实时检测颤振 |
| 自适应进给 | Fuzzy logic + 神经网络 | 根据切削力实时调整进给率 |
| 轨迹优化 | 遗传算法 / 粒子群 | 优化加减速曲线 |

但 LLM 可以在**控制器之外的"慢环"**发挥作用：
- 分析历史加工日志，给出优化建议
- 解释控制器报错信息（"GRBL error:8 是什么意思？我该怎么修？"）
- 生成 GRBL/FluidNC 配置文件

---

## 5. 加工阶段 — "对话式助手 + 预测诊断"

### ⑦ 对话式故障诊断

当前最接近商业化的 LLM + CNC 应用场景。

[ChatCNC](https://www.researchgate.net/publication/389894040_ChatCNC_Conversational_machine_monitoring_via_large_language_model_and_real-time_data_retrieval_augmented_generation) 用 RAG 让 LLM 与实时机器数据对话；[SprutCAM X](https://www.aerospacemanufacturinganddesign.com/article/ai-virtual-assistant-for-cnc-tasks/) 已集成 ChatGPT 做虚拟助手；[ARUM](https://www.microsoft.com/en/customers/story/26357-arum-azure-ai-search) 让操作工通过自然语言控制加工中心。

实际场景：

```
用户: "加工出来的表面有波纹，什么原因？"
  ↓
AI: 结合当前加工参数分析:
  "最可能的原因是颤振（Chatter），因为你当前:
   - 切深 2mm 对于 6mm 刀来说偏高（33% 刀径）
   - 进给率 800mm/min 对应切屑负载仅 0.04mm（偏低 → 摩擦）

   建议尝试:
   1. 降低切深到 1mm
   2. 提高进给到 1000mm/min（切屑负载 0.05mm）
   3. 如果仍有波纹，试试改变转速到 12000 RPM（避开共振频率）"
```

### ⑧ 预测性维护

LLM 不是直接做预测（传统 ML 更适合），而是**解释和整合**预测结果：

```
传感器数据 → LSTM 模型 → "主轴轴承预计 48 小时后需要更换"（置信度 85%）
                                    ↓
                    LLM → 生成可读报告:
                    "根据振动频谱分析，主轴轴承出现早期疲劳特征。
                     建议在完成当前批次后停机检查。
                     备件信息: NSK 6205-2RS，库存还有 3 个。
                     更换教程: [链接]"
```

[多传感器融合](https://pmc.ncbi.nlm.nih.gov/articles/PMC11054666/) + [RoughLSTM 振动分析](https://www.mdpi.com/2076-3417/15/6/3179) 是当前研究热点。

### ⑨ 加工质量诊断

```
用户拍一张加工件的照片
  ↓
多模态 LLM（视觉 + 语言）:
  "检测到以下问题:
   1. 表面有明显的刀纹（tool marks）→ 建议减小步距到 30%
   2. 内角有过切（overcutting）→ 可能是刀具磨损，建议换刀
   3. 边缘有毛刺（burrs）→ 最后一刀进给太慢，建议提高到 400mm/min"
```

---

## 总结：LLM 在各阶段的价值定位

| 阶段 | LLM 角色 | 成熟度 | 消费级价值 |
|------|----------|--------|-----------|
| **CAD** | 自然语言 → CAD 脚本生成 | 中（简单形状可用） | **高** — 降低设计门槛 |
| **CAD** | DFM 可加工性检查 | 中 | **高** — 防止设计出不可加工的东西 |
| **CAM** | 切削参数推荐 | 中（规则库+RAG 可用） | **极高** — 解决最大痛点 |
| **CAM** | 刀具路径策略选择 | 低-中 | **高** — 替代专业工艺知识 |
| **G-code** | 直接生成 G-code | 低（安全风险） | **低** — 不建议直接用 |
| **G-code** | 审查/调试 G-code | 中 | **中** — 辅助检查 |
| **控制器** | （不适合 LLM） | — | — |
| **控制器** | 解释报错/配置辅助 | 高 | **中** — 降低调试门槛 |
| **加工** | 对话式故障诊断 | 中-高 | **极高** — 实时排障 |
| **加工** | 预测性维护（解释层） | 中 | **中** — 需要传感器硬件 |
| **加工** | 视觉质量检查 | 低-中 | **高** — 多模态 LLM 的潜力 |

**核心结论：** LLM 的核心价值不是"代替"工程计算，而是**把专业切削知识翻译成普通人能理解的语言和操作**。最大价值点在 CAM 参数推荐和加工故障诊断 — 这两个环节正是消费级用户最无助的地方。

---

## 数据来源

- [LLMs for Manufacturing – arXiv](https://arxiv.org/html/2410.21418v1)
- [LLMs for Manufacturing – ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0278612526000439)
- [Text-to-CadQuery – arXiv](https://arxiv.org/abs/2505.06507)
- [Zoo.dev Zookeeper (Text-to-CAD)](https://zoo.dev/zookeeper)
- [CloudNC – AI Cutting Parameters](https://www.cloudnc.com/news-room/new-cutting-parameters-ai-solution)
- [DELMIA – AI Machining Operations](https://blog.3ds.com/brands/delmia/revolutionizing-machining-operations-with-artificial-intelligence/)
- [CNN Toolpath Generation – MDPI](https://www.mdpi.com/2075-1702/14/1/89)
- [AI Toolpath Optimisation 2026 – Machine Tool News](https://machinetoolnews.ai/ai-toolpath-optimisation-2026/)
- [LLM for G-code Generation – i4valley](https://i4valley.com/resources/leveraging-llm-for-complex-g-code-generation-in-cnc-machinery/)
- [ChatGPT vs GPT-4o for G-code – ResearchGate](https://www.researchgate.net/publication/397562415)
- [Automatic Machining Process Route Generation – ACM](https://dl.acm.org/doi/10.1145/3756423.3756528)
- [ChatCNC – Conversational CNC Monitoring](https://www.researchgate.net/publication/389894040_ChatCNC_Conversational_machine_monitoring_via_large_language_model_and_real-time_data_retrieval_augmented_generation)
- [SprutCAM X AI Assistant](https://www.aerospacemanufacturinganddesign.com/article/ai-virtual-assistant-for-cnc-tasks/)
- [ARUM – LLM for Machining (Microsoft)](https://www.microsoft.com/en/customers/story/26357-arum-azure-ai-search)
- [Körber Digital – LLM Troubleshooting](https://www.koerber-digital.com/blog/leveraging-ai-trouble-shooting-assistant-for-machine-operators)
- [Multimodal Manufacturing Safety Chatbot – arXiv](https://arxiv.org/html/2511.11847v2)
- [RoughLSTM Vibration Anomaly – MDPI](https://www.mdpi.com/2076-3417/15/6/3179)
- [Multi-Sensor Tool Wear – PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC11054666/)
- [Real-Time Tool Anomaly Detection – Newswise](https://www.newswise.com/articles/an-innovative-strategy-for-real-time-tool-anomaly-detection-in-cnc-milling-processes-using-time-series-monitoring)
- [MIT – LLMs for Design & Manufacturing](https://mit-genai.pubpub.org/pub/nmypmnhs)
- [MIT CDFG – LLMs for Design & Manufacturing (PDF)](https://cdfg.mit.edu/assets/files/llms_for_cdam.pdf)
- [CAM as a Service – ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0278612525000664)
- [Reddit – ChatGPT-like AI for CNC Technicians](https://www.reddit.com/r/CNC/comments/1n08p39/is_it_nice_to_have_a_chatgpt_like_ai_assistant/)
