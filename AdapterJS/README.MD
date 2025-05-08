## WebRTC 基础概念

WebRTC (Web Real-Time Communication) 是一项开源技术，允许网页浏览器之间进行实时音视频通信，无需安装插件或第三方软件。主要特点：

- 点对点通信：减少服务器负担，降低延迟
- 开放标准：由 W3C 和 IETF 标准化
- 跨平台：支持多种浏览器和设备

## AdapterJS 的作用

AdapterJS 是一个兼容层，解决不同浏览器对 WebRTC API 实现不一致的问题：

```JS
AdapterJS.webRTCReady(function(isUsingPlugin) {
  // WebRTC 代码
});
```

- 提供统一的 API 接口
- 支持旧版浏览器（如 IE）通过插件实现 WebRTC
- 自动检测环境并适配相应实现

## 媒体流获取

通过 getUserMedia() API 获取本地摄像头和麦克风：

```js
navigator.mediaDevices.getUserMedia({
  audio: true,
  video: true,
})
```

常见错误处理：

- NotFoundError：找不到媒体设备
- NotAllowedError：用户拒绝权限
- NotReadableError：设备被占用

## RTCPeerConnection

RTCPeerConnection 是 WebRTC 的核心 API，用于建立和管理连接：

```js
peerConnection = new RTCPeerConnection(pcConfig)
peerConnection.onicecandidate = handleIceCandidate
peerConnection.onaddstream = handleRemoteStreamAdded
```

主要功能：

- 管理音视频流
- 处理 ICE 候选
- 创建和交换 SDP 信息

## 信令过程

信令是 WebRTC 通信的基础，通过信令服务器传递 SDP 信息：

- 发送方创建 offer，发送给信令服务器
- 接收方收到 offer，创建 answer，发送给信令服务器
- 双方通过信令服务器交换 SDP 信息

```JS
// 创建 Offer/Answer ：
peerConnection.createOffer()
  .then(setLocalAndSendMessage)

// 交换 SDP
peerConnection.setLocalDescription(sessionDescription)
// 通过信令服务器发送到对方
peerConnection.setRemoteDescription(new RTCSessionDescription(sessionDescription))

// 处理 ICE 候选
peerConnection.addIceCandidate(new RTCIceCandidate(candidate))
```

## ICE 框架

Interactive Connectivity Establishment (ICE) 帮助穿透 NAT 和防火墙：

```js
const pcConfig = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
  ],
}
```

- STUN 服务器：帮助发现公网 IP 和端口
- TURN 服务器：当直接连接失败时提供中继

## 媒体流处理

```js
// 添加本地流到连接
peerConnection.addStream(localStream)

// 接收远程流
function handleRemoteStreamAdded(event) {
  remoteVideo.srcObject = event.stream
}
```

## 连接生命周期管理

```js
// 创建连接
createPeerConnection()

// 关闭连接
if (peerConnection) {
  peerConnection.close()
  peerConnection = null
}
```

## 设备检测与适配

```js
navigator.mediaDevices.enumerateDevices().then((devices) => {
  const hasVideoInput = devices.some((device) => device.kind === 'videoinput')
  const hasAudioInput = devices.some((device) => device.kind === 'audioinput')
})
```

## 注意事项

- 真实环境需要信令服务器 ：

  - WebSocket 或 Socket.IO 实现信令
  - 服务器转发 SDP 和 ICE 候选信息

- 安全性考虑 ：

  - 使用 HTTPS
  - 实现用户认证
  - 加密信令通道

- 性能优化 ：

  - 适应不同网络条件
  - 动态调整视频质量
  - 处理网络波动
