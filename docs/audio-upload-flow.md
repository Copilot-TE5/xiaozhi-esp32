# 音频采集与上传流程详解

本文档详细说明设备从麦克风采集音频，经过处理、编码，最终上传到服务器的完整流程与内部机制。

---

## 1. 整体架构概览

整个上行（设备 → 服务器）音频链路由以下四个主要阶段组成：

```
麦克风 (I2S)
    │
    ▼
[AudioInputTask]  ──读取 PCM──▶  [AudioProcessor / AFE]
                                        │
                                   VAD 状态通知
                                        │
                                        ▼
                               audio_encode_queue_  (最多 2 帧)
                                        │
                                        ▼
                               [OpusCodecTask]  ──Opus 编码──▶  audio_send_queue_  (最多 40 包)
                                                                          │
                                                                          ▼
                                                             [Application::Run() 主循环]
                                                                          │
                                                                          ▼
                                                           protocol_->SendAudio()
                                                                          │
                                                                          ▼
                                                              WebSocket / MQTT
                                                                          │
                                                                          ▼
                                                                    云端服务器
```

---

## 2. 各阶段详细说明

### 2.1 阶段一：硬件采集（AudioInputTask）

**文件：** `main/audio/audio_service.cc`，函数 `AudioService::AudioInputTask()`

**职责：** 以固定时间间隔从 I2S 麦克风读取原始 PCM 数据，并送往下一阶段。

**关键参数：**
- 采样率：16 000 Hz（16 kHz），若硬件编解码器的原生采样率不同，则通过 `OpusResampler` 先做重采样
- 帧长：每次读取 160 个采样点（= 10 ms @ 16 kHz）
- 格式：有符号 16-bit PCM，单声道

**执行逻辑：**
1. 等待 `AS_EVENT_AUDIO_PROCESSOR_RUNNING` 事件位，确认处理器已就绪
2. 调用 `codec_->InputData()` 通过 I2S DMA 读取一帧原始 PCM
3. 若硬件采样率 ≠ 16 kHz，则调用 `input_resampler_` 进行重采样
4. 将处理后的 PCM 送入 `audio_processor_->Feed()`

> 此任务运行于独立的 FreeRTOS 任务中（栈大小约 8 KB，固定在 Core 0 执行），不会阻塞主循环。

---

### 2.2 阶段二：音频前端处理（AfeAudioProcessor）

**文件：** `main/audio/processors/afe_audio_processor.cc`，函数 `AfeAudioProcessor::AudioProcessorTask()`

**职责：** 对原始 PCM 进行实时音频增强处理，输出干净的 60 ms 帧，并检测语音活动（VAD）。

**处理模块（ESP Audio Front-End，AFE）：**
| 模块 | 全称 | 作用 |
|------|------|------|
| AEC | Acoustic Echo Cancellation | 消除扬声器回声 |
| VAD | Voice Activity Detection | 检测是否有人在说话 |
| NSNet | Noise Suppression Network | 神经网络降噪 |
| AGC | Automatic Gain Control | 自动增益（可选） |

**VAD 状态机：**

```
        ┌────────────────────────────────┐
        │   AFE 每帧输出 vad_state       │
        └──────┬─────────────────────────┘
               │
    ┌──────────┼──────────────┐
    │ VAD_SILENCE             │ VAD_SPEECH
    ▼                         ▼
is_speaking_=false       is_speaking_=true
    │                         │
vad_state_change_callback_(false)   vad_state_change_callback_(true)
    │                         │
    └────────── 通知 ──────────┘
                  │
          voice_detected_ 标志位
          + on_vad_change() 回调
                  │
       应用层根据当前侦听模式决策：
       - auto 模式：静音时自动停止上传
       - manual/realtime 模式：由用户或定时器控制
```

**输出帧尺寸：** 960 个采样点（= 60 ms @ 16 kHz），与 Opus 编码器所需帧长严格对齐。

**数据流出：** 处理后的 PCM 通过 `output_callback_` 回调传给 `AudioService::PushTaskToEncodeQueue()`。

---

### 2.3 阶段三：Opus 编码（OpusCodecTask）

**文件：** `main/audio/audio_service.cc`，函数 `AudioService::OpusCodecTask()`

**职责：** 从编码队列取出原始 PCM，使用 Opus 编解码器压缩，并将数据包推入发送队列。

**Opus 编码器配置：**
```c
// audio_service.h 中的宏定义
{
    .sample_rate        = ESP_AUDIO_SAMPLE_RATE_16K,   // 16 kHz
    .channel            = ESP_AUDIO_MONO,               // 单声道
    .bits_per_sample    = ESP_AUDIO_BIT16,              // 16-bit
    .bitrate            = ESP_OPUS_BITRATE_AUTO,        // VBR 自适应码率
    .frame_duration     = 60ms,                         // 与 AFE 输出帧对齐
    .application_mode   = AUDIO,
    .complexity         = 0,                            // 低复杂度（实时优先）
    .enable_fec         = false,
    .enable_dtx         = true,                         // 静音帧压缩
    .enable_vbr         = true,                         // 可变码率
}
```

**DTX（不连续传输）说明：**  
`enable_dtx = true` 意味着纯静音帧会被编码成极小的包（约 10 字节）甚至跳过，减少网络流量。

**执行逻辑（编码方向）：**
```
等待条件：audio_encode_queue_ 非空 && audio_send_queue_.size() < 40
    │
    ▼
从 audio_encode_queue_ 取出 AudioTask（含 PCM 数据）
    │
    ▼
调用 esp_opus_enc_process() 编码 960 采样点 → Opus 二进制包
    │
    ▼
创建 AudioStreamPacket：
  ├── sample_rate   = 16000
  ├── frame_duration = 60
  ├── timestamp     = 从 timestamp_queue_ 取出（用于服务器端 AEC）
  └── payload       = Opus 二进制数据（约 30～100 字节）
    │
    ▼
推入 audio_send_queue_
    │
    ▼
调用 callbacks_.on_send_queue_available()
→ 在主事件组中置位 MAIN_EVENT_SEND_AUDIO
```

**队列限流：**
- `audio_encode_queue_` 最多容纳 2 帧，防止内存溢出
- `audio_send_queue_` 最多容纳 40 包 (= 2400 ms 缓冲)，吸收网络抖动

---

### 2.4 阶段四：发送至服务器（Application 主循环 + Protocol）

**文件：** `main/application.cc`，`Application::Run()`；`main/protocols/websocket_protocol.cc`

**执行逻辑：**
```
Application::Run() 主循环
    │
    ├── 等待 xEventGroupWaitBits(MAIN_EVENT_SEND_AUDIO | ...)
    │
    ▼
bits & MAIN_EVENT_SEND_AUDIO
    │
    ▼
while (audio_service_.HasPacketsToSend())
    │
    ├── packet = audio_service_.PopPacketFromSendQueue()
    │
    └── protocol_->SendAudio(std::move(packet))
             │
             ▼
    WebsocketProtocol::SendAudio()
             │
    ┌────────┴──────────────┐
    │ 根据协议版本序列化包  │
    └────────┬──────────────┘
             │
    ┌────────┴──────────────────────────────────────────┐
    │ 版本 1：直接发送裸 Opus 负载                       │
    │ 版本 2：附加 16 字节头（含 timestamp，用于 AEC）  │
    │ 版本 3：附加 4 字节头（type + payload_size）       │
    └────────┬──────────────────────────────────────────┘
             │
    websocket_->Send(data, size, binary=true)
             │
             ▼
         云端服务器
```

---

## 3. 二进制传输协议格式

### 版本 1（默认，轻量）
```
┌─────────────────────────────┐
│     Opus 负载（N 字节）     │
└─────────────────────────────┘
```

### 版本 2（含时间戳，支持服务器 AEC）
```
┌────────────┬────────────┬──────────────┬─────────────────┬────────────────────┬──────────────────┐
│ version    │ type       │ reserved     │ timestamp       │ payload_size       │ payload          │
│ (2 字节)   │ (2 字节)   │ (4 字节)     │ (4 字节, ms)    │ (4 字节)           │ (N 字节)         │
└────────────┴────────────┴──────────────┴─────────────────┴────────────────────┴──────────────────┘
  合计头部 16 字节，大端序（网络字节序）
```

### 版本 3（紧凑型）
```
┌────────────┬────────────┬────────────────────┬──────────────────┐
│ type       │ reserved   │ payload_size        │ payload          │
│ (1 字节)   │ (1 字节)   │ (2 字节, 大端序)    │ (N 字节)         │
└────────────┴────────────┴────────────────────┴──────────────────┘
  合计头部 4 字节
```

其中 `type` 字段：`0` = Opus 音频，`1` = JSON 文本。

---

## 4. 会话建立流程（WebSocket 通道）

在音频上传开始之前，设备必须先与服务器完成握手：

```
1. 设备发起网络连接
   └── 携带请求头：
       Authorization: Bearer <token>
       Protocol-Version: <1|2|3>
       Device-Id: <MAC 地址>
       Client-Id: <UUID>

2. 设备发送 JSON hello 消息
   {
     "type": "hello",
     "version": 2,
     "features": { "mcp": true },
     "transport": "websocket",
     "audio_params": {
       "format": "opus",
       "sample_rate": 16000,
       "channels": 1,
       "frame_duration": 60
     }
   }

3. 服务器返回 hello 响应（10 秒超时）
   {
     "type": "hello",
     "transport": "websocket",
     "session_id": "...",
     "audio_params": {
       "format": "opus",
       "sample_rate": 24000,   // 服务器下行 TTS 音频使用 24 kHz，与设备上行的 16 kHz 不同
       "channels": 1,
       "frame_duration": 60
     }
   }
   // 注意：设备上行（麦克风→服务器）始终使用 16 kHz；
   // 服务器下行（TTS→扬声器）可使用更高采样率（如 24 kHz）以提升语音质量。
   // OpusCodecTask 在解码时会自动对下行音频进行重采样至硬件编解码器所需的采样率。

4. 握手成功，音频通道打开
   └── 触发 on_audio_channel_opened_() 回调
   └── 设备状态切换至 kDeviceStateListening

5. 开始上传 Opus 音频包（二进制帧）
```

---

## 5. 状态机与上传触发时机

设备并非一直上传音频；只有在特定状态下音频才会被编码并发送。

```
kDeviceStateIdle（空闲）
    │
    ├── 检测到唤醒词（如"你好小智"）
    │       └── 发送 SendWakeWordDetected() JSON 消息
    │
    ├── 用户按下物理按键（手动触发）
    │
    └── 进入 kDeviceStateConnecting（连接中）
              │ OpenAudioChannel() 成功
              ▼
         kDeviceStateListening（侦听中）◄── 音频上传激活
              │
              ├── VAD 检测到静音（auto 模式）→ SendStopListening()
              │
              ├── 用户再次按键（manual 模式）→ SendStopListening()
              │
              └── 服务器返回 TTS 音频
                        │
                        ▼
               kDeviceStateSpeaking（说话/播放中）
                        │
                   播放完毕 / 唤醒词打断
                        │
                        ▼
               kDeviceStateIdle（返回空闲）
```

> **关键点：** `AudioService::StartRecording()` 和 `StopRecording()` 控制 AFE 处理器的启停；AFE 只有在 `kDeviceStateListening` 时才会向编码队列推送数据，其他状态下麦克风仍然工作但不送入上传链路（仅用于唤醒词检测）。

---

## 6. 线程模型与并发控制

整个上行链路由三个 FreeRTOS 任务并行运行，通过共享队列（受互斥锁保护）通信：

```
┌──────────────────────────────────────────────────────────────────────┐
│ Core 0                          Core 1 (或同 Core 0)                 │
│                                                                      │
│  AudioInputTask ──PCM──▶  audio_encode_queue_  ──▶  OpusCodecTask   │
│  (栈 8 KB, 高优先级)           (互斥锁保护)         (栈 24 KB)       │
│                                                         │             │
│                                                   audio_send_queue_  │
│                                                         │             │
│                                                  MAIN_EVENT_SEND_AUDIO│
│                                                         │             │
│                                              Application::Run() 主循环│
│                                              (发送至 WebSocket)       │
└──────────────────────────────────────────────────────────────────────┘
```

**条件变量（`audio_queue_cv_`）** 用于在任务间同步，避免忙等待：
- `OpusCodecTask` 在队列为空时挂起，有数据推入时被唤醒
- 主循环通过事件组位（`MAIN_EVENT_SEND_AUDIO`）被唤醒执行发送

---

## 7. 时序与缓冲策略

| 阶段 | 帧/包大小 | 时长 | 队列深度 | 总缓冲容量 |
|------|-----------|------|----------|------------|
| 麦克风读取 | 160 采样点 | 10 ms | — | — |
| AFE 输出 | 960 采样点 | 60 ms | — | — |
| 编码等待队列 | 960 采样点 PCM | 60 ms | 最多 2 帧 | 120 ms |
| 发送队列 | Opus 包（~30–100 B） | 60 ms | 最多 40 包 | 2400 ms |

**设计意图：**
- 编码队列故意保持极小（2 帧），确保低延迟；若网络不通导致发送队列满，会自然向前产生背压，丢弃最老的帧
- 发送队列容量 2400 ms 用于吸收短暂的网络抖动或 WebSocket 写入阻塞

---

## 8. 功耗管理

当超过 `AUDIO_POWER_TIMEOUT_MS`（默认 15 秒）没有音频活动时，`CheckAndUpdateAudioPowerState()` 定时器会自动关闭麦克风（ADC）或扬声器（DAC），再次有音频需求时重新上电，延迟约 10–20 ms。

---

## 9. 总结

| 步骤 | 关键代码位置 | 说明 |
|------|-------------|------|
| 1. 麦克风读取 | `AudioService::AudioInputTask()` | I2S DMA 读取，10 ms 一帧 |
| 2. 重采样（可选）| `esp_ae_rate_cvt_process()` | 将硬件采样率转换为 16 kHz |
| 3. AFE 处理 | `AfeAudioProcessor::AudioProcessorTask()` | AEC + VAD + 降噪，输出 60 ms 帧 |
| 4. 推入编码队列 | `AudioService::PushTaskToEncodeQueue()` | 最多 2 帧缓冲 |
| 5. Opus 编码 | `AudioService::OpusCodecTask()` | PCM → Opus，DTX/VBR 优化 |
| 6. 推入发送队列 | `audio_send_queue_.push_back()` | 最多 40 包缓冲，触发事件位 |
| 7. 主循环发送 | `Application::Run()` | 事件驱动，逐包弹出 |
| 8. 协议序列化 | `WebsocketProtocol::SendAudio()` | 按版本添加二进制头 |
| 9. 网络传输 | `websocket_->Send()` | WebSocket 二进制帧，送达服务器 |
