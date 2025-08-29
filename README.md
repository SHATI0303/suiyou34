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
<?php
// データベース接続
$dbh = new PDO('mysql:host=mysql;dbname=example_db', 'root', '');

// 以下、投稿処理・削除処理・ページネーションなど
// （フルコードは省略せずにここへ記述）

</details>
完了 🎉

docker-compose up -d を実行すれば、掲示板サービスが起動します。

http://<EC2のパブリックIP>/ にアクセスして動作を確認してください。
