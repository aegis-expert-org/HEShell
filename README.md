# HEShell
C2-AE-FAW 神盾殼 HEShell USB 硬體金鑰 + HE 防護 AI 介面殼層 for 資安敏感企業、法律與金融
# HEShell (AEGIS FHE AI Wrapper with USB)

> **倉庫識別碼**：`C3-AE-FAW-F` (`aegis-fhe-ai-wrapper-app`)  
> **系統定位**：具備硬體金鑰邊界之同態加密（Homomorphic Encryption, HE）AI 路由殼層  
> **維護團隊**：`team-aegis-fhe-ai-wrapper` (`dev-aegis-fhe-ai-wrapper-app@aegis.expert`)

---

## 1. 產品定義與工程邊界

### 1.1 系統本質
HEShell 是一套**客戶端安全殼層（Secure Client-Side Shell）與外部模型路由中繼**，本身**不訓練、不微調基礎語言模型**，亦**不實作底層同態加密密碼學基元（Cryptographic Primitives）**。其功能在於：
1. 作為本機前端介面，接收使用者輸入。
2. 透過已掛載的專用 USB 硬體邊界管理本次對話之秘密金鑰（Secret Key, $sk$）。
3. 呼叫外部經認證的同態加密 API（HE Provider API）將明文編碼並加密為密文向量／多項式張量。
4. 將密文與評估金鑰（Evaluation Key, $evk$ / Galois Keys / Relin Keys）送往宣告支援密文推論的端點，執行同態推論。
5. 接收推論完成之密文結果，經 USB 驗證後於本機解密並即時渲染。

```
[本機使用者]
     │ (明文，僅在 USB 掛載時存在記憶體)
     ▼
[HEShell 本機殼層] ◄─── (秘密金鑰 sk 永不出界) ───► [USB 安全硬體]
     │
     │ 呼叫本地/外掛 HE API 進行 Encrypt(sk, Prompt)
     ▼
[密文請求體 (Ciphertext)] + [評估金鑰 evk]
     │ (網路上無明文、無反向還原能力)
     ▼
[端點路由器 (Router)]
     │
     ├── (L2 具能力端點) ──► [密文推論引擎] ──► [密文回應 (Ciphertext)]
     └── (不支援端點) ──► 拒絕路由 (FAIL-CLOSED，嚴禁退回明文)
```

### 1.2 能力成熟度分級（Taxonomy）
本專案嚴格遵守以下四級能力劃分，禁止向市場或工程團隊混稱能力等級：

| 級別 | 定義 | 傳輸態 | 計算態 | v1 狀態與驗收標準 |
| :--- | :--- | :--- | :--- | :--- |
| **L0** | 傳輸層加密 (TLS 1.3 / HTTPS) | 密文 | 伺服器端明文解密 | **非本產品**。不列為隱私保護賣點。 |
| **L1** | 指定 HE API 密文運算 | 密文 | 伺服器端純密文（固定 Circuit） | **本期交付**。支援特定 Embedding 比對、距離計算、數值評分與分類。 |
| **L2** | 端點密文模型推論 (Encrypted Inference) | 密文 | 伺服器端純密文推論並回傳密文 | **條件交付**。僅限端點具備明確能力宣告時路由，非通用端點。 |
| **L3** | 任意閉源 API 密文即時生成 | 密文 | 任意商用 LLM 直接解讀密文 | **非現行科技**。嚴禁承諾，不列入 v1 驗收與 Roadmap。 |

### 1.3 延遲預期與非即時互動規格
依據 2026 年公開密碼學與硬體加速研究，使用 CKKS 或 TFHE/FHEW 針對約 80 億參數（8B）模型執行端到端密文推論，相較於明文推論具有約 $10^4$（四個數量級）的運算代價：
- 短序列（Prompt $\le 64$ tokens）推論延遲約落在 **數十秒至數分鐘** 級距。
- 密文尺寸膨脹量：單一上下文狀態向量可能由數十 KB 膨脹為數十至數百 MB。
- **介面規範**：HEShell 介面**嚴禁提供虛假的「逐字打字機串流（Token Streaming）」動態**；必須提供進度階段指示器（Phase Indicator：金鑰封裝中 $\rightarrow$ 密文傳輸中 $\rightarrow$ 密文同態運算中 $\rightarrow$ 密文下載中 $\rightarrow$ 本機解密驗證），明示批次等待時間。

---

## 2. 模組架構切分

本倉庫採嚴格模組化架構，以避免記憶體跨層洩漏明文：

```
heshell-core/
├── apps/
│   └── desktop-ui/            # 前端桌面殼層 (Tauri/Rust + 安全渲染)
├── packages/
│   ├── usb-sentinel/          # USB 生命週期監控與硬體金鑰介面
│   ├── he-client-sdk/         # 外部同態加密 API 消費端 (抽象介面)
│   ├── zero-leak-router/      # 密文端點路由決策器 (Capability Matcher)
│   ├── secure-mem/            # 防交換 (mlock)、敏感字串即時覆寫工具
│   └── sanitized-logger/      # 零明文稽核日誌與崩潰處理器
```

### 模組權責表

| 模組名稱 | 權責邊界 | 嚴格禁止事項 |
| :--- | :--- | :--- |
| `usb-sentinel` | 偵測 USB 拔插事件、挑戰應答、對話 Session Key 配對。 | 禁止將 Secret Key 導出至未保護的堆積記憶體（Heap）；禁止雲端備份。 |
| `secure-mem` | 管理明文展示緩衝區，使用 `mlock` 鎖定記憶體，拔除時執行 `memset_s`。 | 禁止將明文寫入 OS Pagefile/Swap；禁止常規垃圾回收延遲釋放。 |
| `he-client-sdk` | 封裝外部 HE API，送出明文編碼要求並接回密文，或接收本機金鑰完成封裝。 | 禁止內嵌不符合標準之自製密碼原語；失敗時禁止降級為明文。 |
| `zero-leak-router`| 檢查遠端端點之 Capability Manifest；若非 L1/L2 支援則阻斷。 | 禁止向未宣告能力之端點送出任何對話封包；禁止「改送明文後備」。 |
| `sanitized-logger`| 記錄連線延遲、密文字節數、錯誤代碼、金鑰識別雜湊（Key ID Hash）。 | 禁止將 Prompt、回覆原文、非對稱私鑰字串寫入 Log 或 Crash Dump。 |

---

## 3. USB 狀態機與生命週期規範

解密金鑰嚴格受限於 USB 物理硬體邊界。本機殼層實作以下狀態機：

```
      [ 0. DISCONNECTED ] ◄──────────────────────────────────┐
               │                                             │
               │ (USB 插入 & 握手成功)                         │
               ▼                                             │
      [ 1. ATTACHED_IDLE ]                                   │
               │                                             │
               │ (發起對話: 生成 Session Key & 註冊)          │ (USB 拔除事件觸發)
               ▼                                             │
      [ 2. SESSION_ACTIVE ]                                  │
               │                                             │
               │ (推論完成密文回傳: 請求解密)                  │
               ▼                                             │
      [ 3. DECRYPTING ]                                      │
               │                                             │
               │ (解密成功: 明文渲染至畫面)                   │
               ▼                                             │
      [ 4. DISPLAYING_PLAINTEXT ] ───────────────────────────┘
```

### 狀態移轉與安全動作定義

1. **DISCONNECTED $\rightarrow$ ATTACHED_IDLE**：
   - 殼層偵測到裝置插入，執行硬體憑證與非對稱挑戰握手。
   - 驗證成功後，系統允許建立新的對話 Session。此時記憶體中無任何對話金鑰。
2. **ATTACHED_IDLE $\rightarrow$ SESSION_ACTIVE**：
   - 針對該次對話生成 Session Key Identifier ($SessionID$)。
   - USB 端點生成或導出該次對話專用之密鑰對：秘密金鑰 $sk$ 留存於 USB 本地邊界，評估金鑰 $evk$ 與公開金鑰 $pk$ 提供給殼層。
3. **SESSION_ACTIVE $\rightarrow$ DECRYPTING**：
   - 接收到來自 L2 雲端推論之結果密文 $C_{out}$。
   - 殼層將 $C_{out}$ 送交 USB 驅動邊界或本地受限環境，以 $sk$ 執行解密。
4. **DECRYPTING $\rightarrow$ DISPLAYING_PLAINTEXT**：
   - 密文解密為明文字串，存放於受 `mlock` 保護之隔離記憶體緩衝區中，直接映射至畫面。
5. **任意狀態 $\rightarrow$ DISCONNECTED（緊急阻斷程序）**：
   - **觸發條件**：USB 中斷連線（中斷信號延遲不得超過 50 毫秒）。
   - **強制連鎖反應**：
     1. **中斷執行**：立即中止正在進行中的網路傳輸與解密運算。
     2. **清除畫面**：GUI 視窗立刻以空白填滿，抹除畫面上之對話明文。
     3. **抹除記憶體**：對所有存放明文、對話快取、中介變數之緩衝區執行零值抹寫（Zeroize / `memset_s`）。
     4. **銷毀金鑰句柄**：本機端銷毀所有通往該 USB 的 Context Handle。
     5. **禁止回退**：嚴禁自磁碟快取或系統暫存區還原剛才對話之明文。

---

## 4. HE API 介面契約 (API Specification)

殼層不自行實作密碼原語，完全依循此介面契約消費外部 HE 服務或本機 HE 模組。

### 4.1 端點合約定義

#### A. 建立會話金鑰 (Session Key Initialization)
- **Method**: `POST /v1/he/session/init`
- **Request Body**:
  ```json
  {
    "scheme": "CKKS",
    "params": {
      "poly_modulus_degree": 32768,
      "scale_bits": 40,
      "security_level": "HEStd_128_classic"
    },
    "client_id": "heshell-node-a18f"
  }
  ```
- **Response Body**:
  ```json
  {
    "session_id": "sess_01J9X8K...",
    "public_key_ref": "pk_01J9X8K...",
    "eval_key_required": ["relin_keys", "galois_keys"],
    "created_at": 1791628800
  }
  ```
- **說明**：秘密金鑰 $sk$ 由本機 USB 生成，永遠不透過此 API 上傳。

#### B. 密文封裝/加密 (Client Ciphertext Package)
- **說明**：此步驟由本機 USB 驅動搭配本機 HE Client Engine 執行，不經過遠端未受信任伺服器。
- **輸入**：明文 Prompt、Schema 標註。
- **輸出結構**：
  ```json
  {
    "session_id": "sess_01J9X8K...",
    "format": "CKKS_PACKED_V1",
    "num_slots": 16384,
    "ciphertext_payload": "BASE64_ENCODED_CIPHERTEXT_BLOB...",
    "digest_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
  }
  ```

#### C. 密文請求轉送 (Encrypted Inference Dispatch)
- **Method**: `POST /v1/he/inference/dispatch`
- **Request Body**:
  ```json
  {
    "session_id": "sess_01J9X8K...",
    "target_model_capability": "HE_LLM_L2",
    "ciphertext_payload": "BASE64_ENCODED_CIPHERTEXT_BLOB...",
    "eval_keys": {
      "relin_key": "BASE64_ENCODED_EVAL_KEY...",
      "galois_keys": "BASE64_ENCODED_GALOIS_KEYS..."
    },
    "max_computation_depth": 14
  }
  ```
- **Response Body**:
  ```json
  {
    "task_id": "task_he_01J9X8M...",
    "status": "PROCESSING",
    "estimated_latency_sec": 45
  }
  ```

#### D. 密文結果輪詢與接收 (Fetch Result)
- **Method**: `GET /v1/he/inference/result/{task_id}`
- **Response Body (COMPLETED)**:
  ```json
  {
    "task_id": "task_he_01J9X8M...",
    "status": "COMPLETED",
    "ciphertext_result": "BASE64_ENCODED_RESULT_CIPHERTEXT_BLOB...",
    "consumed_depth": 12,
    "proof_of_execution": "SIG_RSA_SHA256_..."
  }
  ```

#### E. 解密 (Local Decrypt)
- **說明**：嚴格於本機且 USB 在位時呼叫本地安全模組執行。
- **輸入**：$sk$ (來自 USB 授權通道)、`ciphertext_result`。
- **輸出**：明文 Tokens。若 USB 已離線，解密呼叫立即遭硬體拒絕並拋錯。

### 4.2 標準錯誤碼規範

| 錯誤碼 | 符號常數 | 觸發情境與系統行為 |
| :--- | :--- | :--- |
| `HE_001` | `ERR_USB_MISSING` | 未偵測到 USB 或硬體授權失效。立即停止連線，清空工作區。 |
| `HE_002` | `ERR_CAPABILITY_UNSUPPORTED`| 目標端點未宣告 L1/L2 能力。**主動熔斷，嚴禁降級送出明文**。 |
| `HE_003` | `ERR_PLAINTEXT_LEAK_GUARD` | 檢驗模組偵測到輸出管道包含非密文格式數據。核心硬中斷退出。 |
| `HE_004` | `ERR_SESSION_MISMATCH` | 密文對話關聯之 Session Key ID 與當前掛載 USB 不相符。拒絕解密。 |
| `HE_005` | `ERR_CIPHERTEXT_CORRUPTED` | 密文雜湊校驗錯誤或同態深度耗盡（Noise Budget 溢出）。推論失敗。 |
| `HE_006` | `ERR_USB_PREMATURE_EJECT` | 運算或解密進行途中 USB 遭移除。拋出例外，執行全域記憶體洗刷。 |

---

## 5. 路由規則引擎 (Routing Rules Engine)

路由器 (`zero-leak-router`) 必須嚴格遵從 **Fail-Closed（防護性關閉）** 原則。

```
                    [ 使用者送出請求 ]
                           │
                           ▼
                  [ 檢查 USB 是否在位? ]
                  ├── 否 ──► 終止 (代碼 HE_001)
                  │
                  ▼ 是
             [ 查詢目標端點 Capability ]
                           │
      ┌────────────────────┴────────────────────┐
      ▼                                         ▼
[ 宣告具備 L1/L2 能力 ]                 [ 未宣告 / 僅支援明文 (L0) ]
      │                                         │
      ▼                                         ▼
[ 呼叫 HE API 本機加密 ]               [ 拒絕路由 (代碼 HE_002) ]
      │                                    [ 終止傳輸 ]
      ▼                                (嚴禁回退至明文 API)
[ 送出密文推論請求 ]
```

### 路由不變量（Routing Invariants）
1. **絕不自動降級**：當目標端點不支援密文同態推論時，路由器**不得提供**「是否改以標準明文 API 發送？」的選項；該請求必須被直接宣告為無效。
2. **能力宣告白名單檢驗**：端點必須提供經伺服器私鑰簽章之 Capability Manifest，且必須明記：
   - 支援方案：`CKKS` / `BFV` / `TFHE`
   - 最高乘法深度能力（Multiplicative Depth Capable）
   - 密文輸入輸出支援證明
3. **明文審查閘門**：出境封包發送前，由位址檢查器確認 Payload 內容。若無法識別為標準密文二進位結構或包含 ASCII/UTF-8 自然語言明文字元，該 Socket 即刻強制關閉。

---

## 6. 威脅模型 (Threat Model)

### 6.1 系統邊界與信任假設

```
[可信任邊界 (TCB)]
┌──────────────────────────────────────────────┐
│  - 本機實體 USB 安全硬體                      │
│  - HEShell 受 mlock 保障之客戶端核心程序      │
└──────────────────────────────────────────────┘
                       ▲
                       │ (受監控之本機介面)
                       ▼
[不可信任環境 (Untrusted)]
┌──────────────────────────────────────────────┐
│  - 宿主作業系統其他程序 / Pagefile / Swap    │
│  - 公共網路 / 代理伺服器                     │
│  - 遠端推論伺服器 (Untrusted Compute Server) │
│  - 雲端日誌系統與第三者計費端點              │
└──────────────────────────────────────────────┘
```

### 6.2 威脅情境與緩解措施

| 威脅編號 | 威脅情境 (Threat Vector) | 嚴重度 | 緩解措施 (Mitigations) |
| :--- | :--- | :---: | :--- |
| **TM-01** | **惡意伺服器端竊聽對話內容**：運算節點具備惡意，試圖查看使用者輸入之機密資料。 | 高 | 進入傳輸管道之對話完全為 HE 密文，伺服器僅持有評估金鑰（$evk$），密碼學保證無 $sk$ 絕對無法還原明文。 |
| **TM-02** | **USB 拔除後殘留明文竊取**：使用者在公用或受限場所拔除 USB 後，本機遭人提取記憶體。 | 高 | 註冊作業系統底層硬體事件；拔除瞬間執行 `memset_s` 強制抹除明文，介面即刻白屏，無磁碟明文快取。 |
| **TM-03** | **記憶體交換檔洩漏**：作業系統記憶體壓力過大，將對話明文寫入硬碟 Swap/Pagefile。 | 中 | 對話輸入輸出緩衝區全面配置虛擬鎖定（Linux: `mlock`, Windows: `VirtualLock`），強制禁止分頁寫入磁碟。 |
| **TM-04** | **日誌與當機回報外洩**：系統 Crash 生成 Dump 或 Logger 寫入硬碟記錄檔。 | 高 | 自定義 Logger 管道；結構化 Log 僅允許數值型指標（Latency、Bytes）。Crash Handler 封鎖 Full Memory Dump，僅允許無狀態 Stack Trace。 |
| **TM-05** | **評估金鑰逆向解密攻擊**：攻擊者意圖透過收集之 $evk$ 與輸出密文解算輸入明文。 | 低 | 嚴格依循同態加密國際標準（Homomorphic Encryption Standard）參數設定（至少 128-bit 經典安全強度），使逆向在計算上不可行。 |
| **TM-06** | **中間人竄改能力宣告欺騙降級**：中間人攔截網路，偽造不支援密文之信號，誘騙殼層改送明文。 | 高 | 路由規則強制 Fail-Closed。遇到非密文回覆直接報錯中斷，不具備明文回退邏輯路徑。 |

---

## 7. 開發里程碑與驗收標準 (Milestones & Acceptance)

### 7.1 開發里程碑規劃

```
M1 (第 1-3 週)  ──► M2 (第 4-6 週)  ──► M3 (第 7-9 週)  ──► M4 (第 10-12 週)
硬體金鑰介面與       外部 HE API 整合與     零洩漏路由器與       端到端密文推論整合
本機安全記憶體       密文封裝測試           UI 狀態機實作       與故障驗收 (GA)
```

- **M1：硬體金鑰介面與本機安全記憶體**
  - 完成 USB 插入/拔出監控守護程序。
  - 完成 `secure-mem` 記憶體鎖定與即時覆寫機制（測試 50ms 內清空驗證）。
- **M2：外部 HE API 整合與密文封裝測試**
  - 串接外部 HE Provider API，完成 Session 建立與金鑰交換介面。
  - 建立本機封裝 SDK，驗證密文產生正確性與參數結構合規性。
- **M3：零洩漏路由器與 UI 狀態機實作**
  - 實作桌面殼層介面，支援非即時進度指示（Phase Indicator）。
  - 實作 `zero-leak-router`，驗證遇到 L0 端點時 100% 觸發阻斷中斷。
- **M4：端到端密文推論整合與故障驗收**
  - 連接已宣告支援密文計算之 L2 測試端點。
  - 實施安全紅隊測試（USB 物理拔除、網路中間人攔截、記憶體傾印掃描）。

### 7.2 嚴格驗收判準 (Definition of Done)

本專案**不以「成功串接任一公有對話 API 並秒回文字」作為驗收標準**。驗收必須通過以下全部 Gate 檢查：

1. **Gate 1：金鑰邊界不可侵犯性**
   - 拔除 USB 狀態下發起解密請求，系統必須回傳 `ERR_USB_MISSING (HE_001)`，且畫面上與記憶體內均不得出現明文。
   - 驗證任何網路出境封包，絕不包含秘密金鑰（$sk$）。
2. **Gate 2：出境明文清零驗證**
   - 於本機端網卡執行封包捕獲（Wireshark / eBPF Trace），檢查送往遠端之推論請求 Body。
   - 請求必須為標準 Base64 密文或二進位 Payload；凡捕獲到任何自然語言文字片段，即判定為驗收失敗。
3. **Gate 3：非相容端點強行中斷驗證**
   - 將路由目標設定為標準公有商用 LLM 端點（L0）。
   - 系統必須拒絕送出，並回傳 `ERR_CAPABILITY_UNSUPPORTED (HE_002)`；**嚴禁以「降級發送明文」使對話成功**。
4. **Gate 4：緊急拔除白屏與記憶體抹寫測試**
   - 於明文對話渲染完成後，隨機拔除 USB。
   - 監控程序必須在 50ms 內完成畫面白屏，並對對話緩衝區實施 `memset_s`。透過外部 Dump 工具檢查記憶體，明文字串命中數必須為 0。
5. **Gate 5：日誌安全審計**
   - 檢索所有系統執行紀錄、崩潰傾印檔案（Crash Dump）與路由追蹤檔，明文出現率必須為 0%。
