[English](https://github.com/Anan-up/message-board/blob/main/README_English.md) | [简中](https://github.com/Anan-up/message-board/blob/main/README_Simplified_Chinese.md) | [文言](https://github.com/Anan-up/message-board/blob/main/README_Classical_Chinese.md)

# 自零構築功能完備之留言板網站

> 一份完備之部署指南，含源碼、配置之說明及常見疑難之排解

## 📌 引言

方今互聯之世，留言板猶爲極具價值之交互之器。無論個人博客、小型社羣，抑或企業內之通聯，一簡潔而功能完備之留言板，皆能大展其用。

本文將引君自零而始，於 Linux 服務器之上部署一功能豐贍之留言板網站。此項目取 **Nginx + PHP + SQLite** 之輕量架構，無須另裝 MySQL，所耗資源甚微，宜部署於雲服務器，乃至樹莓派之上。

**最終之效，預覽如左：**

- 可發留言（含暱稱、內容、附件上傳）
- 圖片/視頻自動辨識而示之，餘者供下載
- 點贊無上限（長按連贊，兼有心形飄落之效）
- 留言可排序（最新 / 熱門）
- 管理員可置頂、刪除
- 訪客統計（IP、設備、瀏覽之次、點贊之總數）
- 每日一言（每刷則隨機而示）
- 總訪問量之統計
- 草稿自動保存
- 背景音樂播放器
- 全站 HTTPS 之支持

---

## 🛠 技術棧

| 組件 | 版本 | 說明 |
|------|------|------|
| 操作系統 | Debian 13.2 | 亦可兼容 Ubuntu 22.04+ |
| Web 服務器 | Nginx 1.24+ | 高性能 HTTP 服務 |
| PHP | PHP 8.3 / 8.4 | 後端邏輯之處理 |
| 數據庫 | SQLite 3 | 輕量級文件數據庫 |
| 前端 | 原生 HTML + CSS + JavaScript | 無須任何框架 |

---

## 📦 環境要求

- 一臺 Linux 服務器（建議 Debian 12+ 或 Ubuntu 22.04+）
- 一個已解析至服務器 IP 之域名（用於 HTTPS）
- SSL 證書（本教程用已有證書，亦可用 Let's Encrypt 免費簽發）

---

## 🚀 第一步：環境準備

登錄服務器後，先更新系統，並安裝所需之軟件包：

```bash
# 更新系統
sudo apt update && sudo apt upgrade -y

# 安裝 Nginx、PHP 及相關擴展
sudo apt install -y nginx php-fpm php-sqlite3 sqlite3 curl wget unzip

# 查看 PHP 版本（後續配置需要）
php -v
```

---

## 📁 第二步：創建網站目錄

```bash
# 創建網站根目錄
sudo mkdir -p /var/www/your_domain

# 創建上傳目錄（用於存放用戶上傳的附件）
sudo mkdir -p /var/www/your_domain/uploads

# 設置權限
sudo chown -R www-data:www-data /var/www/your_domain
sudo chmod 755 /var/www/your_domain/uploads
```

> **說明**：`your_domain` 請易爲汝之實際域名。

---

## 🔐 第三步：準備 SSL 證書（可選而推薦）

若已有證書文件（`.crt` 與 `.key`），置之服務器上（如 `/path/to/ssl/`）。若無，可用 Let's Encrypt 免費簽發：

```bash
# 安裝 Certbot
sudo apt install -y certbot python3-certbot-nginx

# 自動簽發證書（域名需要已解析）
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
```

---

## 📄 第四步：Nginx 站點配置

創建 Nginx 虛擬主機配置文件：

```bash
sudo nano /etc/nginx/sites-available/your_domain
```

將下列內容粘貼其中（依實際情況改域名與證書路徑）：

```nginx
# HTTP -> HTTPS 重定向
server {
    listen 80;
    listen [::]:80;
    server_name your_domain.com www.your_domain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS 主站點
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name your_domain.com www.your_domain.com;

    root /var/www/your_domain;
    index index.php index.html;

    # SSL 證書配置
    ssl_certificate     /path/to/ssl/your_domain_bundle.crt;
    ssl_certificate_key /path/to/ssl/your_domain.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # 日誌（可選）
    access_log /var/log/nginx/your_domain.access.log;
    error_log  /var/log/nginx/your_domain.error.log;

    # 上傳目錄：禁止執行 PHP
    location /uploads/ {
        location ~ \.php$ {
            deny all;
        }
    }

    # 放開上傳大小限制（建議與 PHP 配置一致）
    client_max_body_size 10G;

    # 處理 PHP 請求
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 600s;
    }

    # 禁止訪問隱藏文件
    location ~ /\. {
        deny all;
    }
}
```

啓用站點並重載 Nginx：

```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🐘 第五步：調整 PHP 配置

編輯 PHP-FPM 配置文件（路徑或因版本而異）：

```bash
sudo nano /etc/php/8.4/fpm/php.ini
```

改下列參數（以適大文件上傳）：

```ini
upload_max_filesize = 10G
post_max_size = 10G
max_execution_time = 600
max_input_time = 600
memory_limit = 512M
```

重啓 PHP-FPM：

```bash
sudo systemctl restart php8.4-fpm
```

---

## 📄 第六步：完整源碼 `index.php`

此乃項目之核心文件，留言板之一切功能盡在其中。存之爲 `/var/www/your_domain/index.php`：

```php
<?php
date_default_timezone_set('Asia/Shanghai');
session_start();

// ========== 數據庫初始化 ==========
$db_file = 'messages.db';
$db = new SQLite3($db_file);

// 創建留言表（如果不存在）
$db->exec("CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    content TEXT NOT NULL,
    media_path TEXT,
    ip_address TEXT,
    user_agent TEXT,
    likes INTEGER DEFAULT 0,
    is_pinned INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
)");

// 檢查並添加缺失的列（兼容舊庫）
$cols = $db->query("PRAGMA table_info(messages)");
$has_ip = false; $has_ua = false; $has_likes = false; $has_pinned = false;
while ($col = $cols->fetchArray(SQLITE3_ASSOC)) {
    if ($col['name'] === 'ip_address') $has_ip = true;
    if ($col['name'] === 'user_agent') $has_ua = true;
    if ($col['name'] === 'likes') $has_likes = true;
    if ($col['name'] === 'is_pinned') $has_pinned = true;
}
if (!$has_ip) $db->exec("ALTER TABLE messages ADD COLUMN ip_address TEXT");
if (!$has_ua) $db->exec("ALTER TABLE messages ADD COLUMN user_agent TEXT");
if (!$has_likes) $db->exec("ALTER TABLE messages ADD COLUMN likes INTEGER DEFAULT 0");
if (!$has_pinned) $db->exec("ALTER TABLE messages ADD COLUMN is_pinned INTEGER DEFAULT 0");

$cols = $db->query("PRAGMA table_info(messages)");
$has_media = false;
while ($col = $cols->fetchArray(SQLITE3_ASSOC)) {
    if ($col['name'] === 'media_path') $has_media = true;
}
if (!$has_media) $db->exec("ALTER TABLE messages ADD COLUMN media_path TEXT");

// ========== 創建訪客日誌表 ==========
$db->exec("CREATE TABLE IF NOT EXISTS visitor_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ip_address TEXT NOT NULL,
    user_agent TEXT,
    action TEXT NOT NULL,
    message_id INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
)");

// ========== 管理員密碼 ==========
define('ADMIN_PASSWORD', 'your_password_here'); // 請修改爲強密碼

// ========== 處理管理員登錄 ==========
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['admin_password'])) {
    if ($_POST['admin_password'] === ADMIN_PASSWORD) {
        $_SESSION['admin'] = true;
        header('Location: ?login_success=1');
        exit;
    } else {
        $login_error = '密碼錯誤，請重試。';
    }
}

// ========== 處理管理員退出 ==========
if (isset($_GET['logout'])) {
    unset($_SESSION['admin']);
    session_destroy();
    header('Location: ?logout_success=1');
    exit;
}

// ========== 處理刪除留言 ==========
if (isset($_GET['delete']) && isset($_SESSION['admin']) && $_SESSION['admin'] === true) {
    $id = intval($_GET['delete']);
    if ($id > 0) {
        $stmt = $db->prepare("SELECT media_path FROM messages WHERE id = :id");
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $row = $result->fetchArray(SQLITE3_ASSOC);
        if ($row) {
            if (!empty($row['media_path'])) {
                $file_path = __DIR__ . '/' . $row['media_path'];
                if (file_exists($file_path)) {
                    unlink($file_path);
                }
            }
            $del = $db->prepare("DELETE FROM messages WHERE id = :id");
            $del->bindValue(':id', $id, SQLITE3_INTEGER);
            $del->execute();
        }
        header('Location: ?delete_success=1');
        exit;
    }
}

// ========== 處理置頂/取消置頂 ==========
if (isset($_GET['pin']) && isset($_SESSION['admin']) && $_SESSION['admin'] === true) {
    $id = intval($_GET['pin']);
    if ($id > 0) {
        $stmt = $db->prepare("UPDATE messages SET is_pinned = 1 WHERE id = :id");
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $stmt->execute();
    }
    header('Location: ?pin_success=1');
    exit;
}
if (isset($_GET['unpin']) && isset($_SESSION['admin']) && $_SESSION['admin'] === true) {
    $id = intval($_GET['unpin']);
    if ($id > 0) {
        $stmt = $db->prepare("UPDATE messages SET is_pinned = 0 WHERE id = :id");
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $stmt->execute();
    }
    header('Location: ?unpin_success=1');
    exit;
}

// ========== 處理點贊（AJAX） ==========
if (isset($_GET['ajax']) && $_GET['ajax'] === 'like' && isset($_GET['id'])) {
    header('Content-Type: application/json');
    $id = intval($_GET['id']);
    if ($id > 0) {
        $stmt = $db->prepare("UPDATE messages SET likes = likes + 1 WHERE id = :id");
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $stmt->execute();
        $res = $db->query("SELECT likes FROM messages WHERE id = $id");
        $row = $res->fetchArray(SQLITE3_ASSOC);
        $ip = $_SERVER['REMOTE_ADDR'];
        $ua = $_SERVER['HTTP_USER_AGENT'] ?? '';
        $logStmt = $db->prepare("INSERT INTO visitor_logs (ip_address, user_agent, action, message_id) VALUES (:ip, :ua, 'like', :mid)");
        $logStmt->bindValue(':ip', $ip, SQLITE3_TEXT);
        $logStmt->bindValue(':ua', $ua, SQLITE3_TEXT);
        $logStmt->bindValue(':mid', $id, SQLITE3_INTEGER);
        $logStmt->execute();
        echo json_encode(['status' => 'success', 'likes' => $row['likes']]);
    } else {
        echo json_encode(['status' => 'error', 'message' => '無效ID']);
    }
    exit;
}

// ========== 記錄頁面瀏覽 ==========
if (!isset($_GET['ajax']) && !isset($_GET['login_success']) && !isset($_GET['logout_success']) 
    && !isset($_GET['success']) && !isset($_GET['delete_success']) && !isset($_GET['like_success'])
    && !isset($_GET['page']) && !isset($_GET['pin_success']) && !isset($_GET['unpin_success'])) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $ua = $_SERVER['HTTP_USER_AGENT'] ?? '';
    $stmt = $db->prepare("INSERT INTO visitor_logs (ip_address, user_agent, action) VALUES (:ip, :ua, 'view')");
    $stmt->bindValue(':ip', $ip, SQLITE3_TEXT);
    $stmt->bindValue(':ua', $ua, SQLITE3_TEXT);
    $stmt->execute();
}

// ========== 輔助函數 ==========
function parse_user_agent($ua) {
    if (empty($ua)) return '未知設備';
    $os = '未知OS'; $browser = '未知瀏覽器';
    if (stripos($ua, 'Windows NT 10.0') !== false) $os = 'Windows 10';
    elseif (stripos($ua, 'Windows NT 6.3') !== false) $os = 'Windows 8.1';
    elseif (stripos($ua, 'Windows NT 6.2') !== false) $os = 'Windows 8';
    elseif (stripos($ua, 'Windows NT 6.1') !== false) $os = 'Windows 7';
    elseif (stripos($ua, 'Windows NT 6.0') !== false) $os = 'Windows Vista';
    elseif (stripos($ua, 'Windows NT 5.1') !== false) $os = 'Windows XP';
    elseif (stripos($ua, 'Mac OS X') !== false) $os = 'macOS';
    elseif (stripos($ua, 'iPhone') !== false) $os = 'iPhone';
    elseif (stripos($ua, 'iPad') !== false) $os = 'iPad';
    elseif (stripos($ua, 'Android') !== false) $os = 'Android';
    elseif (stripos($ua, 'Linux') !== false) $os = 'Linux';
    if (stripos($ua, 'Edg/') !== false) $browser = 'Edge';
    elseif (stripos($ua, 'OPR/') !== false || stripos($ua, 'Opera') !== false) $browser = 'Opera';
    elseif (stripos($ua, 'Firefox') !== false) $browser = 'Firefox';
    elseif (stripos($ua, 'Chrome') !== false && stripos($ua, 'Edg') === false) $browser = 'Chrome';
    elseif (stripos($ua, 'Safari') !== false && stripos($ua, 'Chrome') === false) $browser = 'Safari';
    elseif (stripos($ua, 'MSIE') !== false || stripos($ua, 'Trident') !== false) $browser = 'Internet Explorer';
    return $os . ' / ' . $browser;
}

function formatLocalTime($utc_string) {
    if (empty($utc_string)) return '未知';
    try {
        $dt = new DateTime($utc_string, new DateTimeZone('UTC'));
        $dt->setTimezone(new DateTimeZone('Asia/Shanghai'));
        return $dt->format('Y-m-d H:i');
    } catch (Exception $e) {
        return $utc_string;
    }
}

function formatFileSize($bytes) {
    if ($bytes >= 1073741824) return number_format($bytes / 1073741824, 2) . ' GB';
    if ($bytes >= 1048576) return number_format($bytes / 1048576, 2) . ' MB';
    if ($bytes >= 1024) return number_format($bytes / 1024, 2) . ' KB';
    return $bytes . ' B';
}

// ========== 每日一言（每次刷新隨機） ==========
function getDailyQuote() {
    $quotes = [
        ['生活不止眼前的苟且，還有詩和遠方。', '高曉松'],
        ['願你出走半生，歸來仍是少年。', '網絡'],
        ['世界以痛吻我，要我報之以歌。', '泰戈爾'],
        ['生如夏花之絢爛，死如秋葉之靜美。', '泰戈爾'],
        ['黑夜給了我黑色的眼睛，我卻用它尋找光明。', '顧城'],
        ['面朝大海，春暖花開。', '海子'],
        ['每一個不曾起舞的日子，都是對生命的辜負。', '尼采'],
        ['生活不是等待風暴過去，而是學會在雨中跳舞。', '網絡'],
        ['我們曾如此渴望命運的波瀾，到最後才發現，人生最曼妙的風景，竟是內心的淡定與從容。', '楊絳'],
        ['一個人可以被毀滅，但不能被打敗。', '海明威'],
        ['世上只有一種英雄主義，就是在認清生活真相之後依然熱愛生活。', '羅曼·羅蘭'],
        ['路漫漫其修遠兮，吾將上下而求索。', '屈原'],
        ['長風破浪會有時，直掛雲帆濟滄海。', '李白'],
        ['天生我材必有用，千金散盡還復來。', '李白'],
        ['安得廣廈千萬間，大庇天下寒士俱歡顏。', '杜甫'],
        ['採菊東籬下，悠然見南山。', '陶淵明'],
        ['沉舟側畔千帆過，病樹前頭萬木春。', '劉禹錫'],
        ['山重水複疑無路，柳暗花明又一村。', '陸游'],
        ['橫看成嶺側成峯，遠近高低各不同。', '蘇軾'],
        ['但願人長久，千里共嬋娟。', '蘇軾'],
        ['人有悲歡離合，月有陰晴圓缺。', '蘇軾'],
        ['不畏浮雲遮望眼，自緣身在最高層。', '王安石'],
        ['問渠那得清如許？爲有源頭活水來。', '朱熹'],
        ['紙上得來終覺淺，絕知此事要躬行。', '陸游'],
        ['寶劍鋒從磨礪出，梅花香自苦寒來。', '網絡'],
        ['業精於勤，荒於嬉；行成於思，毀於隨。', '韓愈'],
        ['書山有路勤爲徑，學海無涯苦作舟。', '韓愈'],
        ['三人行，必有我師焉。', '孔子'],
        ['學而不思則罔，思而不學則殆。', '孔子'],
        ['溫故而知新，可以爲師矣。', '孔子'],
        ['君子和而不同，小人同而不和。', '孔子'],
        ['君子坦蕩蕩，小人長慼慼。', '孔子'],
        ['窮則獨善其身，達則兼濟天下。', '孟子'],
        ['富貴不能淫，貧賤不能移，威武不能屈。', '孟子'],
        ['天行健，君子以自強不息。', '周易'],
        ['地勢坤，君子以厚德載物。', '周易'],
        ['疾風知勁草，板蕩識誠臣。', '李世民'],
        ['海闊憑魚躍，天高任鳥飛。', '網絡'],
        ['海內存知己，天涯若比鄰。', '王勃'],
        ['落霞與孤鶩齊飛，秋水共長天一色。', '王勃'],
        ['先天下之憂而憂，後天下之樂而樂。', '范仲淹'],
        ['不以物喜，不以己悲。', '范仲淹'],
        ['醉臥沙場君莫笑，古來征戰幾人回。', '王翰'],
    ];
    $index = mt_rand(0, count($quotes) - 1);
    return $quotes[$index];
}

// ========== 獲取當前訪客信息 ==========
$visitor_ip = $_SERVER['REMOTE_ADDR'];
$visitor_ua = $_SERVER['HTTP_USER_AGENT'] ?? '';
$visitor_device = parse_user_agent($visitor_ua);

// ========== 獲取總訪問量 ==========
$totalViews = $db->querySingle("SELECT COUNT(*) FROM visitor_logs WHERE action='view'");
if ($totalViews === null || $totalViews === false) $totalViews = 0;

// ========== 獲取每日一言 ==========
$dailyQuote = getDailyQuote();

// ========== 文件上傳處理 ==========
$upload_error = null;
define('MAX_FILE_SIZE', 10 * 1024 * 1024 * 1024);
$media_exts = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'mp4', 'webm', 'ogg'];
$allowed_exts = array_merge($media_exts, ['pdf', 'zip', 'rar', '7z', 'doc', 'docx', 'xls', 'xlsx', 'ppt', 'pptx', 'txt', 'csv', 'md', 'json', 'xml', 'mp3', 'wav', 'flac', 'aac']);
$blocked_exts = ['php', 'phtml', 'php3', 'php4', 'php5', 'phps', 'cgi', 'pl', 'py', 'sh', 'htaccess', 'htpasswd', 'exe', 'bat', 'cmd'];

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['name']) && isset($_POST['content']) && !isset($_POST['admin_password'])) {
    $name = SQLite3::escapeString(trim($_POST['name']));
    $content = SQLite3::escapeString(trim($_POST['content']));
    $media_path = null;
    $ip = $_SERVER['REMOTE_ADDR'];
    $user_agent = $_SERVER['HTTP_USER_AGENT'] ?? '';
    if (!empty($name) && !empty($content)) {
        if (isset($_FILES['media']) && $_FILES['media']['error'] !== UPLOAD_ERR_NO_FILE) {
            $file = $_FILES['media'];
            $upload_dir = __DIR__ . '/uploads/';
            if ($file['error'] !== UPLOAD_ERR_OK) {
                $upload_error = '文件上傳失敗，錯誤碼：' . $file['error'];
            } else {
                if ($file['size'] > MAX_FILE_SIZE) {
                    $upload_error = '文件大小超過 ' . (MAX_FILE_SIZE / 1024 / 1024 / 1024) . 'GB 限制。';
                } else {
                    $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
                    if (in_array($ext, $blocked_exts)) {
                        $upload_error = '不支持上傳此類型文件，請上傳其他格式。';
                    } else {
                        $finfo = finfo_open(FILEINFO_MIME_TYPE);
                        $mime = finfo_file($finfo, $file['tmp_name']);
                        finfo_close($finfo);
                        $new_name = time() . '_' . bin2hex(random_bytes(8)) . '.' . $ext;
                        $dest = $upload_dir . $new_name;
                        if (move_uploaded_file($file['tmp_name'], $dest)) {
                            $media_path = 'uploads/' . $new_name;
                        } else {
                            $upload_error = '文件保存失敗，請檢查目錄權限。';
                        }
                    }
                }
            }
        }
        if ($upload_error === null) {
            $stmt = $db->prepare("INSERT INTO messages (name, content, media_path, ip_address, user_agent) VALUES (:name, :content, :media, :ip, :ua)");
            $stmt->bindValue(':name', $name, SQLITE3_TEXT);
            $stmt->bindValue(':content', $content, SQLITE3_TEXT);
            $stmt->bindValue(':media', $media_path, SQLITE3_TEXT);
            $stmt->bindValue(':ip', $ip, SQLITE3_TEXT);
            $stmt->bindValue(':ua', $user_agent, SQLITE3_TEXT);
            $stmt->execute();
            header('Location: ?success=1');
            exit;
        }
    } else {
        $upload_error = '暱稱和內容不能爲空。';
    }
}

// ========== 獲取排序參數 ==========
$sort = isset($_GET['sort']) && in_array($_GET['sort'], ['latest', 'hot']) ? $_GET['sort'] : 'latest';

// ========== 獲取留言列表 ==========
$orderBy = $sort === 'latest' ? 'created_at DESC' : 'likes DESC';
$query = "SELECT * FROM messages ORDER BY is_pinned DESC, $orderBy";
$result = $db->query($query);
$messages = [];
while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
    $messages[] = $row;
}

$is_admin = isset($_SESSION['admin']) && $_SESSION['admin'] === true;

// ========== 獲取訪客統計數據 ==========
$logsQuery = $db->query("SELECT * FROM visitor_logs ORDER BY created_at DESC LIMIT 2000");
$rawLogs = [];
while ($row = $logsQuery->fetchArray(SQLITE3_ASSOC)) {
    $rawLogs[] = $row;
}

$statsMap = [];
foreach ($rawLogs as $log) {
    $ip = $log['ip_address'];
    if (!isset($statsMap[$ip])) {
        $statsMap[$ip] = [
            'ip' => $ip,
            'user_agent' => $log['user_agent'],
            'view_count' => 0,
            'like_count' => 0,
            'last_time' => $log['created_at'],
            'last_action' => $log['action'],
            'last_message_id' => $log['message_id'] ?? null
        ];
    }
    if ($log['action'] === 'view') {
        $statsMap[$ip]['view_count']++;
    } elseif ($log['action'] === 'like') {
        $statsMap[$ip]['like_count']++;
    }
}

$visitorStats = array_values($statsMap);
usort($visitorStats, function($a, $b) {
    return strtotime($b['last_time']) - strtotime($a['last_time']);
});

$totalStats = count($visitorStats);
$perPage = 20;
$currentPage = 1;
if ($is_admin) {
    $currentPage = isset($_GET['page']) ? max(1, intval($_GET['page'])) : 1;
    $totalPages = ceil($totalStats / $perPage);
    if ($currentPage > $totalPages && $totalPages > 0) $currentPage = $totalPages;
    $start = ($currentPage - 1) * $perPage;
    $displayStats = array_slice($visitorStats, $start, $perPage);
} else {
    $displayStats = array_slice($visitorStats, 0, 5);
}
?>
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=5.0">
    <title>✨ 安然小棧 · 留言板</title>
    <style>
        /* ======================================================
                   完整樣式表（見下方完整代碼）
                   ====================================================== */
        * { margin: 0; padding: 0; box-sizing: border-box; }
        html { font-size: 16px; }
        body {
            font-family: 'Segoe UI', Roboto, system-ui, -apple-system, sans-serif;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 1.25rem;
            background: #fdf6f0;
            background: linear-gradient(135deg, #fdf6f0 0%, #f0e6ff 30%, #e0f0ff 60%, #fdf6f0 100%);
            background-size: 300% 300%;
            animation: gradientFlow 16s ease-in-out infinite alternate;
            position: relative;
            overflow-x: hidden;
        }
        @keyframes gradientFlow {
            0% { background-position: 0% 50%; }
            50% { background-position: 100% 50%; }
            100% { background-position: 0% 50%; }
        }
        body::before {
            content: '';
            position: fixed;
            width: 31.25rem;
            height: 31.25rem;
            border-radius: 50%;
            background: rgba(255, 215, 215, 0.3);
            filter: blur(100px);
            top: -10%;
            left: -10%;
            z-index: 0;
            animation: floatGlow 20s ease-in-out infinite alternate;
        }
        body::after {
            content: '';
            position: fixed;
            width: 25rem;
            height: 25rem;
            border-radius: 50%;
            background: rgba(200, 230, 255, 0.3);
            filter: blur(100px);
            bottom: -10%;
            right: -10%;
            z-index: 0;
            animation: floatGlow 22s ease-in-out infinite alternate-reverse;
        }
        @keyframes floatGlow {
            0% { transform: translate(0, 0) scale(1); }
            100% { transform: translate(2.5rem, -1.875rem) scale(1.2); }
        }
        .container {
            max-width: 53.75rem;
            width: 100%;
            position: relative;
            z-index: 1;
            background: rgba(255, 255, 255, 0.6);
            backdrop-filter: blur(20px);
            -webkit-backdrop-filter: blur(20px);
            border-radius: 2.5rem;
            padding: 1.5rem 2rem 2rem;
            box-shadow: 0 20px 60px rgba(0,0,0,0.08), 0 0 0 1px rgba(255,255,255,0.8);
            border: 1px solid rgba(255,255,255,0.3);
        }
        .header {
            display: flex;
            flex-wrap: wrap;
            align-items: center;
            justify-content: space-between;
            gap: 0.5rem 1rem;
            margin-bottom: 0.25rem;
        }
        .header-title {
            font-size: 1.8rem;
            font-weight: 600;
            color: #1e293b;
            text-shadow: 0 0.125rem 0.625rem rgba(0,0,0,0.04);
            letter-spacing: -0.5px;
            line-height: 1.2;
        }
        .header-title small {
            font-size: 0.8rem;
            font-weight: 400;
            color: #64748b;
            margin-left: 0.5rem;
        }
        .header-right {
            display: flex;
            align-items: center;
            gap: 0.75rem;
            flex-wrap: wrap;
        }
        .admin-bar {
            display: flex;
            align-items: center;
            gap: 0.5rem;
            background: rgba(255,255,255,0.5);
            padding: 0.25rem 0.75rem 0.25rem 1rem;
            border-radius: 2.5rem;
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255,255,255,0.6);
            flex-wrap: wrap;
        }
        .admin-status {
            font-size: 0.75rem;
            color: #475569;
            white-space: nowrap;
        }
        .admin-status .logged-in { color: #16a34a; font-weight: 600; }
        .admin-status .logged-out { color: #dc2626; font-weight: 600; }
        .admin-bar input[type="password"] {
            background: rgba(255,255,255,0.8);
            border: 1px solid #d1d5db;
            border-radius: 1.875rem;
            padding: 0.2rem 0.75rem;
            font-size: 0.75rem;
            color: #1e293b;
            width: 6.5rem;
            transition: 0.2s;
        }
        .admin-bar input[type="password"]::placeholder { color: #94a3b8; }
        .admin-bar input[type="password"]:focus {
            outline: none;
            border-color: #818cf8;
            box-shadow: 0 0 0 3px rgba(129, 140, 248, 0.2);
        }
        .admin-bar .btn-admin {
            background: #4f46e5;
            color: #fff;
            border: none;
            padding: 0.2rem 1rem;
            border-radius: 1.875rem;
            cursor: pointer;
            font-weight: 500;
            font-size: 0.75rem;
            transition: 0.2s;
        }
        .admin-bar .btn-admin:hover { background: #4338ca; }
        .admin-bar .btn-logout {
            background: #e2e8f0;
            color: #1e293b;
            border: 1px solid #cbd5e1;
            padding: 0.2rem 1rem;
            border-radius: 1.875rem;
            cursor: pointer;
            font-weight: 500;
            font-size: 0.75rem;
            text-decoration: none;
            transition: 0.2s;
        }
        .admin-bar .btn-logout:hover { background: #cbd5e1; }
        .admin-error {
            color: #dc2626;
            font-size: 0.7rem;
        }
        .visitor-info {
            display: flex;
            align-items: center;
            gap: 0.4rem 1rem;
            flex-wrap: wrap;
            font-size: 0.8rem;
            color: #475569;
            background: rgba(129, 140, 248, 0.08);
            padding: 0.3rem 1rem;
            border-radius: 2rem;
            margin: 0.2rem 0 0.8rem 0;
            border: 1px solid rgba(129, 140, 248, 0.15);
        }
        .visitor-info .label {
            font-weight: 500;
            color: #4f46e5;
        }
        .visitor-info .ip {
            font-family: 'Courier New', monospace;
            background: rgba(0,0,0,0.04);
            padding: 0 0.4rem;
            border-radius: 0.5rem;
        }
        .visitor-info .device {
            background: rgba(0,0,0,0.04);
            padding: 0 0.4rem;
            border-radius: 0.5rem;
        }
        .visitor-info .sep {
            color: #cbd5e1;
        }
        .visitor-info .stat-item {
            opacity: 0.6;
            font-size: 0.7rem;
            display: inline-flex;
            align-items: center;
            gap: 0.2rem;
            white-space: nowrap;
        }
        .daily-quote {
            background: rgba(255, 215, 0, 0.08);
            border-left: 4px solid #f59e0b;
            padding: 0.5rem 1.2rem;
            border-radius: 1rem;
            margin: 0.2rem 0 1.2rem 0;
            font-size: 0.95rem;
            color: #1e293b;
            display: flex;
            align-items: center;
            gap: 0.5rem;
            flex-wrap: wrap;
        }
        .daily-quote .quote-icon { font-size: 1.4rem; }
        .daily-quote .quote-text { flex: 1; font-style: italic; }
        .daily-quote .quote-author {
            font-size: 0.8rem;
            color: #64748b;
            white-space: nowrap;
        }
        .sort-bar {
            display: flex;
            gap: 0.5rem;
            margin-bottom: 1rem;
            flex-wrap: wrap;
        }
        .sort-bar a {
            padding: 0.2rem 1rem;
            border-radius: 2rem;
            background: rgba(255,255,255,0.5);
            border: 1px solid rgba(0,0,0,0.05);
            text-decoration: none;
            font-size: 0.8rem;
            color: #475569;
            transition: 0.15s;
        }
        .sort-bar a:hover {
            background: rgba(79, 70, 229, 0.08);
            border-color: #4f46e5;
        }
        .sort-bar a.active {
            background: #4f46e5;
            color: #fff;
            border-color: #4f46e5;
        }
        .toast {
            padding: 0.5rem 1rem;
            border-radius: 2rem;
            margin-bottom: 1rem;
            text-align: center;
            font-weight: 500;
            backdrop-filter: blur(10px);
            font-size: 0.85rem;
        }
        .toast.success {
            background: rgba(74, 222, 128, 0.15);
            color: #166534;
            border: 1px solid rgba(74, 222, 128, 0.2);
        }
        .toast.error {
            background: rgba(248, 113, 113, 0.15);
            color: #991b1b;
            border: 1px solid rgba(248, 113, 113, 0.2);
        }
        .toast.info {
            background: rgba(129, 140, 248, 0.15);
            color: #3730a3;
            border: 1px solid rgba(129, 140, 248, 0.2);
        }
        .form-card {
            background: rgba(255,255,255,0.7);
            backdrop-filter: blur(12px);
            border-radius: 1.5rem;
            padding: 1.25rem 1.5rem 1.5rem;
            margin-bottom: 1.5rem;
            border: 1px solid rgba(255,255,255,0.8);
            box-shadow: 0 2px 12px rgba(0,0,0,0.02);
        }
        .form-group { margin-bottom: 0.75rem; }
        .form-group label {
            display: block;
            font-weight: 500;
            color: #334155;
            margin-bottom: 0.2rem;
            font-size: 0.85rem;
        }
        .form-group input[type="text"],
        .form-group textarea {
            width: 100%;
            padding: 0.6rem 0.8rem;
            border: 1px solid #e2e8f0;
            border-radius: 1rem;
            font-size: 0.95rem;
            transition: 0.2s;
            background: rgba(255,255,255,0.9);
            color: #1e293b;
        }
        .form-group input[type="text"]:focus,
        .form-group textarea:focus {
            outline: none;
            border-color: #818cf8;
            background: #ffffff;
            box-shadow: 0 0 0 3px rgba(129, 140, 248, 0.15);
        }
        .form-group textarea {
            resize: vertical;
            min-height: 4.5rem;
        }
        .form-group input[type="file"] {
            border: 1px dashed #d1d5db;
            border-radius: 1rem;
            padding: 0.6rem;
            width: 100%;
            background: rgba(255,255,255,0.6);
            color: #1e293b;
            cursor: pointer;
            transition: 0.2s;
            font-size: 0.9rem;
        }
        .form-group input[type="file"]:hover {
            border-color: #818cf8;
            background: rgba(255,255,255,0.8);
        }
        .file-hint {
            font-size: 0.7rem;
            color: #6b7280;
            margin-top: 0.2rem;
        }
        .btn-submit {
            background: #4f46e5;
            color: #fff;
            border: none;
            padding: 0.6rem 1.5rem;
            border-radius: 2.5rem;
            font-size: 0.95rem;
            font-weight: 600;
            cursor: pointer;
            transition: 0.2s;
            box-shadow: 0 4px 16px rgba(79, 70, 229, 0.25);
            width: 100%;
        }
        .btn-submit:hover {
            background: #4338ca;
            transform: translateY(-1px);
            box-shadow: 0 6px 20px rgba(79, 70, 229, 0.35);
        }
        .draft-notice {
            font-size: 0.7rem;
            color: #6b7280;
            text-align: right;
            margin-top: 0.3rem;
            opacity: 0;
            transition: opacity 0.3s;
            height: 1.2rem;
        }
        .draft-notice.visible { opacity: 0.8; }
        .msg-list {
            display: flex;
            flex-direction: column;
            gap: 0.8rem;
        }
        .msg-item {
            background: rgba(255,255,255,0.5);
            backdrop-filter: blur(8px);
            border-radius: 1.25rem;
            padding: 0.9rem 1.2rem;
            border: 1px solid rgba(255,255,255,0.8);
            transition: 0.15s;
            position: relative;
        }
        .msg-item.pinned {
            border-color: #f59e0b;
            background: rgba(255, 215, 0, 0.08);
        }
        .msg-item.pinned::before {
            content: '📌';
            position: absolute;
            top: -0.5rem;
            right: -0.3rem;
            font-size: 1.2rem;
            opacity: 0.7;
        }
        .msg-item:hover {
            background: rgba(255,255,255,0.7);
            border-color: #e2e8f0;
        }
        .msg-header {
            display: flex;
            justify-content: space-between;
            align-items: baseline;
            flex-wrap: wrap;
            gap: 0.2rem 0.5rem;
            margin-bottom: 0.2rem;
        }
        .msg-name {
            font-weight: 600;
            color: #1e293b;
            font-size: 1rem;
        }
        .msg-meta {
            font-size: 0.7rem;
            color: #64748b;
            display: flex;
            gap: 0.5rem;
            flex-wrap: wrap;
            align-items: baseline;
        }
        .msg-meta .ip {
            color: #4b5563;
            font-family: 'Courier New', monospace;
            background: rgba(0,0,0,0.04);
            padding: 0 0.3rem;
            border-radius: 0.25rem;
        }
        .msg-meta .device {
            color: #4b5563;
            background: rgba(0,0,0,0.04);
            padding: 0 0.3rem;
            border-radius: 0.25rem;
        }
        .msg-meta .time { color: #94a3b8; }
        .msg-content {
            color: #334155;
            line-height: 1.5;
            word-break: break-word;
            margin: 0.2rem 0 0.5rem;
            font-size: 0.9rem;
        }
        .msg-media {
            margin-top: 0.4rem;
            border-radius: 0.75rem;
            overflow: hidden;
            max-width: 100%;
            text-align: center;
        }
        .msg-media img {
            max-width: 100%;
            max-height: 55vh;
            border-radius: 0.75rem;
            display: block;
            margin: 0 auto;
            object-fit: contain;
        }
        .msg-media video {
            max-width: 100%;
            max-height: 55vh;
            border-radius: 0.75rem;
            background: #000;
        }
        .attachment-download {
            display: inline-flex;
            align-items: center;
            gap: 0.4rem;
            background: rgba(79, 70, 229, 0.08);
            border: 1px solid rgba(79, 70, 229, 0.2);
            padding: 0.3rem 0.8rem;
            border-radius: 2rem;
            text-decoration: none;
            color: #4f46e5;
            font-size: 0.8rem;
            transition: 0.15s;
            margin-top: 0.4rem;
        }
        .attachment-download:hover {
            background: rgba(79, 70, 229, 0.15);
            transform: translateY(-1px);
        }
        .attachment-download .file-icon { font-size: 1rem; }
        .attachment-download .file-size {
            font-size: 0.65rem;
            color: #64748b;
        }
        .msg-footer {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-top: 0.5rem;
            border-top: 1px solid rgba(0,0,0,0.04);
            padding-top: 0.5rem;
            flex-wrap: wrap;
            gap: 0.4rem;
        }
        .msg-actions {
            display: flex;
            gap: 0.5rem;
            align-items: center;
            flex-wrap: wrap;
        }
        .btn-like {
            background: none;
            border: none;
            cursor: pointer;
            font-size: 0.95rem;
            display: inline-flex;
            align-items: center;
            gap: 0.2rem;
            color: #475569;
            transition: 0.15s;
            text-decoration: none;
            padding: 0.15rem 0.5rem;
            border-radius: 1.5rem;
            background: rgba(0,0,0,0.02);
            user-select: none;
            -webkit-touch-callout: none;
            touch-action: manipulation;
        }
        .btn-like:hover {
            background: rgba(239, 68, 68, 0.1);
            color: #dc2626;
        }
        .btn-like:active { transform: scale(0.9); }
        .btn-like .count {
            font-weight: 600;
            font-size: 0.85rem;
            min-width: 1rem;
            display: inline-block;
        }
        .btn-admin-action {
            background: rgba(79, 70, 229, 0.08);
            border: 1px solid rgba(79, 70, 229, 0.15);
            padding: 0.1rem 0.6rem;
            border-radius: 1.875rem;
            font-size: 0.65rem;
            cursor: pointer;
            transition: 0.15s;
            font-weight: 500;
            text-decoration: none;
            display: inline-block;
            color: #4f46e5;
        }
        .btn-admin-action:hover { background: rgba(79, 70, 229, 0.15); }
        .btn-admin-action.pin {
            color: #f59e0b;
            border-color: #f59e0b;
        }
        .btn-admin-action.pin:hover { background: rgba(245, 158, 11, 0.1); }
        .btn-delete {
            background: rgba(239, 68, 68, 0.08);
            color: #dc2626;
            border: 1px solid rgba(239, 68, 68, 0.15);
            padding: 0.1rem 0.8rem;
            border-radius: 1.875rem;
            font-size: 0.7rem;
            cursor: pointer;
            transition: 0.15s;
            font-weight: 500;
            text-decoration: none;
            display: inline-block;
        }
        .btn-delete:hover {
            background: rgba(239, 68, 68, 0.15);
            border-color: #dc2626;
            color: #991b1b;
        }
        .empty-msg {
            text-align: center;
            color: #94a3b8;
            padding: 1.5rem 0;
            font-size: 1rem;
        }
        .stats-section {
            margin-top: 1.5rem;
            border-top: 1px solid rgba(0,0,0,0.06);
            padding-top: 1rem;
        }
        .stats-section h3 {
            font-size: 1rem;
            font-weight: 600;
            color: #1e293b;
            margin-bottom: 0.5rem;
        }
        .stats-table {
            width: 100%;
            border-collapse: collapse;
            font-size: 0.75rem;
        }
        .stats-table th {
            text-align: left;
            padding: 0.25rem 0.4rem;
            background: rgba(0,0,0,0.02);
            font-weight: 600;
            color: #475569;
        }
        .stats-table td {
            padding: 0.25rem 0.4rem;
            border-bottom: 1px solid rgba(0,0,0,0.04);
            color: #334155;
        }
        .stats-table tr:last-child td { border-bottom: none; }
        .stats-table .ip-cell {
            font-family: 'Courier New', monospace;
            color: #4b5563;
        }
        .stats-table .device-cell { color: #4b5563; }
        .stats-table .action-badge {
            display: inline-block;
            padding: 0.05rem 0.5rem;
            border-radius: 1rem;
            font-size: 0.65rem;
            font-weight: 500;
        }
        .action-view {
            background: rgba(59, 130, 246, 0.1);
            color: #2563eb;
        }
        .action-like {
            background: rgba(239, 68, 68, 0.1);
            color: #dc2626;
        }
        .stats-note {
            font-size: 0.7rem;
            color: #94a3b8;
            margin-top: 0.4rem;
            text-align: center;
        }
        .stats-note a {
            color: #4f46e5;
            text-decoration: none;
            cursor: pointer;
        }
        .stats-note a:hover { text-decoration: underline; }
        .pagination {
            display: flex;
            justify-content: center;
            gap: 0.3rem;
            margin-top: 0.8rem;
            flex-wrap: wrap;
        }
        .pagination a, .pagination span {
            display: inline-block;
            padding: 0.2rem 0.6rem;
            border-radius: 0.3rem;
            font-size: 0.75rem;
            text-decoration: none;
            color: #4f46e5;
            background: rgba(255,255,255,0.5);
            border: 1px solid rgba(0,0,0,0.05);
            transition: 0.15s;
        }
        .pagination a:hover {
            background: rgba(79, 70, 229, 0.1);
            border-color: #4f46e5;
        }
        .pagination .current {
            background: #4f46e5;
            color: #fff;
            border-color: #4f46e5;
            font-weight: 600;
        }
        .pagination .disabled {
            color: #94a3b8;
            cursor: not-allowed;
            background: rgba(0,0,0,0.02);
        }
        footer {
            margin-top: 1.5rem;
            text-align: center;
            font-size: 0.7rem;
            color: #94a3b8;
            border-top: 1px solid rgba(0,0,0,0.04);
            padding-top: 1rem;
        }
        footer a {
            color: #4f46e5;
            text-decoration: none;
        }
        footer a:hover { color: #4338ca; }

        #music-btn {
            position: fixed;
            bottom: 1.2rem;
            right: 1.2rem;
            z-index: 999;
            width: 3rem;
            height: 3rem;
            border-radius: 50%;
            background: rgba(255,255,255,0.7);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255,255,255,0.5);
            box-shadow: 0 4px 20px rgba(0,0,0,0.1);
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 1.5rem;
            color: #4f46e5;
            transition: all 0.3s ease;
            user-select: none;
        }
        #music-btn:hover {
            background: rgba(255,255,255,0.9);
            transform: scale(1.05);
            box-shadow: 0 6px 28px rgba(0,0,0,0.15);
        }
        #music-btn:active { transform: scale(0.95); }
        @media (max-width: 480px) {
            #music-btn {
                width: 2.6rem;
                height: 2.6rem;
                font-size: 1.3rem;
                bottom: 0.8rem;
                right: 0.8rem;
            }
        }

        .heart-float {
            position: fixed;
            pointer-events: none;
            font-size: 1.8rem;
            z-index: 9999;
            animation: floatUp 1.2s ease-out forwards;
            opacity: 1;
        }
        @keyframes floatUp {
            0% { opacity: 1; transform: translateY(0) scale(1) rotate(0deg); }
            100% { opacity: 0; transform: translateY(-200px) scale(0.5) rotate(40deg); }
        }
        .heart-float.small { font-size: 1.2rem; }
        .heart-float.medium { font-size: 1.8rem; }
        .heart-float.large { font-size: 2.5rem; }

        @media (max-width: 640px) {
            .container { padding: 1rem 1.2rem; border-radius: 2rem; }
            .header-title { font-size: 1.5rem; }
            .header-title small { font-size: 0.7rem; }
            .header-right { width: 100%; justify-content: space-between; }
            .admin-bar { width: 100%; justify-content: center; padding: 0.3rem 0.5rem; }
            .admin-bar input[type="password"] { width: 100%; }
            .visitor-info { font-size: 0.7rem; padding: 0.2rem 0.8rem; gap: 0.2rem; }
            .daily-quote { font-size: 0.85rem; padding: 0.4rem 0.8rem; }
            .form-card { padding: 1rem; }
            .msg-item { padding: 0.7rem 0.9rem; }
            .msg-meta { font-size: 0.6rem; gap: 0.3rem; flex-direction: column; align-items: flex-start; }
            .stats-table { font-size: 0.65rem; }
            .stats-table th, .stats-table td { padding: 0.2rem 0.3rem; }
            .btn-submit { font-size: 0.9rem; }
            .stats-table .action-badge { font-size: 0.55rem; padding: 0.05rem 0.4rem; }
            .attachment-download { font-size: 0.7rem; }
            .pagination a, .pagination span { font-size: 0.7rem; padding: 0.15rem 0.5rem; }
            .sort-bar a { font-size: 0.7rem; padding: 0.15rem 0.6rem; }
            .msg-item.pinned::before { font-size: 1rem; top: -0.3rem; right: 0rem; }
        }
        @media (max-width: 380px) {
            html { font-size: 14px; }
            .container { padding: 0.8rem; }
            .header-title { font-size: 1.3rem; }
        }
        @media (min-width: 1400px) {
            .container { max-width: 64rem; }
            .header-title { font-size: 2.4rem; }
        }
    </style>
</head>
<body>
<div class="container">
    <div class="header">
        <div class="header-title">
            📬 安然小棧
            <small>圖文·視頻·附件</small>
        </div>
        <div class="header-right">
            <div class="admin-bar">
                <?php if ($is_admin): ?>
                    <span class="admin-status">✅ 管理員 <span class="logged-in">已登錄</span></span>
                    <a href="?logout=1" class="btn-logout">🚪 退出</a>
                <?php else: ?>
                    <span class="admin-status">🔐 管理員 <span class="logged-out">未登錄</span></span>
                    <form method="POST" action="" style="display: flex; gap: 0.3rem; flex-wrap: wrap; align-items: center;">
                        <input type="password" name="admin_password" placeholder="密碼" required>
                        <button type="submit" class="btn-admin">登錄</button>
                    </form>
                    <?php if (isset($login_error)): ?>
                        <span class="admin-error"><?= htmlspecialchars($login_error) ?></span>
                    <?php endif; ?>
                <?php endif; ?>
            </div>
        </div>
    </div>

    <div class="visitor-info">
        <span class="label">📡 您的信息</span>
        <span class="ip">IP: <?= htmlspecialchars($visitor_ip) ?></span>
        <span class="sep">|</span>
        <span class="device">設備: <?= htmlspecialchars($visitor_device) ?></span>
        <span class="sep">|</span>
        <span class="stat-item">📊 總訪問量: <?= number_format($totalViews) ?></span>
        <span class="stat-item">📎 文件上限 <?= MAX_FILE_SIZE / 1024 / 1024 / 1024 ?>GB</span>
    </div>

    <div class="daily-quote">
        <span class="quote-icon">💬</span>
        <span class="quote-text">“<?= htmlspecialchars($dailyQuote[0]) ?>”</span>
        <span class="quote-author">—— <?= htmlspecialchars($dailyQuote[1]) ?></span>
    </div>

    <?php if (isset($_GET['success'])): ?>
        <div class="toast success">✅ 留言發送成功！感謝你的分享 💬</div>
    <?php endif; ?>
    <?php if (isset($_GET['delete_success'])): ?>
        <div class="toast success">🗑️ 留言已成功刪除。</div>
    <?php endif; ?>
    <?php if (isset($_GET['login_success'])): ?>
        <div class="toast success">✅ 管理員登錄成功！</div>
    <?php endif; ?>
    <?php if (isset($_GET['logout_success'])): ?>
        <div class="toast info">👋 已退出管理。</div>
    <?php endif; ?>
    <?php if (isset($_GET['pin_success'])): ?>
        <div class="toast success">📌 已置頂留言。</div>
    <?php endif; ?>
    <?php if (isset($_GET['unpin_success'])): ?>
        <div class="toast info">📌 已取消置頂留言。</div>
    <?php endif; ?>
    <?php if ($upload_error !== null && $_SERVER['REQUEST_METHOD'] === 'POST' && !isset($_POST['admin_password'])): ?>
        <div class="toast error">⚠️ <?= htmlspecialchars($upload_error) ?></div>
    <?php endif; ?>

    <div class="form-card">
        <form method="POST" action="" enctype="multipart/form-data" id="messageForm">
            <div class="form-group">
                <label for="name">👤 暱稱</label>
                <input type="text" id="name" name="name" placeholder="你的名字（可選）" maxlength="30">
            </div>
            <div class="form-group">
                <label for="content">✏️ 留言內容</label>
                <textarea id="content" name="content" placeholder="說點什麼吧…" required></textarea>
            </div>
            <div class="form-group">
                <label for="media">📎 上傳文件（圖片/視頻/文檔等）</label>
                <input type="file" id="media" name="media">
                <div class="file-hint">支持圖片、視頻、PDF、ZIP、Word、Excel、PPT、TXT 等常見格式（最大 <?= MAX_FILE_SIZE / 1024 / 1024 / 1024 ?>GB）</div>
            </div>
            <button type="submit" class="btn-submit">📨 發送留言</button>
        </form>
        <div class="draft-notice" id="draftNotice">💾 草稿已保存</div>
    </div>

    <div class="sort-bar">
        <a href="?sort=latest" class="<?= $sort === 'latest' ? 'active' : '' ?>">🕒 最新</a>
        <a href="?sort=hot" class="<?= $sort === 'hot' ? 'active' : '' ?>">🔥 熱門</a>
    </div>

    <div class="msg-list">
        <?php if (count($messages) === 0): ?>
            <div class="empty-msg">🌱 還沒有留言，來做第一個吧！</div>
        <?php else: ?>
            <?php foreach ($messages as $msg): ?>
                <div class="msg-item <?= $msg['is_pinned'] ? 'pinned' : '' ?>" data-id="<?= $msg['id'] ?>">
                    <div class="msg-header">
                        <span class="msg-name"><?= htmlspecialchars($msg['name'] ?: '匿名') ?></span>
                        <div class="msg-meta">
                            <span class="ip">IP: <?= htmlspecialchars($msg['ip_address'] ?? '未知') ?></span>
                            <span class="device">設備: <?= htmlspecialchars(parse_user_agent($msg['user_agent'] ?? '')) ?></span>
                            <span class="time"><?= formatLocalTime($msg['created_at']) ?></span>
                        </div>
                    </div>
                    <div class="msg-content"><?= nl2br(htmlspecialchars($msg['content'])) ?></div>
                    <?php if (!empty($msg['media_path'])): 
                        $file_path = htmlspecialchars($msg['media_path']);
                        $ext = strtolower(pathinfo($file_path, PATHINFO_EXTENSION));
                        $full_path = __DIR__ . '/' . $msg['media_path'];
                        $file_size = file_exists($full_path) ? filesize($full_path) : 0;
                    ?>
                        <?php if (in_array($ext, ['jpg', 'jpeg', 'png', 'gif', 'webp'])): ?>
                            <div class="msg-media">
                                <img src="<?= $file_path ?>" alt="留言圖片" loading="lazy">
                            </div>
                        <?php elseif (in_array($ext, ['mp4', 'webm', 'ogg'])): ?>
                            <div class="msg-media">
                                <video controls preload="none" width="100%">
                                    <source src="<?= $file_path ?>" type="video/<?= $ext === 'mp4' ? 'mp4' : ($ext === 'webm' ? 'webm' : 'ogg') ?>">
                                    您的瀏覽器不支持視頻播放。
                                </video>
                            </div>
                        <?php else: ?>
                            <div>
                                <a href="<?= $file_path ?>" class="attachment-download" download>
                                    <span class="file-icon">📄</span>
                                    下載附件 (<?= htmlspecialchars($ext) ?>)
                                    <?php if ($file_size > 0): ?>
                                        <span class="file-size"><?= formatFileSize($file_size) ?></span>
                                    <?php endif; ?>
                                </a>
                            </div>
                        <?php endif; ?>
                    <?php endif; ?>
                    <div class="msg-footer">
                        <div class="msg-actions">
                            <button class="btn-like" data-id="<?= $msg['id'] ?>">
                                ❤️ <span class="count"><?= $msg['likes'] ?></span>
                            </button>
                            <?php if ($is_admin): ?>
                                <?php if ($msg['is_pinned']): ?>
                                    <a href="?unpin=<?= $msg['id'] ?>" class="btn-admin-action pin">取消置頂</a>
                                <?php else: ?>
                                    <a href="?pin=<?= $msg['id'] ?>" class="btn-admin-action">📌 置頂</a>
                                <?php endif; ?>
                                <a href="?delete=<?= $msg['id'] ?>" class="btn-delete" onclick="return confirm('確定要刪除這條留言嗎？')">🗑️ 刪除</a>
                            <?php endif; ?>
                        </div>
                    </div>
                </div>
            <?php endforeach; ?>
        <?php endif; ?>
    </div>

    <?php if (!empty($visitorStats)): ?>
    <div class="stats-section">
        <h3>📊 訪客統計</h3>
        <table class="stats-table">
            <thead>
                <tr>
                    <th>IP 地址</th>
                    <th>設備</th>
                    <th>瀏覽次數</th>
                    <th>點贊總數</th>
                    <th>最後活躍時間</th>
                    <th>最近操作</th>
                </tr>
            </thead>
            <tbody>
            <?php foreach ($displayStats as $stat): 
                $ip = $stat['ip'];
                if (!$is_admin) {
                    $parts = explode('.', $ip);
                    if (count($parts) == 4) {
                        $ip = $parts[0] . '.' . $parts[1] . '.*.*';
                    } else {
                        $ip = substr($ip, 0, 8) . '...';
                    }
                }
                $device = parse_user_agent($stat['user_agent']);
                $lastTime = formatLocalTime($stat['last_time']);
                $action = $stat['last_action'];
                if ($action === 'view') {
                    $actionLabel = '瀏覽';
                    $actionClass = 'action-view';
                } else {
                    $actionLabel = '點贊';
                    if ($is_admin && $stat['last_message_id']) {
                        $actionLabel .= ' (留言#' . $stat['last_message_id'] . ')';
                    }
                    $actionClass = 'action-like';
                }
            ?>
                <tr>
                    <td class="ip-cell"><?= htmlspecialchars($ip) ?></td>
                    <td class="device-cell"><?= htmlspecialchars($device) ?></td>
                    <td><?= $stat['view_count'] ?></td>
                    <td><?= $stat['like_count'] ?></td>
                    <td><?= $lastTime ?></td>
                    <td><span class="action-badge <?= $actionClass ?>"><?= $actionLabel ?></span></td>
                </tr>
            <?php endforeach; ?>
            </tbody>
        </table>

        <?php if ($is_admin && $totalStats > $perPage): ?>
        <div class="pagination">
            <?php if ($currentPage > 1): ?>
                <a href="?page=<?= $currentPage - 1 ?>&sort=<?= $sort ?>">‹ 上一頁</a>
            <?php else: ?>
                <span class="disabled">‹ 上一頁</span>
            <?php endif; ?>
            <?php
            $startPage = max(1, $currentPage - 4);
            $endPage = min($totalPages, $currentPage + 5);
            if ($startPage > 1) echo '<span>…</span>';
            for ($i = $startPage; $i <= $endPage; $i++):
            ?>
                <?php if ($i == $currentPage): ?>
                    <span class="current"><?= $i ?></span>
                <?php else: ?>
                    <a href="?page=<?= $i ?>&sort=<?= $sort ?>"><?= $i ?></a>
                <?php endif; ?>
            <?php endfor; ?>
            <?php if ($endPage < $totalPages): ?>
                <span>…</span>
            <?php endif; ?>
            <?php if ($currentPage < $totalPages): ?>
                <a href="?page=<?= $currentPage + 1 ?>&sort=<?= $sort ?>">下一頁 ›</a>
            <?php else: ?>
                <span class="disabled">下一頁 ›</span>
            <?php endif; ?>
        </div>
        <div class="stats-note">共 <?= $totalStats ?> 條記錄，當前第 <?= $currentPage ?> / <?= $totalPages ?> 頁</div>
        <?php endif; ?>

        <?php if (!$is_admin && count($visitorStats) > 5): ?>
            <div class="stats-note">🔒 僅顯示前5條，<a href="#" onclick="document.querySelector('.admin-bar input[type=password]').focus(); return false;">登錄管理員</a>可查看全部。</div>
        <?php endif; ?>
        <?php if ($is_admin && $totalStats <= $perPage): ?>
            <div class="stats-note">👑 管理員已登錄，共 <?= $totalStats ?> 條記錄。</div>
        <?php endif; ?>
    </div>
    <?php endif; ?>

    <footer>
        ❤️ Powered by <a href="https://debian.org" target="_blank">Debian 13</a> &amp; <a href="https://nginx.org" target="_blank">Nginx</a> · 安然如故
    </footer>
</div>

<audio id="bgMusic" src="/music/background.mp3" preload="metadata" loop></audio>
<button id="music-btn" aria-label="切換背景音樂">🎵</button>

<script>
    (function() {
        // ------ 音樂播放器 ------
        const audio = document.getElementById('bgMusic');
        const btn = document.getElementById('music-btn');
        btn.textContent = '🎵';
        btn.addEventListener('click', function() {
            if (audio.paused) {
                audio.play().catch(function(e) { console.log('播放失敗:', e); });
                btn.textContent = '🔊';
            } else {
                audio.pause();
                btn.textContent = '🎵';
            }
        });
        audio.addEventListener('ended', function() { btn.textContent = '🎵'; });
        audio.addEventListener('error', function() {
            console.warn('音樂加載失敗，請檢查 /music/background.mp3');
            btn.textContent = '⚠️';
            btn.style.cursor = 'default';
            btn.disabled = true;
        });

        // ------ 點贊功能 ------
        document.querySelectorAll('.btn-like').forEach(function(btn) {
            let timer = null;

            function doLike() {
                const id = btn.dataset.id;
                const countSpan = btn.querySelector('.count');
                let current = parseInt(countSpan.textContent) || 0;
                countSpan.textContent = current + 1;

                const rect = btn.getBoundingClientRect();
                const x = rect.left + rect.width / 2;
                const y = rect.top + rect.height / 2;
                const heart = document.createElement('div');
                heart.className = 'heart-float';
                const sizes = ['small', 'medium', 'large'];
                heart.classList.add(sizes[Math.floor(Math.random() * sizes.length)]);
                const colors = ['#e74c3c', '#f06292', '#ff6b6b', '#ee5a24', '#fd79a8'];
                heart.style.color = colors[Math.floor(Math.random() * colors.length)];
                heart.style.left = (x + (Math.random() - 0.5) * 60) + 'px';
                heart.style.top = y + 'px';
                heart.textContent = '❤️';
                document.body.appendChild(heart);
                setTimeout(() => heart.remove(), 1200);

                fetch('?ajax=like&id=' + id)
                    .then(response => response.json())
                    .then(data => {
                        if (data.status === 'success') {
                            countSpan.textContent = data.likes;
                        }
                    })
                    .catch(error => console.error('AJAX錯誤:', error));
            }

            function startHold(e) {
                e.preventDefault();
                if (timer) return;
                doLike();
                timer = setInterval(doLike, 150);
            }
            function endHold(e) {
                e.preventDefault();
                if (timer) { clearInterval(timer); timer = null; }
            }
            btn.addEventListener('mousedown', startHold);
            btn.addEventListener('mouseup', endHold);
            btn.addEventListener('mouseleave', endHold);
            btn.addEventListener('touchstart', startHold, { passive: false });
            btn.addEventListener('touchend', endHold, { passive: false });
            btn.addEventListener('touchcancel', endHold, { passive: false });
            btn.addEventListener('contextmenu', e => e.preventDefault());
        });

        // ------ 自動保存草稿 ------
        const nameInput = document.getElementById('name');
        const contentInput = document.getElementById('content');
        const draftNotice = document.getElementById('draftNotice');
        const form = document.getElementById('messageForm');

        function saveDraft() {
            const draft = { name: nameInput.value, content: contentInput.value };
            localStorage.setItem('message_draft', JSON.stringify(draft));
            draftNotice.classList.add('visible');
            clearTimeout(window.draftTimeout);
            window.draftTimeout = setTimeout(() => draftNotice.classList.remove('visible'), 1500);
        }

        function loadDraft() {
            const stored = localStorage.getItem('message_draft');
            if (stored) {
                try {
                    const draft = JSON.parse(stored);
                    if (draft.name) nameInput.value = draft.name;
                    if (draft.content) contentInput.value = draft.content;
                    if (draft.name || draft.content) {
                        draftNotice.textContent = '📂 草稿已恢復';
                        draftNotice.classList.add('visible');
                        setTimeout(() => {
                            draftNotice.classList.remove('visible');
                            draftNotice.textContent = '💾 草稿已保存';
                        }, 2500);
                    }
                } catch (e) {}
            }
        }

        nameInput.addEventListener('input', saveDraft);
        contentInput.addEventListener('input', saveDraft);

        if (new URLSearchParams(window.location.search).has('success')) {
            localStorage.removeItem('message_draft');
        }

        loadDraft();

        window.addEventListener('beforeunload', function() {
            if (nameInput.value || contentInput.value) saveDraft();
        });

        function checkEmptyDraft() {
            if (!nameInput.value && !contentInput.value) {
                localStorage.removeItem('message_draft');
                draftNotice.classList.remove('visible');
            }
        }
        nameInput.addEventListener('blur', checkEmptyDraft);
        contentInput.addEventListener('blur', checkEmptyDraft);
    })();
</script>
</body>
</html>
```

> **注意**：務必改 `define('ADMIN_PASSWORD', 'your_password_here');` 中之密碼爲強密碼。

---

## 🎵 第七步：背景音樂（可選）

欲加背景音樂，依下法行之：

```bash
# 創建音樂目錄
mkdir -p /var/www/your_domain/music

# 上傳你的 MP3 文件（使用 scp 或 sftp）
# 示例：scp /本地路徑/background.mp3 root@服務器IP:/var/www/your_domain/music/
```

確保文件名爲 `background.mp3`，代碼中之播放器自會加載。

---

## 🧪 第八步：測試與驗證

既畢，訪汝之域名：

```
https://your_domain.com
```

當見完整之留言板界面。可試：

1. 發佈一條留言（帶或不帶附件）
2. 點按點贊之鈕（長按可連贊，兼有愛心之效）
3. 切換「最新」「熱門」之排序
4. 管理員登錄（點右上角「管理員」，輸入密碼）
5. 置頂或刪除留言
6. 刷新頁面，觀「每日一言」之變化
7. 觀訪客統計

---

## 🔧 常見問題排查

### Q1：上傳文件失敗（413 Request Entity Too Large）

**原因**：Nginx 或 PHP 上傳大小之限不足。

**解決**：查 Nginx 配置中之 `client_max_body_size` 與 PHP 配置中之 `upload_max_filesize`、`post_max_size`，確保足夠大，然後重啓服務。

### Q2：留言中之圖片/視頻不顯示

**原因**：上傳目錄權限不當，或 Nginx 未正確訪問。

**解決**：

```bash
sudo chown -R www-data:www-data /var/www/your_domain/uploads
sudo chmod 755 /var/www/your_domain/uploads
```

### Q3：管理員密碼遺忘

**解決**：於 `index.php` 中改 `define('ADMIN_PASSWORD', 'new_password');`，然後刷新頁面。

### Q4：SQLite 數據庫寫入失敗

**原因**：目錄權限不足。

**解決**：

```bash
sudo chown www-data:www-data /var/www/your_domain
sudo chmod 755 /var/www/your_domain
```

---

## 📊 數據備份建議

數據庫文件爲 `/var/www/your_domain/messages.db`，建議定期備份：

```bash
# 手動備份
sudo cp /var/www/your_domain/messages.db /backup/messages_$(date +%Y%m%d).db

# 或設置定時任務（crontab）
# 每天凌晨2點備份
0 2 * * * cp /var/www/your_domain/messages.db /backup/messages_$(date +\%Y\%m\%d).db
```

---

## 🚀 性能優化建議

1. **開啓 Nginx 緩存**：爲靜態資源（圖片、CSS、JS）設緩存頭。
2. **用 Redis 緩存**：若有高併發之需，可引入 Redis 緩存熱門留言列表。
3. **定期清理上傳目錄**：過期或無關聯之附件，可寫腳本定期清理。
4. **啓用 Gzip 壓縮**：減傳輸之體積，增加載之速。

---

## 🎯 總結

至此，君已成功部署一功能完備之留言板網站。此項目具以下特點：

- ✅ **輕量級**：基於 SQLite，無須額外數據庫服務
- ✅ **功能豐富**：含點贊、置頂、排序、附件上傳、訪客統計等
- ✅ **安全可靠**：支持 HTTPS，上傳文件類型受限，防注入之處理
- ✅ **響應式設計**：適配桌面與移動設備
- ✅ **易於維護**：單文件架構，便於修改與擴展

君可依實際之需繼續定製，例如：

- 增驗證碼之功能
- 加郵件通知新留言
- 接入 CDN 以加速靜態資源
- 增用戶註冊登錄之系統
