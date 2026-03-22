# 瀏覽器錢包與 DApp 完整溝通流程

## 目錄

1. [名詞說明](#名詞說明)
2. [擴充套件內部架構](#擴充套件內部架構)
3. [底層通訊協議](#底層通訊協議)
4. [連接請求流程 eth_requestAccounts](#連接請求流程-eth_requestaccounts)
5. [交易簽名流程 eth_sendTransaction](#交易簽名流程-eth_sendtransaction)
6. [訊息簽名（非交易）](#訊息簽名非交易)
7. [事件訂閱系統](#事件訂閱系統)
8. [權限系統 EIP-2255](#權限系統-eip-2255)
9. [金鑰系統與加密機制](#金鑰系統與加密機制)
10. [安全邊界總結](#安全邊界總結)

---

## 名詞說明

**DApp（Decentralized Application）**
去中心化應用程式。核心邏輯寫在智能合約上，部署至區塊鏈後自動執行，無法被任何人單方面修改或關閉。使用者透過錢包地址（而非帳號密碼）進行身份驗證。

**瀏覽器錢包擴充套件**
安裝在瀏覽器中的擴充套件，負責管理私鑰、顯示確認視窗、與區塊鏈網路溝通。常見有 MetaMask、Phantom、Rabby、Coinbase Wallet、OKX Wallet 等。

**Provider（`window.ethereum`）**
錢包擴充套件注入至每個網頁的 JavaScript 物件，是 DApp 與錢包溝通的唯一介面，規格由 EIP-1193 定義。

**RPC 節點**
錢包本身不運行完整區塊鏈節點，而是透過 RPC（Remote Procedure Call）節點服務（如 Infura、Alchemy）與鏈溝通。

---

## 擴充套件內部架構

瀏覽器中存在三個互相隔離的執行環境，各有不同的權限與職責。

### 環境一：DApp 分頁（網頁 JS 環境）

- 一般網頁的 JavaScript context，由 DApp 網站自身的程式碼組成
- 可存取 `window.ethereum`（由 Content Script 注入）
- 無法直接讀取擴充套件內部狀態或私鑰

### 環境二：Content Script（注入腳本）

- 由擴充套件注入至每個分頁，與 DApp 共享 DOM，但擁有獨立的 JS scope
- 提供 `window.ethereum` 這個 Provider 物件給 DApp 使用
- 作為 DApp 與擴充套件環境之間的傳話者
- 透過 `chrome.runtime.sendMessage()` 與 Background SW 通訊，不能直接呼叫對方函數

### 環境三：擴充套件環境（Extension Process，完全隔離）

包含兩個子元件：

**Background Service Worker（背景服務工作者）**
- 常駐背景的核心邏輯處理器
- 負責：金鑰管理、RPC 廣播、請求排隊、權限 store 管理
- 接收來自 Content Script 的 JSON-RPC 請求並處理
- 從加密金鑰庫解密取得私鑰後進行簽名，用完立即清除記憶體

**Popup UI（彈出視窗）**
- 使用者點擊錢包 icon 或需要確認操作時彈出
- 顯示待確認的請求內容（連接授權、交易詳情、簽名內容）
- 透過 `chrome.runtime.sendMessage()` 與 Background SW 通訊

**加密金鑰庫（Keystore）**
- 私鑰以 AES-256-GCM 加密後存放於本機磁碟
- 解密時需要使用者輸入的密碼
- 私鑰永遠不離開本機裝置

---

## 底層通訊協議

### JSON-RPC 2.0

所有 DApp 與錢包的溝通都基於 JSON-RPC 2.0 格式。每個請求的結構如下：

```json
{
  "jsonrpc": "2.0",
  "method": "eth_sendTransaction",
  "params": [
    {
      "from": "0xAbCd...1234",
      "to": "0xDeFg...5678",
      "value": "0xDE0B6B3A7640000",
      "data": "0x..."
    }
  ],
  "id": 1
}
```

回應格式：

```json
{
  "jsonrpc": "2.0",
  "result": "0xtxhash...",
  "id": 1
}
```

### 訊息傳遞路徑

```
DApp 網頁
  ↓  window.ethereum.request(payload)
Content Script
  ↓  chrome.runtime.sendMessage({ payload, origin })
Background Service Worker
  ↓  處理邏輯（查詢 nonce、估算 gas、解密私鑰、簽名）
  ↓  HTTP POST to RPC endpoint (Infura / Alchemy / 自架節點)
區塊鏈網路
```

---

## 連接請求流程 eth_requestAccounts

這是 DApp 首次請求取得使用者錢包地址的流程，遵循 EIP-1193 標準。

### 觸發方式

使用者點擊 DApp 上的「Connect Wallet」按鈕，DApp 執行：

```javascript
const accounts = await window.ethereum.request({
  method: 'eth_requestAccounts'
});
// accounts = ['0xAbCd...1234']
```

### 完整步驟

1. **DApp 呼叫** `window.ethereum.request({ method: 'eth_requestAccounts' })`
2. **Content Script 轉發** 呼叫 `chrome.runtime.sendMessage()` 將請求連同 `origin`（如 `https://app.uniswap.org`）一起送至 Background SW
3. **Background SW 建立 pending request** 將請求放入等待佇列
4. **Popup 彈出** 顯示請求連接的 DApp origin、要求取得的帳戶地址
5. **使用者決策**：
   - **拒絕** → 回傳 `Error: { code: 4001, message: 'User rejected request' }`
   - **允許** → Background SW 將此 origin 記錄於 permissions store，回傳地址陣列 `['0xAbCd...1234']`

### 權限記憶機制

連接成功後，`https://app.uniswap.org` 這個 origin 會被記錄在 Background SW 的 permissions store 中。下次同一網站再呼叫 `eth_accounts` 時，不需要彈窗重新授權，直接回傳地址。

可隨時在錢包 UI 中「中斷連接（Disconnect）」特定網站，等同呼叫 `wallet_revokePermissions`。

---

## 交易簽名流程 eth_sendTransaction

這是最核心的流程，涵蓋從 DApp 發起到交易上鏈的完整過程。

### 觸發方式

```javascript
const txHash = await window.ethereum.request({
  method: 'eth_sendTransaction',
  params: [{
    from: '0xAbCd...1234',    // 發送方（必須是已連接帳戶）
    to: '0xDeFg...5678',      // 收款方
    value: '0xDE0B6B3A7640000', // 金額（wei，16進位）
    data: '0x...',            // 合約呼叫資料（若非合約互動則省略）
    gas: '0x5208',            // Gas limit（可選，錢包可自動估算）
    gasPrice: '0x...'         // Gas price（或改用 EIP-1559 的 maxFeePerGas）
  }]
});
```

### 完整步驟

**步驟 1：DApp 呼叫 eth_sendTransaction**
傳入交易參數物件。

**步驟 2：Background SW 解析與補全參數**
- 若未提供 `nonce`，向 RPC 節點查詢 `eth_getTransactionCount`
- 若未提供 `gas`，向 RPC 節點呼叫 `eth_estimateGas` 估算
- 若使用 EIP-1559，查詢 `eth_maxPriorityFeePerGas` 等費用建議值

**步驟 3：Popup 彈出確認視窗**
顯示以下資訊供使用者審閱：
- 收款方地址
- 轉帳金額
- 手續費（Gas fee）預估
- 若為合約互動，顯示函數名稱與參數（如「Swap 0.1 ETH for USDC」）
- 安全警告（如：此合約未經驗證、交易模擬結果）

**步驟 4：使用者確認，Background SW 進行簽名**
- 從 Keystore 解密取出私鑰（Ethereum 使用 secp256k1 曲線；Solana 使用 Ed25519）
- 對交易資料進行雜湊（Keccak-256）
- 用私鑰產生數位簽章，得到 `r`、`s`、`v` 三個值
- 私鑰從記憶體中清除

**步驟 5：RLP 編碼並廣播**
- 將交易欄位（nonce, gasPrice, gasLimit, to, value, data, v, r, s）以 RLP 格式序列化
- 呼叫 RPC 節點的 `eth_sendRawTransaction`，傳入序列化後的 hex 字串
- RPC 節點將交易廣播至區塊鏈網路的 Mempool

**步驟 6：回傳交易雜湊**
```javascript
// DApp 收到
txHash = '0xabc123...'
// 可用此 hash 查詢交易狀態：eth_getTransactionReceipt
```

**步驟 7：等待打包**
礦工（PoW）或驗證者（PoS）從 Mempool 選取交易打包進區塊，交易被包含後狀態從 pending 變為 confirmed。

---

## 訊息簽名（非交易）

不移動資金、不消耗 Gas，純粹用於身份驗證或授權。

### personal_sign

簽署任意文字字串，常用於登入驗證（如：「Sign in with Ethereum」）：

```javascript
const signature = await window.ethereum.request({
  method: 'personal_sign',
  params: ['0x' + Buffer.from('Login at 2024-01-01T00:00:00Z').toString('hex'), account]
});
```

錢包會在訊息前自動加上前綴 `"\x19Ethereum Signed Message:\n"` 再進行雜湊，防止惡意 DApp 騙使用者簽署真實交易。

### eth_signTypedData_v4（EIP-712）

簽署結構化資料，UI 上顯示格式化欄位，安全性較高，使用場景包括：
- OpenSea 的上架訂單
- Uniswap 的 `permit` 授權（允許合約動用代幣，不需先送 `approve` 交易）
- Gasless 交易的 meta-transaction

```javascript
const signature = await window.ethereum.request({
  method: 'eth_signTypedData_v4',
  params: [account, JSON.stringify({
    domain: { name: 'Uniswap', version: '1', chainId: 1, verifyingContract: '0x...' },
    types: { Permit: [{ name: 'owner', type: 'address' }, ...] },
    message: { owner: account, spender: '0x...', value: '1000000', nonce: 0, deadline: 9999999999 }
  })]
});
```

---

## 事件訂閱系統

DApp 可監聽錢包的狀態變化，Background SW 偵測到變動時主動推送事件。

```javascript
// 帳戶切換（使用者在錢包中切換帳號）
window.ethereum.on('accountsChanged', (accounts) => {
  if (accounts.length === 0) {
    // 使用者鎖定錢包或中斷連接
  } else {
    // accounts[0] 是新的當前帳號
    currentAccount = accounts[0];
  }
});

// 網路切換（如 Ethereum 主網 → Polygon）
window.ethereum.on('chainChanged', (chainId) => {
  // chainId 為 16 進位字串，如 '0x1' = Ethereum, '0x89' = Polygon
  // 通常需要重新整理頁面
  window.location.reload();
});

// 首次連接成功
window.ethereum.on('connect', ({ chainId }) => {
  console.log('Connected to chain:', chainId);
});

// 斷線（RPC 節點無法連線等）
window.ethereum.on('disconnect', (error) => {
  console.error('Disconnected:', error);
});
```

### 推送機制

Background SW 偵測到帳戶或網路變動後，透過 Content Script 向所有已連接的 DApp 分頁廣播事件。Content Script 呼叫注入進網頁的 EventEmitter，觸發 DApp 已綁定的 listener。

---

## 權限系統 EIP-2255

`eth_requestAccounts` 本質上是申請 `eth_accounts` 這個 permission。Background SW 維護一個 permissions store，記錄每個 origin 被授予的權限。

### 查詢當前權限

```javascript
const permissions = await window.ethereum.request({
  method: 'wallet_getPermissions'
});
// 回傳：[{ invoker: 'https://app.uniswap.org', parentCapability: 'eth_accounts', ... }]
```

### 申請權限

```javascript
await window.ethereum.request({
  method: 'wallet_requestPermissions',
  params: [{ eth_accounts: {} }]
});
```

### 撤銷權限

```javascript
await window.ethereum.request({
  method: 'wallet_revokePermissions',
  params: [{ eth_accounts: {} }]
});
```

---

## 金鑰系統與加密機制

### 助記詞 → 私鑰 → 公鑰 → 地址

```
助記詞（12/24 個英文單字）
  ↓  BIP-39：單字 → 512-bit seed
  ↓  BIP-32：HD 錢包，seed → 主金鑰
  ↓  BIP-44：路徑衍生，m/44'/60'/0'/0/0 = 第一個 ETH 帳戶
私鑰（32 bytes 隨機數，secp256k1）
  ↓  橢圓曲線乘法（不可逆）
公鑰（65 bytes 未壓縮 / 33 bytes 壓縮）
  ↓  Keccak-256 雜湊，取後 20 bytes
地址（0x 開頭，20 bytes，EIP-55 混合大小寫校驗）
```

Solana 使用 Ed25519 曲線，公鑰直接等於錢包地址（Base58 編碼，約 44 字元），不需要額外的雜湊轉換。

### 私鑰的本地加密儲存

```
使用者密碼
  ↓  PBKDF2 / scrypt（key stretching，防暴力破解）
加密金鑰（Derived Key）
  ↓  AES-256-GCM 加密私鑰
加密後的 Keystore JSON（存本機磁碟）
```

MetaMask 的 Keystore 格式（vault）儲存在瀏覽器的 `chrome.storage.local`，格式遵循 Web3 Secret Storage Definition（EIP-55）。

---

## 安全邊界總結

| 元件 | 可存取私鑰 | 可讀取帳戶地址 | 可發送交易 |
|------|-----------|--------------|-----------|
| DApp 網頁 | 不可 | 授權後可 | 需使用者確認 |
| Content Script | 不可 | 不可 | 不可 |
| Background SW | 暫時可（簽名後清除）| 可 | 需 Popup 確認 |
| Popup UI | 不可 | 可 | 使用者確認後觸發 |
| RPC 節點 | 不可 | 可（IP + 查詢行為）| 只能看到已簽名交易 |

### 核心安全機制

**私鑰不離開本機**
私鑰以 AES-256-GCM 加密存放本機，簽名在本地完成，只有已簽名的交易資料（hex 字串）會被送出。

**人工確認防線**
每一個涉及金錢移動或簽名的操作，都必須通過 Popup 讓使用者手動確認。這是整個安全模型最關鍵的防線，任何 DApp 都無法繞過。

**Origin 隔離**
Content Script 在轉發請求時會附上 origin，Background SW 據此判斷請求來源是否已授權，不同網站之間完全隔離。

**簽名前綴機制**
`personal_sign` 自動加上前綴，防止惡意 DApp 誘使使用者簽署真實交易格式的資料。EIP-712 進一步要求結構化格式，讓使用者能看懂自己在簽什麼。

### 常見風險與注意事項

- **RPC 隱私**：RPC 服務商（Infura 等）可看到你的 IP 與鏈上查詢行為，可自架節點或使用隱私 RPC 服務降低風險
- **惡意 approve**：`approve` 合約呼叫若授予無限金額（`uint256 max`），合約可隨時提走你的代幣，交易前需仔細確認
- **釣魚網站**：模仿真實 DApp 的網站會顯示相同的 Popup，但 `to` 地址不同，交易前務必核對收款方地址
- **定期撤銷授權**：可使用 Revoke.cash 等工具檢查並撤銷不再使用的合約授權
