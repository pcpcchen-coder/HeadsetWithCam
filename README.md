# HeadsetWithCam

**E900 攝影耳機 × QRing R08 智慧戒指 × iPhone 中繼 App × Mac mini AI Agent**

本專案旨在建立可自行擴充的穿戴式攝影控制系統：以智慧戒指輸入手勢，由自製 iPhone App 接收 BLE 事件，向攝影耳機下達拍照／錄影命令，再將媒體檔案同步至 Mac mini 供 AI Agent 進行分析。

> **截至 2026-10-15：Research / Pre-purchase 階段。** 本 Repo 保存技術調查、產品線索及驗證計畫；**尚未取得 E900 實機並驗證 BLE 控制、錄影或 Wi-Fi API 相容性**。不要將 E09 眼鏡協定視為 E900 已支援的介面。

## 預計操作流程

1. 在 **QRing R08** 點擊或雙擊。
2. **iPhone RingCamBridge** 透過 CoreBluetooth 解析手勢，執行命令映射。
3. Bridge 透過經過實機驗證的 **E900 BLE Camera Adapter** 控制拍照／開始或停止錄影。
4. E900 在裝置上存檔；需要時透過 Wi-Fi 將照片與影片同步至 iPhone。
5. iPhone 使用網路 API 將媒體交給 **Mac mini AI Agent**（場景理解、OCR、摘要、事件標記等）。

```text
QRing R08 --BLE notify--> iPhone RingCamBridge --BLE command?--> E900
                               |
                               +<-- Wi-Fi media sync? -------- E900
                               |
                               +---- HTTPS/LAN upload --------> Mac mini AI Agent
```

問號表示尚待 E900 實機測試。

## 研究與交接文件

| 文件 | 用途 |
|---|---|
| [E900 購買與硬體調查](docs/E900_RESEARCH_2026-10-14.md) | 台灣賣場、OEM 交叉比對、開放介面與未確認事項 |
| [R08 / E09 協定線索](docs/PROTOCOL_RESEARCH.md) | BLE UUID、社群逆向指令、風險及授權提示 |
| [系統架構與驗證計畫](docs/ARCHITECTURE_AND_TEST_PLAN.md) | iOS Bridge 設計、階段性驗收、失敗判準 |
| [AI Agent 接手指引](docs/AGENT_HANDOFF.md) | 下一位 Coding Agent 的開發原則、交付格式與 Prompt |

## 最優先待確認

- [ ] 收到的產品實際型號、硬體批次、韌體版本，以及實機是否使用賣家確認的 GlassX App（賣家於 2026-10-15 回覆 GlassX；實機尚未驗證）。
- [ ] R08 的 Service / Characteristic 與觸控事件是否符合公開的 Halo-Ring 協定。
- [ ] E900 的 BLE GATT 與 E09 相似程度；是否能真正「外部命令開始／停止錄影」。
- [ ] E900 影片是否可經可公開描述的 Wi-Fi API 取得（不能依賴原廠 App 自動化的假設）。
- [ ] iPhone 鎖屏背景下的可靠性、斷線復原、權限與電量。

## 原則

- 把 **公開參考**、**供應商宣稱**、**推論**、**本機實測** 明確區分。
- 設備操作不得將 BLE「寫入成功」當作「已開始錄影」；需另行讀取狀態或確認實際影片。
- 初期只做唯讀探測、必要且可復原的控制指令；**禁止未核准的 OTA、格式化、刪除媒體、出廠重置**。
- 上游專案有不同授權，**參考互通事實不等於可直接複製受授權保護的程式碼**，詳見協定研究文件。
- 攝影錄音遵守當地法律並尊重他人隱私；使用可見的拍攝指示與明確的啟停操作。

研究歸檔日期：**2026-10-14（Asia/Taipei）**；賣家 App 回覆更新：**2026-10-15**。