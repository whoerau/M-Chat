# M-Chat 原生音视频通话 App 实施计划

## 1. 目标与范围

在 M-Chat 仓库建立 Android、iOS、Go 服务端和部署配置：

- Android：Kotlin、Jetpack Compose，最低 Android 10。
- iOS：Swift 6、SwiftUI，最低 iOS 17。
- 业务服务：Go REST API + WebSocket，部署在 `<BUSINESS_HOST>`。
- 通话中继：自托管 LiveKit，部署在 `<RELAY_HOST>`。
- v1 仅支持 1:1，同时最多 1–2 路通话。
- 支持注册登录、设备审批、好友、在线状态、纯文字聊天、送达/已读回执、通话记录、后台及进程被杀后的系统来电。
- 不包含群聊、附件、录制、屏幕共享、邮件验证、账户恢复、应用商店发布。
- v1 暂不实现文字或媒体端到端加密；使用 HTTPS/WSS 与 DTLS-SRTP。

仓库规划：

```text
M-Chat/
  android/
  ios/
  server/
  contracts/
  deploy/
  docs/
```

## 2. 总体架构

客户端通过部署时注入的 `<API_DOMAIN>` 完成认证、好友、消息和通话控制。被叫接受后，服务端签发短期 LiveKit 房间令牌；媒体通过 `<RTC_DOMAIN>` 和 `<TURN_DOMAIN>` 进入 `<RELAY_HOST>` 上的 LiveKit SFU。真实域名和主机名只保存在部署平台配置中，不写入仓库。

LiveKit 服务端使用固定版本和镜像摘要，不 fork。Android、Swift SDK 分别建立最小公共 fork并锁定提交，改动范围仅包括：

- 强制硬件视频编解码和禁用软件回退。
- 暴露完整摄像头、编码器、解码器和实际媒体统计。
- 实现 TCP 优先、TCP only、UDP 优先和自动传输策略。
- 支持锁定画质模式以及运行时音频参数更新。

LiveKit 当前 Android、Swift SDK 和服务端已经声明 H.265 支持，因此不重写 SFU 或自建媒体协议：

- [Android VideoCodec](https://github.com/livekit/client-sdk-android/blob/main/livekit-android-sdk/src/main/java/io/livekit/android/room/track/LocalVideoTrackOptions.kt)
- [Swift VideoCodec](https://github.com/livekit/client-sdk-swift/blob/main/Sources/LiveKit/Types/Codec/VideoCodec.swift)
- [LiveKit 服务端配置](https://github.com/livekit/livekit/blob/master/config-sample.yaml)

## 3. 账户、好友和消息

### 3.1 认证与设备

- 密码使用 Argon2id。
- API 使用 15 分钟 Ed25519 JWT；刷新令牌为 30 天、逐设备保存、哈希存储、每次刷新轮换。
- 注册需要高熵注册令牌。令牌不绑定邮箱、无到期时间、允许重复使用，直到管理员撤销。
- 管理员 CLI 支持创建、列出和撤销注册令牌；令牌明文只显示一次，每次注册均记录令牌 ID、用户、设备、时间及安全审计信息。
- 首台设备注册后自动批准。新设备登录必须同时满足密码正确和已批准旧设备确认。
- 丢失全部已批准设备或忘记密码时，v1 不提供账户恢复或管理员密码重置。
- 本地访问凭据保存在 iOS Keychain 或 Android Keystore；推送令牌在服务端加密保存。

### 3.2 好友和在线状态

- 用户必须输入完整邮箱精确查找，不提供公开用户目录或模糊搜索。
- 支持发送、取消、接受、拒绝好友请求及删除好友。
- 只有已接受好友可以查看在线状态、发送消息或发起通话。
- Redis 保存在线状态、心跳、限流、WebSocket 分发和临时通话状态；离线状态由 TTL 收敛。

### 3.3 文字消息

- 仅支持 1:1 纯文字；不支持附件、编辑和撤回。
- 客户端生成幂等 ID，服务端生成 UUIDv7 消息 ID、权威时间和分页游标。
- 客户端支持离线队列和 `pending`、`sent`、`delivered`、`read` 状态。
- PostgreSQL 永久保留消息与回执，不自动过期。
- v1 正文使用 `encoding=plaintext`；信封包含版本字段，后续可增加 `encoding=e2ee`，无需破坏现有 API。

## 4. 来电和通话控制

- 呼叫仅允许好友之间发起；单个 LiveKit 房间最多 2 名参与者。
- 来电事件通过 WebSocket发送，同时通过 APNs/FCM 覆盖后台和离线设备。
- 同一账户所有批准设备同时响铃。第一台接听的设备事务性获胜，其他设备收到取消事件。
- 30 秒无应答记为未接；支持取消、拒接、忙线、异常断开和正常结束状态。
- LiveKit 令牌仅在接听成功后签发，绑定用户 ID、设备 ID、房间 ID和最小权限。
- iOS 使用 PushKit + CallKit，并仅在 CallKit 激活窗口启动音频会话。
- Android 使用 FCM 高优先级数据消息、Core Telecom/ConnectionService 和 CallStyle 通知。

## 5. 媒体能力与质量设置

### 5.1 视频

- v1 视频编码仅提供 H.264 和 H.265，采用 8-bit SDR。
- Android 检测 Camera2 高速格式、MediaCodec 硬件标志、profile/level 和 PerformancePoint。
- iOS 检测 AVCaptureDevice 格式、帧率范围及 VideoToolbox 硬件编解码能力。
- App 与对端交换 `MediaCapabilities`，再与服务端策略取交集。缺少摄像头、硬编码或硬解码支持的组合不显示或禁用，并给出原因。
- 可选分辨率和帧率来自真实设备格式，不接受任意无效输入；上限为 3840×2160、120fps。
- 默认视频码率策略上限为 120Mb/s，并受本机、对端和服务端能力进一步约束。
- 显示发送设置和接收上限；实际协商结果必须同时满足双方能力。
- 1:1 默认关闭 simulcast，仅创建一个硬件编码层，避免重复编码和内存带宽浪费。

默认使用自适应模式；高级设置提供锁定模式：

- 自适应模式允许根据网络和热状态调整分辨率、帧率和码率，所有调整必须在 UI 明示。
- 锁定模式不自动改变用户选择的编码、分辨率或帧率。
- WebRTC 拥塞控制始终保留。发送队列有界，过期帧丢弃；极端拥塞时暂停视频并继续音频，避免无限延迟和内存增长。
- 实际线路码率可能受拥塞控制影响；UI 同时显示请求值、协商值和实际值，禁止静默伪装为目标码率。

性能实现：

- Android 优先 Camera2/Surface/MediaCodec 零拷贝路径。
- iOS 优先 AVCaptureSession/CVPixelBuffer/VideoToolbox/Metal 路径。
- 渲染层避免无关 Compose/SwiftUI 重绘，离屏视频立即停止渲染并释放引用。
- 监控热状态、编码队列、帧丢弃、编码耗时和内存压力。
- 若只有软件视频编码器，相关档位直接禁用，不自动降级到软件编码。

### 5.2 音频

音频仅使用 Opus。移动平台没有可用硬件 Opus 时使用 SDK 原生优化实现；硬件强制规则主要针对高负载视频。

预设：

- 通话：32kb/s、单声道、DTX/FEC/RED 开启、AEC/NS/AGC 开启。
- 高清：64kb/s、单声道、FEC/RED 开启、DTX 关闭。
- 音乐：160kb/s、立体声、DTX/RED 关闭、语音处理关闭。
- 极致：256kb/s、立体声、DTX/RED 关闭、语音处理关闭。

高级设置允许 16–510kb/s、单双声道以及 DTX、FEC、RED、AEC、NS、AGC 独立设置。关闭回声消除且使用扬声器时显示反馈风险提示。

### 5.3 运行时统计

通话页和诊断页显示：

- 请求、协商和实际分辨率、帧率、码率。
- 编码器、解码器名称及硬件状态。
- RTT、抖动、丢包、帧丢弃、关键帧和重连次数。
- ICE 候选类型、最终传输协议、TURN/直连状态及回退原因。

## 6. TCP 优先传输

定义以下配置值：

```text
server_default
tcp_preferred
tcp_only
udp_preferred
auto
```

- 服务端默认 `tcp_preferred`；用户可在每台设备的高级设置中覆盖。
- `tcp_preferred`：先使用 `turns:<TURN_DOMAIN>:443?transport=tcp` 和 relay-only 建连。3 秒未成功则执行 ICE restart，放开 LiveKit 标准 ICE-TCP/UDP 回退。
- `tcp_only`：仅使用 TURN/TLS，不回退 UDP；失败时明确报告原因。
- `udp_preferred`：先使用 UDP，3 秒未成功再放开标准自动回退。
- `auto`：使用 LiveKit 上游默认 ICE 行为。
- 设置在下一次通话生效。诊断页显示首选阶段、回退阶段及最终路径。

SDK fork负责两阶段连接、超时、ICE restart 和统计暴露；不通过修改 SFU 或关闭拥塞保护实现。

## 7. 公共 API 和数据模型

`contracts/` 保存 OpenAPI 3.1 和版本化 WebSocket 事件定义。

核心公共类型：

```text
MediaCapabilities
VideoProfile
AudioProfile
QualityMode
TransportPolicy
NegotiatedMediaProfile
ActiveMediaStats
```

REST API 分组：

```text
/v1/auth/*
/v1/devices/*
/v1/users/by-email
/v1/friends/*
/v1/conversations/*
/v1/messages/*
/v1/calls/*
/v1/client-config
/v1/ws
```

WebSocket 事件统一包含：

```json
{
  "version": 1,
  "event_id": "uuidv7",
  "type": "event.name",
  "occurred_at": "RFC3339 timestamp",
  "payload": {}
}
```

PostgreSQL 主要实体包括用户、密码凭据、注册令牌、注册审计、设备、刷新会话、推送令牌、好友请求、好友关系、会话、消息、回执、通话和逐设备通话状态。

## 8. 部署方案

### 8.1 业务服务主机

- 通过 Dokploy 部署单实例 Go API。
- 复用现有私有 PostgreSQL 和 Redis；创建独立数据库、用户、迁移历史及 `mchat:` Redis 命名空间。
- `<API_DOMAIN>` 通过现有 Traefik 443 提供 HTTPS/WSS。
- 数据库和 Redis 不开放公共端口。

### 8.2 通话中继主机

- 通过 Dokploy 部署单节点 LiveKit，不增加 LiveKit Redis、coturn 或转码服务。
- `<RTC_DOMAIN>:443` 通过 Traefik HTTP 路由转发到容器内部 7880。
- `<TURN_DOMAIN>:443` 使用 Traefik TCP SNI 和 TLS 终止，转发到启用 `external_tls` 的 LiveKit TURN。
- 公开 `7881/tcp` 供 ICE-TCP 使用。
- UDP 端口：`3478/udp`、`50000–50031/udp` RTC、`30000–30031/udp` TURN relay。
- 实施前确认现有 `7881/tcp` Matrix LiveKit 防火墙规则已无归属；仅在确认遗留后更新注释或替换规则。
- 两个 RTC 域名均使用 DNS-only，避免代理改写媒体连接。

### 8.3 镜像、监控和密钥

- GitHub Actions 构建 Go ARM64 镜像并以提交 SHA 发布到 GHCR；部署锁定镜像摘要。
- LiveKit 使用官方固定版本和摘要。
- API 和 LiveKit 提供 Prometheus 指标；复用现有 Prometheus/Grafana，覆盖 API 延迟、错误、在线设备、通话数、码率、丢包、RTT、CPU、内存和重连。
- 指标端口仅通过私有网络或 Tailscale 暴露。
- PostgreSQL、Redis、JWT、LiveKit、APNs、FCM 和签名凭据只存入 macOS Keychain、GitHub Secret 或 Dokploy Secret，禁止写入仓库、`.env` 和命令参数。

## 9. 构建、测试和验收

### 9.1 自动化测试

- Go：认证、令牌撤销、设备审批、好友权限、消息幂等、离线同步、回执、多设备抢接、迁移、WebSocket 重连和 LiveKit token 权限。
- Android/iOS：能力矩阵、硬件编码过滤、H.264/H.265 协商、音频档位、锁定/自适应、生命周期、内存泄漏、热状态和本地队列。
- 合同测试：OpenAPI、WebSocket 事件和跨端共享 JSON 样例。
- CI：Go 测试和镜像构建、Android unit/lint/assemble、iOS XCTest/build、依赖许可证和密钥泄漏检查。

### 9.2 推送和系统来电

使用用户提供的 APNs、FCM 和签名资料，在真机验证：

- App 前台、后台及进程被杀状态。
- 接听、拒接、取消、忙线、30 秒超时。
- 多设备首接获胜及重复推送幂等。
- Android 和 iOS 系统通话 UI、音频焦点及蓝牙/听筒/扬声器切换。

### 9.3 网络测试

- 分别阻断 UDP、TURN/TLS 和 ICE-TCP。
- 验证 `tcp_preferred` 的 3 秒回退、`tcp_only` 的明确失败、`udp_preferred` 和 `auto`。
- 确认 TCP 优先成功时最终路径确为 TURN/TLS TCP，不能静默使用 UDP。
- 验证网络切换、ICE restart、断网恢复和锁定画质下的有界队列。

### 9.4 性能验收

- 使用预编码 3840×2160、120fps 的 H.264 和 H.265 流，在 `<RELAY_HOST>` 分别进行 1 路及 2 路并发、每组 30 分钟压测。
- 要求无容器重启或 OOM，LiveKit 平均 CPU 低于 70%、RSS 低于 1.5GB、服务端引入丢包低于 0.5%。
- 验证现有 Plex、录制器和 Traefik 不受影响。
- 无兼容 Android/iOS 真机时，4K/120 移动端状态标记为“能力实现完成、待真机验证”，不得宣传为已验证。
- 若 `<RELAY_HOST>` 压测不达标，服务端策略暂时隐藏 4K/120，直到主机升级或调优通过。

## 10. 交付默认值

- Android applicationId、iOS bundle ID：`com.whoerau.mchat`。
- 域名：仓库仅引用 `<API_DOMAIN>`、`<RTC_DOMAIN>`、`<TURN_DOMAIN>`；真实值通过 Dokploy Secret 或部署环境注入。
- 首版 UI：简体中文；字符串资源化，便于后续本地化。
- 默认质量模式：自适应；手动锁定可选。
- 默认传输：`tcp_preferred`；用户可逐设备覆盖。
- 默认视频编码：在双方硬件支持交集中优先 H.265，否则 H.264；不使用软件视频编码回退。
- 默认音频：Opus 高清档。
- 交付签名 release APK/AAB、开发或 Ad Hoc IPA；不提交 App Store 或 Google Play。
- 4K/120 真机验收待兼容设备提供后补做；服务端转发压测仍属于首版验收。
