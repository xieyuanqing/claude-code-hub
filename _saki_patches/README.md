# Saki 的 CCH 补丁手册

## 补丁目的
为了让下游的 `notion2api` 能够实现真正的“会话复用”且“不复读”，我们对 CCH 做了微调。
核心逻辑：将 CCH 内部维护的稳定 `sessionId` 透传给下游 `openai-compatible` 供应商。

## 修改详情
- **目标文件**: `src/app/v1/_lib/proxy/forwarder.ts`
- **改动点**: 在 `ProxyForwarder.buildHeaders` 函数中，针对 `openai-compatible` 类型的供应商，额外注入 `overrides["x-cch-session-id"] = session.sessionId;`。

## 为什么这么改？
1. **唯一标��**: 保证不同客户端（如酒馆）即便发全量历史，只要 CCH session 不变，下游 ID 就固定。
2. **安全隔离**: 只对 `openai-compatible` 生效，不干扰官方 Claude/Gemini 请求。

## 如何恢复？
如果 CCH 更新覆盖了代码，按以下步骤操作：
1. **自动恢复**: 尝试在 CCH 源码根目录执行 `git apply _saki_patches/cch_session_forwarding.patch`。
2. **手动恢复**: 
   - 找到 `src/app/v1/_lib/proxy/forwarder.ts`
   - 搜 `overrides["user-agent"] = resolvedUA;`
   - 在下面添加如下代码：
     ```typescript
     if (provider.providerType === "openai-compatible" && session.sessionId) {
       overrides["x-cch-session-id"] = session.sessionId;
     }
     ```
3. **重建部署**:
   - `docker build -f deploy/Dockerfile -t ghcr.io/ding113/claude-code-hub:latest .`
   - `cd ../deploy-cch && docker compose up -d --force-recreate app`

---
*记录时间: 2026-03-21*
*执行人: Saki (OpenClaw Personal Assistant)*
