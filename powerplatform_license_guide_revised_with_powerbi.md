# Power Platform（Power Apps / Power BI）ライセンス選択ガイド（修正版／Microsoft公式情報反映）

> **注意（価格表記について）**  
> - **円建て価格**は、日本の Microsoft 公式価格ページに掲載されている場合はそれを採用しています。  
> - 一部のプラン（例：Azure/Fabric 容量、Power BI Embedded 等）は、**Azure課金（従量／予約）**で価格が変動します。  
> - 価格は契約形態・購入チャネル・為替等で変動し得るため、稟議では**見積（CSP/EA 等）**で確定してください。

---

# 1. Power Apps（Power Platform）ライセンス

## 1.1 ライセンス比較（表形式）

| 比較項目 | Case A: Premium（推奨：コア） | Case B: Per app（コスト効率） | Case C: Pay-as-you-go（柔軟：従量） |
|---|---|---|---|
| **参考価格** | **¥2,998 / ユーザー / 月相当（年払い、税別）** | **$5 / ユーザー / アプリ / 月（USD）** | **$10 / アクティブユーザー / アプリ / 月（USD、Azure課金）** |
| **対象ユーザー** | 全社横断で複数アプリを使う層（管理者、保全リーダー、改善推進など） | **特定アプリのみ**を定常的に使う層（現場作業員など） | 低頻度アクセス層・スポット利用（監査、応援、たまに閲覧など） |
| **アプリ数／範囲** | **無制限**（割り当て済みユーザーが Power Apps/Power Pages を無制限に利用） | **特定環境で「1アプリ」または「1ポータル」** | アプリ数の上限はなく、**使った分（ユーザー×アプリ）**課金 |
| **利用シーン** | 複数の業務アプリを横断利用／頻繁な作成・編集／改善活動を主導 | 「この1本の業務アプリを日常的に使う」用途に最適 | 「必要な月だけ」「必要な人だけ」使う用途に向く（未使用月は実質0に近い） |
| **コスト特性** | 予算化しやすいが、対象者が増えると固定費が増える | ユーザー×アプリ数に比例。アプリが少ないほど有利 | 月内の**アクティブユーザー数×アプリ数**に比例。利用の波があると有利 |
| **推奨目安（例）** | **Core 10–30%**（開発・管理・改善推進など） | **Mass 70–90%**（特定アプリの利用者が多数） | **Spot**（低頻度・短期・監査用途のバッファ） |

---

## 1.2 推奨パターンとコスト試算（ハイブリッド構成：例）

### ライセンス戦略（例）
- **コアユーザー（約30%）**：Premium  
- **一般・周辺ユーザー（約70%）**：Per app（特定アプリ1本想定）  
- **予備バッファ**：Pay-as-you-go（スポット利用）

### 月額ライセンス費用試算例（ユーザー数 100 名想定）
> **重要**：Per app / Pay-as-you-go の金額は USD 表記です（地域・契約・為替で変動）。  
> Premium は日本公式価格ページの円建て掲載（年払い月額相当）です。

| 対象ユーザー層 | 単価 | 人数 | 小計（月額） | 備考 |
|---|---:|---:|---:|---|
| **コアユーザー** | ¥2,998 / ユーザー / 月相当（年払い） | 30 | **¥89,940** | 円建て（日本公式価格ページ） |
| **周辺ユーザー** | $5 / ユーザー / **アプリ** / 月 | 70 | **$350 / アプリ** | 1人が使うアプリ数に比例（例：2アプリなら $700） |
| **予備バッファ（従量）** | $10 / **アクティブユーザー** / **アプリ** / 月 | - | （従量） | 月内にアプリを開いたユーザー×アプリで課金 |

**合計目安（式）**  
- `¥89,940 + ($350 × 利用アプリ数) + 従量（$10 × アクティブユーザー数 × アプリ数）`

---

# 2. Power BI ライセンス（追加・ブラッシュアップ）

## 2.1 ライセンス比較（表形式）

| 比較項目 | Case A: Power BI Pro | Case B: Premium Per User（PPU） | Case C: 容量（Fabric F SKU / Power BI Premium P SKU） |
|---|---|---|---|
| **参考価格** | **¥2,098 / ユーザー / 月相当（年払い、税別）** | **¥3,598 / ユーザー / 月相当（年払い、税別）** | **Azure課金（Fabric 容量：従量／予約）** ※価格はリージョン・通貨等で変動 |
| **対象ユーザー** | レポート作成・公開・共有・共同作業を行う一般的な利用者 | Pro に加えて、より高度な Premium 機能を **ユーザー単位**で使うデータ担当者 | 組織として容量を購入し、**配布（閲覧者含む）をスケール**させたい場合 |
| **共有／配布の考え方** | 基本は **Pro 同士**で共有・共同作業 | PPU ワークスペースの共有・共同作業は **PPU 同士が前提**（容量に置く場合は別） | **容量ワークスペース**に置くことで、無料ユーザーを含む同僚への配布が可能（条件あり） |
| **特徴** | Power BI サービスでの公開・共有の標準 | Pro 機能 + ほとんどの Premium 機能（容量を専用化せずに解放） | 容量（Fabric / Premium）を基盤に、Power BI を含むワークロードを支える |
| **注意点（重要）** | - | - | **Power BI Premium（P-SKU）は段階的に廃止**され、更新・移行が必要なケースあり（後述） |

### 補足：無料（Fabric (Free)）について
- 無料ライセンスでも **自分用のレポート作成**は可能。  
- ただし、共有・共同作業・他人ワークスペースへの発行はできません。  
- 一方で、コンテンツが **Premium 容量／Fabric F64+ 等**でホストされている場合、Pro/PPU ユーザーが共有したコンテンツを無料ユーザーが利用できるケースがあります（シナリオ・条件に依存）。  

---

## 2.2 容量（Power BI Premium / Fabric）に関する重要アップデート（要点）
- Microsoft は Microsoft Fabric の一般提供に伴い、**Power BI Premium per capacity（P-SKU）を段階的に廃止**し、Fabric 容量へ移行する方針を示しています。  
- 例：**新規顧客は 2024/7/1 以降 P-SKU を購入できない**、既存顧客（EAなし）は **2025/2/1 まで更新可能**など、契約形態により影響が異なります。  

---

## 2.3 Power BI コスト設計の「よくある型」（表形式）

| パターン | ねらい | 構成の例（概念） | コストの主因 |
|---|---|---|---|
| **全員 Pro** | シンプルに全員が共有・閲覧・編集できる | Pro × 全員 | `Pro単価 × 人数` |
| **作成者 Pro + 閲覧者（無料） + 容量** | 大人数への配布をスケールさせる | 作成者（Pro or PPU） + 容量（Fabric/ Premium） + 閲覧者（無料含む） | `作成者ライセンス + 容量（Azure）` |
| **高度分析者だけ PPU** | 高度機能を必要な人に限定 | 一部ユーザーに PPU、他は Pro/無料（配置次第） | `PPU人数 × 単価 +（必要に応じて容量）` |

---

# 出典（Microsoft公式）
## Power Apps
- Power Apps 価格（日本公式）：  
  https://www.microsoft.com/ja-jp/power-platform/products/power-apps/pricing/
- Per app（概要・制約）：  
  https://learn.microsoft.com/ja-jp/power-platform/admin/about-powerapps-perapp
- Pay-as-you-go（概要）：  
  https://learn.microsoft.com/ja-jp/power-platform/admin/pay-as-you-go-overview
- Pay-as-you-go（メーター）：  
  https://learn.microsoft.com/ja-jp/power-platform/admin/pay-as-you-go-meters
- Power Platform Licensing Guide（JA-JP PDF）：  
  https://www.microsoft.com/content/dam/microsoft/final/en-us/microsoft-brand/documents/Power-Platform-Licensing-Guide-May-2025_JA-JP.pdf

## Power BI
- Power BI 価格（日本公式）：  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing
- ライセンス種類別の機能（Free/Pro/PPU/容量）：  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type
- Power BI Premium per capacity（P-SKU）廃止・移行方針（Power BI Blog）：  
  https://powerbi.microsoft.com/en-us/blog/important-update-coming-to-power-bi-premium-licensing/
- Microsoft Fabric 容量価格（Azure）：  
  https://azure.microsoft.com/ja-jp/pricing/details/microsoft-fabric/
