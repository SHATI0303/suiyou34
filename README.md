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
