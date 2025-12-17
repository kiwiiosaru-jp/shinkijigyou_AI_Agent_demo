# 保全・修理受付データモデル ER図（Mermaid）

以下は、先ほど提示したテーブル（マスタ／トランザクション）に基づくER図です。  
Mermaid対応のMarkdownビューア（GitHub / VS Code拡張 / Notion等）で表示できます。

```mermaid
erDiagram
    %% =========================
    %% Master tables
    %% =========================
    M_LINE ||--o{ M_EQUIPMENT : "所属"
    M_SYMPTOM_CATEGORY ||--o{ M_SYMPTOM_CATEGORY : "親子(階層)"
    M_SYMPTOM_CATEGORY ||--o{ M_QUESTION_SET : "分類→質問割当"
    M_QUESTION ||--o{ M_QUESTION_SET : "質問→割当"
    M_CAUSE ||--o{ T_TICKET : "主原因"
    M_ACTION ||--o{ T_TICKET : "主処置"

    %% =========================
    %% Transaction tables
    %% =========================
    M_LINE ||--o{ T_TICKET : "ライン"
    M_EQUIPMENT ||--o{ T_TICKET : "設備"
    M_SYMPTOM_CATEGORY ||--o{ T_TICKET : "現象分類"

    T_TICKET ||--o{ T_TICKET_ANSWER : "補足回答"
    M_QUESTION ||--o{ T_TICKET_ANSWER : "質問"

    %% =========================
    %% Entities (fields)
    %% =========================
    M_LINE {
        string line_id PK "ラインID"
        string line_code "ラインコード"
        string line_name "ライン名"
        boolean is_active "有効フラグ"
    }

    M_EQUIPMENT {
        string equipment_id PK "設備ID"
        string equipment_code "設備コード"
        string equipment_name "設備名"
        string line_id FK "所属ラインID"
        string equipment_system "設備系統"
        string equipment_area "設置エリア"
        string equipment_part_master_ref "部位マスタ参照(任意)"
    }

    M_SYMPTOM_CATEGORY {
        string symptom_category_id PK "現象分類ID"
        string parent_category_id FK "親分類ID(自己参照)"
        int category_level "階層レベル"
        string category_code "分類コード"
        string category_name "分類名"
        boolean is_active "有効フラグ"
    }

    M_CAUSE {
        string cause_id PK "原因ID"
        string cause_code "原因コード"
        string cause_name_std "原因標準語"
        string cause_category "原因カテゴリ"
        string applies_system "適用設備系統(任意)"
        boolean is_active "有効フラグ"
    }

    M_ACTION {
        string action_id PK "処置ID"
        string action_code "処置コード"
        string action_name_std "処置標準語"
        string action_category "処置カテゴリ"
        boolean is_active "有効フラグ"
    }

    M_QUESTION {
        string question_id PK "質問ID"
        string question_code "質問コード"
        string question_text "質問文"
        string input_type "入力タイプ(choice/multi/number/text/yesno)"
        string option_set_ref "選択肢参照(任意)"
        string unit "単位(任意)"
        boolean is_active "有効フラグ"
    }

    M_QUESTION_SET {
        string question_set_id PK "質問セット割当ID"
        string symptom_category_id FK "現象分類ID"
        string question_id FK "質問ID"
        boolean required_flg "必須フラグ"
        int display_order "表示順"
        string default_value_rule "初期値ルール(任意)"
    }

    T_TICKET {
        string ticket_id PK "チケットID"
        string ticket_no "チケット番号"
        datetime occurred_at "発生日時(必須)"
        datetime reported_at "受付日時(必須)"
        string line_id FK "ライン(必須)"
        string equipment_id FK "設備(必須)"
        string symptom_raw "現象(自由記述)(必須)"
        string symptom_category_id FK "現象分類(推奨:必須)"
        string timing_code "発生タイミング"
        string frequency_code "発生頻度"
        boolean impact_stop_flg "影響:停止"
        boolean impact_quality_flg "影響:品質"
        boolean impact_safety_flg "影響:安全"
        string reproducibility_code "再現性"
        string recovery_operation_code "復旧操作(一次対応)"
        string location_text "発生箇所(記述)"
        string attachment_ref "添付参照"
        string status_code "ステータス"
        datetime completed_at "完了日時"
        string cause_raw "原因(自由記述)"
        string action_raw "処置(自由記述)"
        string primary_cause_id FK "主原因(確定)"
        string primary_action_id FK "主処置(確定)"
        string data_quality_status "データ品質ステータス"
        string created_by "登録者"
        datetime updated_at "更新日時"
    }

    T_TICKET_ANSWER {
        string ticket_answer_id PK "回答ID"
        string ticket_id FK "チケットID"
        string question_id FK "質問ID"
        string value_choice "回答(選択肢)"
        string value_multi "回答(複数選択)"
        string value_text "回答(テキスト)"
        float value_number "回答(数値)"
        boolean value_bool "回答(Yes/No)"
        string filled_by_type "入力者種別(人/AI提案/自動抽出)"
        datetime filled_at "入力日時"
    }
```

## 補足
- **M_SYMPTOM_CATEGORY** は自己参照で階層（例：機械 → 異音）を表現します。  
- 「カテゴリ別の固定補足質問」は **M_QUESTION_SET**（分類→質問割当）で管理し、回答は **T_TICKET_ANSWER** に縦持ちで保存します。  
- 主原因・主処置は **T_TICKET.primary_cause_id / primary_action_id** で1つずつ確定し、必要なら別途「副要因」「複数処置」用の関連テーブルを追加します。
