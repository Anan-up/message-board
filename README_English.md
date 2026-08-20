# Build a Full-Featured Message Board Website from Scratch

> A complete deployment guide with source code, configuration notes, and troubleshooting

## 📌 Introduction

In the internet age, a message board is still a highly valuable interactive tool. Whether it's a personal blog, a small community, or internal corporate communication, a simple and full-featured message board can play an important role.

This guide walks you through deploying a feature-rich message board website from scratch on a Linux server. The project uses a lightweight **Nginx + PHP + SQLite** architecture — no need to install MySQL — with very low resource consumption, making it suitable for cloud servers and even a Raspberry Pi.

**Final feature preview:**

- Post messages (nickname, content, and file attachments)
- Images/videos are auto-detected and displayed; other files are offered as downloads
- Unlimited likes (hold to like repeatedly + floating-heart effect)
- Message sorting (Latest / Hot)
- Admin pinning and deletion
- Visitor statistics (IP, device, view count, total likes)
- Daily quote (random on each refresh)
- Total visit counter
- Auto-saved drafts
- Background music player
- Full-site HTTPS support

---

## 🛠 Tech Stack

| Component | Version | Description |
|------|------|------|
| Operating System | Debian 13.2 | Also compatible with Ubuntu 22.04+ |
| Web Server | Nginx 1.24+ | High-performance HTTP server |
| PHP | PHP 8.3 / 8.4 | Backend logic |
| Database | SQLite 3 | Lightweight file-based database |
| Frontend | Native HTML + CSS + JavaScript | No framework required |

---

## 📦 Requirements

- A Linux server (Debian 12+ or Ubuntu 22.04+ recommended)
- A domain name pointing to your server's IP (for HTTPS)
- An SSL certificate (this guide uses an existing certificate; you can also obtain a free one from Let's Encrypt)

---

## 🚀 Step 1: Environment Setup

After logging into the server, first update the system and install the required packages:

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Install Nginx, PHP, and related extensions
sudo apt install -y nginx php-fpm php-sqlite3 sqlite3 curl wget unzip

# Check the PHP version (needed for later configuration)
php -v
```

---

## 📁 Step 2: Create the Site Directory

```bash
# Create the site root directory
sudo mkdir -p /var/www/your_domain

# Create the upload directory (for storing user-uploaded attachments)
sudo mkdir -p /var/www/your_domain/uploads

# Set permissions
sudo chown -R www-data:www-data /var/www/your_domain
sudo chmod 755 /var/www/your_domain/uploads
```

> **Note**: Replace `your_domain` with your actual domain name.

---

## 🔐 Step 3: Prepare the SSL Certificate (optional but recommended)

If you already have certificate files (`.crt` and `.key`), place them on the server (for example `/path/to/ssl/`). If not, you can obtain a free certificate from Let's Encrypt:

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Automatically issue a certificate (the domain must already resolve)
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
```

---

## 📄 Step 4: Nginx Site Configuration

Create the Nginx virtual host configuration file:

```bash
sudo nano /etc/nginx/sites-available/your_domain
```

Paste the following content (adjust the domain and certificate paths to match your setup):

```nginx
# HTTP -> HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name your_domain.com www.your_domain.com;
    return 301 https://$server_name$request_uri;
}

# Main HTTPS site
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name your_domain.com www.your_domain.com;

    root /var/www/your_domain;
    index index.php index.html;

    # SSL certificate configuration
    ssl_certificate     /path/to/ssl/your_domain_bundle.crt;
    ssl_certificate_key /path/to/ssl/your_domain.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Logs (optional)
    access_log /var/log/nginx/your_domain.access.log;
    error_log  /var/log/nginx/your_domain.error.log;

    # Upload directory: disable PHP execution
    location /uploads/ {
        location ~ \.php$ {
            deny all;
        }
    }

    # Raise the upload size limit (keep it consistent with the PHP config)
    client_max_body_size 10G;

    # Handle PHP requests
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 600s;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
```

Enable the site and reload Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🐘 Step 5: Tune the PHP Configuration

Edit the PHP-FPM configuration file (the path may vary by version):

```bash
sudo nano /etc/php/8.4/fpm/php.ini
```

Modify the following parameters (to support large file uploads):

```ini
upload_max_filesize = 10G
post_max_size = 10G
max_execution_time = 600
max_input_time = 600
memory_limit = 512M
```

Restart PHP-FPM:

```bash
sudo systemctl restart php8.4-fpm
```

---

## 📄 Step 6: Complete Source Code `index.php`

This is the core file of the project and contains all the message board features. Save it as `/var/www/your_domain/index.php`:

```php
<?php
date_default_timezone_set('Asia/Shanghai');
session_start();

// ========== Database initialization ==========
$db_file = 'messages.db';
$db = new SQLite3($db_file);

// Create the messages table (if it doesn't exist)
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

// Check for and add any missing columns (for compatibility with older databases)
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

// ========== Create the visitor log table ==========
$db->exec("CREATE TABLE IF NOT EXISTS visitor_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ip_address TEXT NOT NULL,
    user_agent TEXT,
    action TEXT NOT NULL,
    message_id INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
)");

// ========== Admin password ==========
define('ADMIN_PASSWORD', 'your_password_here'); // Change this to a strong password

// ========== Handle admin login ==========
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['admin_password'])) {
    if ($_POST['admin_password'] === ADMIN_PASSWORD) {
        $_SESSION['admin'] = true;
        header('Location: ?login_success=1');
        exit;
    } else {
        $login_error = 'Incorrect password. Please try again.';
    }
}

// ========== Handle admin logout ==========
if (isset($_GET['logout'])) {
    unset($_SESSION['admin']);
    session_destroy();
    header('Location: ?logout_success=1');
    exit;
}

// ========== Handle message deletion ==========
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

// ========== Handle pin/unpin ==========
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

// ========== Handle likes (AJAX) ==========
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
        echo json_encode(['status' => 'error', 'message' => 'Invalid ID']);
    }
    exit;
}

// ========== Log page views ==========
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

// ========== Helper functions ==========
function parse_user_agent($ua) {
    if (empty($ua)) return 'Unknown device';
    $os = 'Unknown OS'; $browser = 'Unknown browser';
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
    if (empty($utc_string)) return 'Unknown';
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

// ========== Daily quote (random on each refresh) ==========
function getDailyQuote() {
    $quotes = [
        ['Life is not only about the struggles in front of you, but also poetry and distant lands.', 'Gao Xiaosong'],
        ['May you return still young at heart after a long journey through life.', 'Anonymous'],
        ['The world has kissed my soul with its pain, asking for its return in songs.', 'Rabindranath Tagore'],
        ['Let life be beautiful like summer flowers and death like autumn leaves.', 'Rabindranath Tagore'],
        ['The dark night gave me black eyes, but I use them to seek the light.', 'Gu Cheng'],
        ['Facing the sea, with spring flowers blossoming.', 'Hai Zi'],
        ['Every day without dancing is a betrayal of life.', 'Friedrich Nietzsche'],
        ['Life isn\'t about waiting for the storm to pass, but learning to dance in the rain.', 'Anonymous'],
        ['We once longed for the waves of destiny, only to find that life\'s most beautiful scenery is the calm and composure within.', 'Yang Jiang'],
        ['A man can be destroyed but not defeated.', 'Ernest Hemingway'],
        ['There is only one heroism in the world: to see the world as it is and to love it.', 'Romain Rolland'],
        ['The road ahead is long and far; I shall search high and low.', 'Qu Yuan'],
        ['A time will come to ride the wind and cleave the waves; I\'ll set my cloud-white sail and cross the sea.', 'Li Bai'],
        ['Heaven gave me talents for a purpose; a thousand gold pieces spent will come back again.', 'Li Bai'],
        ['Could I get mansions covering ten thousand miles, I\'d house all the poor scholars and make them beam with smiles.', 'Du Fu'],
        ['Plucking chrysanthemums by the eastern hedge, I gaze at the southern hills in leisure.', 'Tao Yuanming'],
        ['A thousand sails pass by the sunken ship; ten thousand trees burst into spring before the withered tree.', 'Liu Yuxi'],
        ['Mountains multiply and streams double back — no way out; then willows darken and flowers brighten — another village appears.', 'Lu You'],
        ['Seen from the side it\'s a range, from the front a peak; far, near, high and low, all different.', 'Su Shi'],
        ['May we all be blessed with longevity; though far apart, we share the same moon.', 'Su Shi'],
        ['Men have sorrow and joy, parting and reunion; the moon has shade and light, waxing and waning.', 'Su Shi'],
        ['I fear not the clouds that obscure my view, for I stand on the highest level.', 'Wang Anshi'],
        ['Ask the channel how it stays so clear — for fresh water flows in from the source.', 'Zhu Xi'],
        ['What\'s learned on paper always feels shallow; to truly know a thing you must practice it yourself.', 'Lu You'],
        ['The sharpest sword comes from grinding; the plum blossom\'s fragrance comes from the bitter cold.', 'Anonymous'],
        ['Mastery comes from diligence and is ruined by play; deeds are accomplished through thought and destroyed by carelessness.', 'Han Yu'],
        ['Diligence is the path up the mountain of books; hard work is the boat across the boundless sea of learning.', 'Han Yu'],
        ['When three walk together, one is sure to be my teacher.', 'Confucius'],
        ['Learning without thought is labor lost; thought without learning is perilous.', 'Confucius'],
        ['Reviewing the old and learning the new makes one fit to be a teacher.', 'Confucius'],
        ['The noble person seeks harmony but not uniformity; the petty person seeks uniformity but not harmony.', 'Confucius'],
        ['The noble person is calm and at ease; the petty person is fretful and anxious.', 'Confucius'],
        ['In poverty, cultivate yourself; in prosperity, benefit the world.', 'Mencius'],
        ['Neither riches nor honors can corrupt, neither poverty nor lowliness can sway, neither threats nor force can bend.', 'Mencius'],
        ['As heaven keeps moving vigorously, the noble person strives unceasingly to strengthen themselves.', 'I Ching'],
        ['As the earth is vast and generous, the noble person bears all things with great virtue.', 'I Ching'],
        ['Strong wind reveals the sturdy grass; troubled times reveal the loyal minister.', 'Li Shimin'],
        ['The wide sea lets fish leap freely; the vast sky lets birds fly high.', 'Anonymous'],
        ['A bosom friend afar brings a distant land near.', 'Wang Bo'],
        ['Sunset clouds fly with the lonely wild duck; autumn water merges with the boundless sky.', 'Wang Bo'],
        ['Be the first to worry about the world\'s troubles and the last to enjoy its pleasures.', 'Fan Zhongyan'],
        ['Do not rejoice over external gains nor grieve over personal losses.', 'Fan Zhongyan'],
        ['Don\'t laugh as I lie drunk on the battlefield — how many warriors ever return from war?', 'Wang Han'],
    ];
    $index = mt_rand(0, count($quotes) - 1);
    return $quotes[$index];
}

// ========== Get current visitor info ==========
$visitor_ip = $_SERVER['REMOTE_ADDR'];
$visitor_ua = $_SERVER['HTTP_USER_AGENT'] ?? '';
$visitor_device = parse_user_agent($visitor_ua);

// ========== Get total views ==========
$totalViews = $db->querySingle("SELECT COUNT(*) FROM visitor_logs WHERE action='view'");
if ($totalViews === null || $totalViews === false) $totalViews = 0;

// ========== Get the daily quote ==========
$dailyQuote = getDailyQuote();

// ========== File upload handling ==========
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
                $upload_error = 'File upload failed, error code: ' . $file['error'];
            } else {
                if ($file['size'] > MAX_FILE_SIZE) {
                    $upload_error = 'File size exceeds the ' . (MAX_FILE_SIZE / 1024 / 1024 / 1024) . 'GB limit.';
                } else {
                    $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
                    if (in_array($ext, $blocked_exts)) {
                        $upload_error = 'This file type is not supported. Please upload a different format.';
                    } else {
                        $finfo = finfo_open(FILEINFO_MIME_TYPE);
                        $mime = finfo_file($finfo, $file['tmp_name']);
                        finfo_close($finfo);
                        $new_name = time() . '_' . bin2hex(random_bytes(8)) . '.' . $ext;
                        $dest = $upload_dir . $new_name;
                        if (move_uploaded_file($file['tmp_name'], $dest)) {
                            $media_path = 'uploads/' . $new_name;
                        } else {
                            $upload_error = 'Failed to save the file. Please check directory permissions.';
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
        $upload_error = 'Name and content cannot be empty.';
    }
}

// ========== Get the sort parameter ==========
$sort = isset($_GET['sort']) && in_array($_GET['sort'], ['latest', 'hot']) ? $_GET['sort'] : 'latest';

// ========== Get the message list ==========
$orderBy = $sort === 'latest' ? 'created_at DESC' : 'likes DESC';
$query = "SELECT * FROM messages ORDER BY is_pinned DESC, $orderBy";
$result = $db->query($query);
$messages = [];
while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
    $messages[] = $row;
}

$is_admin = isset($_SESSION['admin']) && $_SESSION['admin'] === true;

// ========== Get visitor statistics ==========
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
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=5.0">
    <title>✨ Anran's Nook · Message Board</title>
    <style>
        /* ======================================================
                   Full stylesheet (see the complete code below)
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
            📬 Anran's Nook
            <small>Images · Videos · Attachments</small>
        </div>
        <div class="header-right">
            <div class="admin-bar">
                <?php if ($is_admin): ?>
                    <span class="admin-status">✅ Admin <span class="logged-in">logged in</span></span>
                    <a href="?logout=1" class="btn-logout">🚪 Logout</a>
                <?php else: ?>
                    <span class="admin-status">🔐 Admin <span class="logged-out">logged out</span></span>
                    <form method="POST" action="" style="display: flex; gap: 0.3rem; flex-wrap: wrap; align-items: center;">
                        <input type="password" name="admin_password" placeholder="Password" required>
                        <button type="submit" class="btn-admin">Login</button>
                    </form>
                    <?php if (isset($login_error)): ?>
                        <span class="admin-error"><?= htmlspecialchars($login_error) ?></span>
                    <?php endif; ?>
                <?php endif; ?>
            </div>
        </div>
    </div>

    <div class="visitor-info">
        <span class="label">📡 Your info</span>
        <span class="ip">IP: <?= htmlspecialchars($visitor_ip) ?></span>
        <span class="sep">|</span>
        <span class="device">Device: <?= htmlspecialchars($visitor_device) ?></span>
        <span class="sep">|</span>
        <span class="stat-item">📊 Total views: <?= number_format($totalViews) ?></span>
        <span class="stat-item">📎 Max file <?= MAX_FILE_SIZE / 1024 / 1024 / 1024 ?>GB</span>
    </div>

    <div class="daily-quote">
        <span class="quote-icon">💬</span>
        <span class="quote-text">“<?= htmlspecialchars($dailyQuote[0]) ?>”</span>
        <span class="quote-author">—— <?= htmlspecialchars($dailyQuote[1]) ?></span>
    </div>

    <?php if (isset($_GET['success'])): ?>
        <div class="toast success">✅ Message posted! Thanks for sharing 💬</div>
    <?php endif; ?>
    <?php if (isset($_GET['delete_success'])): ?>
        <div class="toast success">🗑️ Message deleted successfully.</div>
    <?php endif; ?>
    <?php if (isset($_GET['login_success'])): ?>
        <div class="toast success">✅ Admin logged in successfully!</div>
    <?php endif; ?>
    <?php if (isset($_GET['logout_success'])): ?>
        <div class="toast info">👋 Logged out of admin.</div>
    <?php endif; ?>
    <?php if (isset($_GET['pin_success'])): ?>
        <div class="toast success">📌 Message pinned.</div>
    <?php endif; ?>
    <?php if (isset($_GET['unpin_success'])): ?>
        <div class="toast info">📌 Message unpinned.</div>
    <?php endif; ?>
    <?php if ($upload_error !== null && $_SERVER['REQUEST_METHOD'] === 'POST' && !isset($_POST['admin_password'])): ?>
        <div class="toast error">⚠️ <?= htmlspecialchars($upload_error) ?></div>
    <?php endif; ?>

    <div class="form-card">
        <form method="POST" action="" enctype="multipart/form-data" id="messageForm">
            <div class="form-group">
                <label for="name">👤 Name</label>
                <input type="text" id="name" name="name" placeholder="Your name (optional)" maxlength="30">
            </div>
            <div class="form-group">
                <label for="content">✏️ Message</label>
                <textarea id="content" name="content" placeholder="Say something…" required></textarea>
            </div>
            <div class="form-group">
                <label for="media">📎 Upload a file (image/video/document, etc.)</label>
                <input type="file" id="media" name="media">
                <div class="file-hint">Supports images, videos, PDF, ZIP, Word, Excel, PPT, TXT and other common formats (up to <?= MAX_FILE_SIZE / 1024 / 1024 / 1024 ?>GB)</div>
            </div>
            <button type="submit" class="btn-submit">📨 Post message</button>
        </form>
        <div class="draft-notice" id="draftNotice">💾 Draft saved</div>
    </div>

    <div class="sort-bar">
        <a href="?sort=latest" class="<?= $sort === 'latest' ? 'active' : '' ?>">🕒 Latest</a>
        <a href="?sort=hot" class="<?= $sort === 'hot' ? 'active' : '' ?>">🔥 Hot</a>
    </div>

    <div class="msg-list">
        <?php if (count($messages) === 0): ?>
            <div class="empty-msg">🌱 No messages yet — be the first!</div>
        <?php else: ?>
            <?php foreach ($messages as $msg): ?>
                <div class="msg-item <?= $msg['is_pinned'] ? 'pinned' : '' ?>" data-id="<?= $msg['id'] ?>">
                    <div class="msg-header">
                        <span class="msg-name"><?= htmlspecialchars($msg['name'] ?: 'Anonymous') ?></span>
                        <div class="msg-meta">
                            <span class="ip">IP: <?= htmlspecialchars($msg['ip_address'] ?? 'Unknown') ?></span>
                            <span class="device">Device: <?= htmlspecialchars(parse_user_agent($msg['user_agent'] ?? '')) ?></span>
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
                                <img src="<?= $file_path ?>" alt="Message image" loading="lazy">
                            </div>
                        <?php elseif (in_array($ext, ['mp4', 'webm', 'ogg'])): ?>
                            <div class="msg-media">
                                <video controls preload="none" width="100%">
                                    <source src="<?= $file_path ?>" type="video/<?= $ext === 'mp4' ? 'mp4' : ($ext === 'webm' ? 'webm' : 'ogg') ?>">
                                    Your browser does not support video playback.
                                </video>
                            </div>
                        <?php else: ?>
                            <div>
                                <a href="<?= $file_path ?>" class="attachment-download" download>
                                    <span class="file-icon">📄</span>
                                    Download attachment (<?= htmlspecialchars($ext) ?>)
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
                                    <a href="?unpin=<?= $msg['id'] ?>" class="btn-admin-action pin">Unpin</a>
                                <?php else: ?>
                                    <a href="?pin=<?= $msg['id'] ?>" class="btn-admin-action">📌 Pin</a>
                                <?php endif; ?>
                                <a href="?delete=<?= $msg['id'] ?>" class="btn-delete" onclick="return confirm('Delete this message?')">🗑️ Delete</a>
                            <?php endif; ?>
                        </div>
                    </div>
                </div>
            <?php endforeach; ?>
        <?php endif; ?>
    </div>

    <?php if (!empty($visitorStats)): ?>
    <div class="stats-section">
        <h3>📊 Visitor statistics</h3>
        <table class="stats-table">
            <thead>
                <tr>
                    <th>IP address</th>
                    <th>Device</th>
                    <th>Views</th>
                    <th>Total likes</th>
                    <th>Last active</th>
                    <th>Recent action</th>
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
                    $actionLabel = 'View';
                    $actionClass = 'action-view';
                } else {
                    $actionLabel = 'Like';
                    if ($is_admin && $stat['last_message_id']) {
                        $actionLabel .= ' (message #' . $stat['last_message_id'] . ')';
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
                <a href="?page=<?= $currentPage - 1 ?>&sort=<?= $sort ?>">‹ Prev</a>
            <?php else: ?>
                <span class="disabled">‹ Prev</span>
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
                <a href="?page=<?= $currentPage + 1 ?>&sort=<?= $sort ?>">Next ›</a>
            <?php else: ?>
                <span class="disabled">Next ›</span>
            <?php endif; ?>
        </div>
        <div class="stats-note"><?= $totalStats ?> records in total, page <?= $currentPage ?> / <?= $totalPages ?></div>
        <?php endif; ?>

        <?php if (!$is_admin && count($visitorStats) > 5): ?>
            <div class="stats-note">🔒 Showing the first 5 only — <a href="#" onclick="document.querySelector('.admin-bar input[type=password]').focus(); return false;">log in as admin</a> to view all.</div>
        <?php endif; ?>
        <?php if ($is_admin && $totalStats <= $perPage): ?>
            <div class="stats-note">👑 Admin logged in, <?= $totalStats ?> records in total.</div>
        <?php endif; ?>
    </div>
    <?php endif; ?>

    <footer>
        ❤️ Powered by <a href="https://debian.org" target="_blank">Debian 13</a> &amp; <a href="https://nginx.org" target="_blank">Nginx</a> · Peaceful as ever
    </footer>
</div>

<audio id="bgMusic" src="/music/background.mp3" preload="metadata" loop></audio>
<button id="music-btn" aria-label="Toggle background music">🎵</button>

<script>
    (function() {
        // ------ Music player ------
        const audio = document.getElementById('bgMusic');
        const btn = document.getElementById('music-btn');
        btn.textContent = '🎵';
        btn.addEventListener('click', function() {
            if (audio.paused) {
                audio.play().catch(function(e) { console.log('Playback failed:', e); });
                btn.textContent = '🔊';
            } else {
                audio.pause();
                btn.textContent = '🎵';
            }
        });
        audio.addEventListener('ended', function() { btn.textContent = '🎵'; });
        audio.addEventListener('error', function() {
            console.warn('Failed to load music, please check /music/background.mp3');
            btn.textContent = '⚠️';
            btn.style.cursor = 'default';
            btn.disabled = true;
        });

        // ------ Like feature ------
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
                    .catch(error => console.error('AJAX error:', error));
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

        // ------ Auto-save draft ------
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
                        draftNotice.textContent = '📂 Draft restored';
                        draftNotice.classList.add('visible');
                        setTimeout(() => {
                            draftNotice.classList.remove('visible');
                            draftNotice.textContent = '💾 Draft saved';
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

> **Important**: Be sure to change the password in `define('ADMIN_PASSWORD', 'your_password_here');` to a strong password.

---

## 🎵 Step 7: Background Music (optional)

To add background music, follow these steps:

```bash
# Create the music directory
mkdir -p /var/www/your_domain/music

# Upload your MP3 file (using scp or sftp)
# Example: scp /local/path/background.mp3 root@server_ip:/var/www/your_domain/music/
```

Make sure the file is named `background.mp3`; the player in the code will load it automatically.

---

## 🧪 Step 8: Test and Verify

After completing the steps above, visit your domain:

```
https://your_domain.com
```

You should see the full message board interface. Try the following:

1. Post a message (with or without an attachment)
2. Click the like button (hold to like repeatedly, with a floating-heart effect)
3. Switch between "Latest" and "Hot" sorting
4. Log in as admin (click "Admin" in the top-right corner and enter the password)
5. Pin or delete messages
6. Refresh the page to see the "Daily Quote" change
7. View visitor statistics

---

## 🔧 Troubleshooting

### Q1: File upload fails (413 Request Entity Too Large)

**Cause**: The Nginx or PHP upload size limit is too small.

**Solution**: Check `client_max_body_size` in the Nginx config and `upload_max_filesize` / `post_max_size` in the PHP config, make sure they are large enough, then restart the services.

### Q2: Images/videos in messages don't display

**Cause**: Incorrect upload directory permissions, or Nginx cannot access it properly.

**Solution**:
```bash
sudo chown -R www-data:www-data /var/www/your_domain/uploads
sudo chmod 755 /var/www/your_domain/uploads
```

### Q3: Forgot the admin password

**Solution**: Change `define('ADMIN_PASSWORD', 'new_password');` in `index.php`, then refresh the page.

### Q4: SQLite database write fails

**Cause**: Insufficient directory permissions.

**Solution**:
```bash
sudo chown www-data:www-data /var/www/your_domain
sudo chmod 755 /var/www/your_domain
```

---

## 📊 Data Backup Recommendations

The database file is `/var/www/your_domain/messages.db`; back it up regularly:

```bash
# Manual backup
sudo cp /var/www/your_domain/messages.db /backup/messages_$(date +%Y%m%d).db

# Or set up a cron job
# Back up daily at 2:00 AM
0 2 * * * cp /var/www/your_domain/messages.db /backup/messages_$(date +\%Y\%m\%d).db
```

---

## 🚀 Performance Optimization Tips

1. **Enable Nginx caching**: Set cache headers for static assets (images, CSS, JS).
2. **Use Redis caching**: If you have high-concurrency needs, introduce Redis to cache the popular message list.
3. **Clean up the upload directory regularly**: Write a script to periodically remove expired or orphaned attachments.
4. **Enable Gzip compression**: Reduce transfer size and speed up loading.

---

## 🎯 Conclusion

You have now successfully deployed a fully functional message board website. The project has these features:

- ✅ **Lightweight**: Built on SQLite, no extra database service required
- ✅ **Feature-rich**: Includes likes, pinning, sorting, file uploads, visitor statistics, and more
- ✅ **Secure**: HTTPS support, upload file-type restrictions, injection protection
- ✅ **Responsive**: Works on desktop and mobile devices
- ✅ **Easy to maintain**: Single-file architecture, simple to modify and extend

You can continue customizing it to suit your needs, for example:
- Add a CAPTCHA
- Add email notifications for new messages
- Use a CDN to speed up static assets
- Add a user registration/login system
