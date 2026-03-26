# WebRTC 知识点清单

> WebRTC 核心概念、原理、面试题速查手册，助力技术面试准备。

---

## 1. 核心概念词汇表

### 1.1 基础概念

| 术语 | 英文 | 解释 |
|------|------|------|
| **WebRTC** | Web Real-Time Communication | 浏览器和移动应用的实时通信框架 |
| **P2P** | Peer-to-Peer | 点对点通信，无需中心服务器 |
| **SDP** | Session Description Protocol | 会话描述协议，描述媒体会话参数 |
| **ICE** | Interactive Connectivity Establishment | 交互式连接建立，NAT 穿透框架 |
| **STUN** | Session Traversal Utilities for NAT | NAT 穿透工具协议 |
| **TURN** | Traversal Using Relays around NAT | 使用中继穿透 NAT |
| **NAT** | Network Address Translation | 网络地址转换 |

### 1.2 媒体相关

| 术语 | 英文 | 解释 |
|------|------|------|
| **RTP** | Real-time Transport Protocol | 实时传输协议 |
| **RTCP** | RTP Control Protocol | RTP 控制协议 |
| **SRTP** | Secure RTP | 安全 RTP，加密传输 |
| **Codec** | Coder-Decoder | 编解码器 |
| **Opus** | - | 开源音频编解码器 |
| **VP8/VP9** | - | Google 开发的视频编解码器 |
| **H.264** | - | 广泛使用的视频编解码器 |

### 1.3 安全相关

| 术语 | 英文 | 解释 |
|------|------|------|
| **DTLS** | Datagram Transport Layer Security | 数据包传输层安全协议 |
| **SRTP** | Secure Real-time Transport Protocol | 安全实时传输协议 |
| **Fingerprint** | - | 证书指纹，用于验证身份 |
| **ECDHE** | Elliptic Curve Diffie-Hellman Ephemeral | 椭圆曲线临时 Diffie-Hellman 密钥交换 |

### 1.4 网络相关

| 术语 | 英文 | 解释 |
|------|------|------|
| **Jitter** | - | 抖动，网络延迟的变化 |
| **RTT** | Round-Trip Time | 往返时延 |
| **Packet Loss** | - | 丢包率 |
| **Bandwidth** | - | 带宽 |
| **FEC** | Forward Error Correction | 前向纠错 |
| **NACK** | Negative Acknowledgment | 否定确认，请求重传 |

---

## 2. 技术原理速查

### 2.1 WebRTC 连接建立流程

```
WebRTC 连接建立四步法：

1. 信令交换
   ┌─────────┐              ┌─────────┐
   │ 用户 A  │              │ 用户 B  │
   └────┬────┘              └────┬────┘
        │                        │
        │  ──── Offer (SDP) ────►│
        │                        │
        │  ◄─── Answer (SDP) ────│
        │                        │
   
2. ICE 候选交换
   用户 A ◄──────────────────────► 用户 B
   交换 ICE 候选（host/srflx/relay）

3. DTLS 握手
   用户 A ◄──────────────────────► 用户 B
   协商加密密钥

4. 媒体传输
   用户 A ◄════ SRTP ═════════► 用户 B
   加密的音视频数据
```

### 2.2 SDP 结构解析

```
SDP 关键字段：

会话级别：
v=0                          # 版本
o=- 461173... 2 IN IP4 ...   # 发起者信息
s=-                          # 会话名
t=0 0                        # 时间
a=group:BUNDLE 0 1           # 多路复用

媒体级别：
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 ...  # 音频媒体行
m=video 9 UDP/TLS/RTP/SAVPF 96 97 ...    # 视频媒体行

ICE 相关：
a=ice-ufrag:GsO4             # ICE 用户名片段
a=ice-pwd:kyVq...            # ICE 密码
a=candidate:...              # ICE 候选

安全相关：
a=fingerprint:sha-256 4B:8B:... # DTLS 证书指纹
a=setup:actpass              # DTLS 角色

编解码：
a=rtpmap:111 opus/48000/2    # RTP 映射
a=fmtp:111 minptime=10       # 格式参数
a=rtcp-fb:96 nack            # RTCP 反馈
```

### 2.3 ICE 候选类型

```
ICE 候选类型对比：

┌────────────┬────────────────┬──────────┬──────────┐
│ 类型       │ 来源           │ 优先级   │ 延迟     │
├────────────┼────────────────┼──────────┼──────────┤
│ host       │ 本地网络接口   │ 最高(126)│ 最低     │
│ srflx      │ STUN 服务器    │ 高(100)  │ 低       │
│ prflx      │ NAT 映射发现   │ 中(110)  │ 中       │
│ relay      │ TURN 服务器    │ 最低(0)  │ 最高     │
└────────────┴────────────────┴──────────┴──────────┘

优先级计算：
priority = 2^24 × type_preference + 
           2^8 × local_preference + 
           2^0 × (256 - component_id)
```

### 2.4 NAT 类型及穿透方法

```
NAT 类型：

1. Full Cone NAT（完全锥形）
   - 一旦建立映射，任何外部主机都可发送
   - 穿透：最简单，STUN 即可

2. Restricted Cone NAT（受限锥形）
   - 只有曾发送过数据的外部 IP 可发送
   - 穿透：STUN + ICE

3. Port Restricted Cone NAT（端口受限锥形）
   - 只有曾发送过数据的外部 IP:Port 可发送
   - 穿透：STUN + ICE

4. Symmetric NAT（对称型）
   - 对不同目标使用不同映射
   - 穿透：必须使用 TURN 中继

穿透成功率统计：
- Full Cone: 100%
- Restricted Cone: ~95%
- Port Restricted: ~90%
- Symmetric: 需要 TURN
```

---

## 3. 常见面试题

### 3.1 基础题

#### Q1: 什么是 WebRTC？它的主要应用场景有哪些？

```
答：
WebRTC（Web Real-Time Communication）是一个支持浏览器和移动应用
进行实时音视频通信的开源项目。

核心特点：
1. 无需插件，浏览器原生支持
2. P2P 通信，无需中心服务器转发媒体
3. 开源免费，跨平台支持

应用场景：
- 视频会议（Zoom、腾讯会议）
- 在线教育（直播课堂）
- 远程协作（屏幕共享）
- 社交娱乐（视频聊天）
- IoT 设备（监控摄像头）
```

#### Q2: WebRTC 的核心组件有哪些？

```
答：

1. getUserMedia API
   - 获取摄像头、麦克风权限
   - 采集音视频流

2. RTCPeerConnection API
   - 建立 P2P 连接
   - 处理 ICE、DTLS、SRTP
   - 管理音视频传输

3. RTCDataChannel API
   - 建立 P2P 数据通道
   - 支持可靠/不可靠传输

4. 信令服务器（非 WebRTC 内置）
   - 交换 SDP
   - 交换 ICE 候选
   - 自定义协议（WebSocket/SIP）
```

#### Q3: WebRTC 如何实现 NAT 穿透？

```
答：

NAT 穿透三层方案：

1. STUN 服务器
   - 获取公网 IP 和端口
   - 检测 NAT 类型
   - 成功率：约 80%

2. ICE 框架
   - 收集所有候选地址
   - 按优先级排序
   - 进行连通性检测
   - 选择最佳路径

3. TURN 服务器
   - 作为最后手段
   - 通过服务器中继
   - 成功率：100%
   - 代价：延迟高、带宽成本

流程：
收集候选 → 排序优先级 → 连通性检测 → 选择路径
```

### 3.2 进阶题

#### Q4: 解释 SDP Offer/Answer 交换过程

```
答：

角色定义：
- Offerer（呼叫方）：创建 Offer
- Answerer（被叫方）：创建 Answer

交换流程：

1. Offerer 创建 Offer
   const offer = await pc.createOffer();
   await pc.setLocalDescription(offer);
   
2. 通过信令服务器发送 Offer 给 Answerer

3. Answerer 接收并设置远程描述
   await pc.setRemoteDescription(offer);
   
4. Answerer 创建 Answer
   const answer = await pc.createAnswer();
   await pc.setLocalDescription(answer);
   
5. 通过信令服务器发送 Answer 给 Offerer

6. Offerer 设置远程描述
   await pc.setRemoteDescription(answer);

关键点：
- 必须先 setLocalDescription 再发送
- Trickle ICE 可以先发 SDP，后发候选
```

#### Q5: WebRTC 如何保证传输安全？

```
答：

三层安全机制：

1. 信令安全（应用层）
   - 使用 WSS（WebSocket Secure）
   - TLS 1.2+ 加密
   - 防止信令劫持

2. 传输安全（DTLS）
   - DTLS 握手协商密钥
   - 双向认证（证书指纹）
   - 使用 ECDHE 实现前向保密

3. 媒体安全（SRTP）
   - AES-128-CTR 加密
   - HMAC-SHA1 认证
   - 防止窃听和篡改

安全流程：
DTLS 握手 → 导出密钥 → SRTP 加密传输

关键：
- 每个 PeerConnection 独立密钥
- 证书指纹通过 SDP 传递验证
```

#### Q6: 解释 GCC 拥塞控制算法原理

```
答：

GCC（Google Congestion Control）核心思想：
基于延迟梯度和丢包率调整发送速率

算法流程：

1. 延迟梯度计算
   delay_gradient = (arrival_time - send_time) - prev_delay
   
2. 趋势检测
   使用线性回归或卡尔曼滤波
   判断网络是否拥塞
   
3. 速率调整
   if 拥塞:
       target_bitrate *= 0.85  # 乘性减
   else:
       target_bitrate *= 1.05  # 加性增

4. 丢包响应
   if loss_rate > 10%:
       target_bitrate *= 0.7
   elif loss_rate < 2%:
       target_bitrate *= 1.02

特点：
- 基于延迟，非基于丢包
- AIMD（加性增乘性减）
- 公平性好，收敛快
```

### 3.3 高级题

#### Q7: 设计一个支持 100 人的视频会议系统架构

```
答：

架构选择：SFU（Selective Forwarding Unit）

           ┌─────────────┐
           │    SFU      │
           │  (媒体路由) │
           └──────┬──────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───┴───┐    ┌───┴───┐    ┌───┴───┐
│ 客户端 │    │ 客户端 │    │ 客户端 │
│  1    │    │  2    │    │  ...  │
└───────┘    └───────┘    └───────┘

SFU 架构优点：
- 客户端只需发送一路流
- SFU 只转发不混流
- 延迟低，扩展性好

关键技术：

1. Simulcast
   - 客户端发送多路不同质量
   - SFU 根据接收者带宽选择
   
2. 视频布局
   - 服务端合成布局（MCU 模式）
   - 或客户端接收多路自己布局
   
3. 带宽管理
   - 每路视频动态分配
   - 发言者优先

4. 服务器部署
   - 多区域部署降低延迟
   - 负载均衡
```

#### Q8: 如何优化弱网环境下的视频通话质量？

```
答：

多层次优化策略：

1. 网络层优化
   - FEC（前向纠错）：添加冗余数据
   - NACK：请求重传丢失包
   - ARQ：自动重传请求

2. 编码层优化
   - 自适应码率（GCC）
   - 自适应分辨率
   - 自适应帧率

3. 应用层优化
   - 丢包隐藏（PLC）
   - 视频帧内插
   - 音频优先策略

4. 用户体验优化
   - 网络质量指示器
   - 自动降级提示
   - 重连机制

实现方案：

class AdaptiveQualityController {
    void onNetworkChanged(Quality quality) {
        switch(quality) {
            case Excellent:
                setResolution(1920, 1080);
                setBitrate(1500);
                setFps(30);
                break;
            case Good:
                setResolution(1280, 720);
                setBitrate(800);
                setFps(25);
                break;
            case Poor:
                setResolution(640, 480);
                setBitrate(300);
                setFps(15);
                enableFEC(true);
                break;
        }
    }
};
```

#### Q9: WebRTC 的 Jitter Buffer 是如何工作的？

```
答：

Jitter Buffer 目的：
平滑网络抖动导致的包到达时间不规则

工作原理：

1. 接收端缓冲
   ┌───────────────────────────────────┐
   │  Jitter Buffer                    │
   │  ┌─┐┌─┐  ┌─┐  ┌─┐┌─┐┌─┐         │
   │  │1││2│  │3│  │4││5││6│  → 播放  │
   │  └─┘└─┘  └─┘  └─┘└─┘└─┘         │
   │                                   │
   │  接收：不规则  输出：均匀间隔      │
   └───────────────────────────────────┘

2. 自适应调整
   - 网络好：减小缓冲，降低延迟
   - 网络差：增大缓冲，减少卡顿

3. 关键参数
   target_delay = jitter_estimate + safety_margin

4. 音视频处理
   - 音频：加速/减速播放
   - 视频：丢帧/重复帧

实现要点：
- 使用环形缓冲区
- 根据抖动动态调整大小
- 平滑播放避免卡顿
```

---

## 4. 代码实战题

### 4.1 实现一个简单的 WebRTC 连接

```javascript
// 本地端
const localPc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
});

// 添加本地流
const stream = await navigator.mediaDevices.getUserMedia({
    video: true, audio: true
});
stream.getTracks().forEach(track => localPc.addTrack(track, stream));

// 收集 ICE 候选
localPc.onicecandidate = (e) => {
    if (e.candidate) {
        // 发送给远程端
        signaling.send({ type: 'ice', candidate: e.candidate });
    }
};

// 创建并发送 Offer
const offer = await localPc.createOffer();
await localPc.setLocalDescription(offer);
signaling.send({ type: 'offer', sdp: offer });

// 远程端接收
const remotePc = new RTCPeerConnection({...});

// 接收远程流
remotePc.ontrack = (e) => {
    remoteVideo.srcObject = e.streams[0];
};

// 设置远程描述并创建 Answer
await remotePc.setRemoteDescription(offer);
const answer = await remotePc.createAnswer();
await remotePc.setLocalDescription(answer);
signaling.send({ type: 'answer', sdp: answer });

// 本地端接收 Answer
await localPc.setRemoteDescription(answer);
```

### 4.2 实现网络质量检测

```javascript
class NetworkQualityMonitor {
    constructor(pc) {
        this.pc = pc;
        this.prevStats = {};
    }
    
    async collect() {
        const stats = await this.pc.getStats();
        const quality = {};
        
        stats.forEach(report => {
            // 计算 RTT
            if (report.type === 'candidate-pair' && report.state === 'succeeded') {
                quality.rtt = report.currentRoundTripTime * 1000;
            }
            
            // 计算丢包率
            if (report.type === 'inbound-rtp') {
                const total = report.packetsReceived + report.packetsLost;
                quality.packetLoss = report.packetsLost / total;
            }
            
            // 计算抖动
            if (report.type === 'inbound-rtp') {
                quality.jitter = report.jitter * 1000;
            }
        });
        
        return this.evaluate(quality);
    }
    
    evaluate(quality) {
        let score = 100;
        
        if (quality.rtt > 100) score -= 10;
        if (quality.rtt > 200) score -= 20;
        if (quality.rtt > 300) score -= 30;
        
        if (quality.packetLoss > 0.02) score -= 10;
        if (quality.packetLoss > 0.05) score -= 20;
        if (quality.packetLoss > 0.10) score -= 30;
        
        if (quality.jitter > 30) score -= 10;
        if (quality.jitter > 50) score -= 20;
        
        if (score >= 80) return 'Excellent';
        if (score >= 60) return 'Good';
        if (score >= 40) return 'Fair';
        return 'Poor';
    }
}
```

---

## 5. 性能指标参考

### 5.1 关键指标阈值

| 指标 | 优秀 | 良好 | 一般 | 差 |
|------|------|------|------|-----|
| **RTT** | < 50ms | < 100ms | < 200ms | > 300ms |
| **丢包率** | < 1% | < 3% | < 5% | > 10% |
| **抖动** | < 10ms | < 30ms | < 50ms | > 100ms |
| **帧率** | 30 fps | 25 fps | 15 fps | < 10 fps |
| **端到端延迟** | < 100ms | < 200ms | < 300ms | > 500ms |

### 5.2 MOS 评分标准

```
MOS (Mean Opinion Score) 音频质量评分：

5: 优秀 - 完美无瑕
4: 良好 - 偶有瑕疵
3: 一般 - 略有干扰
2: 较差 - 明显干扰
1: 很差 - 无法使用

MOS 估算公式：
MOS = 4.5 - 0.035 × R - 0.000007 × R × R × 
      (R - 60) × (R - 60)

其中 R 为 R-factor：
R = 93.2 - Ie - (95 - Ie) × Ppl / (Ppl + Bpl)

- Ie: 设备损伤因子
- Ppl: 丢包率
- Bpl: 丢包鲁棒性因子
```

---

## 6. 调试工具

### 6.1 Chrome 调试工具

```
chrome://webrtc-internals

功能：
1. PeerConnection 列表
2. SDP 历史记录
3. ICE 候选列表
4. 统计数据图表
5. 事件日志

使用技巧：
- 查看 ICE 连接状态
- 分析编解码器协商
- 监控网络质量
- 调试 SDP 问题
```

### 6.2 命令行工具

```bash
# 测试 STUN 服务器
stun stun.l.google.com 19302

# 测试 TURN 服务器
turnutils_uclient -v -t -u user -w pass turn.server.com

# 抓包分析
tcpdump -i any udp port 3478 -w stun.pcap

# Wireshark 过滤
stun || dtls || rtp
```

---

## 7. 学习资源

### 7.1 官方资源

- [WebRTC 官网](https://webrtc.org/)
- [W3C WebRTC API](https://www.w3.org/TR/webrtc/)
- [IETF WebRTC 标准](https://datatracker.ietf.org/wg/rtcweb/documents/)

### 7.2 开源项目

- [libwebrtc](https://webrtc.googlesource.com/src/) - Google 官方实现
- [Pion](https://github.com/pion/webrtc) - Go 语言实现
- [aiortc](https://github.com/aiortc/aiortc) - Python 实现
- [Jitsi](https://github.com/jitsi) - 开源会议系统

### 7.3 推荐书籍

- 《WebRTC 权威指南》
- 《Real-Time Communication with WebRTC》
- 《Learning WebRTC》

---

## 8. 面试准备清单

```markdown
## WebRTC 面试准备清单

### 基础知识
- [ ] WebRTC 概念和应用场景
- [ ] 核心组件和 API
- [ ] P2P 通信原理

### 协议理解
- [ ] SDP 格式和作用
- [ ] ICE 框架原理
- [ ] STUN/TURN 协议
- [ ] RTP/RTCP 协议
- [ ] DTLS/SRTP 安全

### 网络知识
- [ ] NAT 类型分类
- [ ] NAT 穿透技术
- [ ] 网络质量指标
- [ ] 拥塞控制算法

### 实践能力
- [ ] WebRTC 连接建立代码
- [ ] 网络质量监控实现
- [ ] 弱网优化策略
- [ ] 故障排查经验

### 架构设计
- [ ] Mesh vs SFU vs MCU
- [ ] 大规模会议系统设计
- [ ] 高可用架构设计

### 项目经验
- [ ] 项目技术亮点
- [ ] 遇到的挑战和解决方案
- [ ] 性能优化实践
```

---

## 9. 快速记忆卡片

### 卡片 1: 连接建立流程

```
WebRTC 连接建立 4 步：
1. 信令交换（SDP Offer/Answer）
2. ICE 候选交换
3. DTLS 握手
4. 媒体传输

记住：信 → 候 → D → 媒
```

### 卡片 2: ICE 候选类型

```
ICE 候选优先级：
Host > Prflx > Srflx > Relay

Host: 本地地址
Srflx: STUN 获取的公网地址
Prflx: NAT 映射地址
Relay: TURN 中继地址

记住：本 → 映 → 公 → 中
```

### 卡片 3: NAT 类型

```
NAT 类型穿透难度：
Full Cone < Restricted < Port Restricted < Symmetric

穿透方式：
- Full Cone: STUN 直接
- Restricted: STUN + ICE
- Symmetric: 需要 TURN

记住：锥 → 限 → 端限 → 对称（最难）
```

### 卡片 4: 安全机制

```
三层安全：
1. 信令：WSS (TLS)
2. 传输：DTLS (密钥协商)
3. 媒体：SRTP (加密传输)

记住：W → D → S
```

### 卡片 5: 质量指标

```
质量红线：
- RTT < 200ms
- 丢包 < 5%
- 抖动 < 30ms
- 帧率 > 15fps

记住：200-5-30-15
```
