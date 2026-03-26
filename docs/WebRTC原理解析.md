# WebRTC 原理解析

> 深入理解 WebRTC 核心技术原理，掌握实时音视频通信的底层机制。

---

## 1. WebRTC 架构总览

### 1.1 WebRTC 协议栈

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              JavaScript API / Native API                 │   │
│  │  getUserMedia | RTCPeerConnection | RTCDataChannel       │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                        会话层                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐   │
│  │    SDP     │  │   ICE      │  │      STUN / TURN       │   │
│  │  会话描述  │  │  连接建立  │  │      NAT 穿透          │   │
│  └────────────┘  └────────────┘  └────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                        传输层                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐   │
│  │   SRTP     │  │   DTLS     │  │      SCTP              │   │
│  │ 安全媒体   │  │  加密通道  │  │    数据通道            │   │
│  └────────────┘  └────────────┘  └────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    UDP / TCP                             │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                        媒体层                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐   │
│  │  音频引擎  │  │  视频引擎  │  │      网络传输          │   │
│  │  Opus/ACC  │  │ VP8/VP9/H264│  │   RTP/RTCP            │   │
│  └────────────┘  └────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件

| 组件 | 功能 | 关键技术 |
|------|------|----------|
| **PeerConnection** | 建立 P2P 连接 | ICE、DTLS、SRTP |
| **MediaStream** | 媒体流管理 | 音视频采集、处理 |
| **DataChannel** | 数据通道 | SCTP、可靠/不可靠传输 |
| **getUserMedia** | 媒体采集 | 设备访问、权限管理 |

---

## 2. SDP 协议详解

### 2.1 SDP 是什么？

**SDP（Session Description Protocol）** 是一种会话描述格式，用于描述多媒体会话的参数。

```
SDP 的作用：
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  发送方                      接收方                         │
│  ┌─────────┐                ┌─────────┐                   │
│  │ 创建    │   SDP Offer    │ 接收    │                   │
│  │ Offer   │ ──────────────►│ Offer   │                   │
│  │         │                │         │                   │
│  │         │   SDP Answer   │ 创建    │                   │
│  │ 接收    │ ◄──────────────│ Answer  │                   │
│  │ Answer  │                │         │                   │
│  └─────────┘                └─────────┘                   │
│                                                            │
│  Offer/Answer 交换后，双方知道：                            │
│  - 对方支持哪些编解码器                                     │
│  - 媒体传输地址和端口                                       │
│  - 加密参数                                                 │
│  - 带宽限制                                                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 2.2 SDP 结构解析

一个完整的 SDP 示例：

```sdp
v=0
o=- 4611731400430051336 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=msid-semantic: WMS

m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:GsO4
a=ice-pwd:kyVqr8hY0cXvU8b6oUj3LhVz
a=ice-options:trickle
a=fingerprint:sha-256 4B:8B:3E:9E:7A:1B:5C:3D:2E:9F:8A:6B:4C:7D:1E:2F:3A:5B:6C:8D:9E:1F:2A:3B:4C:5D:6E:7F:8A:9B:1C:2D
a=setup:actpass
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=sendrecv
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
a=rtpmap:9 G722/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:106 CN/32000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:110 telephone-event/48000
a=rtpmap:112 telephone-event/32000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
a=ssrc:3839626815 cname:RnKjNqS3dKJhXQvM
a=ssrc:3839626815 msid:stream track
a=ssrc:3839626815 mslabel:stream
a=ssrc:3839626815 label:track

m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 122 127 121 125 107 108 109 124 120
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:GsO4
a=ice-pwd:kyVqr8hY0cXvU8b6oUj3LhVz
a=ice-options:trickle
a=fingerprint:sha-256 4B:8B:3E:9E:7A:1B:5C:3D:2E:9F:8A:6B:4C:7D:1E:2F:3A:5B:6C:8D:9E:1F:2A:3B:4C:5D:6E:7F:8A:9B:1C:2D
a=setup:actpass
a=mid:1
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=sendrecv
a=rtcp-mux
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 transport-cc
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=rtpmap:98 VP9/90000
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 transport-cc
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:100 H264/90000
a=rtcp-fb:100 goog-remb
a=rtcp-fb:100 transport-cc
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=fmtp:100 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=ssrc-group:FID 3674880519 3674880520
a=ssrc:3674880519 cname:RnKjNqS3dKJhXQvM
a=ssrc:3674880519 msid:stream track
a=ssrc:3674880519 mslabel:stream
a=ssrc:3674880519 label:track
a=ssrc:3674880520 cname:RnKjNqS3dKJhXQvM
a=ssrc:3674880520 msid:stream track
a=ssrc:3674880520 mslabel:stream
a=ssrc:3674880520 label:track
```

### 2.3 SDP 字段详解

#### 会话级别字段

| 字段 | 格式 | 说明 |
|------|------|------|
| `v=` | `v=0` | SDP 版本，固定为 0 |
| `o=` | `o=<username> <sess-id> <sess-version> <nettype> <addrtype> <unicast-address>` | 会话发起者信息 |
| `s=` | `s=<session name>` | 会话名称 |
| `t=` | `t=<start-time> <stop-time>` | 会话时间，0 0 表示持久会话 |
| `a=group:BUNDLE` | `a=group:BUNDLE <mids>` | 多路复用，多个媒体流共用一个连接 |

#### 媒体级别字段

| 字段 | 格式 | 说明 |
|------|------|------|
| `m=` | `m=<media> <port> <proto> <fmt> ...` | 媒体描述行 |
| `c=` | `c=IN IP4 <connection-address>` | 连接信息 |
| `a=ice-ufrag` | `a=ice-ufrag:<ufrag>` | ICE 用户名片段 |
| `a=ice-pwd` | `a=ice-pwd:<password>` | ICE 密码 |
| `a=fingerprint` | `a=fingerprint:<hash-func> <fingerprint>` | DTLS 证书指纹 |
| `a=setup` | `a=setup:<role>` | DTLS 角色：active/passive/actpass |
| `a=mid` | `a=mid:<mid>` | 媒体标识，用于 BUNDLE |
| `a=sendrecv` | - | 媒体方向：发送并接收 |
| `a=rtpmap` | `a=rtpmap:<pt> <encoding-name>/<clock-rate>[/<encoding-params>]` | RTP 映射 |
| `a=fmtp` | `a=fmtp:<pt> <format-specific-params>` | 格式参数 |
| `a=rtcp-fb` | `a=rtcp-fb:<pt> <feedback-type>` | RTCP 反馈机制 |
| `a=ssrc` | `a=ssrc:<ssrc-id> <attribute>` | 同步源标识 |

### 2.4 Offer/Answer 交换流程

```
┌──────────────────────────────────────────────────────────────────┐
│                    Offer/Answer 交换流程                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  客户端 A (Caller)                          客户端 B (Callee)    │
│       │                                           │              │
│       │  1. createOffer()                         │              │
│       │  ───────────────┐                         │              │
│       │                  │                         │              │
│       │  2. setLocalDescription(offer)            │              │
│       │  ───────────────┐                         │              │
│       │                  │                         │              │
│       │  3. 信令服务器转发 Offer                   │              │
│       │  ─────────────────────────────────────────►              │
│       │                                           │              │
│       │                            4. setRemoteDescription(offer)│
│       │                            ─────────────────┘              │
│       │                                           │              │
│       │                            5. createAnswer()              │
│       │                            ─────────────────┘              │
│       │                                           │              │
│       │                            6. setLocalDescription(answer) │
│       │                            ─────────────────┘              │
│       │                                           │              │
│       │  7. 信令服务器转发 Answer                  │              │
│       │  ◄─────────────────────────────────────────              │
│       │                                           │              │
│       │  8. setRemoteDescription(answer)          │              │
│       │  ───────────────────────┘                 │              │
│       │                                           │              │
│       │◄══════════════ WebRTC 连接建立 ══════════►│              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. ICE 协议详解

### 3.1 ICE 是什么？

**ICE（Interactive Connectivity Establishment）** 是一种用于在 NAT 环境中建立连接的框架协议。

```
ICE 的目标：
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│   NAT 环境                       NAT 环境                      │
│  ┌─────────┐                    ┌─────────┐                   │
│  │ 客户端 A │                    │ 客户端 B │                   │
│  │192.168.1│                    │10.0.0.1 │                   │
│  └────┬────┘                    └────┬────┘                   │
│       │                              │                         │
│    ┌──┴──┐                        ┌──┴──┐                     │
│    │ NAT │                        │ NAT │                     │
│    │203.0│                        │198.51│                     │
│    └──┬──┘                        └──┬──┘                     │
│       │                              │                         │
│       └──────────── 互联网 ──────────┘                         │
│                                                                │
│   问题：A 无法直接访问 B 的内网地址                             │
│   解决：ICE 找到双方都可以访问的地址                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 ICE 候选类型

```
┌────────────────────────────────────────────────────────────────┐
│                      ICE 候选类型                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Host Candidate (主机候选)                                  │
│     ─────────────────────                                      │
│     本地网络接口的 IP 地址                                      │
│     格式: a=candidate:1 1 UDP 2122260223 192.168.1.100 54321   │
│                                       typ host                 │
│                                                                │
│     优先级: 最高                                               │
│     延迟: 最低                                                 │
│     适用: 同一局域网                                           │
│                                                                │
│  2. Server Reflexive Candidate (服务器反射候选)                │
│     ─────────────────────────────────────                      │
│     通过 STUN 服务器获取的公网 IP                              │
│     格式: a=candidate:2 1 UDP 1686052607 203.0.113.5 54322     │
│                                       typ srflx raddr ...      │
│                                                                │
│     优先级: 高                                                 │
│     延迟: 中等                                                 │
│     适用: 大多数 NAT 环境                                      │
│                                                                │
│  3. Peer Reflexive Candidate (对等反射候选)                    │
│     ───────────────────────────────────                        │
│     在 ICE 检测过程中动态发现的地址                            │
│     格式: a=candidate:3 1 UDP 1686052606 203.0.113.10 54323    │
│                                       typ prflx raddr ...      │
│                                                                │
│     优先级: 中                                                 │
│     延迟: 中等                                                 │
│     适用: 特殊 NAT 配置                                        │
│                                                                │
│  4. Relay Candidate (中继候选)                                 │
│     ─────────────────────                                      │
│     TURN 服务器的中继地址                                      │
│     格式: a=candidate:4 1 UDP 41885439 198.51.100.1 3478       │
│                                       typ relay raddr ...      │
│                                                                │
│     优先级: 最低                                               │
│     延迟: 最高                                                 │
│     适用: 对称 NAT、严格防火墙                                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 3.3 候选优先级计算

```
优先级 = (2^24) * (type preference) + 
         (2^8) * (local preference) + 
         (2^0) * (component ID)

type preference:
  - host:     126
  - srflx:    100
  - prflx:    110
  - relay:    0

local preference:
  - IPv6:     50
  - IPv4:     10
  - VPN/隧道: 更低

component ID:
  - RTP:      1
  - RTCP:     2

示例：
Host 候选优先级 = 2^24 * 126 + 2^8 * 50 + 1 = 2130706177
Relay 候选优先级 = 2^24 * 0 + 2^8 * 10 + 1 = 2561
```

### 3.4 ICE 连通性检测

```
┌────────────────────────────────────────────────────────────────┐
│                    ICE 连通性检测流程                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  状态转换：                                                    │
│                                                                │
│  new ──► checking ──► connected ──► completed                  │
│              │            │                                    │
│              ▼            ▼                                    │
│           failed      disconnected                             │
│                           │                                    │
│                           ▼                                    │
│                        closed                                  │
│                                                                │
│  检测流程：                                                    │
│  ┌─────────┐                    ┌─────────┐                   │
│  │ 客户端 A │                    │ 客户端 B │                   │
│  └────┬────┘                    └────┬────┘                   │
│       │                              │                         │
│       │ STUN Binding Request         │                         │
│       │──────────────────────────────►                         │
│       │  (from candidate pair)       │                         │
│       │                              │                         │
│       │ STUN Binding Response        │                         │
│       │◄──────────────────────────────                         │
│       │  (success response)          │                         │
│       │                              │                         │
│       │ 检测成功，标记为 valid pair   │                         │
│       │                              │                         │
│       ▼                              ▼                         │
│  ICE 状态 → connected/completed                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. RTP/RTCP 协议详解

### 4.1 RTP 协议

**RTP（Real-time Transport Protocol）** 是实时传输协议，用于传输音视频数据。

#### RTP 包头结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            contributing source (CSRC) identifiers             |
|                             ....                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            payload                            |
|                             ....                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### 字段说明

| 字段 | 位数 | 说明 |
|------|------|------|
| V (Version) | 2 | 版本号，固定为 2 |
| P (Padding) | 1 | 填充标志 |
| X (Extension) | 1 | 扩展头标志 |
| CC (CSRC Count) | 4 | CSRC 计数 |
| M (Marker) | 1 | 标记位，帧边界 |
| PT (Payload Type) | 7 | 负载类型 |
| Sequence Number | 16 | 序列号，用于检测丢包 |
| Timestamp | 32 | 时间戳，用于播放同步 |
| SSRC | 32 | 同步源标识 |
| CSRC | 32×CC | 贡献源标识 |

#### 常见负载类型

| PT | 编码名称 | 媒体类型 | 时钟频率 |
|----|----------|----------|----------|
| 0 | PCMU | 音频 | 8000 |
| 8 | PCMA | 音频 | 8000 |
| 9 | G722 | 音频 | 8000 |
| 96-127 | 动态分配 | - | - |
| 常用 96 | VP8 | 视频 | 90000 |
| 常用 98 | VP9 | 视频 | 90000 |
| 常用 100 | H264 | 视频 | 90000 |
| 常用 111 | Opus | 音频 | 48000 |

### 4.2 RTCP 协议

**RTCP（RTP Control Protocol）** 是 RTP 的控制协议，用于传输统计信息和控制信息。

#### RTCP 包类型

```
┌────────────────────────────────────────────────────────────────┐
│                      RTCP 包类型                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. SR (Sender Report) - 发送端报告                            │
│     ─────────────────────────────────                          │
│     发送端的统计信息                                            │
│     包含：SSRC、NTP 时间、RTP 时间戳、发送包数、发送字节数       │
│                                                                │
│  2. RR (Receiver Report) - 接收端报告                          │
│     ──────────────────────────────────                         │
│     接收端的统计信息                                            │
│     包含：SSRC、丢包率、累计丢包数、最高序列号、抖动、LSR、DLSR │
│                                                                │
│  3. SDES (Source Description) - 源描述                         │
│     ──────────────────────────────                             │
│     描述源的信息                                                │
│     包含：CNAME、NAME、EMAIL、PHONE、LOC、TOOL 等               │
│                                                                │
│  4. BYE - 离开会话                                             │
│     ───────────                                                │
│     通知离开会话                                                │
│     包含：SSRC 列表、离开原因                                   │
│                                                                │
│  5. APP (Application) - 应用自定义                             │
│     ──────────────────────────                                 │
│     应用定义的包类型                                            │
│                                                                │
│  6. 特殊反馈包 (Feedback Messages)                             │
│     ──────────────────────────────                             │
│     - NACK: 负面确认，请求重传                                 │
│     - PLI (Picture Loss Indication): 请求关键帧                │
│     - FIR (Full Intra Request): 请求全帧                       │
│     - REMB (Receiver Estimated Maximum Bitrate): 带宽估算      │
│     - Transport-CC: 传输层拥塞控制                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### RTCP 报告间隔

```
RTCP 报告间隔计算：
interval = random [0.5, 1.5] × avg_rtcp_size / rtcp_bw

其中：
- avg_rtcp_size: 平均 RTCP 包大小
- rtcp_bw: RTCP 带宽（通常为会话带宽的 5%）
- 随机因子避免同步

典型间隔：
- 小型会议（2-5 人）：1-5 秒
- 大型会议：更长间隔
```

### 4.3 关键统计指标

```javascript
// 通过 RTCP 计算关键指标

// 1. 丢包率
packetLossRate = packetsLost / (packetsReceived + packetsLost);

// 2. 往返时延 (RTT)
rtt = (arrivalTime - lastSR) - delaySinceLastSR;
// arrivalTime: 当前时间
// lastSR: 最后一个 SR 的 NTP 时间
// delaySinceLastSR: DLSR，接收端处理延迟

// 3. 抖动 (Jitter)
J = J + (|D(i-1,i)| - J) / 16
// D(i-1,i) = (Ri - Ri-1) - (Si - Si-1)
// R: 接收时间，S: 发送时间

// 4. 带宽估算
availableBandwidth = REMB value from RTCP
```

---

## 5. SRTP 加密机制

### 5.1 SRTP 概述

**SRTP（Secure RTP）** 是 RTP 的安全扩展，提供加密、认证和重放保护。

```
┌────────────────────────────────────────────────────────────────┐
│                    SRTP 安全机制                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  RTP 包                                                        │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Header │ Payload │                                        │ │
│  └────────┴─────────┴────────────────────────────────────────┘ │
│       │        │                                               │
│       │        ▼                                               │
│       │   ┌──────────────────┐                                │
│       │   │ 加密 (AES-CTR)   │                                │
│       │   └────────┬─────────┘                                │
│       │            │                                          │
│       │            ▼                                          │
│       │   ┌──────────────────┐                                │
│       │   │ 加密后的 Payload │                                │
│       │   └──────────────────┘                                │
│       │                                                       │
│       └───────────────► 不加密（用于路由）                     │
│                                                                │
│  添加认证标签：                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Header │ Encrypted Payload │ Auth Tag                    │ │
│  └────────┴───────────────────┴──────────┘                   │
│                                           │                   │
│                                           ▼                   │
│                              ┌──────────────────┐             │
│                              │ HMAC-SHA1 认证   │             │
│                              └──────────────────┘             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 5.2 DTLS 握手

SRTP 的密钥通过 **DTLS（Datagram TLS）** 握手协商。

```
┌────────────────────────────────────────────────────────────────┐
│                    DTLS 握手流程                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  客户端 A                                    客户端 B           │
│      │                                          │              │
│      │ ClientHello                              │              │
│      │ (支持的加密套件、随机数)                  │              │
│      │─────────────────────────────────────────►│              │
│      │                                          │              │
│      │ ServerHello                              │              │
│      │ (选定的加密套件、随机数)                  │              │
│      │◄─────────────────────────────────────────│              │
│      │                                          │              │
│      │ Certificate                              │              │
│      │ (服务器证书)                              │              │
│      │◄─────────────────────────────────────────│              │
│      │                                          │              │
│      │ Certificate                              │              │
│      │ (客户端证书)                              │              │
│      │─────────────────────────────────────────►│              │
│      │                                          │              │
│      │ ClientKeyExchange                        │              │
│      │ (ECDHE 公钥)                              │              │
│      │─────────────────────────────────────────►│              │
│      │                                          │              │
│      │ CertificateVerify                        │              │
│      │ (签名验证)                                │              │
│      │─────────────────────────────────────────►│              │
│      │                                          │              │
│      │ ChangeCipherSpec                         │              │
│      │ Finished                                 │              │
│      │─────────────────────────────────────────►│              │
│      │                                          │              │
│      │ ChangeCipherSpec                         │              │
│      │ Finished                                 │              │
│      │◄─────────────────────────────────────────│              │
│      │                                          │              │
│      │◄═══════ 加密通道建立，导出 SRTP 密钥 ═════│              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 5.3 密钥导出

```
DTLS 握手完成后，导出 SRTP 密钥：

key_block = PRF(master_secret, "EXTRACTOR", client_random + server_random, length)

从 key_block 中提取：
- client_write_SRTP_master_key
- server_write_SRTP_master_key  
- client_write_SRTP_master_salt
- server_write_SRTP_master_salt

每个方向使用独立的密钥和盐值
```

---

## 6. Jitter Buffer 工作原理

### 6.1 Jitter Buffer 概述

**Jitter Buffer（抖动缓冲）** 用于平滑网络抖动导致的包到达时间不规则。

```
┌────────────────────────────────────────────────────────────────┐
│                    抖动问题示意                                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  发送时间（均匀间隔）：                                         │
│  ──●──────●──────●──────●──────●──────●──────●──              │
│    0ms    20ms   40ms   60ms   80ms   100ms  120ms             │
│                                                                │
│  接收时间（不规则间隔）：                                       │
│  ──●────●──────────●───●────────●───────●───●──              │
│    0ms  15ms      55ms 62ms     95ms    118ms 125ms            │
│         ↑               ↑                                     │
│      早到             晚到                                      │
│                                                                │
│  没有 Jitter Buffer：                                          │
│  - 播放卡顿、断续                                              │
│  - 音视频不同步                                                │
│                                                                │
│  有 Jitter Buffer：                                            │
│  ──●──────●──────●──────●──────●──────●──────●──              │
│    延迟播放，平滑输出                                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 6.2 Jitter Buffer 策略

```
┌────────────────────────────────────────────────────────────────┐
│                  Jitter Buffer 策略                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 静态 Jitter Buffer                                         │
│     ────────────────────                                       │
│     固定延迟大小                                                │
│     - 简单，但效率低                                           │
│     - 延迟固定，可能过大或过小                                  │
│                                                                │
│  2. 自适应 Jitter Buffer                                       │
│     ──────────────────────                                     │
│     根据网络状况动态调整                                        │
│     - 网络好：减少延迟                                          │
│     - 网络差：增加延迟                                          │
│     - WebRTC 默认使用                                          │
│                                                                │
│  调整算法：                                                    │
│                                                                │
│  target_delay = max(min_delay, jitter_estimate + safety_margin)│
│                                                                │
│  其中：                                                        │
│  - jitter_estimate: 估算的网络抖动                             │
│  - safety_margin: 安全边界（如 20ms）                          │
│  - min_delay: 最小延迟（如 20ms）                              │
│                                                                │
│  当前延迟 > target_delay：加速播放                              │
│  当前延迟 < target_delay：减速播放                              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 6.3 丢包隐藏（PLC）

```
┌────────────────────────────────────────────────────────────────┐
│                   丢包隐藏机制                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  收到的包：   P1    P2    [丢失]    P4    P5                   │
│              ●─────●─────    ○    ───●─────●──                 │
│                                                                │
│  PLC（Packet Loss Concealment）处理：                          │
│                                                                │
│  1. 重复前一个包的最后部分                                      │
│     ─────────────────────                                      │
│     P1 ── P2 ── P2' ── P4 ── P5                               │
│                 ↑                                              │
│              重复部分                                           │
│                                                                │
│  2. 波形插值                                                   │
│     ────────                                                   │
│     根据前后包插值生成                                          │
│     P1 ── P2 ── [插值] ── P4 ── P5                            │
│                                                                │
│  3. 编码器内嵌 FEC（Opus）                                      │
│     ─────────────────────                                      │
│     每个包包含前一包的低质量版本                                │
│     P1 ── P2(+P1') ── P3(+P2') ── P4(+P3') ── P5              │
│                如果 P2 丢失，解码 P1'                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 7. MiniMeeting 中的 WebRTC 应用

### 7.1 编解码器选择

```cpp
// 音频编解码器优先级
const char* kAudioCodecs[] = {
    "opus",     // 优先：高音质、低延迟、抗丢包
    "ISAC",     // 备选：宽带音频
    "G722",     // 备选：G.722 宽带
    "PCMU",     // 兼容：G.711 μ-law
    "PCMA",     // 兼容：G.711 A-law
};

// 视频编解码器优先级
const char* kVideoCodecs[] = {
    "VP9",      // 优先：高效压缩、免费
    "VP8",      // 主流：广泛支持
    "H264",     // 兼容：硬件编码支持
};

// 配置编解码器
webrtc::PeerConnectionInterface::RTCConfiguration config;
config.audio_codecs = kAudioCodecs;
config.video_codecs = kVideoCodecs;
```

### 7.2 带宽管理

```cpp
// 设置带宽限制
void setBandwidthConstraints(PeerConnection* pc) {
    // 音频带宽
    pc->SetAudioMaxBitrate(64);  // 64kbps
    
    // 视频带宽
    pc->SetVideoMinBitrate(100);   // 最小 100kbps
    pc->SetVideoMaxBitrate(1500);  // 最大 1500kbps
    
    // 设置带宽估计策略
    pc->SetBitratePolicy(kBitratePolicyThroughput);
}
```

### 7.3 网络优先级

```cpp
// ICE 网络优先级配置
webrtc::PeerConnectionInterface::RTCConfiguration config;

// 网络类型优先级
config.network_preference = webrtc::PeerConnectionInterface::kNetworkPreferenceLowCost;

// 禁用某些网络类型
config.disable_ipv6 = false;
config.disable_ipv6_wifi = false;

// 候选收集策略
config.ice_candidate_pool_size = 10;
```

---

## 8. WebRTC 调试技巧

### 8.1 chrome://webrtc-internals

```
访问 chrome://webrtc-internals 查看：

1. PeerConnection 列表
   - 所有活动的连接
   - 创建时间
   - 状态

2. SDP 历史记录
   - Offer/Answer 内容
   - 修改历史

3. ICE 候选
   - 本地/远程候选
   - 候选对状态

4. 统计数据图表
   - 码率曲线
   - 丢包率
   - 帧率
   - RTT

5. 事件日志
   - 连接状态变化
   - 媒体事件
```

### 8.2 获取 WebRTC 统计

```javascript
// 完整统计采集
async function getWebRTCStats(pc) {
    const stats = await pc.getStats();
    const result = {};
    
    stats.forEach(report => {
        result[report.id] = {
            type: report.type,
            timestamp: report.timestamp,
            // 所有属性
            ...report
        };
    });
    
    return result;
}

// 关键指标提取
async function getMetrics(pc) {
    const stats = await pc.getStats();
    const metrics = {};
    
    stats.forEach(report => {
        if (report.type === 'outbound-rtp' && report.kind === 'video') {
            metrics.video = {
                bytesSent: report.bytesSent,
                packetsSent: report.packetsSent,
                framesEncoded: report.framesEncoded,
                frameWidth: report.frameWidth,
                frameHeight: report.frameHeight,
                framesPerSecond: report.framesPerSecond
            };
        }
        
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
            metrics.videoRecv = {
                bytesReceived: report.bytesReceived,
                packetsLost: report.packetsLost,
                framesDecoded: report.framesDecoded,
                jitter: report.jitter
            };
        }
        
        if (report.type === 'candidate-pair' && report.state === 'succeeded') {
            metrics.network = {
                rtt: report.currentRoundTripTime * 1000,
                availableBitrate: report.availableOutgoingBitrate
            };
        }
    });
    
    return metrics;
}
```

---

## 9. 总结

### WebRTC 核心技术栈

```
┌─────────────────────────────────────────────────────────────────┐
│                   WebRTC 技术栈总结                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  媒体协商                                                       │
│  ────────                                                       │
│  SDP ←→ Offer/Answer 交换                                       │
│  定义：编解码器、媒体格式、传输参数                              │
│                                                                 │
│  连接建立                                                       │
│  ────────                                                       │
│  ICE ←→ STUN/TURN ←→ NAT 穿透                                   │
│  过程：收集候选 → 排序 → 连通性检测 → 建立连接                   │
│                                                                 │
│  安全传输                                                       │
│  ────────                                                       │
│  DTLS ←→ 握手协商密钥 ←→ SRTP 加密                              │
│  保护：加密、认证、重放保护                                      │
│                                                                 │
│  媒体传输                                                       │
│  ────────                                                       │
│  RTP ←→ 数据传输 ←→ RTCP 控制                                   │
│  机制：时间戳、序列号、SSRC、统计反馈                            │
│                                                                 │
│  质量保障                                                       │
│  ────────                                                       │
│  Jitter Buffer ←→ 抖动平滑                                      │
│  GCC ←→ 拥塞控制                                                │
│  FEC/NACK ←→ 丢包恢复                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键概念速查

| 概念 | 全称 | 作用 |
|------|------|------|
| **SDP** | Session Description Protocol | 描述会话参数 |
| **ICE** | Interactive Connectivity Establishment | NAT 穿透框架 |
| **STUN** | Session Traversal Utilities for NAT | 获取公网地址 |
| **TURN** | Traversal Using Relays around NAT | 中继传输 |
| **RTP** | Real-time Transport Protocol | 实时传输 |
| **RTCP** | RTP Control Protocol | 传输控制 |
| **SRTP** | Secure RTP | 加密传输 |
| **DTLS** | Datagram TLS | 密钥协商 |
| **GCC** | Google Congestion Control | 拥塞控制 |
