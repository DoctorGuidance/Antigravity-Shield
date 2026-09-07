# 503 错误（Service Unavailable）修复验证指南

本指南针对近期反馈的 503 错误（Issue #1794 及后端容量限制）提供测试验证示例。

## 1. 验证 Project ID 获取失败后的自动回退 (Issue #1794)

**场景描述**：
部分账号（特别是 Free 账号或受限账号）在调用官方接口获取项目 ID 时会报错 `账号无资格获取官方 cloudaicompanionProject`。在修复前，系统会直接跳过该账号导致最终返回 503；修复后，系统将自动使用通用 Project ID (`bamboo-precept-lgxtn`)。

### A. 使用 `curl` 进行基础连通性测试
请使用一个之前报错 503 的账号对应的 API Key（或直接通过代理）：

```bash
curl http://localhost:8045/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-antigravity-key" \
  -d '{
    "model": "gemini-2.0-flash",
    "messages": [
      {"role": "user", "content": "你好，请确认你的工作状态。"}
    ],
    "stream": false
  }'
```

### B. 观察服务端日志 (npm run tauri dev)
**预期现象**：
当系统检测到权限问题时，日志中会出现如下 **Warn** 信息，但请求**不应报错 503**，而是继续执行：

```text
WARN Failed to fetch project_id for user@example.com, using fallback: Account is not eligible for official cloudaicompanionProject
DEBUG [TokenManager] Using project_id: bamboo-precept-lgxtn for request
```

---

## 2. 验证 Quota Protection（配额保护）对 503 的预防

**场景描述**：
当账号配额耗尽或后端因高负载返回 503 时，系统应正确识别并尝试轮换账号，而不是直接透传 503 给客户端。

### 测试指令 (Claude CLI)
```bash
claude "这段代码哪里有 Bug？[附带一段长代码]"
```

**验证点**：
- 如果当前账号返回 503，日志中应显示 `[RetryStrategy] Status 503 detected, rotating account...`。
- 系统应自动尝试下一个可用账号，直到获得成功响应或消耗完重试次数。

---

## 3. 区分“代码 Bug”与“后端容量限制” (Opus 4.6)

**场景描述**：
由于 `claude-opus-4-6-thinking` 模型目前处于试验阶段，Google 后端时常返回 `No capacity available` (503)。

### 测试指令
```bash
claude --model claude-opus-4-6-thinking "执行一次深度的推理任务，比较 Rust 和 C++ 的异步内存模型。"
```

**预期结果分析**：
1. **如果返回 503 且消息包含 "No capacity available"**：
   - 这是 **Google 后端容量限制**，并非本软件 Bug。
   - 代理会自动通过重试策略尝试其他账号，但如果所有账号都遇到容量限制，最终会透传此 503。
   - **建议**：在此负载高峰期切换到 `gemini-2.0-flash-thinking-exp` 或 `claude-3-7-sonnet` 进行测试。

2. **如果返回成功**：
   - 说明当前后端容量充足。

---

## 调试辅助技巧

如果您想强制模拟 Project ID 失败的场景进行代码级验证，可以在 `src-tauri/src/proxy/token_manager.rs` 中暂时修改模拟逻辑。但在大多数情况下，通过观察日志中是否出现 `using fallback: ...` 字样即可确认修复生效。
