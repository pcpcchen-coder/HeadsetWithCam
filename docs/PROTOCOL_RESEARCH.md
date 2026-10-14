# R08 戒指及 E09 攝影設備協定研究（**E900 尚未驗證**）

研究日期：2026-10-14；證據來源：外部開源專案文件。**下列協定屬 R08／E09 參考機型，不等於 E900 的已測規格。**

## 證據代碼

- **REF-R08**：Halo-Ring 作者在一枚 R08 實機上的逆向與測試。
- **REF-E09**：EYEVUE E09 眼鏡的逆向／部分實機測試。
- **VENDOR-E900**：E900 供應商宣稱。
- **LOCAL-TBD**：本專案購買的 R08 / E900 尚未做本機重現。

## A. QRing R08 BLE

主要來源：
- [Halo-Ring 專案](https://github.com/MRziyi/Halo-Ring)
- [R08 完整 BLE 協定](https://github.com/MRziyi/Halo-Ring/blob/main/Doc/09-r08-ble-protocol-spec.md)
- 作者的測試韌體 **RT08_3.10.46_250621**、硬體 **RT08_V3.1**（不是本專案實機的已知版本）。

公開文件中的 GATT 結構：

| 用途 | UUID |
|---|---|
| Primary command service | `6e40fff0-b5a3-f393-e0a9-e50e24dcca9e` |
| Write | `6e400002-b5a3-f393-e0a9-e50e24dcca9e` |
| Notify | `6e400003-b5a3-f393-e0a9-e50e24dcca9e` |
| Firmware revision | 標準 Device Information `0x2A26` |
| Hardware revision | 標準 Device Information `0x2A27` |

公開文件列出 `73 2D XX ...` 手勢上報範例（`XX` 是手勢 code）：

| XX | 單次原子事件 |
|---|---|
| `01` | 上滑 |
| `02` | 下滑 |
| `03` | Tap |
| `04` | 長按 |

**重要陷阱：訂閱 Notify 不一定馬上收到手勢。** 參考文件指出需要 TouchControl `0x3B` 的兩階段初始化（TOUCH_ENABLE，再設置帶 `appType=9` 的 TOUCH_SLEEP_MODE）；在完成配對與讀取韌體後，才按照對應版本文件執行。不要直接把未加 checksum 的字節片段寫入裝置。

**雙擊不是上述四種原子事件中的獨立碼**：由 App 對 Tap 連續事件做有界時間窗聚合（例如初期可測 250–350 ms）。需處理單擊延遲確認、抖動、長按優先級與去重。

**晶片備註／訂正**：Halo-Ring 現行文件將測試戒指辨識為 *Realtek 8772-family（推論）*。過去討論曾提到另一種晶片名稱；不應把該資訊視為本案 R08 的確定晶片。以實際韌體、硬體及 BLE 特徵為準。

## B. E09／EYEVUE 攝影眼鏡協定（**參考，不可直接下達至 E900**）

主要來源：
- [EYEVUE E09 BLE 逆向協定主文件](https://github.com/sctg-development/ai-smart-glasses-e09/blob/main/REVERSE_ENGINEERING_EYEVUE_BLE_PROTOCOL.md)
- [E09 macOS BLE Probe](https://github.com/sctg-development/ai-smart-glasses-e09-reverse)
- [E09 眼鏡 OEM 規格](https://nafantech.com/8mp-smart-ai-glasses-e09/)

E09 文件中列出的 GATT：

| 用途 | UUID |
|---|---|
| BLE service | `0000aa12-0000-1000-8000-00805f9b34fb` |
| Command write | `0000aa13-0000-1000-8000-00805f9b34fb` |
| Command notification | `0000aa14-0000-1000-8000-00805f9b34fb` |
| Photo transfer notification | `0000aa15-0000-1000-8000-00805f9b34fb` |
| Standard battery service | `0x180F`（文件記載 E09 實機存在） |

命令 ID（**僅 E09**）：

| ID | 公開文件中的意義 | 本案 E900 狀態 |
|---|---|---|
| `0x22` | 拍照 | 未驗證 |
| `0x23` | 開始錄影 (`RECORD_VIDEO`) | 未驗證 |
| `0x24` | 停止錄影 (`STOP_RECORD`) | 未驗證 |
| `0x39` | Wi-Fi 媒體傳輸流程相關命令 | 未驗證 |
| `0x67` | 開啟 live preview 相關命令 | 未驗證 |

**命令 ID 不等於完整 BLE frame**：發送前須弄清楚 header、length、payload、checksum / CRC、初始化／配對、ACK 與狀態查詢。避免將任何命令 ID 直接作為單位元組送出。

E09 研究亦有以下媒體傳輸結果：
- BLE 縮圖下載可能只有低解析度（文件實測約 320×180），不等同原始照片。
- 原始照片／影片經 E09 眼鏡啟動的 Wi-Fi 熱點與 HTTP 機制取回。
- E09 實測 RTSP H.264 約 1600×1200；**不能聲稱 E900 也支援相同 URL 或串流**。
- E09 文獻的「0x23/0x24」是逆向 app 得到的指令語意，仍需核對真機錄影 ACK、實際影片持久化與停止邊界。

## C. JieLi SDK 可用性邊界

- [Jieli iOS-JL_Bluetooth](https://github.com/Jieli-Tech/iOS-JL_Bluetooth)
- [Jieli Android-JL_Bluetooth](https://github.com/Jieli-Tech/Android-JL_Bluetooth)
- [Jieli 文件中心](https://doc.zh-jieli.com/)

JieLi 提供 RCSP 音訊裝置藍牙 SDK；**只表示晶片廠有開發資源**，不表示 E900 商用品韌體有啟用 RCSP、自訂命令或攝影錄影功能。須先證實 GATT / 握手，才能判定可用性。

## D. 最小安全探測次序

1. 先在 Mac 上依公開專案的 README 建置診斷工具，僅做掃描、列舉 Service／Characteristic 和可讀 Device Info。
2. 記錄 advertised name、GATT UUID、版本號、可讀特徵與連線錯誤（共享截圖須遮蔽 MAC／序號／Token）。
3. 確認 R08 Notify 可訂閱，再依其實際硬體版本進行手勢初始化與被動事件捕獲。
4. 對 E900 只先做 **GATT 對照**；若符合 E09，仍必須先復現原廠 App 拍照／錄影與 BLE 封包特徵，再寫入最小控制命令。
5. 錄影驗證必須同時滿足「命令送出」「裝置確認」「實際檔案生成」；建立操作與結果的時間戳記。
6. 切換 Wi-Fi 前保留 BLE 協調狀態，驗證連線切換是否影響原有網路與 App。

**不要測試** OTA 刷寫、恢復原廠、清空存儲、讀取不屬於自己的資料，或對未知指令做暴力枚舉。

## E. 參考專案授權

- Halo-Ring README 聲明 **AGPLv3／商用雙授權**，協定文件及原始碼有授權聲明；若要直接引用其程式碼，必須事先釐清公開衍生作與對應原始碼等義務。
- E09 研究 README 標示 MIT；引用程式碼前仍應實際核對上游各子模組／第三方依賴的 LICENSE。
- 本 Repo 原則是記錄 **互通性事實、自行撰寫測試與驅動程式**；不直接複製逆向專案的大段原文或其受授權原始碼。

## F. 試驗記錄模板

```text
Date / Tester:
Device exact model / FW / App:
GATT evidence file:
Procedure:
Command sent (redacted, with timestamps):
Observed ACK/state:
Observed camera indicator:
Media file path, size, duration, codec:
Result: PASS / FAIL / INCONCLUSIVE
Known limitations:
```

本文件全部 E09 / R08 成果均屬**外部研究，不是 HeadsetWithCam 已完成的驗證**。