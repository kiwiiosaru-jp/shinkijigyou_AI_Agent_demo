# Azure OpenAI 推論費（従量）見積：算出式とプロンプト例一覧（23万件バッチ）

更新日：2026-01-17

この資料は、先に提示した「23万件バッチ処理の推論費概算」を、**算出式（トークン→金額）**と、各バッチで想定する**プロンプト（具体例）**として一覧化したものです。

---

## 1. 見積の前提（今回の数値に使った置き方）

### 1.1 呼び出し回数（件数）
- Batch A（10%サンプル解析）：23,000 回
- Batch B（残り90%の候補判定＋未マッチ抽出）：207,000 回
- Batch C（全件補正＋マッピング＋信頼度返却）：230,000 回
- クラスタ命名（クラスタ単位）：500 回（例：設備・部品・分類などを合わせて **約500クラスタ**を想定）

### 1.2 1回あたりの入出力トークン仮定（安全側にやや多め）

| 区分 | 入力トークン/回 | 出力トークン/回 | 備考 |
|---|---:|---:|---|
| Batch A | 600 | 200 | レコードから用語抽出＋軽い正規化 |
| Batch B | 800 | 120 | Python側で候補（上位5〜10）に絞った上で選択＋未マッチ抽出 |
| Batch C | 900 | 250 | 補正結果（JSON）と信頼度、要確認フラグ等を返すため出力が増える想定 |
| クラスタ命名 | 1000 | 300 | 代表語・頻出語・例文から標準名称・別名・階層を生成 |

---

## 2. 算出式（トークン → 金額）

### 2.1 トークン総量

**入力トークン総量**

```text
TotalInputTokens = Σ(Calls_i × InputTokensPerCall_i)
```

**出力トークン総量**

```text
TotalOutputTokens = Σ(Calls_i × OutputTokensPerCall_i)
```

### 2.2 金額（USD）

```text
CostUSD = (TotalInputTokens / 1,000,000) × InputPricePer1M
        + (TotalOutputTokens / 1,000,000) × OutputPricePer1M
```

**Batch API（利用できる場合）**：50%割引を前提に

```text
CostUSD_BatchAPI = CostUSD × 0.5
```

---

## 3. バッチ別のトークン量と金額（内訳）

- 入力トークン合計：**386.9M** tokens（386,900,000）
- 出力トークン合計：**87.09M** tokens（87,090,000）

### 3.1 トークン内訳（バッチ別）

| バッチ | 回数 | 入力/回 | 入力合計 | 出力/回 | 出力合計 |
|---|---:|---:|---:|---:|---:|
| Batch A | 23,000 | 600 | 13.8M | 200 | 4.6M |
| Batch B | 207,000 | 800 | 165.6M | 120 | 24.84M |
| Batch C | 230,000 | 900 | 207.0M | 250 | 57.5M |
| クラスタ命名 | 500 | 1000 | 0.5M | 300 | 0.15M |

### 3.2 金額（モデル別）

価格前提（USD / 100万トークン）：

| モデル | 入力単価 | 出力単価 |
|---|---:|---:|
| GPT-5 | $1.25 | $10.00 |
| GPT-5-mini | $0.25 | $2.00 |

| モデル | Standard（従量） | Batch API（50%） |
|---|---:|---:|
| GPT-5 | **$1,354.53** | **$677.26** |
| GPT-5-mini | **$270.90** | **$135.45** |

参考：バッチ別の内訳（Standard、USD）

| バッチ | GPT-5 | GPT-5-mini |
|---|---:|---:|
| Batch A（10%サンプル解析） | $63.25 | $12.65 |
| Batch B（90%候補判定＋未マッチ抽出） | $455.40 | $91.08 |
| Batch C（全件補正＋マッピング＋信頼度） | $833.75 | $166.75 |
| クラスタ命名（クラスタ単位） | $2.12 | $0.42 |

---

## 4. プロンプト例（バッチ別：具体例）

> 注：以下は **“バッチ処理で必要となる最小限の指示”**を想定したテンプレートです。
> - **出力は必ずJSONのみ**（説明文を出さない）
> - 候補マスタ（候補ID/正式名称/別名）は **Python側で上位5〜10件に絞って渡す**（入力トークン抑制）
> - 返却スキーマを固定（後段処理が安定し、リトライが容易）

### 4.1 Batch A：サンプル解析（用語抽出＋軽い正規化）

**システムプロンプト（例）**
```text
あなたは保全記録のデータクレンジング担当です。
入力レコードから「設備」「部品」「現象」「原因」「処置」に関する用語を抽出し、表記ゆれを正規化してください。
出力は必ずJSONのみ。日本語。推測は避け、確信が低い場合は要確認にする。
```

**ユーザープロンプト（例：1レコード）**
```json
{
  "record_id": "R00012345",
  "raw": {
    "free_text": "モータ異音。軸受けガタ。ベアリング交換実施。",
    "equipment": "",
    "part": "",
    "phenomenon": "",
    "cause": "",
    "action": ""
  },
  "normalization_rules": {
    "normalize_kana": true,
    "normalize_fullwidth_halfwidth": true,
    "normalize_symbols": true
  },
  "output_schema": {
    "equipment_terms": ["..."],
    "part_terms": ["..."],
    "phenomenon_terms": ["..."],
    "cause_terms": ["..."],
    "action_terms": ["..."],
    "normalized_text": "...",
    "needs_review": true,
    "review_reasons": ["..."]
  }
}
```

**期待する出力（例）**
```json
{
  "equipment_terms": ["モーター"],
  "part_terms": ["軸受", "ベアリング"],
  "phenomenon_terms": ["異音"],
  "cause_terms": ["ガタ"],
  "action_terms": ["ベアリング交換"],
  "normalized_text": "モーター異音。軸受ガタ。ベアリング交換。",
  "needs_review": false,
  "review_reasons": []
}
```

### 4.2 Batch B：候補判定＋未マッチ抽出（マスタ拡張のため）

**システムプロンプト（例）**
```text
あなたは保全記録の名寄せ・標準化担当です。
入力レコードの用語を、与えられた候補マスタ（上位候補）から最適なものにマッピングしてください。
一致しない場合は未マッチ用語として返してください。出力はJSONのみ。
```

**ユーザープロンプト（例）**
```json
{
  "record_id": "R00012345",
  "extracted": {
    "equipment_terms": ["モーター"],
    "part_terms": ["ベアリング"],
    "phenomenon_terms": ["異音"],
    "cause_terms": ["ガタ"],
    "action_terms": ["交換"]
  },
  "candidate_masters": {
    "equipment": [
      {"id": "EQ_001", "name": "モーター", "aliases": ["モータ", "MTR"]},
      {"id": "EQ_014", "name": "ファンモーター", "aliases": ["FAN MTR"]}
    ],
    "part": [
      {"id": "PT_088", "name": "ベアリング", "aliases": ["軸受", "BRG"]},
      {"id": "PT_102", "name": "ギア", "aliases": ["歯車"]}
    ]
  },
  "output_schema": {
    "mapped": {
      "equipment": [{"term": "...", "master_id": "...", "confidence": 0.0}],
      "part": [{"term": "...", "master_id": "...", "confidence": 0.0}]
    },
    "unmatched_terms": {"equipment": ["..."], "part": ["..."], "other": ["..."]},
    "needs_review": true,
    "review_reasons": ["..."]
  }
}
```

**期待する出力（例）**
```json
{
  "mapped": {
    "equipment": [{"term": "モーター", "master_id": "EQ_001", "confidence": 0.93}],
    "part": [{"term": "ベアリング", "master_id": "PT_088", "confidence": 0.95}]
  },
  "unmatched_terms": {"equipment": [], "part": [], "other": ["ガタ"]},
  "needs_review": false,
  "review_reasons": []
}
```

### 4.3 Batch C：全件補正＋マッピング＋信頼度（最終クレンジング返却）

**システムプロンプト（例）**
```text
あなたは保全報告のクレンジングエンジンです。
入力レコードを、確定マスタ（v2）と標準用語辞書に基づいて補正・正規化し、マスタIDへマッピングしてください。
出力はJSONのみ。必ず信頼度（0〜1）と要確認フラグを含め、推測は要確認にする。
```

**ユーザープロンプト（例）**
```json
{
  "record_id": "R00012345",
  "raw": {
    "equipment_text": "モータ",
    "part_text": "BRG",
    "phenomenon_text": "異音",
    "cause_text": "ガタ",
    "action_text": "交換",
    "note": "停止後に再発なし"
  },
  "masters_v2": {
    "equipment": [{"id": "EQ_001", "name": "モーター", "aliases": ["モータ", "MTR"]}],
    "part": [{"id": "PT_088", "name": "ベアリング", "aliases": ["軸受", "BRG"]}]
  },
  "standard_dictionary": {
    "replace": [{"from": "BRG", "to": "ベアリング"}]
  },
  "output_schema": {
    "cleaned": {
      "equipment_id": "...",
      "equipment_name": "...",
      "part_id": "...",
      "part_name": "...",
      "phenomenon": "...",
      "cause": "...",
      "action": "...",
      "note": "..."
    },
    "confidence": {
      "equipment": 0.0,
      "part": 0.0,
      "overall": 0.0
    },
    "needs_review": true,
    "review_reasons": ["..."],
    "unmatched_terms": ["..."]
  }
}
```

**期待する出力（例）**
```json
{
  "cleaned": {
    "equipment_id": "EQ_001",
    "equipment_name": "モーター",
    "part_id": "PT_088",
    "part_name": "ベアリング",
    "phenomenon": "異音",
    "cause": "ガタ",
    "action": "交換",
    "note": "停止後に再発なし"
  },
  "confidence": {"equipment": 0.93, "part": 0.95, "overall": 0.92},
  "needs_review": true,
  "review_reasons": ["原因(ガタ)は標準語辞書に未登録の可能性"],
  "unmatched_terms": ["ガタ"]
}
```

### 4.4 クラスタ命名：標準名称・別名・階層（クラスタ単位）

**システムプロンプト（例）**
```text
あなたは設備・部品・分類のマスタ整備担当です。
クラスタ内の用語（表記ゆれ）を代表する標準名称を提案し、別名リストと分類階層を返してください。
出力はJSONのみ。標準名称は社内で使いやすい自然な日本語にする。
```

**ユーザープロンプト（例）**
```json
{
  "cluster_id": "CL_PT_0123",
  "cluster_type": "part",
  "examples": [
    "BRG", "ベアリング", "軸受", "ベアリング交換", "BRG取替"
  ],
  "context_examples": [
    "モータ異音のためBRG交換",
    "軸受けガタ発生、ベアリング取替"
  ],
  "output_schema": {
    "standard_name": "...",
    "aliases": ["..."],
    "category": {"level1": "...", "level2": "...", "level3": "..."},
    "notes": "..."
  }
}
```

**期待する出力（例）**
```json
{
  "standard_name": "ベアリング",
  "aliases": ["BRG", "軸受", "軸受け"],
  "category": {"level1": "回転機器", "level2": "軸受", "level3": "ベアリング"},
  "notes": "『BRG』は略語として別名に保持し、標準名称は日本語に統一"
}
```

---

## 5. 見積精度を上げるための“実測”ポイント（推奨）

この概算は「1回あたりトークン」を仮定しているため、次の3点を**代表100件**で実測すると、見積精度が大きく上がります。

1. 1レコードあたりの平均文字数（現象／原因／処置／設備／部品の自由記述）
2. 候補マスタを何件渡すか（上位5/10/20）
3. JSON出力の項目数（返す粒度：信頼度の内訳、未マッチ語の詳細など）

実測値に差し替えれば、同じ算出式で金額を即時再計算できます。