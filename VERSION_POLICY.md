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

### sing-box: neko 1.12.x → 1.13.x  ❌ **暂不升级（基于完整调研）**

**完整调研见 [RESEARCH_singbox_1.13.md](./RESEARCH_singbox_1.13.md)**

**调研维度：**
1. 生态现状：MatsuriDayo + starifly 的 sing-box fork **均无 1.13.x 分支**
2. 1.13.0 新特性：NaiveProxy outbound、CCM/OCM、Tailscale TUN、ICMP 路由、kTLS、ECH 增强
3. 破坏性变更：删除 NaiveProxy `certificate_public_key_sha256`、要求 Go 1.24+、Android 5.0 即将弃支持
4. API 兼容性：`outbound.Register` 签名一致 ✅，但 `sing` v0.7→v0.8、`sing-tun` v0.7→v0.8 大版本升级，`tfo-go` 换包名
5. 协议可用性：1.13 官方**无 snell/juicity**（starifly patch 必须重新移植），SSR option 有但 protocol 需确认
6. 间接依赖：quic-go v0.52→v0.59、gomobile v0.1.8→v0.1.12、tailscale v1.80→v1.92、新增 anthropic/openai/cronet SDK
7. 工作量估计：~30-65 工时（含 patch 移植、API 适配、CI 调试、真机测试）
8. 风险：sing/sing-tun API break、gomobile 兼容、quic-go 行为变更、neko patch 冲突

**决策：暂不升级**
- 生态未跟上（neko fork 无 1.13.x 分支）
- 间接依赖大版本升级，API 适配工作量大
- 无 Go 构建环境，无法本地验证
- 收益有限（关键新特性对 NB4A 用户非必需）
- 稳定性优先（1.13.18 虽稳定但 neko 生态未验证）

**替代方案：保持 1.12.x neko + starifly 扩展**
- 当前 `starifly/sing-box 1.12.x` (`7567ef4`, 2026-07-26) 是 neko 1.12.x 最新
- 等 MatsuriDayo/starifly 创建 1.13.x 分支后跟随升级

**重新评估条件：**
- MatsuriDayo/starifly 创建 1.13.x 分支
- sing-box 1.13.20+（更稳定）
- 有 Go 构建环境

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
