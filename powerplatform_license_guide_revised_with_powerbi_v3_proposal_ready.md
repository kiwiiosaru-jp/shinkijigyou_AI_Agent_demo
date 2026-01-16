# Power Platform（Power Apps / Power BI）ライセンス選択ガイド（提案書向け：用語・条件を明確化）

更新日：2026-01-16  
（本資料は Microsoft 公式ドキュメントに基づき、提案書に記載できる粒度まで「条件」「用語」「具体例」を明確化したものです）

---

# 0. 前提と用語（略語は使わない）

## 0.1 ライセンスの大分類
- **ユーザーごとのライセンス（個人に割り当てる）**  
  例：Power BI Pro、Power BI Premium Per User、Fabric (Free)（無料）
- **容量ライセンス（組織として購入するコンピューティング容量）**  
  例：Microsoft Fabric 容量（容量レベル F2 / F4 / … / F64 など）、Power BI Premium（容量レベル P1 / P2 / … など）  
  容量ライセンスは「ワークスペース」に割り当てて利用し、コンテンツの格納場所（共有容量か、容量ワークスペースか）により、閲覧に必要なユーザーライセンスが変わります。  
  根拠：Microsoft Learn「Power BI サービスのライセンスの種類別機能」  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type

## 0.2 「共有容量」と「容量ワークスペース」
- **共有容量（既定の共有環境）**  
  容量に割り当てられていないワークスペース。原則として、他人のコンテンツを共有・共同作業・閲覧するには Power BI Pro などの有料ユーザーライセンスが必要です。  
  根拠：Microsoft Learn  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type
- **容量ワークスペース（専用容量に割り当てたワークスペース）**  
  Microsoft Fabric 容量または Power BI Premium（容量ベース）に割り当てたワークスペース。条件を満たせば、無料ユーザーを含む配布（閲覧）が可能になります。  
  根拠：Microsoft Learn / Power BI 価格ページ脚注  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing

---

# 1. Power BI：曖昧表現の「提案書記載用」書き換え

以下は、質問で挙がった曖昧箇所を、提案書に貼り付け可能な文章に置き換えたものです。

---

## 1.1 「Microsoft Azure 課金（Microsoft Fabric 容量：従量課金／予約） ※価格はリージョン・通貨等で変動」

### 提案書記載用（そのまま貼り付け可）
Microsoft Fabric の容量は、Microsoft Azure 上で購入する「組織のコンピューティング容量（容量ライセンス）」である。購入形態は、(1) 利用時間に応じて支払う従量課金、(2) 一定期間のコミットにより割引が適用される予約、の 2 つが提示されている。  
価格は、Microsoft Azure 価格ページ上の表示（リージョン・通貨選択）に基づくが、同ページに「表示価格は見積もりであり、契約形態・購入日・為替レート等により実際の価格は変動し得る」と明記されているため、最終金額は見積書（販売チャネルの見積）で確定する。

### 根拠URL
- Microsoft Fabric 価格（従量課金／予約、スケール、一時停止、購入方法等）  
  https://azure.microsoft.com/ja-jp/pricing/details/microsoft-fabric/

---

## 1.2 「（例：F64+ / P1+ 等の条件）」

### 提案書記載用（そのまま貼り付け可）
Power BI コンテンツを、閲覧者（Fabric (Free) の無料ユーザーライセンスを含む）に追加の有料ユーザーライセンスなしで配布するためには、コンテンツを「容量ワークスペース」に配置し、容量が以下のいずれかの閾値を満たす必要がある。  
- Microsoft Fabric 容量：容量レベル F64 以上  
- Power BI Premium（容量ベース）：容量レベル P1 以上  
上記閾値は Power BI 価格ページの脚注として明記されている。

### 根拠URL
- Power BI 価格（脚注に「P1 以降（および F64 以降）」等の条件が明記）  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing
- Microsoft Learn（無料ユーザー配布に「Premium 容量または Fabric F64 以上」の条件が明記）  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type  
  https://learn.microsoft.com/en-us/power-bi/consumer/end-user-features

---

## 1.3 「共有・配布の可否は『格納場所（共有/容量）』と『ライセンス組み合わせ』に依存」

### 提案書記載用（そのまま貼り付け可）
Power BI で「誰が何を閲覧できるか」は、(1) コンテンツを保存しているワークスペースが共有容量か容量ワークスペースか、(2) 配布側・閲覧側のユーザーライセンスの組み合わせ、の 2 点により決まる。  
- 共有容量に保存されたコンテンツは、原則として Power BI Pro 等の有料ユーザーライセンスを持つ利用者同士で共有・共同作業・閲覧する。  
- 容量ワークスペース（Microsoft Fabric 容量または Power BI Premium 容量ベース）に保存されたコンテンツは、容量が所定の閾値（Microsoft Fabric 容量レベル F64 以上、または Power BI Premium 容量レベル P1 以上）を満たす場合、無料ユーザーライセンスを含む利用者に配布できる。

### 根拠URL
- Microsoft Learn（格納場所とライセンス組み合わせで必要条件が変わる旨を明記）  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type
- Power BI 価格（脚注：P1 以降および F64 以降の条件）  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing

---

## 1.4 「容量（Fabric / Premium） （変動） 1 + 容量コスト Azure価格で別途」

### 提案書記載用（そのまま貼り付け可）
容量ライセンス（Microsoft Fabric 容量または Power BI Premium 容量ベース）は、ユーザー人数ではなく「容量レベル（例：F64）」に対して課金される。したがって費用計上は「容量 1 契約（組織単位）」として別建てで記載する。  
購入形態は従量課金または予約であり、従量課金は需要に応じたスケールアップ／スケールダウンや一時停止が可能である。加えて、ストレージ（OneLake ストレージ）やネットワークなど、容量以外の課金項目があり得るため、運用開始後は Microsoft Azure 側で使用量とコストをモニタリングする。

### 根拠URL
- Microsoft Fabric 価格（容量課金、購入形態、スケール、一時停止、ストレージ等）  
  https://azure.microsoft.com/ja-jp/pricing/details/microsoft-fabric/

---

## 1.5 「注意：PPU ワークスペースで共有すると閲覧側にも PPU が必要になる等、配置・運用設計が重要」

（※略語は使わず、Power BI Premium Per User と記載する）

### 提案書記載用（そのまま貼り付け可）
Power BI Premium Per User のワークスペース（またはアプリ）に配置したコンテンツは、原則としてアクセスする利用者全員に Power BI Premium Per User のユーザーライセンスが必要である。  
たとえば、分析担当者が Power BI Premium Per User を用いて Power BI Premium Per User のワークスペースにレポートを発行した場合、閲覧者が Power BI Pro のみを保有していても当該レポートにはアクセスできず、閲覧者側にも Power BI Premium Per User の付与が必要になる。  
閲覧者が多数でユーザー課金が不利な場合は、コンテンツを容量ワークスペース（Microsoft Fabric 容量レベル F64 以上、または Power BI Premium 容量レベル P1 以上）に配置して配布する設計（容量課金）を検討する。

### 根拠URL
- Microsoft Learn「Power BI Premium Per User のよくある質問」（アクセス要件）  
  https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-premium-per-user-faq
- Microsoft Learn（Power BI Premium Per User のワークスペース共有は同ライセンスが必要等の説明）  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type

---

# 2. 具体例（100名規模：提案書向けの設計例）

## 2.1 例：閲覧者が多いので「作成者のみ Power BI Pro + 容量ワークスペース」に寄せる
- 作成者（レポート作成・発行）：20名 → Power BI Pro を付与  
- 閲覧者：70名 → Fabric (Free)（無料ユーザーライセンス）  
- 予備バッファ（スポット閲覧）：10名 → Fabric (Free)（無料ユーザーライセンス）  
- コンテンツの格納先：容量ワークスペース  
- 容量条件：Microsoft Fabric 容量レベル F64 以上 または Power BI Premium 容量レベル P1 以上

根拠URL
- 作成・発行に Power BI Pro が必要（注意）  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing  
  https://azure.microsoft.com/ja-jp/pricing/details/microsoft-fabric/
- 無料ユーザー配布の条件（F64 以上 / P1 以上 等）  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type  
  https://learn.microsoft.com/en-us/power-bi/consumer/end-user-features

---

# 3. 参考URL（Microsoft公式）
- Power BI サービスのライセンスの種類別機能  
  https://learn.microsoft.com/ja-jp/power-bi/fundamentals/service-features-license-type
- Power BI 価格  
  https://www.microsoft.com/ja-jp/power-platform/products/power-bi/pricing
- Microsoft Fabric 価格（Microsoft Azure）  
  https://azure.microsoft.com/ja-jp/pricing/details/microsoft-fabric/
- Power BI Premium Per User のよくある質問  
  https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-premium-per-user-faq
- 無料ユーザーが閲覧できる条件（Power BI コンシューマー向け）  
  https://learn.microsoft.com/en-us/power-bi/consumer/end-user-features
