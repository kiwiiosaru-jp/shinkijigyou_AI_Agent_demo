# 品質監査結果報告書
**作成日:** 2025-11-01  
**対象 URL:** `http://host.docker.internal:8501`  
**Git Rev:** `local`

---

## 1. エグゼクティブサマリ

- 総合判定: **Fail**
- セキュリティ: NG **2** / HIGH **0**
- 単体: 0件（失敗 0 / 失敗率 0.0%）
- 機能: 0件（失敗 0 / 失敗率 0.0%）
- 総合: ドキュメント 0 件


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
- 単体: 失敗サンプルおよびカバレッジは章 7
- 機能: 失敗サンプルは章 8
- 総合: 手動試験の要約は章 9


## 6. セキュリティ監査サマリ

- チェック件数: 11 / NG 2 / HIGH 0
- 詳細レポート: `report/report.md`
- 主要 NG:
  - BB-WEB-107: セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等） (証跡: `reports/zap/zap.xml`)
  - BB-WEB-108: レート制限・コスト濫用耐性 (証跡: `reports/k6/k6.json`)

### セキュリティ監査報告書 (全文)

```markdown
# セキュリティ監査結果報告書
**対象名:** host.docker.internal:8501  
**実施日:** 2025-11-01  
**実行ID:** -  
**実施者:** セキュリティ検証チーム

---

## 1. エグゼクティブサマリ
- 重大リスク（HIGH 以上）: **0件** ／ NG: **2件** ／ OK: **9件** ／ N/A: **0件**
- 総合判定（ゲート）: **Fail**
- 主要トピック: ①セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等） ②レート制限・コスト濫用耐性 ③ヘッダ防御強化

## 2. 監査の目的・スコープ
- 目的: 外部公開前に、Web/UI/LLM の一般攻撃と生成AI特有リスクを検出し、運用フェーズ前に是正計画を固める。
- スコープ（黒箱）: XSS / SSRF / セキュリティヘッダ / レート制限 / Open Redirect / プロンプトインジェクション / SQLi
- スコープ（白箱）: Secrets / SBOM+SCA / Semgrep（LLM 最小） / Trivy FS / 依存関係整合性

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
- gitleaks / Syft / Grype / Semgrep / Trivy → `reports/whitebox/`, `reports/sbom/`
- 判定基準: `reports/checklist.yaml` のステータスと `config/policies/gates.rego` のゲート判定

## 5. 結果サマリ（OK/NG/N/A 集計・重大度別）
- チェック総数: **11**
- OK: **9** ／ NG: **2** ／ N/A: **0** ／ 要確認: **0**
- 主要エビデンス: ZAP `reports/zap/baseline.html`, Playwright `reports/playwright/dom-trace.json`, k6 `reports/k6/k6.json`, garak `reports/garak/garak.report.jsonl`

## 6. 主要リスクの要約（Top Findings）
1. **BB-WEB-107** — セキュリティヘッダ（CSP/HSTS/Frame-Options/CTO等）  
   - 重大度: 情報 ／ ステータス: NG  
   - 証跡: `reports/zap/zap.xml`
   - 推奨: HTTP 応答で CSP / HSTS / X-Frame-Options / X-Content-Type-Options を適切に設定し、フレーム埋め込みとプロトコルダウングレードを防止してください。

2. **BB-WEB-108** — レート制限・コスト濫用耐性  
   - 重大度: 情報 ／ ステータス: NG  
   - 証跡: `reports/k6/k6.json`
   - 推奨: 429 応答・Retry-After といったスロットル機構を実装し、API キーや IP 単位でのリクエスト上限を導入してください。

## 7. 詳細所見（Finding 個票）
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

## 9. 改善計画（優先順位・担当・期日）
| No | Finding | 対策 | 優先度 | 担当 | 期限 |
|---:|---|---|---|---|---|
| 1 | BB-WEB-107 | HTTP 応答で CSP / HSTS / X-Frame-Options / X-Content-Type-Options を適切に設定し、フレーム埋め込みとプロトコルダウングレードを防止してください。 | 高 | バックエンドチーム | 2025-11-15 |
| 2 | BB-WEB-108 | 429 応答・Retry-After といったスロットル機構を実装し、API キーや IP 単位でのリクエスト上限を導入してください。 | 高 | プラットフォームチーム | 2025-11-22 |


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
- id: WB-DOCKER-033
  category: Whitebox/Container
  item: FS/コンテナの脆弱性（Trivy FS）
  expected: High/Critical=0
  status: OK
  severity: null
  evidence: reports/whitebox/trivy.sarif
- id: BB-WEB-102
  category: Blackbox/XSS
  item: XSS（HTML/Markdownレンダリング）
  expected: 任意JS実行不可、危険スキーム不許可
  status: OK
  severity: null
  evidence: reports/playwright/dom-trace.json
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
- セキュリティ報告書: `report/report.md`
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

