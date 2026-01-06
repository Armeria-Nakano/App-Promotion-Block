# クロスプロモーション機能 共通仕様書

## 📋 概要

当社が開発する全Shopifyアプリに「おすすめアプリ」セクションを設置し、相互にアプリを紹介する機能。

### 対象アプリ

| アプリ名 | ID | 状態 |
|---------|-----|------|
| クーポン.JP | `coupon-jp` | リリース済 |
| メールフォーム.JP | `mailform-jp` | リリース済 |
| カレンダー.JP | `calendar-jp` | リリース済 |

---

## ⚖️ Shopify規約の遵守事項

### ✅ 許可される条件

1. **閉じられる（Dismissible）** — 「×」ボタンで非表示にできること
2. **再表示しない** — 閉じた後、同じ内容が再度表示されないこと
3. **必須と暗示しない** — オンボーディングの必須ステップとして表示しないこと

### ❌ 禁止事項

| 場所 | 規約 |
|------|------|
| **テーマApp Block / App Embed** | アプリ宣伝・レビュー依頼は禁止 |
| **App Storeリスティング** | 他アプリへの参照禁止 |
| **虚偽の評価表示** | 実際と異なる星評価の表示禁止 |

### 参考リンク

- [Built for Shopify requirements](https://shopify.dev/docs/apps/launch/built-for-shopify/requirements)
- [App Store requirements](https://shopify.dev/docs/apps/launch/app-requirements-checklist)

---

## 🏗️ アーキテクチャ

### 方式：GitHub Pages + JSON

```
┌─────────────────────────────────────────────────────────────┐
│  GitHub リポジトリ: App-Promotion-Block                   │
│  ├── apps.json    ← アプリ情報を一元管理                     │
│  └── README.md                                              │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ GitHub Pages で公開
                           ▼
        https://armeria-nakano.github.io/App-Promotion-Block/apps.json
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │カレンダー │    │クーポン   │    │メールフォーム│
    │   .JP    │    │   .JP    │    │    .JP    │
    └──────────┘    └──────────┘    └──────────┘
      自分以外を        自分以外を       自分以外を
       表示             表示            表示
```

### 選定理由

| 方式 | メリット | デメリット | 採用 |
|------|----------|------------|------|
| **GitHub Pages + JSON** | 無料、簡単、即時反映 | Git操作が必要 | ✅ |
| Cloudflare Workers | 無料枠大、高速 | セットアップ必要 | - |
| Google スプレッドシート | 非エンジニアでも編集可 | API制限あり | - |
| 専用管理サーバー | 管理画面作れる | オーバーエンジニアリング | - |

---

## 📁 GitHubリポジトリ構成

### リポジトリ名

```
App-Promotion-Block
```

### ファイル構成

```
App-Promotion-Block/
├── apps.json           # メインのアプリ情報
└── README.md
```

**注意**: アイコン画像は自前でホストせず、Shopify CDNを直接参照する（後述）。

---

## 🖼️ アイコン画像の管理

### Shopify CDN URL構造

Shopifyにアップロードしたアプリアイコンは以下の形式でCDNに保存される：

```
https://cdn.shopify.com/s/files/applications/5a90ee4fddb74d1d6e5993c6e398bf3a.png?1758638395
                                            └──────────────────────────────────┘ └────────┘
                                                  アプリ固有のID（固定）           キャッシュバスター
                                                                                  （更新時のみ変わる）
```

### 重要なポイント

| 項目 | 内容 |
|------|------|
| ファイル名（ハッシュ） | アプリ登録時に発行され、**永続的に固定** |
| クエリパラメータ（`?`以降） | キャッシュ無効化用。**削除してOK** |

### apps.json に記載するURL

**クエリパラメータを除去したURLを使用する**

```
✅ https://cdn.shopify.com/s/files/applications/5a90ee4fddb74d1d6e5993c6e398bf3a.png
❌ https://cdn.shopify.com/s/files/applications/5a90ee4fddb74d1d6e5993c6e398bf3a.png?1758638395
```

クエリパラメータなしでも常に最新のアイコンが取得される。

### メリット

- 自前でアイコン画像をホストする必要なし
- Shopify Partner Dashboardでアイコンを更新すれば自動反映
- CDNなので高速配信

---

## 📄 apps.json 構造

実際のデータは `apps.json` を参照。

```json
{
  "apps": [
    { "id": "...", "name": "...", "description": "...", "icon": "...", "url": "...", "enabled": true, "priority": 1 }
  ]
}
```

### フィールド説明

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `id` | string | アプリ識別子（自分自身を除外するため使用） |
| `name` | string | 表示名 |
| `description` | string | アプリの説明文 |
| `icon` | string | アイコン画像URL（Shopify CDN） |
| `url` | string | App StoreのURL |
| `enabled` | boolean | 表示/非表示の制御 |
| `priority` | number | 表示順（小さいほど上） |

---

## 🔧 各アプリ側の実装

### 1. 型定義

```typescript
// app/types/promotions.ts

export interface AppPromotion {
  id: string;
  name: string;
  description: string;
  icon: string;
  url: string;
  enabled: boolean;
  priority: number;
}

export interface PromotionsData {
  apps: AppPromotion[];
}
```

### 2. サーバーサイド処理

```typescript
// app/utils/promotions.server.ts

import type { PromotionsData, AppPromotion } from "~/types/promotions";

const PROMOTIONS_URL = "https://armeria-nakano.github.io/App-Promotion-Block/apps.json";
const CURRENT_APP_ID = "calendar-jp"; // ← 各アプリで変更

// インメモリキャッシュ
let cache: { data: PromotionsData | null; fetchedAt: number } = {
  data: null,
  fetchedAt: 0,
};

export async function getPromotions(): Promise<AppPromotion[]> {
  const now = Date.now();
  const cacheTTL = 60 * 60 * 1000; // 1時間

  // キャッシュが有効ならそれを返す
  if (cache.data && now - cache.fetchedAt < cacheTTL) {
    return filterApps(cache.data);
  }

  try {
    const response = await fetch(PROMOTIONS_URL, {
      headers: { "Accept": "application/json" },
    });
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data: PromotionsData = await response.json();
    cache = { data, fetchedAt: now };
    
    return filterApps(data);
  } catch (error) {
    console.error("Failed to fetch promotions:", error);
    // フェッチ失敗時は古いキャッシュを返すか空配列
    if (cache.data) {
      return filterApps(cache.data);
    }
    return [];
  }
}

function filterApps(data: PromotionsData): AppPromotion[] {
  return data.apps
    .filter(app => app.enabled && app.id !== CURRENT_APP_ID)
    .sort((a, b) => a.priority - b.priority);
}
```

### 3. UIコンポーネント

**レイアウト**: 2列横並び（アプリ2つを左右に配置）

```tsx
// app/components/AppPromotions.tsx

import type { AppPromotion } from "../types/promotions";

interface Props {
  apps: AppPromotion[];
  dismissed: boolean;
  onDismiss: () => void;
}

export function AppPromotions({ apps, dismissed, onDismiss }: Props) {
  if (dismissed || apps.length === 0) {
    return null;
  }

  return (
    <s-section>
      <s-card>
        <div style={{ 
          display: "flex", 
          justifyContent: "space-between", 
          alignItems: "center",
          marginBottom: "12px"
        }}>
          <s-text variant="headingSm">おすすめアプリ</s-text>
          <button 
            onClick={onDismiss}
            style={{ 
              background: "none", 
              border: "none", 
              cursor: "pointer", 
              fontSize: "20px",
              color: "#666",
              padding: "4px 8px"
            }}
            aria-label="閉じる"
          >
            ×
          </button>
        </div>
        
        {/* 2列レイアウト */}
        <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "12px" }}>
          {apps.map(app => (
            <a
              key={app.id}
              href={app.url}
              target="_blank"
              rel="noopener noreferrer"
              style={{ 
                display: "flex", 
                gap: "12px", 
                alignItems: "center",
                textDecoration: "none",
                color: "inherit",
                padding: "12px",
                borderRadius: "8px",
                border: "1px solid #e0e0e0",
                transition: "background-color 0.2s"
              }}
              onMouseOver={(e) => e.currentTarget.style.backgroundColor = "#f9f9f9"}
              onMouseOut={(e) => e.currentTarget.style.backgroundColor = "transparent"}
            >
              <img 
                src={app.icon} 
                alt={app.name} 
                style={{ 
                  width: "48px", 
                  height: "48px", 
                  borderRadius: "8px",
                  flexShrink: 0
                }} 
              />
              <div>
                <s-text variant="bodyMd" fontWeight="semibold">{app.name}</s-text>
                <s-text variant="bodySm" tone="subdued">{app.description}</s-text>
              </div>
            </a>
          ))}
        </div>
      </s-card>
    </s-section>
  );
}
```

### 4. ページでの使用例

```tsx
// app/routes/app._index.tsx（例）

import { json } from "@remix-run/node";
import { useLoaderData, useFetcher } from "@remix-run/react";
import { getPromotions } from "~/utils/promotions.server";
import { AppPromotions } from "~/components/AppPromotions";

export async function loader({ request }) {
  const apps = await getPromotions();
  
  // TODO: DBから dismissed 状態を取得
  const dismissed = false;
  
  return json({ apps, dismissed });
}

export async function action({ request }) {
  const formData = await request.formData();
  const intent = formData.get("intent");
  
  if (intent === "dismiss-promotions") {
    // TODO: DBに dismissed 状態を保存
    return json({ success: true });
  }
  
  return json({ error: "Unknown intent" }, { status: 400 });
}

export default function Index() {
  const { apps, dismissed } = useLoaderData<typeof loader>();
  const fetcher = useFetcher();
  
  const handleDismiss = () => {
    fetcher.submit(
      { intent: "dismiss-promotions" },
      { method: "post" }
    );
  };
  
  return (
    <s-page title="ダッシュボード">
      {/* メインコンテンツ */}
      
      {/* おすすめアプリ（ページ下部） */}
      <AppPromotions
        apps={apps}
        dismissed={dismissed || fetcher.state !== "idle"}
        onDismiss={handleDismiss}
      />
    </s-page>
  );
}
```

### 5. DB保存（dismissed状態）

各アプリのDBスキーマに追加：

```prisma
// prisma/schema.prisma

model Shop {
  id                    String   @id
  // ... 既存フィールド
  
  promotionsDismissed   Boolean  @default(false)
  promotionsDismissedAt DateTime?
}
```

---

## 📍 配置場所

### 推奨

- **設定ページの下部** — メイン機能の邪魔にならない

### 避けるべき

- ホームページ（ダッシュボード）— メイン導線を阻害する
- オンボーディング内（必須と暗示される）
- モーダル・ポップアップ（押し付けがましい）
- 全ページ固定表示（うるさい）

---

## 🔄 運用フロー

### 新アプリ追加時

1. Shopify Partner Dashboardでアプリアイコンを確認
   - アイコンURLの取得方法: Dev Dashboard → アプリ → アイコン画像を右クリック → URLをコピー
   - `?`以降のクエリパラメータは削除する
   - 例: `https://cdn.shopify.com/s/files/applications/xxxxx.png?123456` → `https://cdn.shopify.com/s/files/applications/xxxxx.png`
2. `apps.json` を編集してアプリ情報を追加
3. `git commit && git push`
4. 数分で GitHub Pages に反映
5. 全アプリが次回フェッチ時（最大1時間後）に自動で新アプリを表示

### 一時的に非表示にしたい場合

1. `apps.json` で対象アプリの `"enabled": false` に変更
2. `git commit && git push`
