# 在線語音網路穩定性分析

本文件分析 xiaozhi-esp32 專案在線語音功能針對 **網路不穩定**（封包遺失、高延遲）以及 **網路完全中斷**（斷線）兩種情境的應對機制，並說明各項設計如何提升使用者的使用體驗。

---

## 目錄

1. [在線語音架構概述](#在線語音架構概述)
2. [通訊協定](#通訊協定)
3. [網路不穩定的應對機制](#網路不穩定的應對機制)
4. [網路斷線的應對機制](#網路斷線的應對機制)
5. [使用者體驗回饋](#使用者體驗回饋)
6. [逾時與重試參數一覽](#逾時與重試參數一覽)
7. [改善建議](#改善建議)

---

## 在線語音架構概述

xiaozhi-esp32 採用 **雲端語音辨識 + LLM + TTS** 的全線上串流架構，裝置端僅保留喚醒詞偵測（離線）：

```
麥克風 → AFE (音頻前端處理)
        → VAD (語音活動偵測)
        → OPUS 編碼 (16 kHz / 60 ms 幀)
        → 傳輸協定 (WebSocket 或 MQTT+UDP)
        → 雲端 STT → LLM → TTS
        ← OPUS 解碼
        ← 喇叭播放
```

裝置狀態機（`DeviceStateMachine`）定義五個核心狀態：

```
IDLE → [喚醒詞] → CONNECTING → LISTENING → SPEAKING → IDLE
```

網路問題發生在 `CONNECTING`、`LISTENING`、`SPEAKING` 三個狀態時影響最大。

---

## 通訊協定

專案支援兩種協定，依 NVS 設定自動選擇：

### WebSocket（`/main/protocols/websocket_protocol.cc`）

- **連線建立**：向伺服器傳送 `hello` 訊息，等待回應（10 秒逾時）。
- **音訊格式**：OPUS，封裝於版本化二進位幀（v1/v2/v3）。
- **文字訊息**：JSON（`tts`、`stt`、`llm`、`listen`、`goodbye`、`alert` 等）。
- **發送緩衝**：40 個 OPUS 封包（約 2.4 秒）。

### MQTT + UDP（`/main/protocols/mqtt_protocol.cc`）

- **控制通道**：MQTT over TLS，用於 JSON 訊息交換。
- **音訊通道**：UDP，使用 AES-256-CTR 加密。
- **保活機制**：MQTT keepalive 預設 240 秒（可設定）。
- **重連間隔**：60 秒後自動重連（僅在 IDLE 狀態觸發）。

---

## 網路不穩定的應對機制

### 1. 握手逾時保護

兩種協定在建立連線後，均等待伺服器回傳 `hello` 訊息。若 **10 秒內** 未收到回應，視為連線失敗：

```cpp
// websocket_protocol.cc / mqtt_protocol.cc
EventBits_t bits = xEventGroupWaitBits(
    event_group_handle_, SERVER_HELLO_EVENT,
    pdTRUE, pdFALSE, pdMS_TO_TICKS(10000)  // 10 秒
);
if (!(bits & SERVER_HELLO_EVENT)) {
    SetError(Lang::Strings::SERVER_TIMEOUT);  // "等待响应超时"
    return false;
}
```

觸發後裝置回到 IDLE 狀態，並顯示錯誤提示，避免永久阻塞。

### 2. 通道閒置逾時（2 分鐘）

`Protocol::IsTimeout()`（`/main/protocols/protocol.cc`）持續追蹤最後一次接收資料的時間。若超過 **120 秒**未收到任何資料，視為通道失效並強制關閉：

```cpp
bool Protocol::IsTimeout() const {
    const int kTimeoutSeconds = 120;
    auto duration = std::chrono::duration_cast<std::chrono::seconds>(
        std::chrono::steady_clock::now() - last_incoming_time_
    );
    if (duration.count() > kTimeoutSeconds) {
        ESP_LOGE(TAG, "Channel timeout %ld seconds", (long)duration.count());
        return true;
    }
    return false;
}
```

此機制防止裝置在網路品質極差（封包大量遺失）時卡在通話狀態。

### 3. OTA 版本檢查指數退避重試

在啟動流程中，OTA 版本檢查採用 **指數退避（Exponential Backoff）** 策略（`/main/application.cc`），最多重試 10 次，避免在網路不穩時大量衝擊伺服器：

```cpp
const int MAX_RETRY = 10;
int retry_delay = 10;  // 秒
while (true) {
    esp_err_t err = ota_->CheckVersion();
    if (err != ESP_OK) {
        retry_count++;
        if (retry_count >= MAX_RETRY) {
            ESP_LOGE(TAG, "Too many retries, exit version check");
            return;
        }
        retry_delay *= 2;  // 每次失敗後等待時間加倍
        for (int i = 0; i < retry_delay; i++) {
            vTaskDelay(pdMS_TO_TICKS(1000));
            if (GetDeviceState() == kDeviceStateIdle) break;  // 使用者操作可提前中止
        }
        continue;
    }
    retry_count = 0;
    retry_delay = 10;  // 成功後重置
}
```

### 4. MQTT Keepalive 心跳

MQTT 協定透過 keepalive（預設 240 秒）定期向 Broker 傳送 PING，提早偵測連線異常，避免等到通道逾時才發現斷線。

---

## 網路斷線的應對機制

### 1. 立即偵測並優雅關閉通道

當作業系統底層偵測到網路斷線事件，`HandleNetworkDisconnectedEvent()`（`/main/application.cc`）會立即被呼叫：

```cpp
void Application::HandleNetworkDisconnectedEvent() {
    auto state = GetDeviceState();
    if (state == kDeviceStateConnecting ||
        state == kDeviceStateListening ||
        state == kDeviceStateSpeaking) {
        ESP_LOGI(TAG, "Closing audio channel due to network disconnection");
        protocol_->CloseAudioChannel();  // 優雅關閉，釋放資源
    }
    // 立即更新狀態列，告知使用者已斷線
    Board::GetInstance().GetDisplay()->UpdateStatusBar(true);
}
```

此設計確保：
- 裝置不會停留在等待狀態
- 網路資源被正確釋放
- 使用者即時看到連線狀態變化

### 2. 狀態機回到 IDLE

通道關閉後，錯誤事件（`MAIN_EVENT_ERROR`）經由事件佇列通知主迴圈，裝置回到 **IDLE 狀態**，等待使用者再次呼喚或重新連線。

```
CloseAudioChannel() → on_disconnected_() → Schedule(MAIN_EVENT_ERROR)
→ 主迴圈 → Alert() 顯示錯誤 → 回到 IDLE
```

### 3. MQTT 自動重連（60 秒）

MQTT 協定斷線後會自動啟動重連計時器，**60 秒後**嘗試重新連線到 Broker：

```cpp
// mqtt_protocol.cc
mqtt_->OnDisconnected([this]() {
    if (on_disconnected_ != nullptr) on_disconnected_();
    // 60 秒後重連，避免立即衝擊伺服器
    esp_timer_start_once(reconnect_timer_, MQTT_RECONNECT_INTERVAL_MS * 1000);
});

// 重連前確認裝置處於 IDLE，避免打斷使用者操作
reconnect_timer_callback = [](void* arg) {
    if (Application::GetInstance().GetDeviceState() == kDeviceStateIdle) {
        protocol->StartMqttClient(false);
    }
};
```

**延遲重連** 的設計避免了斷線後立即大量重試造成的「重連風暴」。

### 4. 協定優先選擇邏輯

```cpp
// application.cc
if (ota_->HasMqttConfig()) {
    protocol_ = std::make_unique<MqttProtocol>();
} else if (ota_->HasWebsocketConfig()) {
    protocol_ = std::make_unique<WebsocketProtocol>();
} else {
    protocol_ = std::make_unique<MqttProtocol>();  // 預設
}
```

系統依據 NVS 中的設定自動選擇最適合的協定，不需使用者手動切換。

---

## 使用者體驗回饋

### 1. 顯示器通知

裝置透過 `Alert()` 函式（`/main/application.cc`）同步更新顯示器、表情圖示和聊天訊息：

```cpp
void Application::Alert(const char* status, const char* message,
                        const char* emotion, const std::string_view& sound) {
    display->SetStatus(status);           // 頂部狀態列文字
    display->SetEmotion(emotion);         // 表情圖示（circle_xmark / triangle_exclamation）
    display->SetChatMessage("system", message);  // 聊天區說明文字
    if (!sound.empty()) audio_service_.PlaySound(sound);  // 音效提示
}
```

| 網路事件 | 顯示文字（中文）| 表情 | 音效 |
|----------|----------------|------|------|
| 無法連線伺服器 | 无法连接服务，请稍后再试 | `circle_xmark` | 嗶聲 |
| 等待回應逾時 | 等待响应超时 | `circle_xmark` | 嗶聲 |
| 伺服器錯誤 | 发送失败，请检查网络 | `circle_xmark` | 嗶聲 |
| 找不到伺服器設定 | 正在寻找可用服务 | `circle_xmark` | 嗶聲 |
| SIM 卡未插入 | 请插入 SIM 卡 | `triangle_exclamation` | SIM 錯誤音 |
| 行動網路無法存取 | 无法接入网络，请检查流量卡状态 | `triangle_exclamation` | 網路錯誤音 |

### 2. WiFi 連線過程通知

在 WiFi 掃描、連線、成功等各階段均有對應的顯示更新（`/main/application.cc:101-157`）：

| 事件 | 顯示文字 | 持續時間 |
|------|---------|---------|
| 掃描 WiFi | 扫描 WiFi... | 30 秒 |
| 連線中 | 连接 \<SSID\>... | 30 秒 |
| 連線成功 | 已连接 \<SSID\> | 30 秒 |
| 斷線 | （狀態列立即更新）| 即時 |
| 登入伺服器 | 登录服务器... | 即時 |

### 3. LED 視覺回饋

LED 依裝置狀態顯示不同動畫，讓使用者不看螢幕也能得知目前狀況：

| 狀態 | LED 行為 |
|------|---------|
| IDLE（待機）| 微弱常亮 |
| CONNECTING（連線中）| 脈衝動畫 |
| LISTENING（聆聽中）| 明亮常亮 |
| SPEAKING（回應中）| 藍色常亮 |
| ERROR（錯誤）| 紅色閃爍 |

### 4. 狀態列持續顯示

狀態列每秒更新一次，持續顯示：
- WiFi / 行動網路圖示與訊號強度
- 電池電量
- 時間（NTP 同步後）

當網路斷線時，狀態列的網路圖示立即切換為斷線樣式，給予使用者即時視覺提示。

---

## 逾時與重試參數一覽

| 參數 | 預設值 | 定義位置 | 說明 |
|------|--------|----------|------|
| 握手逾時（Hello Wait）| 10 秒 | `websocket_protocol.cc`、`mqtt_protocol.cc` | 等待伺服器 hello 回應 |
| 通道閒置逾時 | 120 秒 | `protocol.cc` | 未收到任何資料後強制關閉 |
| MQTT 重連間隔 | 60 秒 | `mqtt_protocol.h:22` | 斷線後等待重連的時間 |
| MQTT Keepalive | 240 秒 | `mqtt_protocol.cc`（可設定）| 心跳包間隔 |
| OTA 首次重試延遲 | 10 秒 | `application.cc` | 指數退避起始值 |
| OTA 最大重試次數 | 10 次 | `application.cc` | 超過後放棄重試 |
| WiFi 掃描顯示時間 | 30 秒 | `application.cc` | 顯示器保持「掃描 WiFi」文字的時間 |

---

## 改善建議

以下為根據現有架構分析所整理的潛在優化方向，供參考：

### 1. 語音通道主動重試

目前語音通道在斷線後不會自動重試，需等待使用者再次喚醒。可考慮在 **短暫斷線後**（如 1-3 秒內重新連線）自動恢復通話，提升使用者體驗的連貫性。

### 2. WebSocket 協定加入重連機制

MQTT 協定有 60 秒重連機制，但 WebSocket 協定目前無對等設計。可考慮加入類似邏輯，使兩種協定的韌性一致。

### 3. 弱訊號時降低音訊品質

目前 OPUS 固定使用 16 kHz 編碼。在偵測到網路延遲或封包遺失增加時，可嘗試動態降低位元率，以維持通話的連續性。

### 4. 網路品質視覺化

狀態列目前顯示 WiFi 訊號強度，可進一步根據實際延遲（RTT）動態更新圖示，讓使用者更清楚瞭解當前語音品質。

### 5. 本機快取喚醒詞回覆

在網路不穩定時，可播放本機預錄的「正在連線，請稍候」語音，避免使用者認為裝置未響應而反覆呼喚。

---

## 相關原始碼檔案

| 檔案 | 說明 |
|------|------|
| `/main/application.cc` | 主狀態機、網路事件處理、Alert() 函式 |
| `/main/protocols/protocol.cc` | 通道逾時偵測（IsTimeout）|
| `/main/protocols/websocket_protocol.cc` | WebSocket 連線與握手逾時 |
| `/main/protocols/mqtt_protocol.cc` | MQTT 連線、自動重連、握手逾時 |
| `/main/assets/locales/zh-CN/language.json` | 中文錯誤訊息字串 |
| `/main/device_state_machine.cc` | 裝置狀態轉換規則 |
| `/docs/websocket.md` | WebSocket 協定規格文件 |
| `/docs/mqtt-udp.md` | MQTT+UDP 協定規格文件 |
