# 提案方針・フィットギャップ（GenU フルオプション基盤ベース）

## 1. 当社提案方針の案です
- **短期（3か月）で「共通RAG/LLMゲートウェイ＋AgentCore」を最小構築し、全社標準APIでガバナンスを効かせる。**
- **データ統合は Box → 共通パイプライン（前処理/メタ付与/チャンク設定可）→ ベクトルDB（既定: Bedrock KB/AOSS、高要求: Aurora pgvector）で段階拡張。**
- **非機能（性能・可用性・監視・セキュリティ）はフェーズごとに強化し、オートスケールと復旧方式を要素別に設計（アプリ/API/LLM/RAG/ベクトルDB）。**
- **AIエージェントの将来展開を見据え、フェーズ1から AgentCore を基盤に組込み、以降のエージェント開発は AgentCore API/ガバナンス準拠を必須化。**

## 2. エグゼクティブサマリ
- GenUフルオプションを核に、**共通LLMゲートウェイ/APIを全社標準インターフェース**として提示し、以降のアプリはこのAPIを利用するガバナンスモデルを採用。
- **3か月MVPで実装すべき最小スコープ**: 認証（Entra SAML）、共通API（LLM/RAG）、Box取込パイプライン最小版、RAG検索（KB/AOSS or Aurora pgvectorのどちらかを優先選択）、監視/バックアップ基礎、AgentCore組込み。
- **フェーズ2以降**で: Box差分/Webhook・メタ付与/Q&A化、管理UI拡張（容量/部門/お知らせ/ログ）、セキュリティ強化（WAF/PrivateLink/GuardDuty/Security Hub）、オートスケール/性能・可用性設計の細分化、Fargate/ALBが必要ならここで切替。
- **リスク/ギャップ**: Box連携（pgvector側はAPI実装が必要）、PostgreSQLベクトル指定、ALB+Fargate要求、管理UI拡張。いずれも追加開発で吸収可。工数は前回より辛めに再積算。

## 3. 段階開発プラン
- **フェーズ1（～3か月、MVP）**  
  - 認証: Entra SAML + Cognito。  
  - 共通API/ゲートウェイ: LLM/RAG/AgentCore を API Gateway + Lambda で提供し、社内アプリはこのAPI利用を必須化。  
  - RAG: KB（Bedrock KB/AOSS 既定）または Aurora pgvector のどちらかを優先採用（両対応も可だが期間タイトなら片側）。  
  - Box連携: 定期Pull＋手動起動の最小パイプライン（前処理/チャンク/メタ最小）。  
  - AgentCore: 基盤として組込み（今後のエージェント開発は AgentCore API へ集約）。  
  - 非機能: RPO/RTO基準、DDB PITR、S3バージョン管理、基本監視/ダッシュボード、ガードレール基礎。  
  - 可用性/性能: サーバレス優先、チャンク上限/軽量モデル/同時実行の制御を初期設定。

- **フェーズ2（+1–2か月）**  
  - Box Webhook/差分取込、メタ付与（要約/タグ/Q&A）、部門別アクセス制御/容量管理。  
  - 管理UI拡張（容量/部門/お知らせ/ログ閲覧）、セキュリティ強化（WAF、PrivateLink/VPC内化、GuardDuty/SecurityHub/Config統合）。  
  - オートスケール調整: フロント（CF/Lambda@Edge or Fargate）、バック/API、LLM呼び出し同時実行、RAG（KB/Kendra or pgvector）、ベクトルDBのスロット/キャパ。  
  - Fargate/ALB構成が必須ならここで切替。

- **フェーズ3（+1–2か月）**  
  - DR演習、性能テスト自動化、AIガバナンス強化（バイアス/PII）。  
  - メタデータ高度化（分類・辞書補正自動化）、部門ダッシュボード高度化、追加SaaS連携（SharePoint 等）。  
  - コスト最適化（自動起動/停止、モデル/チャンク最適化）。

## 4. フィットギャップ一覧（実装フェーズつき、工数はざっくりご参考です）
| 項目 | RFP要件 | GenU標準/フル | 対応方針 | 実装フェーズ | 追加開発/設定目安（人日） |
| --- | --- | --- | --- | --- | --- |
| 認証・SSO | Entra SAML | ◯ | IdP統合・ドメイン設定・テスト強化 | P1 | 8–12 |
| API/ガバナンス | 全社標準APIを義務化 | △（ゲートウェイはあるが標準化前提なし） | API利用必須の方針化、API仕様書/SDKを整備 | P1 | 5–8 |
| ネットワーク/セキュリティ | WAF/PrivateLink/多層監視 | △（WAF無効、パブリックCF+APIGW） | WAF追加、VPC+PrivateLink/Lambda内向き、SecurityHub/GuardDuty/Config統合 | P2 | 20–25 |
| ネットワーク構成 | ALB+Fargate経由 | ×（CF+S3+APIGW+Lambda） | 必須なら Fargate/ALB に切替。不要なら現行サーバレス提案 | P2 | 20–25 |
| RAG KB（既定） | ベクトルDB=PostgreSQL指定 | △（Bedrock KB+AOSS） | Aurora pgvectorを追加オプション。KB/AOSSは既定高速ルート | P1(片側) / P2(両対応) | 25–30 |
| RAG Kendra | Kendra利用 | △（標準機能） | 必須でなければスコープ外。要なら維持 | P1/Optional | 0–10 |
| Boxパイプライン | Box起点、前処理/メタ付与、スケジュール/通知 | × | Box SDK/Webhook＋Step Functions/Batch/Lambda、メタ付与可 | P1(最小) / P2(差分・Webhook) | 30–40 |
| Box→pgvector | Boxアダプタ無し | × | API連携実装（Box API → S3 → 前処理 → pgvector） | P1 | 12–18 |
| 前処理カスタム | Python差込・標準ロジック置換 | △（サンプル最小） | 前処理ステップをモジュール化し差替え可能に設計 | P1 | 10–15 |
| リランク（pgvector） | リランク必須 | △（標準は KB+AOSS のみ） | 新規実装（再ランキング API 呼び出し or 内製） | P1/P2 | 10–15 |
| 管理UI | 容量/部門/お知らせ/ログ | △（簡素） | 管理UI拡張 | P2 | 18–24 |
| AgentCore | エージェント基盤 | ◯（現行あり） | P1で組込み、以降のエージェント開発を AgentCore 準拠に統制 | P1 | 5–8 |
| メタ付与/Q&A | 要約/タグ/Q&A化 | △ | 前処理で生成系ジョブ追加 | P2 | 12–18 |
| オートスケール | 要素別（フロント/バック/LLM/RAG/DB） | △ | 各コンポーネントで同時実行/スロット/スケール設定 | P2 | 10–15 |
| 性能 | 応答10–60秒 | ◯（要チューニング） | モデル/チャンク上限、同時実行、キャッシュ調整 | P1 | 5–8 |
| 可用性 | RPO24h/RTO12h、稼働率99.7% | △ | DDB PITR、Aurora バックアップ/マルチAZ、Runbook/DR手順 | P1/P2 | 8–12 |
| 監視 | トークン/ファイル/異常検知 | △ | Dashboard項目追加、アラート調整 | P1 | 6–8 |

## 5. 工数目安（概算・人日、実装中心／ざっくりご参考の工数です。AWSアーキテクトに見ていただいた方がいいかも）
- フェーズ1: **60–80人日**  
  - SAML統合 8–12、API標準化 5–8、RAG配線 KB or pgvector 25–30、Box最小取込 20–25、前処理差込 10–15、AgentCore組込み 5–8、性能/可用性初期設定 8–10、監視/Runbook 6–8
- フェーズ2: **55–75人日**  
  - Box Webhook/差分 12–18、メタ付与/Q&A 12–18、管理UI拡張 18–24、Security/WAF/PrivateLink/Hub/Duty 20–25、オートスケール/性能調整 10–15、Fargate/ALB切替（必要時）20–25（重複工数は調整可能）
- フェーズ3: **40–55人日**  
  - DR/性能テスト自動化 10–14、AIガバナンス強化 8–12、ダッシュボード高度化 10–12、追加SaaS連携 10–15、コスト最適化 6–8

※要件定義・テスト・PMは別途。スコープに応じ前後します。

## 6. メモ・補足
- Kendra: RFPに明示なし。不要なら外し、KB/pgvectorに一本化。  
- Box: Kendra向けアダプタはあるが、pgvectorルートは自前API実装が必要。  
- オートスケール/可用性:  
  - フロント/静的: CF+S3（サーバレス）。Fargate採用時はタスク/ターゲット数でスケール。  
  - API/Lambda: Concurrency・Retry・タイムアウト調整。APIGWスロット設定。  
  - LLM: モデル同時実行上限・軽量モデル優先・チャンク数制御。  
  - RAG KB: KB/AOSS スロット、Aurora pgvector は ACU/最大接続/マルチAZ。  
  - 可用性: サーバレスはマネージド冗長。AuroraはマルチAZ、バックアップ方針明示。DRはコールド/ウォーム選択（RTO/RPOとコストで決定）。
