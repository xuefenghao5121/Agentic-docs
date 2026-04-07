# OpenClaw Agentic 工作分享

> 基于鸡你太美团队真实项目经验总结
> 更新时间: 2026-04-07
> GitHub: https://github.com/xuefenghao5121/Agentic-docs

---

## 目录

- [一、我们遇到的问题与解决方案](#一我们遇到的问题与解决方案)
  - [1.1 记忆系统问题](#11-记忆系统问题)
  - [1.2 Agent Team 系统问题](#12-agent-team-系统问题)
  - [1.3 飞书连接问题](#13-飞书连接问题)
- [二、记忆系统架构](#二记忆系统架构)
  - [2.1 三层记忆架构](#21-三层记忆架构)
  - [2.2 OpenViking 上下文数据库](#22-openviking-上下文数据库)
  - [2.3 SynapMind 突触记忆系统](#23-synapmind-突触记忆系统)
- [三、进化系统架构](#三进化系统架构)
  - [3.1 三层进化机制](#31-三层进化机制)
  - [3.2 灵感引擎 - 隔离版](#32-灵感引擎---隔离版)
  - [3.3 SynapMind 权重管理](#33-synapmind-权重管理)
- [四、Agent Team 系统架构](#四agent-team-系统架构)
  - [4.1 团队类型划分](#41-团队类型划分)
  - [4.2 团队结构设计](#42-团队结构设计)
  - [4.3 隔离控制架构](#43-隔离控制架构)
  - [4.4 正确激活流程](#44-正确激活流程)
- [五、关键经验总结](#五关键经验总结)
  - [5.1 必记规则表](#51-必记规则表)
  - [5.2 双写双审对抗编码范式](#52-双写双审对抗编码范式)
  - [5.3 Claude Code 调用规范](#53-claude-code-调用规范)
- [六、项目成果展示](#六项目成果展示)
  - [6.1 天渊团队: MHF-NeuralOperator](#61-天渊团队-mhf-neuraloperator)
  - [6.2 灵码团队: Sci-Orbit](#62-灵码团队-sci-orbit)
- [七、常见错误速查](#七常见错误速查)
- [八、下一步规划](#八下一步规划)

---

## 一、我们遇到的问题与解决方案

### 1.1 记忆系统问题

#### 问题 1: Context Overflow (上下文溢出)

**症状**:
- GLM-5 频繁出现 `context overflow`，回复突然中断
- 长对话后 token 累积超过模型上下文限制
- 飞书和 webchat 会话独立，上下文无法跨会话共享

**根因分析**:
- 没有自动压缩机制，token 只增不减
- 所有对话都放在 Layer 0，没有分层
- 重要信息和临时对话混在一起

**解决方案: 三层递归压缩机制**

```python
# Layer 0 (当前会话): 完整对话 → 超过 70% token → 压缩到 Layer 1
# Layer 1 (短期记忆): 完整对话 → 摘要 (10:1 压缩比)
# Layer 1 → 超过 token → 提取重要信息到 Layer 2
# Layer 2 (长期记忆): 摘要 → 蒸馏 → MEMORY.md (100:1 压缩比)
# Layer 3: MEMORY.md 人工维护 + OpenViking 向量索引
```

**压缩策略** (动态):
| Token 占比 | 压缩策略 | 信息损失 |
|-----------|----------|----------|
| < 70% | ❌ 不压缩 | 0% |
| 70% ~ 85% | ✨ lossless-claw 无损压缩 | ≈ 0% |
| > 85% | 🔄 递归压缩 (Layer 0 → Layer 1) | 低 |

**性能收益**:
- 100 轮对话: 节省 **85%** tokens
- 500 轮对话: 节省 **94%** tokens
- 1000 轮对话: 节省 **95%** tokens

**触发条件**:
- 对话轮数 > 30
- Token 使用 > 70%
- 时间间隔 > 2 小时

**代码位置**: `memory/context-optimizer/`

---

#### 问题 2: 记忆分散，无法跨会话检索

**症状**:
- 飞书交互的知识没有被记住
- 下次会话丢失上下文
- 重要决策和经验无法跨会话复用

**根因分析**:
- 飞书会话和 webchat 会话独立
- 没有统一的记忆存储
- 没有自动同步机制

**解决方案: 飞书记忆实时同步系统**

```
飞书消息 → 自动抓取 → 存储到 memory/feishu/YYYY-MM-DD.md →
  → 语义索引 → OpenViking → Memos 双向同步
```

**同步流程**:
- 每次心跳自动同步当天飞书消息
- 训练相关对话自动识别 → 存储到教练 Agent 私有记忆
- 重要技术讨论 → 提炼到 MEMORY.md

**代码位置**: `memory/scripts/feishu_memory_store.py`

---

### 1.2 Agent Team 系统问题

#### 问题 1: Team vs Subagent 混淆

**历史教训**: 三次启动团队任务全部错误

**根因**:
- 没有真正理解 Agent Team 和普通 subagent 的区别
- 偷懒跳过了读取 `teams.json` 配置的步骤
- 错误地使用了默认模型而不是团队配置的模型

**关键区别**:

| 特性 | 普通 subagent | Agent Team |
|------|-------------|------------|
| 本质 | 单个临时 worker | 有组织的多人协作团队 |
| 模型 | 默认模型 (zai/glm-5-turbo) | `teams.json` 中配置 |
| 角色分工 | 无 | architect/developer/researcher |
| 记忆空间 | 无 | 团队私有 `memory_namespace` |
| 协作模式 | 独立工作 | hierarchical/parallel |

**正确激活流程**:
```
1. 读取 `memory/agent_teams/teams.json` → 获取完整团队配置
2. 按团队结构分配任务 → 每个成员启动 subagent
3. 使用团队配置模型 → 不是默认模型
4. 协作完成任务 → 收集结果汇总
```

---

#### 问题 2: 柱子哥 vs Claude Code 混淆

**症状**: 多次把柱子哥和 Claude Code 混为一谈

**澄清**:

| 身份 | 柱子哥 | Claude Code |
|------|-----------|-------------|
| 本质 | OpenClaw 内部 subagent | 外部 CLI 工具 |
| 运行方式 | `sessions_spawn` | `claude --print ...` |
| 模型 | `volcengine-plan/ark-code-latest` | `claude-opus-4-6` (via API) |
| 上下文 | ✅ 可访问 OpenClaw 记忆 | ❌ 需要手动注入 |
| 知识库 | 74MB Kenneth Lee 思想克隆 | 无 |
| 核心职责 | 方案设计、技术分析、架构审查 | 编码实现 |
| 调用方式 | `sessions_spawn(cwd="/home/huawei/.openclaw/workspace/zhuzige/")` | 命令行调用 |

**重要**: 柱子哥必须指定 cwd 才能加载知识库，否则"裸跑"失去 Kenneth Lee 知识体系支撑。

---

### 1.3 飞书连接问题

#### 问题: 飞书消息无回复

**排查流程** (按优先级，从高到低):

| 步骤 | 检查项 | 命令 | 修复方案 |
|------|--------|------|----------|
| **0** | 检查大模型配额 | 看日志 `Weekly/Monthly Limit Exhausted` | 切换默认模型到 `volcengine-plan/ark-code-latest` |
| **1** | 检查日志错误 | `tail -100 /tmp/openclaw/openclaw-*.log \| grep -i "feishu\|error\|fail"` | 看具体错误信息 |
| **2** | 检查模型 fallback 循环 | 日志大量 `model_fallback_decision` | 确保 `agents.defaults.model.primary` 带 provider 前缀 |
| **3** | 检查 dmPolicy | `grep dmPolicy ~/.openclaw/openclaw.json` | 必须设为 `"open"`，否则拦截消息 |
| **4** | 检查 WebSocket | DNS 解析失败/断开 | 重启 gateway 重建连接 |

**关键配置** (必须正确):
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "zai/glm-5-turbo"  // ✅ 必须带 provider 前缀
      }
    }
  },
  "channels": {
    "feishu": {
      "accounts": {
        "main": {
          "dmPolicy": "open"  // ✅ 必须打开
        }
      }
    }
  }
}
```

**永久修复已经生效**:
- ✅ `agents.defaults.model.primary` = `zai/glm-5-turbo`
- ✅ `channels.feishu.accounts.main.dmPolicy` = `"open"`
- ✅ 火山 provider 配置正确

---

## 二、记忆系统架构

### 2.1 三层记忆架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 长期记忆                                            │
│  文件: MEMORY.md + OpenViking/memories/user/                 │
│  模型: zai/glm-5                                             │
│  预算: 低                                                    │
│  内容: 个人偏好、行为模式、进化记录                          │
│  加载: 每次会话启动                                          │
│  更新: 重要事件后                                            │
└─────────────────────────────────────────────────────────────┘
                            ↑ 蒸馏/沉淀 (OpenViking 自动压缩)
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 短期记忆                                            │
│  文件: memory/*.md + OpenViking/memories/                     │
│  模型: qwen3.5-plus                                          │
│  上下文: 977k                                                │
│  内容: 项目上下文、任务历史、语义搜索                          │
│  加载: 相关任务时                                            │
│  更新: 每日                                                   │
│  保留: 30天                                                  │
│  OpenViking: memories/sessions/ + memories/projects/          │
└─────────────────────────────────────────────────────────────┘
                            ↑ 压缩/提取
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 工作记忆                                            │
│  存储: agents.db + session.json                               │
│  模型: qwen3-max                                             │
│  推理: 强                                                    │
│  内容: 当前会话、Agent状态、中断恢复                             │
│  加载: 会话级                                                │
│  更新: 实时                                                   │
│  持久化: 会话结束时                                          │
└─────────────────────────────────────────────────────────────┘
```

**设计理念**:
- **按需加载**: 不一次加载所有记忆，节省 token
- **层级过滤**: 不重要信息下沉，重要信息上浮
- **自动压缩**: token 超量自动触发压缩

---

### 2.2 OpenViking (Context Database)

**核心特性**:
1. **文件系统范式**: 用"目录/文件"方式统一管理记忆、资源、技能
   - 你可以像操作文件系统一样操作记忆
   - 目录定位 → 文件读取，比向量数据库更直观

2. **分层加载**:
   - Layer 3 长期: 只加载摘要
   - Layer 2 短期: 相关项目加载全文
   - Layer 1 当前: 完整加载

3. **递归检索**:
   - 先目录定位
   - 再文件内语义搜索
   - 比全局向量搜索更精准

4. **可视化轨迹**: 检索过程可观察、可调试

**存储路径**:
```
memory/openviking/
├── memories/
│   ├── user/           # 用户偏好、长期记忆
│   ├── sessions/       # 会话记录
│   └── projects/       # 项目相关记忆
├── resources/
│   ├── projects/       # 项目资源
│   ├── papers/         # 论文资源
│   └── knowledge/      # 知识库
└── skills/             # 技能文件
```

---

### 2.3 SynapMind (突触记忆系统)

**核心设计思想**: 神经科学启发，模拟人脑突触可塑性。

**核心模块**:

| 模块 | 功能 | 对应神经科学 |
|------|------|-------------|
| **权重管理** | LTP/LTD 动态调整记忆权重 | 长时程增强/长时程抑制 |
| **扩散检索** | 联想记忆多跳推理 | 扩散激活 |
| **语义编码** | 深度加工建立关联 | 语义编码 |
| **元启发突变** | 保持记忆多样性 | 遗传变异/进化 |
| **隔离存储** | 多 Agent 完全隔离 | 功能分区 |

**权重公式**:
```
W = W0 × [1 + 访问×0.1] × 衰减 × 优先级
```
- 多次访问 → 权重增强 (LTP)
- 长期不访问 → 权重衰减 (LTD)
- 重要信息 → 高优先级权重

**代码位置**: `memory/synapmind/`

---

## 三、进化系统架构

### 3.1 三层进化机制

| 层级 | 触发时间 | 核心功能 | 执行者 |
|------|----------|----------|--------|
| **每日进化** | 每天 23:00 | 总结当日任务、提取新知识、更新索引 | 自动 |
| **每周进化** | 每周日 10:00 | 行为模式分析、记忆蒸馏、去重 | 自动 |
| **每月进化** | 每月最后一天 | 长期记忆优化、低权重清理 | 自动 + 人工 |

**触发流程**:
- 集成在 HEARTBEAT.md 中，每次心跳自动检查是否需要执行
- 到点自动执行，不需要人工干预
- 进化记录保存到 `memory/evolution/logs/`

---

### 3.2 灵感引擎 (隔离版)

**核心特性**:
- 6 种创新策略:
  1. **combination**: 跨领域组合
  2. **counterfactual**: 如果反过来会怎样？
  3. **analogy**: 类比迁移
  4. **scamper**: 替代/组合/适应/修改/转换/消除/反向
  5. **cross_domain**: 跨领域知识迁移
  6. **constraint_removal**: 移除约束想创新

- **每个 Agent 独立存储**:
  - 教练 Agent: 训练灵感独立存储
  - 主 Agent: 技术灵感独立存储
  - 完全隔离，互不干扰

- **启发式规则提取**: 成功灵感自动提取规则，下次类似场景自动触发

**生成流程**:
```
每次心跳 → 检查今日是否生成 → 如果没有 → 为每个 Agent 生成 → 保存到隔离存储
```

**输出位置**: `memory/inspiration-reports/`

---

### 3.3 SynapMind 权重管理

**心跳同步**:
- 每次心跳检查权重衰减
- 海马体回放: 高权重记忆随机回放，增强固化
- 低权重记忆清理建议

**代码位置**: `memory/synapmind/weights/weight_manager.py`

---

## 四、Agent Team 系统架构

### 4.1 团队类型划分

按项目成熟度分为三类:

| 分类 | 团队 | 核心任务 | 状态 |
|------|------|----------|------|
| **产品化项目组** | 灵码 | Sci-Orbit 参数补全服务开发 | ✅ 活跃 |
| | 天权-HPC | PTO-Gromacs 优化 | ⏸️ 暂停 |
| | 天渊 | 神经网络算子 MHF 优化 | ✅ 活跃 |
| **研究探索组** | 算子框架 | 深度学习算子优化论文学习 | 🔄 进行中 |
| | 北斗 | CPU 内存优化 | 🌙 休息 |
| | 太微 | AI OS 研究 | 🌙 休息 |
| | 玄武 | 游戏引擎优化 | 🌙 休息 |
| **暂停归档** | 天玑量化 | A 股量化分析 | ⏸️ 暂停 |

---

### 4.2 团队结构设计

每个团队在 `teams.json` 中的配置示例:

```json
{
  "team_id": "team_tianyuan_fft",
  "name": "天渊团队",
  "project_id": "proj_tianyuan_fft",
  "agents": [
    {
      "agent_id": "team_tianyuan_fft_tianyuan",
      "role": "architect",
      "skills": ["系统设计", "技术选型", "架构规划", "方案审查"],
      "model": "volcengine-plan/glm-4.7",
      "thinking": "on",
      "memory_namespace": "team_memory/team_tianyuan_fft/tianyuan",
      "inherit_from_main": true
    },
    {
      "agent_id": "team_tianyuan_fft_tianqujing",
      "role": "developer",
      "skills": ["编码", "实现", "调试", "代码优化"],
      "model": "volcengine-plan/glm-4.7",
      "thinking": "on",
      "memory_namespace": "team_memory/team_tianyuan_fft/tianqujing",
      "inherit_from_main": true
    },
    {
      "agent_id": "team_tianyuan_fft_tianchi",
      "role": "researcher",
      "skills": ["论文研读", "技术调研", "方案分析", "实验设计"],
      "model": "volcengine-plan/glm-4.7",
      "thinking": "on",
      "memory_namespace": "team_memory/team_tianyuan_fft/tianchi",
      "inherit_from_main": true
    }
  ],
  "leader_role": "architect",
  "collaboration_mode": "hierarchical",
  "memory_namespace": "team_memory/team_tianyuan_fft",
  "status": "active"
}
```

**设计要点**:
- 每个成员有独立的记忆命名空间
- 角色明确分工
- 支持不同协作模式 (`hierarchical`/`parallel`)
- 继承主上下文

---

### 4.3 隔离控制架构

```
主Agent (西西)
    │
    ├── ✅ 可以控制所有子Agent和Team
    │      - 分配任务
    │      - 发送命令
    │      - 获取执行结果
    │
    └── ❌ 不能访问其他Agent的记忆
           - 子Agent记忆隔离
           - Team记忆隔离
```

**权限矩阵**:

| 请求者 → 目标 | 西西 | 教练 | 天玑 |
|--------------|------|-------|------|
| 西西 | - | ✅ 控制 | ✅ 控制 |
| 教练 | ❌ | - | ❌ |
| 天玑 | ❌ | ❌ | - |

**隔离存储结构**:
```
memory/synapmind/production_memory/
├── agents/
│   ├── agent_main_xixi/           # 主Agent (西西)
│   │   └── memories.json
│   └── agent_coach/             # 子Agent (教练)
│       └── memories.json
└── agent_teams/
    └── team_tianyuan_fft/       # 天渊团队
        └── memories.json
```

**违规记录**:
- 所有跨 Agent 记忆访问尝试会被记录到 `memory/agent-isolation/violations.jsonl`
- 用于调试，不影响运行

---

### 4.4 正确激活流程

```
用户请求"天渊团队做XX"
    ↓
1. 读取 `memory/agent_teams/teams.json`
    ↓
2. 获取团队配置 (成员/角色/模型/协作模式)
    ↓
3. 按角色逐个启动 subagent
    ↓
4. 分配任务 (architect 负责方案设计 → developer 负责实现 → researcher 负责实验)
    ↓
5. 收集结果 → 汇总 → 交付用户
```

**常见错误**:
- ❌ 不要用 Claude CLI 启动团队任务（脱离 OpenClaw 上下文）
- ❌ 不要用普通 subagent 冒充 Agent Team
- ❌ 不要跳过读取 `teams.json`

---

## 五、关键经验总结

### 5.1 必记规则表

| 优先级 | 规则 | 重要性 |
|--------|------|--------|
| ★★★ | Agent Team ≠ 普通 subagent，启动必须读 teams.json | 启动错了就是错，重来 |
| ★★★ | 柱子哥 = OpenClaw subagent，Claude Code = 外部 CLI | 调用方式完全不同 |
| ★★★ | `volcengine-plan` 模型 **没有后缀**，带后缀会报错 | `volcengine-plan/glm-4.7` ✅ |
| ★★★ | 飞书无回复，**第一步检查配额** | Weekly Limit Exhausted 是当前最常见问题 |
| ★★ | 敏感外部操作必须问用户 | 删除/推送/发送都要确认 |
| ★★ | 重要结论必须写文件 | 脑内记忆不持久，文件才持久 |

---

### 5.2 双写双审对抗编码范式

**核心思路**: OpenClaw 和 Claude Code 对同一任务产出两份代码，交叉审查，分歧发现漏洞，OpenClaw 是最终裁决者。

**完整流程**:

```
Step 1: 准备工作区
mkdir -p /tmp/dual-task/{claude,openclaw,review,final}
    ↓
Step 2: 并行双写
- Claude Code: exec background → 写代码
- OpenClaw: 自己写代码到 /tmp/dual-task/openclaw/
    ↓
Step 3: 交叉审查
- Claude Code 审查 OpenClaw 代码 → 输出 JSON 评分/bugs/suggestions/verdict
- OpenClaw 审查 Claude Code 代码 → 输出 JSON 评分/bugs/suggestions/verdict
    ↓
Step 4: OpenClaw 裁决合并
- 双 pass → 选分数高的
- 一 pass 一 fix → 选 pass，合并 fix 建议
- 双 fix → OpenClaw 基于 review 重写
- 输出到 /tmp/dual-task/final/
```

**评分/裁决格式**:
```json
{
  "score": 8,
  "bugs": [],
  "suggestions": ["建议..."],
  "verdict": "pass/fix/fail"
}
```

**适合场景**:
- ✅ 核心算法
- ✅ 安全代码
- ✅ 架构级改动 (>50 行)

**不适合场景**:
- ❌ 简单 CRUD
- ❌ 配置文件修改
- ❌ 文档编辑

**代码位置**: `agent-collaboration/packages/adversarial/`

---

### 5.3 Claude Code 调用规范

**正确姿势 (6 个关键变量，缺一不可)**:

```bash
cd /tmp/claude-workdir && \
CLAUDECODE= claude --bare -p "任务描述" \
  --output-format json \
  --allowedTools "Read,Edit,Write,Bash" \
  --max-turns 10 \
  --max-budget-usd 2.00 \
  --permission-mode bypassPermissions
```

**每个变量的作用**:

| 变量 | 作用 |
|------|------|
| `CLAUDECODE=` | 清除环境变量，避免嵌套 session 检测失败 |
| `--bare` | 跳过 hooks/plugins/MCP servers，干净环境 |
| `--output-format json` | ⭐ **最关键**，非交互模式必须，否则无输出 |
| `--allowedTools` | 限制工具范围，避免 Claude 做太多操作卡死 |
| `--max-turns` | 限制最大推理轮数，防止死循环 |
| `--permission-mode bypassPermissions` | 绕过确认对话框，自动运行 |

**工作目录限制**:
- ✅ `/tmp/` 下正常工作
- ❌ `~/.openclaw/workspace` 下卡死 (3.2GB 目录扫描卡住)
- 解决办法: `--add-dir /path/to/source` 添加需要访问的源码目录

**多轮会话管理**:
```bash
# 创建会话
session_id=$(claude --bare -p "第一步" --output-format json ... | jq -r '.session_id')

# 恢复会话继续
claude --bare -p "第二步" --resume "$session_id" --output-format json ...
```

---

## 六、项目成果展示

### 6.1 天渊团队: MHF-NeuralOperator

**项目目标**: 对 neuraloperator 库中 11 个算子做 MHF (Multi-Resolution Hierarchical Factorization) 优化，减少参数量，保持精度。

**CoDA 实验结果** (Darcy 32x32 真实数据集):

| 模型 | 参数量 | L2 Loss | 精度变化 |
|------|--------|---------|----------|
| Baseline TFNO | 27,841 | 0.018394 | baseline |
| MHF | 27,841 | 0.018418 | +0.13% |
| **MHF+CoDA** | **28,960** | **0.015217** | **-17.3% ↓** |

**结论**: CoDA (Cross-head Attention) 只增加 4% 参数，却带来 17.3% 精度提升。

**昨日优化成果** (推理时延优化):

| CoDA 实现 | 参数量 | 模块延迟 | 优化幅度 |
|----------|--------|----------|----------|
| 原始实现 | 1,253 | 0.151 ms | 0% |
| **轻量化 SE 风格** | **512** | 0.037 ms | **75.7% ↓** |
| **+ torch.compile** | **981** | 0.001 ms | **99.5% ↓** |

**完整模型预计延迟**:
- MHF + 原始 CoDA: 1.05 ms
- MHF + 优化 CoDA: **~0.44 ms** (58.1% 改善)

**代码位置**: `/home/huawei/Desktop/home/xuefenghao/workspace/MHF-NeuralOperator/`

**GitHub**: https://github.com/xuefenghao5121/mhf-neuraloperator

---

### 6.2 灵码团队: Sci-Orbit

**项目定位**: AI for Science MCP Server，论文→实验→部署全流程工具链。

**关键里程碑**:

**v0.5.0 (2026-04-02)**:
- 工具精简: 从 48 个 → 17 个 (减少 65%)
- 只保留 Claude Code 做不到的领域特定能力
- 新增 `openclaw-plugin` 直接集成到 OpenClaw

**v0.6.0 (2026-04-06)**:
- ✅ 参数补全服务 V2 重构完成 (分层推断架构)
- ✅ 1189 行完整重写
- ✅ 支持 5 种科学计算工具: VASP/ABACUS/QE/CP2K/GROMACS
- ✅ 分层推断: 环境推断 → 任务推断 → 规则推断 → 默认回退

**当前状态**:
- 测试通过: 104/104 (100%)
- 已推送到 GitHub feature 分支
- 等待 npm 发布

**GitHub**: https://github.com/xuefenghao5121/Sci-Orbit

---

## 七、常见错误速查

| 错误 | 根因 | 解决方案 |
|------|------|----------|
| `model 400 Bad Request` | 模型格式错误 | `volcengine-plan` 模型 **没有后缀** → 检查 `teams.json` |
| 团队任务启动后没有响应 | 没有读 `teams.json`，用了默认模型 | 删除重新按正确流程启动 |
| `Context overflow` | token 累积过多 | 等待心跳自动压缩，或手动运行 `python memory/context-optimizer/auto_compress.py --force` |
| Claude Code 运行卡住 | 工作目录太大 | 必须在 `/tmp/` 运行，用 `--add-dir` 添加源码 |
| Claude Code 没有输出 | 忘记加 `--output-format json` | 非交互模式必须指定 |
| 飞书没有回复 | 配额耗尽/模型配置错/dmPolicy错 | 按排查流程检查 |

---

## 八、下一步规划

### 产品化项目组
- **灵码**: v0.6.0 npm 发布
- **天权-HPC**: ARM 超算环境验证 PTO 优化
- **天渊**: 完成真实数据集推理时延优化测试 → 合并主线

### 研究探索组
- **算子框架**: 深度学习算子优化论文系统性学习
- **北斗**: CPU 内存池化运行时优化
- **太微**: AI OS 架构研究

---

## 附录

### 关于本文

本文档基于鸡你太美团队**一个月实际项目经验**总结，记录了:
- 我们遇到的问题
- 问题的根因分析
- 验证过的解决方案
- 最终架构设计

欢迎参考，持续更新。

**文档版本**: v1.1.0
**最后更新**: 2026-04-07
**维护者**: 西西 (OpenClaw 主 Agent)
