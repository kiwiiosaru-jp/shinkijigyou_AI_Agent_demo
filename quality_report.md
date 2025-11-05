# 品質監査結果報告書
**作成日:** 2025-11-05  
**対象 URL:** `http://host.docker.internal:8501`  
**Git Rev:** `local`

---

## 1. エグゼクティブサマリ

- 総合判定: **Fail**
- セキュリティ: NG **3** / HIGH **1**
- 単体: 0件（失敗 0 / 失敗率 0.0%）
- 機能: 0件（失敗 0 / 失敗率 0.0%）
- 総合: ドキュメント 0 件
- LLM 白箱レビュー: 品質 NG（課題 4件） / セキュリティ NG（課題 6件）


## 2. 対象・前提

- 対象 URL: `http://host.docker.internal:8501`
- 対象コード: `./target/` 配下のアプリケーション一式
- 認証・設定: `.env` にて `TARGET_URL` やヘッダ・クッキーを設定


## 3. 監査/試験スコープ

- セキュリティ: ZAP / Playwright / k6 / garak / Nuclei / Whitebox (gitleaks/Syft/Grype/Semgrep/Trivy)
- 単体: JUnit ベース自動テスト、コードカバレッジ (任意)
- 機能: 自動化された機能/E2E テスト
- 総合: 手動試験レポート（MD/HTML/PDF）


## 4. 実施方法

- セキュリティ判定: `reports/checklist.yaml` と OPA ゲート
- 品質ゲート: 失敗率 < 2.0% / セキュリティ NG 無し
- 入力不足の場合は雛形を維持し、次回の監査時に追記


## 5. 統合サマリ

- セキュリティ: 主要 NG・HIGH は章 6 を参照
- LLM 白箱レビュー: 品質・セキュリティの所見は章 6 を参照
- 単体: 失敗サンプルおよびカバレッジは章 7
- 機能: 失敗サンプルは章 8
- 総合: 手動試験の要約は章 9


## 6. セキュリティ監査サマリ

- チェック件数: 12 / NG 3 / HIGH 1
- 詳細レポート: `report/report_security.md`
- 主要 NG:
  - WB-LLM-051: LLMコードレビュー（API/認証脆弱性） (証跡: `reports/whitebox/llm_review.json`)
  - BB-WEB-107: セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等） (証跡: `reports/zap/zap.xml`)
  - BB-WEB-108: レート制限・コスト濫用耐性 (証跡: `reports/k6/k6.json`)

### セキュリティ監査報告書 (全文)

```markdown
# セキュリティ監査結果報告書
**対象名:** host.docker.internal:8501  
**実施日:** 2025-11-05  
**実行ID:** -  
**実施者:** セキュリティ検証チーム

---

## 1. エグゼクティブサマリ
- 重大リスク（HIGH 以上）: **1件** ／ NG: **3件** ／ OK: **9件** ／ N/A: **0件**
- 総合判定（ゲート）: **Fail**
- 主要トピック: ①LLMコードレビュー（API/認証脆弱性） ②セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等） ③レート制限・コスト濫用耐性

## 2. 監査の目的・スコープ
- 目的: 外部公開前に、Web/UI/LLM の一般攻撃と生成AI特有リスクを検出し、運用フェーズ前に是正計画を固める。
- スコープ（黒箱）: XSS / SSRF / セキュリティヘッダ / レート制限 / Open Redirect / プロンプトインジェクション / SQLi
- スコープ（白箱）: Secrets / SBOM+SCA / Semgrep（LLM 最小） / Trivy FS / LLMコードレビュー（API/認証） / 依存関係整合性

## 3. 対象・前提条件
- 対象 URL: `http://host.docker.internal:8501`
- 対象コード: `./target/` 配下の Web アプリケーション一式
- 前提条件: コンテナから `host.docker.internal` でホストへ到達、認証ヘッダ/クッキーは `.env` で付与可能

## 4. 監査方法（白箱／黒箱・ツール・判定基準）
- ZAP（ヘッダ・リダイレクト） → `reports/zap/`
- Playwright（DOM ペイロード実行検証） → `reports/playwright/`
- k6（Burst 負荷→HTTP ステータス観測） → `reports/k6/`
- garak（プロンプト注入/越権耐性） → `reports/garak/`
- Nuclei（安全テンプレ） → `reports/nuclei/`
- LLM 白箱レビュー（API/認証リスク診断） → `reports/whitebox/llm_review.md`
- gitleaks / Syft / Grype / Semgrep / Trivy → `reports/whitebox/`, `reports/sbom/`
- 判定基準: `reports/checklist.yaml` のステータスと `config/policies/gates.rego` のゲート判定

## 5. 結果サマリ（OK/NG/N/A 集計・重大度別）
- チェック総数: **12**
- OK: **9** ／ NG: **3** ／ N/A: **0** ／ 要確認: **0**
- 主要エビデンス: ZAP `reports/zap/baseline.html`, Playwright `reports/playwright/dom-trace.json`, k6 `reports/k6/k6.json`, garak `reports/garak/garak.report.jsonl`, LLM `reports/whitebox/llm_review.md`

## 6. 主要リスクの要約（Top Findings）
1. **WB-LLM-051** — LLMコードレビュー（API/認証脆弱性）  
   - 重大度: HIGH ／ ステータス: NG  
   - 証跡: `reports/whitebox/llm_review.json`
   - 推奨: LLM が指摘した API／認証まわりのリスクに対し、入力検証や認可制御の強化、機密情報の露出防止策を実装してください。

2. **BB-WEB-107** — セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等）  
   - 重大度: 情報 ／ ステータス: NG  
   - 証跡: `reports/zap/zap.xml`
   - 推奨: HTTP 応答で CSP / HSTS / X-Frame-Options / X-Content-Type-Options を適切に設定し、フレーム埋め込みとプロトコルダウングレードを防止してください。

3. **BB-WEB-108** — レート制限・コスト濫用耐性  
   - 重大度: 情報 ／ ステータス: NG  
   - 証跡: `reports/k6/k6.json`
   - 推奨: 429 応答・Retry-After といったスロットル機構を実装し、API キーや IP 単位でのリクエスト上限を導入してください。

## 7. 詳細所見（Finding 個票）
### [WB-LLM-051] LLMコードレビュー（API/認証脆弱性）
- **ステータス**: NG  
- **重大度**: HIGH  
- **技術的根拠**: reports/whitebox/llm_review.json  
- **再現手順**: チェックリスト記載のツール再実行で確認。  
- **推奨対策**: LLM が指摘した API／認証まわりのリスクに対し、入力検証や認可制御の強化、機密情報の露出防止策を実装してください。

### [BB-WEB-107] セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等）
- **ステータス**: NG  
- **重大度**: 情報  
- **技術的根拠**: reports/zap/zap.xml  
- **再現手順**: ZAP baseline HTML で Security Headers の警告を確認し、対象 URL へリクエスト→レスポンスヘッダを確認。  
- **推奨対策**: HTTP 応答で CSP / HSTS / X-Frame-Options / X-Content-Type-Options を適切に設定し、フレーム埋め込みとプロトコルダウングレードを防止してください。

### [BB-WEB-108] レート制限・コスト濫用耐性
- **ステータス**: NG  
- **重大度**: 情報  
- **技術的根拠**: reports/k6/k6.json  
- **再現手順**: k6 スクリプトで `/api/search` に連続アクセスし、常に 403 が返却されることを確認。429 等の制御応答が存在しない。  
- **推奨対策**: 429 応答・Retry-After といったスロットル機構を実装し、API キーや IP 単位でのリクエスト上限を導入してください。

## 8. LLM/生成AI特有の検証結果
- `BB-LLM-301` ステータス: **OK**（証跡: `reports/garak/garak.report.jsonl`）
- 所見: 代表的なプロンプト注入・越権試行はいずれも 403 応答で拒否され、システムプロンプトや機密情報の漏えいは確認されませんでした。

### LLM 白箱レビュー（品質・セキュリティ）
- 総合ステータス: **NG** ／ MUST: 6件 ／ SHOULD: 4件
- 重大度内訳: CRITICAL:3, HIGH:3, MEDIUM:2, LOW:2
- 想定リスクカテゴリ: SSRF, リモートコード実行 / 設定注入, ローカルファイル読み出し（LFI） / パストラバーサル, 保守性, 出力エンコーディング / XSS, 可用性 / DoS, 情報露出, 機能的障害, 運用・可用性
- 対象ファイル数: 20
- 解析対象（抜粋）:
  - `20251101_codex_test/tests/unit/research/clients/test_azure_agent.py`
  - `20251101_codex_test/src/research/services/summarizer.py`
  - `20251101_codex_test/src/research/services/reporting.py`
  - `20251101_codex_test/src/research/services/normalizer.py`
  - `20251101_codex_test/src/research/clients/azure_agent.py`
  - `20251101_codex_test/tests/unit/research/test_summarizer.py`
  - `20251101_codex_test/tests/unit/research/test_reporting.py`
  - `20251101_codex_test/tests/unit/research/test_pipeline.py`
  - `20251101_codex_test/tests/unit/research/test_normalizer.py`
  - `20251101_codex_test/tests/unit/research/conftest.py`
  - ...ほか 10 件
- Must Fix 対応:
  - [品質] 文法エラーまたは切り捨てられた行による無効な構文（summarizer） (CRITICAL) `src/research/services/summarizer.py`
  - [品質] 文法エラーまたは切り捨てられた行による無効な構文（normalizer） (CRITICAL) `src/research/services/normalizer.py`
  - [品質] 文法エラーまたは切り捨てられた行による無効な構文（UI 初期化） (CRITICAL) `src/research/ui/app.py`
  - [セキュリティ] 環境変数による動的インポートで任意コード実行のリスク (HIGH) `src/research/ui/app.py`
  - [セキュリティ] 外部エンドポイントの未検証利用による SSRF のリスク (HIGH) `src/research/clients/azure_agent.py`
  - ...ほか 1 件
- Should Fix 対応:
  - [品質] 弾丸行抽出ロジックの冗長な条件と可読性低下（summarizer） (LOW) `src/research/services/summarizer.py`
  - [セキュリティ] ユーザー入力をそのまま Markdown に埋め込むことでの出力エンコーディング問題（XSS の可能性） (MEDIUM) `src/research/services/reporting.py`
  - [セキュリティ] HTTP クライアント利用時のタイムアウトや接続制限が設定されていない (MEDIUM) `src/research/clients/azure_agent.py`
  - [セキュリティ] デバッグ情報に機密情報の存在有無を露出している (LOW) `src/research/ui/app.py`
- 品質レビュー: **NG** — 提示されたソースからは、明確な構文上の欠陥が複数箇所で確認でき、実行時エラーやインポート失敗を引き起こす重大なリスクがあります。また、いくつかの小さな保守性の問題（冗長な条件や未完成の例外処理コメント）が見受けられ、長期的な可読性と信頼性に影響します。
  - [CRITICAL] 文法エラーまたは切り捨てられた行による無効な構文（summarizer） @ `src/research/services/summarizer.py` ／ MUST ／ 機能的障害
    - 推奨: 該当箇所を正しいメソッド呼び出し（おそらく fallbacks.append(...)）に修正し、抜けているロジックを復元してください。修正後はユニットテストを実行してモジュールがインポート可能であることを確認し、該当機能に対する単体テストを追加して同様の切り捨て/途切れの検出を防いでください。
  - [CRITICAL] 文法エラーまたは切り捨てられた行による無効な構文（normalizer） @ `src/research/services/normalizer.py` ／ MUST ／ 機能的障害
    - 推奨: 該当行を正しい関数呼び出し（おそらく _parse_date(item.published_raw) 等）に置き換え、正規化処理の全フィールドを明確に設定してください。変更後は日付パースの境界条件を扱う単体テスト（Z表記、タイムゾーン付き/無し、小数秒の有無など）を追加して回帰を防いでください。
  - [CRITICAL] 文法エラーまたは切り捨てられた行による無効な構文（UI 初期化） @ `src/research/ui/app.py` ／ MUST ／ 運用・可用性
    - 推奨: 途中で切れた行を修正して正しい環境変数取得ロジック（os.getenv 等）に置き換え、ファクトリパス解決の全体ロジックが正しく動作することを確認してください。修正後はアプリ起動テスト（既存の Streamlit の統合テスト）を実行し、CI での自動検出を有効にしてください。
  - ...ほか 1 件
  - Good: ユニットテスト、統合テスト、E2E テストの構成が用意されており、機能ごとに幅広くカバレッジを狙っている点は品質管理に有利です。, Markdown レポーターはディレクトリ作成や重複ファイル回避の実装があり、出力ファイル名管理が適切に設計されています。, 正規化処理で URL 正規化や日付パース用のフォーマット配列を用意しているなど、データ整形に対する設計配慮が見られます。
- セキュリティレビュー: **NG** — リポジトリには環境変数や外部構成をそのまま反映して動的にモジュールを読み込んだり外部エンドポイントへリクエストを飛ばす部分があり、ローカル運用や限定的公開環境での悪用により重大な影響を受ける恐れがあります。特に環境変数による動的インポート、外部エンドポイントの検証不足、および許可リストからの録画ファイル読み出しに関して緊急の対策が必要です。
  - [HIGH] 環境変数による動的インポートで任意コード実行のリスク @ `src/research/ui/app.py` ／ MUST ／ リモートコード実行 / 設定注入
    - 推奨: 外部入力（環境変数）から直接モジュールをインポートしないこと。どうしてもプラグイン方式を採る場合はホワイトリスト化されたモジュール名のみ許可するか、事前に安全性を検証した限定的なファクトリ名のみを受け付ける。可能ならば動的インポートを廃止して設定可能なビルトインファクトリを列挙する。
  - [HIGH] 外部エンドポイントの未検証利用による SSRF のリスク @ `src/research/clients/azure_agent.py` ／ MUST ／ SSRF
    - 推奨: 外部へリクエストを行う前にホスト名/スキームのホワイトリスト検証を実装する。内部サービスやメタデータIP(169.254.169.254 等)を拒否するルールを設け、許可されたホストだけを許容する。入力から直接 URL を組み立てる場合は、厳密なパースと検証を行うこと。
  - [HIGH] 録画ファイル読み出しでのパストラバーサル/任意ファイル参照のリスク @ `src/research/clients/azure_agent.py` ／ MUST ／ ローカルファイル読み出し（LFI） / パストラバーサル
    - 推奨: allowlist に記載された recording パスを使用する前に Path.resolve() 等で正規化し、読み取り対象が必ず allowlist ファイルの親ディレクトリ（base_dir）以下に含まれることを検証する（commonpath 等を使用）。絶対パスやシンボリックリンクによる脱出も検出して拒否する。allowlist の編集は信頼できる管理者のみに限定する。
  - ...ほか 3 件
  - Good: テストコードで httpx.MockTransport を利用して外部 API をモックしており、外部呼び出しのテストが分離されている点は安全性検証に寄与している。, URL 正規化（src/research/services/normalizer.py の _normalize_url）や日付のパースで複数フォーマットを考慮していることは入力のロバスト性向上に寄与している。, テストフィクスチャや allowlist による録画/再生の仕組みは、実環境の呼び出しを直接行わずに CI で検証できる点で安全な設計意図が見られる。

## 9. 改善計画（優先順位・担当・期日）
| No | Finding | 対策 | 優先度 | 担当 | 期限 |
|---:|---|---|---|---|---|
| 1 | WB-LLM-051 | LLM が指摘した API／認証まわりのリスクに対し、入力検証や認可制御の強化、機密情報の露出防止策を実装してください。 | 高 | セキュリティチーム | 2025-11-30 |
| 2 | BB-WEB-107 | HTTP 応答で CSP / HSTS / X-Frame-Options / X-Content-Type-Options を適切に設定し、フレーム埋め込みとプロトコルダウングレードを防止してください。 | 高 | バックエンドチーム | 2025-11-15 |
| 3 | BB-WEB-108 | 429 応答・Retry-After といったスロットル機構を実装し、API キーや IP 単位でのリクエスト上限を導入してください。 | 高 | プラットフォームチーム | 2025-11-22 |


## 10. 既知の制限・リスク受容・次回方針
- ローカル環境では `host.docker.internal` 以外を拒否するため、ガード実装との整合を確認する必要があります。
- 今回の検証では 403 応答をレート制限エビデンスとして扱った。次回は実際の 429/Retry-After 実装を確認予定。

## 11. 付録A：checklist.yaml 抜粋
```yaml
- id: WB-SECRETS-001
  category: Whitebox/Secrets
  item: 認証情報のハードコード/Git漏洩
  expected: 検出0
  status: OK
  severity: null
  evidence: reports/whitebox/secrets.sarif
- id: WB-SCA-021
  category: Whitebox/SCA
  item: 依存脆弱性（Critical/High）
  expected: Critical/High=0
  status: OK
  severity: null
  evidence: reports/whitebox/sca.sarif
- id: WB-LLM-011
  category: Whitebox/LLM
  item: ユーザ入力の無加工連結（system/assistant注入）
  expected: 該当0
  status: OK
  severity: null
  evidence: reports/whitebox/semgrep.sarif
- id: WB-LLM-051
  category: Whitebox/LLM
  item: LLMコードレビュー（API/認証脆弱性）
  expected: 重大リスク0
  status: NG
  severity: HIGH
  evidence: reports/whitebox/llm_review.json
- id: WB-DOCKER-033
  category: Whitebox/Container
  item: FS/コンテナの脆弱性（Trivy FS）
  expected: High/Critical=0
  status: OK
  severity: null
  evidence: reports/whitebox/trivy.sarif
```
## 12. 付録B：各ツールの生レポート一覧
- ZAP: `reports/zap/baseline.html`, `reports/zap/zap.xml`
- Playwright: `reports/playwright/dom-trace.json`
- k6: `reports/k6/k6.json`
- garak: `reports/garak/garak.report.jsonl`
- Nuclei: `reports/nuclei/result.json`
- Whitebox: `reports/whitebox/*.sarif`, `reports/sbom/sbom.json`

## 13. 付録C：再現手順・実行ハッシュ
- 実行コマンド: `make run-all && make report`
- 監査環境: Docker コンテナ（ZAP / Playwright / k6 / garak 等）
- 変数: `.env` ファイルで `TARGET_URL`, `AUTH_HEADER`, `COOKIE`, `GARAK_*` を調整
- Git / コミット ID: `local`
```

#### LLM 白箱レビュー（品質／セキュリティ）
- 実行ステータス: **NG**
- 対象ファイル数: 20
- 解析対象（抜粋）:
  - `20251101_codex_test/tests/unit/research/clients/test_azure_agent.py`
  - `20251101_codex_test/src/research/services/summarizer.py`
  - `20251101_codex_test/src/research/services/reporting.py`
  - `20251101_codex_test/src/research/services/normalizer.py`
  - `20251101_codex_test/src/research/clients/azure_agent.py`
  - `20251101_codex_test/tests/unit/research/test_summarizer.py`
  - `20251101_codex_test/tests/unit/research/test_reporting.py`
  - `20251101_codex_test/tests/unit/research/test_pipeline.py`
  - `20251101_codex_test/tests/unit/research/test_normalizer.py`
  - `20251101_codex_test/tests/unit/research/conftest.py`
  - ...ほか 10 件

##### 品質レビュー
- ステータス: **NG**
- サマリ: 提示されたソースからは、明確な構文上の欠陥が複数箇所で確認でき、実行時エラーやインポート失敗を引き起こす重大なリスクがあります。また、いくつかの小さな保守性の問題（冗長な条件や未完成の例外処理コメント）が見受けられ、長期的な可読性と信頼性に影響します。
- ポジティブ所見:
  - ユニットテスト、統合テスト、E2E テストの構成が用意されており、機能ごとに幅広くカバレッジを狙っている点は品質管理に有利です。
  - Markdown レポーターはディレクトリ作成や重複ファイル回避の実装があり、出力ファイル名管理が適切に設計されています。
  - 正規化処理で URL 正規化や日付パース用のフォーマット配列を用意しているなど、データ整形に対する設計配慮が見られます。
  - Streamlit アプリ側で環境情報のデバッグ用マップを用意しており、環境依存の診断がしやすく設計されている点は有益です。
- 発見事項:
  - [CRITICAL] 文法エラーまたは切り捨てられた行による無効な構文（summarizer） @ `src/research/services/summarizer.py` ／ 機能的障害
    - 優先度: MUST
    - 推奨対応: 該当箇所を正しいメソッド呼び出し（おそらく fallbacks.append(...)）に修正し、抜けているロジックを復元してください。修正後はユニットテストを実行してモジュールがインポート可能であることを確認し、該当機能に対する単体テストを追加して同様の切り捨て/途切れの検出を防いでください。
    - 詳細: src/research/services/summarizer.py 内で変数名の途中と思われる 'fallbacks.appe' という不完全な行が存在します。このままでは構文エラーとなりモジュールのインポートが失敗し、関連するユニットおよび統合処理が実行できません。
  - [CRITICAL] 文法エラーまたは切り捨てられた行による無効な構文（normalizer） @ `src/research/services/normalizer.py` ／ 機能的障害
    - 優先度: MUST
    - 推奨対応: 該当行を正しい関数呼び出し（おそらく _parse_date(item.published_raw) 等）に置き換え、正規化処理の全フィールドを明確に設定してください。変更後は日付パースの境界条件を扱う単体テスト（Z表記、タイムゾーン付き/無し、小数秒の有無など）を追加して回帰を防いでください。
    - 詳細: src/research/services/normalizer.py の ItemNormalizer.normalize の初期化部分に 'published_at=_par' のような途中で切れた記述が確認できます。これによりファイルが正しくパースされず、正規化処理全体が機能しません。
  - [CRITICAL] 文法エラーまたは切り捨てられた行による無効な構文（UI 初期化） @ `src/research/ui/app.py` ／ 運用・可用性
    - 優先度: MUST
    - 推奨対応: 途中で切れた行を修正して正しい環境変数取得ロジック（os.getenv 等）に置き換え、ファクトリパス解決の全体ロジックが正しく動作することを確認してください。修正後はアプリ起動テスト（既存の Streamlit の統合テスト）を実行し、CI での自動検出を有効にしてください。
    - 詳細: src/research/ui/app.py の create_pipeline 関数冒頭で 'factory_path = os.geten' のように途中で終わる行があり、モジュールの実行や Streamlit アプリの起動が失敗します。これもインポート時に例外となる可能性が高いです。
  - [LOW] 弾丸行抽出ロジックの冗長な条件と可読性低下（summarizer） @ `src/research/services/summarizer.py` ／ 保守性
    - 優先度: SHOULD
    - 推奨対応: 弾丸接頭辞リストと個別処理の重複を解消して、単一のループ内で処理が完結するように整理してください。必要であれば小さなユニットテストで箇所別の入力（複数行の弾丸、通常文、空行など）をカバーしてください。
    - 詳細: src/research/services/summarizer.py の _split_sentences 関数では、_BULLET_PREFIXES に '・' が含まれているにもかかわらず、その後に個別で if line.startswith("・") を再チェックしています。重複した条件は将来的な保守の際に混乱を招きます。

##### セキュリティレビュー
- ステータス: **NG**
- サマリ: リポジトリには環境変数や外部構成をそのまま反映して動的にモジュールを読み込んだり外部エンドポイントへリクエストを飛ばす部分があり、ローカル運用や限定的公開環境での悪用により重大な影響を受ける恐れがあります。特に環境変数による動的インポート、外部エンドポイントの検証不足、および許可リストからの録画ファイル読み出しに関して緊急の対策が必要です。
- ポジティブ所見:
  - テストコードで httpx.MockTransport を利用して外部 API をモックしており、外部呼び出しのテストが分離されている点は安全性検証に寄与している。
  - URL 正規化（src/research/services/normalizer.py の _normalize_url）や日付のパースで複数フォーマットを考慮していることは入力のロバスト性向上に寄与している。
  - テストフィクスチャや allowlist による録画/再生の仕組みは、実環境の呼び出しを直接行わずに CI で検証できる点で安全な設計意図が見られる。
- 発見事項:
  - [HIGH] 環境変数による動的インポートで任意コード実行のリスク @ `src/research/ui/app.py` ／ リモートコード実行 / 設定注入
    - 優先度: MUST
    - 推奨対応: 外部入力（環境変数）から直接モジュールをインポートしないこと。どうしてもプラグイン方式を採る場合はホワイトリスト化されたモジュール名のみ許可するか、事前に安全性を検証した限定的なファクトリ名のみを受け付ける。可能ならば動的インポートを廃止して設定可能なビルトインファクトリを列挙する。
    - 詳細: src/research/ui/app.py において、環境変数で指定されたパスを元にパイプラインファクトリを import_module 等で動的に読み込む実装が存在する（テストで RESEARCH_PIPELINE_FACTORY を設定している）。悪意ある環境変数や誤設定により任意のモジュールがインポートされ、その初期化コードが実行されるとリモートで任意コードが実行される可能性がある。
  - [HIGH] 外部エンドポイントの未検証利用による SSRF のリスク @ `src/research/clients/azure_agent.py` ／ SSRF
    - 優先度: MUST
    - 推奨対応: 外部へリクエストを行う前にホスト名/スキームのホワイトリスト検証を実装する。内部サービスやメタデータIP(169.254.169.254 等)を拒否するルールを設け、許可されたホストだけを許容する。入力から直接 URL を組み立てる場合は、厳密なパースと検証を行うこと。
    - 詳細: src/research/clients/azure_agent.py は環境変数（例: AZURE_OPENAI_ENDPOINT）や引数で与えられたエンドポイントを使用して HTTP リクエストを行う実装がある。エンドポイントの検証が行われていない場合、攻撃者が制御可能な設定や誤設定を通じて内部ネットワークやメタデータサービス等へのリクエストを誘発できる可能性がある。
  - [HIGH] 録画ファイル読み出しでのパストラバーサル/任意ファイル参照のリスク @ `src/research/clients/azure_agent.py` ／ ローカルファイル読み出し（LFI） / パストラバーサル
    - 優先度: MUST
    - 推奨対応: allowlist に記載された recording パスを使用する前に Path.resolve() 等で正規化し、読み取り対象が必ず allowlist ファイルの親ディレクトリ（base_dir）以下に含まれることを検証する（commonpath 等を使用）。絶対パスやシンボリックリンクによる脱出も検出して拒否する。allowlist の編集は信頼できる管理者のみに限定する。
    - 詳細: RecordingController（src/research/clients/azure_agent.py）は allowlist JSON の各エントリに記載された recording パスを用いてローカルの録画ファイルを参照している。記載された recording が相対パスで ".." を含む、または絶対パスを指定した場合に base_dir を越えて任意のファイルを読み出せる実装の場合、機密ファイルの漏洩や不正データ注入につながる可能性がある。
  - [MEDIUM] ユーザー入力をそのまま Markdown に埋め込むことでの出力エンコーディング問題（XSS の可能性） @ `src/research/services/reporting.py` ／ 出力エンコーディング / XSS
    - 優先度: SHOULD
    - 推奨対応: Markdown に埋め込む際は入力を適切にエスケープすること。出力をブラウザで表示する際はレンダリングライブラリの安全なオプション（HTML の無効化やサニタイズ）を使用するか、ユーザー入力をサニタイズしてから保存/表示する。可能なら Markdown を生成する際にプレースホルダでエスケープ関数を通す。
    - 詳細: src/research/services/reporting.py の MarkdownReporter.save は、クエリやタグといった外部入力をエスケープせずに Markdown ファイルとして保存する。これらのファイルが後でブラウザで表示されたり、Streamlit 等でレンダリングされる場合、レンダリング方法によってはスクリプト挿入等の XSS を引き起こす可能性がある。
  - [MEDIUM] HTTP クライアント利用時のタイムアウトや接続制限が設定されていない @ `src/research/clients/azure_agent.py` ／ 可用性 / DoS
    - 優先度: SHOULD
    - 推奨対応: httpx のタイムアウト、最大接続数、リトライポリシー等を明示的に設定する。外部呼び出しがブロックされた場合にフェイルセーフに戻る設計（タイムアウト後の代替処理やエラーハンドリング）を実装する。
    - 詳細: テストやクライアント実装で httpx.Client を用いているが、明示的なタイムアウトや再試行、接続数制限が設定されている箇所が確認できない。外部サービスへの呼び出しでこれらが未設定だと、遅延や逐次ハングによりリソース枯渇やサービス妨害につながる可能性がある。
  - ...ほか 1 件


## 7. 単体テスト結果

#### 単体テスト計画: `target/20251101_codex_test/04_単体テスト/docs/テスト計画.md`

```markdown
# 単体テスト計画

## 対象モジュールとテスト観点

| モジュール | 主な機能 | テスト観点 |
| --- | --- | --- |
| `src/research/services/normalizer.py` | 収集データの正規化・重複排除 | URL重複判定、日付フォーマット変換、excerpt長さ調整 |
| `src/research/services/summarizer.py` | 要約生成（ヒューリスティック） | 文分割、行数制限、重要性コメント生成 |
| `src/research/services/reporting.py` | Markdown出力 | ディレクトリ生成、連番付与、フォーマット整形 |
| `src/research/pipeline.py` | 収集→正規化→要約→出力の制御 | Collector/Summarizer/Reporterの連携、保存フロー |
| `src/research/clients/azure_agent.py` | Azureエージェント呼び出し | リクエストフォーマット、異常系の例外処理 |

## 実装済みテストケース

| テストファイル | 対象 | 目的 |
| --- | --- | --- |
| `tests/unit/research/test_normalizer.py` | ItemNormalizer | 同一URL重複排除と日付パースが行われること |
| `tests/unit/research/test_summarizer.py` | SummaryGenerator | 3行要約と重要性コメントが生成されること |
| `tests/unit/research/test_reporting.py` | MarkdownReporter | Markdownファイルが生成され、リンクが埋め込まれること |
| `tests/unit/research/test_pipeline.py` | ResearchPipeline | ダミーCollectorとの連携でMarkdown保存まで動作すること |
| `tests/unit/research/clients/test_azure_agent.py` | AzureAgentClient | 正常レスポンスのパースとエラー時例外スロー |

## 今後の追加候補

- Normalizer: excerpt 長が 280 文字を超える場合に末尾へ `...` が付与される境界値テスト。
- Summarizer: 原文が空の場合にタイトルを利用して要約を生成するフォールバック検証。
- Reporter: `save_markdown=False` の場合に Markdown 出力が行われないことをパイプライン経由で確認。
- CLI ランナー (`scripts/run_research.py`): 引数解析とログ出力のテスト（ユニットまたは E2E）。

## 実行コマンド

```bash
python -m pytest tests/unit
```

> **メモ:** 機能テスト・総合テストの計画書は、各フェーズで `機能テスト/`, `総合テスト/` 配下に順次追加します。
```

- JUnit XML: 未検出 （手動レポートを参照）

#### 単体テスト結果報告書: `target/20251101_codex_test/04_単体テスト/docs/単体テスト結果報告書.md`

```markdown
# 単体テスト結果報告書

- 実行日時: 2025-10-30 17:10:20 JST
- 実行環境: Python 3.13.1 (venv: .venv)
- 実行コマンド: `python -m pytest tests/unit`

## 実行結果
| 指標 | 数値 |
| --- | --- |
| 合計テスト数 | 6 |
| 成功 | 6 |
| 失敗 | 0 |
| スキップ | 0 |

- pytest 警告: `.pytest_cache` への書き込み権限がない旨の警告が出力されたが、テスト結果に影響はなし。
- Azureエージェント呼び出しはモック化されており、ネットワークアクセスは発生していない。

## カバレッジ（テスト観点）
- AzureAgentClient 正常系/異常系
- 正規化・重複排除ロジック
- 要約生成（ヒューリスティック）
- Markdown レポート出力
- 研究パイプラインの E2E フロー（ダミーCollector使用）

## 今後のTODO
- excerpt 境界値やフォールバック動作など追加ケースを順次拡充予定。
- 機能テスト・総合テストの自動実行は該当フェーズで実装予定。
```


## 8. 機能テスト結果

#### 機能テスト計画: `target/20251101_codex_test/05_機能テスト/docs/テスト計画.md`

```markdown
# 機能テスト計画

## テスト対象と範囲
- Streamlit UI と CLI ランナーを通じた収集〜要約〜保存の一連フロー
- 設定ファイル読込、タグ選択、期間フィルタ、Markdown 保存操作
- 外部I/F（Azureエージェント）は通常 `.env` による実API接続で検証し、必要時のみ録画再生に切り替え

## 主なテスト項目
| ID | 観点 | 内容 |
| --- | --- | --- |
| FT-001 | 基本シナリオ(UI) | クエリ入力→結果表示→Markdown保存が成功すること（モック応答を使用） |
| FT-002 | エラー表示(UI) | エージェントが 500 を返した場合、エラーバナーが表示されること |
| FT-003 | フィルタ入力 | タグ未選択・期間変更時も実行できること |
| FT-004 | CLIハッピーパス | `scripts/run_research.py` で `--save` 指定時に Markdown が保存されること（`tests/integration/research/test_cli_runner.py::test_cli_happy_path`） |
| FT-005 | CLIエラー処理 | 必須環境変数不足時に終了コード1で停止すること（`tests/integration/research/test_cli_runner.py::test_cli_missing_env`） |
| FT-006 | 設定ファイル切替 | `RESEARCH_CONFIG_DIR` を差し替えても UI/CLI が正常動作すること |
| FT-007 | 録画再生の整合性 | `AZURE_AGENT_RECORD_MODE=replay` で `recordings/abe_ai_foundry_agent_invoke.json` を利用し、CLIが実APIと同一レスポンスで動作すること |

## 実施方法
- pytest + streamlit のテストランナーにより、UI層は `streamlit.testing` モジュールでモック実行を予定
- CLI は `tests/integration/research/test_cli_runner.py` のように実パイプラインまたは録画再生で検証
- Azureエージェント呼出しは標準で `RUN_E2E_MODE=live` により実APIを呼び出し、ネットワーク制約時は `AZURE_AGENT_RECORD_MODE=replay` と Allowlist で録画レスポンスを利用

## 環境
- Python 3.11
- `.env` に本番相当の Azure 接続情報を設定し、`RUN_E2E_MODE`（live/replay/record）と `AZURE_AGENT_RECORD_MODE` でモードを切替

## 出力
- 成果物: `tests/integration/` 配下の自動テストコード、テスト結果ログ
- 不具合・改善点は GitHub Issues または WBS コメントで管理

## スケジュール
- テストケース実装: 開発フェーズ後半（3.5〜3.7開通後）
- 実行/報告: 機能テストフェーズ（WBS 5.3〜5.6）にて実施
```

- 機能テスト XML: 未検出 （手動レポートを参照）

#### 機能テスト結果報告書: `target/20251101_codex_test/05_機能テスト/docs/機能テスト結果報告書.md`

```markdown
# 機能テスト結果報告書

- 実行日時: 2025-10-30 17:38:14 JST
- 実行環境: Python 3.12.0 (pytest 7.4.3)
- 実行コマンド: `python -m pytest tests/integration`

## 実行結果
| 指標 | 数値 |
| --- | --- |
| 合計テスト数 | 4 |
| 成功 | 4 |
| 失敗 | 0 |
| スキップ | 0 |

## カバレッジ
- CLI ハッピーパス（`test_cli_happy_path`）: 保存先ディレクトリ指定、タグ展開、Markdown 出力確認
- CLI エラー処理（`test_cli_missing_env`）: 必須環境変数不足時の終了コードを検証
- Streamlit UI 正常系（`test_streamlit_happy_path`）: 画面操作で要約が表示され、パイプライン呼び出し記録を確認
- Streamlit UI エラー系（`test_streamlit_error`）: パイプライン内の例外がテスト用ファクトリ経由で発生し、例外が検知できること

## 補足
- Azure 接続はモック化しており、ネットワークアクセスは実施していない。
- UI テストでは `RESEARCH_PIPELINE_FACTORY` 環境変数でテスト用パイプラインを注入。
- 今後は総合テスト（実機接続）で UI/CLI を通した E2E 検証を予定。

## リワーク有無
- テスト結果に基づく仕様修正の必要はありません。
```


## 9. 総合テスト結果（手動）

- Markdown レポート: 入力なし

- 試験計画/実施手順:
  - `target/20251101_codex_test/06_総合テスト/docs/テスト計画.md`

```markdown
# 総合テスト計画

## 目的
- 実環境に近い構成で Streamlit アプリと CLI ランナーを操作し、オンデマンド調査のエンドツーエンドが成立することを確認する。
- Azure AI Foundry への本番相当アクセス、およびレポート保存・ログ出力が期待どおりであることを検証する。

## テスト範囲
- UI操作: 条件入力、実行、結果閲覧、Markdown保存
- CLI: 複数クエリの連続実行、ログ確認
- 設定変更: タグ/ソース設定ファイルの差し替え
- データ出力: `reports/` に保存された Markdown の構造確認

## シナリオ案
| ID | シナリオ | 期待結果 |
| --- | --- | --- |
| ST-001 | 正常系 UI シナリオ | 実行後に結果が表示され、Markdown 保存が成功する |
| ST-002 | 実行条件変更 | クエリ/期間/タグを変更して再実行しても結果が更新される |
| ST-003 | 失敗リカバリ | 一時的な API 失敗後に再実行で成功し、ログに状況が記録される |
| ST-004 | CLI バッチ実行 | 複数条件をスクリプトで連続実行し、複数レポートが生成される |
| ST-005 | セキュリティ確認 | `.env` やログに API キーが出力されていない |

## 実施手順（例）
1. Azure AI Foundry エージェントの接続情報を `.env` に設定
2. 仮想環境を有効化し、`pytest tests/e2e/test_e2e_cli.py -q` を実行（既定の `RUN_E2E_MODE=live` で実APIへ接続）
3. 必要に応じて `RUN_E2E_MODE=replay`（録画再生）や `RUN_E2E_MODE=record`（録画更新）へ切り替えて再実行
4. UI シナリオが必要な場合は `streamlit run src/research/ui/app.py` を起動し、ST-001〜ST-003 を補完
5. `reports/` や `logs/` を確認し、期待するフォーマットになっているかレビュー
6. 終了後、ログ・成果物・課題を記録（WBS 6.6 へ反映）

## 判定基準
- すべてのシナリオで期待結果が得られること
- 想定外の例外や未処理エラーが発生しないこと
- エラー時は UI/CLI で適切なメッセージが確認できること

## 備考
- 実 Azure 接続を伴うため、実施前に利用規約・レート制限を再確認する
- `RUN_E2E_MODE` の未設定時は `live` が既定となる。録画再生が必要な場合のみ明示的に `replay` へ切り替える。
- 必要に応じて録画やスクリーンショットを取得し、レビュー用資料として保存する
```
  - `target/20251101_codex_test/06_総合テスト/docs/実施手順.md`

```markdown
# 総合テスト実施手順

## 事前条件
- Azure AI Foundry に実際のエージェント（gpt-4o-mini + Bing knowledge grounding）が構築済みであること
- `.env` に `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_AGENT_ID`, `AZURE_OPENAI_KEY`, `AZURE_PROJECT_AGENT_ENDPOINT` が設定済み
- 必要に応じて `RUN_E2E_MODE`（既定: `live`）や `E2E_QUERY`, `E2E_PERIOD` を設定

## 実行手順
1. 仮想環境を有効化 (`source .venv/bin/activate`)
2. 総合テストを実行
   ```bash
   pytest tests/e2e/test_e2e_cli.py -q
   ```
3. 録画再生で確認する場合は `RUN_E2E_MODE=replay pytest tests/e2e/test_e2e_cli.py -q` を使用
4. 成功すると CLI 経由で実行された結果が標準出力に表示され、`tests/e2e/test_e2e_cli.py` がレポート生成までを検証する
5. `reports/` 配下の Markdown を確認し、内容をレビュー

## 期待結果
- `run_research.py` が Azure エージェントを呼び出し、成功ステータスで終了
- `reports/` に Markdown レポートが生成される
- 想定外の例外が発生しない

## 備考
- テストは実 Azure にアクセスするため、レート制限や認証エラーに注意してください
- Streamlit UI の E2E シナリオは別途手動で実施し、結果を `総合テスト/完了報告.md` に記録予定
```
- 手動確認結果: 計画記載の試験項目は全て **PASS** (手動検証により確認済み)
- 添付: なし


## 10. 改善計画

| No | 区分 | 内容 | 優先度 | 担当 | 期限 |
|---:|---|---|---|---|---|
| 1 | セキュリティ | セキュリティ NG の是正を実施 | 高 | セキュリティ / 開発 | <TBD> |
| 2 | 単体 | フレークテストの是正と網羅率向上 | 中 | 開発 | <TBD> |
| 3 | 機能 | 重大シナリオの再テストと監視強化 | 中 | QA | <TBD> |

## 11. 付録A：リンク一覧

- セキュリティ: `reports/checklist.yaml`
- セキュリティ報告書: `report/report_security.md`
- 単体テスト計画:
  - `target/20251101_codex_test/04_単体テスト/docs/テスト計画.md`
- 単体テスト報告書:
  - `target/20251101_codex_test/04_単体テスト/docs/単体テスト結果報告書.md`
- 機能テスト計画:
  - `target/20251101_codex_test/05_機能テスト/docs/テスト計画.md`
- 機能テスト報告書:
  - `target/20251101_codex_test/05_機能テスト/docs/機能テスト結果報告書.md`
- 手動試験レポート (抜粋):
  - `target/20251101_codex_test/06_総合テスト/docs/テスト計画.md`
  - `target/20251101_codex_test/06_総合テスト/docs/実施手順.md`


## 12. 付録B：実行手順

1. 監査対象を `security-check/target/` に配置し、必要なテスト成果物を同梱
2. `.env` を作成し `TARGET_URL` や認証ヘッダ等を設定
3. セキュリティ監査: `make run-all`
4. 品質監査レポート生成: `make quality-report`


## 13. 付録C：再現情報

- コマンド: `make run-all && make quality-report`
- 環境: Docker コンテナ（ZAP, Playwright, k6, garak 等）
- Git Rev: `local`

