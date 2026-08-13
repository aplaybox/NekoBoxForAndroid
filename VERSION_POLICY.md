# 依赖版本策略 (VERSION_POLICY)

> 本文档记录所有依赖升级的决策依据。**任何依赖升级必须先经过调研审视，不允许盲目升级。**

## 当前基线

| 组件 | 版本 | commit/tag | 来源 | 备注 |
|------|------|-----------|------|------|
| NekoBoxForAndroid | 1.4.2 | — | MatsuriDayo (via starifly fork) | starifly fork 已含全协议支持 |
| sing-box (neko fork) | 1.12.19-neko-1 | `aed32ee3066cdbc7d471e3e0415c5134088962df` | MatsuriDayo/sing-box 1.12.x | starifly 在此基础上加 snell/xhttp/juicity |
| sing-box (starifly) | 1.12.x-neko+extensions | `7567ef412c90660ddb94ee8b33ddd3a876e0c748` | starifly/sing-box 1.12.x | 2026-07-26 "update snell" |
| libneko | — | `1c47a3af71990a7b2192e03292b4d246c308ef0b` | MatsuriDayo/libneko | 与 sing-box 配套 |
| metacubex/utls | v1.8.4 | `cf49b0864331` | indirect via sing-box | 与 sing-box 1.12.x 配套 |
| sagernet/quic-go | v0.52.0-sing-box-mod.3 | — | sing-box mod | 与 sing-box 配套 |
| Go toolchain | 1.24.7 (runtime 1.24.9) | — | go.mod | sing-box 1.13+ 才需要 1.24 |

## 升级决策记录

### metacubex/utls: v1.8.4 → v1.8.7  ✅ **已升级**

**完整调研见 [RESEARCH_utls_v1.8.7.md](./RESEARCH_utls_v1.8.7.md)**

**调研过程：**
- v1.8.4 → v1.8.7 共 5 个 commit：
  1. `c6c374d4` Hide ALPN in ECH (2026-03-05) — ECH 隐私修复：ALPN 不再泄漏到 outer ClientHello
  2. `28786a97` README 修复 (2026-03-05) — 无代码变更
  3. `5bd8c376` Expose ServerHello (un)marshal (2026-03-05) — 纯新增导出 API
  4. `7700575e` REALITY 16KiB buffer (2026-06-26) — 服务端改动，客户端不受影响
  5. `f7d52c22` lazy init reality server cert (2026-06-26) — 服务端改动，客户端不受影响

**关键证据：**
- sing-box 官方稳定版 v1.13.18 仍用 utls v1.8.4（保守）
- sing-box v1.14.0-beta.14 已升级到 v1.8.7（上游已验证可用）
- sing-box 1.12.x（neko base）使用 ECH（utls_client.go:218）→ **受益于 ALPN 泄漏修复**
- sing-box 1.12.x 不含 REALITY 服务端代码 → 服务端改动无影响
- sing-box 1.12.x 的 ALPN 处理通过字段赋值 + BuildHandshakeState 重建，不依赖 writeToUConn 副作用
- 5 个 commit 无删除/修改导出 API 签名，仅新增 API

**决策：升级到 v1.8.7**
- ECH 隐私修复对 NB4A ECH 用户是实质收益
- 无 API 破坏，向后兼容
- 风险低，最坏情况 ECH 协商失败回退非 ECH
- 回滚方案：git revert

### sing-box: neko 1.12.x → 1.13.x  ⏸️ **暂不升级**

**调研结论：**

**1.13.0 的重要变更（2026-08-09 发布 1.13.18）：**
- 新增：naiveproxy outbound（仅 Apple/Android/Windows/部分 Linux）
- 新增：NaiveProxy QUIC 支持 + ECH 支持
- 新增：CCM/OCM 服务（Claude/OpenAI Code Multiplexer）
- 新增：Tailscale system TUN interface
- 新增：bind_address_no_port、TCP keep-alive 可配
- 新增：kTLS、mTLS、curve preferences、pinned public key SHA256
- 新增：ICMP echo 路由（reject/drop/reply）
- 新增：interface address / preferred_by 路由规则
- 新增：Wi-Fi state 监控（Linux/Windows）
- 新增：auto_redirect pre-match + bypass action
- 新增：Chrome Root Store 证书选项
- **破坏性变更：** 删除 NaiveProxy 的 `certificate_public_key_sha256`
- **要求：** Go 1.24+（我们已满足）
- **Android 5.0 支持将移除**（需 `-legacy-android-5` 单独构建）

**为什么暂不升级：**
1. **neko fork 还在 1.12.x**：MatsuriDayo/sing-box 1.12.x 最新是 `aed32ee` (1.12.19-neko-1, 2026-02-02)，没有 1.13.x 分支
2. **neko fork 有大量自定义 patch**：直接 rebase 到 1.13.x 需要逐个验证 patch 兼容性，工作量巨大
3. **starifly fork 也基于 1.12.x**：starifly/sing-box 1.12.x 最新 `7567ef4` (2026-07-26)，已包含 snell/xhttp 等扩展
4. **1.13.x 的关键新特性**（naiveproxy outbound、CCM/OCM、Tailscale TUN）对 NB4A 用户非必需
5. **稳定性优先**：1.13.18 虽然是稳定版，但 neko fork 生态尚未跟上，盲升会脱离生态

**何时升级：**
- 等 MatsuriDayo/sing-box 创建 1.13.x-neko 分支
- 或 starifly fork 迁移到 1.13.x
- 届时一起升级 sing-box + utls + quic-go + 其他间接依赖

### 其他依赖  ⏸️ **全部保持现状**

**原则：**
- 所有 `github.com/sagernet/*` 包必须与 sing-box 版本配套，不单独升级
- 所有 `github.com/metacubex/*` 包必须与 mihomo/utls 生态配套
- `golang.org/x/*` 包跟随 Go toolchain 版本
- 任何 indirect 依赖升级都需要验证：
  1. 是否被直接依赖（sing-box 等）的 go.mod require
  2. 是否有 breaking change
  3. 是否有 API 变更影响调用方

## 版本检查方法（workflow 用）

**不需要用户提供的 gv.996855.xyz API**（那个是给无法访问 GitHub 的环境用的，我们出口是 GCP 能直连）。

**推荐方法：GitHub API + token**
```bash
# 查最新 release
curl -s -H "Authorization: token $GH_TOKEN" \
  "https://api.github.com/repos/SagerNet/sing-box/releases/latest" \
  | jq -r '.tag_name'

# 查最新 tag
curl -s -H "Authorization: token $GH_TOKEN" \
  "https://api.github.com/repos/SagerNet/sing-box/tags?per_page=5" \
  | jq -r '.[].name'

# 查某 commit 信息
curl -s -H "Authorization: token $GH_TOKEN" \
  "https://api.github.com/repos/MatsuriDayo/sing-box/commits/1.12.x?per_page=1" \
  | jq -r '.[0].sha, .[0].commit.message'
```

**workflow 策略：固定版本**
- 所有依赖版本在 go.mod / get_source_env.sh 中硬编码
- workflow 不自动升级任何版本
- 升级时人工调研 → 修改 VERSION_POLICY.md → 修改版本号 → CI 验证

## 升级检查清单（人工执行）

升级任何依赖前，必须完成：
- [ ] 读取 release notes（所有中间版本）
- [ ] 检查 breaking changes
- [ ] 检查 go.mod / go.sum 是否需要同步更新其他依赖
- [ ] 检查调用方代码是否需要适配
- [ ] 检查 neko fork 是否已跟进（sing-box 系依赖）
- [ ] 在 VERSION_POLICY.md 记录决策
- [ ] CI 构建验证
- [ ] 真机冒烟测试（至少连接 + 测速 + 切换节点）
