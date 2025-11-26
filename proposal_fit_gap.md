# RFP フィットギャップおよび段階開発方針（GenU フルオプションベース）

## 1. フィットギャップ一覧（GenU 標準＋フルオプションとの比較）
| 項目 | RFP要件 | GenU標準/フル | 対応方針 | 追加開発/設定目安 |
| --- | --- | --- | --- | --- |
| 認証・SSO | Entra ID SAML | ◯（Cognito SAML） | IdP設定とドメイン調整 | 5–8人日 |
| セキュリティ多層 | WAF/PrivateLink/多層監視 | △（現行はパブリックCF+APIGW+Guardrail、WAF未使用） | WAF(us-east-1)追加、VPC+PrivateLink/Lambda内向き構成、Security Hub/GuardDuty連携 | 15–20人日 |
| ネットワーク | ALB+Fargate経由プロキシ | ×（CF+S3+APIGW+Lambda） | 準拠が必須ならFargate/ALBで静的配信・APIプロキシを追加。準拠不要なら現行サーバレスを提案 | 15–20人日（Fargate採用時） |
| RAG KB | ベクトルDBはPostgreSQL | △（現行 Bedrock KB + AOSS, Kendra） | Aurora Serverless pgvectorオプションを追加し、KBパスを二系統化（既定はBedrock KB/AOSSで高速、PGは要件準拠オプション） | 20–25人日 |
| RAG Kendra | Kendra検索+リランク | ◯ | そのまま適用 | 0 |
| RAGパイプライン | Boxから取得、前処理、メタデータ付与、スケジュール/通知 | ×（S3前提サンプルのみ） | Box SDK/Webhook＋Step Functions/Batch/Lambda で取込パイプライン実装、メタデータ付与とリトライ | 25–35人日 |
| アップロードUI | 画面アップロード | ◯ | Box連携をUIに追加（Box送信用API呼び出し） | 5–8人日 |
| メタ情報付与 | タグ/要約/Q&A変換等 | △（メタデータなし前提。サンプル可） | 生成系前処理（Bedrock invoke/Converse）をパイプラインに組込み | 10–15人日 |
| 管理者機能 | ダッシュボード、ログ/履歴、容量管理、部門管理 | △（利用状況はCloudWatch/DDBにあり、UI簡素） | 専用管理画面拡張（部門/容量/お知らせ/ログ閲覧）、CloudWatchメトリクス集約 | 15–20人日 |
| 運用 | バックアップ、DR、監視、RPO/RTO | △（DDB/S3自動冗長、Dashboardあり、RPO/RTO未定義） | Aurora pgvector自動バックアップ、DDB PITR、CloudWatch/Security Hub/Config、障害Runbook | 8–12人日 |
| オートスケール | バッチ/サービスのオートスケール | △（サーバレスは自動、Fargate採用時は設定要） | Fargate/HPA設定、Lambda Concurrency/Retry、Kendra/KB同時実行上限調整 | 6–10人日 |
| 画面要件 | 検索結果プレビュー/引用/履歴/フィードバック | ◯ | 既存機能でカバー | 0 |
| 非機能（性能） | 応答10–60秒 | ◯（モデル/トークン次第） | チャンク数上限、軽量モデル初期値、バックプレッシャ制御 | 3–5人日 |
| 非機能（可用性） | 稼働率99.7%、RPO24h/RTO12h | △ | Aurora Serverless-MultiAZ、DDB PITR、運用Runbook、S3バージョン管理 | 5–8人日 |
| データ削除 | 30日以内検索対象除外 | ◯（KB/Kendraの削除同期） | バックアップ保持ポリシー併記 | 0 |
| 監視 | 追加要件（トークン/ファイル数/異常検知） | △ | 既存Dashboardに項目追加、アラートチューニング | 5人日 |

## 2. 段階開発プラン（3か月内をフェーズ1）
- **フェーズ1（～3か月、MVP基盤）**  
  - コア: 認証（SAML含む）、RAG（Bedrock KB/AOSS or Aurora pgvectorは一方選択）、Kendra、UI（検索/引用/履歴/フィードバック）、基本ガードレール、CloudWatch監視、最小管理画面（利用状況/ログ参照）、基本バックアップ/タグ付け、Boxパイプライン最小版（定期Pull + 手動実行）、Box→S3インジェスト＋メタデータ最小（ファイル属性）。  
  - 非機能: RPO/RTO定義、Lambda/KB/Kendraスロット設定、DDB PITR、S3バージョン管理、初期Runbook。

- **フェーズ2（+1–2か月）**  
  - Box連携強化（Webhook通知・差分取込）、メタ情報付与（要約/タグ/Q&A）、部門別アクセス制御/容量管理、管理UI拡張（容量/部門管理/お知らせ）、Security Hub/GuardDuty/Config統合ダッシュボード、Fargate/ALB構成が必須ならここで切替。  
  - オートスケール調整（Fargate/HPA、Kendra/KB同時実行上限、モデル/チャンク制御）。

- **フェーズ3（+1–2か月）**  
  - 高度運用（DR演習、性能テスト自動化、品質/敵対的テスト）、メタデータ高度化（分類・辞書補正自動化）、部門ダッシュボード高度化、SaaS連携拡張（SharePoint など）、マルチテナント強化。  
  - 追加のAIガバナンス（バイアス/PII強化）、コスト最適化（自動起動/停止）。

## 3. 工数目安（概算・人日、実装中心）
- フェーズ1: 45–60人日  
  - Box最小取込 15、SAML 5、RAG配線/PGオプション 15、監視/Runbook 5、管理UI最小 5、非機能設定 5
- フェーズ2: 40–55人日  
  - Box Webhook/差分 10、メタ付与/Q&A 10–15、管理UI拡張 10、Security Hub/GuardDuty/Config統合 5–8、Fargate化/ALB（必要時）15–20、オートスケール/パフォーマンス調整 5
- フェーズ3: 30–45人日  
  - DR/性能テスト自動化 8–12、AIガバナンス強化 5–8、ダッシュボード高度化 8–10、追加SaaS連携 8–12

※要件定義・テスト・PMは別途。短納期のためスコープ優先度を調整。

## 4. エグゼクティブサマリ（報告用）
- GenUフルオプションをベースに、RFPの大半をカバー可能。認証・RAG検索（Kendra/KB）・UI・監視・ガードレールは標準機能で充当。  
- ギャップは「Box起点パイプライン」「PostgreSQLベクトルDB指定」「ALB+Fargate/Private構成」「管理UI拡張（容量/部門/お知らせ）」が中心。これらは追加開発で対応可能。  
- 3か月MVPで「Box連携最小＋SAML＋RAG（KB or pgvector）＋Kendra＋監視＋基本管理UI」を提供し、導入部門への早期展開を優先。Phase2以降でBox差分連携/メタ付与・管理UI高度化・セキュリティ強化（WAF/PrivateLink/GuardDuty）を段階的に実装。  
- 性能・信頼性（RPO/RTO/オートスケール）はフェーズ1で基準値を設定し、フェーズ2で負荷テスト・スケール調整を行う。  
- Box連携とPGベクトル対応を早期に進めることで、RFPの必須スコープを確実に満たしつつ、サーバレス基盤の強み（迅速デリバリ・コスト最適化）を活かす提案が妥当。

## 5. 当社提案方針（社長向け進言）
- 初期3か月で「安全に使えるRAG基盤」を最速で提供し、ROIの高い部門から展開する。Box連携とPGベクトルをオプション設計し、RFP必須を満たしつつスピードを優先。  
- 非機能（性能・可用性・監視）はフェーズ1で基準を敷き、フェーズ2でWAF/PrivateLink/GuardDuty/性能テストを強化。  
- ALB+Fargateが必須の場合はPhase2で切替、不要なら現行のサーバレス（CF+APIGW+Lambda+KB/Kendra）で短納期・低運用を提案。  
- フィット・ギャップを明示し、追加開発工数を透明化。段階開発でリスクを抑え、2026年4月までの全社展開を見据えた拡張計画を提示する。
