# Pipeサービス フロントエンド概要

## アーキテクチャ

Pipeサービスのフロントエンドは、以下の技術スタックで構築されています：

### コア技術

- **フレームワーク**: Next.js 14
- **言語**: TypeScript
- **状態管理**: Zustand
- **スタイリング**: Tailwind CSS
- **UIコンポーネント**: shadcn/ui

## プロジェクト構造

```
frontend/
├── app/                    # Nextjs App Router
│   ├── (auth)/            # 認証関連ページ
│   ├── (dashboard)/       # ダッシュボード
│   └── layout.tsx         # ルートレイアウト
├── components/            # 共通コンポーネント
│   ├── ui/               # 基本UIコンポーネント
│   ├── forms/            # フォームコンポーネント
│   └── layouts/          # レイアウトコンポーネント
├── hooks/                # カスタムフック
├── lib/                  # ユーティリティ
├── stores/               # 状態管理
└── types/                # 型定義
```

## 主要コンポーネント

### ダッシュボード

```typescript
// app/(dashboard)/page.tsx
export default function DashboardPage() {
  return (
    <DashboardLayout>
      <PipelineList />
      <MetricsOverview />
      <RecentActivity />
    </DashboardLayout>
  );
}
```

### パイプライン設定

```typescript
// components/pipeline/PipelineConfig.tsx
export function PipelineConfig({ pipelineId }: { pipelineId: string }) {
  const { data, isLoading } = usePipeline(pipelineId);
  
  return (
    <div className="space-y-4">
      <ConfigForm 
        initialData={data}
        onSubmit={handleSubmit}
      />
      <SourceConfig />
      <DestinationConfig />
    </div>
  );
}
```

## 状態管理

### Zustandストア例

```typescript
// stores/pipelineStore.ts
interface PipelineStore {
  pipelines: Pipeline[];
  activePipeline: Pipeline | null;
  setActivePipeline: (pipeline: Pipeline) => void;
  fetchPipelines: () => Promise<void>;
}

export const usePipelineStore = create<PipelineStore>((set) => ({
  pipelines: [],
  activePipeline: null,
  setActivePipeline: (pipeline) => set({ activePipeline: pipeline }),
  fetchPipelines: async () => {
    const data = await api.getPipelines();
    set({ pipelines: data });
  },
}));
```

## APIとの連携

### カスタムフック

```typescript
// hooks/usePipeline.ts
export function usePipeline(id: string) {
  const { data, error } = useSWR<Pipeline>(
    `/api/pipelines/${id}`,
    fetcher
  );

  return {
    pipeline: data,
    isLoading: !error && !data,
    isError: error
  };
}
```

## エラーハンドリング

```typescript
// components/ErrorBoundary.tsx
export class ErrorBoundary extends React.Component<Props, State> {
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  render() {
    if (this.state.hasError) {
      return <ErrorDisplay error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

## パフォーマンス最適化

### 画像最適化

```typescript
// components/ui/OptimizedImage.tsx
export function OptimizedImage({ src, alt }: ImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={800}
      height={600}
      placeholder="blur"
      loading="lazy"
    />
  );
}
```

### コンポーネントメモ化

```typescript
// components/PipelineCard.tsx
export const PipelineCard = memo(function PipelineCard({ 
  pipeline 
}: PipelineCardProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{pipeline.name}</CardTitle>
      </CardHeader>
      <CardContent>
        {/* パイプライン詳細 */}
      </CardContent>
    </Card>
  );
});
```

## アクセシビリティ

- WAI-ARIA準拠のコンポーネント
- キーボードナビゲーション対応
- スクリーンリーダー対応
- カラーコントラスト基準の遵守

```typescript
// components/ui/Button.tsx
export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ children, ...props }, ref) => {
    return (
      <button
        ref={ref}
        {...props}
        className={cn(
          "focus:ring-2 focus:ring-offset-2",
          props.className
        )}
      >
        {children}
      </button>
    );
  }
);
```

## テスト戦略

### ユニットテスト

```typescript
// __tests__/components/PipelineCard.test.tsx
describe('PipelineCard', () => {
  it('正しくパイプライン情報を表示する', () => {
    const pipeline = {
      id: '1',
      name: 'テストパイプライン'
    };
    
    render(<PipelineCard pipeline={pipeline} />);
    expect(screen.getByText('テストパイプライン')).toBeInTheDocument();
  });
});
```

### E2Eテスト

```typescript
// cypress/e2e/pipeline.cy.ts
describe('パイプライン操作', () => {
  it('新規パイプラインを作成できる', () => {
    cy.login();
    cy.visit('/pipelines/new');
    cy.get('[data-testid="name-input"]').type('新規パイプライン');
    cy.get('[data-testid="submit-button"]').click();
    cy.url().should('include', '/pipelines');
  });
});
```

## デプロイメントフロー

1. プルリクエスト作成
2. 自動テスト実行
3. Vercelプレビューデプロイ
4. レビュー承認
5. 本番環境デプロイ

## パフォーマンスモニタリング

- Lighthouse スコア監視
- Core Web Vitals 追跡
- エラー追跡（Sentry）
- ユーザー行動分析

---

このドキュメントは継続的に更新され、新しい機能や改善点が追加されます。
詳細な実装ガイドラインは各コンポーネントのドキュメントを参照してください。
