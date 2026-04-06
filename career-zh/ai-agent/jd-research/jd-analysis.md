# AI Agent Engineer JD 搜索与分析报告

> 岗位：AI Agent Engineer（区别于 Agent Infrastructure Engineer）
> 搜索时间：2026-04-06
> 成功获取：8 份完整 JD + 多份职位列表

---

## 一、岗位定义与区分

**AI Agent Engineer** 聚焦于 Agent **应用层**：构建 Agent 产品、设计 Prompt 策略、集成 RAG、优化 Agent 行为、评估 Agent 质量。
**AI Agent Infrastructure Engineer** 聚焦于 Agent **底座层**：Runtime 引擎、进程隔离、安全沙箱、重试引擎、协议栈。

两者有重叠（如 Tool Use、Context Management），但核心关注点不同。

---

## 二、JD 汇总表

| # | 公司 | 职位名称 | 薪资范围 (USD) | 地点 | 关键侧重 |
|---|------|---------|---------------|------|----------|
| 1 | Anthropic | Research Engineer, Agents | $500K-$850K | SF/Seattle/NYC (Remote) | Agent harness 设计、benchmark、finetuning、multi-agent |
| 2 | Anthropic | Research Engineer, AI Observability | $320K-$405K | SF | AI 监控系统、agentic integration、大规模数据分析 |
| 3 | Cohere | Applied AI Engineer, Agentic Workflows | 未披露 | SF/NYC/Toronto/London | 企业级 Agent、RAG、tool-calling、evaluation |
| 4 | Scale AI | ML Research Engineer, Agents | $189K-$237K | SF/NYC | Agent RL、post-training、多 Agent 系统 |
| 5 | Scale AI | Sr/Staff ML Engineer, General Agents | $218K-$273K | SF/NYC | 端到端 Agent 系统、evaluation、tool use、部署 |
| 6 | Scale AI | Deep Research Agent Tech Lead | $264K-$331K | SF/NYC | 知识检索、Text-to-SQL、多模态、multi-agent |
| 7 | Scale AI | Applied AI Engineer, Enterprise GenAI | $216K-$270K | SF/NYC | 企业 AI 产品、multimodal、tool-calling |
| 8 | Scale AI | Evals Engineer, Applied AI | $216K-$270K | SF/NYC | LLM-as-Judge、evaluation 框架、Agent 行为分析 |

---

## 三、完整 JD 内容

### JD 1：Anthropic — Research Engineer, Agents

**薪资**: $500,000 – $850,000 USD | **地点**: Remote-Friendly (Travel-Required) | SF | Seattle | NYC

**关于角色**：专注于让 Claude 在长时间和多 Agent 任务中更有效。团队跨越 agent harness 设计、基础设施、affordance 和微调。

**职责**：
- 开发和比较 agent harness（memory、context compression、communication architecture）
- 设计大规模 agentic 任务的定量 benchmark
- 协助模型和 prompt 评估的自动化
- 与产品团队协作解决 Agent 产品化的挑战
- 创建和优化 agentic 任务性能的训练数据

**必要资质**：
- 使用 LLM 开发复杂 agentic 系统的经验
- 显著的软件工程和 ML 经验
- 使用语言模型做 prompting 和/或构建产品的经验
- 良好的沟通能力和协作研究兴趣
- 对安全和有益 AI 的热情

**加分资质**：
- Large-scale RL on language models
- Multi-agent systems

**代表性项目**：
- 在 coding/knowledge work benchmark 上超越现有 agent 的新型 agent harness
- 衡量多 Agent 群体解决问题能力的 evals
- 为特定工具/harness 微调 Claude

---

### JD 2：Anthropic — Research Engineer, AI Observability

**薪资**: $320,000 – $405,000 USD | **地点**: SF

**关于角色**：设计和构建让 AI 分析大规模非结构化数据集并生成结构化、可信洞察的系统。

**职责**：
- 构建 AI 训练和部署的 AI-based 监控系统
- 开发处理大量非结构化文本的核心框架
- 与研究员和安全团队合作构建解决方案
- 开发用于自主调查和发现处理的 agentic integration
- 参与团队战略规划

**必要资质**：
- 5+ 年软件工程经验，有 ML 系统接触
- 熟悉 LLM 应用开发（context engineering、evaluation、orchestration）
- 热衷于构建他人使用的工具
- 能在深度基础设施和产品思维之间切换

**加分资质**：
- AI 安全/对齐/负责任部署研究经验
- 大规模数据处理框架经验
- 监控或可观测性系统背景

---

### JD 3：Cohere — Applied AI Engineer, Agentic Workflows

**地点**: SF/NYC/Toronto/Montreal/London

**关于角色**：为企业客户构建生产级 AI Agent。设计、构建和部署 LLM 驱动的 agentic workflow，从原型到生产。Agent 必须从一开始就"可靠、可观测、安全、可审计"。

**职责**：
- 将模糊的企业业务问题转化为定义明确的 agentic 问题
- 领导 LLM-powered agent 的设计和交付——"推理、规划、跨工具和数据源行动"
- 定义评估和质量标准，衡量成功、失败和回归
- 调试 Agent 行为，改进 prompts、workflows、tools 和 guardrails
- 跨分布式团队指导工程师

**必要资质**：
- **Production Engineering**: Python/TypeScript 交付生产级软件
- **Agentic Architectures**: 使用 ReAct、Plan-and-Execute 等模式，与外部 API/工具交互
- **LLM Stack**: GPT/Claude/Gemini、RAG、向量数据库（Pinecone/Weaviate）、编排框架（LangGraph/CrewAI/自定义状态机）
- **Rigorous Evaluation**: 构建度量 Agent 准确性、安全性和延迟的评估框架
- 领导企业客户技术讨论，将模糊需求转化为技术规范

---

### JD 4：Scale AI — ML Research Engineer, Agents

**薪资**: $189,600 – $237,000 USD | **地点**: SF/NYC

**关于角色**：Enterprise ML Research Lab，应用 agent-based RL 和算法到企业数据集。

**职责**：
- 为企业客户训练和部署 SOTA 模型
- 研究前沿算法集成到训练栈
- 使用专有算法构建 agent，包括设计高性能工具、多 agent 系统和复杂奖励函数

**理想资质**：
- 1-3 年生产级 LLM 构建经验
- 熟悉 post-training 方法（RLHF/RLVR）和 PPO/GRPO 算法
- 顶会论文（NeurIPS/ICLR/ICML）
- CS 相关硕士/博士

---

### JD 5：Scale AI — Sr/Staff ML Engineer, General Agents

**薪资**: $218,000 – $273,000 USD | **地点**: SF/NYC

**关于角色**：General Agents 团队，构建面向企业客户的生产级 AI Agent，侧重泛化性、可扩展性和多客户部署。

**职责**：
- 设计端到端 Agent 系统——组合 LLM 推理、tool use、memory 和控制逻辑
- 构建可扩展、可靠的 Agent 架构
- 开发评估框架、数据集和指标
- 产品化前沿技术（planning、multi-step reasoning、tool-use、multi-agent patterns）
- 负责部署、监控、失败分析和持续改进

**必要资质**：
- 5+ 年 ML/AI 系统生产部署经验
- 深入理解现代 LLM、prompt/context/system-level 优化和 agentic 系统设计
- 强 Python 能力
- 模型与外部工具/API/数据库集成经验

**加分资质**：
- 使用现代 GenAI 栈构建 AI Agent 的实操经验
- Agent 框架/编排层/workflow 系统经验
- 评估、监控和 LLM 可观测性经验
- 大规模云部署和 ML 系统运维

---

### JD 6：Scale AI — Deep Research Agent Tech Lead

**薪资**: $264,800 – $331,000 USD | **地点**: SF/NYC

**关于角色**：企业级深度研究 Agent 的 Tech Lead，将 GenAI/LLM/agentic 框架研究转化为生产系统。

**职责**：
- 负责企业深度研究 Agent 的技术战略、架构和路线图
- 核心 Agent 能力：高级知识检索、Text-to-SQL 数据分析 Agent、多模态 AI 集成
- 设计可扩展、低延迟的多 Agent 编排和评估基础设施
- 指导 ML 工程师和研究科学家

**必要资质**：
- 8+ 年软件开发，6+ 年 ML/DL/Applied Research 生产经验
- 2+ 年 Tech Lead 经历
- 深入 GenAI 和 LLM 专业知识
- 生产级 AI Agent 构建和部署经验
- 大规模分布式系统和实时数据处理经验

**加分资质**：
- 生产级 Text-to-SQL 经验
- 多模态 AI 经验（OCR、vision-language models）
- RL、推理/规划、agentic 系统研究背景
- 向量数据库和高级检索经验

---

### JD 7：Scale AI — Applied AI Engineer, Enterprise GenAI

**薪资**: $216,000 – $270,000 USD | **地点**: SF/NYC

**关于角色**：为企业客户构建前沿 AI 产品，涵盖网络安全到新闻到基因组学。

**职责**：
- 负责和优化企业客户技术挑战的 AI 解决方案
- 使用 SGP 构建高级 AI Agent，包括多模态功能、tool-calling 等
- 收集业务需求并转化为技术方案
- 跨多环境推送生产代码

**必要资质**：
- 数据驱动 ML 迭代和理解数据集变化对模型结果的影响
- 云经验（AWS/GCP）
- Python 熟练度

---

### JD 8：Scale AI — Evals Engineer, Applied AI

**薪资**: $216,000 – $270,000 USD | **地点**: SF/NYC

**关于角色**：企业评估团队，确保 LLM-powered workflows 和 agents 的安全性、可靠性和持续改进。

**职责**：
- 将模糊需求转化为结构化评估数据，创建黄金标准人类评级数据集和专家 rubric
- 设计和开发 LLM-as-a-Judge autorater 框架和 AI 辅助评估系统
- 驱动自动分析、评估和改进企业 Agent 行为的新方法研究

**必要资质**：
- 2+ 年 Applied ML/评估基础设施经验
- LLM 和 GenAI 实操经验
- 前沿模型评估方法的深度理解
- Python + PyTorch/TensorFlow

**加分资质**：
- LLM-as-a-Judge 或自动评估系统经验
- 大规模模型/Agent 评估和监控的可扩展 pipeline 经验

---

## 四、职位列表（未抓取完整 JD）

### Cohere — Agentic Platform 团队

| 职位 | 地点 | 类型 |
|------|------|------|
| Engineering Manager, North | London/Toronto | 管理 |
| Forward Deployed Engineer, Agentic Platform | Toronto/US | 应用 |
| Forward Deployed Engineer, Agentic Platform (Public Sector) | Ottawa | 应用 |
| Forward Deployed Engineer, Infrastructure Specialist | Japan/Korea/Singapore | 基础设施 |
| Forward Deployed Engineer, Prompt Specialist | Toronto/NYC/SF | Prompt |
| Lead Data Scientist | US/Canada | 数据 |
| Member of Technical Staff, Agent Code | London/Paris/Toronto/NYC | 工程 |
| Member of Technical Staff, Agents Modeling | NYC | ML |

### Scale AI — Agent 相关团队

| 职位 | 地点 |
|------|------|
| ML Systems Research Engineer, Agent Post-training | SF/NYC |
| Staff ML Research Engineer, Agent Post-training | SF/NYC |
| ML Research Engineer, Agent Data Foundation | SF/NYC |
| Senior Forward Deployed AI Engineer, Enterprise | SF/NYC |
| Staff Software Engineer, Enterprise GenAI | SF/NYC |

---

## 五、高频技能关键词提取（30 个）

从上述 JD 中提取的最高频技能要求：

| 频率 | 关键词 |
|------|--------|
| **极高** (6+/8 JD) | LLM/Generative AI, Python, Agent System Design, Tool Use/Function Calling, Evaluation/Testing |
| **高** (4-5/8 JD) | Production Deployment, RAG/Retrieval, Multi-Agent, Prompt Engineering, Reasoning/Planning |
| **中** (2-3/8 JD) | Streaming, RL/Post-training, Vector Database, Enterprise Customer-Facing, Distributed Systems |
| **低** (1/8 JD) | Text-to-SQL, Multimodal AI, RLHF/RLVR, Fine-tuning, OCR/Document Intelligence |

### 与 Agent Infra 岗位的关键词差异

| AI Agent Engineer 特有 | 两者共有 | Agent Infra 特有 |
|------------------------|---------|--to-----|
| Evaluation/Evals 框架 | Tool Use/Function Calling | Process Isolation |
| RAG/Vector Database | Multi-Agent | Security Sandbox |
| Prompt Engineering (重点) | LLM API Integration | Retry Engine |
| RL/Post-training | Context Management | MCP Protocol Stack |
| Enterprise Customer-Facing | Streaming | Permission System |
| Agent Harness Design | Error Handling | Configuration Pipeline |
| LLM-as-Judge | Token Management | Session Persistence |
| Fine-tuning | Observability | Graceful Shutdown |
