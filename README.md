1. AWS EC2 インスタンスの準備
まず、アプリケーションを動かすためのサーバー（EC2インスタンス）をAWS上に用意します。

インスタンスの起動: AWSマネジメントコンソールでEC2ダッシュボードに移動し、「インスタンスを起動」をクリックします。

AMIの選択: Amazon Linux 2 または Ubuntu Server を選択します。Amazon Linux 2はAWSのサービスとの連携がスムーズなためおすすめです。

インスタンスタイプの選択: 開発用には費用が抑えられる t2.micro または t3.micro で十分です。

セキュリティグループの設定: サーバーへのアクセスを制御するセキュリティグループを作成します。

SSH (ポート22) を自分のIPアドレスからのみ許可し、サーバーに接続できるようにします。

HTTP (ポート80) と HTTPS (ポート443) を0.0.0.0/0（すべてのIPアドレス）から許可し、ウェブサイトを公開します。

起動と接続: キーペアを作成してインスタンスを起動し、SSHを使ってサーバーに接続します。

2. Docker 環境のセットアップ
EC2インスタンスに接続したら、DockerとDocker Composeをインストールします。

パッケージの更新: システムのパッケージを最新の状態にします。

Bash
sudo yum update -y
Dockerのインストール: Dockerエンジンをインストールします。

Bash

sudo yum install docker -y
Dockerサービスの開始: Dockerデーモンを起動し、起動時に自動で立ち上がるように設定します。

Bash

sudo systemctl start docker
sudo systemctl enable docker
ユーザーをdockerグループに追加: sudoなしでDockerコマンドを実行できるようにします。

Bash

sudo usermod -a -G docker ec2-user
この変更を反映させるために、一度ログアウトして再ログインしてください。

Docker Composeのインストール: Docker Composeをダウンロードして実行権限を付与します。

Bash

sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
3. Docker コンテナの定義と設定
次に、アプリケーションを構成するコンテナ（PHP、Nginx、MySQL）を定義します。

プロジェクトディレクトリの作成: プロジェクト用のディレクトリを作成し、移動します。

Bash

mkdir bulletin-board
cd bulletin-board
docker-compose.ymlの作成: このファイルに、各サービスの構成を記述します。

Bash

vim docker-compose.yml
YAML

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
php: PHPの実行環境を定義します。

nginx: ウェブサーバーを定義し、ポート80でリクエストを受け付けます。

mysql: データベースを定義します。volumesでデータを永続化します。

4. アプリケーションコードと設定ファイルの配置
コンテナが参照する実際のファイルを配置します。

PHPのDockerfileを作成: プロジェクトルートに以下のファイルを作成します。

Bash

vim Dockerfile
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
php:8.0-fpm をベースに、MySQLと画像処理に必要なライブラリをインストールします。

Nginxの設定ファイルを作成: nginx ディレクトリを作成し、default.confを配置します。

Bash

mkdir nginx
vim nginx/default.conf
Nginx

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
PHPコードの配置: 以前にご提供いただいたkadai.phpファイルをpublicディレクトリを作成して配置します。

Bash

mkdir public
vim public/kadai.php
ここに、これまでのやり取りで完成させた最新のkadai.phpコードをすべて貼り付けてください。

5. コンテナの起動とデータベース設定
すべてのファイルが揃ったら、コンテナを起動し、データベースを準備します。

コンテナのビルド・起動: プロジェクトディレクトリで以下のコマンドを実行します。

Bash

docker-compose up -d --build
データベースに接続: MySQLコンテナに接続し、テーブルを作成します。

Bash

docker-compose exec mysql mysql -u root -p
パスワードは docker-compose.yml で設定したroot_passwordです。
MySQLプロンプトで以下のSQLを実行します。

SQL

USE example_db;

CREATE TABLE bbs_entries (
    id INT AUTO_INCREMENT PRIMARY KEY,
    body TEXT,
    image_filename VARCHAR(255),
    delete_password VARCHAR(255),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE deleted_entries (
    id INT PRIMARY KEY,
    deleted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
6. 動作確認
ブラウザでEC2インスタンスのパブリックIPアドレスにアクセスしてください。掲示板が表示され、投稿、画像アップロード、そして削除機能が意図通りに動作することを確認できます。
