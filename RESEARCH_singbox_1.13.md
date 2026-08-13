# sing-box 1.12.x → 1.13.x 升级调研报告

> 调研日期：2026-08-13
> 当前基线：MatsuriDayo/sing-box 1.12.x (`aed32ee` 1.12.19-neko-1) + starifly 扩展 (`7567ef4`)
> 目标版本：SagerNet/sing-box v1.13.18 (稳定版) / v1.14.0-beta.14 (测试版)

## 1. 生态现状

| 仓库 | 1.12.x | 1.13.x | 1.14.x |
|------|--------|--------|--------|
| SagerNet/sing-box（上游） | ✅ 1.12.19 | ✅ 1.13.18 | ✅ beta.14 |
| MatsuriDayo/sing-box（neko fork） | ✅ `aed32ee` | ❌ **无分支** | ❌ 无 |
| starifly/sing-box（扩展 fork） | ✅ `7567ef4` | ❌ **无分支** | ❌ 无 |

**关键事实：** neko 生态（MatsuriDayo + starifly）**尚未启动 1.13.x 迁移**。

## 2. 1.13.0 新特性（2026-01-07 alpha.1 → 2026-08-09 1.13.18）

### 新增功能
- **NaiveProxy outbound**（仅 Apple/Android/Windows/部分 Linux）— 需 libcronet
- NaiveProxy QUIC 支持 + ECH 支持
- **CCM/OCM 服务**（Claude Code / OpenAI Code Multiplexer）
- Tailscale system TUN interface
- `bind_address_no_port`、TCP keep-alive 可配
- kTLS、mTLS、curve preferences、pinned public key SHA256
- ICMP echo 路由（reject/drop/reply）
- interface address / preferred_by 路由规则
- Wi-Fi state 监控（Linux/Windows）
- auto_redirect pre-match + bypass action
- Chrome Root Store 证书选项
- ACME DNS-01 challenge 扩展
- ECH `tls.ech.query_server_name` 选项

### 破坏性变更
- **删除** NaiveProxy 的 `certificate_public_key_sha256` 选项
- **要求** Go 1.24+（我们已满足，go.mod 是 1.24.7）
- Android 5.0 支持将移除（需 `-legacy-android-5` 单独构建）

### Bug 修复（1.13.1-1.13.18）
- 1.13.0-beta.6: utls v1.8.2（Chrome 120+ padding 修复）
- 1.13.18: naiveproxy 更新到 v150.0.7871.63-1
- "Fixes and improvements"（具体未全部列出）

## 3. API 兼容性分析

### ✅ 核心注册 API 未变
- `outbound.Register[Options]` 签名 1.12.x 与 1.13.x **完全一致**
- `outbound.Registry` 结构未变
- `inbound.Registry` / `endpoint.Registry` / `dns.TransportRegistry` 包结构一致

### ⚠️ 协议可用性
| 协议 | 1.12.x 官方 | 1.13.x 官方 | starifly fork 1.12.x | 迁移影响 |
|------|------------|------------|---------------------|----------|
| shadowsocksr | ❌ | ✅（option 有，protocol 404 需确认） | ✅（自定义 patch） | 需移植或用官方 |
| snell | ❌ | ❌ | ✅（自定义 patch） | **必须重新移植** |
| juicity | ❌ | ❌ | ✅（自定义 patch） | **必须重新移植** |
| naive outbound | ❌ | ✅（新增） | ❌ | 可选新增 |

### ⚠️ 间接依赖大幅升级
1.13.18 vs 1.12.19 的依赖变化（节选）：

| 依赖 | 1.12.x | 1.13.x | 变化 |
|------|--------|--------|------|
| sagernet/sing | v0.7.18 | v0.8.12 | **大版本** |
| sagernet/quic-go | v0.52.0-mod.3 | v0.59.0-mod.4 | **大版本** |
| sagernet/gvisor | 20250325 | 20250811 | 升级 |
| sagernet/sing-tun | v0.7.10 | v0.8.12 | **大版本** |
| sagernet/sing-vmess | v0.2.7 | v0.2.8 | 小版本 |
| sagernet/tailscale | v1.80.3-mod.2 | v1.92.4-mod.9 | 大版本 |
| sagernet/wireguard-go | v0.0.1-beta.7 | v0.0.4 | 升级 |
| sagernet/gomobile | v0.1.8 | v0.1.12 | 升级 |
| metacubex/utls | v1.8.4 | v1.8.4 | 不变（1.14 beta 才 v1.8.7） |
| metacubex/tfo-go | 旧 | → database64128/tfo-go/v2 v2.3.2 | **换包名** |
| golang.org/x/crypto | v0.41.0 | v0.48.0 | 升级 |
| golang.org/x/net | v0.43.0 | v0.50.0 | 升级 |
| go-chi/chi | v5.2.2 | v5.2.5 | 小版本 |
| miekg/dns | v1.1.67 | v1.1.72 | 小版本 |
| 新增 | — | anthropic-sdk-go, openai-go, cronet-go, wingoes, keychain | 新依赖 |

**风险：** `sagernet/sing` v0.7→v0.8、`sing-tun` v0.7→v0.8 是大版本升级，API 可能有 breaking change。`tfo-go` 换了包名（metacubex → database64128/v2），所有 import 路径要改。

## 4. NekoBox libcore 调用点影响

NekoBox libcore 重度使用以下 sing-box API：
- `adapter/outbound`, `adapter/inbound`, `adapter/endpoint`, `adapter/service`
- `protocol/*`（注册 20+ 协议）
- `option/*`（配置结构体）
- `dns/*`（DNS transport）
- `common/dialer`, `common/conntrack`, `constant`

升级 1.13.x 需要：
1. **移植 starifly 的 snell patch**（官方无 snell）
2. **移植 starifly 的 juicity patch**（官方无 juicity）
3. **验证 SSR**：1.13 option 有 shadowsocksr.go，但 protocol 目录 404（需 clone 确认是编译 tag 排除还是真没有）
4. **适配 sing v0.8 API 变更**（待具体 diff）
5. **适配 sing-tun v0.8 API 变更**（待具体 diff）
6. **改 tfo-go import 路径**（metacubex → database64128/v2）
7. **适配 quic-go v0.59**（v0.52→v0.59 跨多个版本）
8. **验证 gomobile v0.1.12**（NekoBox 用 gomobile-matsuri，可能不兼容）

## 5. 工作量评估

| 任务 | 难度 | 工时估计 |
|------|------|----------|
| 切换 sing-box commit 到 1.13.18 | 低 | 0.5h |
| 移植 snell patch 到 1.13.x | 中 | 2-4h |
| 移植 juicity patch 到 1.13.x | 中 | 2-4h |
| 验证/移植 SSR | 中 | 2-4h |
| 适配 sing v0.8 API | 高 | 4-8h |
| 适配 sing-tun v0.8 API | 高 | 4-8h |
| 改 tfo-go import | 低 | 1h |
| 适配 quic-go v0.59 | 中 | 2-4h |
| 验证 gomobile 兼容 | 高 | 2-8h |
| 间接依赖 go.sum 重算 | 高 | 需 Go 环境 |
| CI 构建调试 | 高 | 8-16h |
| 真机测试（全协议） | 高 | 4-8h |
| **总计** | — | **~30-65 工时** |

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| sing v0.8 API break 导致编译失败 | 高 | 高 | 需逐文件适配 |
| sing-tun v0.8 TUN 接口变更 | 高 | 高 | VPN 模式可能失效 |
| gomobile v0.1.12 与 matsuri fork 不兼容 | 中 | 高 | 需 fork gomobile |
| snell/juicity patch 移植冲突 | 中 | 中 | 手动 rebase |
| quic-go v0.59 行为变更 | 中 | 中 | Hysteria/TUIC 需测 |
| 1.13.x 本身 bug（虽稳定版） | 低 | 中 | 等几个 patch 版本 |

## 7. 决策

### ❌ **暂不升级到 1.13.x（当前阶段）**

**理由：**

1. **生态未跟上**：MatsuriDayo 和 starifly 的 sing-box fork 都没有 1.13.x 分支。neko fork 有大量自定义 patch（tuic 修改、xhttp、snell、juicity、SSR、wireguard 修复等），这些 patch 都需要重新移植到 1.13.x，工作量巨大且容易出错。

2. **间接依赖大版本升级**：`sagernet/sing` v0.7→v0.8、`sing-tun` v0.7→v0.8 是核心库大版本升级，API 可能有 breaking change，需要逐文件适配。`tfo-go` 换了包名。这些不是简单的版本号改动。

3. **gomobile 兼容性未知**：NekoBox 用的是 `MatsuriDayo/gomobile` fork（gomobile-matsuri），sing-box 1.13.x 要求 `sagernet/gomobile v0.1.12`，两者是否兼容未知，可能需要 fork gomobile。

4. **无 Go 构建环境**：当前环境无 Go 工具链，无法运行 `go mod tidy` 重算 go.sum，无法本地编译验证。升级后只能靠 CI 试错，效率低。

5. **收益有限**：1.13.x 的关键新特性（NaiveProxy outbound、CCM/OCM、Tailscale TUN、ICMP 路由）对 NB4A 用户非必需。NB4A 已通过 plugin 方式支持 naiveproxy。ECH/kTLS 等增量改进价值不高。

6. **稳定性优先**：1.13.18 虽是稳定版，但 neko 生态尚无验证。贸然升级可能引入连接失败、VPN 失效、协议异常等回归 bug。

7. **1.14.x 更不稳定**：beta 版不考虑。

### ✅ **替代方案：保持 1.12.x neko + starifly 扩展，关注上游动态**

- 当前 `starifly/sing-box 1.12.x` (`7567ef4`, 2026-07-26) 是 neko 1.12.x 系列最新，包含 snell/xhttp/juicity 等扩展
- 当 MatsuriDayo 或 starifly 创建 1.13.x 分支时，跟随升级
- 升级时连带 utls、quic-go、sing、sing-tun 等一起升

### ✅ **可独立做的：utls v1.8.7 升级**

utls 是 indirect 依赖，且调研已确认 v1.8.4→v1.8.7 无 API 破坏、有 ECH 隐私修复，可单独升级（已完成）。

## 8. 何时重新评估升级

满足以下任一条件时重新评估：
- [ ] MatsuriDayo/sing-box 创建 1.13.x-neko 分支
- [ ] starifly/sing-box 创建 1.13.x 分支
- [ ] sing-box 1.13.x 发布 3+ 个 patch 版本（1.13.20+，更稳定）
- [ ] 1.13.x 关键 bug 修复（如连接、VPN 相关）
- [ ] 有 Go 构建环境可本地验证
