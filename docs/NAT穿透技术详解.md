# NAT 穿透技术详解

> 深入理解 NAT 穿透技术原理，掌握 STUN、TURN、ICE 等核心穿透方案。

---

## 1. NAT 基础知识

### 1.1 什么是 NAT？

**NAT（Network Address Translation）** 是一种将私有 IP 地址转换为公网 IP 地址的技术，用于解决 IPv4 地址短缺问题。

```
┌────────────────────────────────────────────────────────────────┐
│                      NAT 工作原理                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   私有网络                          公网                       │
│  ┌─────────┐                      ┌─────────┐                │
│  │ 内网主机 │                      │ 公网服务器│                │
│  │10.0.0.5 │                      │203.0.113│                │
│  └────┬────┘                      └────┬────┘                │
│       │                                │                      │
│       │ 源: 10.0.0.5:12345             │                      │
│       │ 目: 203.0.113.5:80             │                      │
│       │ ──────────────►                │                      │
│    ┌──┴──┐                             │                      │
│    │ NAT │ 转换                         │                      │
│    │ 网关│                             │                      │
│    └──┬──┘                             │                      │
│       │ 源: 198.51.100.1:54321         │                      │
│       │ 目: 203.0.113.5:80             │                      │
│       │ ──────────────────────────────►│                      │
│       │                                │                      │
│       │         NAT 映射表             │                      │
│       │  ┌───────────────────────────┐│                      │
│       │  │ 内网地址        │ 外网地址 ││                      │
│       │  │ 10.0.0.5:12345 │ 198.51.100.1:54321 │             │
│       │  └───────────────────────────┘│                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 NAT 类型分类

```
┌────────────────────────────────────────────────────────────────┐
│                      NAT 类型分类                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Full Cone NAT（完全锥形）                                   │
│     ────────────────────────                                   │
│     特点：一旦建立映射，任何外部主机都可以通过该映射发送数据     │
│                                                                │
│     内网主机 ───► NAT ───► 映射地址                             │
│     (10.0.0.5:1234)    (198.51.100.1:5000)                     │
│                                                                │
│     任何外部主机 ◄──── 都可以发送到 198.51.100.1:5000          │
│                                                                │
│     穿透难度：★★☆☆☆ 最容易                                    │
│                                                                │
│  2. Restricted Cone NAT（受限锥形）                             │
│     ──────────────────────────────                             │
│     特点：只有内网主机曾经发送过数据的外部 IP 才能发送回来       │
│                                                                │
│     内网主机 ───► NAT ───► 外部主机 A (203.0.113.5)             │
│                                                                │
│     外部主机 A ◄──── 可以发送回来                               │
│     外部主机 B ✖──── 无法发送（IP 不在允许列表）                 │
│                                                                │
│     穿透难度：★★★☆☆ 中等                                      │
│                                                                │
│  3. Port Restricted Cone NAT（端口受限锥形）                    │
│     ──────────────────────────────────────                     │
│     特点：只有内网主机曾经发送过数据的外部 IP:Port 才能发送回来  │
│                                                                │
│     内网主机 ───► NAT ───► 外部主机 A:80                        │
│                                                                │
│     外部主机 A:80 ◄──── 可以发送回来                            │
│     外部主机 A:81 ✖──── 无法发送（端口不匹配）                   │
│                                                                │
│     穿透难度：★★★☆☆ 中等                                      │
│                                                                │
│  4. Symmetric NAT（对称型）                                     │
│     ─────────────────────                                       │
│     特点：对不同的目标地址使用不同的映射                         │
│                                                                │
│     内网主机 ───► NAT ───► 外部主机 A ──► 映射 1                 │
│     内网主机 ───► NAT ───► 外部主机 B ──► 映射 2（不同！）       │
│                                                                │
│     相同的内网地址，不同的目标，产生不同的映射                   │
│                                                                │
│     穿透难度：★★★★★ 最难                                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 NAT 类型对比

| NAT 类型 | 映射规则 | 穿透难度 | 常见场景 |
|----------|----------|----------|----------|
| Full Cone | 一对一映射 | 低 | 家用路由器 |
| Restricted Cone | IP 受限 | 中 | 企业防火墙 |
| Port Restricted Cone | IP+Port 受限 | 中 | 企业防火墙 |
| Symmetric | 目标相关 | 高 | 移动网络、严格防火墙 |

---

## 2. STUN 协议详解

### 2.1 STUN 是什么？

**STUN（Session Traversal Utilities for NAT）** 是一种用于检测 NAT 类型和获取公网 IP 地址的协议。

```
┌────────────────────────────────────────────────────────────────┐
│                    STUN 工作原理                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   内网主机                         STUN 服务器                   │
│  ┌─────────┐                     ┌───────────┐                │
│  │ 10.0.0.5│                     │ 公网 IP   │                │
│  │         │                     │ 已知      │                │
│  └────┬────┘                     └─────┬─────┘                │
│       │                                │                       │
│       │ 1. STUN Binding Request        │                       │
│       │    源: 10.0.0.5:12345          │                       │
│       │ ──────────────────────────────►│                       │
│       │                                │                       │
│       │ 2. STUN Binding Response       │                       │
│       │    反射地址: 198.51.100.1:54321│                       │
│       │ ◄──────────────────────────────│                       │
│       │                                │                       │
│       │ 内网主机现在知道自己的公网地址  │                       │
│       │ 198.51.100.1:54321             │                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 STUN 消息格式

```
STUN 消息头格式（20 字节）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0 0|     STUN Message Type     |         Message Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Magic Cookie                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                     Transaction ID (96 bits)                  |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

消息类型：
- 0x0001: Binding Request
- 0x0101: Binding Response
- 0x0111: Binding Error Response

Magic Cookie: 0x2112A442 (固定值)

Transaction ID: 96 位唯一标识符
```

### 2.3 STUN 属性

| 属性 | 类型值 | 说明 |
|------|--------|------|
| **MAPPED-ADDRESS** | 0x0001 | 映射地址（NAT 后的公网地址） |
| **XOR-MAPPED-ADDRESS** | 0x0020 | XOR 编码的映射地址 |
| **CHANGE-REQUEST** | 0x0003 | 请求改变 IP/端口 |
| **RESPONSE-ORIGIN** | 0x802B | 响应来源地址 |
| **OTHER-ADDRESS** | 0x802C | 备用服务器地址 |

### 2.4 NAT 类型检测流程

```
┌────────────────────────────────────────────────────────────────┐
│                   NAT 类型检测流程                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  测试 1：发送 Binding Request 到 STUN 服务器                   │
│  ───────────────────────────────────────────                  │
│  客户端 ──► STUN 服务器（IP1:Port1）                           │
│                                                                │
│  结果分析：                                                    │
│  - 收到响应？                                                  │
│    - 否 → UDP 被阻断                                          │
│    - 是 → 比较映射地址和本地地址                               │
│      - 相同 → 无 NAT（公网 IP）                               │
│      - 不同 → 有 NAT，继续测试                                │
│                                                                │
│  测试 2：发送到 STUN 服务器的不同端口                          │
│  ───────────────────────────────────────────                  │
│  客户端 ──► STUN 服务器（IP1:Port2）                          │
│                                                                │
│  结果分析：                                                    │
│  - 映射地址是否改变？                                          │
│    - 改变 → Symmetric NAT                                     │
│    - 不变 → 继续测试 3                                        │
│                                                                │
│  测试 3：请求从不同 IP/Port 响应                               │
│  ───────────────────────────────────────────                  │
│  客户端 ──► STUN 服务器（IP1:Port1，请求从 IP2:Port2 响应）   │
│                                                                │
│  结果分析：                                                    │
│  - 收到响应？                                                  │
│    - 是 → Full Cone NAT                                       │
│    - 否 → 继续测试 4                                          │
│                                                                │
│  测试 4：请求从不同 Port 响应（相同 IP）                       │
│  ───────────────────────────────────────────                  │
│  客户端 ──► STUN 服务器（IP1:Port1，请求从 IP1:Port2 响应）   │
│                                                                │
│  结果分析：                                                    │
│  - 收到响应？                                                  │
│    - 是 → Restricted Cone NAT                                 │
│    - 否 → Port Restricted Cone NAT                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.5 STUN 服务器配置

```cpp
// WebRTC STUN 服务器配置
webrtc::PeerConnectionInterface::IceServer stun_server;
stun_server.urls.push_back("stun:stun.l.google.com:19302");
stun_server.urls.push_back("stun:stun1.l.google.com:19302");

// 自建 STUN 服务器（coturn）
stun_server.urls.push_back("stun:stun.minimeeting.com:3478");

// 添加到配置
webrtc::PeerConnectionInterface::RTCConfiguration config;
config.servers.push_back(stun_server);
```

---

## 3. TURN 协议详解

### 3.1 TURN 是什么？

**TURN（Traversal Using Relays around NAT）** 是一种中继协议，当直接 P2P 连接无法建立时，通过服务器中继传输数据。

```
┌────────────────────────────────────────────────────────────────┐
│                    TURN 中继原理                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   客户端 A                         TURN 服务器                 │
│  ┌─────────┐                     ┌───────────┐                │
│  │ 对称 NAT │                     │ 公网 IP   │                │
│  │无法直连  │                     │           │                │
│  └────┬────┘                     └─────┬─────┘                │
│       │                                │                       │
│       │ TURN Allocate                  │                       │
│       │ ──────────────────────────────►│                       │
│       │                                │                       │
│       │ 分配中继地址                    │                       │
│       │ 198.51.100.1:50000             │                       │
│       │ ◄──────────────────────────────│                       │
│       │                                │                       │
│       │ 通过中继发送数据                │                       │
│       │ ──────────────────────────────►│                       │
│       │                                │                       │
│       │                         转发给客户端 B                  │
│       │                         ─────────┐                     │
│       │                                  │                     │
│       │                                  ▼                     │
│       │                            ┌─────────┐                 │
│       │                            │ 客户端 B │                 │
│       │                            │ 对称 NAT │                 │
│       │                            └─────────┘                 │
│                                                                │
│   客户端 A 和 B 都通过 TURN 服务器中继通信                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 TURN 消息类型

```
┌────────────────────────────────────────────────────────────────┐
│                     TURN 消息类型                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  分配方法（Allocate）                                          │
│  ──────────────────                                            │
│  - Allocate: 请求分配中继地址                                  │
│  - Refresh: 刷新分配                                           │
│  - CreatePermission: 创建权限                                  │
│  - ChannelBind: 绑定通道                                       │
│                                                                │
│  数据传输方法                                                  │
│  ────────────────                                              │
│  - Send: 发送数据（需指定目标地址）                             │
│  - Data: 接收数据（包含来源地址）                               │
│                                                                │
│  通道方法（高效传输）                                          │
│  ────────────────────                                          │
│  - ChannelBind: 创建通道（4 字节头代替 Send 的属性头）          │
│  - ChannelData: 通道数据                                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 3.3 TURN 认证

```
TURN 认证流程：

1. 长期凭证（Long-Term Credential）
   ───────────────────────────────
   用户名和密码预配置在服务器

   客户端 ──► Allocate Request（无认证）
            ◄── 401 Unauthorized（包含 REALM 和 NONCE）
            ──► Allocate Request（含认证信息）
                - USERNAME
                - REALM
                - NONCE
                - MESSAGE-INTEGRITY（HMAC-SHA1）

2. 短期凭证（Short-Term Credential）
   ────────────────────────────────
   通过其他方式获取临时凭证

   - 更安全，适合 WebRTC
   - 凭证有时效性

认证信息计算：
MESSAGE-INTEGRITY = HMAC-SHA1(password, message)
```

### 3.4 TURN 服务器配置

```cpp
// TURN 服务器配置
webrtc::PeerConnectionInterface::IceServer turn_server;
turn_server.urls.push_back("turn:turn.minimeeting.com:3478");
turn_server.urls.push_back("turns:turn.minimeeting.com:5349");  // TLS
turn_server.username = "username";
turn_server.password = "password";

// 添加到配置
config.servers.push_back(turn_server);

// TURN 传输协议
turn_server.urls.push_back("turn:turn.minimeeting.com:3478?transport=tcp");
turn_server.urls.push_back("turn:turn.minimeeting.com:3478?transport=udp");
```

### 3.5 coturn 服务器搭建

```bash
# 安装 coturn
sudo apt-get install coturn

# 配置文件 /etc/turnserver.conf
listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
relay-ip=YOUR_SERVER_IP
external-ip=YOUR_PUBLIC_IP

# 认证
realm=minimeeting.com
lt-cred-mech
user=username:password

# TLS 证书
cert=/etc/ssl/certs/cert.pem
pkey=/etc/ssl/private/key.pem

# 启动服务
sudo systemctl start coturn
sudo systemctl enable coturn

# 验证服务
turnutils_uclient -v -t -u username -w password turn.minimeeting.com
```

---

## 4. ICE 协议详解

### 4.1 ICE 概述

**ICE（Interactive Connectivity Establishment）** 是一个综合性的 NAT 穿透框架，整合了 STUN 和 TURN。

```
┌────────────────────────────────────────────────────────────────┐
│                    ICE 架构图                                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│                     ICE 框架                                   │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                                                        │   │
│  │   候选收集          连通性检测        候选选择          │   │
│  │  ┌─────────┐      ┌─────────┐      ┌─────────┐        │   │
│  │  │ Local   │      │ STUN    │      │ 优先级  │        │   │
│  │  │ STUN    │ ───► │ Binding │ ───► │ 排序   │        │   │
│  │  │ TURN    │      │ Check   │      │ 选择   │        │   │
│  │  └─────────┘      └─────────┘      └─────────┘        │   │
│  │       │                 │                │            │   │
│  │       ▼                 ▼                ▼            │   │
│  │  收集候选：         检测可达性：      选择最佳：        │   │
│  │  - host             - 发送 STUN       - 优先级最高    │   │
│  │  - srflx (STUN)     - 等待响应        - 可达的候选对  │   │
│  │  - relay (TURN)     - 验证成功                         │   │
│  │                                                        │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 ICE 候选收集

```cpp
// ICE 候选收集过程
void collectCandidates(PeerConnection* pc) {
    // 1. Host 候选
    // 从本地网络接口获取
    for (auto& interface : NetworkInterface::getAll()) {
        Candidate host;
        host.type = "host";
        host.address = interface.ip;
        host.port = allocatePort();
        host.priority = calculatePriority("host", interface);
        
        pc->addLocalCandidate(host);
    }
    
    // 2. Server Reflexive 候选（通过 STUN）
    // 向 STUN 服务器发送 Binding Request
    stunClient->sendBindingRequest(stunServer, [](const Address& mapped) {
        Candidate srflx;
        srflx.type = "srflx";
        srflx.address = mapped.ip;
        srflx.port = mapped.port;
        srflx.relatedAddress = localAddress;
        
        pc->addLocalCandidate(srflx);
    });
    
    // 3. Relay 候选（通过 TURN）
    // 向 TURN 服务器请求分配
    turnClient->allocate(turnServer, [](const Address& relayed) {
        Candidate relay;
        relay.type = "relay";
        relay.address = relayed.ip;
        relay.port = relayed.port;
        relay.relatedAddress = mappedAddress;
        
        pc->addLocalCandidate(relay);
    });
}
```

### 4.3 候选优先级计算

```
优先级计算公式：

priority = (2^24) * (type preference) +
           (2^8) * (local preference) +
           (2^0) * (256 - component ID)

参数说明：
──────────
type preference（类型偏好）：
  - host:   126  （最高）
  - prflx:  110
  - srflx:  100
  - relay:  0    （最低）

local preference（本地偏好）：
  - IPv6:  50
  - IPv4:  10
  - 根据网络类型调整

component ID（组件 ID）：
  - RTP:  1
  - RTCP: 2

计算示例：
──────────
Host 候选（IPv4，RTP）：
= 2^24 * 126 + 2^8 * 10 + 256 - 1
= 2113929215 + 2560 + 255
= 2113932030

Relay 候选（IPv4，RTP）：
= 2^24 * 0 + 2^8 * 10 + 256 - 1
= 0 + 2560 + 255
= 2815
```

### 4.4 连通性检测流程

```
┌────────────────────────────────────────────────────────────────┐
│                  连通性检测详细流程                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  客户端 A                                        客户端 B       │
│  候选: A1(host), A2(srflx)              候选: B1(host), B2(srflx)│
│                                                                │
│  候选对（由高到低优先级）：                                     │
│  1. A1-B1 (host-host)                                          │
│  2. A1-B2 (host-srflx)                                         │
│  3. A2-B1 (srflx-host)                                         │
│  4. A2-B2 (srflx-srflx)                                        │
│                                                                │
│  检测过程（以 A1-B1 为例）：                                    │
│                                                                │
│  1. A 发送 STUN Binding Request                                │
│     ┌───────────────────────────────────────────────────────┐ │
│     │ 源: A1 (192.168.1.100:5000)                           │ │
│     │ 目: B1 (10.0.0.5:5001)                                │ │
│     │ 包含: ICE-CONTROLLING, PRIORITY, USE-CANDIDATE        │ │
│     └───────────────────────────────────────────────────────┘ │
│     A1 ──────────────────────────────────────────────────► B1 │
│                                                                │
│  2. B 收到请求，记录来源地址                                    │
│     如果来源地址与 B 知道的 A 的候选不同，发现新的 prflx 候选    │
│                                                                │
│  3. B 发送 STUN Binding Response                               │
│     ┌───────────────────────────────────────────────────────┐ │
│     │ 映射地址: A 的公网地址（如果经过 NAT）                  │ │
│     │ 包含: MAPPED-ADDRESS, XOR-MAPPED-ADDRESS              │ │
│     └───────────────────────────────────────────────────────┘ │
│     B1 ◄────────────────────────────────────────────────── A1 │
│                                                                │
│  4. A 收到响应，确认连通性                                      │
│     标记 A1-B1 为 valid pair                                   │
│                                                                │
│  5. 双方继续检测其他候选对，直到找到最佳路径                    │
│                                                                │
│  ICE 角色控制：                                                │
│  - CONTROLLING: Offer 发送方，选择最终候选对                   │
│  - CONTROLLED: Offer 接收方，接受对方选择                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4.5 ICE 状态转换

```
ICE 连接状态：

  new
   │
   │ start gathering candidates
   ▼
checking ───────────────────────────┐
   │                                 │
   │ found working candidate pair    │ all checks failed
   ▼                                 │
connected ◄──────────────────────────┘
   │                                 
   │ all checks completed
   ▼
completed ──────┐
   │            │
   │ connection │ lost all connections
   │ issues     ▼
   │        disconnected ───────────────► closed
   │            │
   │            │ reconnection
   ▼            ▼
failed ◄──── connected

状态说明：
- new: 初始状态
- checking: 正在检测候选对
- connected: 至少有一个工作连接
- completed: 所有检测完成
- failed: 连接失败
- disconnected: 连接断开
- closed: 连接关闭
```

### 4.6 Trickle ICE

```
传统 ICE vs Trickle ICE：

传统 ICE：
────────
1. 收集所有候选
2. 等待收集完成
3. 一次性发送所有候选
4. 开始连通性检测

缺点：需要等待所有候选收集完成，延迟高

Trickle ICE：
───────────
1. 收集到候选立即发送
2. 边收集边检测
3. 发现有效候选立即使用

优点：更快建立连接

实现示例：
───────────

// WebRTC 原生 API
pc.onicecandidate = (event) => {
    if (event.candidate) {
        // 立即发送候选到远程
        signaling.send({
            type: 'ice_candidate',
            candidate: event.candidate.toJSON()
        });
    } else {
        // 候选收集完成
        console.log('ICE gathering complete');
    }
};

// SDP 中标记支持 Trickle
a=ice-options:trickle
```

---

## 5. MiniMeeting NAT 穿透方案

### 5.1 穿透策略

```
┌────────────────────────────────────────────────────────────────┐
│                MiniMeeting NAT 穿透策略                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  优先级顺序：                                                  │
│                                                                │
│  1. 直连（P2P）                                                │
│     ─────────────                                              │
│     条件：同局域网 或 Full Cone NAT                            │
│     延迟：< 50ms                                               │
│     带宽：无限制                                               │
│                                                                │
│  2. STUN 穿透                                                  │
│     ───────────                                                │
│     条件：非对称 NAT                                           │
│     延迟：50-100ms                                             │
│     带宽：无限制                                               │
│                                                                │
│  3. TURN 中继                                                  │
│     ───────────                                                │
│     条件：对称 NAT 或严格防火墙                                │
│     延迟：100-200ms                                            │
│     带宽：受服务器限制                                         │
│                                                                │
│  决策流程：                                                    │
│                                                                │
│  收集候选 ──► 排序 ──► 检测 ──► 选择                           │
│      │          │        │        │                            │
│      ▼          ▼        ▼        ▼                            │
│   host       优先级高   发送     host 直连                     │
│   srflx      的先检测   STUN    成功则使用                     │
│   relay      请求      否则尝试 srflx                          │
│                              最后尝试 relay                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 5.2 ICE 配置

```cpp
// 完整 ICE 配置
webrtc::PeerConnectionInterface::RTCConfiguration config;

// STUN 服务器
webrtc::PeerConnectionInterface::IceServer stun;
stun.urls.push_back("stun:stun.minimeeting.com:3478");
stun.urls.push_back("stun:stun.l.google.com:19302");
config.servers.push_back(stun);

// TURN 服务器
webrtc::PeerConnectionInterface::IceServer turn;
turn.urls.push_back("turn:turn.minimeeting.com:3478");
turn.urls.push_back("turn:turn.minimeeting.com:3478?transport=tcp");
turn.urls.push_back("turns:turn.minimeeting.com:5349");
turn.username = generateUsername();  // 动态生成
turn.password = generatePassword();
config.servers.push_back(turn);

// ICE 传输策略
config.type = webrtc::PeerConnectionInterface::kAll;

// 候选网络策略
config.network_preference = webrtc::PeerConnectionInterface::kNetworkPreferenceLowCost;

// 候选池大小
config.ice_candidate_pool_size = 10;

// 超时设置
config.ice_connection_reception_timeout = 1000;  // ms
```

### 5.3 NAT 类型适配

```
┌────────────────────────────────────────────────────────────────┐
│                  NAT 类型适配方案                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  场景 1：Full Cone NAT                                         │
│  ────────────────────────                                      │
│  A: Full Cone ───► B: Full Cone                                │
│                     │                                          │
│                     ▼                                          │
│              直接 P2P 连接 ✓                                   │
│              使用 srflx 候选                                   │
│                                                                │
│  场景 2：Symmetric NAT 对 Symmetric NAT                        │
│  ─────────────────────────────────────────                     │
│  A: Symmetric ───► B: Symmetric                                │
│        │              │                                        │
│        │  STUN 穿透失败                                       │
│        ▼              ▼                                        │
│      使用 TURN 中继                                           │
│                                                                │
│  场景 3：混合 NAT 类型                                         │
│  ─────────────────────                                         │
│  A: Full Cone ───► B: Symmetric                                │
│        │              │                                        │
│        │   A 可以主动连接 B                                   │
│        │   B 的 NAT 只允许 A 的响应                           │
│        ▼              ▼                                        │
│      P2P 连接可能成功                                         │
│      如果失败则使用 TURN                                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 6. 穿透问题诊断

### 6.1 常见穿透失败原因

| 问题 | 症状 | 诊断方法 | 解决方案 |
|------|------|----------|----------|
| **STUN 不可达** | 无 srflx 候选 | 检查 STUN 服务器连通性 | 检查防火墙，更换 STUN 服务器 |
| **TURN 认证失败** | 401 错误 | 检查用户名密码 | 验证凭证配置 |
| **对称 NAT** | 只有 host 候选 | NAT 类型检测 | 配置 TURN 服务器 |
| **防火墙阻断** | 无任何候选 | 端口扫描 | 开放 UDP 端口 |
| **UPnP 未启用** | 无法自动映射 | 检查路由器设置 | 启用 UPnP 或手动映射 |

### 6.2 诊断工具

```bash
# 1. 检测 NAT 类型
stun stun.l.google.com:19302

# 2. 测试 STUN 连通性
turnutils_uclient -v -t stun.minimeeting.com

# 3. 测试 TURN 服务
turnutils_uclient -v -t -u username -w password turn.minimeeting.com

# 4. 端口扫描
nc -zuv stun.minimeeting.com 3478

# 5. 抓包分析
tcpdump -i any udp port 3478 -w stun.pcap
```

### 6.3 日志分析

```javascript
// ICE 候选收集日志
console.log('ICE candidate:', {
    type: candidate.type,           // host/srflx/relay
    protocol: candidate.protocol,   // udp/tcp
    address: candidate.address,
    port: candidate.port,
    priority: candidate.priority,
    relatedAddress: candidate.relatedAddress,
    relatedPort: candidate.relatedPort
});

// 判断穿透成功条件
if (candidate.type === 'srflx') {
    console.log('✓ STUN 穿透成功，获得公网地址');
} else if (candidate.type === 'relay') {
    console.log('⚠ 使用 TURN 中继');
} else if (candidate.type === 'host') {
    console.log('本地候选，可能无法穿透 NAT');
}
```

---

## 7. TURN 服务器部署

### 7.1 coturn 安装配置

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install coturn

# CentOS/RHEL
sudo yum install coturn

# 配置文件
sudo vi /etc/turnserver.conf
```

### 7.2 完整配置示例

```conf
# /etc/turnserver.conf

# 监听端口
listening-port=3478
tls-listening-port=5349

# 监听地址
listening-ip=0.0.0.0
relay-ip=YOUR_SERVER_IP
external-ip=YOUR_PUBLIC_IP

# 域名
realm=minimeeting.com

# 认证方式
lt-cred-mech

# 用户账号（可多个）
user=admin:password123
user=webrtc:webrtc123

# TLS 证书
cert=/etc/letsencrypt/live/minimeeting.com/fullchain.pem
pkey=/etc/letsencrypt/live/minimeeting.com/privkey.pem

# 安全选项
no-multicast-peers
no-cli

# 日志
log-file=/var/log/turnserver.log
verbose

# 性能调优
min-port=49152
max-port=65535
max-bps=0  # 无限制
total-quota=100
user-quota=10

# 指纹（增强安全性）
fingerprint

# 不允许 TCP 回退到 STUN
no-stun-backward-compatibility
```

### 7.3 启动与管理

```bash
# 启动服务
sudo systemctl start coturn
sudo systemctl enable coturn

# 检查状态
sudo systemctl status coturn

# 查看日志
sudo tail -f /var/log/turnserver.log

# 测试服务
turnutils_uclient -v -t -u admin -w password123 YOUR_SERVER_IP
```

### 7.4 监控指标

```bash
# 查看当前分配
turnadmin -l -N

# 查看连接数
netstat -an | grep 3478 | wc -l

# 监控带宽
iftop -i eth0
```

---

## 8. 最佳实践

### 8.1 部署建议

```
┌────────────────────────────────────────────────────────────────┐
│                   NAT 穿透部署最佳实践                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 服务器部署                                                 │
│     ──────────                                                 │
│     - STUN: 公网服务器，无需认证                                │
│     - TURN: 高可用部署，负载均衡                               │
│     - 地理分布: 靠近用户群体                                   │
│                                                                │
│  2. 安全配置                                                   │
│     ────────                                                   │
│     - 使用 TLS（TURN over TLS）                                │
│     - 动态凭证生成                                             │
│     - 限制带宽和配额                                           │
│     - 定期轮换凭证                                             │
│                                                                │
│  3. 性能优化                                                   │
│     ──────────                                                 │
│     - 使用 UDP 而非 TCP                                        │
│     - 调整端口范围                                             │
│     - 监控资源使用                                             │
│     - 限制最大连接数                                           │
│                                                                │
│  4. 客户端配置                                                 │
│     ──────────                                                 │
│     - 配置多个 STUN 服务器                                     │
│     - 支持 Trickle ICE                                         │
│     - 实现重连机制                                             │
│     - 优雅降级处理                                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 8.2 常见问题解决

```cpp
// 问题 1：ICE 连接失败
// 解决：检查服务器配置，添加更多候选

// 问题 2：只有 host 候选
// 解决：确保 STUN 服务器可达

if (candidates.size() == 1 && candidates[0].type == "host") {
    // 检查 STUN 服务器
    if (!testStunConnectivity()) {
        logError("STUN server unreachable");
        // 尝试备用 STUN 服务器
        addFallbackStunServer();
    }
}

// 问题 3：延迟过高
// 解决：检查是否使用了 TURN，优化服务器位置

if (selectedCandidate.type == "relay") {
    logWarning("Using TURN relay, latency may be higher");
    // 考虑部署更近的 TURN 服务器
}

// 问题 4：连接不稳定
// 解决：实现重连机制

pc.oniceconnectionstatechange = () => {
    if (pc.iceConnectionState === 'failed' || 
        pc.iceConnectionState === 'disconnected') {
        // 重启 ICE
        pc.restartIce();
    }
};
```

---

## 9. 总结

### NAT 穿透技术对比

| 技术 | 作用 | 适用场景 | 限制 |
|------|------|----------|------|
| **STUN** | 获取公网地址 | 大多数 NAT 类型 | 对称 NAT 无法穿透 |
| **TURN** | 中继传输 | 所有 NAT 类型 | 延迟高、带宽受限 |
| **ICE** | 综合穿透框架 | 所有场景 | 需要服务器支持 |

### 关键要点

1. **理解 NAT 类型**：不同 NAT 需要不同的穿透策略
2. **配置 STUN/TURN**：STUN 用于发现地址，TURN 作为兜底
3. **ICE 是核心**：整合 STUN/TURN，自动选择最佳路径
4. **监控与诊断**：及时发现和解决穿透问题
5. **安全配置**：使用 TLS 和动态凭证保护 TURN 服务
