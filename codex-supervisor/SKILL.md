# Codex Supervisor

## ⚠️ CRITICAL EXECUTION RULE

**WHEN THIS SKILL IS INVOKED, YOU MUST IMMEDIATELY:**

1. **STOP** - Do NOT execute any tasks in the main process
2. **DELEGATE** - Use `mcp__codex__codex` tool to launch Codex sub-agent
3. **MONITOR** - Only check progress periodically, do NOT consume main context

**IF YOU START CREATING SCRIPTS, RUNNING COMMANDS, OR EXECUTING TASKS IN MAIN PROCESS:**
- ❌ YOU ARE DOING IT WRONG
- ❌ Main process token will explode
- ❌ This violates the entire purpose of this skill

## ⚠️ CRITICAL MODEL CONFIGURATION

**ENSURE CONFIG.TOML IS PROPERLY CONFIGURED:**

Before using this skill, verify that `~/.codex/config.toml` contains the following settings:

```toml
model = "gpt-5.4"
fast_mode = true
model_reasoning_effort = "xhigh"
approval_policy = "never"
sandbox_mode = "danger-full-access"
```

**Why these settings are critical:**
- `model_reasoning_effort = "xhigh"`: ALL tasks benefit from maximum reasoning capability
- `fast_mode = true`: Accelerates response time without sacrificing quality
- `approval_policy = "never"`: Allows sub-agent to execute commands without blocking
- `sandbox_mode = "danger-full-access"`: Gives sub-agent permission to modify files

**These settings apply globally to all Codex calls** - you don't need to pass them in each MCP invocation.

**Correct MCP invocation:**
```json
{
  "prompt": "...",
  "cwd": "/path/to/working/directory"
}
```

**Note**: All settings (model, sandbox, xhigh, fast_mode, approval_policy) from config.toml apply automatically.

## Purpose

Orchestrate complex, multi-step tasks by delegating execution to Codex sub-agents while maintaining supervision and preventing context explosion in the main process.

## When to Use

- Complex research or implementation tasks requiring multiple steps
- Tasks that would consume excessive context in the main conversation
- Workflows requiring iterative refinement with acceptance criteria
- Long-running experiments or reproductions that need isolation

## Core Pattern

**Main Process (Claude Code)**: Supervises progress, validates results, makes decisions
**Sub-Agent (Codex)**: Executes specific tasks in isolated context, reports completion

## Workflow

1. **Task Decomposition**
   - Break down the user's request into discrete subtasks
   - Create a todo list with clear acceptance criteria
   - Identify dependencies between tasks

2. **Sub-Agent Delegation**
   - Launch Codex sub-agent via MCP for each subtask
   - Provide clear instructions and context
   - Sub-agent works in isolated context

3. **Progress Monitoring**
   - **CRITICAL**: Main process MUST NOT stop working while waiting
   - Set up periodic status checks (every 30-60 seconds)
   - Use TaskOutput tool to query sub-agent progress
   - Validate results against acceptance criteria
   - Track overall progress
   - Continue monitoring until explicit completion signal

4. **Iterative Refinement**
   - If acceptance criteria not met, analyze failure
   - Create new todo list for improvements
   - Re-delegate to new Codex sub-agent

5. **Completion**
   - All tasks completed and validated
   - Acceptance criteria met
   - Report final status to user

## Implementation Steps

### MANDATORY FIRST STEP: Delegate to Codex

**Before doing ANYTHING else, you MUST:**

```
Use mcp__codex__codex tool with:
- prompt: Clear task description with acceptance criteria
- cwd: Working directory
- model: (omit to use config.toml default)
- sandbox: (omit to use config.toml default)

Note: Global settings (xhigh, fast_mode, approval_policy) from config.toml apply automatically
```

**DO NOT:**
- Create scripts in main process
- Run monitoring loops in main process
- Execute any task logic in main process
- Load large outputs into main context

### Step 1: Analyze Request
```markdown
- Understand the user's goal
- Identify acceptance criteria
- Assess task complexity
- Determine if delegation is appropriate
```

### Step 2: Create Task List
```markdown
Use TaskCreate to build todo list:
- Break down into atomic subtasks
- Define clear acceptance criteria for each
- Establish dependencies
- Prioritize by execution order
```

### Step 3: Delegate to Codex
```markdown
For each task:
1. Use mcp__codex__codex tool with:
   - Clear prompt describing the subtask
   - Relevant context (file paths, requirements)
   - Acceptance criteria
   - Working directory (cwd parameter)

2. Codex sub-agent executes in isolation:
   - Has own context window
   - Can use shell commands, read/write files
   - Reports completion with results

3. Main process ACTIVELY monitors (DO NOT WAIT PASSIVELY):
   - Set periodic check interval (30-60 seconds)
   - Use TaskOutput tool to query progress
   - Log status updates for user visibility
   - Continue until completion signal received
```

### Step 3.5: Active Monitoring Loop
```markdown
CRITICAL: Main process must implement active monitoring:

While sub-agent is running:
1. Wait 30-60 seconds
2. Use TaskOutput(task_id=codex_task_id, block=false) to check status
3. Report progress to user if available
4. If still running, repeat from step 1
5. If completed, proceed to validation

DO NOT:
- Simply wait for completion without checking
- Stop working while sub-agent executes
- Assume task will complete without monitoring
```

### Step 4: Validate Results
```markdown
- Check if acceptance criteria met
- Review Codex output
- Verify file changes if applicable
- Update task status (completed/needs-refinement)
```

### Step 5: Iterate or Complete
```markdown
If validation fails:
- Analyze root cause
- Create new subtasks for fixes
- Re-delegate to fresh Codex sub-agent

If validation passes:
- Mark task complete
- Move to next task
- Report progress to user
```

## Example Usage

### User Request
"Reproduce the AllMem experiment from the paper on the smallest dataset"

### Main Process Actions

1. **Task Decomposition**
```markdown
- [ ] Analyze paper and identify experiment parameters
- [ ] Locate smallest dataset in docs/
- [ ] Set up experiment environment
- [ ] Run experiment with paper parameters
- [ ] Validate results against paper metrics
```

2. **Delegate First Task**
```python
# Use mcp__codex__codex tool
{
  "prompt": "Analyze the AllMem paper documentation in docs/ and extract: 1) Main experiment parameters for qwen3-0.6B, 2) Dataset specifications, 3) Expected metrics. Report findings in structured format.",
  "cwd": "/path/to/hpc3/mount",
  
}
```

3. **Monitor Progress**
```markdown
Main process ACTIVELY monitors (DO NOT WAIT PASSIVELY):
- Set periodic check interval (30-60 seconds)
- Use TaskOutput tool to query progress
- Report status updates to user
- Does NOT execute the analysis itself
- Does NOT consume context with paper details
- Only receives structured findings when complete
```

4. **Validate & Continue**
```markdown
- Review Codex findings
- If incomplete: create new task for clarification
- If complete: mark task done, delegate next task
```

5. **Iterate Until Complete**
```markdown
Repeat delegation cycle until:
- All tasks completed
- Acceptance criteria met (experiment results match paper)
- User satisfied with outcome
```

## Key Principles

1. **Context Isolation**: Sub-agents work in separate context, preventing main process bloat
2. **Single Responsibility**: Main process supervises, sub-agents execute
3. **Clear Handoffs**: Each delegation has explicit input/output contract
4. **Validation Gates**: Results validated before proceeding
5. **Iterative Refinement**: Failed validations trigger new delegation cycles
6. **Active Monitoring**: Main process MUST continuously monitor, never stop working while waiting

## MCP Codex Tool Parameters

### Tool 1: mcp__codex__codex (启动新会话)

**用途**: 启动一个新的 Codex 子代理会话

**必需参数**:
- `prompt` (string): 清晰的任务描述，包含验收标准

**可选参数**:
- `cwd` (string): 工作目录路径（相对于服务器进程的当前目录）
- `model` (string): 模型名称，默认使用 `gpt-5.4`
- `config` (object): 配置选项 ⚠️ **必须包含 model_reasoning_effort: "xhigh"**
  - `approval-policy`: 审批策略
    - `"untrusted"`: 不信任的命令需要审批
    - `"on-failure"`: 失败时需要审批（推荐）
    - `"on-request"`: 请求时审批
    - `"never"`: 从不审批
  - `model_reasoning_effort`: 推理级别 ⚠️ **强制使用 "xhigh"**
    - `"xhigh"`: 超高推理能力（必需，用于复杂任务）
    - ❌ 不要使用 "medium" 或 "low"，会导致任务失败
- `base-instructions` (string): 替代默认指令的自定义指令
- `developer-instructions` (string): 作为开发者角色消息注入的指令

**使用示例**:
```json
{
  "prompt": "监控训练进程 PID 1361157，每 2 分钟检查一次日志，报告进度和任何错误",
  "cwd": "/Users/h1syu1/hpc3_mount",
  
  
}
```

**Note**: Global config.toml settings (xhigh, fast_mode, approval_policy) apply automatically.

**返回值**:
- 包含 `threadId` 的响应，用于后续的 `codex-reply` 调用
- Codex 子代理的执行结果

### Tool 2: mcp__codex__codex-reply (继续现有会话)

**用途**: 在现有的 Codex 会话中继续对话

**必需参数**:
- `threadId` (string): 之前 Codex 会话返回的线程 ID
- `prompt` (string): 新的指令或问题

**使用示例**:
```json
{
  "threadId": "previous-thread-id-from-codex-response",
  "prompt": "训练是否完成？如果完成，报告最终指标；如果失败，分析错误原因"
}
```

**使用场景**:
- 需要向同一个 Codex 子代理发送后续指令
- 子代理需要基于之前的上下文继续工作
- 迭代改进：根据验证结果要求子代理调整

### 工具使用流程

**单次任务**:
```
1. 使用 mcp__codex__codex 启动子代理
2. 子代理执行任务并返回结果
3. 主进程验证结果
```

**多轮迭代**:
```
1. 使用 mcp__codex__codex 启动子代理（获得 threadId）
2. 子代理执行第一步并返回结果
3. 主进程验证，发现需要调整
4. 使用 mcp__codex__codex-reply(threadId, new_prompt) 继续
5. 重复步骤 2-4 直到验收标准达成
```

### 完整示例

**启动监控任务**:
```json
{
  "prompt": "你是 Codex 子代理，负责持续监控训练任务。\n\n任务：\n1. 检查进程 PID 1361157 是否运行\n2. 每 2 分钟读取最新的训练日志\n3. 提取关键指标（loss, accuracy, epoch）\n4. 如果训练完成或失败，立即报告\n5. 持续监控直到我告诉你停止\n\n工作目录：/Users/h1syu1/hpc3_mount\n验收标准：能够实时报告训练状态和指标",
  "cwd": "/Users/h1syu1/hpc3_mount",
  
  
  "config": {
    "approval-policy": "on-failure",
    "model_reasoning_effort": "xhigh",
    "fast_mode": true
  }
}
```

**继续对话**:
```json
{
  "threadId": "abc123-thread-id",
  "prompt": "训练已经运行了 30 分钟，当前指标如何？是否需要调整学习率？"
}
```

## Continuation Pattern

### 使用 mcp__codex__codex-reply 进行多轮对话

当需要与同一个 Codex 子代理进行多轮交互时：

1. **保存 threadId**: 首次调用 `mcp__codex__codex` 后，从响应中提取 `threadId`
2. **后续调用**: 使用 `mcp__codex__codex-reply` 并传入 `threadId`
3. **上下文保持**: 子代理会记住之前的对话内容

**示例场景 - 迭代改进**:
```
第 1 轮：
mcp__codex__codex({
  prompt: "运行实验并报告结果",
  ...
})
→ 返回 threadId: "thread-001"

第 2 轮（如果结果不理想）：
mcp__codex__codex-reply({
  threadId: "thread-001",
  prompt: "结果显示 accuracy 只有 60%，请调整超参数并重新运行"
})

第 3 轮（继续改进）：
mcp__codex__codex-reply({
  threadId: "thread-001",
  prompt: "现在 accuracy 达到 75%，继续优化直到达到 80%"
})
```

**何时使用新会话 vs 继续会话**:
- ✅ 使用 `codex-reply` (同一 threadId): 任务相关，需要上下文
- ✅ 使用新的 `codex` 会话: 完全独立的新任务，不需要之前的上下文

## Anti-Patterns

❌ **Don't**: Execute tasks yourself that should be delegated
❌ **Don't**: Load large files/data into main context
❌ **Don't**: Run experiments directly in main process
❌ **Don't**: Keep sub-agent context alive unnecessarily
❌ **Don't**: Passively wait for completion without monitoring
❌ **Don't**: Stop working while sub-agent executes

✅ **Do**: Delegate execution to Codex sub-agents
✅ **Do**: Receive only summary results in main process
✅ **Do**: Validate before proceeding
✅ **Do**: Create fresh sub-agents for new tasks
✅ **Do**: Actively monitor with periodic status checks
✅ **Do**: Continue working and reporting progress

## Success Criteria

- Main process context remains manageable (<30% of limit)
- Tasks completed in isolated sub-agent contexts
- Clear audit trail of delegations and validations
- Acceptance criteria met
- User can follow progress without context overload

## Notes

- Sub-agents are ephemeral: created per task, destroyed after completion
- Main process maintains task list and validation logic
- Codex sub-agents have full tool access within their sandbox
- Use `threadId` for multi-turn conversations with same sub-agent
- Balance between task granularity and delegation overhead
