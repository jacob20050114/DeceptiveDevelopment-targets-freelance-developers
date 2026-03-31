# DeceptiveDevelopment — MITRE ATT&CK TTP 分析

> 基於 MITRE ATT&CK Framework v16，針對 ESET 研究報告「DeceptiveDevelopment targets freelance developers」之完整攻擊行動 TTP 對照分析。

## 攻擊行動概述

DeceptiveDevelopment 是一個與北韓有關的攻擊行動群集，自 2023 年 11 月起活躍。攻擊者偽裝為招募人員，透過求職與自由接案平台接觸目標（主要為加密貨幣相關的軟體開發者），以面試程式測試為藉口，散布包含惡意程式碼的專案，最終部署兩階段惡意軟體：**BeaverTail**（資訊竊取器/下載器）與 **InvisibleFerret**（資訊竊取器/RAT）。

---

## 1. Resource Development（資源開發）

攻擊者在行動開始前所準備的基礎設施與工具。

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1583.003** | Acquire Infrastructure: Virtual Private Server | 攻擊者向商業主機商（RouterHosting/Cloudzy、Stark Industries Solutions、Pier7ASN）租用 VPS 作為 C&C 與暫存伺服器，伺服器 API 以 Node.js 撰寫，包含 9 個端點（`/pdown`、`/uploads`、`/client`、`/payload`、`/brow`、`/adc`、`/mclip`、`/keys`、`/api/clip`）。 |
| **T1587.001** | Develop Capabilities: Malware | 攻擊者自行開發 BeaverTail（JavaScript 版本與 Qt/C++ 原生版本）和 InvisibleFerret（Python 模組化惡意軟體），並持續迭代更新（2024 年 8 月新增 loader 架構、`ssh_zcp` 指令；12 月新增 `mlip` 鍵盤側錄模組）。 |
| **T1585.001** | Establish Accounts: Social Media Accounts | 攻擊者在 LinkedIn、Upwork、Freelancer.com、We Work Remotely、Moonlight、Crypto Jobs List 等平台建立假招募人員帳號，複製真實人物的個人資料或建構全新人設。部分帳號可能是入侵後修改的真實帳號，擁有大量追蹤者以增加可信度。 |
| **T1608.001** | Stage Capabilities: Upload Malware | InvisibleFerret 的各模組（payload、browser、AnyDesk、keylogger）預先上傳至 C&C 暫存伺服器，受害者遭入侵後由 BeaverTail 自動從伺服器下載。Python 執行環境（`p2.zip`）也預先存放於伺服器。 |

---

## 2. Initial Access（初始存取）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1566.003** | Phishing: Spearphishing via Service | 攻擊者透過求職與自由接案平台（LinkedIn、Upwork 等）直接聯繫目標開發者，以面試程式測試或修復 Bug 為藉口，提供包含惡意程式碼的 GitHub/GitLab/Bitbucket 專案連結。另一手法是提供假的視訊會議軟體下載連結（如模仿 MiroTalk 的 `mirotalk[.]net`），軟體內含 BeaverTail 惡意程式。 |

**詳細說明：**
- 專案通常設為 **private repository**，要求受害者提供帳號 ID 或 email 才給予存取權限，藉此隱匿惡意活動
- 木馬化專案分為四類：面試挑戰、加密貨幣專案、遊戲（含區塊鏈功能）、博弈（含加密貨幣功能）
- 攻擊者冒用合法專案名稱，加上 LLC、Ag、Inc 等公司後綴以增加可信度

---

## 3. Execution（執行）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1204.002** | User Execution: Malicious File | 受害者被要求下載專案後建置並執行以測試功能，觸發隱藏在專案中的 BeaverTail 惡意程式碼。攻擊者將惡意程式碼藏在長註解後方的同一行，利用不換行的特性使其超出螢幕可視範圍（GitHub 的程式碼編輯器預設不啟用自動換行）。 |
| **T1059.007** | Command and Scripting Interpreter: JavaScript/JScript | BeaverTail 的 JavaScript 版本直接嵌入木馬化專案中，使用 Base64 編碼與字串分割混淆。2024 年 8 月新版改為 loader 架構，僅下載並執行遠端 payload。 |
| **T1059.006** | Command and Scripting Interpreter: Python | InvisibleFerret 完全以 Python 撰寫，BeaverTail 會先下載獨立的 Python 執行環境（`p2.zip`），再下載 InvisibleFerret 的 `.npl` 檔案並用該環境執行。 |
| **T1059.003** | Command and Scripting Interpreter: Windows Command Shell | InvisibleFerret 的 `ssh_obj` 指令透過 Python 的 `subprocess` 模組執行系統 shell 指令，實現遠端命令執行。`ssh_kill` 指令在 Windows 上使用 `taskkill` 終止 Chrome 與 Brave 瀏覽器行程。 |

---

## 4. Persistence（持續性）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1133** | External Remote Services | InvisibleFerret 的 `adc` 模組下載並安裝 AnyDesk 遠端桌面軟體，將攻擊者預設的密碼雜湊值、密碼鹽值、token 鹽值寫入 AnyDesk 設定檔。若設定檔修改失敗，則建立 PowerShell 腳本 `conf.ps1` 來完成設定。這是整個攻擊鏈中**唯一的持續性機制**。安裝完成後攻擊者可隨時透過 AnyDesk 遠端連入並重新執行 InvisibleFerret。 |

---

## 5. Defense Evasion（防禦規避）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1140** | Deobfuscate/Decode Files or Information | BeaverTail JavaScript 版本使用 Base64 編碼、字串分割與重組（將 IP 位址分成三段後交換順序）來混淆 C&C 位址。其他字串也以 Base64 編碼，並在前方加入一個多餘字元來阻止簡單解碼。InvisibleFerret 的 C&C 位址同樣以 Base64 編碼後分成兩半並交換。 |
| **T1027.013** | Obfuscated Files or Information: Encrypted/Encoded File | InvisibleFerret 各模組的 payload 使用 XOR 加密後再以 Base64 編碼，前 4 bytes 為 XOR 金鑰。執行時先解密再透過 `exec` 執行。檔案傳輸時部分檔案使用 XOR 金鑰 `G01d*8@(` 加密。 |
| **T1564.001** | Hide Artifacts: Hidden Files and Directories | InvisibleFerret 檔案以隱藏屬性儲存於磁碟（如 `.npl`、`.n2/pay`、`.n2/bow`），檔名以「.」開頭在 Linux/macOS 上預設為隱藏。 |
| **T1564.003** | Hide Artifacts: Hidden Window | InvisibleFerret 在 Windows 上以 `CREATE_NO_WINDOW` 和 `CREATE_NEW_PROCESS_GROUP` 旗標建立新行程，隱藏視窗避免被受害者察覺。 |

**額外規避手法（未列入 ATT&CK 但值得注意）：**
- 惡意程式碼藏在長註解後方同一行，超出螢幕可視範圍
- 使用 private repository 限制研究人員存取
- 使用免費混淆工具處理程式碼

---

## 6. Credential Access（憑證存取）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1555.001** | Credentials from Password Stores: Keychain | BeaverTail 竊取 macOS 的 `/Library/Keychains/login.keychain` 和 Linux 的 `~/.local/share/keyrings/` 中的登入資訊。InvisibleFerret 同樣具備此功能。 |
| **T1555.003** | Credentials from Password Stores: Credentials from Web Browsers | InvisibleFerret 的 `bow` 模組針對 Chrome、Brave、Opera、Yandex、Edge 等 Chromium 瀏覽器，從本機儲存資料夾複製登入資料庫（`LoginData.db`）與付款資訊資料庫（`webdata.db`）至暫存目錄，使用 AES 解密金鑰解密後回傳 C&C。解密金鑰取得方式依作業系統不同：Windows 從 `Local State` 檔案擷取、Linux 透過 `secretstorage` 套件、macOS 透過 `security` 工具。 |
| **T1552.001** | Unsecured Credentials: Credentials In Files | BeaverTail 竊取 `~/.config/solana/id.json`（Solana 金鑰）、Firefox 的 `key3.db`、`key4.db`、`logins.json`。InvisibleFerret 新版 `ssh_zcp` 指令可竊取 1Password、Electrum、WinAuth、Proxifier4、Dashlane 等密碼管理器資料，以及 Atomic 和 Exodus 加密貨幣錢包。 |

**竊取的瀏覽器延伸套件（BeaverTail 初始版本，共 9 個）：**
- MetaMask（Chrome/Edge）、BNB Chain Wallet、Coinbase Wallet、TronLink、Phantom、Ronin Wallet、Coin98 Wallet、Crypto.com Wallet

**新增的延伸套件（2024 年 8 月，共 4 個）：**
- Kaia Wallet、Rabby Wallet、Argent X (Starknet)、Exodus Web3 Wallet

**InvisibleFerret 新版針對的延伸套件：共 88 個**（包含 MetaMask、Phantom、Trust、UniSat、Keplr、LastPass、GoogleAuth 等主流錢包與密碼管理器）

---

## 7. Discovery（探索）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1082** | System Information Discovery | BeaverTail 收集電腦主機名稱與時間戳記。InvisibleFerret `pay` 模組收集 UUID、作業系統類型、電腦名稱、使用者名稱、系統版本。 |
| **T1016** | System Network Configuration Discovery | InvisibleFerret 收集本機 IP 位址與公開 IP 位址（透過查詢 `http://ip-api.com/json`）。 |
| **T1614** | System Location Discovery | InvisibleFerret 透過 IP 地理位置 API 取得區域名稱、國家、城市、郵遞區號、ISP、經緯度等地理資訊。 |
| **T1124** | System Time Discovery | InvisibleFerret 收集系統時間資訊。 |
| **T1083** | File and Directory Discovery | InvisibleFerret 後門可瀏覽檔案系統，支援目錄遍歷與檔案搜尋（`sdira`、`sdir`、`sfinda`、`sfindr`、`sfind` 子指令）。 |
| **T1010** | Application Window Discovery | InvisibleFerret 鍵盤側錄器收集目前使用中視窗的名稱。2024 年 12 月新版限縮至僅監控 `chrome.exe` 與 `brave.exe` 行程。 |
| **T1217** | Browser Bookmark Discovery | InvisibleFerret 搜尋並竊取瀏覽器儲存的憑證與相關資料。 |

---

## 8. Lateral Movement（橫向移動）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1021.001** | Remote Services: Remote Desktop Protocol | 透過安裝 AnyDesk 並設定攻擊者的憑證，實現遠端桌面存取。攻擊者可隨時連入受害系統進行後續操作。 |

---

## 9. Collection（資料收集）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1056.001** | Input Capture: Keylogging | InvisibleFerret 在 Windows 上使用 `pyWinHook` 實作鍵盤側錄，在獨立執行緒中持續記錄所有按鍵。2024 年 12 月新版（`mlip` 模組）限縮至僅記錄 Chrome 與 Brave 行程的按鍵。 |
| **T1115** | Clipboard Data | InvisibleFerret 使用 `pyperclip` 套件監控剪貼簿變更，將內容存入全域緩衝區。透過 `ssh_clip` 指令可將鍵盤側錄與剪貼簿資料回傳 C&C。 |
| **T1119** | Automated Collection | BeaverTail 自動收集瀏覽器延伸套件的 `.ldb` 與 `.log` 檔案、Solana 金鑰、Keychain/Keyrings 資料、Firefox 登入資料庫。InvisibleFerret 初始指紋辨識也是自動執行。 |
| **T1005** | Data from Local System | 兩種惡意軟體皆從本機系統竊取資料。InvisibleFerret 的 `ssh_env` 指令可上傳 Windows 上的 Documents、Downloads 資料夾及 D-I 磁碟內容；其他系統則上傳整個家目錄與 `/Volumes` 目錄。 |
| **T1025** | Data from Removable Media | InvisibleFerret 掃描可移除媒體（`/Volumes` 目錄中的掛載磁碟）中的檔案進行竊取。 |
| **T1074.001** | Data Staged: Local Data Staging | InvisibleFerret 將瀏覽器資料庫複製至暫存目錄（Windows 的 `%Temp%` 或其他系統的 `/tmp`）進行處理。使用 ZIP/7z 壓縮時也先在本機建立壓縮檔再上傳。`ssh_zcp` 指令將瀏覽器延伸套件資料放入暫存目錄的 staging 資料夾。 |
| **T1560.002** | Archive Collected Data: Archive via Library | InvisibleFerret 使用 Python 的 `py7zr` 和 `pyzipper` 套件壓縮竊取的資料。 |

---

## 10. Command and Control（命令與控制）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1071.001** | Application Layer Protocol: Web Protocols | BeaverTail 透過 HTTP POST 上傳竊取資料至 `/uploads` 端點。InvisibleFerret 透過 HTTP POST 上傳系統資訊至 `/keys` 端點、鍵盤側錄資料至 `/api/clip` 端點。 |
| **T1071.002** | Application Layer Protocol: File Transfer Protocols | InvisibleFerret 的 `ssh_upload` 與 `ssh_env` 指令使用 FTP 協定上傳檔案，FTP 伺服器位址與憑證由 C&C 指令參數指定。 |
| **T1095** | Non-Application Layer Protocol | InvisibleFerret 的後門使用 TCP socket 連線進行雙向通訊，以 JSON 格式傳送指令（包含 `command` 與 `args` 欄位）。 |
| **T1571** | Non-Standard Port | BeaverTail 與 InvisibleFerret 使用非標準通訊埠：**1224**、**1244**（HTTP C&C）、**1245**（TCP 後門），偶爾使用 80、2245、3000、3001、5000、5001。 |
| **T1219** | Remote Access Tools | InvisibleFerret 下載並安裝合法的 AnyDesk 遠端管理軟體，作為持續性機制與遠端存取工具。 |

---

## 11. Exfiltration（資料外洩）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1041** | Exfiltration Over C2 Channel | BeaverTail 透過 HTTP POST 將收集的資料（瀏覽器延伸套件檔案、Keychain、Solana 金鑰等）上傳至 C&C 的 `/uploads` 端點。InvisibleFerret 透過 `/keys` 端點上傳系統資訊與瀏覽器憑證。 |
| **T1567.004** | Exfiltration Over Web Service: Exfiltration Over Webhook | InvisibleFerret 新版 `ssh_zcp` 指令將竊取的瀏覽器延伸套件與密碼管理器資料壓縮後，透過 **Telegram Bot API** 上傳至指定的 Telegram 聊天室。 |
| **T1030** | Data Transfer Size Limits | InvisibleFerret 在部分情況下限制外洩檔案大小：`sdir` 指令僅上傳小於 100 MB 的檔案、`ssh_env` 指令僅上傳小於 20 MB 的檔案。 |

---

## 12. Impact（影響）

| ID | 技術名稱 | 攻擊行動中的具體應用 |
|---|---|---|
| **T1657** | Financial Theft | 此攻擊行動的**主要目標**為竊取加密貨幣。攻擊者針對加密貨幣錢包延伸套件（MetaMask、Phantom、Coinbase 等 88+ 個）、桌面錢包（Atomic、Exodus、Electrum）、Solana 金鑰檔案進行竊取。InvisibleFerret 也竊取瀏覽器儲存的信用卡資訊。 |

---

## 攻擊鏈摘要（Kill Chain）

```
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Resource Development                                                │
│     建立假招募帳號 → 開發 BeaverTail/InvisibleFerret → 租用 VPS          │
├─────────────────────────────────────────────────────────────────────────┤
│  2. Initial Access                                                      │
│     透過求職平台 Spearphishing → 提供木馬化專案或假會議軟體               │
├─────────────────────────────────────────────────────────────────────────┤
│  3. Execution                                                           │
│     受害者執行專案 → 觸發 BeaverTail（JS/原生）                          │
├─────────────────────────────────────────────────────────────────────────┤
│  4. Defense Evasion                                                     │
│     程式碼隱藏於長註解後 → Base64/XOR 混淆 → 隱藏檔案與視窗              │
├─────────────────────────────────────────────────────────────────────────┤
│  5. Credential Access & Collection                                      │
│     竊取瀏覽器憑證/Keychain/加密貨幣錢包 → 鍵盤側錄 → 剪貼簿竊取         │
├─────────────────────────────────────────────────────────────────────────┤
│  6. C&C & Exfiltration                                                  │
│     HTTP/TCP/FTP 多通道通訊 → Telegram webhook 外洩 → 非標準通訊埠       │
├─────────────────────────────────────────────────────────────────────────┤
│  7. Persistence & Lateral Movement                                      │
│     安裝 AnyDesk → 寫入攻擊者憑證 → 遠端桌面持續存取                     │
├─────────────────────────────────────────────────────────────────────────┤
│  8. Impact                                                              │
│     加密貨幣竊取 → 信用卡資訊竊取 → Financial Theft                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 參考資料

- ESET Research: [DeceptiveDevelopment targets freelance developers](https://www.welivesecurity.com/en/eset-research/deceptivedevelopment-targets-freelance-developers/)
- MITRE ATT&CK Framework v16: https://attack.mitre.org/
- 相關活動群集：Contagious Interview、DEV#POPPER、Moonstone Sleet、Lazarus Operation DreamJob
