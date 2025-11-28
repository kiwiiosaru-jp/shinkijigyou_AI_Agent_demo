| 作業項目(大) | 作業項目(中) | タスク説明 | 作成物 | 顧客側 (いすゞ/M31/業務ユーザ) | DATUM STUDIO/ちゅらデータ |
| --- | --- | --- | --- | --- | --- |
| PJ運営 | PJ推進/管理 | WBS策定・更新、課題/リスク管理、アーキレビュー対応、RFP提示スケジュール遵守 | 進捗WBS、課題/リスク一覧、アーキレビュー回答 | A | R |
| PJ運営 | 定例/報告 | 定例MTG運営、議事、最終報告資料作成 | 定例資料、議事録、最終報告資料 | A | R |
| アーキ設計 | 全体アーキ/セキュリティ | RFP標準構成に基づくシステム構成図、VPC/閉域(PrivateLink/VPCE)、WAF/Shield、監査/ログ(CloudTrail/CW/Config)設計、ガイドライン準拠 | アーキテクチャ図、設計書、レビュー回答 | A | R |
| アーキ設計 | 非機能・監視 | 非機能要件(RPO/RTO/稼働率/性能)反映、監視・運用Runbook(CloudWatch/Athena/Security Hub等)設計 | 非機能設計書、運用/監視Runbook | A | R |
| インフラ構築 | VPC/ネットワーク | VPC、VPN/TGW、VPCE(Private APIGW等)、ALB+Fargate、SG/NACL構築と疎通確認 | 構成図、設定パラメータ、疎通結果 | I | R |
| インフラ構築 | セキュリティ/監査 | WAF/Shield設定、CloudTrail/Config/GuardDuty/SecurityHub/Inspector/KMS設定、Secrets管理 | 設定資料、運用Runbook | I | R |
| インフラ構築 | 認証基盤 | CognitoとEntra SAML連携設定、属性マッピング、コールバック整合、E2Eテスト | 設定手順、接続試験結果 | I | R |
| アプリ開発 | 共通API/LLMゲートウェイ | APIGW+Lambda（router/adapter/Predict 等）実装、リランク処理、認可/メタフィルタ実装、ログ/履歴(DynamoDB) | ソースコード、API仕様、テスト結果 | I | R |
| アプリ開発 | フロント（画面） | React SPA 実装（出典/プレビュー/フィルタ/履歴/評価等）、ALB+Fargate配信設定、Cognito連携 | ソースコード、UI仕様、テスト結果 | I | R |
| データパイプライン | 取得/抽出/加工/チャンク/ベクトル化 | Box/S3/SharePoint連携、Python前処理（抽出、クレンジング、正規化、辞書補正、Q&A化、メタ付与）、チャンク分割、Embedding、KB登録/pgvector格納 | パイプライン実装、手順書、動作確認レポート | I | R |
| データパイプライン | スケジューリング/自動化 | バッチスケジュール設定（EventBridge/Step Functions等）、エラー再実行、監視連携 | スケジュール設定書、運用手順 | I | R |
| 管理機能 | 管理UI/部門UI | システム/部門管理UI（ダッシュボード、ログ/履歴閲覧、ユーザ/部門/容量、お知らせ、文書・ACL管理、アップロード/インデックス管理） | 画面仕様、実装、テスト結果 | I | R |
| 運用設計 | 運用/リリース/復旧 | 運用手順、リリース手順、バックアップ/復旧手順（DynamoDB PITR、Auroraスナップショット）、性能/負荷/セキュリティテスト | 運用手順書、復旧手順書、テストレポート | A | R |
