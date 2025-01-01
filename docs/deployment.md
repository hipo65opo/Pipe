# Pipeサービス デプロイメントガイド

## 概要

Pipeサービスは、マイクロサービスアーキテクチャに基づいて構築されており、以下のコンポーネントごとに個別にデプロイされます。

## デプロイメント環境

### 環境構成

| 環境 | 用途 | URLパターン |
|-----|------|------------|
| 開発(dev) | 開発者テスト | `dev.pipe.dev` |
| ステージング(stg) | QAテスト | `stg.pipe.dev` |
| 本番(prod) | 本番サービス | `pipe.dev` |

## インフラストラクチャ

### クラウドプロバイダー
- **主要インフラ**: AWS
- **CDN**: Cloudflare
- **モニタリング**: Datadog

### Kubernetes構成

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipe-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pipe-api
  template:
    spec:
      containers:
      - name: api
        image: pipe/api:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
```

## デプロイメントプロセス

### 1. フロントエンド

```bash
# フロントエンドのビルドとデプロイ
npm run build
vercel deploy --prod
```

### 2. バックエンドAPI

```bash
# Dockerイメージのビルド
docker build -t pipe/api:${VERSION} .

# イメージのプッシュ
docker push pipe/api:${VERSION}

# Kubernetesデプロイメント更新
kubectl apply -f k8s/production/
```

### 3. データベースマイグレーション

```bash
# マイグレーション実行
npm run migrate:prod
```

## CI/CD パイプライン

### GitHub Actions ワークフロー

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: npm test

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          ./scripts/deploy.sh
```

## ロールバック手順

### 1. アプリケーションロールバック

```bash
# 前バージョンへのロールバック
kubectl rollout undo deployment/pipe-api

# 特定バージョンへのロールバック
kubectl rollout undo deployment/pipe-api --to-revision=2
```

### 2. データベースロールバック

```bash
# マイグレーションのロールバック
npm run migrate:rollback

# 特定バージョンへのロールバック
npm run migrate:rollback --to 20240301
```

## モニタリングとアラート

### 1. メトリクス監視

- CPU使用率
- メモリ使用率
- レスポンスタイム
- エラーレート

### 2. アラート設定

```yaml
# alertmanager/rules.yaml
groups:
- name: pipe-alerts
  rules:
  - alert: HighErrorRate
    expr: error_rate > 0.01
    for: 5m
    labels:
      severity: critical
```

## セキュリティ対策

### 1. SSL/TLS設定

```nginx
# nginx/conf.d/ssl.conf
server {
    listen 443 ssl http2;
    server_name pipe.dev;
    
    ssl_certificate /etc/ssl/pipe.crt;
    ssl_certificate_key /etc/ssl/pipe.key;
    ssl_protocols TLSv1.2 TLSv1.3;
}
```

### 2. ファイアウォール設定

```bash
# セキュリティグループ設定
aws ec2 authorize-security-group-ingress \
    --group-id sg-123456 \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0
```

## 障害復旧手順

### 1. システム障害時

1. アラートの確認
2. ログ分析
3. 影響範囲の特定
4. 必要に応じてロールバック
5. 原因究明と再発防止

### 2. データ復旧

```bash
# バックアップからのリストア
aws s3 cp s3://pipe-backup/latest.dump /tmp/
pg_restore -d pipe_db /tmp/latest.dump
```

## 運用チェックリスト

### デプロイ前

- [ ] テストの完了確認
- [ ] セキュリティスキャン実施
- [ ] パフォーマンステスト実施
- [ ] ドキュメント更新

### デプロイ後

- [ ] ヘルスチェック確認
- [ ] メトリクス監視
- [ ] ログ確認
- [ ] ユーザー影響の確認

## トラブルシューティング

### よくある問題と解決策

1. **デプロイメント失敗**
   ```bash
   # ログの確認
   kubectl logs deployment/pipe-api
   # ステータス確認
   kubectl describe deployment pipe-api
   ```

2. **パフォーマンス低下**
   ```bash
   # メトリクス確認
   kubectl top pods
   # トレース分析
   datadog-trace-agent status
   ```

---

このドキュメントは定期的に更新され、新しいデプロイメント要件や手順が追加されます。
最新の運用手順については、運用チームに確認してください。


このデプロイメントドキュメントには以下の要素が含まれています：
デプロイメント環境の説明
インフラストラクチャ構成
デプロイメントプロセス
CI/CDパイプライン設定
ロールバック手順
モニタリングとアラート設定
セキュリティ対策
障害復旧手順
運用チェックリスト
必要に応じて、以下の追加ドキュメントも作成可能です：
1. 環境別の詳細設定
スケーリング戦略
コスト最適化ガイド
災害復旧計画

