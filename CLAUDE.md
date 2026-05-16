# AI CAM — 消费级 CNC 切削参数智能推荐系统

## 项目概述

为消费级 CNC 用户（创客/爱好者）构建 AI 驱动的 CAM 辅助系统。核心解决用户缺乏材料力学、刀具几何、切削动力学专业知识导致无法正确设置加工参数的痛点。

技术路线：**确定性计算 + RAG 知识库 + LLM 推理** 混合架构。确定性公式负责安全关键的计算，LLM 负责高层决策、自然语言交互和经验推理。

---

## 目录结构

```
CNC/
├── CLAUDE.md                              # 本文件：项目说明与规范
│
├── references/                            # 参考资料与市场研究
│   ├── consumer-cnc-business-plan.md      #   消费级 CNC 市场分析与商业计划
│   └── knowledge-base/                    #   RAG 知识库源文件（材料/刀具/安全）
│
├── notes/                                 # 学习笔记与技术分析
│   ├── cnc-workflow-deep-dive.md          #   CNC 全流程详解（CAD→CAM→G-code→控制器→加工）
│   │
│   ├── cutting-basics/                    #   切削参数基础知识
│   │   ├── cutting-parameters-explained.md  # 所有符号逐个解释（RPM/Feed/ap/ae/fz/Fc/Kc...）
│   │   └── feed-rate-formula-explained.md   # 进给率公式详解（Feed = RPM × z × fz）
│   │
│   └── llm-applications/                  #   LLM 在 CNC 各阶段的应用分析
│       ├── llm-in-cnc-workflow.md         #   LLM 全流程应用场景（CAD/CAM/G-code/控制器/加工）
│       └── llm-toolpath-strategy.md       #   LLM 实现刀具路径策略选择
│
├── docs/                                  # 设计文档（从 notes 中提炼的正式方案）
│   └── ai-cam-technical-design.md         #   AI CAM 系统技术设计方案
│
├── tmp/                                   # 临时文件（gitignore）
│
└── src/                                   # 源代码（Phase 1 实现后启用）
    └── cnc_advisor/                       #   主包
```

---

## 核心设计原则

### 确定性优先，LLM 增强

| 决策类型 | 谁负责 | 理由 |
|----------|--------|------|
| RPM / Feed / 切深 计算 | 确定性公式 | 数学计算，没有歧义 |
| 安全约束检查 | 确定性代码 | 安全关键，不可覆盖 |
| 力估算 | 确定性公式 | 物理定律 |
| 参数解释/故障诊断 | LLM | 自然语言生成 |
| 罕见材料-刀具组合 | LLM 辅助 | 数据不足时的补充 |
| 刀具路径策略选择 | LLM + 规则引擎 | 经验推理 + 规则兜底 |
| G-code 生成 | 确定性代码 | 安全关键，LLM 幻觉不可接受 |
| G-code 审查/解释 | LLM | 模式识别 + 自然语言 |

### 消费级场景聚焦

- 2.5D 加工为主（平面图形 + 不同深度）
- 常见材料：铝合金、木材、MDF、亚克力
- 常见机器：3018 / X-Carve / Shapeoko / Carvera
- 刚性分级：VERY_LOW / LOW / MEDIUM / HIGH
- 常见固件：GRBL / FluidNC

---

## 已完成的研究

- [x] 消费级 CNC 市场分析与商业计划
- [x] CNC 全流程技术详解（5 个阶段）
- [x] LLM 在 CNC 各阶段的应用场景分析
- [x] 切削参数公式与符号详解
- [x] 进给率公式逐符号拆解
- [x] AI CAM 系统技术设计方案
- [x] LLM 刀具路径策略选择方案

## 待研究 / 待实现

- [ ] G-code 阶段的 LLM 应用（审查/解释/调试）
- [ ] RAG 知识库内容编写（材料指南/刀具指南/安全规范/故障排查）
- [ ] Phase 1 实现：确定性计算引擎 + FastAPI
- [ ] Phase 2 实现：RAG + LLM 集成
- [ ] Phase 3 实现：Web UI / 社区反馈循环

---

## 编码规范

- **语言**：Python 3.11+
- **框架**：FastAPI + SQLModel + Pydantic v2
- **环境**：`/home/ropliu/miniconda3/envs/occ` conda 环境
- **数据库**：SQLite（桌面友好，无需外部服务）
- **向量库**：ChromaDB（纯 Python，文件存储）
- **测试**：pytest，目标 80%+ 覆盖率
- **文档语言**：中文（面向中文用户）
- **代码注释**：最小化，只在 WHY 不明显时加注释
