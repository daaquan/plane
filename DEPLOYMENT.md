# Plane Production Deployment

本番環境向けのPlaneデプロイ構成。インフラサービス (PostgreSQL, Redis, RabbitMQ) はホストマシンのものを使用し、S3ストレージは外部RustFSを使用する。

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Docker Containers (docker-compose.production)  │
│                                                 │
│  ┌─────┐ ┌───────┐ ┌───────┐ ┌──────┐          │
│  │ web │ │ space │ │ admin │ │ live │  Frontend │
│  │:3000│ │:3000  │ │:3000  │ │:3000 │          │
│  └─────┘ └───────┘ └───────┘ └──────┘          │
│                                                 │
│  ┌─────┐ ┌────────┐ ┌──────────┐ ┌──────────┐  │
│  │ api │ │ worker │ │ beat-    │ │ migrator │  │
│  │:8000│ │        │ │ worker  │ │ (初回)   │  │
│  └──┬──┘ └───┬────┘ └────┬────┘ └────┬─────┘  │
│     │        │           │           │  Backend │
└─────┼────────┼───────────┼───────────┼─────────┘
      │        │           │           │
      ▼        ▼           ▼           ▼
┌──────────────────────────────────────────────┐
│  Host Services (192.168.0.239)               │
│  ┌────────────┐ ┌───────┐ ┌──────────┐      │
│  │ PostgreSQL │ │ Redis │ │ RabbitMQ │      │
│  │ :5432      │ │ :6379 │ │ :5672    │      │
│  │ DB: plane  │ │       │ │ vhost:   │      │
│  │ user:      │ │       │ │  plane   │      │
│  │  postgres  │ │       │ │          │      │
│  └────────────┘ └───────┘ └──────────┘      │
└──────────────────────────────────────────────┘
      │
      ▼
┌──────────────────────┐
│  RustFS (S3)         │
│  10.100.0.10:9000    │
│  Bucket: plane       │
└──────────────────────┘
```

## Files

| File | Purpose |
|------|---------|
| `docker-compose.production.yml` | サービス定義 (インフラコンテナなし) |
| `.env.production` | 環境変数 (認証情報含む) |
| `nginx-plane.conf` | Nginx リバースプロキシ設定 (plane.daaquan.com) |

## Services

### Containers (Docker)

| Container | Image | Container Port | Host Port | Purpose |
|-----------|-------|---------------|-----------|---------|
| plane-web | plane-frontend:stable | 3000 | 127.0.0.1:3000 | メインWebアプリ |
| plane-space | plane-space:stable | 3000 | 127.0.0.1:3002 | 公開ワークスペース共有 |
| plane-admin | plane-admin:stable | 3000 | 127.0.0.1:3003 | 管理パネル (God Mode) |
| plane-live | plane-live:stable | 3000 | 127.0.0.1:3004 | リアルタイムコラボレーション |
| plane-api | plane-backend:stable | 8000 | 127.0.0.1:8080 | Django API サーバー |
| plane-bgworker | plane-backend:stable | - | - | Celery バックグラウンドワーカー |
| plane-beatworker | plane-backend:stable | - | - | Celery Beat スケジューラ |
| plane-migrator | plane-backend:stable | - | - | DBマイグレーション (起動時のみ) |

### Host Infrastructure (192.168.0.239)

| Service | Port | Details |
|---------|------|---------|
| PostgreSQL | 5432 | DB: `plane`, User: `postgres` |
| Redis | 6379 | キャッシュ、セッション |
| RabbitMQ | 5672 | メッセージキュー (vhost: `plane`, user: `plane`) |

### External Storage

| Service | Endpoint | Details |
|---------|----------|---------|
| RustFS (S3互換) | `http://10.100.0.10:9000` | Bucket: `plane` |

## Prerequisites

### 1. PostgreSQL - `plane` データベース

```sql
psql -h 192.168.0.239 -U postgres -c "CREATE DATABASE plane;"
```

### 2. RabbitMQ - vhost/ユーザー

```bash
sudo rabbitmqctl add_vhost plane
sudo rabbitmqctl add_user plane plane
sudo rabbitmqctl set_permissions -p plane plane ".*" ".*" ".*"
```

### 3. RustFS - `plane` バケット

```bash
mc alias set rustfs http://10.100.0.10:9000 <ACCESS_KEY> <SECRET_KEY>
mc mb rustfs/plane --ignore-existing
```

## Usage

### Start

```bash
cd /opt/plane
docker compose -f docker-compose.production.yml --env-file .env.production up -d
```

### Stop

```bash
docker compose -f docker-compose.production.yml --env-file .env.production down
```

### Restart

```bash
docker compose -f docker-compose.production.yml --env-file .env.production restart
```

### View Logs

```bash
# All services
docker compose -f docker-compose.production.yml --env-file .env.production logs -f

# Specific service
docker compose -f docker-compose.production.yml --env-file .env.production logs -f api
docker compose -f docker-compose.production.yml --env-file .env.production logs -f worker
```

### Upgrade

```bash
cd /opt/plane

# Pull latest images
docker compose -f docker-compose.production.yml --env-file .env.production pull

# Recreate containers
docker compose -f docker-compose.production.yml --env-file .env.production up -d
```

特定バージョンにする場合は `.env.production` で `APP_RELEASE` を変更:

```
APP_RELEASE=v0.24-dev
```

### Check Status

```bash
docker compose -f docker-compose.production.yml --env-file .env.production ps
```

## Reverse Proxy (Nginx)

proxyコンテナは含まれていないため、ホストのNginxでリバースプロキシを行う。

**URL**: `https://plane.daaquan.com`

### ルーティング

| Path | Host Port | Upstream | Notes |
|------|-----------|----------|-------|
| `/` | 3000 | plane-web | メインUI |
| `/api/`, `/auth/`, `/static/` | 8080 | plane-api | Backend API |
| `/god-mode/` | 3003 | plane-admin | 管理パネル |
| `/spaces/` | 3002 | plane-space | 公開スペース |
| `/live/` | 3004 | plane-live | WebSocket対応 |
| `/plane` | - | 10.100.0.10:9000 | RustFS アップロード (末尾スラッシュなし必須) |

### セットアップ

```bash
# 設定ファイルを配置
sudo cp /opt/plane/nginx-plane.conf /etc/nginx/sites-available/plane.daaquan.com.conf
sudo ln -s /etc/nginx/sites-available/plane.daaquan.com.conf /etc/nginx/sites-enabled/

# SSL証明書を取得 (初回のみ)
sudo certbot --nginx -d plane.daaquan.com

# 設定テスト & 反映
sudo nginx -t && sudo systemctl reload nginx
```

設定ファイルの詳細は `nginx-plane.conf` を参照。

> **注意**: RustFS (S3) の location は `location /plane`（末尾スラッシュなし）にすること。
> `USE_MINIO=1` 時、presigned POST URL は `/plane`（スラッシュなし）になるため、
> `location /plane/` だとマッチせずアップロードが失敗する。

## Troubleshooting

### migrator が長時間実行される

初回起動時は100以上のマイグレーションが実行されるため正常。`docker logs plane-migrator` で進捗確認。

### plane-space が unhealthy

公式イメージにcurlが含まれていないためヘルスチェックが失敗する既知の問題。サービス自体は正常に動作する。

### API が "Waiting for database migrations" のまま

migrator の完了を待っている状態。`docker logs plane-migrator` でマイグレーション状況を確認。

### プロフィール画像・カバー画像のアップロード失敗

"Failed to upload cover image" / "Error!" が表示される場合:

1. **nginx location を確認** — `location /plane` (末尾スラッシュなし) であること。`/plane/` だと presigned POST がマッチしない。
2. **RustFS に到達できるか確認** — `curl -s -o /dev/null -w '%{http_code}' http://10.100.0.10:9000/minio/health/live`
3. **API ログ確認** — `POST /api/assets/v2/workspaces/...` が 200 を返していれば、API 側は正常。問題はブラウザ → RustFS の経路。

### RabbitMQ 接続エラー

```bash
# vhost/ユーザー確認
sudo rabbitmqctl list_vhosts
sudo rabbitmqctl list_users
sudo rabbitmqctl list_permissions -p plane
```
