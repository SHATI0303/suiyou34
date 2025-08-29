# 掲示板サービス構築ガイド

このガイドでは、**AWS EC2** と **Docker** を使用して、シンプルな Web 掲示板サービスを構築する手順を説明します。

---

## 要件

- テキスト投稿、画像アップロード機能  
- 投稿内容の MySQL への保存  
- XSS および SQL インジェクション対策済み  
- 自動連番と投稿日時の付与  
- 5MB を超える画像は JavaScript で自動縮小  

---

## 1. AWS EC2 インスタンスの準備

1. AWS マネジメントコンソールで **EC2 ダッシュボード**に移動し、「インスタンスを起動」をクリック  
2. AMI は **Amazon Linux 2** を選択  
3. インスタンスタイプは **t2.micro** を推奨  
4. セキュリティグループ設定  
   - SSH (22): 自分の IP から許可  
   - HTTP (80): すべての IP (0.0.0.0/0) から許可  
5. キーペアを作成してインスタンスを起動  
6. SSH でインスタンスに接続  

---

## 2. Docker 環境のセットアップ

EC2 に接続後、以下を実行してください。

# システムパッケージ更新
sudo yum update -y

# Docker インストール
sudo yum install docker -y

# Docker 起動 & 自動起動設定
sudo systemctl start docker
sudo systemctl enable docker

# sudo なしで利用できるようにユーザーを追加
sudo usermod -a -G docker ec2-user

# Docker Compose インストール
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

## 3. Docker コンテナの定義と設定
プロジェクトディレクトリ作成
mkdir bulletin-board
cd bulletin-board

docker-compose.yml
version: '3.8'

services:
  php:
    build: .
    volumes:
      - ./public:/var/www/public
    ports:
      - "9000:9000"
    depends_on:
      - mysql

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./public:/var/www/public
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
      - mysql

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: example_db
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:

Dockerfile
FROM php:8.0-fpm
RUN apt-get update && apt-get install -y \
    libonig-dev \
    libzip-dev \
    libjpeg-dev \
    libpng-dev \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-install pdo_mysql gd

ディレクトリ作成
mkdir nginx
mkdir public


4. アプリケーションとサーバー設定
nginx/default.conf
server {
    listen 80;
    server_name _;
    root /var/www/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

public/kadai.php
<details> <summary>クリックして表示</summary>

```php
<?php
// データベース接続
$dbh = new PDO('mysql:host=mysql;dbname=example_db', 'root', '');

// アラートメッセージ用の変数を初期化
$alert_message = null;

// 新規投稿処理
if (isset($_POST['body'])) {
    // 削除パスワードが空の場合はnullをセット
    $delete_password = !empty($_POST['delete_password']) ? password_hash($_POST['delete_password'], PASSWORD_DEFAULT) : null;
    
    $image_filename = null;
    if (isset($_FILES['image']) && !empty($_FILES['image']['tmp_name'])) {
        // アップロードされたファイルが画像であることを確認
        $mime_type = mime_content_type($_FILES['image']['tmp_name']);
        if (preg_match('/^image\//', $mime_type) !== 1) {
            header("Location: ./kadai.php");
            return;
        }

        // ファイル名を生成して保存
        $pathinfo = pathinfo($_FILES['image']['name']);
        $extension = $pathinfo['extension'];
        $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.' . $extension;
        $filepath =  '/var/www/upload/image/' . $image_filename;
        move_uploaded_file($_FILES['image']['tmp_name'], $filepath);
    }

    // データベースに投稿を挿入
    $insert_sth = $dbh->prepare("INSERT INTO bbs_entries (body, image_filename, delete_password) VALUES (:body, :image_filename, :delete_password)");
    $insert_sth->execute([
        ':body' => $_POST['body'],
        ':image_filename' => $image_filename,
        ':delete_password' => $delete_password,
    ]);

    // 処理後にリダイレクト
    header("Location: ./kadai.php");
    return;
}

// 投稿削除処理
if (isset($_POST['delete_id']) && isset($_POST['delete_password_check'])) {
    $select_sth = $dbh->prepare("SELECT delete_password FROM bbs_entries WHERE id = :id");
    $select_sth->execute([':id' => $_POST['delete_id']]);
    $entry = $select_sth->fetch();

    if ($entry && password_verify($_POST['delete_password_check'], $entry['delete_password'])) {
        // 削除成功時にIDをdeleted_entriesテーブルに記録
        $delete_id = $_POST['delete_id'];
        $insert_deleted_sth = $dbh->prepare("INSERT INTO deleted_entries (id, deleted_at) VALUES (:id, NOW())");
        $insert_deleted_sth->execute([':id' => $delete_id]);

        $delete_sth = $dbh->prepare("DELETE FROM bbs_entries WHERE id = :id");
        $delete_sth->execute([':id' => $delete_id]);
    } else {
        // パスワードが間違っていた場合、アラートメッセージを設定
        $alert_message = 'パスワードが違います。';
    }

    // リダイレクトはアラートメッセージがなければ実行
    if ($alert_message === null) {
        header("Location: ./kadai.php");
        return;
    }
}

// ページネーション設定
$page = isset($_GET['page']) ? (int)$_GET['page'] : 1;
$limit = 10;
$offset = ($page - 1) * $limit;

// 投稿データの取得 (ページネーション適用)
$select_sth = $dbh->prepare("SELECT * FROM bbs_entries ORDER BY created_at DESC LIMIT :limit OFFSET :offset");
$select_sth->bindValue(':limit', $limit, PDO::PARAM_INT);
$select_sth->bindValue(':offset', $offset, PDO::PARAM_INT);
$select_sth->execute();

// 総投稿数を取得
$total_count_sth = $dbh->prepare("SELECT COUNT(*) FROM bbs_entries");
$total_count_sth->execute();
$total_count = $total_count_sth->fetchColumn();
$total_pages = ceil($total_count / $limit);

// 投稿IDとページ番号の対応表を作成
// ページをまたぐレスアンカーのために必要
$all_ids_sth = $dbh->prepare("SELECT id FROM bbs_entries ORDER BY created_at DESC");
$all_ids_sth->execute();
$all_ids = $all_ids_sth->fetchAll(PDO::FETCH_COLUMN);

$id_to_page = [];
foreach ($all_ids as $index => $id) {
    $page_number = floor($index / $limit) + 1;
    $id_to_page[$id] = $page_number;
}
?>

<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>掲示板</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        body {
            font-family: sans-serif;
            line-height: 1.6;
            padding: 1em;
            margin: 0 auto;
            max-width: 600px;
            background-color: #f4f4f4;
            color: #333;
        }

        form {
            background: #fff;
            padding: 1.5em;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        }

        textarea {
            width: 100%;
            padding: 0.8em;
            box-sizing: border-box;
            border: 1px solid #ccc;
            border-radius: 4px;
            font-size: 1em;
            margin-bottom: 1em;
        }

        input[type="file"], input[type="password"] {
            display: block;
            width: 100%;
            padding: 0.8em;
            box-sizing: border-box;
            border: 1px solid #ccc;
            border-radius: 4px;
            margin-bottom: 1em;
        }

        button {
            width: 100%;
            padding: 0.8em;
            background-color: #007bff;
            color: #fff;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 1em;
        }

        button:hover {
            background-color: #0056b3;
        }

        hr {
            border: 0;
            height: 1px;
            background: #ccc;
            margin: 2em 0;
        }

        .entry {
            background: #fff;
            padding: 1.5em;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
            margin-bottom: 1em;
            position: relative;
        }

        .entry dt {
            font-weight: bold;
            color: #555;
            margin-top: 0.5em;
        }

        .entry dd {
            margin: 0;
            padding-bottom: 0.5em;
        }

        .entry img {
            max-width: 100%;
            height: auto;
            display: block;
            margin-top: 1em;
            border-radius: 4px;
            box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
        }

        .entry-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        
        .entry-id {
            font-size: 1.2em;
            color: #007bff;
            font-weight: bold;
            cursor: pointer;
            text-decoration: none;
        }
        
        .entry-id:hover {
            text-decoration: underline;
        }

        .entry-body-content {
            margin-top: 1em;
        }

        .res-link {
            color: #007bff;
            text-decoration: none;
        }

        .res-link:hover {
            text-decoration: underline;
        }

        .entry-footer {
            margin-top: 1em;
            display: flex;
            justify-content: flex-end;
            align-items: center;
        }

        .entry-footer input[type="password"] {
            margin-right: 0.5em;
            padding: 0.5em;
            width: auto;
            display: inline-block;
        }

        .delete-btn {
            padding: 0.5em 1em;
            background-color: #dc3545;
            color: #fff;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 1em;
            width: auto;
        }

        .delete-btn:hover {
            background-color: #c82333;
        }
        
        .pagination {
            display: flex;
            justify-content: center;
            align-items: center;
            margin-top: 2em;
            margin-bottom: 2em;
        }

        .pagination a, .pagination span {
            padding: 0.5em 1em;
            margin: 0 0.2em;
            border: 1px solid #ccc;
            text-decoration: none;
            color: #007bff;
            border-radius: 4px;
        }

        .pagination a:hover {
            background-color: #e9ecef;
        }

        .pagination .current-page {
            background-color: #007bff;
            color: #fff;
            border-color: #007bff;
        }
    </style>
</head>
<body>

<?php if ($alert_message): ?>
<script>
    alert('<?= htmlspecialchars($alert_message) ?>');
</script>
<?php endif; ?>

<form method="POST" action="./kadai.php" enctype="multipart/form-data" id="uploadForm">
  <textarea name="body" required placeholder="ここに本文を入力してください" id="bodyTextarea"></textarea>
  <div style="margin: 1em 0;">
    <input type="file" accept="image/*" name="image" id="imageInput">
  </div>
  <input type="password" name="delete_password" placeholder="削除パスワード (任意)">
  <button type="submit">送信</button>
</form>

<hr>

<div class="pagination">
    <?php for ($i = 1; $i <= $total_pages; $i++): ?>
        <?php if ($i == $page): ?>
            <span class="current-page"><?= $i ?></span>
        <?php else: ?>
            <a href="?page=<?= $i ?>"><?= $i ?></a>
        <?php endif; ?>
    <?php endfor; ?>
</div>

<?php foreach($select_sth as $entry): ?>
  <dl class="entry" id="entry-<?= htmlspecialchars($entry['id']) ?>">
    <div class="entry-header">
        <div class="entry-id" data-id="<?= htmlspecialchars($entry['id']) ?>">No.<?= htmlspecialchars($entry['id']) ?></div>
    </div>
    <dt>日時</dt>
    <dd><?= htmlspecialchars($entry['created_at']) ?></dd>
    <dt>内容</dt>
    <dd class="entry-body-content">
      <?php
      $escaped_body = htmlspecialchars($entry['body']);
      // ページをまたぐレスアンカーを実装するために、preg_replace_callbackを使用
      $linked_body = preg_replace_callback('/&gt;&gt;(\d+)/', function($matches) use ($id_to_page) {
          $target_id = $matches[1];
          $target_page = isset($id_to_page[$target_id]) ? $id_to_page[$target_id] : null;
          if ($target_page) {
              return '<a href="?page=' . $target_page . '#entry-' . $target_id . '" class="res-link">>>' . $target_id . '</a>';
          } else {
              return '>>' . $target_id; // 該当IDがない場合はリンクにしない
          }
      }, $escaped_body);
      echo nl2br($linked_body);
      ?>
      <?php if(!empty($entry['image_filename'])): ?>
      <div>
        <img src="/image/<?= htmlspecialchars($entry['image_filename']) ?>">
      </div>
      <?php endif; ?>
    </dd>
    <div class="entry-footer">
        <?php if (!empty($entry['delete_password'])): ?>
        <form method="POST" action="./kadai.php">
            <input type="hidden" name="delete_id" value="<?= htmlspecialchars($entry['id']) ?>">
            <input type="password" name="delete_password_check" placeholder="パスワード">
            <button type="submit" class="delete-btn">削除</button>
        </form>
        <?php endif; ?>
    </div>
  </dl>
<?php endforeach ?>

<script>
// 画像を5MB以下に自動縮小するスクリプト
document.getElementById('uploadForm').addEventListener('submit', function(e) {
  const imageInput = document.getElementById('imageInput');
  const file = imageInput.files[0];

  if (file && file.size > 5 * 1024 * 1024) {
    e.preventDefault();
    const reader = new FileReader();
    reader.onload = function(event) {
      const img = new Image();
      img.onload = function() {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        let width = img.width;
        let height = img.height;
        let quality = 0.9;
        const maxFileSize = 5 * 1024 * 1024;
        let resizedBlob;

        function processImage() {
            return new Promise(resolve => {
                canvas.width = width;
                canvas.height = height;
                ctx.clearRect(0, 0, width, height);
                ctx.drawImage(img, 0, 0, width, height);
                canvas.toBlob(blob => {
                    resizedBlob = blob;
                    resolve();
                }, 'image/jpeg', quality);
            });
        }

        async function resizeAndSubmit() {
            try {
                while (true) {
                    await processImage();
                    if (resizedBlob.size <= maxFileSize) {
                        break;
                    }
                    quality -= 0.1;
                    if (quality < 0.1) {
                        const scale = Math.sqrt(maxFileSize / resizedBlob.size);
                        width *= scale;
                        height *= scale;
                        quality = 0.9;
                    }
                }
                
                const resizedFile = new File([resizedBlob], file.name, {
                  type: resizedBlob.type,
                  lastModified: Date.now()
                });

                const dataTransfer = new DataTransfer();
                dataTransfer.items.add(resizedFile);
                imageInput.files = dataTransfer.files;

                document.getElementById('uploadForm').submit();
            } catch (error) {
                alert('画像の処理に失敗しました。別の画像をお試しください。');
                console.error('画像処理エラー:', error);
            }
        }
        resizeAndSubmit();
      };
      img.src = event.target.result;
    };
    reader.onerror = function() {
        alert('画像の読み込みに失敗しました。');
    };
    reader.readAsDataURL(file);
  }
});

// レスアンカー機能のスクリプトとスクロール機能
document.querySelectorAll('.entry-id').forEach(button => {
    button.addEventListener('click', event => {
        const entryId = event.target.dataset.id;
        const textarea = document.getElementById('bodyTextarea');
        
        const currentPos = textarea.selectionStart;
        const textToInsert = '>>' + entryId + '\n';
        const currentValue = textarea.value;
        
        textarea.value = currentValue.slice(0, currentPos) + textToInsert + currentValue.slice(currentPos);
        
        const formElement = document.getElementById('uploadForm');
        formElement.scrollIntoView({ behavior: 'smooth' });

        textarea.focus();
        const newCursorPos = currentPos + textToInsert.length;
        textarea.setSelectionRange(newCursorPos, newCursorPos);
    });
});
</script>
</body>
</html>


</details>
完了 🎉

docker compose up を実行すれば、掲示板サービスが起動します。

http://<EC2のパブリックIP>/ にアクセスして動作を確認してください。
