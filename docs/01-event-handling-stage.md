# 事件处理阶段详解

事件处理阶段是整个 Claude Code Action 的"感知层"，负责解析 GitHub 事件、验证触发条件、校验权限，并选择合适的执行模式。本文档详细说明其工作原理、代码结构和数据流。

## 整体架构图

```mermaid
graph TD
    A["GitHub Webhook 事件"] --> B["GitHub Actions 触发"]
    B --> C["src/github/context.ts"]
    C --> D["解析事件上下文"]
    D --> E["src/modes/registry.ts"]
    E --> F["选择执行模式"]
    F --> G["src/github/validation/*"]
    G --> H["触发条件验证"]
    H --> I["权限校验"]
    I --> J["人员验证"]
    J --> K["模式特定处理"]
    K --> L["继续执行或跳过"]
```

## 事件类型分类图

```mermaid
graph LR
    subgraph "Entity Events"
        A1["issues"] --> A2["Issue 操作"]
        A3["issue_comment"] --> A4["Issue 评论"]
        A5["pull_request"] --> A6["PR 操作"]
        A7["pull_request_review"] --> A8["PR 评审"]
        A9["pull_request_review_comment"] --> A10["PR 评审评论"]
    end
    
    subgraph "Automation Events"
        B1["workflow_dispatch"] --> B2["手动触发"]
        B3["schedule"] --> B4["定时触发"]
    end
    
    subgraph "Event Processing"
        C1["parseGitHubContext()"] --> C2["类型识别"]
        C2 --> C3["上下文解析"]
        C3 --> C4["输入参数解析"]
    end
    
    A2 --> C1
    A4 --> C1
    A6 --> C1
    A8 --> C1
    A10 --> C1
    B2 --> C1
    B4 --> C1
```

## 触发条件验证流程

```mermaid
graph TD
    A["checkContainsTrigger()"] --> B{直接提示?}
    B -->|是| C["直接触发"]
    B -->|否| D{Issue 指派?}
    D -->|是| E["检查指派用户"]
    E --> F{匹配触发用户?}
    F -->|是| G["触发"]
    F -->|否| H["检查标签触发"]
    D -->|否| H
    H --> I{标签匹配?}
    I -->|是| G
    I -->|否| J["检查内容触发"]
    J --> K{标题/正文包含触发词?}
    K -->|是| L["精确匹配验证"]
    K -->|否| M["检查评论触发"]
    L --> N{边界词匹配?}
    N -->|是| G
    N -->|否| M
    M --> O{评论包含触发词?}
    O -->|是| L
    O -->|否| P["不触发"]
    
    G --> Q["返回 true"]
    P --> R["返回 false"]
```

## 详细数据流图

```mermaid
graph LR
    subgraph "输入层"
        A1["GitHub Event Payload"] --> A2["环境变量"]
        A2 --> A3["Action Inputs"]
    end
    
    subgraph "解析层"
        B1["context.ts"] --> B2["事件类型解析"]
        B2 --> B3["Repository 信息"]
        B3 --> B4["Actor 信息"]
        B4 --> B5["Payload 解析"]
    end
    
    subgraph "模式选择层"
        C1["registry.ts"] --> C2["模式验证"]
        C2 --> C3["事件兼容性检查"]
        C3 --> C4["模式实例化"]
    end
    
    subgraph "验证层"
        D1["trigger.ts"] --> D2["触发条件检查"]
        D2 --> D3["正则表达式匹配"]
        D4["permissions.ts"] --> D5["权限级别检查"]
        D6["actor.ts"] --> D7["人员类型验证"]
    end
    
    subgraph "决策层"
        E1["shouldTrigger()"] --> E2["综合判断"]
        E2 --> E3["执行/跳过决策"]
    end
    
    A1 --> B1
    A2 --> B1
    A3 --> B1
    B5 --> C1
    C4 --> D1
    C4 --> D4
    C4 --> D6
    D3 --> E1
    D5 --> E1
    D7 --> E1
```

## 核心模块详解

### 1. 事件解析：`src/github/context.ts`

```typescript
// 核心函数：parseGitHubContext()
export function parseGitHubContext(): GitHubContext {
  const context = github.context;
  
  // 1. 解析通用字段
  const commonFields = {
    runId: process.env.GITHUB_RUN_ID!,
    eventAction: context.payload.action,
    repository: {
      owner: context.repo.owner,
      repo: context.repo.repo,
      full_name: `${context.repo.owner}/${context.repo.repo}`,
    },
    actor: context.actor,
    inputs: {
      mode: modeInput as ModeName,
      triggerPhrase: process.env.TRIGGER_PHRASE ?? "@claude",
      assigneeTrigger: process.env.ASSIGNEE_TRIGGER ?? "",
      labelTrigger: process.env.LABEL_TRIGGER ?? "",
      allowedTools: parseMultilineInput(process.env.ALLOWED_TOOLS ?? ""),
      disallowedTools: parseMultilineInput(process.env.DISALLOWED_TOOLS ?? ""),
      customInstructions: process.env.CUSTOM_INSTRUCTIONS ?? "",
      directPrompt: process.env.DIRECT_PROMPT ?? "",
      // ... 更多配置
    },
  };
  
  // 2. 根据事件类型解析特定字段
  switch (context.eventName) {
    case "issues":
      return {
        ...commonFields,
        eventName: "issues",
        payload: context.payload as IssuesEvent,
        entityNumber: payload.issue.number,
        isPR: false,
      };
    case "issue_comment":
      return {
        ...commonFields,
        eventName: "issue_comment", 
        payload: context.payload as IssueCommentEvent,
        entityNumber: payload.issue.number,
        isPR: Boolean(payload.issue.pull_request),
      };
    // ... 其他事件类型
  }
}
```

**支持的事件类型：**
```typescript
// Entity Events (实体事件)
- issues: Issue 的创建、编辑、指派、标签等
- issue_comment: Issue 或 PR 的评论
- pull_request: PR 的创建、更新、关闭等
- pull_request_review: PR 的评审
- pull_request_review_comment: PR 评审中的评论

// Automation Events (自动化事件)  
- workflow_dispatch: 手动触发工作流
- schedule: 定时触发工作流
```

### 2. 触发验证：`src/github/validation/trigger.ts`

```typescript
// 核心函数：checkContainsTrigger()
export function checkContainsTrigger(context: ParsedGitHubContext): boolean {
  const { inputs: { assigneeTrigger, labelTrigger, triggerPhrase, directPrompt } } = context;
  
  // 1. 直接提示优先级最高
  if (directPrompt) {
    console.log(`Direct prompt provided, triggering action`);
    return true;
  }
  
  // 2. 检查 Issue 指派触发
  if (isIssuesAssignedEvent(context)) {
    let triggerUser = assigneeTrigger.replace(/^@/, "");
    const assigneeUsername = context.payload.assignee?.login || "";
    
    if (triggerUser && assigneeUsername === triggerUser) {
      console.log(`Issue assigned to trigger user '${triggerUser}'`);
      return true;
    }
  }
  
  // 3. 检查标签触发
  if (isIssuesEvent(context) && context.eventAction === "labeled") {
    const labelName = context.payload.label?.name || "";
    
    if (labelTrigger && labelName === labelTrigger) {
      console.log(`Issue labeled with trigger label '${labelTrigger}'`);
      return true;
    }
  }
  
  // 4. 检查内容触发（Issue/PR 标题和正文）
  if (isIssuesEvent(context) && context.eventAction === "opened") {
    const issueBody = context.payload.issue.body || "";
    const issueTitle = context.payload.issue.title || "";
    const regex = new RegExp(`(^|\\s)${escapeRegExp(triggerPhrase)}([\\s.,!?;:]|$)`);
    
    if (regex.test(issueBody) || regex.test(issueTitle)) {
      return true;
    }
  }
  
  // 5. 检查评论触发
  if (isIssueCommentEvent(context) || isPullRequestReviewCommentEvent(context)) {
    const commentBody = context.payload.comment.body;
    const regex = new RegExp(`(^|\\s)${escapeRegExp(triggerPhrase)}([\\s.,!?;:]|$)`);
    
    if (regex.test(commentBody)) {
      console.log(`Comment contains exact trigger phrase '${triggerPhrase}'`);
      return true;
    }
  }
  
  return false;
}
```

**触发词匹配策略：**
```typescript
// 精确匹配正则表达式
const regex = new RegExp(`(^|\\s)${escapeRegExp(triggerPhrase)}([\\s.,!?;:]|$)`);

// 示例匹配：
✅ "@claude please help"     // 词边界匹配
✅ "Hello @claude!"          // 标点符号边界
✅ "@claude"                 // 行首/行尾
❌ "email@claude.ai"         // 避免误匹配
❌ "@claude123"              // 避免部分匹配
```

### 3. 权限验证：`src/github/validation/permissions.ts`

```typescript
// 核心函数：checkWritePermissions()
export async function checkWritePermissions(
  octokit: Octokit,
  context: ParsedGitHubContext,
): Promise<boolean> {
  const { repository, actor } = context;
  
  try {
    // 检查协作者权限级别
    const response = await octokit.repos.getCollaboratorPermissionLevel({
      owner: repository.owner,
      repo: repository.repo,
      username: actor,
    });
    
    const permissionLevel = response.data.permission;
    core.info(`Permission level retrieved: ${permissionLevel}`);
    
    // 只允许 admin 和 write 权限
    if (permissionLevel === "admin" || permissionLevel === "write") {
      core.info(`Actor has write access: ${permissionLevel}`);
      return true;
    } else {
      core.warning(`Actor has insufficient permissions: ${permissionLevel}`);
      return false;
    }
  } catch (error) {
    core.error(`Failed to check permissions: ${error}`);
    throw new Error(`Failed to check permissions for ${actor}: ${error}`);
  }
}
```

**权限级别说明：**
```typescript
// GitHub 仓库权限级别
- admin: 完全管理权限 ✅
- maintain: 维护权限（无删除）❌  
- write: 写入权限 ✅
- triage: 分类权限（只读 + Issue/PR 管理）❌
- read: 只读权限 ❌
```

### 4. 人员验证：`src/github/validation/actor.ts`

```typescript
// 核心函数：checkHumanActor()
export async function checkHumanActor(
  octokit: Octokit,
  githubContext: ParsedGitHubContext,
) {
  // 获取用户信息
  const { data: userData } = await octokit.users.getByUsername({
    username: githubContext.actor,
  });
  
  const actorType = userData.type;
  console.log(`Actor type: ${actorType}`);
  
  // 只允许真实用户触发
  if (actorType !== "User") {
    throw new Error(
      `Workflow initiated by non-human actor: ${githubContext.actor} (type: ${actorType}).`,
    );
  }
  
  console.log(`Verified human actor: ${githubContext.actor}`);
}
```

**Actor 类型说明：**
```typescript
// GitHub Actor 类型
- User: 真实用户账号 ✅
- Bot: 机器人账号 ❌
- Organization: 组织账号 ❌
```

### 5. 模式选择：`src/modes/registry.ts`

```typescript
// 核心函数：getMode()
export function getMode(name: ModeName, context: GitHubContext): Mode {
  const mode = modes[name];
  if (!mode) {
    const validModes = VALID_MODES.join("', '");
    throw new Error(
      `Invalid mode '${name}'. Valid modes are: '${validModes}'.`,
    );
  }
  
  // 验证模式与事件类型的兼容性
  if (name === "tag" && isAutomationContext(context)) {
    throw new Error(
      `Tag mode cannot handle ${context.eventName} events. Use 'agent' mode for automation events.`,
    );
  }
  
  return mode;
}

// 可用模式
const modes = {
  tag: tagMode,                    // 传统触发模式
  agent: agentMode,               // 自动化代理模式  
  "experimental-review": reviewMode, // 实验性评审模式
} as const;
```

## 模式特定处理流程

```mermaid
graph TD
    A["模式选择"] --> B{tag 模式}
    A --> C{agent 模式}
    A --> D{experimental-review 模式}
    
    B --> E["实体事件检查"]
    E --> F["触发条件验证"]
    F --> G["人员验证"]
    G --> H["创建跟踪评论"]
    H --> I["分支设置"]
    
    C --> J["自动化事件处理"]
    J --> K["直接执行"]
    
    D --> L["PR 事件检查"]
    L --> M["特定 Action 验证"]
    M --> N["评审工具配置"]
    
    I --> O["继续执行"]
    K --> O
    N --> O
```

### Tag 模式处理逻辑

```typescript
// src/modes/tag/index.ts
export const tagMode: Mode = {
  name: "tag",
  description: "Traditional implementation mode triggered by @claude mentions",
  
  shouldTrigger(context) {
    // 只处理实体事件
    if (!isEntityContext(context)) {
      return false;
    }
    return checkContainsTrigger(context);
  },
  
  async prepare({ context, octokit, githubToken }: ModeOptions): Promise<ModeResult> {
    if (!isEntityContext(context)) {
      throw new Error("Tag mode requires entity context");
    }
    
    // 1. 检查是否为人类操作者
    await checkHumanActor(octokit.rest, context);
    
    // 2. 创建初始跟踪评论
    const commentData = await createInitialComment(octokit.rest, context);
    
    // 3. 获取 GitHub 数据
    const githubData = await fetchGitHubData({
      octokits: octokit,
      repository: `${context.repository.owner}/${context.repository.repo}`,
      prNumber: context.entityNumber.toString(),
      isPR: context.isPR,
      triggerUsername: context.actor,
    });
    
    // 4. 设置分支
    const branchInfo = await setupBranch(octokit, githubData, context);
    
    // 5. 配置 Git 认证
    if (!context.inputs.useCommitSigning) {
      await configureGitAuth(githubToken, context, commentData.user);
    }
    
    // 6. 创建提示词
    const modeContext = this.prepareContext(context, {
      commentId: commentData.id,
      baseBranch: branchInfo.baseBranch,
      claudeBranch: branchInfo.claudeBranch,
    });
    
    await createPrompt(tagMode, modeContext, githubData, context);
    
    // 7. 准备 MCP 配置
    const mcpConfig = await prepareMcpConfig({
      githubToken,
      owner: context.repository.owner,
      repo: context.repository.repo,
      branch: branchInfo.claudeBranch || branchInfo.currentBranch,
      baseBranch: branchInfo.baseBranch,
      additionalMcpConfig: process.env.MCP_CONFIG || "",
      claudeCommentId: commentData.id.toString(),
      allowedTools: context.inputs.allowedTools,
      context,
    });
    
    return {
      commentId: commentData.id,
      branchInfo,
      mcpConfig,
    };
  },
};
```

## 事件处理时序图

```mermaid
sequenceDiagram
    participant G as GitHub
    participant A as Actions Runner
    participant C as context.ts
    participant R as registry.ts
    participant V as validation/*
    participant M as Mode Handler
    
    G->>A: Webhook 事件
    A->>C: parseGitHubContext()
    C->>C: 解析事件类型和 Payload
    C->>C: 解析环境变量和输入
    C-->>A: GitHubContext
    
    A->>R: getMode()
    R->>R: 验证模式有效性
    R->>R: 检查事件兼容性
    R-->>A: Mode 实例
    
    A->>M: mode.shouldTrigger()
    M->>V: checkContainsTrigger()
    V->>V: 检查直接提示
    V->>V: 检查指派/标签触发
    V->>V: 检查内容触发
    V-->>M: 触发结果
    
    alt 需要触发
        M-->>A: true
        A->>V: checkWritePermissions()
        V->>G: API 调用检查权限
        G-->>V: 权限级别
        V-->>A: 权限验证结果
        
        A->>V: checkHumanActor()
        V->>G: API 调用获取用户信息
        G-->>V: 用户类型
        V-->>A: 人员验证结果
        
        A->>M: mode.prepare()
        M-->>A: 准备结果
    else 不需要触发
        M-->>A: false
        A-->>A: 跳过执行
    end
```

## 配置解析和验证

### 输入参数解析

```typescript
// parseMultilineInput() - 解析多行输入
export function parseMultilineInput(s: string): string[] {
  return s
    .split(/,|[\n\r]+/)           // 按逗号或换行分割
    .map((tool) => tool.replace(/#.+$/, ""))  // 移除注释
    .map((tool) => tool.trim())    // 去除空白
    .filter((tool) => tool.length > 0);  // 过滤空项
}

// 示例输入：
// ALLOWED_TOOLS="Edit,MultiEdit  # 文件编辑工具
//               Glob,Grep       # 文件搜索工具
//               LS,Read,Write"  # 基础文件操作
// 
// 解析结果：["Edit", "MultiEdit", "Glob", "Grep", "LS", "Read", "Write"]
```

### 权限配置解析

```typescript
// parseAdditionalPermissions() - 解析额外权限
export function parseAdditionalPermissions(s: string): Map<string, string> {
  const permissions = new Map<string, string>();
  if (!s || !s.trim()) return permissions;
  
  const lines = s.trim().split("\n");
  for (const line of lines) {
    const trimmedLine = line.trim();
    if (trimmedLine) {
      const [key, value] = trimmedLine.split(":").map((part) => part.trim());
      if (key && value) {
        permissions.set(key, value);
      }
    }
  }
  return permissions;
}

// 示例输入：
// ADDITIONAL_PERMISSIONS="actions: read
//                        packages: read"
//
// 解析结果：Map { "actions" => "read", "packages" => "read" }
```

## 错误处理机制

### 事件解析错误

```typescript
// 1. 不支持的事件类型
default:
  throw new Error(`Unsupported event type: ${context.eventName}`);

// 2. 无效的模式
if (!isValidMode(modeInput)) {
  throw new Error(`Invalid mode: ${modeInput}.`);
}

// 3. 模式与事件不兼容
if (name === "tag" && isAutomationContext(context)) {
  throw new Error(
    `Tag mode cannot handle ${context.eventName} events. Use 'agent' mode for automation events.`,
  );
}
```

### 验证错误

```typescript
// 1. 权限不足
if (permissionLevel !== "admin" && permissionLevel !== "write") {
  core.warning(`Actor has insufficient permissions: ${permissionLevel}`);
  return false;
}

// 2. 非人类操作者
if (actorType !== "User") {
  throw new Error(
    `Workflow initiated by non-human actor: ${githubContext.actor} (type: ${actorType}).`,
  );
}

// 3. API 调用失败
catch (error) {
  core.error(`Failed to check permissions: ${error}`);
  throw new Error(`Failed to check permissions for ${actor}: ${error}`);
}
```

## 事件类型判断和类型守卫

```typescript
// 类型守卫函数
export function isIssuesEvent(context: GitHubContext): context is ParsedGitHubContext & { payload: IssuesEvent } {
  return context.eventName === "issues";
}

export function isIssueCommentEvent(context: GitHubContext): context is ParsedGitHubContext & { payload: IssueCommentEvent } {
  return context.eventName === "issue_comment";
}

export function isPullRequestEvent(context: GitHubContext): context is ParsedGitHubContext & { payload: PullRequestEvent } {
  return context.eventName === "pull_request";
}

// 组合判断
export function isIssuesAssignedEvent(context: GitHubContext): context is ParsedGitHubContext & { payload: IssuesAssignedEvent } {
  return isIssuesEvent(context) && context.eventAction === "assigned";
}

// 上下文类型判断
export function isEntityContext(context: GitHubContext): context is ParsedGitHubContext {
  return ENTITY_EVENT_NAMES.includes(context.eventName as EntityEventName);
}

export function isAutomationContext(context: GitHubContext): context is AutomationContext {
  return AUTOMATION_EVENT_NAMES.includes(context.eventName as AutomationEventName);
}
```

## 故障排除

### 常见问题

1. **事件未触发**
   ```bash
   # 检查触发条件
   echo "Trigger phrase: $TRIGGER_PHRASE"
   echo "Direct prompt: $DIRECT_PROMPT"
   echo "Event name: $GITHUB_EVENT_NAME"
   echo "Event action: $GITHUB_EVENT_ACTION"
   ```

2. **权限不足**
   ```bash
   # 错误：Actor has insufficient permissions
   # 解决：确保触发用户有仓库的 write 或 admin 权限
   ```

3. **模式不兼容**
   ```bash
   # 错误：Tag mode cannot handle workflow_dispatch events
   # 解决：为自动化事件使用 agent 模式
   ```

4. **非人类操作者**
   ```bash
   # 错误：Workflow initiated by non-human actor
   # 解决：确保是真实用户而非机器人触发
   ```

### 调试技巧

```bash
# 查看完整事件信息
echo "$GITHUB_EVENT_PATH" | jq '.'

# 检查解析后的上下文
echo "Repository: $GITHUB_REPOSITORY"
echo "Actor: $GITHUB_ACTOR"
echo "Event action: $GITHUB_EVENT_ACTION"

# 验证触发词匹配
grep -E "(^|\s)@claude([\s.,!?;:]|$)" <<< "$COMMENT_BODY"

# 检查权限
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$GITHUB_REPOSITORY/collaborators/$GITHUB_ACTOR/permission"
```

## 与其他阶段的集成

### 向 Prepare 阶段传递数据

```bash
# 核心输出
contains_trigger=true/false
mode=tag/agent/experimental-review

# 解析后的上下文
REPOSITORY=$GITHUB_REPOSITORY
ACTOR=$GITHUB_ACTOR
ENTITY_NUMBER=$PR_NUMBER/$ISSUE_NUMBER
IS_PR=true/false

# 触发信息
TRIGGER_PHRASE="@claude"
DIRECT_PROMPT="Please review this code"
CUSTOM_INSTRUCTIONS="Follow our coding standards"
```

### 模式特定配置

```bash
# Tag 模式
BRANCH_PREFIX="claude/"
USE_STICKY_COMMENT=true
USE_COMMIT_SIGNING=false

# Agent 模式
MAX_TURNS=10
TIMEOUT_MINUTES=30

# Review 模式
ALLOWED_TOOLS="mcp__github_inline_comment__create_inline_comment"
```

## 总结

事件处理阶段是整个 Claude Code Action 的入口和决策中心，它：

1. **事件感知**：解析和分类 GitHub 事件，提取关键信息
2. **触发决策**：通过多种策略（直接提示、指派、标签、内容匹配）判断是否需要执行
3. **安全验证**：确保只有有权限的真实用户可以触发执行
4. **模式选择**：根据事件类型和配置选择合适的执行模式
5. **上下文准备**：为后续阶段提供完整的执行上下文

这个阶段确保了 Claude Code Action 能够安全、智能地响应各种 GitHub 事件，为后续的数据收集和执行奠定了坚实的基础。