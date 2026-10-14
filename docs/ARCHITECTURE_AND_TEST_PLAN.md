# 系統架構與實機驗證計畫

- 更新日期：2026-10-14
- 目標：**QRing R08 觸控 → iPhone Bridge → E900 開始／停止錄影 → 影片經手機同步到 Mac mini AI Agent**
- 狀態：概念與工程驗證方案；**尚未建置 App，也沒有實機測試結果**。

## 1. 系統邊界

```text
 [QRing R08]                         [E900 camera earphone]
       | BLE GATT notifications               ^  | local media
       v                                      |  v
 [R08Driver]                              [E900Driver] ----> [WiFiMediaImporter]
       | gesture events                        ^                  |
       v                                       | BLE commands?    |
 [GestureRouter] ---> [RecordingStateMachine]-+                  |
             \                               iPhone              v
              +--- events/metadata ------------------------> [UploadQueue]
                                                               |
                                                               v
                                                Mac mini MediaIngestAPI
                                                               |
                                                               v
                                                   Vision / OCR / Summary
```

核心設計：讓手機是雙 BLE Central 與命令路由器；E900 本機錄影，媒體非同步匯入。**E900 BLE 與 Wi-Fi 是否開放要用實測決定**，在此之前 camera driver 只提供 Mock 行為。

## 2. iOS App 模組

| 元件 | 職責 | 不能省略的細節 |
|---|---|---|
| `R08Driver` | 連線、版本讀取、觸控模式初始化、Notify 解析 | BLE callback 和設備連線狀態分離 |
| `E900Driver` | 協定辨識、拍照／開始／停止錄影、ACK | 初期以 `MockCameraDriver` 測試，真實命令必須由實機驗證啟用 |
| `BLECoordinator` | 雙設備 lifecycle、斷線重連 | 避免與 QRing／GlassX 原廠 App 搶連線 |
| `GestureRouter` | Tap、雙擊／長按、去抖與映射 | 單擊等候雙擊時間窗；避免多次啟動 |
| `RecordingStateMachine` | 裝置狀態、轉換、錯誤回復 | 實際落盤才算成功 |
| `WiFiMediaImporter` | 列檔、下載、恢復傳輸 | E900 API 未明，先做抽象介面 |
| `UploadQueue` | 將媒體交給 Mac mini | checksum、時間戳、重試、授權、去重 |
| `Diagnostics` | GATT、手勢、狀態、延遲、錄影證據 | 預設遮蔽個資／序號／MAC／影像 |

建議技術：Swift + SwiftUI + CoreBluetooth；媒體傳輸使用 `URLSession` 或所需之經驗證 transport；Mac mini 上提供 HTTPS/LAN ingestion API。

**iOS 限制**：原廠 GlassX／EYEVUE 未確認有可呼叫的 App Intent 或 URL Scheme，不能預設可由自製 App 直接操控；必須以自製 BLE driver 控制裝置（在協定相容的條件下）。iPhone 背景藍牙可採 `bluetooth-central` 與 CoreBluetooth state preservation/restoration，但並不保證每次低延遲觸發或被使用者強制關閉後自動恢復。鎖屏、系統終止與手動強制關閉要分別驗證。

Apple 參考：[CoreBluetooth](https://developer.apple.com/documentation/corebluetooth)、[iOS Background Processing](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html)。

## 3. 建議手勢（**自訂需求，非原廠功能**）

| R08 手勢 | Bridge 動作 | 說明 |
|---|---|---|
| 單擊 | 拍照 | 必須延後到雙擊判斷窗結束 |
| 雙擊 | 錄影開始／停止切換 | 非冪等的 toggle 僅能依可信設備狀態決定 |
| 長按 | 加入事件標記 | 不等於直接觸發錄影；可配置 |
| 上滑 | AI 分析標記／請求 | 若只儲存在 E900，待媒體匯入才能分析 |
| 下滑 | 顯示狀態／變更模式 | 初期優先做狀態查詢 |

## 4. 錄影狀態機

```text
Disconnected -> Idle -> Starting -> Recording -> Stopping -> Saved
       |          |        |           |            |
       +----------+--------+-----------+------------+--> Error / Unknown
```

### 不可違反的規則

1. `Starting` 期間忽略或合併重複 Start，保留 command correlation ID。
2. ACK 只表示命令已被接受（若有 ACK），**不得自動宣告錄影成功**。
3. `Recording` 需有裝置回報、指示燈觀察或可校驗影片證據支持；未知時呈現 Unknown，不隱藏不確定性。
4. `Stopping` 要等待裝置停止與存檔完成或回傳可查證的媒體，不能只在 UI 翻成 Idle。
5. 切換網路／BLE 斷線時維護期望狀態與已知實際狀態的差異，不要盲目重送 toggle。
6. 媒體檔案帶唯一 ID 與 SHA-256，以避免重複上傳及檔案截斷。
7. 錄影開始／停止與失敗都要有清楚 UI 或可辨識提示；不可做隱蔽偷拍功能。

## 5. 開發里程碑與驗收標準

| Gate | 要求 | 驗收證據 |
|---|---|---|
| G0 選對硬體 | 型號、App、版本、選項、Wi-Fi、電池與相機正常 | 商品頁存證、包裝／App 截圖、基線影片 |
| G1 R08 可收事件 | 正確觸控初始化、4 種基本事件可分辨 | 原始 byte 與解析後事件對照、每種 ≥20 次 |
| G2 E900 可控 | 可由 Mac／自製 App 發出 Start/Stop 並真正生成影片 | 連續 20 次起停，檔案時長／大小核對 |
| G3 iPhone 前景串接 | R08 雙擊控制 E900、避免誤觸重複事件 | 前景 ≥100 次、成功率 ≥99%（定義為存在正確影片） |
| G4 iPhone 鎖屏 | 鎖屏保持接收／下達命令、恢復機制 | 100 次鎖屏測試；記錄成功率與 P50/P95 延遲，未達標不可宣稱可靠 |
| G5 媒體匯入 | App 可取回原始檔案並驗證完整性 | 20 個檔案校驗、斷線／復原及重複匯入 |
| G6 AI Agent | 影片可同步 Mac mini，AI 分析結果可查原始片段 | 可追溯 ID、上傳重試、時間戳、OCR／摘要輸出 |

**若 G2 失敗**：先確認原廠 App 是否支援遠端控制，再比較初始化、加密／配對與 GATT。不要以 E09 相同 SoC 為理由繼續硬編固定指令；應停在協定研究或選擇其它具明確 SDK 的攝影設備。

## 6. 風險與替代路線

| 風險 | 實際影響 | 替代／緩解 |
|---|---|---|
| E900 韌體封閉、不同 E09 | 無法從外部命令錄影 | `CameraDriver` 抽象化，更換硬體；或僅用 E900 實體鍵 |
| R08 與官方 App BLE 連線競爭 | 觸控事件不穩定 | 前景測試時先關閉官方 App，建立專用連線 |
| iOS 背景限制 | 鎖屏觸發延遲、錯失事件 | 背景權限、連線保活策略和實機測試；不可承諾 always-on |
| E900 熱點切換造成 iPhone 網路中斷 | 無法同步雲端／Mac | 媒體先存 iPhone、排隊重試 |
| 電力和儲存限制 | 分段停止、檔案遺失 | 檔案完整性檢查與電量監測 |
| 相機錄音涉及隱私 | 他人未同意錄製 | 明確的拍攝 UI／指示，使用者決策及資料安全 |

## 7. 測試紀錄結構

```text
docs/evidence/YYYY-MM-DD/
  device_inventory.md      # firmware / App / serial redacted
  r08_gatt.json             # UUIDs only (redact identifiers)
  e900_gatt.json
  gesture_cases.csv         # timestamp, raw code, decoded event
  camera_cases.csv          # request, ack, actual file, latency, result
  media_checksums.txt
  test_summary.md           # PASS / FAIL / NOT TESTED and blockers
```

測試 Case 必須記錄 `NOT TESTED` 與已知限制；「需求－測試－輸出證據」要能互相追溯，不能只寫 Agent 自評 PASS。