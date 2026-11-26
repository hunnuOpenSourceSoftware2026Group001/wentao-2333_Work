# practice-04

**标题：** 实践课4：Pi Agent 模型调用与运行流程 zhangwentao  
**个人作业仓库：** <https://github.com/hunnuOpenSourceSoftware2026Group001/wentao-2333_Work>  
**源码依据：** 小组仓库 `pi-hunnu-2026` `main` 分支提交 `340c6e048a189fda989c9bb06cb425b6d5bc6007`，个人重点修改模块：`packages/agent`（Agent 循环）。

---

## 1. 源码修改（在 Agent 循环加两行日志）

文件：`packages/agent/src/agent-loop.ts`

在 Agent 循环的模型调用前后，各插入一行 `[P4]` 日志，用于记录“即将请求模型”和“工具结果已加入上下文”。

### 修改位置 A：模型调用前

上下文（`runLoop` 内，模型请求前）：

```ts
    // 在请求模型前打印日志
+   console.error("[P4] Agent 即将请求模型");
    const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFunction);
```

### 修改位置 B：工具结果加入上下文后

上下文（工具执行后把结果推入上下文）：

```ts
    currentContext.messages.push(result);
+   console.error("[P4] 工具结果已加入上下文");
```

> **截图：** 提交两处日志的代码差异或截图，标清插入位置。

---

## 2. 重新编译并启动

在组织仓库根目录重新编译：

```powershell
npm run build
```

验证构建成功（`$LASTEXITCODE` 为 0，`packages/coding-agent/dist/bundle/cli.js` 已更新）：

```powershell
node .\packages\coding-agent\dist\bundle\cli.js
```

**截图：** 提交重新编译和启动的记录，以及终端里 `[P4]` 日志的首次出现。

---

## 3. 对比两次运行

启动本地源码版 Pi，分别运行会话 A 与会话 B，记录输入指令、是否调用工具、按出现顺序记录的 `[P4]` 日志、最终回复。

### 会话 A：直接问答

- **输入指令：** `请直接用一句话解释什么是 Git 分支。`
- **是否调用工具：** 否（预期不调用工具，直接回答）
- **[P4] 日志（按出现顺序）：**

```
[P4] Agent 即将请求模型
```

（实际次数按运行记录填写，通常进入 1 轮模型调用）

- **最终回复：** （填写 Pi 对 Git 分支的一句话解释）

**截图 S5：** 显示会话 A 的输入指令、Pi 最终回复和实际出现的 `[P4]` 日志。结合日志记录是否调用了工具，以及进入了几轮模型调用。

### 会话 B：读取文件

- **输入指令：** `请用 read 工具读取当前目录的 practice4.txt，并告诉我文件中的实验代号。`
- **是否调用工具：** 是（调用 `read` 工具读取 `practice4.txt`）
- **[P4] 日志（按出现顺序，典型完整顺序）：**

```
[P4] Agent 即将请求模型        ← 第 1 轮：决定调用 read 工具
[P4] 工具结果已加入上下文       ← 工具结果写回上下文
[P4] Agent 即将请求模型        ← 第 2 轮：带着工具结果再次请求模型
```

（实际次数按运行记录填写）

- **最终回复：** Pi 是否读到了 `practice4.txt` 中的实验代号，并输出代号内容。

**截图 S6：** 显示会话 B 的输入指令、`read` 工具读取 `practice4.txt` 的记录、Pi 返回的实验代号，以及实际出现的两类 `[P4]` 日志。终端一屏放不下时，可按发生顺序提交 S6a、S6b 等截图，并在报告中按顺序抄录日志。

---

## 4. 运行流程分析

### 谁实际读了文件？

是 **Pi（coding-agent / agent 运行时）调用了内置 `read` 工具**，由工具把文件内容写入上下文，而不是模型直接“看”文件。

### 下一轮模型请求多带了什么？

下一轮模型请求多带了 **`read` 工具返回的文件内容（工具结果）**，写入上下文后随消息一起发给模型，模型据此回答实验代号。

### 流程简图

见下方 `流程图.mmd`，说明 `coding-agent`、`agent`、`ai` 和远程模型服务的协作关系。

```mermaid
flowchart LR
    subgraph 用户层
    U[用户 / 终端输入]
    end
    subgraph Pi 客户端（coding-agent）
    CA[coding-agent CLI / 交互层]
    end
    subgraph Agent 运行时（agent）
    AG[agent 运行时 / Agent 循环]
    end
    subgraph 模型层（ai）
    AI[ai：统一 Provider API]
    end
    subgraph 远程模型服务
    RM[远程 LLM 服务]
    end

    U -->|指令| CA
    CA -->|驱动| AG
    AG -->|第 n 轮请求模型| AI
    AI -->|调用模型| RM
    RM -->|模型回复| AI
    AI -->|返回（含工具调用/最终答案）| AG
    AG -->|工具结果写入上下文| AG
    AG -->|呈现回复| CA
    CA -->|展示| U

    style RM fill:#f9f,stroke:#333
    style AI fill:#bbf,stroke:#333
```

> 箭头说明：`coding-agent` 负责交互入口与工具调度；`agent` 负责 Agent 循环与上下文管理；`ai` 负责与不同模型厂商对齐的 Provider API；`RM` 是远程模型服务。工具结果先写回 `agent` 上下文，再随下一轮请求发给 `ai` → `RM`。

---

## 5. 作业提交

在个人作业仓库提交一份 `practice-04.md`，包含：

1. **源码修改：** 两处日志的代码差异或截图，标清插入位置。
2. **运行证据：** 重新编译和启动的记录，以及两次会话实际观察到的 `[P4]` 日志和结果。截图中隐去授权信息。
3. **运行流程：** 对比两次会话。提交 A（直接问答）和 B（读取文件）的实际记录。每次写明输入指令、是否调用工具、按出现顺序记录的 `[P4]` 日志，以及最终回复。说明两次分别进入了几轮模型调用，并结合 B 的日志回答：谁实际读了文件？下一轮模型请求多带了什么？附一张简图标出 coding-agent、`agent`、`ai` 和远程模型服务。

在课程作业仓库 <https://github.com/hunnuOpenSourceSoftware2026/assignments/issues/new/choose> 提交 Issue：

- **标题：** `实践课4：Pi Agent模型调用与运行流程 zhangwentao`
- **正文：** 附个人报告链接和小组代码仓库链接。
