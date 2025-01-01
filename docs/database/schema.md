# Pipeサービス データベーススキーマ設計

## 概要

Pipeサービスでは、PostgreSQLをメインデータベースとして使用し、以下のスキーマ構成を採用しています。

## テーブル構成

### users（ユーザー情報）

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

### organizations（組織情報）

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    plan_type VARCHAR(50) DEFAULT 'free',
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### organization_members（組織メンバー）

```sql
CREATE TABLE organization_members (
    organization_id UUID REFERENCES organizations(id),
    user_id UUID REFERENCES users(id),
    role VARCHAR(20) NOT NULL,
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (organization_id, user_id)
);
```

### pipelines（パイプライン定義）

```sql
CREATE TABLE pipelines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    config JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_pipelines_org ON pipelines(organization_id);
```

### pipeline_runs（パイプライン実行履歴）

```sql
CREATE TABLE pipeline_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id UUID REFERENCES pipelines(id),
    status VARCHAR(20) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP WITH TIME ZONE,
    error_message TEXT,
    metrics JSONB,
    created_by UUID REFERENCES users(id)
);

CREATE INDEX idx_pipeline_runs_pipeline ON pipeline_runs(pipeline_id);
CREATE INDEX idx_pipeline_runs_status ON pipeline_runs(status);
```

### storage_files（ストレージファイル情報）

```sql
CREATE TABLE storage_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    path VARCHAR(500) NOT NULL,
    size BIGINT NOT NULL,
    mime_type VARCHAR(100),
    metadata JSONB,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_storage_files_org ON storage_files(organization_id);
CREATE INDEX idx_storage_files_path ON storage_files(path);
```

### api_keys（APIキー）

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name VARCHAR(100) NOT NULL,
    key_hash VARCHAR(255) NOT NULL,
    last_used_at TIMESTAMP WITH TIME ZONE,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_api_keys_org ON api_keys(organization_id);
```

## データ型の説明

### status列で使用される値
- `active`: アクティブ
- `inactive`: 非アクティブ
- `suspended`: 一時停止
- `deleted`: 削除済み

### pipeline_runs.status列で使用される値
- `queued`: 実行待ち
- `running`: 実行中
- `completed`: 完了
- `failed`: 失敗
- `cancelled`: キャンセル

### organization_members.role列で使用される値
- `owner`: 組織オーナー
- `admin`: 管理者
- `member`: 一般メンバー
- `readonly`: 閲覧のみ

## インデックス戦略

主要な検索・参照パターンに基づき、以下のインデックスを作成しています：

1. 主キー（Primary Key）
   - すべてのテーブルでUUIDを使用

2. 外部キー（Foreign Key）
   - 参照整合性の確保
   - 結合クエリの最適化

3. 検索用インデックス
   - ユーザーメールアドレス
   - 組織IDによるフィルタリング
   - パイプラインステータス

## バックアップ戦略

1. **フルバックアップ**
   - 毎日深夜に実行
   - 30日間保持

2. **WALアーカイブ**
   - 継続的なWALアーカイブ
   - Point-in-Time Recovery (PITR)対応

## パーティショニング戦略

### pipeline_runs テーブル
- 月次パーティショニング
- 3ヶ月以上古いデータは自動アーカイブ

```sql
CREATE TABLE pipeline_runs (
    -- 既存の列定義
) PARTITION BY RANGE (started_at);

CREATE TABLE pipeline_runs_y2024m03 PARTITION OF pipeline_runs
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```

## セキュリティ考慮事項

1. **データ暗号化**
   - パスワードはbcryptでハッシュ化
   - APIキーは安全なハッシュ関数で保存

2. **アクセス制御**
   - スキーマレベルでの権限設定
   - アプリケーションユーザーの最小権限原則

3. **監査ログ**
   - 重要な操作は`audit_logs`テーブルに記録
   - データ変更履歴の追跡

## 運用考慮事項

1. **パフォーマンス最適化**
   - 定期的なVACUUM実行
   - 統計情報の更新
   - インデックスメンテナンス

2. **スケーリング戦略**
   - 読み取り負荷の分散（レプリケーション）
   - シャーディングの検討

---

このスキーマ設計は、サービスの成長に応じて継続的に最適化されます。変更履歴は[マイグレーション履歴](./migrations/)で管理されています。


このデータベーススキーマドキュメントには以下の要素が含まれています：
テーブル定義と構造
リレーションシップの説明
インデックス戦略
パーティショニング方針
バックアップ戦略
セキュリティ考慮事項
運用管理のガイドライン
必要に応じて、以下の追加ドキュメントも作成可能です：
データベースマイグレーション手順
パフォーマンスチューニングガイド
バックアップ・リストア手順
監視設定ガイド

