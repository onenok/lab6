## 快速導覽 / 要讓 AI 代理人馬上有用的重點

以下說明以繁體中文呈現，聚焦「本專案的結構、重要約定、測試／執行流程，以及改動時常見的注意事項」。請直接依照檔案與例子修改程式，而不要做廣泛假設。

- 專案類型: 傳統 PHP + MySQL 網站，所有頁面為伺服器端渲染 (no frameworks)。
- 本地啟動: 原始 `readme.md` 指示使用 Apache + MySQL，並匯入 `lab6.sql`（若無 Apache，可用 PHP 內建伺服器測試：`php -S localhost:8000` 並把專案根目錄設為 document root）。
- 資料庫連線設定: 在 `connect.php`，變數 `\$servername`, `\$username`, `\$password`, `\$db_name`。任何修改 DB credential 或預設 DB 名稱時請同步編輯此檔。

## 主要檔案與責任分工（read these first）
- `connect.php`：提供 `safeQuery($sql, $types=null, $params=[])`，回傳一個 object，常見屬性：`success` (bool), `result` (mysqli_result|null), `affected_rows`, `insert_id`, `error`。
- `insert.php`：註冊處理（POST）。目前寫入 `member` 表，會 `password_hash()` 密碼。
- `search.php`：登入處理（POST），會讀 `member` 表後用 `password_verify()` 驗證。
- `update.php`：修改帳號（POST），負責驗證舊密碼、檢查衝突、更新 `member` 資料，更新後會 `unset($_SESSION['login'])` 要求重新登入。
- `delete.php`：刪除帳號（POST），驗證密碼後刪除 `member`。
- `signup.php`, `login.php`, `editAccount.php`, `cancellation.php`, `index.php`：使用者介面頁面，均以 `?msg=...` 方式傳遞錯誤或狀態，檔案內有 `messages` 陣列對應顯示文字。

## 資料庫與 schema 要點
- 目前主要使用表：`member`, `products`, `cart`（你已經讀過 `member` 的 CREATE TABLE 定義）。
- 請注意：早期程式碼曾使用 `login` / `loginname`，修改 schema 時需把所有 SQL 改為 `member` / `member_id`。搜尋關鍵字：`FROM login`、`loginname` 來確認是否漏改。

## 常見程式模式與約定（請遵循）
- 所有以表單提交的處理頁面會先檢查 `$_SERVER['REQUEST_METHOD'] !== 'POST'`，若非 POST 通常 redirect 回表單並帶 `?msg=try_to_access_directly`。
- Session key：登入後會將 `$_SESSION['login']` 設為 `member_id`（字串）。其他頁面以此判定登入狀態。
- SQL 使用 `safeQuery`，請不要直接使用 `mysqli_query`。若需要讀結果，使用 `$res->result->fetch_assoc()`。
- 密碼處理：新密碼請用 `password_hash($pwd, PASSWORD_DEFAULT)` 儲存；登入與驗證請用 `password_verify($inputPwd, $storedHash)`。

## 請求／回應與 UI 約定
- 表單端點（範例）：`signup.php` -> `insert.php`，`login.php` -> `search.php`，`editAccount.php` -> `update.php`，`cancellation.php` -> `delete.php`。
- 所有錯誤或狀態傳遞以 URL query `?msg=KEY`，接收頁面會用一個對照陣列把 KEY 轉成人類顯示訊息，請在修改訊息鍵時同步更新各頁面的對照陣列。

## 開發者工作流程 / commit 風格
- 專案傾向使用簡潔的 conventional-style commit，但格式要求為 `type: desc`（例如 `feat: migrate auth to member table and use password_hash/password_verify`）。
- 常用 types：`feat`（功能/行為變更）、`fix`（安全修正）、`chore`（維護）。使用者偏好以 `type: desc` 單行訊息。

## 立即可用的代碼範例（搜尋與驗證）
safeQuery
```php
function safeQuery($sql, $types = null, $params = []) { }
@param mixed $sql
@param mixed $types
@param array $params
@return \stdClass
```
示例：安全的登入查詢
```php
$sql = "SELECT * FROM member WHERE member_id = ?";
$res = safeQuery($sql, "s", [$name]);
if ($res->result && $row = $res->result->fetch_assoc()) {
    if (password_verify($pwd, $row['pwd'])) {
        $_SESSION['login'] = $row['member_id'];
    }
}
```

示例：使用 `safeQuery` 取得 affected rows
```php
$insert = safeQuery("INSERT INTO member(member_id,pwd,member_name) VALUES(?,?,?)","sss",[$id,$hash,$display]);
if ($insert->affected_rows > 0) { /* success */ }
```

## database structure
```sql
CREATE TABLE `member` (
  `member_id` varchar(50) NOT NULL COMMENT '登入帳號，註冊後不可更改',
  `pwd` varchar(255) NOT NULL COMMENT '密碼（建議使用雜湊）',
  `member_name` varchar(100) NOT NULL COMMENT '顯示名稱，可重複',
  `member_telno` varchar(20) DEFAULT NULL COMMENT '電話號碼，可為空',
  `member_addr` text DEFAULT NULL COMMENT '地址，可為空',
  `created_at` datetime DEFAULT current_timestamp(),
  `updated_at` datetime DEFAULT current_timestamp() ON UPDATE current_timestamp(),
  PRIMARY KEY (`member_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='會員資料表'

CREATE TABLE `cart` (
  `cart_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '購物車項目唯一編號',
  `member_id` varchar(50) NOT NULL COMMENT '會員帳號',
  `product_id` int(11) NOT NULL COMMENT '商品編號（參考 products 表）',
  `snapshot_name` varchar(100) NOT NULL COMMENT '加入時的商品名稱',
  `snapshot_price` decimal(10,2) NOT NULL COMMENT '加入時的價格',
  `snapshot_description` text DEFAULT NULL COMMENT '加入時的描述（可選）',
  `snapshot_type` enum('drinks','food','toy','e-things') NOT NULL COMMENT '加入時的類型',
  `qty` int(11) unsigned NOT NULL DEFAULT 1 COMMENT '數量',
  `created_at` datetime DEFAULT current_timestamp() COMMENT '加入時間',
  `updated_at` datetime DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '最後修改時間',
  PRIMARY KEY (`cart_id`),
  UNIQUE KEY `uk_member_product` (`member_id`,`product_id`),
  KEY `product_id` (`product_id`),
  CONSTRAINT `cart_ibfk_1` FOREIGN KEY (`member_id`) REFERENCES `member` (`member_id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `cart_ibfk_2` FOREIGN KEY (`product_id`) REFERENCES `products` (`product_id`) ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='購物車（快照版）'

CREATE TABLE `products` (
  `product_id` int(11) NOT NULL AUTO_INCREMENT COMMENT '商品唯一編號',
  `product_name` varchar(100) NOT NULL COMMENT '商品名稱',
  `type` enum('drinks','food','toy','e-things') NOT NULL COMMENT '商品類型',
  `supplier` varchar(80) DEFAULT NULL COMMENT '供應商',
  `description` text DEFAULT NULL COMMENT '商品描述',
  `price` decimal(10,2) NOT NULL COMMENT '售價',
   `qty` int(11) unsigned NOT NULL DEFAULT 1 COMMENT 'Stock level of product',
  `updated_at` datetime DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '最後修改時間',
  `created_at` datetime DEFAULT current_timestamp() COMMENT '建立時間',
  PRIMARY KEY (`product_id`)
) ENGINE=InnoDB AUTO_INCREMENT=21 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='商品資料表'
```

## 風險點 & 探查清單（修改時要檢查）
- 搜尋專案中是否仍有 `login` / `loginname` 等舊欄位名稱殘留。
- 當你變更 `connect.php` 的 DB 設定，請確認 `readme.md` 中的啟動指南或部署腳本同步更新。
- 表單驗證目前偏向前端簡易檢查，後端請務必再做完整驗證（非本檔範例的範圍，但若要修改請跟我說）。

## 如果你是 AI 代理人，下一步可以做的安全、低風險修改
- 在 `signup.php` 與 `editAccount.php` 表單加入 `member_name`, `member_telno`, `member_addr` 欄位，並在 `insert.php` / `update.php` 處理這些欄位。
- 用 `grep` 搜尋所有 `FROM login` 或 `loginname`，把它們改為 `member` / `member_id`。

---
若希望我合併此檔案到 `.github/copilot-instructions.md`（或把內容改為英文），或要我把 README 補成更詳細的本機啟動指令，我可以立刻更新，請回覆要調整的部分即可。
