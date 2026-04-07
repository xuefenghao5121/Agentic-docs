# OpenClaw Agentic 工作分享

> 基于鸡你太美团队真实项目经验总结
> 更新时间: 2026-04-07

---

## 一、我们遇到的问题与解决方案

### 1.1 记忆系统问题

#### 问题 1: Context Overflow (上下文溢出)
**症状**: GLM-5 频繁出现 context overflow，回复中断

**根因**: 
- 对话轮数增加后，token 累积超出限制
- 飞书和 webchat 会话独立，上下文无法共享

**解决方案**: 三层递归压缩机制

```python
# Layer 0 → Layer 1: 完整对话 → 摘要 (10:1)
# Layer 1 → Layer 2: 摘要 → 长期记忆 (100:1)
# Layer 2 → Layer 3: 长期记忆 → MEMORY.md (1000:1)
```

**性能收益**:
- 100 轮对话: 节省 85% tokens
- 500 轮对话: 节省 94% tokens
- 1000 轮对话: 节省 95% tokens

#### 问题 2: 记忆分散，无法跨会话检索
**症状**: 飞书交互的知识没有被实时记住，下次会话丢失

**解决方案**: 飞书记忆实时同步系统

```
飞书消息 → 本地记忆 (feishu/YYYY-MM-DD.md) → Memos 双向同步
```

---

### 1.2 Agent Team 系统问题

#### 问题 1: Team vs Subagent 混淆
**症状**: 三次启动团队任务全部错误

**根因**: 
- 把 Agent Team 当成普通 subagent 使用
- 没有读取 teams.json 获取团队配置
- 用了错误的模型和启动方式

**教训**: Agent Team ≠ 普通 subagent

| | 普通 subagent | Agent Team |
|--|--|--|
| 本质 | 单个临时 worker | 有组织的多人团队 |
| 模型 | 默认模型 | teams.json 中配置 |
| 角色 | 无 | architect/developer/researcher |
| 记忆 | 无 | 团队私有 memory_namespace |

**正确流程**:
```
1. 读取 teams.json → 获取团队配置
2. 按团队结构启动 → 携带完整团队上下文
3. 使用团队配置模型 → 不是默认模型
```

#### 问题 2: 柱子哥 vs Claude Code 混淆
**症状**: 多次把柱子哥和 Claude Code 混为一谈

**柱子哥 = OpenClaw 首席架构师 subagent**
- 本质: sessions_spawn 启动的内部 agent
- 模型: volcengine-plan/ark-code-latest
- 知识库: 74MB Kenneth Lee 思想克隆

**Claude Code = 外部 CLI 工具**
- 本质: claude 命令行工具
- 调用: `claude --print --permission-mode bypassPermissions`
- 需要 --output-format json 等参数

---

### 1.3 飞书连接问题

#### 问题: 飞书消息无回复
**排查流程** (按优先级):

```
第 0 步: 检查大模型配额 (Weekly/Monthly Limit Exhausted)
     ↓
第 1 步: 检查日志错误
     ↓
第 2 步: 检查模型 fallback 循环
     ↓
第 3 步: 检查 dmPolicy 配置
     ↓
第 4 步: 检查 WebSocket 连接
```

**关键配置**:
- `agents.defaults.model.primary` = `zai/glm-5-turbo` (带前缀)
- `channels.feishu.accounts.main.dmPolicy` = `"open"`
- 火山 provider 配置正确 (`volcengine-plan`)

---

## 二、记忆系统架构

### 2.1 三层记忆架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 长期记忆 (MEMORY.md + OpenViking/memories/user/) │
│  模型: zai/glm-5 | 预算: 低 | 性能: 高                     │
│  内容: 个人偏好、行为模式、进化记录                          │
│  加载: 每次启动 | 更新: 重要事件后                          │
└─────────────────────────────────────────────────────────────┘
                            ↑ 蒸馏/沉淀
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 短期记忆 (memory/*.md + OpenViking/memories/)     │
│  模型: qwen3.5-plus | 上下文: 977k                         │
│  内容: 项目上下文、任务历史、语义搜索                        │
│  加载: 相关时 | 更新: 每日 | 保留: 30天                      │
└─────────────────────────────────────────────────────────────┘
                            ↑ 压缩/提取
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 工作记忆 (agents.db + session.json)               │
│  模型: qwen3-max | 推理: 强                                │
│  内容: 当前会话、Agent状态、中断恢复                         │
│  加载: 会话级 | 更新: 实时 | 持久化: 会话结束时              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 OpenViking (Context Database)

**核心特性**:
- 文件系统范式统一管理记忆
- 分层加载节省 Token
- 递归检索更精准
- 可视化检索轨迹

### 2.3 SynapMind (突触记忆系统)

**核心模块**:
- 权重管理: LTP/LTD 模拟突触可塑性
- 扩散检索: 联想记忆多跳推理
- 语义编码: 深度加工建立关联
- 元启发突变: 保持记忆多样性

---

## 三、进化系统架构

### 3.1 三层进化机制

| 层级 | 触发时间 | 核心功能 |
|------|----------|----------|
| 每日进化 | 23:00 | 总结、知识提取、索引更新 |
| 每周进化 | 周日 10:00 | 模式分析、记忆蒸馏 |
| 每月进化 | 每月最后一天 | 长期记忆优化 |

### 3.2 灵感引擎 (隔离版)

**核心特性**:
- 6种创新策略 (combination/counterfactual/analogy/scamper/cross_domain/constraint_removal)
- 每个 Agent 独立存储灵感
- 启发式规则自动提取

**隔离设计**:
- 教练 Agent: `agent_2fe01859`
- 主 Agent: `agent_main_xixi`
- 完全隔离，互不访问

---

## 四、Agent Team 系统架构

### 4.1 团队类型

| 团队 | 职能 | 状态 |
|------|------|------|
| 天渊 | MHF-NeuralOperator 优化 | 🔄 进行中 |
| 灵码 | Sci-Orbit AI4S CLI | ✅ 已完成 |
| 天权-HPC | PTO-Gromacs 优化 | ⏸️ 暂停 |
| 天玑量化 | A股量化分析 | ⏸️ 暂停 |
| 北斗 | CPU 内存优化 | 🌙 休息中 |
| 太微 | AI OS 研究 | 🌙 休息中 |
| 玄武 | 游戏引擎优化 | 🌙 休息中 |

### 4.2 团队结构

```json
{
  "team_id": "team_tianyuan_fft",
  "name": "天渊团队",
  "agents": [
    {"role": "architect", "name": "天渊", "model": "glm-4.7"},
    {"role": "developer", "name": "天渠井", "model": "glm-4.7"},
    {"role": "researcher", "name": "天池", "model": "glm-4.7"}
  ],
  "collaboration_mode": "hierarchical"
}
```

### 4.3 隔离架构

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

---

## 五、关键经验总结

### 5.1 必记规则

| 优先级 | 规则 | 重要性 |
|--------|------|--------|
| ★★★ | Agent Team ≠ 普通 subagent | 启动团队任务必须读 teams.json |
| ★★★ | 柱子哥 = subagent, Claude Code = CLI | 调用方式完全不同 |
| ★★★ | 模型格式: volcengine-plan 无后缀 | 带日期后缀会报错 |
| ★★★ | 飞书问题先查配额 | Weekly Limit Exhausted 是第一排查项 |
| ★★ | 敏感操作要问用户 | 外部发送、删除等 |
| ★★ | 记忆要写文件 | 脑内记忆不持久 |

### 5.2 双写双审对抗编码范式

**核心思路**: OpenClaw 和 Claude Code 对同一任务产出两份代码，交叉审查

```
Step 1: 准备工作区
         ↓
Step 2: 并行双写 (Claude Code + OpenClaw)
         ↓
Step 3: 交叉审查 (双盲 review)
         ↓
Step 4: OpenClaw 裁决合并
```

**适合场景**: 核心算法、安全代码、架构级改动 (>50行)

### 5.3 Claude Code 调用规范

**6 个关键变量**:
1. `CLAUDECODE=` — 清除环境变量
2. `--bare` — 跳过 hooks/plugins
3. `--output-format json` — ⭐ 最关键
4. `--allowedTools` — 限制工具范围
5. `--max-turns` — 防止死循环
6. v2.1.81 — 推荐版本

**工作目录限制**:
- ✅ `/tmp/` 下正常工作
- ❌ `~/.openclaw/workspace` 下卡死 (3.2GB 目录扫描)

---

## 六、项目成果展示

### 6.1 天渊团队: MHF-NeuralOperator

**CoDA 实验结果** (Darcy 32x32):

| 模型 | 参数量 | L2 Loss | 精度变化 |
|------|--------|---------|----------|
| Baseline TFNO | 27,841 | 0.018394 | baseline |
| MHF | 27,841 | 0.018418 | +0.13% |
| **MHF+CoDA** | **28,960** | **0.015217** | **-17.3%** |

### 6.2 灵码团队: Sci-Orbit

**工具精简成果**:
- 工具数: 48 → 17 (减少 65%)
- 测试通过: 104/104 (100%)
- 核心模块: 参数补全 V2 (1189 行)

---

## 七、常见错误速查

| 错误 | 原因 | 解决 |
|------|------|------|
| 模型 API 400 | 模型格式错误 | volcengine-plan 无后缀 |
| 团队任务失败 | 未读 teams.json | 按正确流程启动 |
| Context overflow | token 累积过多 | 触发上下文压缩 |
| 飞书无回复 | 配额/配置/dmPolicy | 按排查流程检查 |
| Claude Code 卡死 | 工作目录太大 | 用 /tmp/ 指定目录 |

---

## 八、下一步规划

### 8.1 产品化项目组
- **灵码**: v0.6.0 npm 发布准备
- **天权-HPC**: ARM 超算环境验证
- **天渊**: CoDA 推理时延优化

### 8.2 研究探索组
- 算子框架: 深度学习论文学习
- 北斗: CPU 内存优化
- 太微: AI OS 架构研究

---

_文档版本: v1.0.0_
_基于 2026-04-07 前经验总结_
