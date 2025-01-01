# Pipe API リファレンス

## 概要

Pipe APIは、RESTful APIとして実装されており、データパイプラインの構築と管理を可能にします。

## API エンドポイント

ベースURL: `https://api.pipe.dev/v1`

## 認証

すべてのAPIリクエストには認証が必要です。認証には以下の方法を使用します：

```bash
Authorization: Bearer <your_api_token>
```

## パイプライン管理 API

### パイプライン一覧の取得

```http
GET /pipelines
```

**パラメータ**
| パラメータ | 型 | 説明 |
|------------|------|-------------|
| page | integer | ページ番号（デフォルト: 1） |
| limit | integer | 1ページあたりの件数（デフォルト: 20） |

**レスポンス例**
```json
{
  "data": [
    {
      "id": "pipe_123",
      "name": "データ変換パイプライン",
      "status": "active",
      "created_at": "2024-03-15T09:00:00Z"
    }
  ],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}
```

### パイプラインの作成

```http
POST /pipelines
```

**リクエストボディ**
```json
{
  "name": "新規パイプライン",
  "description": "データ変換用パイプライン",
  "config": {
    "source": {
      "type": "postgresql",
      "connection": {
        "host": "db.example.com",
        "port": 5432
      }
    },
    "destination": {
      "type": "s3",
      "bucket": "my-bucket"
    }
  }
}
```

**レスポンス例**
```json
{
  "id": "pipe_124",
  "name": "新規パイプライン",
  "status": "created",
  "created_at": "2024-03-15T10:00:00Z"
}
```

### パイプラインの実行

```http
POST /pipelines/{pipeline_id}/run
```

**リクエストボディ**
```json
{
  "parameters": {
    "batch_size": 1000,
    "timeout": 3600
  }
}
```

**レスポンス例**
```json
{
  "run_id": "run_789",
  "status": "running",
  "started_at": "2024-03-15T11:00:00Z"
}
```

## ストレージ API

### ファイルのアップロード

```http
POST /storage/upload
```

**マルチパートフォームデータ**
- `file`: アップロードするファイル
- `path`: 保存先のパス

**レスポンス例**
```json
{
  "file_id": "file_456",
  "path": "/uploads/data.csv",
  "size": 1024,
  "created_at": "2024-03-15T12:00:00Z"
}
```

## 監視 API

### メトリクスの取得

```http
GET /metrics
```

**パラメータ**
| パラメータ | 型 | 説明 |
|------------|------|-------------|
| start_time | string | 開始時刻（ISO 8601形式） |
| end_time | string | 終了時刻（ISO 8601形式） |
| metric_type | string | メトリクスタイプ（cpu, memory, etc） |

**レスポンス例**
```json
{
  "metrics": [
    {
      "timestamp": "2024-03-15T13:00:00Z",
      "type": "cpu",
      "value": 45.2
    }
  ]
}
```

## エラーレスポンス

エラーが発生した場合、以下の形式でレスポンスが返されます：

```json
{
  "error": {
    "code": "invalid_request",
    "message": "必要なパラメータが不足しています",
    "details": {
      "missing_fields": ["name"]
    }
  }
}
```

## エラーコード一覧

| コード | 説明 |
|--------|------|
| invalid_request | リクエストが不正です |
| unauthorized | 認証が必要です |
| forbidden | アクセス権限がありません |
| not_found | リソースが見つかりません |
| rate_limit_exceeded | レート制限を超過しました |

## レート制限

- 認証済みユーザー: 1000 リクエスト/時
- 未認証ユーザー: 100 リクエスト/時

レート制限情報はレスポンスヘッダーに含まれます：

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1523456789
```

## SDKとサンプルコード

各種言語向けのSDKを提供しています：

### Node.js
```typescript
import { PipeClient } from '@pipe/sdk';

const client = new PipeClient('your_api_token');

// パイプラインの作成
const pipeline = await client.pipelines.create({
  name: 'データ変換',
  description: '定期実行パイプライン'
});

// パイプラインの実行
const run = await client.pipelines.run(pipeline.id, {
  parameters: {
    batch_size: 1000
  }
});
```

### Python
```python
from pipe_sdk import PipeClient

client = PipeClient('your_api_token')

# パイプラインの作成
pipeline = client.pipelines.create(
    name='データ変換',
    description='定期実行パイプライン'
)

# パイプラインの実行
run = client.pipelines.run(
    pipeline_id=pipeline.id,
    parameters={'batch_size': 1000}
)
```

## サポート

APIに関する質問やサポートが必要な場合は、以下の方法でお問い合わせください：

- 開発者フォーラム: https://community.pipe.dev
- サポートメール: support@pipe.dev
- APIステータス: https://status.pipe.dev

---

このドキュメントは定期的に更新されます。最新のAPI仕様は[OpenAPI仕様書](https://api.pipe.dev/docs/openapi.json)を参照してください。


このAPIドキュメントには以下の要素が含まれています：
API概要と認証方法
2. 各エンドポイントの詳細な説明
リクエスト/レスポンスの例
エラーハンドリング
レート制限の説明
SDKの使用例
サポート情報
必要に応じて、以下の追加ドキュメントも作成可能です：
1. 特定のユースケース向けのチュートリアル
詳細なSDKドキュメント
APIバージョニングポリシー
4. セキュリティガイドライン

