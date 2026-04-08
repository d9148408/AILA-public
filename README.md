# 🚀 AILA — Aspen Intelligent Language Agent

> 💡 *AILA* 在芬蘭語中是「**光的使者**」之意——寓意以 AI 的光芒照亮複雜的化工模擬流程。

**AILA（艾拉）** 是一個基於大型語言模型（LLM）與 LangGraph 代理工作流的 Aspen Plus 自動化控制系統。透過三層式微服務架構（大腦層、控制層、執行層），本系統能讓使用者以自然語言直接操控本地端的 Aspen Plus 軟體，並具備自我錯誤修復（Error Recovery）與安全權限控管能力。

---

## 🏗️ 系統架構 (A ↔ B ↔ C 三層式微服務)

本專案採用 **職責分離 (Separation of Concerns)** 原則，將系統打散至三台（或三類）不同的計算節點，以確保運算資源的最佳配置與終端電腦的安全性。

> 📌 **一般使用者（學生/工程師）只需關注 C 電腦的設定與使用。**
> A 電腦（推理伺服器）與 B 電腦（控制伺服器）由系統管理員負責部署與維護，使用者無需操作。

### 🧠 A 電腦：大腦層 (Inference Server)
* **硬體定位**：具備高階 GPU 的伺服器（如 DGX Spark）。
* **運行服務**：使用 `vLLM` 部署 Qwen3.5 35B A3B FP8模型（未來將掛載 Aspen 專屬的 LoRA Adapter）。
* **核心職責**：
  * 接收來自 B 電腦的 Prompt。
  * 進行 `<think>` 思維鏈推理（生成 COM 路徑或判斷 Input File 語法）。
  * 嚴格輸出符合規範的 `tool_call` JSON 格式代碼。

### ⚙️ B 電腦：中樞控制層 (Orchestration Server)
* **硬體定位**：應用程式伺服器 (IP: `<B_SERVER_IP>`)。
* **運行服務**：LangGraph 狀態機代理 (Agent) + FastAPI (`Port: 8001`)。
* **核心職責**：
  * **路由分配**：透過帳號 (Thread ID) 將對話綁定至指定的 C 電腦 IP。
  * **工作流控制**：執行 Planner (LLM思考) → Executor (派發代碼) 迴圈。
  * **錯誤恢復**：當 C 電腦回報 Aspen 執行錯誤，或模型 JSON 格式損壞時，觸發 `Fallback` 節點要求大腦重新修正。
  * **對話記憶**：維護多輪對話上下文，並在對話過長時觸發 `Summarizer` 壓縮記憶。

### 💻 C 電腦：執行層與終端用戶 (Edge Node & Client)
* **硬體定位**：工程師的 Windows 個人電腦 (安裝有 Aspen Plus 軟體)。
* **運行服務**：Tkinter 圖形化介面 + 背景 Uvicorn Worker (`Port: 9000`)。
* **核心職責**：
  * **用戶互動**：提供登入驗證、發送自然語言指令，並以 Markdown 格式優美渲染 AI 回覆。
  * **被動執行**：背景 FastAPI 伺服器隨時待命，接收來自 B 電腦派發的 Python COM 代碼，透過 `exec()` 控制本機的 Aspen Plus。
  * **結果回傳**：將 Aspen 的成功數據或底層報錯 (Error Traceback) 捕獲並回傳給 B 電腦。

---

## 📂專案資料夾結構

```
AILA/
├── README.md                                      # 本說明文件
├── aspen_lora_project_plan.md                     # LoRA 訓練策略白皮書（最高指導原則）
│
├── A_LLM/                                         # A 電腦：大腦層（推理伺服器）
│   ├── .gitignore                                 # 排除 outputs/ 大型模型權重
│   ├── Spark_vLLM_docker/                         # DGX Spark vLLM Docker 部署設定
│   │   └── recipe/
│   │       └── qwen3.5-35b-a3b-fp8-local.yaml    # vLLM 服務啟動配方檔
│   │
│   ├── training_data/                             # 訓練文本資料集根目錄
│   │   ├── 01_raw_sources/                        # 原始素材（未加工）
│   │   │   ├── scripts/                           # 現有 Aspen Python 自動化腳本
│   │   │   ├── com_path_notes/                    # COM 路徑知識文件（.md / .txt）
│   │   │   └── inp_examples/                      # 歷史 Input File 範例（.inp）
│   │   │
│   │   ├── 02_generated/                          # 依計畫各階段產出的 JSONL
│   │   │   ├── phase1_format/                     # 第一階段：ChatML 格式標準化樣本
│   │   │   ├── phase2_patterns/                   # 第二階段：CoT 模式 + 反向防禦樣本
│   │   │   ├── phase3_scripts/                    # 第三階段：既有腳本原子化轉換
│   │   │   ├── phase4_examples/                   # 第四階段：實戰情境 A-F, K, L, M
│   │   │   └── phase5_advanced/                   # 第五階段：進階工程實務 G-J
│   │   │
│   │   ├── 03_generalization/                     # 變數泛化增強樣本（Block 名稱隨機替換）
│   │   │
│   │   ├── 04_external_general/                   # 通用資料（防災難性遺忘，佔比 10-20%）
│   │   │   ├── python_coding/                     # 通用 Python 演算法題
│   │   │   ├── logic_reasoning/                   # 邏輯推理對話
│   │   │   └── zh_en_dialogue/                    # 中英文日常問答
│   │   │
│   │   └── 05_final/                              # 最終合併資料集（Ready to Train）
│   │       ├── train.jsonl                        # 訓練集（80%）
│   │       └── eval.jsonl                         # 驗證集（20%）
│   │
│   ├── tools/                                     # 資料生成與品質控管工具腳本
│   │   ├── atomize_scripts.py                     # 腳本原子化（步驟一）
│   │   ├── to_chatml_converter.py                 # 轉換為 ChatML 格式（步驟三）
│   │   ├── variable_generalizer.py                # 變數泛化處理（步驟四）
│   │   ├── self_instruct.py                       # Self-Instruct 批次生成腳本
│   │   ├── merge_dataset.py                       # 合併 + shuffle + 切分資料集
│   │   └── validate_jsonl.py                      # 品質驗證（json.loads 可解析檢查）
│   │
│   ├── training/                                  # LoRA 訓練相關
│   │   ├── train_lora.py                          # Unsloth + SFTTrainer 訓練主程式
│   │   └── configs/
│   │       └── lora_config.yaml                   # LoRA 超參數配置（r=64, bf16）
│   │
│   └── outputs/                                   # 訓練輸出（.gitignore 排除）
│       ├── aspen_lora_adapter/                    # 訓練產出的 LoRA adapter 權重
│       └── logs/                                  # 訓練日誌
│
├── B_LangGraph/                                   # B 電腦：中樞控制層（協調伺服器）
│   ├── agent_api.py                               # FastAPI 入口點，提供 /login 與 /chat 端點
│   └── aspen_agent.py                             # LangGraph 狀態機核心，定義四大節點
│
└── C_Client/                                      # C 電腦：執行層與終端用戶
    ├── aspen_client_app.py                        # Tkinter GUI + FastAPI Worker 雙執行緒客戶端
    ├── AILA_environment.yml                       # conda 環境設定檔（conda env create -f 安裝）
    └── nthu_client.ovpn                           # OpenVPN 設定檔 ⚠️ gitignored，向管理員申請
```

---

## 📂 核心檔案說明

### 1. A 電腦 (大腦) 檔案

#### 部署設定
* **`A_LLM/Spark_vLLM_docker/recipe/qwen3.5-35b-a3b-fp8-local.yaml`**
  * **功能**：DGX Spark 上啟動 vLLM 推理服務的 YAML 配方檔。
  * **特點**：
    * 指定從本地路徑 `/models/Qwen3.5-35B-A3B-FP8` 載入 FP8 量化模型，無需連線外部下載。
    * 開啟 `--enable-auto-tool-choice` 及 `--tool-call-parser qwen3_coder`，確保模型能穩定輸出 `tool_call` JSON 格式。
    * 使用 `--reasoning-parser qwen3` 啟用 `<think>` 思維鏈推理分離機制。
    * 預設 Port `8888`，KV Cache 採 FP8 節省視頻記憶體，搭配 `flashinfer` attention 後端加速推理。

#### 訓練資料工作流程
訓練資料依下方流程逐步產出，最終合併為 `05_final/train.jsonl` 作為 LoRA 微調輸入：

```
01_raw_sources/          02_generated/           03_generalization/
  scripts/       ──────► phase1_format/   ──────►  (變數泛化 +5x)
  com_path_notes/ ─────► phase2_patterns/          │
  inp_examples/  ──────► phase3_scripts/           │
                         phase4_examples/          ▼
                         phase5_advanced/  04_external_general/
                                           (通用資料 10-20%)
                                                    │
                                                    ▼
                                          05_final/train.jsonl (80%)
                                          05_final/eval.jsonl  (20%)
```

#### 工具腳本
* **`A_LLM/tools/atomize_scripts.py`** — 腳本原子化：將長篇腳本拆解為單一職責函數
* **`A_LLM/tools/to_chatml_converter.py`** — 將函數封裝轉換為 ChatML 格式（`conversations` 鍵）
* **`A_LLM/tools/variable_generalizer.py`** — Block/Stream 名稱隨機替換，單一樣本衍生 5+ 筆
* **`A_LLM/tools/self_instruct.py`** — 呼叫外部強模型批次翻轉知識文件為問答對（防 CPT 陷阱）
* **`A_LLM/tools/merge_dataset.py`** — 合併所有 JSONL + shuffle + 8:2 切分 train/eval
* **`A_LLM/tools/validate_jsonl.py`** — 品質驗證：`json.loads()` 可解析、`<think>` 比例 ≥75%

#### 訓練設定
* **`A_LLM/training/train_lora.py`** — Unsloth `FastModel` + SFTTrainer 訓練主程式；使用 `train_on_responses_only` 僅對 Assistant 回覆計算 Loss
* **`A_LLM/training/configs/lora_config.yaml`** — LoRA 超參數配置：`r=64`、`lora_alpha=64`、bf16（MoE 架構禁用 QLoRA）

### 2. B 電腦 (控制伺服器) 檔案
* **`agent_api.py`**
  * **功能**：對外提供 RESTful API 服務的進入點。
  * **特點**：包含簡易的帳號密碼驗證資料庫 (`VALID_USERS`)。提供 `/login` (登入驗證) 與 `/chat` (對話接收與轉交 LangGraph) 兩個核心端點。
* **`aspen_agent.py`**
  * **功能**：系統的心臟，定義 LangGraph 狀態機 (`StateGraph`)。
  * **特點**：
    * 定義四大節點：`planner_node` (呼叫大腦)、`executor_node` (遠端派發)、`fallback_node` (錯誤重試)、`summarizer_node` (記憶壓縮)。
    * 內建 `AspenOutputParser`，可剝離 `<think>` 區塊並提取 JSON。
    * 包含 `SESSION_WORKER_MAP` 邏輯，負責將 Thread ID 映射到對應的 C 電腦 IP 資源池。

### 3. C 電腦 (用戶終端) 檔案
* **`nthu_client.ovpn`** ⚠️ *此檔案已加入 `.gitignore`，不會 commit 至 Git，請向管理員索取*
  * **功能**：OpenVPN 用戶端設定檔，用於連線至機構內網。
  * **特點**：
    * 採用 Split Tunneling，**僅將內網网段流量導入 VPN**，其餘網路流量仍走本機路由，不影響一般上網。
    * 需搭配帳號密碼（`auth-user-pass`）與憑證雙重驗證。
    * **安全性說明**：檔案內含個人私鑰（Encrypted Private Key）與憑證，屬高度敏感資料，禁止上傳至任何公開平台。
* **`AILA_environment.yml`**
  * **功能**： conda 環境設定檔，一行指令建立完整 Python 3.11 環境。
  * **用法**：`conda env create -f AILA_environment.yml`
* **`aspen_client_app.py`**
  * **功能**：將 FastAPI Worker 與 Tkinter GUI 雙執行緒打包的客戶端程式。
  * **特點**：
    * **雙生架構**：啟動時自動在背景開啟 `Port 9000` 監聽代碼派發。
    * **安全驗證**：預設鎖定對話框，需輸入帳密驗證成功後方可解鎖，並自動將帳號作為 Thread ID 防止撞號。
    * **強大容錯**：內建 `json_repair` 模組，即使 LLM 出現括號漏寫等格式幻覺，也能自動修復解析。
    * **動態 UI**：支援 `Shift+Enter` 換行、動態進度條 (Progressbar) 以及 Markdown 粗體渲染。

### 4. 專案工程文件
* **`aspen_lora_project_plan.md`**
  * **功能**：專案開發與模型訓練的「最高指導原則」白皮書。
  * **特點**：詳細記載了 Qwen3.5 MoE 架構的 LoRA 訓練策略、JSONL 語料的黃金配比（策略 A）、以及「COM 介面 vs. Input File」雙路徑的邊界判斷邏輯。是未來資料標準化階段的核心依據。

---

## 🔐 VPN 連線設定 (OpenVPN)

> **前置條件**：C 電腦（學生/工程師的 Windows PC）必須透過 OpenVPN 連線至機構內網，才能與 B 電腦通訊。

### 步驟一：安裝 OpenVPN Connect（或 OpenVPN GUI）

1. 前往官方下載頁：https://openvpn.net/client/
2. 下載 **OpenVPN Connect v3**（Windows 64-bit 版本）。
3. 執行安裝程式，全部選項保持預設，直接 Next → Install → Finish。

> 亦可使用 **OpenVPN GUI（Community Edition）**，介面較簡單，操作相同。

### 步驟二：取得設定檔

向系統管理員索取個人專屬的 `nthu_client.ovpn` 設定檔。

> ⚠️ **安全注意事項**：
> - `.ovpn` 檔案內含您的個人私鑰與憑證，請**勿**透過電子郵件、LINE 等未加密管道傳送。
> - 請勿將此檔案上傳至 GitHub、Google Drive 或任何公開雲端儲存。
> - 每位使用者的設定檔**獨立且唯一**，請勿共用。

### 步驟三：匯入設定檔並連線

#### 方法 A：OpenVPN Connect（拖曳匯入）
1. 開啟 **OpenVPN Connect**。
2. 直接將 `nthu_client.ovpn` 拖曳至 OpenVPN Connect 視窗，或點選 **Upload File** → 選取檔案。
3. 出現 Profile Added 提示後，點選該 Profile 旁的切換開關（Toggle）。
4. 輸入管理員提供的 **VPN 帳號密碼**，按 Connect。
5. 圖示變為 **綠色勾勾** 即表示連線成功。

#### 方法 B：OpenVPN GUI（右鍵選單）
1. 將 `nthu_client.ovpn` 複製至：
   ```
   C:\Users\<你的使用者名稱>\OpenVPN\config\
   ```
2. 在系統匣（右下角）的 OpenVPN GUI 圖示按右鍵。
3. 選取 **nthu_client → Connect**。
4. 輸入管理員提供的 **VPN 帳號密碼**，按 OK。
5. 圖示變為 **綠色** 即表示連線成功。

### 步驟四：驗證連線

連線成功後，在 Windows 命令提示字元或 PowerShell 中 ping B 伺服器 IP，收到回應即代表 VPN 連線正常。

### VPN 連線參數說明

| 項目 | 設定值 |
| --- | --- |
| **伺服器** | 向管理員索取 |
| **協定** | UDP |
| **加密演算法** | AES-256-GCM |
| **HMAC 驗證** | SHA256 |
| **路由模式** | Split Tunneling（僅內網流量走 VPN） |
| **驗證方式** | 憑證 + 帳號密碼雙重驗證 |

---

## 💻 C 電腦（用戶端）環境安裝

> 📌 **本節為一般使用者的必要設定。** A、B 電腦為伺服器端，由管理員負責，使用者不需要操作。

### 前置需求

* Windows 10 / 11 (64-bit)
* [Miniconda](https://docs.conda.io/en/latest/miniconda.html) 或 Anaconda（用於建立 conda 虛擬環境）
* Aspen Plus 軟體（已安裝於本機，含 COM 介面授權）
* OpenVPN（詳見上方 [VPN 連線設定](#-vpn-連線設定-openvpn) 章節）

### 步驟一：使用環境設定檔建立 conda 虛擬環境

```powershell
# 切換至 C_Client 資料夾
cd C_Client

# 依 AILA_environment.yml 建立並安裝完整環境（名稱：AILA，Python 3.11）
conda env create -f AILA_environment.yml
```

> `C_Client/AILA_environment.yml` 為由開發環境直接匯出的完整 conda 環境設定檔，包含 `fastapi`、`uvicorn`、`langchain`、`langgraph`、`pywin32`、`pillow`、`json-repair` 等所有必要依賴，一行指令即可完成全套安裝。

### 步驟二：確認 pywin32 後處理

```powershell
# 安裝後執行 pywin32 post-install（Windows COM 介面必要步驟）
python C:\Users\<你的使用者名稱>\miniconda3\envs\AILA\Scripts\pywin32_postinstall.py -install
```

### 步驟三：啟動用戶端程式

```powershell
conda activate AILA
python aspen_client_app.py
```

---

## 🏃‍♂️ 如何啟動系統 (執行順序)

> ⚙️ **步驟 1–2 由系統管理員負責，一般使用者從步驟 3 開始操作。**

1. **[管理員] 啟動 A 電腦 (vLLM 推理伺服器)**
   * 在 DGX 伺服器上執行 vLLM 部署指令，開啟 API 服務 (預設 Port 8888)。
2. **[管理員] 啟動 B 電腦 (LangGraph 控制伺服器)**
   * 在 B 電腦終端機執行：`python agent_api.py`
   * 確認 Uvicorn 成功運行於 `0.0.0.0:8001`。
3. **[使用者] C 電腦連線 VPN（前置，必要）**
   * 開啟 OpenVPN Connect 或 OpenVPN GUI，連線 `nthu_client.ovpn`。
   * 確認可以 ping 通 B 伺服器後，再進行下一步。
4. **[使用者] 啟動 C 電腦用戶端**
   * 確認 Conda 虛擬環境 `AILA` 已依照上方「C 電腦環境安裝」章節設定完畢。
   * 在 C 電腦 (Windows) 執行：`python aspen_client_app.py` (或執行打包好的 `.exe` 檔)。
   * 確認本機的 Aspen Plus 軟體已開啟，並載入相關模擬檔。
   * 於 GUI 介面輸入帳號密碼（向管理員索取），登入後即可開始對話！

---

## 💡 當前專案進度

- [x] A ↔ B ↔ C 微服務物理連線與非同步 API 打通
- [x] LangGraph 狀態機建置與 Error Recovery 閉環測試
- [x] 終端 GUI 登入驗證、動態渲染與 `json_repair` 容錯實作
- [ ] **[下一步] 第三階段：既有腳本標準化 (將 COM 操作轉換為 JSONL 訓練語料)**
- [ ] **[下一步] 第六階段：執行 LoRA 微調訓練**
