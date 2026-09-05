# Claude 4.6 Adaptive Thinking Mode: 测试示例指南

为了验证 Claude 4.6 Adaptive (自适应) Thinking 模式的集成效果，特别是 `effort` 参数的生效情况及 Token 限制的自动调整，请参考以下测试场景。

## 1. 验证 Adaptive 模式激活与 Effort 控制

此测试验证系统能否正确传递 `thinking: { type: "adaptive", effort: "..." }` 参数，并观察模型行为差异。

### 前置条件
*   确保使用支持 Adaptive Thinking 的模型 ID (如 `claude-opus-4-6-thinking` 或映射后的 ID)。
*   在设置中将 "Thinking Budget" 模式设置为 **"Adaptive (自适应)"**。

### 测试指令示例

#### 场景 A: Low Effort (低强度思考)
*   **配置**: 将 Effort 设置为 `Low`。
*   **指令**: `写一个并通过 Rust 编译的 Hello World 程序。`
*   **预期结果**:
    *   Thinking 块应该比较简短，模型认为这是一个简单任务，不需要深入推理。
    *   响应速度较快。

#### 场景 B: High Effort (高强度思考)
*   **配置**: 将 Effort 设置为 `High`。
*   **指令**: `请详细分析 Rust 的 async/await 状态机生成机制，并对比 Go 的 Goroutine 调度模型。请通过思维链深入推导两者的内存开销差异。`
*   **预期结果**:
    *   Thinking 块应该非常长且详细（可能超过 5k tokens）。
    *   模型会尝试进行深度的对比分析和推理。
    *   **关键验证点**: 检查 Antigravity 日志，确认 `generationConfig` 中包含了 `thinkingConfig: { type: "adaptive", effort: "high" }`。

---

## 2. 验证多轮对话中的 Adaptive 状态维持

验证在多轮对话中，Adaptive 模式是否能持续生效，且 Token 限制 (128k) 是否正常工作。

### 场景：复杂算法设计迭代

#### Round 1: 初始设计
*   **指令**:
    ```bash
    claude "设计一个分布式的高并发秒杀系统。需要考虑缓存一致性、库存防超卖、防刷接口等核心问题。请使用 High Effort 进行深度思考。"
    ```
*   **验证点**:
    *   生成了包含架构图和详细逻辑的设计文档。
    *   Thinking 过程详细记录了对不同方案（如 Redis Lua vs 数据库悲观锁）的权衡。
    *   验证响应头或日志中，确认 `maxOutputTokens` 被提升至 **128,000** (或更高)，以容纳长输出。

#### Round 2: 方案挑战 (模拟用户反馈)
*   **指令**:
    ```bash
    claude "你的设计中，Redis 集群如果发生脑裂，如何保证库存数据的强一致性？请重新思考并修正方案。"
    ```
*   **验证点**:
    *   Thinking 块继续保持深度推理，分析 Redlock 或其他一致性算法的适用性。
    *   **签名验证**: 确保多轮对话中 Thinking Block 的签名验证通过（无 `Invalid signature` 报错）。

#### Round 3: 代码实现
*   **指令**:
    ```bash
    claude "请给出库存扣减核心逻辑的 Rust 代码实现。"
    ```
*   **验证点**:
    *   能生成符合之前设计思路的代码。
    *   在高上下文压力下，系统是否自动触发了 Thinking 剥离（如果配置了动态剥离），或者能够正常携带完整历史继续生成。

---

## 3. 验证 Budget 模式与 Adaptive 模式的自动切换

此测试验证当用户在“固定 Budget”与“Adaptive”模式间切换时，后端能否正确转换参数。

### 测试流程
1.  **设置为 Fixed Budget**: 在设置中选择 "Custom" 并设置 Budget 为 `16384`。
    *   发送请求。
    *   *验证*: 后端请求应只包含 `thinkingConfig: { budget: 16384 }`，**不应包含** `effort`。

2.  **切换为 Adaptive**: 在设置中选择 "Adaptive" 并设置 Effort 为 `Medium`。
    *   发送请求。
    *   *验证*: 后端请求应只包含 `thinkingConfig: { type: "adaptive", effort: "medium" }`，**不应包含** `budget`。

---

## 4. 调试建议

在运行上述测试时，建议开启 Debug 日志以观察参数传递：

```bash
RUST_LOG=debug npm run tauri dev
```

在日志中搜索关键词：
*   `[Claude-Request]`: 查看转换后的请求体。
*   `thinkingConfig`: 确认配置注入情况。
*   `maxOutputTokens`: 确认 Token 上限调整情况。
