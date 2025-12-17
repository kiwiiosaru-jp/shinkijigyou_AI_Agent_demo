# To-Be アーキテクチャ図（Mermaid）
以下は、保全日報入力・修理受付のAI活用（Power Apps × Azure AI Search × Azure OpenAI）を想定した To-Be アーキテクチャ図です。

```mermaid
flowchart TB

  %% =========================
  %% Power Platform / UI
  %% =========================
  subgraph PP[Power Platform / UI]
    PA[Power Apps<br/>修理受付・完了<br/>必須: 日時/ライン/設備/現象<br/>分類コード→固定補足質問]
    PAutomate[Power Automate<br/>登録トリガー/通知<br/>同期・索引更新キュー]
    Entra[Entra ID<br/>認証・認可]
    PBI[Power BI<br/>KPI/再発傾向/品質指標]
  end

  %% =========================
  %% Azure App Layer
  %% =========================
  subgraph AZAPP[Azure アプリ層]
    Func[Azure Functions（API集約）<br/>RAG API: 検索→要約→JSON返却<br/>優先ロジック（設備寄り/現象寄り）]
    Monitor[監視/運用<br/>App Insights・ログ<br/>リトライ・品質フラグ]
  end

  %% =========================
  %% Data / Search / AI
  %% =========================
  subgraph DATA[データ / 検索・AI]
    DV[Dataverse（正本データ候補）<br/>T_Ticket（固定列）<br/>T_TicketAnswer（縦持ち）<br/>マスタ（分類/質問セット等）<br/>source_type/needs_review]
    AIS[Azure AI Search（索引）<br/>ハイブリッド: BM25＋ベクトル<br/>synonym map（略語/同義語）<br/>equipmentVector / symptomVector]
    AOAI[Azure OpenAI<br/>類似案件の要約・注意点<br/>（将来）校正/構造化案]
  end

  %% =========================
  %% Legacy ingestion
  %% =========================
  subgraph LEG[レガシー取込（過去PDF等）]
    Store[PDF/文書保管（Blob/SharePoint）<br/>原本凍結]
    DI[Azure AI Document Intelligence<br/>OCR/抽出（JSON）<br/>confidence付与]
    LegacyMove[部分移行<br/>埋められる列だけ投入<br/>欠損はNULL＋品質フラグ]
  end

  %% =========================
  %% Main flows
  %% =========================
  Entra --> PA
  PA -->|RAG呼出| Func
  Func -->|検索| AIS
  Func -->|要約生成| AOAI
  Func -->|類似TopN＋要約返却| PA

  PA -->|登録/更新| DV
  PAutomate -->|登録トリガー| DV
  DV -->|索引更新| AIS
  Func --> Monitor

  DV -->|分析用データ| PBI

  %% Legacy flows
  Store -->|抽出| DI
  DI -->|抽出結果| LegacyMove
  LegacyMove -->|部分投入| DV
  LegacyMove -->|索引化| AIS
```

## 読み方（要点）
- **Power Apps** は薄い入力フォーム（必須入力＋分類コード→固定補足質問）  
- **Azure Functions** が RAG（AI Search検索→OpenAI要約）を集約し、Power AppsへJSONを返す  
- **Dataverse** は新規・レガシーを同一スキーマで保持しつつ、欠損は品質フラグで管理  
- **AI Search** は検索用索引（ハイブリッド＋シノニム）として利用  
- **過去PDF** は Document Intelligence で抽出し、埋められる箇所だけ新スキーマへ部分移行（欠損はNULL）
