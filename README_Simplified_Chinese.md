[English](https://github.com/Anan-up/message-board/blob/main/README.md) | [简体中文](https://github.com/Anan-up/message-board/blob/main/README_Simplified_Chinese.md) | [繁体中文](https://github.com/Anan-up/message-board/blob/main/README_Classical_Chinese.md)

# 从零搭建一个功能完备的留言板网站

> 一份完整的部署指南，包含源码、配置说明和常见问题排查

## 📌 引言

在互联网时代，留言板依然是一个极具价值的交互工具。无论是个人博客、小型社区还是企业内部沟通，一个简洁、功能完备的留言板都能发挥重要作用。

本文将带你从零开始，在 Linux 服务器上部署一个功能丰富的留言板网站。项目采用 **Nginx + PHP + SQLite** 的轻量级架构，无需额外安装 MySQL，对资源消耗极低，适合部署在云服务器甚至树莓派上。

**最终效果预览：**

- 支持发布留言（含昵称、内容、附件上传）
- 图片/视频自动识别展示，其他文件提供下载
- 无限点赞（长按连续点赞 + 爱心飘落特效）
- 留言排序（最新 / 热门）
- 管理员置顶和删除
- 访客统计（IP、设备、浏览次数、点赞总数）
- 每日一言（每次刷新随机展示）
- 总访问量统计
- 草稿自动保存
- 背景音乐播放器
- 全站 HTTPS 支持

---

## 🛠 技术栈

| 组件 | 版本 | 说明 |
|------|------|------|
| 操作系统 | Debian 13.2 | 也可兼容 Ubuntu 22.04+ |
| Web 服务器 | Nginx 1.24+ | 高性能 HTTP 服务 |
| PHP | PHP 8.3 / 8.4 | 后端逻辑处理 |
| 数据库 | SQLite 3 | 轻量级文件数据库 |
| 前端 | 原生 HTML + CSS + JavaScript | 无需任何框架 |

---

## 📦 环境要求

- 一台 Linux 服务器（建议 Debian 12+ 或 Ubuntu 22.04+）
- 一个已解析到服务器 IP 的域名（用于 HTTPS）
- SSL 证书（本教程使用已有证书，也可使用 Let's Encrypt 免费签发）

---

## 🚀 第一步：环境准备

登录服务器后，首先更新系统并安装必要的软件包：

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装 Nginx、PHP 及相关扩展
sudo apt install -y nginx php-fpm php-sqlite3 sqlite3 curl wget unzip

# 查看 PHP 版本（后续配置需要）
php -v
```

---

## 📁 第二步：创建网站目录

```bash
# 创建网站根目录
sudo mkdir -p /var/www/your_domain

# 创建上传目录（用于存放用户上传的附件）
sudo mkdir -p /var/www/your_domain/uploads

# 设置权限
sudo chown -R www-data:www-data /var/www/your_domain
sudo chmod 755 /var/www/your_domain/uploads
```

> **说明**：将 `your_domain` 替换为你的实际域名。

---

## 🔐 第三步：准备 SSL 证书（可选但推荐）

如果你已有证书文件（`.crt` 和 `.key`），将它们放在服务器上（例如 `/path/to/ssl/`）。如果没有，可以使用 Let's Encrypt 免费签发：

```bash
# 安装 Certbot
sudo apt install -y certbot python3-certbot-nginx

# 自动签发证书（域名需要已解析）
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
```

---

## 📄 第四步：Nginx 站点配置

创建 Nginx 虚拟主机配置文件：

```bash
sudo nano /etc/nginx/sites-available/your_domain
```

将以下内容粘贴进去（根据实际情况修改域名和证书路径）：

```nginx
# HTTP -> HTTPS 重定向
server {
    listen 80;
    listen [::]:80;
    server_name your_domain.com www.your_domain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS 主站点
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name your_domain.com www.your_domain.com;

    root /var/www/your_domain;
    index index.php index.html;

    # SSL 证书配置
    ssl_certificate     /path/to/ssl/your_domain_bundle.crt;
    ssl_certificate_key /path/to/ssl/your_domain.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # 日志（可选）
    access_log /var/log/nginx/your_domain.access.log;
    error_log  /var/log/nginx/your_domain.error.log;

    # 上传目录：禁止执行 PHP
    location /uploads/ {
        location ~ \.php$ {
            deny all;
        }
    }

    # 放开上传大小限制（建议与 PHP 配置一致）
    client_max_body_size 10G;

    # 处理 PHP 请求
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 600s;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
    }
}
```

启用站点并重载 Nginx：

```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🐘 第五步：调整 PHP 配置

编辑 PHP-FPM 配置文件（路径可能因版本而异）：

```bash
sudo nano /etc/php/8.4/fpm/php.ini
```

修改以下参数（适应大文件上传）：

```ini
upload_max_filesize = 10G
post_max_size = 10G
max_execution_time = 600
max_input_time = 600
memory_limit = 512M
```

重启 PHP-FPM：

```bash
sudo systemctl restart php8.4-fpm
```

---

## 📄 第六步：完整源码 `index.php`

这是项目的核心文件，包含了留言板的所有功能。将其保存为 `/var/www/your_domain/index.php`：

```php
<?php
date_default_timezone_set('Asia/Shanghai');
session_start();

// ========== 数据库初始化 ==========
$db_file = 'messages.db';
$db = new SQLite3($db_file);

// 创建留言表（如果不存在）
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

// 检查并添加缺失的列（兼容旧库）
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

// ========== 创建访客日志表 ==========
$db->exec("CREATE TABLE IF NOT EXISTS visitor_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ip_address TEXT NOT NULL,
    user_agent TEXT,
    action TEXT NOT NULL,
    message_id INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
)");

// ========== 管理员密码 ==========
define('ADMIN_PASSWORD', 'your_password_here'); // 请修改为强密码

// ========== 处理管理员登录 ==========
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['admin_password'])) {
    if ($_POST['admin_password'] === ADMIN_PASSWORD) {
        $_SESSION['admin'] = true;
        header('Location: ?login_success=1');
        exit;
    } else {
        $login_error = '密码错误，请重试。';
    }
}

// ========== 处理管理员退出 ==========
if (isset($_GET['logout'])) {
    unset($_SESSION['admin']);
    session_destroy();
    header('Location: ?logout_success=1');
    exit;
}

// ========== 处理删除留言 ==========
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

// ========== 处理置顶/取消置顶 ==========
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

// ========== 处理点赞（AJAX） ==========
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
        echo json_encode(['status' => 'error', 'message' => '无效ID']);
    }
    exit;
}

// ========== 记录页面浏览 ==========
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

// ========== 辅助函数 ==========
function parse_user_agent($ua) {
    if (empty($ua)) return '未知设备';
    $os = '未知OS'; $browser = '未知浏览器';
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

// ========== 每日一言（每次刷新随机） ==========
function getDailyQuote() {
    $quotes = [
        ['生活不止眼前的苟且，还有诗和远方。', '高晓松'],
        ['愿你出走半生，归来仍是少年。', '网络'],
        ['世界以痛吻我，要我报之以歌。', '泰戈尔'],
        ['生如夏花之绚烂，死如秋叶之静美。', '泰戈尔'],
        ['黑夜给了我黑色的眼睛，我却用它寻找光明。', '顾城'],
        ['面朝大海，春暖花开。', '海子'],
        ['每一个不曾起舞的日子，都是对生命的辜负。', '尼采'],
        ['生活不是等待风暴过去，而是学会在雨中跳舞。', '网络'],
        ['我们曾如此渴望命运的波澜，到最后才发现，人生最曼妙的风景，竟是内心的淡定与从容。', '杨绛'],
        ['一个人可以被毁灭，但不能被打败。', '海明威'],
        ['世上只有一种英雄主义，就是在认清生活真相之后依然热爱生活。', '罗曼·罗兰'],
        ['路漫漫其修远兮，吾将上下而求索。', '屈原'],
        ['长风破浪会有时，直挂云帆济沧海。', '李白'],
        ['天生我材必有用，千金散尽还复来。', '李白'],
        ['安得广厦千万间，大庇天下寒士俱欢颜。', '杜甫'],
        ['采菊东篱下，悠然见南山。', '陶渊明'],
        ['沉舟侧畔千帆过，病树前头万木春。', '刘禹锡'],
        ['山重水复疑无路，柳暗花明又一村。', '陆游'],
        ['横看成岭侧成峰，远近高低各不同。', '苏轼'],
        ['但愿人长久，千里共婵娟。', '苏轼'],
        ['人有悲欢离合，月有阴晴圆缺。', '苏轼'],
        ['不畏浮云遮望眼，自缘身在最高层。', '王安石'],
        ['问渠那得清如许？为有源头活水来。', '朱熹'],
        ['纸上得来终觉浅，绝知此事要躬行。', '陆游'],
        ['宝剑锋从磨砺出，梅花香自苦寒来。', '网络'],
        ['业精于勤，荒于嬉；行成于思，毁于随。', '韩愈'],
        ['书山有路勤为径，学海无涯苦作舟。', '韩愈'],
        ['三人行，必有我师焉。', '孔子'],
        ['学而不思则罔，思而不学则殆。', '孔子'],
        ['温故而知新，可以为师矣。', '孔子'],
        ['君子和而不同，小人同而不和。', '孔子'],
        ['君子坦荡荡，小人长戚戚。', '孔子'],
        ['穷则独善其身，达则兼济天下。', '孟子'],
        ['富贵不能淫，贫贱不能移，威武不能屈。', '孟子'],
        ['天行健，君子以自强不息。', '周易'],
        ['地势坤，君子以厚德载物。', '周易'],
        ['疾风知劲草，板荡识诚臣。', '李世民'],
        ['海阔凭鱼跃，天高任鸟飞。', '网络'],
        ['海内存知己，天涯若比邻。', '王勃'],
        ['落霞与孤鹜齐飞，秋水共长天一色。', '王勃'],
        ['先天下之忧而忧，后天下之乐而乐。', '范仲淹'],
        ['不以物喜，不以己悲。', '范仲淹'],
        ['醉卧沙场君莫笑，古来征战几人回。', '王翰'],
    ];
    $index = mt_rand(0, count($quotes) - 1);
    return $quotes[$index];
}

// ========== 获取当前访客信息 ==========
$visitor_ip = $_SERVER['REMOTE_ADDR'];
$visitor_ua = $_SERVER['HTTP_USER_AGENT'] ?? '';
$visitor_device = parse_user_agent($visitor_ua);

// ========== 获取总访问量 ==========
$totalViews = $db->querySingle("SELECT COUNT(*) FROM visitor_logs WHERE action='view'");
if ($totalViews === null || $totalViews === false) $totalViews = 0;

// ========== 获取每日一言 ==========
$dailyQuote = getDailyQuote();

// ========== 文件上传处理 ==========
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
                $upload_error = '文件上传失败，错误码：' . $file['error'];
            } else {
                if ($file['size'] > MAX_FILE_SIZE) {
                    $upload_error = '文件大小超过 ' . (MAX_FILE_SIZE / 1024 / 1024 / 1024) . 'GB 限制。';
                } else {
                    $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
                    if (in_array($ext, $blocked_exts)) {
                        $upload_error = '不支持上传此类型文件，请上传其他格式。';
                    } else {
                        $finfo = finfo_open(FILEINFO_MIME_TYPE);
                        $mime = finfo_file($finfo, $file['tmp_name']);
                        finfo_close($finfo);
                        $new_name = time() . '_' . bin2hex(random_bytes(8)) . '.' . $ext;
                        $dest = $upload_dir . $new_name;
                        if (move_uploaded_file($file['tmp_name'], $dest)) {
                            $media_path = 'uploads/' . $new_name;
                        } else {
                            $upload_error = '文件保存失败，请检查目录权限。';
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
        $upload_error = '昵称和内容不能为空。';
    }
}

// ========== 获取排序参数 ==========
$sort = isset($_GET['sort']) && in_array($_GET['sort'], ['latest', 'hot']) ? $_GET['sort'] : 'latest';

// ========== 获取留言列表 ==========
$orderBy = $sort === 'latest' ? 'created_at DESC' : 'likes DESC';
$query = "SELECT * FROM messages ORDER BY is_pinned DESC, $orderBy";
$result = $db->query($query);
$messages = [];
while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
    $messages[] = $row;
}

$is_admin = isset($_SESSION['admin']) && $_SESSION['admin'] === true;

// ========== 获取访客统计数据 ==========
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
    <title>✨ 安然小栈 · 留言板</title>
    <style>
        /* ======================================================
                   完整样式表（见下方完整代码）
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
            📬 安然小栈
            <small>图文·视频·附件</small>
        </div>
        <div class="header-right">
            <div class="admin-bar">
                <?php if ($is_admin): ?>
                    <span class="admin-status">✅ 管理员 <span class="logged-in">已登录</span></span>
                    <a href="?logout=1" class="btn-logout">🚪 退出</a>
                <?php else: ?>
                    <span class="admin-status">🔐 管理员 <span class="logged-out">未登录</span></span>
                    <form method="POST" action="" style="display: flex; gap: 0.3rem; flex-wrap: wrap; align-items: center;">
                        <input type="password" name="admin_password" placeholder="密码" required>
                        <button type="submit" class="btn-admin">登录</button>
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
        <span class="device">设备: <?= htmlspecialchars($visitor_device) ?></span>
        <span class="sep">|</span>
        <span class="stat-item">📊 总访问量: <?= number_format($totalViews) ?></span>
        <span class="stat-item">📎 文件上限 <?= MAX_FILE_SIZE / 1024 / 1024 / 1024 ?>GB</span>
    </div>

    <div class="daily-quote">
        <span class="quote-icon">💬</span>
        <span class="quote-text">“<?= htmlspecialchars($dailyQuote[0]) ?>”</span>
        <span class="quote-author">—— <?= htmlspecialchars($dailyQuote[1]) ?></span>
    </div>

    <?php if (isset($_GET['success'])): ?>
        <div class="toast success">✅ 留言发送成功！感谢你的分享 💬</div>
    <?php endif; ?>
    <?php if (isset($_GET['delete_success'])): ?>
        <div class="toast success">🗑️ 留言已成功删除。</div>
    <?php endif; ?>
    <?php if (isset($_GET['login_success'])): ?>
        <div class="toast success">✅ 管理员登录成功！</div>
    <?php endif; ?>
    <?php if (isset($_GET['logout_success'])): ?>
        <div class="toast info">👋 已退出管理。</div>
    <?php endif; ?>
    <?php if (isset($_GET['pin_success'])): ?>
        <div class="toast success">📌 已置顶留言。</div>
    <?php endif; ?>
    <?php if (isset($_GET['unpin_success'])): ?>
        <div class="toast info">📌 已取消置顶留言。</div>
    <?php endif; ?>
    <?php if ($upload_error !== null && $_SERVER['REQUEST_METHOD'] === 'POST' && !isset($_POST['admin_password'])): ?>
        <div class="toast error">⚠️ <?= htmlspecialchars($upload_error) ?></div>
    <?php endif; ?>

    <div class="form-card">
        <form method="POST" action="" enctype="multipart/form-data" id="messageForm">
            <div class="form-group">
                <label for="name">👤 昵称</label>
                <input type="text" id="name" name="name" placeholder="你的名字（可选）" maxlength="30">
            </div>
            <div class="form-group">
                <label for="content">✏️ 留言内容</label>
                <textarea id="content" name="content" placeholder="说点什么吧…" required></textarea>
            </div>
            <div class="form-group">
                <label for="media">📎 上传文件（图片/视频/文档等）</label>
                <input type="file" id="media" name="media">
                <div class="file-hint">支持图片、视频、PDF、ZIP、Word、Excel、PPT、TXT 等常见格式（最大 <?= MAX_FILE_SIZE / 1024 / 1024 / 1024 ?>GB）</div>
            </div>
            <button type="submit" class="btn-submit">📨 发送留言</button>
        </form>
        <div class="draft-notice" id="draftNotice">💾 草稿已保存</div>
    </div>

    <div class="sort-bar">
        <a href="?sort=latest" class="<?= $sort === 'latest' ? 'active' : '' ?>">🕒 最新</a>
        <a href="?sort=hot" class="<?= $sort === 'hot' ? 'active' : '' ?>">🔥 热门</a>
    </div>

    <div class="msg-list">
        <?php if (count($messages) === 0): ?>
            <div class="empty-msg">🌱 还没有留言，来做第一个吧！</div>
        <?php else: ?>
            <?php foreach ($messages as $msg): ?>
                <div class="msg-item <?= $msg['is_pinned'] ? 'pinned' : '' ?>" data-id="<?= $msg['id'] ?>">
                    <div class="msg-header">
                        <span class="msg-name"><?= htmlspecialchars($msg['name'] ?: '匿名') ?></span>
                        <div class="msg-meta">
                            <span class="ip">IP: <?= htmlspecialchars($msg['ip_address'] ?? '未知') ?></span>
                            <span class="device">设备: <?= htmlspecialchars(parse_user_agent($msg['user_agent'] ?? '')) ?></span>
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
                                <img src="<?= $file_path ?>" alt="留言图片" loading="lazy">
                            </div>
                        <?php elseif (in_array($ext, ['mp4', 'webm', 'ogg'])): ?>
                            <div class="msg-media">
                                <video controls preload="none" width="100%">
                                    <source src="<?= $file_path ?>" type="video/<?= $ext === 'mp4' ? 'mp4' : ($ext === 'webm' ? 'webm' : 'ogg') ?>">
                                    您的浏览器不支持视频播放。
                                </video>
                            </div>
                        <?php else: ?>
                            <div>
                                <a href="<?= $file_path ?>" class="attachment-download" download>
                                    <span class="file-icon">📄</span>
                                    下载附件 (<?= htmlspecialchars($ext) ?>)
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
                                    <a href="?unpin=<?= $msg['id'] ?>" class="btn-admin-action pin">取消置顶</a>
                                <?php else: ?>
                                    <a href="?pin=<?= $msg['id'] ?>" class="btn-admin-action">📌 置顶</a>
                                <?php endif; ?>
                                <a href="?delete=<?= $msg['id'] ?>" class="btn-delete" onclick="return confirm('确定要删除这条留言吗？')">🗑️ 删除</a>
                            <?php endif; ?>
                        </div>
                    </div>
                </div>
            <?php endforeach; ?>
        <?php endif; ?>
    </div>

    <?php if (!empty($visitorStats)): ?>
    <div class="stats-section">
        <h3>📊 访客统计</h3>
        <table class="stats-table">
            <thead>
                <tr>
                    <th>IP 地址</th>
                    <th>设备</th>
                    <th>浏览次数</th>
                    <th>点赞总数</th>
                    <th>最后活跃时间</th>
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
                    $actionLabel = '浏览';
                    $actionClass = 'action-view';
                } else {
                    $actionLabel = '点赞';
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
                <a href="?page=<?= $currentPage - 1 ?>&sort=<?= $sort ?>">‹ 上一页</a>
            <?php else: ?>
                <span class="disabled">‹ 上一页</span>
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
                <a href="?page=<?= $currentPage + 1 ?>&sort=<?= $sort ?>">下一页 ›</a>
            <?php else: ?>
                <span class="disabled">下一页 ›</span>
            <?php endif; ?>
        </div>
        <div class="stats-note">共 <?= $totalStats ?> 条记录，当前第 <?= $currentPage ?> / <?= $totalPages ?> 页</div>
        <?php endif; ?>

        <?php if (!$is_admin && count($visitorStats) > 5): ?>
            <div class="stats-note">🔒 仅显示前5条，<a href="#" onclick="document.querySelector('.admin-bar input[type=password]').focus(); return false;">登录管理员</a>可查看全部。</div>
        <?php endif; ?>
        <?php if ($is_admin && $totalStats <= $perPage): ?>
            <div class="stats-note">👑 管理员已登录，共 <?= $totalStats ?> 条记录。</div>
        <?php endif; ?>
    </div>
    <?php endif; ?>

    <footer>
        ❤️ Powered by <a href="https://debian.org" target="_blank">Debian 13</a> &amp; <a href="https://nginx.org" target="_blank">Nginx</a> · 安然如故
    </footer>
</div>

<audio id="bgMusic" src="/music/background.mp3" preload="metadata" loop></audio>
<button id="music-btn" aria-label="切换背景音乐">🎵</button>

<script>
    (function() {
        // ------ 音乐播放器 ------
        const audio = document.getElementById('bgMusic');
        const btn = document.getElementById('music-btn');
        btn.textContent = '🎵';
        btn.addEventListener('click', function() {
            if (audio.paused) {
                audio.play().catch(function(e) { console.log('播放失败:', e); });
                btn.textContent = '🔊';
            } else {
                audio.pause();
                btn.textContent = '🎵';
            }
        });
        audio.addEventListener('ended', function() { btn.textContent = '🎵'; });
        audio.addEventListener('error', function() {
            console.warn('音乐加载失败，请检查 /music/background.mp3');
            btn.textContent = '⚠️';
            btn.style.cursor = 'default';
            btn.disabled = true;
        });

        // ------ 点赞功能 ------
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
                    .catch(error => console.error('AJAX错误:', error));
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

        // ------ 自动保存草稿 ------
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
                        draftNotice.textContent = '📂 草稿已恢复';
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

> **注意**：请务必将 `define('ADMIN_PASSWORD', 'your_password_here');` 中的密码修改为强密码。

---

## 🎵 第七步：背景音乐（可选）

如果想添加背景音乐，按以下步骤操作：

```bash
# 创建音乐目录
mkdir -p /var/www/your_domain/music

# 上传你的 MP3 文件（使用 scp 或 sftp）
# 示例：scp /本地路径/background.mp3 root@服务器IP:/var/www/your_domain/music/
```

确保文件名为 `background.mp3`，代码中的播放器会自动加载。

---

## 🧪 第八步：测试与验证

完成以上步骤后，访问你的域名：

```
https://your_domain.com
```

你应该能看到完整的留言板界面。可以尝试：

1. 发布一条留言（带或不带附件）
2. 点击点赞按钮（长按可连续点赞，伴有爱心特效）
3. 切换“最新”和“热门”排序
4. 管理员登录（点击右上角“管理员”，输入密码）
5. 置顶或删除留言
6. 刷新页面查看“每日一言”变化
7. 查看访客统计

---

## 🔧 常见问题排查

### Q1：上传文件失败（413 Request Entity Too Large）

**原因**：Nginx 或 PHP 的上传大小限制不足。

**解决**：检查 Nginx 配置中的 `client_max_body_size` 和 PHP 配置中的 `upload_max_filesize`、`post_max_size`，确保它们足够大，然后重启服务。

### Q2：留言中的图片/视频不显示

**原因**：上传目录权限不正确，或 Nginx 没有正确访问。

**解决**：
```bash
sudo chown -R www-data:www-data /var/www/your_domain/uploads
sudo chmod 755 /var/www/your_domain/uploads
```

### Q3：管理员密码忘记

**解决**：在 `index.php` 中修改 `define('ADMIN_PASSWORD', 'new_password');`，然后刷新页面。

### Q4：SQLite 数据库写入失败

**原因**：目录权限不足。

**解决**：
```bash
sudo chown www-data:www-data /var/www/your_domain
sudo chmod 755 /var/www/your_domain
```

---

## 📊 数据备份建议

数据库文件为 `/var/www/your_domain/messages.db`，建议定期备份：

```bash
# 手动备份
sudo cp /var/www/your_domain/messages.db /backup/messages_$(date +%Y%m%d).db

# 或设置定时任务（crontab）
# 每天凌晨2点备份
0 2 * * * cp /var/www/your_domain/messages.db /backup/messages_$(date +\%Y\%m\%d).db
```

---

## 🚀 性能优化建议

1. **开启 Nginx 缓存**：为静态资源（图片、CSS、JS）设置缓存头。
2. **使用 Redis 缓存**：如果有高并发需求，可以引入 Redis 缓存热门留言列表。
3. **定期清理上传目录**：对于过期或无关联的附件，可编写脚本定期清理。
4. **启用 Gzip 压缩**：减少传输体积，提升加载速度。

---

## 🎯 总结

至此，你已经成功部署了一个功能完整的留言板网站。该项目具备以下特点：

- ✅ **轻量级**：基于 SQLite，无需额外数据库服务
- ✅ **功能丰富**：包含点赞、置顶、排序、附件上传、访客统计等
- ✅ **安全可靠**：支持 HTTPS，上传文件类型限制，防注入处理
- ✅ **响应式设计**：适配桌面和移动设备
- ✅ **易于维护**：单文件架构，方便修改和扩展

你可以根据实际需求继续定制，例如：
- 增加验证码功能
- 添加邮件通知新留言
- 接入 CDN 加速静态资源
- 增加用户注册登录系统

