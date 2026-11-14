<!-- 產品／專案：HEShell（神盾殼／AEGIS FHE AI Wrapper with USB）｜法人／事業體：AEGIS（`aegis.expert`；正式法人名稱 [待確認]）｜日期：2026-11-14（Asia/Taipei）｜草稿 v0.4.8 -->
# HEShell
C2-AE-FAW 神盾殼 HEShell USB 硬體金鑰 + HE 防護 AI 介面殼層 for 資安敏感企業、法律與金融

# HEShell（AEGIS FHE AI Wrapper with USB）

> **倉儲識別碼**：`C3-AE-FAW-F`（`aegis-fhe-ai-wrapper-app`）  
> **所屬法人／事業體**：AEGIS（`aegis.expert`）｜正式公司登記名稱 [待確認]
> **系統定位**：具備 USB 硬體金鑰邊界之同態加密（HE）AI 路由殼層  
> **維護團隊**：`team-aegis-fhe-ai-wrapper`（`dev-aegis-fhe-ai-wrapper-app@aegis.expert`）  
> **文件狀態**：草稿 v0.4.8，待人工審核

## 1. 這是什麼（以及不是什麼）

HEShell 是客戶端安全殼層與外部模型路由中繼：

1. 作為本機前端介面接收使用者輸入。
2. 自 HE Provider 取回一把加密私鑰，私鑰於本機當場組合為複合金鑰（Combined Key）後存放於專用 USB；HE Provider 不能接觸到這把私鑰（主 PMO 2026-11-13 答覆）。
3. 於本機加密：先將金鑰組合成當前最新的加密金鑰，再以其加密使用者輸入；明文不送往外部 HE Provider（主 PMO 2026-11-13 答覆）。見 §6 B。
4. 將密文與評估金鑰（evk / Galois / Relin Keys）送往已宣告密文能力之端點。
5. 接收結果並顯示；**結果以明文回傳**（主 PMO 2026-11-13 補充答覆）。明文從不送往 HE Provider；HE Provider 不能接觸到這把私鑰。

HEShell **不訓練、不微調**基礎語言模型，**不實作**底層同態加密密碼學基元。

密文智庫（EncryMind）將密文回傳至地端 SLM／LLM，為不同產品。

斜槓一二三 v1 將整合 HEShell；整合方式與時程 [待確認]。

對外用語（使用者 2026-11-14 14:56 選定 2A）：「HEShell 會從 HE Provider 取回一把加密過的私鑰，在您的電腦上當場組合成『複合金鑰』，存放在您手上的 USB 裡；HE Provider 不能接觸到這把私鑰，您的提問明文也不會送往 HE Provider。」[待確認] 是否需搭配揭露句「結果以明文回傳」由使用者決定。複合金鑰流程尚有 9 項細節（解鎖方式、組件來源、傳輸對象等）仍 [待確認]，本次不結案、不自行補寫。

> 與密文智庫文件之關係：HEShell 為已與該文件「S0 單向輸入加密架構（Legacy）」不同的改版，產品繼續預售；S0「嚴格廢棄」狀態不解除（主 PMO 2026-11-13 補充答覆）。HEShell 採 L0–L3 分級，不以 S0–S3 評級；S0 指架構類型，不指特定產品（主 PMO 2026-11-13 決定，方案 B）。

對外用語（使用者 2026-11-14 14:56 選定 1B）：「S0 單向輸入加密架構（Legacy）是密文智庫（EncryMind）文件裡的一種架構類型，不是指特定產品，這個架構維持『嚴格廢棄』。HEShell 是已與 S0 不同的改版，採用自己的 L0–L3 分級，不以 S0–S3 評級。」

## 2. 能力分級（禁止混稱）

| 級別 | 定義 | 傳輸態 | 計算態 | v1 狀態 |
| :--- | :--- | :--- | :--- | :--- |
| L0 | 傳輸層加密（TLS 1.3 / HTTPS） | 密文 | 伺服器端明文 | **非本產品**，不列為隱私賣點 |
| L1 | 指定 HE API 密文運算 | 密文 | 純密文（固定電路） | **本期交付**：Embedding 比對、距離計算、數值評分與分類 |
| L2 | 端點密文模型推論 | 密文 | 純密文推論並回傳密文 | **條件交付**：僅限具明確能力宣告之端點 |
| L3 | 任意閉源 API 密文即時生成 | 密文 | 任意商用 LLM 直接解讀密文 | **非現行科技**：嚴禁承諾，不列入 v1 驗收與 Roadmap |

> 註：HEShell **結果以明文回傳**。上表 L2「純密文推論並回傳密文」為端點能力描述（README 原設計），對使用者呈現之結果為明文。此列表述是否改寫 [待確認]。密文智庫（EncryMind）將密文回傳至地端 SLM／LLM，為不同產品。

## 3. 延遲預期與介面規範

研究引用（非本產品 SLA；出處 [待確認]）：README 原稿依「2026 年公開密碼學與硬體加速研究」記載，以 CKKS 或 TFHE/FHEW 對約 8B 參數模型做端到端密文推論，相較明文約有 10^4 量級運算代價；短序列（≤ 64 tokens）延遲約數十秒至數分鐘；單一上下文狀態向量可能由數十 KB 膨脹至數十至數百 MB。

介面規範：

- **嚴禁虛假的逐字打字機串流（Token Streaming）**。
- 必須提供階段指示器：金鑰封裝中 → 密文傳輸中 → 密文同態運算中 → 接收結果，並明示批次等待時間。（原設計為「…密文下載中 → 本機解密驗證」；結果以明文回傳，階段名稱是否再調整 [待確認]）

## 4. 模組架構

    heshell-core/
    ├── apps/
    │   └── desktop-ui/          # 桌面殼層（Tauri/Rust + 安全渲染）
    └── packages/
        ├── usb-sentinel/        # USB 生命週期監控與硬體金鑰介面
        ├── he-client-sdk/       # 外部 HE API 消費端（抽象介面）
        ├── zero-leak-router/    # 密文端點路由決策器（Capability Matcher）
        ├── secure-mem/          # mlock 防交換、敏感字串即時覆寫
        └── sanitized-logger/    # 零明文稽核日誌與崩潰處理

| 模組 | 權責 | 嚴格禁止 |
| :--- | :--- | :--- |
| `usb-sentinel` | USB 拔插偵測、挑戰應答、Session Key 配對；存放複合金鑰（Combined Key） | 私鑰導出至未保護 Heap；雲端備份 |
| `secure-mem` | 明文展示緩衝區 mlock；拔除時 memset_s | 明文寫入 Pagefile/Swap；延遲釋放 |
| `he-client-sdk` | 封裝 HE API；自 HE Provider 取回加密私鑰，於本機進行加密（不送明文至 HE Provider） | 自製密碼原語；失敗時降級明文；送出明文至 HE Provider |
| `zero-leak-router` | 檢查端點 Capability Manifest；非 L1/L2 阻斷 | 對未宣告能力端點送封包；明文後備 |
| `sanitized-logger` | 記錄延遲、密文位元組數、錯誤碼、Key ID Hash | Prompt、回覆原文、私鑰寫入 Log 或 Crash Dump |

## 5. USB 狀態機

    DISCONNECTED → ATTACHED_IDLE → SESSION_ACTIVE → DECRYPTING → DISPLAYING_PLAINTEXT
          ▲                                                               │
          └──────────────────── 任意狀態：USB 拔除 ─────────────────────────┘

（結果以明文回傳；原設計狀態名 `DECRYPTING` 是否調整 [待確認]。）

USB 拔除（緊急阻斷）：

1. 立即中止網路傳輸與解密運算
2. GUI 以空白填滿，抹除畫面明文
3. 對明文、對話快取、中介變數緩衝區零值抹寫（Zeroize / memset_s）
4. 銷毀通往該 USB 的 Context Handle
5. 禁止自磁碟快取或暫存區還原明文

工程門檻：中斷信號延遲與白屏／抹寫完成 ≤ 50 ms（驗收門檻，對外承諾 [待確認]）。

金鑰位置：依主 PMO 2026-11-13 答覆，「HE Provider 是取用一把加密私鑰，透過在本機加密之後，然後產生的私鑰組合再放到 USB 裡，所以其實是 Combined Key 的概念。」補充答覆：私鑰於本機當場組合為複合金鑰後存放於 USB；HE Provider 不能接觸到這把私鑰。即私鑰非由 USB 生成。不經上傳、不做雲端備份：對外傳輸的是複合金鑰的組件，而非使用者 AI 提問的文字。USB 位於虛擬機的 USB 埠；使用者可自行備份 USB，但遺失不補發。不做雲端備份。

## 6. HE API 介面契約

殼層不自行實作密碼原語，依下列契約消費外部 HE 服務或本機 HE 模組（欄位值皆為示例）：

- A. `POST /v1/he/session/init`：建立會話（scheme、params、client_id）；回傳 session_id、public_key_ref、所需評估金鑰。自 HE Provider 取回加密私鑰是否經此 API 及其欄位 [待確認]。
- B. 本機加密：自 HE Provider 取回一把加密私鑰，私鑰於本機當場組合為複合金鑰（Combined Key）後存放於 USB；先將金鑰組合成當前最新的加密金鑰，再以其加密使用者輸入；明文不送往外部 HE Provider；HE Provider 不能接觸到這把私鑰（主 PMO 2026-11-13 答覆）。
- C. `POST /v1/he/inference/dispatch`：送出密文、評估金鑰、`target_model_capability`、`max_computation_depth`；回傳 task_id 與 `estimated_latency_sec`（範例值，非承諾）。
- D. `GET /v1/he/inference/result/{task_id}`：取回密文結果、consumed_depth、proof_of_execution。
- E. 本機解密：README 原設計為僅 USB 在位時執行、USB 離線則硬體拒絕並拋錯；依主 PMO 2026-11-13 補充答覆結果以明文回傳，本步驟及 D 之「取回密文結果」如何調整 [待確認]。

錯誤碼：

| 錯誤碼 | 常數 | 行為 |
| :--- | :--- | :--- |
| HE_001 | ERR_USB_MISSING | 停止連線，清空工作區 |
| HE_002 | ERR_CAPABILITY_UNSUPPORTED | 主動熔斷，嚴禁降級送出明文 |
| HE_003 | ERR_PLAINTEXT_LEAK_GUARD | 輸出管道含非密文資料，硬中斷 |
| HE_004 | ERR_SESSION_MISMATCH | Session Key ID 不符，拒絕解密 |
| HE_005 | ERR_CIPHERTEXT_CORRUPTED | 雜湊錯誤或噪聲預算耗盡，推論失敗 |
| HE_006 | ERR_USB_PREMATURE_EJECT | 途中拔除，全域記憶體洗刷 |

## 7. 路由規則（FAIL-CLOSED）

1. **絕不自動降級**：端點不支援密文推論時，不提供「改以明文發送？」選項，請求直接無效。
2. **能力宣告白名單**：端點須提供經伺服器私鑰簽章之 Capability Manifest，載明支援方案（CKKS / BFV / TFHE）、最高乘法深度、密文 I/O 支援證明。
3. **明文審查閘門**：出境 Payload 若非標準密文結構或含自然語言明文字元，Socket 立即關閉。

## 8. 威脅模型摘要

可信任邊界：本機 USB 安全硬體、受 mlock 保護之 HEShell 核心程序。  
不可信任：宿主 OS 其他程序與 Swap、公共網路與代理、遠端推論伺服器、雲端日誌與計費端點。

| 編號 | 威脅 | 緩解 |
| :--- | :--- | :--- |
| TM-01 | 惡意伺服器竊聽 | 明文輸入不送往 HE Provider；送出之輸入為 HE 密文；結果以明文回傳；HE Provider 不能接觸到這把私鑰（主 PMO 2026-11-13 答覆）；伺服器端不持有解密金鑰（使用者 2026-11-13 22:20 指示已核准對外使用）。對外用語（使用者 2026-11-14 14:56 選定 3A）：「伺服器端不持有解密金鑰。」[待確認] 是否需搭配揭露句「結果以明文回傳」由使用者決定 |
| TM-02 | 拔除後殘留明文 | 拔除即 memset_s、白屏、無磁碟明文快取 |
| TM-03 | Swap/Pagefile 洩漏 | mlock / VirtualLock |
| TM-04 | 日誌與 Crash Dump 外洩 | 僅數值指標；封鎖 Full Memory Dump |
| TM-05 | 評估金鑰逆向 | 依 HE 標準參數（至少 128-bit 經典安全強度） |
| TM-06 | 中間人偽造能力宣告誘導降級 | 強制 FAIL-CLOSED，無明文回退路徑 |

## 9. 開發里程碑與驗收

里程碑（內部排程，非對外交期）：M1 硬體金鑰介面與安全記憶體 → M2 外部 HE API 整合 → M3 零洩漏路由器與 UI 狀態機 → M4 端到端密文推論整合與故障驗收。

本專案不以「串接任一公有 API 並秒回文字」作為驗收標準。驗收 Gate：

1. 金鑰邊界：拔除 USB 時解密回 HE_001，畫面與記憶體無明文；出境封包不含 sk。
2. 出境明文清零：封包捕獲不得出現任何自然語言片段。
3. 非相容端點中斷：路由至一般商用 LLM（L0）必須回 HE_002，嚴禁以明文降級成功。
4. 拔除白屏與抹寫：≤ 50 ms 白屏並 memset_s；Dump 檢查明文命中數為 0。
5. 日誌審計：執行紀錄、Crash Dump、路由追蹤檔明文出現率 0%。

## 10. 聯絡

- Dev：`dev-aegis-fhe-ai-wrapper-app@aegis.expert`
- 資安通報窗口 [待確認]

---

## 變更註記（v0.4.8，2026-11-14，Asia/Taipei）

依使用者 2026-11-14 14:56 選定 1B/2A/3A，將三句對外用語原文寫入本 README（S0 關係段、§1、§8 TM-01）。2A、3A 旁加註 [待確認] 是否需搭配揭露句「結果以明文回傳」由使用者決定，不自行加寫揭露句。複合金鑰流程 9 項細節仍 [待確認]，不結案。v0.4.7 僅改預售草稿，未改本 README。本文件為草稿，尚未最終確認。

## 變更註記（v0.4.6，2026-11-13，Asia/Taipei）

依使用者 2026-11-13 22:20 指示，依預售草稿現行事實改寫本 README：結果以明文回傳；明文不送往 HE Provider；HE Provider 不能接觸到私鑰；金鑰流程為複合金鑰（Combined Key），組件於本機組合成當前最新金鑰後再加密使用者輸入；提問不經上傳、不做雲端備份（對外傳輸的是複合金鑰組件，不是提問文字）；USB 位於虛擬機的 USB 埠，可自行備份，遺失不補發；密文智庫（EncryMind）為不同產品（密文回傳至地端 SLM／LLM）；斜槓一二三 v1 將整合 HEShell（方式與時程 [待確認]）。本文件為草稿，尚未最終確認。

### 人工審核清單

- 結果以明文回傳後，L2「回傳密文」、API D／E、狀態機 `DECRYPTING` 與介面階段名稱如何調整，仍 [待確認]，請勿視為已決。
- 取回之加密私鑰於組合時如何解鎖 [待確認]。
- 斜槓一二三 v1 整合 HEShell 的方式與時程 [待確認]。
- 50 ms 白屏門檻、資安通報窗口、正式法人名稱、對外中文正式名稱等既有 [待確認] 維持。
- 未提供之效能數字、價格、交期與合規不得視為已確認。
