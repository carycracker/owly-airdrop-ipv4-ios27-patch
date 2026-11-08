# owly-airdrop-ipv4-ios27-patch

让 **iPhone（iOS 27.2）能在这台 Linux 笔记本上像 AirDrop 一样收发文件**的一组补丁。

不是完整的分叉，是**补丁集**：针对上游 [t4t5/owly](https://github.com/t4t5/owly) **v0.4.2** 的 5 处改动，用 `patch -p1` 打在干净的 v0.4.2 源码上。

- 上游项目：https://github.com/t4t5/owly （MIT，作者 T4T5）
- 上游许可：MIT，本补丁集沿用同一许可，版权归上游作者
- 补丁文件：`patches/owly-0.4.2-airdrop-ipv4-ios27.patch`

---

## 适用环境（本补丁的验证范围）

| 项 | 值 |
|---|---|
| 系统 | Omarchy 4.0.6（Arch Linux 系） |
| 内核 | 7.2.15-1-omarchy |
| 无线网卡 | Realtek RTL8852BE，驱动 `rtw89_8852be`，owly Shared 模式 |
| 网络 | **个人网络**：自有 5 GHz 路由器，AP 在信道 149、80 MHz 带宽 |
| 手机 | iPhone 16 Pro Max，**iOS 27.2**，隔空投送设为「所有人，10 分钟」 |
| owly | v0.4.2（AUR `owly`） |

**不适用于**：校园网 / 企业网（客户端隔离、WPA3-Enterprise、AP 信道不在 6/44/149 时会退化到 exclusive 模式，本补丁未覆盖）。

### 修复前的症状

- iPhone 的分享面板里**始终看不到这台电脑**。
- 链路层完全正常：`owly doctor` 报 `ready`，`owly radio up` 起得来，`awdl0` 在，双方 AWDL peer 同步成功（日志里能看到 `add peer` 和 rssi）。
- 也就是说：**能听见彼此，但 AirDrop 协议层一句话都说不上**。

---

## 五处改动

### 1. 发布并应答 A（IPv4）记录 — `crates/owly-airdrop/src/proto/mdns.rs`

原实现只发布 `PTR / SRV / TXT / AAAA`，`answer()` 也只处理这四种。实测抓包（本机可复现，不需要手机）：

```
Q _airdrop._tcp.local          PTR   -> 有应答
Q owly-<id>.local              AAAA  -> 有应答
Q owly-<id>.local              A     -> 没有任何应答   ← iPhone 问的正是这条
```

iPhone 会去解析 SRV 里的主机名，并且在拿到 AAAA 之后仍然要问 A 记录；`answer()` 返回 `None`，它就没有可连的地址。

改动：`Service` 新增 `addr4: Option<Ipv4Addr>`；`records()` 在有 IPv4 时追加一条 `A` 记录；`answer()` 应答主机名的 A 查询，并在回答 PTR 时把它放进 additional。

### 2. QU / legacy 查询不做 known-answer 抑制 — 同上

iPhone 的查询带 **QU 位**（要求单播应答），并且请求里带了它刚从我们组播里学到的 PTR 记录。原实现按 RFC 6762 §7.1 一律抑制：

```
questions=[("_airdrop._tcp.local", PTR, unicast=true)]
known_answers=[("_airdrop._tcp.local", PTR, ttl=4500, ...)]
-> nothing to answer
```

而 §5.4 的抑制只针对组播应答；QU 查询和 legacy（非 5353 源端口）查询都期望得到应答。

改动：`known` 判定加上 `!q.unicast && !legacy`。

### 3. 单播应答补 scope，失败不再静默 — `crates/owly-airdrop/src/driver/mdns.rs`

```rust
let to = if reply.unicast { from } else { group };
let _ = socket.send_to(&reply.message, to).await;   // 错误被吞掉
```

链路本地地址如果 `scope_id` 为 0，`send_to` 会失败，而错误被 `let _ =` 丢弃 —— 表现就是"应答生成了但手机收不到"，日志里什么都看不到。

改动：单播应答的目标地址强制带上 `self.scope_id`；发送失败用 `tracing::warn!` 输出；同时补了查询/应答的 debug 日志（排查这类问题的关键）。

### 4. `renameat2(RENAME_NOREPLACE)` 在 FUSE/ntfs-3g 上回退 — `crates/owly-airdrop/src/driver/staging.rs`

收完文件后要把 staging 里的文件原子地挪进目标目录。**ntfs-3g 不支持 `RENAME_NOREPLACE`，直接返回 EINVAL**：

```
<- POST /Upload
transfer failed: Invalid argument (os error 22)
-> HTTP/1.1 500 Internal Server Error
```

手机侧显示「AirDrop failed」。文件本体是收全了的，只是最后这一步改名失败。

改动：遇到 `EINVAL` / `EOPNOTSUPP` 时回退到"先 `symlink_metadata` 检查、再 `rename`"（非 Linux 分支原本就是这个逻辑）。在 ext4/btrfs 上行为不变。

### 5. 固定服务 id，避免手机上堆"幽灵"条目 — `crates/owly-cli/src/airdrop.rs`

原实现每次启动都随机生成服务 id，于是每重启一次，iPhone 的分享面板里就多一个同名的死条目（点它一直 waiting）。

改动：`OWLY_SERVICE_ID` 环境变量优先，其次是 `$XDG_STATE_HOME/owly/service-id`，都没有才生成并写盘。另由该接口的 `ip -4` 结果填 `addr4`（改动 1 的数据来源）。

---

## 系统侧的两步（不属于代码改动）

这台机器上还有两个外部条件，否则补丁也跑不起来：

### a. 防火墙放行 awdl0

```bash
sudo ufw allow in on awdl0 proto udp to any port 5353    # mDNS 查询/应答
sudo ufw allow in on awdl0 proto tcp to any port 8771    # AirDrop 传输
```

不放行 mDNS 时，iPhone 的单播查询会被丢掉，表现同样是"找不到人"（内核日志里是 `[UFW BLOCK] IN=awdl0 ... DPT=5353`）。

### b. IPv6 链路本地路由

本机跑 sing-box（TUN 模式），它的 IPv6 封锁规则是：

```
9000:  from all unreachable
```

这条规则排在 `lookup main` 之前，于是**所有 IPv6 单播**都返回 `ENETUNREACH`，而 mDNS 的应答（单播）恰好是 IPv6。组播不受影响，所以广播一直正常、只有应答发不出去 —— 这个现象非常误导人。

```bash
sudo ip -6 rule add to fe80::/10 lookup main pref 400
```

只放行链路本地 `fe80::/10`，不动整体 IPv6 封锁。**这条规则不持久**，重启或 sing-box 重启后需要重新添加（可用 systemd timer 幂等补齐）。

另外，`A` 记录要求 `awdl0` 上有 IPv4，可用链路本地地址：

```bash
sudo ip addr add 169.254.87.1/16 dev awdl0
```

---

## 构建与安装

```bash
# 1. 拿干净的 v0.4.2 源码
curl -L https://github.com/t4t5/owly/archive/refs/tags/v0.4.2.tar.gz -o owly-0.4.2.tar.gz
tar xzf owly-0.4.2.tar.gz && cd owly-0.4.2

# 2. 打补丁
patch -p1 < ../patches/owly-0.4.2-airdrop-ipv4-ios27.patch

# 3. 编译（需要 rust 1.88+）
cargo build --release --locked -p owly-cli

# 4. 安装到 /usr/local/bin，优先于包管理器那份，不动 pacman 的包
sudo install -m755 target/release/owly /usr/local/bin/owly
```

验证（不需要手机）：

```bash
owly doctor                      # 应显示 ready
owly radio up &                  # 起 awdl0
owly receive --dir ~/Downloads --accept
# 在另一个终端向本机查询，应有 A / AAAA / PTR 三种应答
```

手机侧：隔空投送设为「所有人，10 分钟」，分享面板里应出现主机名；点它即可发送。

---

## 已知限制

- 只在一个网卡（RTL8852BE）、一个内核、一个 iOS 版本上验证过。其它组合未测。
- 补丁 1 依赖 `awdl0` 上有 IPv4 地址；没有时不发布 A 记录（行为退化成上游）。
- 补丁 5 固定服务 id 后，之前产生的旧条目会在 iPhone 上自然过期（mDNS TTL），不会自动消失。
- 上游若已修复其中任何一条，对应补丁可以直接丢弃。

## 隐私

本补丁与文档**不含任何 MAC 地址、IPv6 链路本地地址、内网 IP、主机名或设备序列号**。文中出现的 `fe80::1` 是上游测试用例里的固定值，`169.254.87.1` 是通用链路本地示例地址。

## 许可

MIT，与上游一致。上游代码版权归 [T4T5](https://github.com/t4t5) 所有；本仓库只包含针对 v0.4.2 的补丁与其说明。
