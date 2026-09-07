# 修复 Opus 4.6 调用报错 & UserToken 显示优化

## 问题

号池混着 Pro 和 Ultra 账号。Pro 没有 Opus 4.6 权限，Ultra 有。

之前轮询按配额高低选账号，不管订阅等级。用户调 Opus 4.6 时，系统可能选到 Pro 账号，直接报错。

## 改动

### 1. Ultra 优先调度

调 Opus 4.6/4.5 时，先按订阅等级排序：

```
Ultra > Pro > Free
```

同等级再按配额排。其他模型还是老逻辑，配额优先。

匹配规则：模型名包含 `claude-opus-4-6`、`claude-opus-4-5` 或 `opus` 就走 Ultra 优先。

### 2. UserToken 编辑数据不显示

点编辑 Token 时，IP 限制和宵禁时间显示空的。

问题：
- 前端传参用 `undefined`，Rust 需要 `null`
- 读取用 `||`，0 和空字符串被吃掉了，改成 `??`

### 3. 自定义过期时间

创建 Token 多了个 Custom 选项，选日期时间，精确到小时。

## 文件

```
src-tauri/src/proxy/token_manager.rs      # 排序逻辑
src-tauri/src/proxy/tests/mod.rs          # 测试模块
src-tauri/src/proxy/tests/ultra_priority_tests.rs  # Ultra 优先测试
src-tauri/src/commands/user_token.rs      # 自定义过期参数
src-tauri/src/modules/user_token_db.rs    # 数据库
src/pages/UserToken.tsx                   # 前端
```

## 验证

1. 调 Opus 4.6，看日志确认走的是 Ultra 账号
2. 创建 Token 设置 IP 限制和宵禁，编辑时确认数据正常回显
