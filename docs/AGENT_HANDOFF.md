# AI Coding Agent 接手指引（HeadsetWithCam）

日期：2026-10-14｜適用對象：Claude Code / Codex / 其他 Coding Agent

## 初始狀態與可信度

- 此 Repo 於 2026-10-14 以調查文件初始化，**沒有已實作／已測的 iOS App**。
- 真正目標：**R08 觸控 → iPhone 自製中繼 → E900 啟動／停止本機錄影 → 媒體下載 → Mac mini AI 分析**。
- 請先看 [E900 產品研究](E900_RESEARCH_2026-10-14.md)、[協定線索](PROTOCOL_RESEARCH.md) 與 [架構及驗收標準](ARCHITECTURE_AND_TEST_PLAN.md)。
- **E900 協定仍未知**。E09／EYEVUE 的公開逆向結果只是比較對象，不得當作 E900 真實 API。
- 不得虛構使用者已購入、配對成功、Swift 已編譯、前景／背景成功率等狀態。

## 開發順序（證據優先）

1. **Inventory**：先確認設備實際型號、原廠 App、R08／E900 GATT Service 與 FW/HW 版本，記錄證據。
2. **R08 decoder**：依 Halo-Ring 文件獨立實作 R08 packet parser 和 gesture recognizer，建立 byte fixture 與 unit tests；無硬體時以 Mock 驗證。
3. **Camera transport interface**：定義 `CameraDriver` protocol，實作 `MockCameraDriver`；對 E900 只提供安全的 BLE GATT 偵測程式，不可預設支援 E09 寫入指令。
4. **Verified command implementation**：只有在 E900 真機經協定及錄影檔驗證後，才為 camera adapter 啟用 live write path。
5. **SwiftUI RingCamBridge**：雙設備連線、狀態機、手勢設定與診斷頁；加上 CoreBluetooth background mode / state restoration，清楚標示可靠性限制。
6. **Media sync**：確認 E900 真正 Wi-Fi API 後才實作 Wi-Fi downloader；再加入 Mac mini ingest、checksum、重試與 AI 摘要。
7. **Acceptance evidence**：按照 G0–G6 的驗收關卡建立需求／測試／證據矩陣與可視化診斷看板。

## 建議專案結構（提案，尚未建立）

```text
HeadsetWithCam/
  README.md
  AGENTS.md
  docs/
    E900_RESEARCH_2026-10-14.md
    PROTOCOL_RESEARCH.md
    ARCHITECTURE_AND_TEST_PLAN.md
    AGENT_HANDOFF.md
    evidence/
  ios/
    RingCamBridge/
      App/
      Bluetooth/R08Driver.swift
      Bluetooth/E900Driver.swift
      Bluetooth/BLECoordinator.swift
      Gestures/GestureRouter.swift
      Camera/CameraDriver.swift
      Camera/MockCameraDriver.swift
      Camera/RecordingStateMachine.swift
      Media/WiFiMediaImporter.swift
      Diagnostics/
  macos/
    BLEDiagnostic/
  agent/
    MediaIngestAPI/
  tests/
    fixtures/
    unit/
    integration/
```

## 可以直接交給 Agent 的工作指令

> 在 `pcpcchen-coder/HeadsetWithCam` 專案中開發可驗證的 iOS 穿戴攝影控制橋接系統。閱讀 docs 既有研究記錄，不得把 E09 的逆向協定當成 E900 已驗證 API。先建立 SwiftUI + CoreBluetooth Xcode 專案與可獨立跑單元測試的 R08 手勢解析／雙擊時間窗／CameraDriver mock／錄影狀態機。新增診斷畫面，支援列舉 BLE Service / Characteristic、唯讀設備資訊及匿名化封包紀錄。以實機 GATT 比對與明確攝影證據作為啟用 E900 寫入指令的先決條件。加入背景 bluetooth-central 模式與 state restoration，但 UI 必須標示其非保證 always-on。每個操作區分「命令發送」「設備回覆」「實際檔案成功」。建立 tests、CI 可跑測試、驗收矩陣與 docs/evidence 範例，不得假報硬體測試。禁止刷寫韌體、刪除媒體、格式化、隱蔽錄影及未知指令暴力枚舉。最後提交變更摘要、測試輸出與剩餘阻礙。

## Agent 的 Definition of Done

- `xcodebuild` 等必要指令若可在執行環境測試，附上原始結果；若環境無 Xcode，標為 **NOT RUN**。
- 單元測試至少覆蓋：Tap/Double Tap 延遲；手勢去抖／事件重複；錄影狀態非法轉換；斷線重入；命令 ACK 但檔案不存在；媒體重試與 SHA-256 去重。
- 提供可執行測試命令，不能只寫測試名稱。
- 真機控制需明列 **設備型號／韌體／App 版本／原始 BLE 證據／錄影媒體證據**。
- 建立 `README` 或 docs 中的 `Status Matrix`，將 `REFERENCE`、`MOCK PASS`、`HARDWARE PASS`、`BLOCKED`、`NOT TESTED` 區分。
- 驗收儀表板上的指標與原始測試檔一致，不允許單靠 LLM 自評輸出綠燈。
- 不修改與本次任務無關的文件；不覆蓋測試證據或來源連結。

## 注意授權及隱私

- Halo-Ring 為 AGPLv3 / 雙授權；本案先自行實作互通所需事實，複用程式碼前必須確認授權適配性。
- AI 音視訊媒體要具備明確同意與錄製指示，日誌或公開 Repo 不包含私人音視訊、MAC、序號、個資、API key。
- 可研究裝置與自有資料；不作未授權旁聽、偷拍、韌體破壞或存取他人帳號。

## 當前最優先工作項目

**第一優先不是寫 AI 影像辨識，而是先證實 E900 能不能被第三方 BLE 客戶端啟動／停止錄影。** 若此條件無法滿足，停在研究階段並向專案負責人呈報替代機型，不要假裝功能可用。
