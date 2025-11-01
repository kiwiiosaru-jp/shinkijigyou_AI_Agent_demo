# 品質監査結果報告書
**作成日:** 2025-11-02  
**対象 URL:** `http://host.docker.internal:5173`  
**Git Rev:** `local`

---

## 1. エグゼクティブサマリ

- 総合判定: **Fail**
- セキュリティ: NG **1** / HIGH **0**
- 単体: 0件（失敗 0 / 失敗率 0.0%）
- 機能: 0件（失敗 0 / 失敗率 0.0%）
- 総合: ドキュメント 0 件


## 2. 対象・前提

- 対象 URL: `http://host.docker.internal:5173`
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

- チェック件数: 11 / NG 1 / HIGH 0
- 詳細レポート: `report/report.md`
- 主要 NG:
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

- 単体テスト計画: 入力なし
- JUnit XML: 未検出


## 8. 機能テスト結果

- 機能テスト計画: 入力なし
- 機能テスト XML: 未検出


## 9. 総合テスト結果（手動）

- Markdown レポート: 入力なし
- 試験計画/実施手順: 入力なし
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


## 12. 付録B：実行手順

1. 監査対象を `security-check/target/` に配置し、必要なテスト成果物を同梱
2. `.env` を作成し `TARGET_URL` や認証ヘッダ等を設定
3. セキュリティ監査: `make run-all`
4. 品質監査レポート生成: `make quality-report`


## 13. 付録C：再現情報

- コマンド: `make run-all && make quality-report`
- 環境: Docker コンテナ（ZAP, Playwright, k6, garak 等）
- Git Rev: `local`

