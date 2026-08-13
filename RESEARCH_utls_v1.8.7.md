# utls v1.8.4 → v1.8.7 升级调研报告

> 本报告基于对 5 个 commit 的完整 patch 审查、sing-box 1.12.x 调用点分析、以及下游项目使用情况。

## 5 个 commit 逐项审查

### 1. `c6c374d4` Hide ALPN in ECH (2026-03-05) — **行为变更**

**改动文件：** `handshake_messages.go`, `u_tls_extensions.go`

**改动内容：**
- `clientHelloMsg.marshalMsgReorderOuterExts`: ECH inner 时不再把 ALPN 加到 outer extension 列表（原来会加，现在用 `echInner && false` 永远跳过）
- `ALPNExtension.writeToUConn`: 当 `config.EncryptedClientHelloConfigList != nil`（ECH 启用）时，**不再**覆写 `config.NextProtos` 和 `Hello.AlpnProtocols`

**目的：** ECH 场景下，ALPN 应该只出现在 inner ClientHello（被加密），不应出现在 outer（明文）。原行为会导致 ALPN 泄漏到 outer，破坏 ECH 的隐私性。

**对 sing-box 1.12.x 的影响：**
- sing-box `utlsALPNWrapper.HandshakeContext`（utls_client.go:122-140）流程：
  1. `BuildHandshakeState()` → 触发 `ALPNExtension.writeToUConn()`
  2. 手动改 `alpnExtension.AlpnProtocols = c.nextProtocols`
  3. 再次 `BuildHandshakeState()` 重建
- **无 ECH 时**：`EncryptedClientHelloConfigList == nil`，v1.8.7 行为与 v1.8.4 完全一致 ✅
- **有 ECH 时**：v1.8.4 会把 `config.NextProtos` 写到 `Hello.AlpnProtocols`（outer），v1.8.7 不写。但 sing-box 步骤 2 之后又 `BuildHandshakeState()` 重建，最终 outer ALPN 由 `ALPNExtension.AlpnProtocols` 字段决定。**结论：sing-box 的 ALPN 逻辑不受影响** ✅
- **副作用：** v1.8.7 修复了 ECH 场景下 ALPN 泄漏到 outer 的隐私 bug。对使用 ECH + ALPN（如 h2）的 NB4A 用户，这是**隐私增强**。

**评级：** 🟢 **安全 + 有益**（ECH 隐私修复，无破坏性）

### 2. `28786a97` README: Fix incorrect link (2026-03-05) — **文档**

**改动文件：** `README.md`（仅文档链接修复）

**评级：** ⚪ **无影响**

### 3. `5bd8c376` Expose ServerHello (un)marshal (2026-03-05) — **新增 API**

**改动文件：** `u_public.go`

**改动内容：** 新增 `UnmarshalServerHello(data []byte) *PubServerHelloMsg` 和 `PubServerHelloMsg.Marshal() ([]byte, error)` 两个导出函数

**目的：** 允许外部代码解析原始 ServerHello 消息（之前只能解析 ClientHello）

**对 sing-box 1.12.x 的影响：**
- grep 确认 sing-box 1.12.x **不使用** `UnmarshalServerHello` 或 `PubServerHelloMsg`
- 纯新增 API，不修改任何现有 API

**评级：** 🟢 **安全**（纯新增，向后兼容）

### 4. `7700575e` increase REALITY target TLS record buffer to 16 KiB (2026-06-26) — **服务端优化**

**改动文件：** `reality.go`

**改动内容：** `realitySize` 常量从 `8192` 改为 `16384`

**目的：** REALITY **服务端**镜像连接的 TLS record 缓冲区从 8KiB 增到 16KiB，避免大 record 被截断

**对 sing-box 1.12.x 的影响：**
- grep 确认 sing-box 1.12.x **不包含 REALITY 服务端代码**（只有 `reality_client.go`）
- NB4A 是客户端应用，不作为 REALITY 服务端
- 此改动只影响 utls 内部的 `realityMirrorConn`，sing-box 客户端不调用

**评级：** 🟢 **安全**（不影响客户端）

### 5. `f7d52c22` lazy init reality server cert (2026-06-26) — **服务端优化**

**改动文件：** `reality.go`

**改动内容：**
- 把 REALITY 服务端证书生成从 `init()` 全局初始化改为 `sync.Once` 懒加载
- 新增内部泛型 helper `onceValues[T1, T2]`
- 只影响 `realityServerHandshakeStateTLS13.handshake()` 路径

**目的：**
- 启动时不再强制生成 REALITY 服务端证书（省启动时间和内存）
- 只在实际有 REALITY 服务端连接时才生成

**对 sing-box 1.12.x 的影响：**
- 同 #4，sing-box 1.12.x 不含 REALITY 服务端代码
- `init()` 改懒加载对客户端无影响（客户端不调用 `realityServerCert()`）
- **间接收益：** 减少启动时的 ed25519 密钥生成和 x509 证书创建开销（虽然客户端原本也不触发）

**评级：** 🟢 **安全**（不影响客户端，启动开销略减）

## 下游项目使用情况

| 项目 | sing-box 版本 | utls 版本 | 说明 |
|------|--------------|-----------|------|
| SagerNet/sing-box v1.13.18 | 1.13.18 | v1.8.4 | 稳定版，保守 |
| SagerNet/sing-box v1.14.0-beta.14 | 1.14.0-beta.14 | v1.8.7 | 测试版，已采用 |
| MatsuriDayo/sing-box 1.12.x | 1.12.19-neko-1 | v1.8.4 | 我们的基础 |

**注意：** 官方 sing-box 1.13.18 仍用 v1.8.4，但这**不代表 v1.8.7 不兼容** —— 只是官方还没切。1.14.0-beta.14 已切到 v1.8.7，说明上游认为 v1.8.7 可用。

## API 兼容性总结

| 检查项 | 结果 |
|--------|------|
| 是否删除任何导出 API | ❌ 无删除 |
| 是否修改任何导出 API 签名 | ❌ 无修改 |
| 是否改变任何导出 API 行为 | ⚠️ `ALPNExtension.writeToUConn` 在 ECH 场景行为变化（但 sing-box 不依赖该副作用） |
| 是否新增导出 API | ✅ 新增 `UnmarshalServerHello` / `PubServerHelloMsg.Marshal` |
| sing-box 1.12.x 是否使用被改动的 API | ✅ 使用 `ALPNExtension`（但通过字段赋值，不依赖 `writeToUConn` 副作用） |
| sing-box 1.12.x 是否使用 ECH | ✅ 是（utls_client.go:218）→ **受益于隐私修复** |
| sing-box 1.12.x 是否含 REALITY 服务端 | ❌ 否 → 服务端改动无影响 |

## 实际收益评估

| 收益类型 | 描述 | 评级 |
|----------|------|------|
| 🔒 隐私修复 | ECH 场景下 ALPN 不再泄漏到 outer ClientHello | 重要（对 ECH 用户） |
| 🚀 启动性能 | REALITY 服务端证书懒加载（客户端不触发，但 init 副作用消除） | 微小 |
| 🛠️ API 扩展 | 新增 ServerHello 解析能力（sing-box 暂未用） | 中性 |
| 🐛 bug 修复 | 无（v1.8.5-v1.8.7 无 bug fix commit） | 无 |
| 🔒 安全修复 | 无 CVE | 无 |

## 最终决策

### ✅ **建议升级到 v1.8.7**

**理由：**
1. **无 API 破坏**：5 个 commit 中无删除/修改导出 API 签名
2. **ECH 隐私修复真实有效**：sing-box 1.12.x 使用 ECH，v1.8.7 修复了 ALPN 泄漏到 outer ClientHello 的隐私问题。对 NB4A 使用 ECH（如 VLESS+ECH、TLS+ECH）的用户是实质收益
3. **REALITY 服务端改动不影响客户端**：sing-box 1.12.x 不含 REALITY 服务端代码
4. **sing-box 1.12.x 的 ALPN 处理逻辑不受影响**：sing-box 通过直接赋值 `ALPNExtension.AlpnProtocols` 字段 + `BuildHandshakeState()` 重建，不依赖 `writeToUConn` 的副作用
5. **上游已验证**：sing-box v1.14.0-beta.14 已采用 v1.8.7
6. **风险可控**：即使 ECH 行为有边缘 case，最坏情况是 ECH 协商失败回退到非 ECH，不会导致整体连接失败（sing-box 有 fallback 逻辑）

**回滚方案：** 若 CI 或真机测试发现 ECH 连接异常，git revert 即可回到 v1.8.4

## 验证清单（升级后需测试）

- [ ] CI 构建通过
- [ ] 普通连接（无 ECH）正常
- [ ] VLESS + TLS + ECH 连接正常
- [ ] Trojan + TLS + ECH 连接正常
- [ ] REALITY 连接正常（客户端）
- [ ] Hysteria2 连接正常
- [ ] ALPN h2/http1.1 协商正确
- [ ] Chrome/Firefox fingerprint 正常
