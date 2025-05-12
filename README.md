# WebRTC

WebRTC (Web Real-Time Communication) 是一项开源技术，允许网页浏览器之间进行实时音视频通信，无需安装插件或第三方软件。主要特点：

- 点对点通信：减少服务器负担，降低延迟
- 开放标准：由 W3C 和 IETF 标准化
- 跨平台：支持多种浏览器和设备

## 媒体流（MediaStream）

- 由任意数量的媒体信息轨道（MediaStreamTrack）组成。
- 包含音频、视频、文本（如字幕）等。
- 用于发送和接收实时或存储的媒体信息。

## 数据通道 (RTCDataChannel)

WebRTC 不仅支持音视频传输，还提供了数据通道功能，用于在浏览器之间传输任意数据：

```JS
// 创建数据通道
const dataChannel = peerConnection.createDataChannel("myChannel", {
  ordered: true,  // 是否保证消息顺序
  maxRetransmits: 3  // 最大重传次数
});

// 监听数据通道事件
dataChannel.onopen = () => {
  console.log("数据通道已打开");
  dataChannel.send("Hello from data channel!");
};

dataChannel.onmessage = (event) => {
  console.log("收到消息:", event.data);
};

// 接收方监听数据通道
peerConnection.ondatachannel = (event) => {
  const receivedChannel = event.channel;
  receivedChannel.onmessage = (e) => {
    console.log("收到数据:", e.data);
  };
};
```

数据通道应用场景：

- 文字聊天
- 文件传输
- 游戏状态同步
- 远程控制命令

## 屏幕共享实现

WebRTC 可以实现屏幕共享功能，允许用户共享他们的桌面或应用程序：

```JS
// 获取屏幕媒体流
navigator.mediaDevices.getDisplayMedia({
  video: {
    cursor: "always"  // 显示鼠标指针
  },
  audio: false
}).then(screenStream => {
  // 替换视频轨道
  const videoTrack = screenStream.getVideoTracks()[0];
  const sender = peerConnection.getSenders().find(s =>
    s.track.kind === "video"
  );

  if (sender) {
    sender.replaceTrack(videoTrack);
  }

  // 监听屏幕共享结束
  videoTrack.onended = () => {
    console.log("屏幕共享已结束");
    // 恢复摄像头视频
  };
}).catch(err => {
  console.error("获取屏幕共享失败:", err);
});
```

## 多人会议架构

### 1. 网状结构 (Mesh)

每个参与者与其他所有参与者建立点对点连接：

- 优点：延迟低，服务器负担小
- 缺点：连接数量呈指数增长，不适合大规模会议
- 适用场景：4 人以下的小型会议

```js
// 为每个远程用户创建一个 RTCPeerConnection
const peerConnections = {}

function createPeerConnection(userId) {
  peerConnections[userId] = new RTCPeerConnection(pcConfig)
  // 设置事件处理...
  return peerConnections[userId]
}
```

### 2. SFU (Selective Forwarding Unit)

中央服务器接收所有参与者的媒体流，并选择性地转发给其他参与者：

- 优点：扩展性好，参与者上传一次流
- 缺点：需要专门的服务器，有一定延迟
- 适用场景：中大型会议（5-50 人）

### 3. MCU (Multipoint Control Unit)

中央服务器接收、解码、混合所有媒体流，再编码发送给参与者：

- 优点：客户端负担小，适合弱网环境
- 缺点：服务器计算量大，延迟较高
- 适用场景：大型会议或直播

## 媒体约束和质量控制

可以通过详细的媒体约束来控制音视频质量：

```js
navigator.mediaDevices
  .getUserMedia({
    audio: {
      echoCancellation: true, // 回声消除
      noiseSuppression: true, // 噪声抑制
      autoGainControl: true, // 自动增益控制
    },
    video: {
      width: { ideal: 1280 }, // 理想宽度
      height: { ideal: 720 }, // 理想高度
      frameRate: { min: 15, ideal: 30, max: 60 }, // 帧率范围
      facingMode: 'user', // 前置摄像头
    },
  })
  .then((stream) => {
    // 处理媒体流
  })
```

## 带宽适应和拥塞控制

WebRTC 内置了带宽估计和拥塞控制机制，但也可以手动调整：

```js
// 设置视频比特率上限
const sender = peerConnection.getSenders()[0]
const parameters = sender.getParameters()
if (!parameters.encodings) {
  parameters.encodings = [{}]
}
parameters.encodings[0].maxBitrate = 1000000 // 1Mbps
sender.setParameters(parameters)

// 监听连接状态变化
peerConnection.onconnectionstatechange = () => {
  console.log('连接状态:', peerConnection.connectionState)
}

// 获取连接统计信息
setInterval(() => {
  peerConnection.getStats().then((stats) => {
    stats.forEach((report) => {
      if (report.type === 'inbound-rtp' && report.kind === 'video') {
        console.log('接收帧率:', report.framesPerSecond)
        console.log('丢包率:', report.packetsLost / report.packetsReceived)
      }
    })
  })
}, 1000)
```

## 信令服务器实现

```js
// 服务端 (Node.js)
const http = require('http')
const server = http.createServer()
const io = require('socket.io')(server)

io.on('connection', (socket) => {
  console.log('用户连接:', socket.id)

  // 加入房间
  socket.on('join', (room) => {
    socket.join(room)
    const clients = io.sockets.adapter.rooms.get(room)
    const numClients = clients ? clients.size : 0

    // 通知房间内其他用户
    socket.to(room).emit('user-joined', { id: socket.id })

    // 通知当前用户
    socket.emit('room-joined', {
      room,
      numClients,
      isInitiator: numClients === 1,
    })
  })

  // 转发信令消息
  socket.on('message', (data) => {
    socket.to(data.room).emit('message', {
      from: socket.id,
      type: data.type,
      payload: data.payload,
    })
  })

  // 断开连接
  socket.on('disconnect', () => {
    console.log('用户断开:', socket.id)
    io.emit('user-left', { id: socket.id })
  })
})

server.listen(3000, () => {
  console.log('信令服务器运行在端口 3000')
})
```

```js
// 客户端
const socket = io('https://your-signaling-server.com')
let room = 'my-room'

// 加入房间
socket.emit('join', room)

// 监听房间加入事件
socket.on('room-joined', (data) => {
  console.log(`加入房间 ${data.room}，当前 ${data.numClients} 人`)
  isInitiator = data.isInitiator

  if (isInitiator) {
    console.log('我是房间创建者')
  }
})

// 监听新用户加入
socket.on('user-joined', (data) => {
  console.log(`新用户加入: ${data.id}`)
  // 创建对等连接...
})

// 发送信令消息
function sendSignalingMessage(type, payload) {
  socket.emit('message', {
    room: room,
    type: type,
    payload: payload,
  })
}

// 接收信令消息
socket.on('message', (data) => {
  console.log(`收到来自 ${data.from} 的消息:`, data.type)

  switch (data.type) {
    case 'offer':
      handleOffer(data.payload, data.from)
      break
    case 'answer':
      handleAnswer(data.payload)
      break
    case 'candidate':
      handleCandidate(data.payload)
      break
  }
})
```

## WebRTC 安全性考虑

### 1. 端到端加密

WebRTC 默认对所有媒体流和数据通道进行端到端加密，使用 DTLS 和 SRTP 协议：

```js
// 检查是否使用安全连接
peerConnection.getSenders().forEach((sender) => {
  sender.getStats().then((stats) => {
    stats.forEach((report) => {
      if (report.type === 'transport') {
        console.log('传输加密状态:', report.dtlsState)
      }
    })
  })
})
```

### 2. 身份验证

在信令阶段实现用户身份验证：

```js
// 客户端
socket.emit('authenticate', { token: 'user-auth-token' })

// 服务端
socket.on('authenticate', (data) => {
  verifyToken(data.token)
    .then((user) => {
      socket.user = user
      socket.emit('authenticated', { success: true })
    })
    .catch((err) => {
      socket.emit('authenticated', {
        success: false,
        error: '身份验证失败',
      })
    })
})
```

### 3. 权限控制

控制谁可以加入会议、发言或共享屏幕：

```js
// 服务端实现权限检查
socket.on('join', (room) => {
  if (!hasPermission(socket.user, room, 'join')) {
    return socket.emit('error', { message: '没有权限加入此房间' })
  }
  // 允许加入...
})
```

## 性能优化技巧

### 1. 自适应比特率

```js
// 根据网络状况调整视频质量
peerConnection.addEventListener('connectionstatechange', () => {
  if (peerConnection.connectionState === 'connected') {
    // 连接建立后定期检查
    setInterval(async () => {
      const stats = await peerConnection.getStats()
      let videoSender

      stats.forEach((report) => {
        if (report.type === 'outbound-rtp' && report.kind === 'video') {
          videoSender = peerConnection
            .getSenders()
            .find((s) => s.track.kind === 'video')

          // 根据丢包率调整比特率
          if (report.retransmittedPacketsSent > 0) {
            const lossRate =
              report.retransmittedPacketsSent / report.packetsSent

            if (lossRate > 0.1) {
              // 丢包率超过10%
              reduceVideoBitrate(videoSender)
            } else if (lossRate < 0.05) {
              // 丢包率低于5%
              increaseVideoBitrate(videoSender)
            }
          }
        }
      })
    }, 3000)
  }
})
```

### 2. 视频分层编码

使用 SVC (Scalable Video Coding) 实现更好的适应性

```js
navigator.mediaDevices
  .getUserMedia({
    video: {
      // 启用分层编码
      scalabilityMode: 'L3T3', // 3个空间层和3个时间层
    },
  })
  .then((stream) => {
    // 使用支持SVC的编码器
  })
```

## 调试与故障排除

### 1. WebRTC 内部统计

```js
// 定期收集并分析连接统计
function analyzeConnection() {
  peerConnection.getStats(null).then((stats) => {
    stats.forEach((report) => {
      switch (report.type) {
        case 'inbound-rtp':
          console.log('接收字节:', report.bytesReceived)
          console.log('丢包数:', report.packetsLost)
          break
        case 'outbound-rtp':
          console.log('发送字节:', report.bytesSent)
          console.log('发送包数:', report.packetsSent)
          break
        case 'candidate-pair':
          if (report.state === 'succeeded') {
            console.log('当前RTT:', report.currentRoundTripTime)
            console.log('可用带宽:', report.availableOutgoingBitrate)
          }
          break
      }
    })
  })
}

setInterval(analyzeConnection, 2000)
```

### 2. ICE 连接问题排查

```js
// 监控ICE连接状态
peerConnection.oniceconnectionstatechange = () => {
  console.log('ICE连接状态:', peerConnection.iceConnectionState)

  switch (peerConnection.iceConnectionState) {
    case 'failed':
      console.error('ICE连接失败')
      // 尝试重启ICE
      peerConnection.restartIce()
      break
    case 'disconnected':
      console.warn('ICE连接断开')
      // 可能是临时网络问题，等待恢复
      break
    case 'connected':
      console.log('ICE连接成功')
      break
  }
}

// 记录ICE候选收集过程
peerConnection.onicegatheringstatechange = () => {
  console.log('ICE收集状态:', peerConnection.iceGatheringState)
}
```

## 实际应用

1. 视频会议系统

```js
// 动态布局管理
function updateLayout() {
  const videoElements = document.querySelectorAll('.remote-video')
  const count = videoElements.length

  // 根据参与者数量调整布局
  if (count <= 1) {
    // 全屏显示
    videoElements.forEach((el) => {
      el.style.width = '100%'
      el.style.height = '100%'
    })
  } else if (count <= 4) {
    // 2x2网格
    videoElements.forEach((el) => {
      el.style.width = '50%'
      el.style.height = '50%'
    })
  } else {
    // 多人网格
    const width = Math.floor(100 / Math.ceil(Math.sqrt(count)))
    videoElements.forEach((el) => {
      el.style.width = `${width}%`
      el.style.height = `${width}%`
    })
  }
}

// 当有参与者加入或离开时更新布局
function addParticipant(id, stream) {
  const videoEl = document.createElement('video')
  videoEl.id = `video-${id}`
  videoEl.className = 'remote-video'
  videoEl.autoplay = true
  videoEl.srcObject = stream

  document.getElementById('videos-container').appendChild(videoEl)
  updateLayout()
}
```
