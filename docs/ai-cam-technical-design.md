# AI CAM 切削参数推荐系统 — 技术方案

> 最后更新：2025-05-15

---

## 1. 系统架构

### 1.1 分层设计

```
API 层 (FastAPI)
    ↓
编排层 (RecommendationEngine — 协调计算/检索/推理)
    ↓
┌──────────────────┬──────────────────┬──────────────────┐
│ 计算引擎         │ LLM 推理层       │ 知识/数据层      │
│ (确定性公式)     │ (自然语言)       │ (RAG + SQLite)   │
│                  │                  │                  │
│ • RPM/进给/切深  │ • 参数解释       │ • 材料数据库     │
│ • 力估算         │ • 意图理解       │ • 刀具数据库     │
│ • 安全约束       │ • 故障诊断       │ • 向量知识库     │
│                  │                  │ • 社区反馈记录   │
└──────────────────┴──────────────────┴──────────────────┘
```

### 1.2 核心原则：确定性优先，LLM 增强

切削参数受物理定律约束，LLM 幻觉在此场景不可接受。

| 决策类型 | 谁负责 | 理由 |
|----------|--------|------|
| RPM = (Vc×1000)/(π×D) | **确定性公式** | 数学计算，没有歧义 |
| Feed = RPM × z × fz | **确定性公式** | 同上 |
| 最大切深（基于刚性等级） | **查表 + 约束** | 安全关键 |
| 解释"为什么推荐这些参数" | **LLM** | 自然语言生成 |
| 处理罕见材料-刀具组合 | **LLM 辅助推理** | 数据不足时的补充 |
| 安全约束检查 | **确定性代码** | 最后一道防线，不可覆盖 |

### 1.3 数据流

```
用户输入 (材料 + 刀具 + 机器)
    ↓
[1] 输入解析 → 映射到 material_id, tool_id, machine_profile
    ↓
[2] 确定性计算（永远先执行）
    → 查材料属性（Kc, Vc 范围）
    → 查刀具属性（直径, 齿数, 涂层）
    → 公式计算 RPM / Feed / Depth / Stepover
    → 力估算 + 安全约束
    ↓
[3] 知识检索（与步骤2并行）
    → 结构化查询：材料+刀具的精确匹配记录
    → 语义搜索：相关加工指南和社区经验
    ↓
[4] LLM 推理（可选，增强步骤2）
    → 接收确定性结果 + 检索上下文
    → 生成参数调整建议 + 解释 + 警告
    → LLM 输出必须通过安全验证器
    ↓
[5] 安全验证（永远最后执行）
    → 硬约束检查（chip load 范围、力上限、机器极限）
    → 覆盖任何违反约束的 LLM 建议
    ↓
输出: {rpm, feed_rate, depth, stepover, explanation, warnings}
```

---

## 2. 数据模型

所有模型使用 Pydantic v2 验证 + SQLModel ORM 映射到 SQLite。

### 2.1 材料数据库

```python
class Material:
    id: str                      # "al_6061_t6"
    name: str                    # "6061-T6 铝合金"
    category: MaterialCategory   # ALUMINUM, STEEL, WOOD, PLASTIC, BRASS, COPPER

    # 切削关键属性
    kc: float                    # 比切削力 (N/mm²) — Kienzle
    vc_roughing: (float, float)  # 粗加工推荐切削速度范围 (m/min)
    vc_finishing: (float, float) # 精加工推荐切削速度范围 (m/min)
    fz_factor: float             # 切屑负载修正系数
    hardness_hb: float | None    # 布氏硬度
    machinability_rating: int    # 可加工性评分 1-10

    # 消费级相关
    coolant_required: bool       # 是否需要冷却液
    common_consumer_use: bool    # 爱好者是否常用
    notes: str                   # "切屑容易粘刀，建议使用润滑"
```

关键材料参考值：

| 材料 | Kc (N/mm²) | Vc 硬质合金 (m/min) | 可加工性 |
|------|-----------|-------------------|---------|
| 6061-T6 铝 | ~700 | 300-500 | 8/10 |
| A36 钢 | ~1800 | 80-150 | 4/10 |
| 黄铜 | ~800 | 200-400 | 9/10 |
| MDF | ~30 | 300-600 | 10/10 |
| 亚克力 | ~350 | 150-300 | 6/10 |
| 硬木（橡木） | ~100 | 200-400 | 8/10 |
| 紫铜 | ~900 | 150-300 | 5/10 |
| 软钢 (1045) | ~1500 | 100-200 | 5/10 |

### 2.2 刀具数据库

```python
class Tool:
    id: str                         # "flat_6mm_2f_carbide"
    tool_type: ToolType             # FLAT_ENDMILL, BALL_NOSE, VBIT, DRILL

    # 几何参数
    diameter_mm: float              # 刀具直径 D
    flutes: int                     # 刃数 z
    flute_length_mm: float          # 有效切削长度
    coating: ToolCoating            # UNCOATED, TIN, TIALN, ZRN
    material: ToolMaterial          # HSS, CARBIDE

    # 切削特性
    recommended_chip_load: (float, float)  # fz 范围 (mm/tooth)
    max_engagement_ratio: float     # 最大径向切宽比 (如 0.5 = 50% D)
    max_depth_ratio: float          # 最大轴向切深比 (如 1.0 = 100% D)

    # 消费级
    common_consumer_tool: bool
    collet_size: str                # "ER11"
```

### 2.3 机器配置

```python
class MachineProfile:
    id: str                     # "shapeoko_4"
    name: str                   # "Shapeoko 4"
    max_rpm: int                # 12000
    min_rpm: int                # 8000
    spindle_power_watts: float  # 300
    max_feed_rate_mm_min: float # 3000

    # 刚性等级 — 决定切深上限的关键参数
    rigidity_class: RigidityClass
    # VERY_LOW: 3018 级 (V-wheel + 皮带)
    # LOW:      X-Carve 级
    # MEDIUM:   Shapeoko 级
    # HIGH:     桌面铣床级 (线性导轨 + 滚珠丝杠)

    controller_firmware: str    # "GRBL", "FluidNC"
```

内置预设：

| 预设 ID | 机器 | 刚性 | 最大 RPM | 功率 |
|---------|------|------|---------|------|
| `cnc_3018` | Generic 3018 | VERY_LOW | 10000 | 80W |
| `x_carve` | X-Carve (stock) | LOW | 12000 | 300W |
| `shapeoko_4` | Shapeoko 4 | MEDIUM | 12000 | 300W |
| `shapeoko_5_pro` | Shapeoko 5 Pro | HIGH | 24000 | 650W |
| `carvera` | Makera Carvera | MEDIUM | 20000 | 300W |
| `custom` | 用户自定义 | (自选) | (自设) | (自设) |

### 2.4 社区反馈记录

```python
class CuttingParameterRecord:
    material_id: str
    tool_id: str
    machine_profile_id: str

    # 实际使用的参数
    rpm: int
    feed_rate_mm_min: float
    depth_of_cut_mm: float
    stepover_pct: float

    # 结果评价
    result_quality: ResultQuality  # EXCELLENT, GOOD, ACCEPTABLE, POOR, FAILED
    issues: list[str] | None       # ["chatter", "tool_breakage"]
    notes: str | None

    # 来源
    source: str       # "community", "manufacturer", "lab_test"
    verified: bool
```

---

## 3. 计算引擎

### 3.1 核心公式

每个公式返回 `ParameterRange(min, recommended, max)` 而非单一值。

**RPM 计算：**

```
RPM = (Vc × 1000) / (π × D)

修正系数：
  HSS 刀具:     Vc_base × 0.4
  硬质合金:     Vc_base × 1.0
  TiAlN 涂层:  Vc_base × 1.2
  刚性 VERY_LOW: Vc_base × 0.7
```

**进给率计算：**

```
Feed = RPM × z × fz

修正系数：
  铝合金: fz_base × 1.0
  硬木:   fz_base × 1.2
  MDF:    fz_base × 1.5
  钢铁:   fz_base × 0.3
  刚性 VERY_LOW: fz × 0.6
```

**切深上限（查表）：**

```
最大切深 = f(刚性等级, 材料类别, 刀具直径 D)

              铝     硬木    MDF    钢     塑料
VERY_LOW:   0.10D  0.25D  0.30D  0.03D  0.15D
LOW:        0.15D  0.35D  0.40D  0.05D  0.20D
MEDIUM:     0.25D  0.50D  0.60D  0.08D  0.30D
HIGH:       0.40D  0.75D  1.00D  0.15D  0.40D
```

**力估算（安全检查）：**

```
Fc = Kc × ap × ae × fz  (简化 Kienzle 方程)

安全阈值（按刚性等级）：
  VERY_LOW: 50 N
  LOW:      100 N
  MEDIUM:   200 N
  HIGH:     500 N

超限处理：先减切深 ap，再减切宽 ae，最后减 fz
```

### 3.2 安全约束（硬限制，不可覆盖）

```python
SAFETY = {
    "min_chip_load": 0.01,          # 低于此值 = 摩擦而非切削
    "max_chip_load": "0.04×√D",     # 超过 = 断刀风险
    "max_depth_ratio_consumer": 1.0, # 消费级机器不超过刀具直径
    "max_stepover_steel_pct": 50,    # 钢铁不超过 50%
    "max_stepover_aluminum_pct": 75, # 铝不超过 75%
    "force_limit": "按刚性等级",     # 见上表
}
```

---

## 4. RAG 知识库

### 4.1 什么放入知识库 vs 什么用公式计算

| 放入公式计算 | 放入知识库 |
|-------------|-----------|
| RPM / Feed / 切深的基础值 | 材料特定的加工技巧和注意事项 |
| 力估算 | 故障排查知识（"表面有波纹怎么办"） |
| 安全约束限制 | 刀具涂层选择的指导 |
| — | 社区实际加工记录 |
| — | 安全操作指南 |

### 4.2 混合检索架构

```
查询: material_id + tool_type + rigidity_class
    ↓
┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ SQLite 精确查询  │  │ 向量语义搜索      │  │ 社区记录查询      │
│ (材料/刀具属性)  │  │ (加工指南/技巧)   │  │ (实际使用记录)    │
└────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                    │                      │
         └────────────────────┼──────────────────────┘
                              ↓
                      合并 + 排序
              优先级: 精确数据 > 社区记录 > 语义搜索
```

**文本分块策略：**
- 文本文档（加工指南、教程）：512 tokens，64 token 重叠
- 结构化记录（材料/刀具数据）：不分块，存入 SQLite 精确索引
- 为每条结构化记录生成"自然语言摘要"存入向量库，用于语义搜索

**冲突数据处理：**
1. 取最保守值作为基线
2. 向 LLM 展示范围而非单一值
3. 按来源质量加权：`厂商数据 > 实验数据 > 已验证社区 > 未验证社区`
4. LLM 解析冲突并解释权衡，但安全验证器确保最终值在保守范围内

---

## 5. LLM 集成

### 5.1 提供商抽象

```python
class LLMProvider(Protocol):
    async def complete(self, messages: list[dict], **kwargs) -> str: ...
    async def complete_structured(
        self, messages: list[dict],
        response_schema: type[BaseModel],
        **kwargs
    ) -> BaseModel: ...

class OpenAIProvider(LLMProvider): ...
class AnthropicProvider(LLMProvider): ...
class LocalProvider(LLMProvider): ...    # Ollama / llama.cpp
```

### 5.2 LLM 使用场景

| 场景 | 用 LLM？ | 理由 |
|------|---------|------|
| 标准材料+标准刀具+标准机器 | **否** | 确定性计算足够 |
| 解释推荐理由 | **是** | 自然语言生成 |
| 罕见组合缺少数据 | **是** | 从相似已知案例推理 |
| 处理模糊输入（"我要好的表面光洁度"） | **是** | 意图理解 |
| 安全约束检查 | **否** | 确定性代码执行 |

### 5.3 Prompt 策略

```
SYSTEM: 你是消费级 CNC 切削参数顾问。
收到确定性计算结果和知识库上下文后：
1. 验证参数的物理合理性
2. 在允许范围内调整（如果上下文建议改进）
3. 用简单语言解释推理
4. 标注特定风险

绝不建议超出 [min, max] 范围的参数。
绝不覆盖安全约束。

USER:
材料: 6061-T6 铝合金
刀具: 6mm 2刃硬质合金平底刀
机器: Shapeoko 4 (MEDIUM 刚性, 12000 RPM)

确定性计算结果:
- RPM: 10610 (范围: 7958-15915)
- Feed: 637 mm/min (范围: 424-849)
- 切深: 1.5mm (最大: 1.5mm)
- 步距: 3mm (50% D)
- 估算切削力: 63 N (限制: 200 N)

知识库上下文: [检索到的相关文档]
社区数据: [相似组合的实际使用记录]

请审查参数，提供最终推荐值 + 解释 + 警告。
```

### 5.4 输出验证

每个 LLM 输出都经过安全验证器：

```python
def validate_llm_output(llm_result, deterministic, safety):
    # Clamp RPM 到机器 min/max
    rpm = clamp(llm_result.rpm, machine.min_rpm, machine.max_rpm)

    # 确保 chip load 在安全范围内
    fz = rpm * tool.flutes / llm_result.feed_rate
    fz_clamped = clamp(fz, safety.min_chip_load, safety.max_chip_load)
    feed_rate = rpm * tool.flutes * fz_clamped

    # 切深不超过安全上限
    depth = min(llm_result.depth_of_cut, deterministic.depth_max)

    # 力估算验证
    force = estimate_force(material.kc, depth, stepover, fz_clamped)
    if force > safety.force_limit:
        depth = reduce_depth_for_force(force, safety.force_limit)

    return ValidatedRecommendation(
        rpm=rpm, feed_rate=feed_rate,
        depth_of_cut=depth, stepover=stepover,
        was_adjusted=(rpm != llm_result.rpm or feed_rate != llm_result.feed_rate),
        explanation=llm_result.explanation,
        warnings=llm_result.warnings
    )
```

### 5.5 无 LLM 降级

LLM 不可用时：
- 返回确定性计算结果
- 使用预写模板解释
- 标注"基线值，连接 AI 获取优化建议"

---

## 6. API 设计

### 6.1 核心端点

```
GET  /api/v1/health                    # 健康检查，LLM 状态
POST /api/v1/recommend                 # 主推荐端点
GET  /api/v1/materials                 # 材料列表（支持搜索）
GET  /api/v1/materials/{id}            # 材料详情
GET  /api/v1/tools                     # 刀具列表（支持搜索）
GET  /api/v1/tools/{id}                # 刀具详情
GET  /api/v1/machines                  # 机器预设列表
GET  /api/v1/machines/{id}             # 机器配置详情
POST /api/v1/machines                  # 创建自定义机器配置
POST /api/v1/feedback                  # 提交加工结果反馈
```

### 6.2 请求/响应示例

```json
// POST /api/v1/recommend
{
  "material_id": "al_6061_t6",
  "tool_id": "flat_6mm_2f_carbide",
  "machine_id": "shapeoko_4",
  "operation_type": "roughing",
  "priority": "balanced",
  "coolant_available": false,
  "use_llm": true
}

// Response 200
{
  "parameters": {
    "rpm": 10610,
    "feed_rate_mm_min": 637,
    "depth_of_cut_mm": 1.5,
    "stepover_mm": 3.0,
    "stepover_pct": 50,
    "chip_load_mm": 0.06,
    "plunge_rate_mm_min": 200,
    "estimated_force_n": 63
  },
  "explanation": "对于 6061 铝合金，使用 6mm 硬质合金刀具...",
  "warnings": [
    {"level": "INFO", "message": "切屑容易粘刀，建议使用 WD-40 润滑"}
  ],
  "calculation_details": {
    "vc_used": 200,
    "fz_used": 0.06,
    "depth_ratio": 0.25,
    "force_limit": 200,
    "force_utilization_pct": 31.5
  },
  "confidence": "HIGH",
  "llm_used": true,
  "data_sources": ["material_db", "tool_db", "community_records"]
}
```

### 6.3 错误处理

| 错误码 | 含义 | 处理 |
|--------|------|------|
| `MATERIAL_NOT_FOUND` | 材料不存在 | 建议相似材料 |
| `INFEASIBLE_COMBINATION` | 组合不可行（如钢在 3018 上） | 解释原因 + 建议替代方案 |
| `LLM_UNAVAILABLE` | LLM 不可用 | 返回确定性结果 + 标注 |
| `SAFETY_VIOLATION` | 内部安全违规（不应到达用户） | 日志 + 降级 |

---

## 7. 项目结构

```
cnc-cutting-advisor/
├── pyproject.toml
├── src/cnc_advisor/
│   ├── main.py                     # FastAPI 入口
│   ├── api/routes/                  # API 路由
│   │   ├── recommend.py            # POST /recommend
│   │   ├── materials.py            # 材料增删查
│   │   ├── tools.py                # 刀具增删查
│   │   └── machines.py             # 机器配置
│   ├── engine/                      # 核心计算引擎
│   │   ├── calculator.py            # 确定性参数计算
│   │   ├── formulas.py              # 纯公式实现
│   │   ├── safety.py                # 安全约束验证
│   │   ├── force.py                 # 力估算
│   │   └── orchestrator.py          # RecommendationEngine 编排器
│   ├── models/                      # 数据模型
│   │   ├── material.py
│   │   ├── tool.py
│   │   ├── machine.py
│   │   ├── parameters.py
│   │   └── database.py              # SQLite (SQLModel)
│   ├── knowledge/                   # RAG 知识库
│   │   ├── retriever.py             # 混合检索
│   │   ├── vector_store.py          # ChromaDB 接口
│   │   ├── embeddings.py            # Embedding 模型管理
│   │   └── indexer.py               # 知识库构建
│   ├── llm/                         # LLM 集成
│   │   ├── client.py                # 提供商抽象
│   │   ├── providers/
│   │   │   ├── openai.py
│   │   │   ├── anthropic.py
│   │   │   └── local.py             # Ollama / llama.cpp
│   │   ├── prompts.py               # Prompt 模板
│   │   ├── output_parser.py         # 结构化输出解析
│   │   └── validator.py             # 输出安全验证
│   └── data/                        # 种子数据
│       ├── materials.json           # 50+ 材料
│       ├── tools.json               # 30+ 常用刀具
│       ├── machines.json            # 6 个机器预设
│       └── knowledge/               # RAG 文本源
│           ├── aluminum_machining_guide.md
│           ├── steel_machining_guide.md
│           ├── wood_machining_guide.md
│           ├── troubleshooting.md
│           └── safety_guidelines.md
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_formulas.py         # 公式验证
│   │   ├── test_safety.py           # 安全约束测试
│   │   ├── test_calculator.py       # 计算引擎测试
│   │   └── test_retriever.py        # 检索测试
│   ├── integration/
│   │   ├── test_recommendation.py   # 端到端推荐测试
│   │   └── test_api.py              # API 测试
│   └── fixtures/
│       ├── test_materials.json
│       └── known_good_parameters.json  # 专家验证的参考值
└── scripts/
    ├── seed_database.py             # 初始化种子数据
    ├── build_knowledge_base.py      # 构建向量知识库
    └── validate_parameters.py       # 交叉验证
```

---

## 8. 分阶段交付

### Phase 1: 确定性核心（Weeks 1-4）

无 LLM，纯公式 + 数据库。

- 数据模型 + SQLite 种子数据（30 材料, 20 刀具, 6 机器）
- 公式引擎（RPM, Feed, Depth, Stepover, Force）
- 安全约束模块
- FastAPI 接口
- 单元测试 + 已知值回归测试

**交付物：** `curl POST /recommend` 返回正确的参数 + 模板解释。

**验证：** 所有测试用例通过专家验证的已知值。力估算在公开值的 30% 以内。

### Phase 2: RAG + LLM 增强（Weeks 5-8）

- ChromaDB 向量知识库 + 混合检索
- LLM 提供商抽象（OpenAI/Anthropic/Ollama）
- Prompt 模板 + 结构化输出解析
- 编排器（合并确定性 + RAG + LLM）
- CLI 界面（离线使用）
- 社区反馈提交接口

**交付物：** 富解释 + 上下文提示 + 离线降级能力。

**验证：** LLM 增强建议与确定性基线偏差不超过 10%。解释经人工审核。

### Phase 3: 完整系统（Weeks 9-16+）

- 社区数据学习循环（成功参数记录影响未来推荐）
- 刀具路径策略推荐（上传几何 → 识别特征 → 推荐加工策略）
- G-code 审查（检查常见错误模式）
- Web UI（非技术用户界面）
- 多语言支持（中文/英文）
- GRBL/FluidNC 直接参数推送

---

## 9. 技术选型

| 选择 | 理由 |
|------|------|
| Python + FastAPI | conda 环境已有，CNC 生态 Python 为主 |
| SQLModel + SQLite | 无需外部数据库，桌面友好 |
| ChromaDB 向量库 | 纯 Python，文件存储，适合中小规模知识库 |
| Pydantic v2 | 类型安全 + 验证 + API schema 一体 |
| 确定性优先架构 | 安全关键，LLM 只在安全范围内增强 |

---

## 10. 验证方式

1. **公式验证**：每个公式用已知输入输出对测试（如 `RPM(Vc=300, D=6) = 15915`）
2. **已知值回归测试**：准备 20-30 组专家验证的参数组合，确保系统输出在合理范围内
3. **安全约束测试**：注入超出范围的值，验证被正确拒绝
4. **LLM 输出验证**：确认 LLM 建议不超出确定性计算的安全范围
5. **端到端测试**：`POST /recommend` 从输入到完整输出的全链路

---

## 11. 关键参考

- [OpenCAMLib (C++)](https://github.com/aewallin/opencamlib) — 开源刀具路径库
- [bCNC (Python)](https://github.com/vlachoudis/bcnc) — GRBL 发送器 + 基础 CAM
- [cncfeedspeed](https://github.com/tiktuk/cncfeedspeed) — Web 切削参数计算器
- [Speeds-And-Feeds](https://github.com/bhowiebkr/Speeds-And-Feeds) — 刀具管理系统
- [CloudNC AI Cutting Parameters](https://www.cloudnc.com/news-room/new-cutting-parameters-ai-solution) — 商业 AI 参数优化
- [DELMIA AI Machining](https://blog.3ds.com/brands/delmia/revolutionizing-machining-operations-with-artificial-intelligence/) — AI 刀具路径提案
- [LLMs for Manufacturing (arXiv)](https://arxiv.org/html/2410.21418v1) — LLM + 制造综述
- [Text-to-CadQuery (arXiv)](https://arxiv.org/abs/2505.06507) — LLM 生成 CAD 代码
- [ChatCNC (ResearchGate)](https://www.researchgate.net/publication/389894040) — 对话式 CNC 监控
