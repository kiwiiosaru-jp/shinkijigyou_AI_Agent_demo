# インデックス定義書（Azure版）テンプレート（RAG / 検索基盤）

> 目的：RAGで参照する「検索インデックス（ベクトル）」を、**どのデータを・どう前処理して・どんなメタデータで・どう更新/削除し・どう監視/セキュアに運用するか**を、インデックス（= コーパス）単位で合意するための雛形です。  
> 注：本テンプレートの「インデックス」は、**Azure AI Search の Index** または **ベクトルDB（例：Azure Database for PostgreSQL + pgvector）の論理コレクション（テーブル/パーティション）** のどちらでも運用できるように書いています。

---

## 0. 基本情報
- 文書名：インデックス定義書（`{index_name}`）
- 対象システム／プロジェクト名：
- 作成日／版数：vX.Y（改定履歴は末尾）
- 承認者：業務／情シス／セキュリティ／開発
- 関連資料：要件定義、運用設計、権限設計、テスト計画、データ仕様、DLP/分類ポリシー 等

---

## 1. インデックス概要
- インデックス名：`{index_name}`
- 用途（ユーザー/ユースケース）：
- 対象範囲（業務領域・部門・期間）：
- 期待する回答範囲（含める/含めない）：
- データ鮮度要求：例）更新から **N分以内**に検索へ反映／日次で十分 等
- 想定規模：
  - 文書数：
  - 総ページ数／総容量：
  - 生成チャンク数見込み：
- SLA/SLO（任意）：
  - 取り込み完了SLA：
  - 検索応答SLO（p95など）：

---

## 2. データソース定義（入力）
- ソース種別：SharePoint / OneDrive / ファイルサーバ / Blob Storage / 手動アップロード 等
- 対象ファイル形式：PDF / Word / Excel / Text 等
- 格納先（例）：
  - Blob Storage：`https://{storage}.blob.core.windows.net/{container}/{path}`
- 取り込み単位：
  - 親ドキュメントID（例）：`original_doc_id`（BlobのURI、SharePointのitemId等）
  - 子チャンクID（例）：`chunk_id`（`{original_doc_id}#{chunk_index}`）
- 更新検知方法（選択）：
  - [ ] イベント駆動（Event Grid / Graph Webhook / SharePoint変更通知 等）
  - [ ] キュー駆動（Service Bus / Storage Queue）
  - [ ] 定期バッチ（Logic Apps / Durable Functions / Data Factory）
  - [ ] 手動（管理UI / 管理者スクリプト）
- 除外条件：拡張子、容量、機密区分、パス、言語、重複 等

---

## 3. 前処理・抽出・正規化
- 文字抽出：
  - [ ] Azure AI Document Intelligence（旧 Form Recognizer）  
  - [ ] その他：{tool}
- OCR方針：必要/不要、対象（スキャンPDF等）、精度要件
- テキスト正規化：改行、ヘッダフッタ除去、表の扱い、文字化け対策、言語判定 等
- 分割（チャンキング）方針：
  - 分割単位：段落／見出し／ページ等
  - チャンクサイズ：`{n_tokens or n_chars}`
  - オーバーラップ：`{overlap}`
  - 参照用：`section_title` / `page_no` / `bbox`（任意）
- 品質チェック：
  - 空チャンク除外
  - 極端に短い/長いチャンク
  - OCR失敗率の閾値
  - サンプリングレビューの手順

---

## 4. メタデータ設計（検索・権限・運用の要）
> **RAGではメタデータが“権限制御”と“絞り込み精度”の生命線**です。インデックスごとに必ず定義します。

### 4.1 メタデータ一覧（例）
| key | 型 | 必須 | 用途 | 例 |
|---|---:|:---:|---|---|
| `tenant_id` | string | ✅ | テナント/部門分離（強制フィルタ） | `SG-Sales` |
| `original_doc_id` | string | ✅ | 親ドキュメントID | `.../manual.pdf` |
| `chunk_id` | string | ✅ | チャンク固有ID（冪等性キー） | `...#12` |
| `file_name` | string | ✅ | 表示/運用 | `manual.pdf` |
| `chunk_index` | int | ✅ | 並び/再構成 | `12` |
| `source_system` | string |  | 来歴 | `SharePoint` |
| `updated_at` | datetime |  | 期間絞り込み | `2026-01-01T...Z` |
| `confidentiality` | string |  | 機密区分 | `internal` |
| `acl` | string/array |  | 文書単位ACL（必要時） | `group:A,group:B` |

### 4.2 アクセス制御のルール（必須）
- 認証：Entra ID（OAuth2/OIDC）
- API保護：Azure API Management もしくは App Gateway/Front Door + Functions/App Service 認証
- テナント解決：
  - トークンクレーム（例：groups / roles / custom claim）から `tenant_id` を確定
- 検索時の強制フィルタ（必須）：
  - `tenant_id = resolved_tenant_id` を **バックエンドで自動強制**
  - UIからの任意フィルタは “追加条件” として扱う（tenantフィルタより弱くしない）

---

## 5. 埋め込み（Embedding）設計
- Embedding提供：Azure OpenAI（推奨） / OSS / 他API
- モデル：`{embedding_model}`
- 次元数：`{dim}`
- 生成ポリシー：
  - [ ] 新規/更新時に再計算
  - [ ] 本文ハッシュ差分でスキップ（任意）
- 再計算条件：モデル変更、チャンキング変更、前処理変更、メタデータ設計変更 等

---

## 6. ベクトル格納・インデックス方式（選択）
### 6.A Azure AI Search（推奨：ハイブリッド検索/運用容易）
- Index名：`{search_index_name}`
- フィールド設計：
  - `content`（検索対象本文）
  - `vector`（ベクトル）
  - `tenant_id`（filterable）
  - その他メタデータ（filterable/sortable/facetable の指定）
- 検索設定：
  - topK、スコア閾値、BM25 + vector の併用（ハイブリッド）
  - 必要に応じて Semantic Ranker / Reranker を利用
- 更新方式：documents upload / merge / delete
- 監視：クエリ数、スロットル、インデックス更新失敗、遅延

### 6.B Azure Database for PostgreSQL + pgvector（互換性重視）
- DB：`{pg_server}/{db}/{schema}`
- テーブル：`{table_name}`（例：chunks）
- インデックス方式：HNSW / IVFFlat 等（採用方式・パラメータを明記）
- パーティション：`tenant_id` 等でのパーティション可否、運用方針
- 検索設定：topK、閾値、再ランキング有無

---

## 7. 更新・削除・再インデックス方針（重要）
- 更新トリガ（選択）：
  - [ ] 同期（アップロード完了まで待つ）
  - [ ] 非同期（推奨：キュー/ワークフローで処理、UIは完了後に反映）
  - [ ] 定期（夜間/日次）
- 更新方式：
  - [ ] 差分（増分追加／更新）
  - [ ] 全量再構築（ブルーグリーン：新インデックス構築→切替→旧破棄）
- 削除SLA：例）削除依頼から **N日以内**に検索対象から除外
- 削除方法：
  - [ ] ソース削除（Blob/SharePoint）→ 追随削除（Index/Delete）
  - [ ] 先に検索側を無効化（tombstone）→ 後で物理削除
- 失敗時運用：
  - 再試行回数／DLQ（Dead Letter）／手動リカバリ手順
  - 失敗通知先（Teams/メール/運用監視）

---

## 8. 監視・運用・品質
- 監視（メトリクス例）：
  - 取り込み件数（文書/チャンク）
  - インデックス更新遅延（滞留時間）
  - 失敗件数（抽出/OCR/Embedding/Index更新）
  - 検索エラー、スロットル、p95応答
- ログ：
  - `tenant_id`, `user_id`, `index_name`, `model`, `input_tokens` 等を必ず記録
  - Log Analytics（KQL）でテナント別・インデックス別の分析ができること
- 品質評価：
  - 代表クエリセット
  - 参照ソースの妥当性
  - 期間ごとの精度ドリフト点検

---

## 9. セキュリティ設計（Azure）
- ネットワーク：
  - [ ] Private Link / Private Endpoint（Storage, OpenAI, Search, DB 等）
  - [ ] VNet統合（Functions/App Service）
  - [ ] ExpressRoute / Site-to-Site VPN（オンプレ接続）
- 認証・認可：
  - Entra ID（OAuth2/OIDC）
  - Managed Identity（取り込み/検索/DBアクセスの基本）
  - RBAC：最小権限（Reader/Contributorのスコープ明確化）
- 暗号化：
  - 保存時：Storage/DB/Search の暗号化（CMK利用可否）
  - 転送時：TLS
- 秘密情報：
  - Azure Key Vault（接続文字列/鍵/証明書）
- 脅威検知・ポスチャ：
  - Microsoft Defender for Cloud、Defender for Storage/SQL 等（有効化方針）
- 監査：
  - Activity Log、Diagnostic Settings、Log Analytics への集約
- コンテンツ安全性（任意）：
  - PII検知/マスキング、DLP、コンテンツフィルタ（Azure OpenAI Content Filter等）

---

## 10. クラウド実装（Azure）— 主要コンポーネント一覧（記入欄）
- フロント：`{Front Door / App Gateway / Static Web Apps}`
- API：`{API Management}`
- アプリ／ゲートウェイ：`{Azure Functions / App Service}`
- ワークフロー：`{Durable Functions / Logic Apps / Data Factory}`
- キュー：`{Service Bus / Storage Queue}`
- ストレージ：`{Blob Storage}`
- OCR：`{Document Intelligence}`
- 埋め込み/LLM：`{Azure OpenAI}`
- 検索：`{Azure AI Search or PostgreSQL+pgvector}`
- 監視：`{Azure Monitor / App Insights / Log Analytics}`
- 秘密：`{Key Vault}`

---

## 11. 改定履歴
| 日付 | 版 | 変更内容 | 変更者 |
|---|---|---|---|

---

# 付録：AWS→Azure 対応表（参考）
| AWS（提案書） | Azureの代表的な置き換え候補 |
|---|---|
| S3 | Blob Storage |
| API Gateway | API Management |
| Lambda | Azure Functions |
| Step Functions / EventBridge Scheduler | Durable Functions / Logic Apps / Data Factory（+必要に応じてService Bus） |
| Textract | Azure AI Document Intelligence |
| Bedrock（LLM/Embedding） | Azure OpenAI |
| Bedrock Knowledge Base | Azure AI Search（データソース）＋RAG Orchestrator（Functions/App） |
| Aurora PostgreSQL + pgvector | Azure Database for PostgreSQL + pgvector |
| CloudWatch / CloudTrail / Config | Azure Monitor / Activity Log / Policy（＋Log Analytics） |
| WAF / Shield | App Gateway WAF / Front Door WAF（＋DDoS Protection） |
| KMS / Secrets Manager | Key Vault（＋CMK） |
| GuardDuty / Security Hub | Microsoft Defender for Cloud |




---

# インデックス定義書（Azure版）テンプレート＋記入例
> 目的：RAGで参照する「検索インデックス（ベクトル）」を、**どのデータを・どう前処理して・どんなメタデータで・どう更新/削除し・どう監視/セキュアに運用するか**を、インデックス（= コーパス）単位で合意するための雛形です。  
> 注：「インデックス」は **Azure AI SearchのIndex** または **Azure Database for PostgreSQL + pgvector の論理コレクション（テーブル/パーティション）** どちらにも対応できるように書いています。

---

## （A）テンプレート（空欄）

### 0. 基本情報
- 文書名：インデックス定義書（`{index_name}`）
- 対象システム／プロジェクト名：
- 作成日／版数：vX.Y（改定履歴は末尾）
- 承認者：業務／情シス／セキュリティ／開発
- 関連資料：要件定義、運用設計、権限設計、テスト計画、データ仕様、DLP/分類ポリシー 等

### 1. インデックス概要
- インデックス名：`{index_name}`
- 用途（ユーザー/ユースケース）：
- 対象範囲（業務領域・部門・期間）：
- 期待する回答範囲（含める/含めない）：
- データ鮮度要求：例）更新から **N分以内**に検索へ反映／日次で十分 等
- 想定規模：
  - 文書数：
  - 総ページ数／総容量：
  - 生成チャンク数見込み：
- SLA/SLO（任意）：
  - 取り込み完了SLA：
  - 検索応答SLO（p95など）：

### 2. データソース定義（入力）
- ソース種別：SharePoint / OneDrive / ファイルサーバ / Blob Storage / 手動アップロード 等
- 対象ファイル形式：PDF / Word / Excel / Text 等
- 格納先（例）：`https://{storage}.blob.core.windows.net/{container}/{path}`
- 取り込み単位：
  - 親ドキュメントID（例）：`original_doc_id`（BlobのURI、SharePointのitemId等）
  - 子チャンクID（例）：`chunk_id`（`{original_doc_id}#{chunk_index}`）
- 更新検知方法（選択）：
  - [ ] イベント駆動（Event Grid / Graph Webhook / SharePoint変更通知 等）
  - [ ] キュー駆動（Service Bus / Storage Queue）
  - [ ] 定期バッチ（Logic Apps / Durable Functions / Data Factory）
  - [ ] 手動（管理UI / 管理者スクリプト）
- 除外条件：拡張子、容量、機密区分、パス、言語、重複 等

### 3. 前処理・抽出・正規化
- 文字抽出：
  - [ ] Azure AI Document Intelligence（旧 Form Recognizer）
  - [ ] その他：{tool}
- OCR方針：必要/不要、対象（スキャンPDF等）、精度要件
- テキスト正規化：改行、ヘッダフッタ除去、表の扱い、文字化け対策、言語判定 等
- 分割（チャンキング）方針：
  - 分割単位：段落／見出し／ページ等
  - チャンクサイズ：`{n_tokens or n_chars}`
  - オーバーラップ：`{overlap}`
  - 参照用：`section_title` / `page_no` / `bbox`（任意）
- 品質チェック：
  - 空チャンク除外
  - 極端に短い/長いチャンク
  - OCR失敗率の閾値
  - サンプリングレビューの手順

### 4. メタデータ設計（検索・権限・運用の要）
> **RAGではメタデータが“権限制御”と“絞り込み精度”の生命線**です。インデックスごとに必ず定義します。

#### 4.1 メタデータ一覧（例）
| key | 型 | 必須 | 用途 | 例 |
|---|---:|:---:|---|---|
| `tenant_id` | string | ✅ | テナント/部門分離（強制フィルタ） | `SG-Sales` |
| `original_doc_id` | string | ✅ | 親ドキュメントID | `.../manual.pdf` |
| `chunk_id` | string | ✅ | チャンク固有ID（冪等性キー） | `...#12` |
| `file_name` | string | ✅ | 表示/運用 | `manual.pdf` |
| `chunk_index` | int | ✅ | 並び/再構成 | `12` |
| `source_system` | string |  | 来歴 | `SharePoint` |
| `updated_at` | datetime |  | 期間絞り込み | `2026-01-01T...Z` |
| `confidentiality` | string |  | 機密区分 | `internal` |
| `acl` | string/array |  | 文書単位ACL（必要時） | `group:A,group:B` |

#### 4.2 アクセス制御のルール（必須）
- 認証：Entra ID（OAuth2/OIDC）
- API保護：Azure API Management もしくは App Gateway/Front Door + Functions/App Service 認証
- テナント解決：
  - トークンクレーム（例：groups / roles / custom claim）から `tenant_id` を確定
- 検索時の強制フィルタ（必須）：
  - `tenant_id = resolved_tenant_id` を **バックエンドで自動強制**
  - UIからの任意フィルタは “追加条件” として扱う（tenantフィルタより弱くしない）

### 5. 埋め込み（Embedding）設計
- Embedding提供：Azure OpenAI（推奨） / OSS / 他API
- モデル：`{embedding_model}`
- 次元数：`{dim}`
- 生成ポリシー：
  - [ ] 新規/更新時に再計算
  - [ ] 本文ハッシュ差分でスキップ（任意）
- 再計算条件：モデル変更、チャンキング変更、前処理変更、メタデータ設計変更 等

### 6. ベクトル格納・インデックス方式（選択）
#### 6.A Azure AI Search（推奨：ハイブリッド検索/運用容易）
- Index名：`{search_index_name}`
- フィールド設計：
  - `content`（検索対象本文）
  - `vector`（ベクトル）
  - `tenant_id`（filterable）
  - その他メタデータ（filterable/sortable/facetable の指定）
- 検索設定：topK、スコア閾値、BM25 + vector の併用（ハイブリッド）、Semantic Ranker等（任意）
- 更新方式：documents upload / merge / delete
- 監視：クエリ数、スロットル、インデックス更新失敗、遅延

#### 6.B Azure Database for PostgreSQL + pgvector（互換性重視）
- DB：`{pg_server}/{db}/{schema}`
- テーブル：`{table_name}`
- インデックス方式：HNSW / IVFFlat 等（採用方式・パラメータを明記）
- パーティション：`tenant_id` 等でのパーティション可否、運用方針
- 検索設定：topK、閾値、再ランキング有無

### 7. 更新・削除・再インデックス方針（重要）
- 更新トリガ（選択）：同期 / 非同期（推奨） / 定期
- 更新方式：差分 / 全量再構築（ブルーグリーン）
- 削除SLA：例）削除依頼から **N日以内**に検索対象から除外
- 削除方法：ソース削除→追随削除、または tombstone→物理削除
- 失敗時運用：再試行回数／DLQ／手動リカバリ／通知先

### 8. 監視・運用・品質
- 監視：取り込み件数、更新遅延、失敗件数、検索エラー、スロットル、p95応答
- ログ：`tenant_id`, `user_id`, `index_name`, `model`, `input_tokens` 等
- 品質評価：代表クエリセット、参照妥当性、ドリフト点検

### 9. セキュリティ設計（Azure）
- ネットワーク：Private Link/Endpoint、VNet統合、ExpressRoute/VPN
- 認証・認可：Entra ID、Managed Identity、RBAC（最小権限）
- 暗号化：保存時（CMK含む方針）、転送時TLS
- 秘密情報：Key Vault
- 脅威検知：Defender for Cloud 等
- 監査：Activity Log、Diagnostic Settings、Log Analytics
- コンテンツ安全性：PII/DLP/Content Filter（任意）

### 10. クラウド実装（Azure）— 主要コンポーネント一覧（記入欄）
- フロント：`{Front Door / App Gateway / Static Web Apps}`
- API：`{API Management}`
- アプリ／ゲートウェイ：`{Azure Functions / App Service}`
- ワークフロー：`{Durable Functions / Logic Apps / Data Factory}`
- キュー：`{Service Bus / Storage Queue}`
- ストレージ：`{Blob Storage}`
- OCR：`{Document Intelligence}`
- 埋め込み/LLM：`{Azure OpenAI}`
- 検索：`{Azure AI Search or PostgreSQL+pgvector}`
- 監視：`{Azure Monitor / App Insights / Log Analytics}`
- 秘密：`{Key Vault}`

### 11. 改定履歴
| 日付 | 版 | 変更内容 | 変更者 |
|---|---|---|---|

---

## （B）記入例：単一汎用インデックス案（できるだけ1つに寄せる）

### 0. 基本情報
- 文書名：インデックス定義書（`corp_knowledge_generic`）
- 対象システム／プロジェクト名：社内RAG基盤
- 作成日／版数：v1.0
- 承認者：業務／情シス／セキュリティ／開発

### 1. インデックス概要
- インデックス名：`corp_knowledge_generic`
- 用途（ユーザー/ユースケース）：
  - イントラ文書・手順書・規程・FAQの横断検索（部門横断。ただし権限はtenantで分離）
- 対象範囲：
  - 全社文書（ただし `tenant_id` によりアクセス制御）
- 期待する回答範囲：
  - “該当部門が閲覧可能な文書のみ” から根拠付き回答
- データ鮮度要求：
  - 反映SLA：**15分以内**
- 想定規模：
  - 文書数：〜200万チャンク相当（段階的拡張）
- SLA/SLO：
  - 検索応答：p95 2s以内（ハイブリッド検索 + topK調整）

### 2. データソース定義（入力）
- ソース種別：SharePoint Online / OneDrive / 一部Blob手動投入
- 対象ファイル形式：PDF/Word/Excel/Text
- 格納先：
  - 生データ：Blob Storage `container=raw-docs`
  - 抽出結果：Blob Storage `container=processed-text`
- 更新検知方法：
  - SharePoint変更通知 → Graph Webhook → Functions
  - キュー：Service Bus（topic: doc-changed）
- 除外条件：
  - 機密区分 `confidentiality=secret` は取り込まない（別基盤）

### 3. 前処理・抽出・正規化
- 文字抽出：Azure AI Document Intelligence
- OCR方針：
  - スキャンPDFのみOCR（confidence閾値<0.7は“要レビュー”）
- チャンキング：
  - 800 tokens、overlap 120
  - 見出し優先分割（Heading単位）
- 品質チェック：
  - 空/短すぎる（<50 tokens）チャンク除外
  - ランダム1%サンプルを週次レビュー

### 4. メタデータ設計
#### 4.1 メタデータ一覧（抜粋）
| key | 型 | 必須 | 用途 | 例 |
|---|---:|:---:|---|---|
| tenant_id | string | ✅ | 強制フィルタ | SG-Sales |
| doc_type | string | ✅ | ユースケース切替（任意） | rule / manual / faq |
| source_system | string | ✅ | 運用 | SharePoint |
| original_doc_id | string | ✅ | 冪等/来歴 | sp-itemId |
| chunk_id | string | ✅ | 冪等 | sp-itemId#12 |
| confidentiality | string | ✅ | DLP | internal |

#### 4.2 アクセス制御
- 認証：Entra ID
- テナント解決：
  - `groups` クレーム → `tenant_id` にマッピング（テーブル管理）
- 検索時の強制フィルタ：
  - `tenant_id = resolved_tenant_id` を必ず付与
  - UIで doc_type を指定しても、tenantフィルタは常に優先

### 5. 埋め込み設計
- Embedding：Azure OpenAI
- モデル：text-embedding-3-large（例）
- 次元数：3072（例）
- 再計算条件：
  - チャンキングルール変更、embeddingモデル変更時は全量再構築（ブルーグリーン）

### 6. ベクトル格納・インデックス方式
#### 6.A Azure AI Search
- Index名：`corp-knowledge`
- フィールド：
  - content（searchable）
  - vector（vector）
  - tenant_id（filterable）
  - doc_type（filterable）
  - updated_at（filterable/sortable）
- 検索設定：
  - ハイブリッド（BM25 + vector）
  - topK=20、スコア閾値=0.2（要実測で調整）

### 7. 更新・削除・再インデックス
- 更新トリガ：非同期（推奨）
- フロー：
  - 変更検知 → Service Bus → Durable Functions（抽出→chunk→embed→index upsert）
- 反映SLA：15分
- 削除：
  - ソース削除通知→Search delete（chunk_id単位）→監査ログ記録
- 失敗時運用：
  - 3回リトライ、DLQへ送る、運用が再投入/再実行

### 8. 監視・品質
- 監視：
  - backlog（キュー滞留時間）>10分でアラート
  - index更新失敗率 >1%でアラート
- ログ：
  - tenant_id/index_name/doc_count/chunk_count/token_usage を必須
- 品質評価：
  - 代表100問×部門（回収・改善）

### 9. セキュリティ
- Private Endpoint：Storage/OpenAI/Search/PostgreSQL（将来）すべて閉域
- Managed Identity：Functions→Search/Storageへアクセス
- Key Vault：秘密管理
- Defender for Cloud：有効化（方針と例外を明文化）
- 監査：Activity Log + Diagnostic to Log Analytics（保持365日）

### 10. クラウド実装
- フロント：Static Web Apps + Front Door
- API：API Management
- アプリ：Azure Functions（VNet統合）
- ワークフロー：Durable Functions
- キュー：Service Bus
- OCR：Document Intelligence
- Embedding/LLM：Azure OpenAI
- 検索：Azure AI Search
- 監視：App Insights + Log Analytics
- 秘密：Key Vault

### 11. 改定履歴
| 日付 | 版 | 変更内容 | 変更者 |
|---|---|---|---|
| 2026-01-13 | 1.0 | 初版 | - |

---

## （C）記入例：複数インデックス案（コーパス分割＋ルーティング）

> 複数インデックス案では、**「インデックス定義書」をインデックスごとに作り**、さらに **“インデックス群のルーティング仕様”** を追加で定義します。

### （C-0）インデックス群ルーティング仕様（追加セクション）
- ルーティング方式：
  - `useCase`（UI選択 or API引数） + `tenant_id`（JWTから決定）で検索先を決定
- 許可制御：
  - `tenant_id` ごとに許可useCaseを管理（マスタ：`tenant_usecase_map`）
  - 未許可useCaseは 403
- ルーティング表（例）：

| useCase | 対象インデックス | 用途 |
|---|---|---|
| `policy` | `idx_policy` | 規程・法務・ガバナンス |
| `manual` | `idx_manual` | 製品/業務手順書（図表多め） |
| `faq` | `idx_faq` | FAQ/問合せナレッジ（短文・即答） |

- ゲートウェイ処理（要点）：
  1) Entra ID JWT検証 → tenant_id決定  
  2) useCase許可チェック（tenant_usecase_map）  
  3) 対象インデックスへ検索（必ず tenant_id を強制フィルタ）  
  4) 監査ログ（tenant_id/useCase/index）を記録

---

### （C-1）インデックス定義書：`idx_policy`（規程・法務）

#### 0. 基本情報
- 文書名：インデックス定義書（`idx_policy`）
- 対象：規程・法務・ガバナンス文書（改定頻度：中）

#### 1. インデックス概要
- 反映SLA：60分以内（即時性は最優先ではない）
- 想定：長文、章立て強い、根拠引用が重要

#### 2. データソース
- SharePoint：`/sites/policy/` 以下
- 除外：ドラフト（metadata: status=draft）

#### 3. 前処理
- チャンキング：見出し単位 + 1000 tokens、overlap 150
- OCR：必要時のみ

#### 4. メタデータ（抜粋）
- tenant_id（必須）
- doc_category=policy（固定）
- effective_date（施行日）、revision（版）

#### 5. 埋め込み
- Azure OpenAI embedding（共通）

#### 6. 格納
- Azure AI Search Index：`policy-knowledge`
- Semantic Ranker：ON（章タイトルのヒットを重視）

#### 7. 更新・削除
- 更新：日次バッチ + 手動即時（緊急改定のみ）
- 削除：即時（コンプラ要請）

#### 8. 監視
- 改定漏れ：SharePoint側の改定通知件数とIndex更新件数の突合

#### 9. セキュリティ
- 監査ログ保持：7年（要件次第）
- 閲覧権限：テナント内でもロールで制限（rolesベースの追加フィルタ）

#### 10. 実装
- パイプライン：Data Factory（日次） + Functions（手動即時）

---

### （C-2）インデックス定義書：`idx_manual`（手順書・マニュアル）

#### 1. インデックス概要
- 反映SLA：15分以内（現場の更新が頻繁）
- 特性：図表や箇条書きが多い

#### 3. 前処理
- Document Intelligenceで表抽出を有効化
- チャンキング：800 tokens、overlap 120（汎用と同等）

#### 6. 格納（選択肢）
- (推奨) Azure AI Search：`manual-knowledge`
- (代替) PostgreSQL+pgvector：手順書だけ別DBでスケール調整

#### 7. 更新
- Event駆動 + キュー（Service Bus）で非同期反映

---

### （C-3）インデックス定義書：`idx_faq`（FAQ・問合せ）

#### 1. インデックス概要
- 反映SLA：5分以内
- 特性：短文、回答候補を素早く返す（topK小さめ）

#### 3. 前処理
- チャンキング：基本不要（1QA=1チャンク）
- 正規化：表記ゆれ辞書、タイトル強調

#### 6. 格納
- Azure AI Search：`faq-knowledge`
- 検索設定：
  - topK=5（小さく）
  - スコア閾値を高め（誤ヒットを抑える）
  - 必要ならFAQ専用のrerank

#### 7. 更新
- 追記が多いので差分upsert中心

---

# 付録：AWS→Azure 対応表（参考）
| AWS | Azureの代表的な置き換え候補 |
|---|---|
| S3 | Blob Storage |
| API Gateway | API Management |
| Lambda | Azure Functions |
| Step Functions / EventBridge | Durable Functions / Logic Apps / Data Factory（+ Service Bus） |
| Textract | Azure AI Document Intelligence |
| Bedrock（LLM/Embedding） | Azure OpenAI |
| Bedrock Knowledge Base | Azure AI Search（index）＋RAGオーケストレーション（Functions/App） |
| Aurora PostgreSQL + pgvector | Azure Database for PostgreSQL + pgvector |
| CloudWatch / CloudTrail / Config | Azure Monitor / Activity Log / Policy（＋Log Analytics） |
| WAF / Shield | App Gateway WAF / Front Door WAF（＋DDoS Protection） |
| KMS / Secrets Manager | Key Vault（＋CMK） |
| GuardDuty / Security Hub | Microsoft Defender for Cloud |


