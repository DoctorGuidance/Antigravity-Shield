# 稳定性与搜索功能：测试示例指南

为了验证近期对 API 400 错误及“搜索文件错误”的修复效果，您可以在 Claude CLI (Claude Code) 中运行以下指令进行实测。

## 1. 验证搜索工具自愈 (Grep/Glob Fix)

针对之前的 "Error searching files" 问题，这些指令将触发 `Grep` 和 `Glob` 工具调用，并验证参数映射是否正确。

### 测试指令示例
*   **指令 A**：`在当前目录中搜索包含 "fn handle_messages" 的 Rust 文件。`
    *   *验证点*：检查代理是否能正确将 `query` 映射为 `pattern`，并注入默认的 `path: "."`。
*   **指令 B**：`列出 src-tauri 目录下所有 .rs 文件。`
    *   *验证点*：验证 `Glob` 工具名是否被正确识别，且路径过滤逻辑正常。

---

## 2. 验证协议顺序与签名稳定性 (Thinking/Signature Fix)

针对之前的 `Found 'text'` 和 `Invalid signature` 400 错误。

### 测试指令示例
*   **指令 A（推理+搜索）**：`分析本项目中处理云端请求的核心逻辑，按调用顺序总结，并给出关键代码行的 Grep 搜索证据。`
    *   *验证点*：验证在“思维 -> 工具调用 -> 结果 -> 继续思维”循环中，块顺序是否正确。
*   **指令 B（历史记录重试）**：在长对话中频繁切换模型，观察系统是否在 400 报错时静默修复签名并重试。

---

## 附录：深度错误对照与修复方案

| 错误类别 | 具体报错特征码 (Error Detail) | 代理采取的修复/应对逻辑 |
| :--- | :--- | :--- |
| **消息流顺序违规** | `If an assistant message contains any thinking blocks... Found 'text'.` | **已修复**：`streaming.rs` 不再允许在文字块之后非法追加思维块。 |
| **思维签名不匹配** | `Invalid signature in thinking block` | **已修复**：优先保留原始名称以保护 Google 后端签名校验。 |
| **思维签名缺失** | `Function call is missing a thought_signature` | **已修复**：自动注入 `skip_thought_signature_validator` 占位符。 |
| **非法缓存标记** | `thinking.cache_control: Extra inputs are not permitted` | **已修复**：全局剔除历史消息中的 `cache_control` 标记。 |
| **Plan Mode 报错**| `EnterPlanMode tool call: InputValidationError: Extra inputs are not permitted` | **已修复**：`streaming.rs` 强制清空工具参数以符合官方无参协议。 |
| **连续 User 消息**| `Consecutive user messages are not allowed` | **已修复**：`merge_consecutive_messages` 自动合并相邻同角色消息。 |

---

## 3. 验证 Claude Code Plan Mode 与角色交替 (Issue #813)

针对 Plan Mode 切换导致的协议报错问题。

### A. 验证 Plan Mode 激活 (UI 状态)
*   **指令**：`进入 Plan Mode 调研 src-tauri 的目录结构。`
*   **预期结果**：
    *   终端左下角应立即出现蓝色的 **`plan mode on`** 标签。
    *   日志中应看到 `[Streaming] Tool Call: 'EnterPlanMode' Args: {}`。

### B. 验证角色交替自愈 (Consecutive Messages)
*   **指令**：`在 Plan Mode 下帮我分析 proxy/mappers/claude/request.rs 的逻辑，然后退出 Plan Mode 并给出一个简要总结。`
*   **预期结果**：
    *   模型切换模式（如从 Plan 到 Code）时不会因“连续两条 User 消息”而报 400 错误。
    *   日志中会体现 `merge_consecutive_messages` 的合并动作。

---

## 4. QuotaData 字段逻辑解析

设置页面中的“账号管理”列表，下方的进度条数据来源于 `QuotaData`。系统会在请求前检查账号配额，并在触及阈值时自动轮换。

---

## 调试建议
```bash
RUST_LOG=debug npm run tauri dev
```
在日志中搜索 `[Claude-Request]`，关注消息角色的排列顺序。

---

## 5. 验证 Thinking 签名持久化与重启容错 (Proxy Restart Test)

此测试模拟代理服务主要逻辑：验证当代理重启（内存签名缓存丢失）后，携带旧签名的历史消息是否会导致 400 错误。这是复现 `Invalid signature` 最有效的方法。

### 测试流程
1.  **生成 Thinking (Step 1)**:
    *   **指令**：`详细分析 proxy/mappers/claude/request.rs 的代码结构，特别是它是如何处理 Thinking Block 的。请展示思维过程。`
    *   *状态*：Claude CLI 会接收到包含 Thinking 和 Signature 的响应。

2.  **模拟环境变更 (Step 2)**:
    *   **动作**：**保持当前 Claude CLI 会话不关闭**。
    *   **动作**：在另一个终端完全重启 Antigravity (或 `npm run tauri dev`)。
    *   *原理*：重启会清空代理内存中的“签名白名单”，这意味着 Step 1 中下发的签名现在对代理来说是“未知/不可信”的。

3.  **触发历史重放 (Step 3)**:
    *   **指令**：`根据上面的分析，总结一下签名验证的核心逻辑。`
    *   *原理*：CLI 会将 Step 1 中的 Thinking Block + Signature 作为历史记录发送给重启后的代理。

### 预期结果 (验证修复)
*   **如果不通过**：报错 `Invalid signature in thinking block` (因为代理无法验证该签名，直接透传给了 Google，被 Google 拒收)。
*   **如果通过 (当前版本)**：代理发现签名不在内存缓存中，**自动触发降级逻辑**（剥离 Thinking Block 或作为纯文本发送），对话正常继续，无报错。

---

## 6. 验证动态思维剥离 (Dynamic Thinking Stripping)

此测试验证系统能否在**高 Context 压力**或**签名失效**场景下，自动剥离无用的历史 Thinking Block，从而解决 "Prompt is too long" 和 "Invalid signature" 错误。

### 前置条件
*   开启 Debug 日志: `RUST_LOG=debug npm run tauri dev`
*   确保使用支持 Thinking 的模型 (如 `claude-3-7-sonnet` 或映射后的 `gemini-2.0-flash-thinking-exp`)

### 验证场景 A: 模拟超长 Context 压力 (Simulate High Load)

此场景验证当对话历史接近 Token 上限时，系统是否会自动清理旧的 Thinking。

1.  **构造长对话**:
    *   **方法 1 (自动生成)**: 运行 `docs/generate_long_payload.sh` 生成 2MB 测试文件。
        ```bash
        chmod +x docs/generate_long_payload.sh
        ./docs/generate_long_payload.sh
        cat docs/long_context_payload.txt | pbcopy
        ```
        然后将剪贴板内容多次粘贴给 Claude，直到感知到显著延迟或收到上下文警告。

    *   **方法 2 (Deep Thinking 诱导 - 持续施压)**:
        以下 Prompts 经过设计，能诱导模型进行极长的思维推理。可以轮流发送：

        > **Round 1 (History)**: "Please analyze the history of computing from the abacus to quantum computers. For every major milestone (at least 20), perform a deep 'thinking' block simulating the thought process of the inventors. Detailed thinking is required. Aim for maximum output tokens."

        > **Round 2 (Math/Logic)**: "Prove the Riemann Hypothesis. Just kidding. But please perform a deep, step-by-step derivation of the Navier-Stokes existence and smoothness problem's core challenges. Explore 10 different mathematical approaches, evaluating the pros and cons of each in extreme detail."

        > **Round 3 (System Architecture)**: "Design a distributed system capable of handling 100 billion requests per second. Detail the consensus execution flow (Paxos/Raft) for a single transaction across 5000 nodes. Simulate the network partition handling logic in your 'thinking' process for at least 50 failure scenarios."

        > **Round 4 (Literature)**: "Write a recursive story where the protagonist is a recursive function. The story must nest at least 20 levels deep, and for each level, you must 'think' about the symbolic meaning of that recursion depth before writing the narrative part."

2.  **观察日志**:
    *   在终端搜索 `[ContextManager]`。
    *   **预期日志**:
        ```
        [INFO] [ContextManager] Context pressure: 95.0% (1900000 / 2000000), Strategy: Aggressive => Purifying history
        [DEBUG] History purified successfully
        ```

3.  **验证结果**:
    *   Request 成功发送给 Gemini，没有报 "Prompt is too long"。
    *   HTTP 响应头包含 `X-Context-Purified: true`。
    *   Claude CLI 用户侧无感（历史记录仍在 CLI 本地显示，但服务端已净化）。

### 验证场景 B: 签名失效免疫 (Signature Immunity via Stripping)

此场景验证即使不触发重试逻辑，高负载下的主动剥离也能顺带解决签名问题。

1.  **生成带签名的 Thinking**:
    *   **指令**: `思考一下 Rust 的所有权机制，写 500 字。`

2.  **重启 Proxy 且注入虚假负载 (可选)**:
    *   重启代理（清空签名缓存）。
    *   继续对话。此时带有旧签名的 Thinking 会被发送给代理。

3.  **预期结果**:
    *   如果上下文压力较大触发了 Stripping，或者因签名报错触发了 RetriedWithoutThinking，系统会剥离 Thinking Block。
    *   **关键点**: 一旦 Thinking Block 被剥离，`thought_signature` 也会随之消失。
    *   Gemini 收到的是纯文本历史，**绝不会报 Invalid Signature**。

---

## 7. OpenCode (Claude Code CLI) 多协议接入测试

**Antigravity 已全面支持 OpenCode 的多协议接入**，彻底解决了 `AI_TypeValidationError` 等兼容性问题。您可以根据需要选择以下任一方式接入。

### 端点配置表

| 协议类型 | Base URL (Antigravity) | 对应的 OpenCode Provider | 备注 |
| :--- | :--- | :--- | :--- |
| **Anthropic (原生)** | `http://localhost:8045/v1` | `anthropic` | **推荐**。支持 Thinking、工具调用、Artifacts 预览。 |
| **OpenAI (标准)** | `http://localhost:8045/v1` | `openai` | 支持通用 OpenAI 客户端逻辑。 |
| **OA-Compatible** | `http://localhost:8045/v1` | `openai-compatible` | 适用于强制指定非标准模型名称的场景。 |
| **Google Gemini** | `http://localhost:8045/v1` | `gemini` | 直接使用 Gemini 协议，支持 Google 原生 SDK 特性。 |

### A. 方式 1：Anthropic 原生协议 (推荐)

此方式能获得最佳的 Claude 原生体验，支持 Thinking 签名保护与 Beta 特性。

1.  **配置**:
    ```bash
    # 设置 Base URL (注意：OpenCode 的 anthropic provider 有时需要完整路径)
    export ANTHROPIC_BASE_URL="http://localhost:8045/v1"
    # 设置 API Key (Antigravity 的密钥)
    export ANTHROPIC_API_KEY="sk-antigravity-key"
    ```

2.  **测试指令**:
    ```bash
    claude "请使用思维链 (Thinking) 分析当前目录下的 Cargo.toml 依赖结构。"
    ```

3.  **验证点**:
    *   **Thinking**: 是否能看到蓝色的思维块输出？
    *   **签名**: 检查 Antigravity 日志，应显示 `Cached signature to session ... [FIFO: true]`。
    *   **无错**: 全程无 `Invalid signature` 报错。

### B. 方式 2：OpenAI 协议 (含 Compatible)

适用于习惯使用 OpenAI 生态或需要特定模型映射的用户。

1.  **配置**:
    ```bash
    # 设置 Base URL
    export OPENAI_BASE_URL="http://localhost:8045/v1"
    export OPENAI_API_KEY="sk-antigravity-key"
    ```

2.  **启动 OpenCode**:
    ```bash
    claude --provider openai --model gemini-2.0-flash
    # 或者使用 compatible 模式
    claude --provider openai-compatible --model gemini-2.0-flash
    ```

3.  **验证点**:
    *   **JSON 错误**: 尝试故意断网或使用无效 Key，OpenCode 应显示友好的 JSON 错误信息（如 `{"error": {"message": "..."}}`），而不再是 Crash。
    *   **非流式兼容**: OpenCode 的某些工具调用可能会使用非流式请求，验证其是否能正常解析 JSON 响应。

### C. 方式 3：Google Gemini 原生协议

Antigravity v4.1.4 新增支持。

1.  **配置**:
    ```bash
    export GEMINI_API_KEY="sk-antigravity-key"
    # 如果 OpenCode 支持 GEMINI_BASE_URL (通常需要反代工具如 cloudflared 或修改 config):
    export GEMINI_BASE_URL="http://localhost:8045/v1"
    ```

2.  **验证点**:
    *   **适配器检测**: Antigravity 日志应显示 `[Gemini] Client Adapter detected`。
    *   **Let It Crash**: 当遇到 403/404 错误时，响应应立即返回，而不是让 OpenCode 挂起等待重试。

### D. 常见问题排查

*   **Q: 报错 `AI_TypeValidationError`？**
    *   **A**: 请确保升级 Antigravity 到 v4.1.2+。旧版本返回的错误格式（纯文本）无法通过 OpenCode 的 Zod 校验。

*   **Q: Thinking 块显示为 `[Redacted]` 或直接消失？**
    *   **A**: 这是正常现象。为了保护 Google 的签名不被破坏，Antigravity 可能会在特定情况下（如高上下文压力或签名验证失败时）主动剥离思维块。只要对话能继续，说明 "Dynamic Stripping" 机制正在工作。

---

## 8. 多轮连续对话压力测试 (Continuous Conversation Stress Test)

此测试旨在验证高频、多轮交互下的 **Signed Session Stability**（签名会话稳定性）。请在一个 OpenCode 会话中**连续**执行以下步骤，不要重启或清空上下文。

### 场景：Rust 项目重构实战

#### 第 1 轮：深度代码审查 (Initial Analysis)
*   **指令**:
    ```bash
    claude "请详细审查 src-tauri/src/proxy/handlers/claude.rs 文件。关注其中的 handle_messages 函数，分析它是如何处理 Beta Headers 注入的。请使用思维链列出你的分析步骤。"
    ```
*   **验证点**:
    *   必须看到通过 `ClientAdapter` 注入 Header 的逻辑分析。
    *   响应包含完整的 Thinking Block。

#### 第 2 轮：模拟修改建议 (Refactoring Proposal)
*   **指令**:
    ```bash
    claude "基于你的分析，如果我要新增一个名为 'CherryStudio' 的适配器，应该在哪些文件中进行修改？请给出一个具体的实现计划，不要直接修改文件。"
    ```
*   **验证点**:
    *   Claude 能准确引用第 1 轮的上下文（证明 Session ID 传递正常）。
    *   Thinking 签名未丢失（若报错 `Invalid signature`，说明签名缓存失效）。

#### 第 3 轮：高频并发测试 (Concurrent Simulation)
*   **背景**: 在此轮中，我们模拟快速连续的追问，测试 FIFO 签名队列的鲁棒性。
*   **指令 (请连续快速执行 3 次)**:
    ```bash
    # 快速输入以下简短指令，模拟用户急促的追问
    claude "刚才的计划中，StreamingState 需要改吗？"
    claude "那 ClientAdapter trait 呢？"
    claude "Cargo.toml 需要加依赖吗？"
    ```
*   **验证点**:
    *   **乱序容忍**: 即使响应到达顺序可能与请求不一致，客户端不应崩溃。
    *   **队列深度**: Antigravity 日志中应显示 Signature Cache 正常更新，未出现覆盖导致的前序签名失效。

#### 第 4 轮：长文本生成 (Output Token Limit)
*   **指令**:
    ```bash
    claude "请为 ClientAdapter trait 编写一份详尽的开发者文档（Markdown格式），包含所有方法的详细注释、三个不同场景的最佳实践示例代码。字数要求 2000 字以上。"
    ```
*   **验证点**:
    *   验证在大输出量下，SSE 流是否稳定。
    *   观察日志中是否触发了 `ContextManager` 的主动纯化（Purify），以及签名是否被安全剥离。

---

## 6. 智能上下文压缩等级验证 (Compression Level Test)

针对新版本引入的 `智能上下文压缩等级` (Disabled / Low / Medium / High)，您可以使用以下步骤验证各等级的净化降噪表现。

### A. 基础测试数据准备

在客户端（如 Cline 或 Cherry Studio）中开启新会话，我们将准备一段混合了**冗余客套口语**和**典型重复构建日志**的文本作为第一轮的请求：

```text
Hello, could you please help me build this package? Actually, basically, I think probably it is failed.

Progress: 10%
Progress: 20%
Progress: 30%
Progress: 40%
Progress: 50%
Error: compilation failed at src/main.rs:25
```

当模型回复后，紧接着进行**第二轮追问**（例如：`How to fix this error?`），此时第一轮消息将成为“历史消息”，从而触发对应的静态清洗规则。

---

### B. 验证 Low 等级 (仅日志降噪)

1.  **设置**：在设置面板中，将 `智能上下文压缩等级` 选择为 **`低度 (Low - 日志降噪)`**。
2.  **测试**：执行上述两轮对话。
3.  **日志与效果验证**：
    *   在 `npm run tauri:debug` 的控制台中，观察第二轮请求发送时的 payload：
        *   **RTK 日志去噪生效**：第一轮消息中的 `Progress: 10% ... 50%` 会被自动净化并折叠成类似 `[Collapsed 3 similar lines]` 的占位符。
        *   **报错安全捞回**：底部的 `Error: compilation failed...` 报错信息被完整保留。
        *   **口语保持原样**：第一轮消息开头的 `Hello, could you please...` 客套话仍然完整保留在 payload 中，未被精简。

---

### C. 验证 Medium 等级 (日志降噪 + 口语净化)

1.  **设置**：在设置面板中，将 `智能上下文压缩等级` 选择为 **`中度 (Medium - 日志+口语)`**。
2.  **测试**：重新开辟会话，并执行上述两轮对话。
3.  **日志与效果验证**：
    *   在控制台中观察第二轮请求发送时的 payload：
        *   **RTK 日志去噪生效**：重复的进度行继续被成功折叠。
        *   **Caveman 口语净化生效**：历史消息中的开头文本被大幅纯化，无实质意义的 `Hello, could you please...` 和 `Actually, basically, I think probably...` 等修饰性语气词被完全剔除，转换为骨架极简的原始人语风。
        *   **代码隔离安全**：所有的代码片段和堆栈路径（如 `src/main.rs:25`）均未被破坏。

---

### D. 验证 High 等级 (日志+口语+动态防暴)

1.  **设置**：在设置面板中，将 `智能上下文压缩等级` 选择为 **`高度 (High - 动态防暴)`**。
2.  **测试**：发送一段较长或包含数万 Token 的项目代码，促使上下文用量上升。
3.  **日志与效果验证**：
    *   **动态三级压强自适应**：在控制台中将能够看到 `ContextManager` 实时估算当前上下文的压强比例（如 `Context pressure: 45.2%`）。
    *   **分级触发日志**：
        *   超出 L1 阈值：触发 `[Layer-1] Tool trimming` 裁剪老旧工具包。
        *   超出 L2 阈值：触发 `[Layer-2] Thinking compression` 压缩历史思考，同时重播并保护签名。
        *   超出 L3 阈值：触发 `[Layer-3] Fork+Summary` 终极重置，控制台将显示重开会话以及摘要的生成，同时首条摘要的 payload 中将显示注入的 `cache_control: {"type": "ephemeral"}` 标记（在后续聊天中，您会在客户端观察到 Prompt Cache 命中率呈几何级飙升，极速响应）。

