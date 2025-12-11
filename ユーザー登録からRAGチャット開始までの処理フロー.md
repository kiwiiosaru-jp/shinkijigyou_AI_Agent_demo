# ユーザー登録からRAGチャット開始までの処理フロー

**作成日**: 2025-12-11
**対象システム**: 全社共通LLMゲートウェイ（マルチテナント RAG システム）
**想定規模**: 8,000名 × 100ユースケース

---

## 1. システム管理者が部門管理者を登録

| 項目 | 内容 |
|------|------|
| **ステップ** | システム管理者による部門管理者の登録 |
| **実行者** | システム管理者（全社IT部門） |
| **トリガー** | 新規ユースケース開始時 / 管理者変更時 |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 1.1 | 管理画面でメールアドレス入力<br>`tanaka@isuzu.co.jp` | - | - | ✅ **開発必要** | **管理画面UI（React）**<br>- メールアドレス入力フォーム<br>- ユースケース選択UI<br>- 登録ボタン |
| 1.2 | 管理画面 → AWS SDK 呼び出し<br>`AdminAddUserToGroup` API | **Cognito Identity Pool**<br>（一時認証情報取得） | - | ✅ **開発必要** | **フロントエンド実装**<br>- Cognito Identity Pool からの一時認証情報取得<br>- AWS SDK クライアント実装 |
| 1.3 | Cognito でユーザー存在確認<br>`AdminGetUser` | **Cognito User Pool** | User Pool 内部DB | ❌ AWS標準API | - |
| 1.4 | ユーザーが未ログイン（存在しない）の場合 | **DynamoDB**<br>（オプション） | `PendingAdminAssignments` テーブル<br>- PK: Email<br>- Attr: Groups[], CreatedAt | ✅ **開発必要** | **DynamoDB テーブル設計**<br>**Lambda 実装**<br>- 保留レコード作成ロジック<br>- Post Authentication Trigger 実装 |
| 1.5 | グループに追加<br>`AdminAddUserToGroup`<br>GroupName: `Admin-RAG` | **Cognito User Pool** | User Pool → Groups → Members<br>`tanaka@isuzu.co.jp` が `Admin-RAG` に所属 | ❌ AWS標準API | - |
| 1.6 | IAM Role 自動適用 | **Cognito Groups**<br>**IAM** | Group に紐づく RoleArn<br>`arn:aws:iam::xxx:role/AdminRole` | ✅ **開発必要** | **CDK実装**<br>- Cognito Groups 定義<br>- IAM Role 作成<br>- Group-Role マッピング |

### データ保存状態（このステップ後）

```json
[Cognito User Pool]
├─ Users: (tanaka はまだ未ログインの場合、空)
└─ Groups:
    └─ Admin-RAG
        ├─ RoleArn: arn:aws:iam::123456789012:role/AdminRole
        ├─ Precedence: 1
        └─ Members: []  // 初回ログイン後に追加される

[DynamoDB: PendingAdminAssignments]  ← オプション
├─ Email: "tanaka@isuzu.co.jp"
├─ Groups: ["Admin-RAG"]
└─ CreatedAt: "2025-12-11T10:00:00Z"
```

---

## 2. 部門管理者が一般ユーザーを登録

| 項目 | 内容 |
|------|------|
| **ステップ** | 部門管理者による一般ユーザーの登録 |
| **実行者** | 部門管理者（田中さん: `tanaka@isuzu.co.jp`） |
| **前提条件** | 田中さんが既に `Admin-RAG` グループに所属済み |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 2.1 | 田中さんがログイン<br>Entra ID SAML認証 | **Cognito User Pool**<br>**Entra ID (SAML IdP)** | User Pool → Users<br>`tanaka@isuzu.co.jp` 作成 | ❌ AWS標準機能 | - |
| 2.2 | Post Authentication Trigger 実行 | **Lambda**<br>**Cognito Trigger** | - | ✅ **開発必要** | **Lambda実装**<br>- DynamoDB から保留レコード取得<br>- `AdminAddUserToGroup` 実行<br>- DynamoDB レコード削除 |
| 2.3 | JWT 発行<br>`cognito:groups: ["Admin-RAG"]` 含む | **Cognito User Pool** | - | ❌ AWS標準機能 | - |
| 2.4 | 管理画面にアクセス<br>JWT → Identity Pool → 一時認証情報 | **Cognito Identity Pool** | - | ✅ **開発必要** | **フロントエンド実装**<br>- JWT から一時認証情報取得<br>- IAM Role 適用確認 |
| 2.5 | ユーザー一覧表示<br>`AdminListUsersInGroup`<br>GroupName: `User-RAG` | **Cognito User Pool** | User Pool → Groups → Members | ✅ **開発必要** | **管理画面UI（React）**<br>- ユーザー一覧表示<br>- ページネーション実装 |
| 2.6 | 新規ユーザー追加<br>`sato@isuzu.co.jp` を `User-RAG` に追加 | **Cognito User Pool**<br>or<br>**DynamoDB** | User Pool → Groups → User-RAG<br>or<br>DynamoDB: PendingUserAssignments | ✅ **開発必要** | **管理画面UI + API実装**<br>- メールアドレス入力フォーム<br>- `AdminAddUserToGroup` 呼び出し<br>- or DynamoDB 書き込み |

### データ保存状態（このステップ後）

```json
[Cognito User Pool]
├─ Users:
│   └─ tanaka@isuzu.co.jp
│       ├─ Attributes: { email: "tanaka@...", email_verified: true }
│       └─ Groups: ["Admin-RAG"]
└─ Groups:
    ├─ Admin-RAG
    │   └─ Members: ["tanaka@isuzu.co.jp"]
    └─ User-RAG
        └─ Members: []  // sato はまだ未ログイン

[DynamoDB: PendingUserAssignments]  ← オプション
├─ Email: "sato@isuzu.co.jp"
├─ Groups: ["User-RAG"]
└─ CreatedBy: "tanaka@isuzu.co.jp"
```

---

## 3. 一般ユーザーが初回ログイン

| 項目 | 内容 |
|------|------|
| **ステップ** | 一般ユーザーの初回ログイン |
| **実行者** | 一般ユーザー（佐藤さん: `sato@isuzu.co.jp`） |
| **前提条件** | 部門管理者が `sato@isuzu.co.jp` を `User-RAG` に登録済み |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 3.1 | ブラウザでアプリURLにアクセス<br>`https://llm-gateway.isuzu.co.jp` | **CloudFront**<br>**ALB**<br>**ECS (Fargate)** | - | ✅ **開発必要** | **フロントエンド（React SPA）**<br>- ログインボタン<br>- Cognito Hosted UI へリダイレクト |
| 3.2 | Cognito Hosted UI 表示<br>"Entra ID でログイン" ボタン | **Cognito User Pool**<br>**Cognito Hosted UI** | - | ❌ AWS標準UI | - |
| 3.3 | Entra ID にリダイレクト<br>SAML認証実行 | **Entra ID (Microsoft)**<br>**SAML 2.0** | - | ❌ 顧客側設定 | **設定作業のみ**<br>- Entra ID → Cognito SAML設定<br>- メタデータXML交換 |
| 3.4 | SAML Assertion 受信<br>ユーザー情報抽出 | **Cognito User Pool** | User Pool → Users<br>`sato@isuzu.co.jp` 作成<br>Attributes: { email, email_verified, identities } | ❌ AWS標準機能 | - |
| 3.5 | Post Authentication Trigger 実行<br>DynamoDB から保留グループ取得 | **Lambda**<br>**DynamoDB** | PendingUserAssignments テーブル | ✅ **開発必要** | **Lambda実装**（既に実装済み）<br>- 保留レコード確認<br>- グループ自動追加 |
| 3.6 | `AdminAddUserToGroup` 実行<br>GroupName: `User-RAG` | **Cognito User Pool** | User Pool → Groups → User-RAG<br>Members: ["sato@isuzu.co.jp"] | ❌ AWS標準API | - |
| 3.7 | JWT 発行<br>`cognito:groups: ["User-RAG"]` 含む | **Cognito User Pool** | - | ❌ AWS標準機能 | - |
| 3.8 | Authorization Code 返却<br>コールバックURL へリダイレクト | **Cognito OAuth 2.0** | - | ❌ AWS標準機能 | - |
| 3.9 | フロントエンドで Token 交換<br>ID Token, Access Token 取得 | **Cognito User Pool** | ブラウザ LocalStorage<br>- idToken<br>- accessToken<br>- refreshToken | ✅ **開発必要** | **フロントエンド実装**<br>- OAuth 2.0 Authorization Code Flow<br>- Token 管理<br>- Refresh Token 自動更新 |

### データ保存状態（このステップ後）

```json
[Cognito User Pool]
├─ Users:
│   ├─ tanaka@isuzu.co.jp (Groups: ["Admin-RAG"])
│   └─ sato@isuzu.co.jp (Groups: ["User-RAG"])  ← 追加された
└─ Groups:
    ├─ Admin-RAG
    │   └─ Members: ["tanaka@isuzu.co.jp"]
    └─ User-RAG
        └─ Members: ["sato@isuzu.co.jp"]  ← 追加された

[ブラウザ LocalStorage]
├─ idToken: "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
│   └─ Claims: {
│       "sub": "uuid-123",
│       "email": "sato@isuzu.co.jp",
│       "cognito:groups": ["User-RAG"],  ← 重要
│       "exp": 1702300000
│     }
├─ accessToken: "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
└─ refreshToken: "..."
```

---

## 4. 一般ユーザーがRAGチャット画面を開く

| 項目 | 内容 |
|------|------|
| **ステップ** | RAGチャット画面の表示 |
| **実行者** | 一般ユーザー（佐藤さん） |
| **前提条件** | ログイン完了、JWT 取得済み |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 4.1 | React SPA で RAGチャット画面表示 | **CloudFront**<br>**S3 (静的ホスティング)** or **ECS** | - | ✅ **開発必要** | **フロントエンド（React）**<br>- チャット画面UI<br>- メッセージ履歴表示<br>- 入力欄<br>- 送信ボタン |
| 4.2 | 所属グループ（部門）を画面に表示<br>JWT の `cognito:groups` から取得 | - | - | ✅ **開発必要** | **フロントエンド実装**<br>- JWT デコード<br>- `cognito:groups` 表示<br>- ヘッダーに "部門: RAGChat" 表示 |
| 4.3 | 過去のチャット履歴取得<br>GET `/api/chat/history` | **API Gateway**<br>**Lambda (GetChatHistory)**<br>**DynamoDB** | DynamoDB: ChatHistory テーブル<br>- PK: userId<br>- SK: chatId<br>- Attr: messages[], createdAt | ✅ **開発必要** | **Lambda実装**<br>- JWT 検証<br>- userId 抽出<br>- DynamoDB Query<br>- 部門フィルタ適用 |

### データ保存状態（このステップ後）

```
変更なし（画面表示のみ）
```

---

## 5. 一般ユーザーが質問を送信

| 項目 | 内容 |
|------|------|
| **ステップ** | ユーザーが質問を入力して送信 |
| **実行者** | 一般ユーザー（佐藤さん） |
| **質問例** | "2024年度の売上予測を教えて" |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 5.1 | 質問入力 + 送信ボタンクリック | - | - | ✅ **開発必要** | **フロントエンド実装**<br>- メッセージ入力欄<br>- 送信ボタン<br>- リアルタイム表示 |
| 5.2 | API リクエスト送信<br>POST `/api/chat/stream`<br>Headers: `Authorization: Bearer {JWT}` | **API Gateway** | - | ✅ **開発必要** | **フロントエンド実装**<br>- Fetch API / Axios<br>- JWT 自動付与<br>- Server-Sent Events 受信 |
| 5.3 | API Gateway で JWT 検証<br>Lambda Authorizer 実行 | **API Gateway**<br>**Lambda Authorizer** | - | ✅ **開発必要** | **Lambda Authorizer 実装**<br>- JWT 署名検証<br>- `cognito:groups` 抽出<br>- Context に部門情報追加 |
| 5.4 | Lambda (PredictStream) 実行開始 | **Lambda** | - | ✅ **開発必要** | **Lambda実装**（GenU V1改修）<br>- イベント受信<br>- Context から部門取得 |
| 5.5 | 部門情報の取得<br>`event.requestContext.authorizer.groups` | - | Lambda 実行コンテキスト<br>- groups: ["User-RAG"] | ✅ **開発必要** | **Lambda実装**<br>- Authorizer Context 読み取り<br>- 部門ID 変数化 |

---

## 6. RAG検索（部門フィルタ付き）

| 項目 | 内容 |
|------|------|
| **ステップ** | ベクトル検索でドキュメント取得 |
| **検索対象** | RAGChat 部門のドキュメントのみ |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 6.1 | Bedrock Knowledge Base Retrieve API 呼び出し<br>部門フィルタ付き | **Bedrock Knowledge Base**<br>**Aurora PostgreSQL (pgvector)** | Aurora PostgreSQL<br>`bedrock_kb` テーブル<br>- department = 'RAG' でフィルタ | ✅ **開発必要** | **Lambda実装**<br>- Knowledge Base Retrieve API<br>- 部門フィルタ構築<br>```python<br>filter = {<br>  "equals": {<br>    "key": "department",<br>    "value": "RAG"<br>  }<br>}<br>``` |
| 6.2 | Aurora PostgreSQL でベクトル検索実行<br>`SELECT * FROM bedrock_kb`<br>`WHERE department = 'RAG'`<br>`ORDER BY embedding <=> query_vector`<br>`LIMIT 5` | **Aurora PostgreSQL Serverless v2** | PostgreSQL テーブル<br>- `bedrock_kb` テーブル<br>- `department` 列でフィルタ<br>- `embedding` 列でベクトル検索 | ❌ Bedrock自動実行 | **事前準備（CDK）**<br>- Aurora スキーマ定義<br>- department 列追加<br>- インデックス作成 |
| 6.3 | 検索結果返却（Top 5） | **Bedrock Knowledge Base** | Lambda メモリ<br>- retrievalResults: [...]<br>  - content: "2024年度予測..."<br>  - metadata: { department: "RAG" } | ❌ Bedrock自動実行 | - |

### データ取得例

```json
{
  "retrievalResults": [
    {
      "content": {
        "text": "2024年度の売上予測は前年比15%増の見込みです..."
      },
      "location": {
        "s3Location": {
          "uri": "s3://bucket/docs/RAG/sales_forecast_2024.pdf"
        }
      },
      "metadata": {
        "department": "RAG",
        "documentId": "doc-123",
        "source": "sales_forecast_2024.pdf"
      },
      "score": 0.92
    }
  ]
}
```

---

## 7. LLM回答生成とストリーミング返却

| 項目 | 内容 |
|------|------|
| **ステップ** | 検索結果を元に回答生成 |
| **使用モデル** | Claude 3.5 Sonnet |

### 処理フロー

| No | 処理内容 | 使用するAWSサービス | データ保存場所 | SI開発要否 | 開発内容 |
|----|---------|------------------|-------------|----------|---------|
| 7.1 | プロンプト構築<br>検索結果 + ユーザー質問 | - | Lambda メモリ | ✅ **開発必要** | **Lambda実装**<br>- RAG プロンプトテンプレート<br>- コンテキスト埋め込み |
| 7.2 | Bedrock InvokeModelWithResponseStream<br>Model: `claude-3-5-sonnet-20240620-v1:0` | **Bedrock**<br>**Claude 3.5 Sonnet** | - | ✅ **開発必要** | **Lambda実装**<br>- Bedrock API 呼び出し<br>- ストリーミング処理<br>```python<br>response = bedrock.invoke_model_with_response_stream(<br>  modelId="...",<br>  body=json.dumps({<br>    "messages": [...],<br>    "max_tokens": 2000<br>  })<br>)<br>``` |
| 7.3 | ストリーミングレスポンス受信<br>チャンク単位で受信 | **Bedrock** | - | ❌ Bedrock自動実行 | - |
| 7.4 | Lambda → API Gateway へストリーミング返却<br>Server-Sent Events (SSE) 形式 | **Lambda**<br>**API Gateway** | - | ✅ **開発必要** | **Lambda実装**<br>- SSE 形式変換<br>- チャンク送信<br>```python<br>for chunk in stream:<br>  yield f"data: {json.dumps(chunk)}\\n\\n"<br>``` |
| 7.5 | フロントエンドで逐次表示<br>リアルタイム更新 | - | React State | ✅ **開発必要** | **フロントエンド実装**<br>- SSE 受信<br>- チャンク結合<br>- リアルタイム表示 |
| 7.6 | 回答完了後、チャット履歴保存<br>DynamoDB に保存 | **DynamoDB** | ChatHistory テーブル<br>- PK: userId<br>- SK: chatId<br>- Attr:<br>  - userMessage<br>  - assistantMessage<br>  - department: "RAG"<br>  - timestamp | ✅ **開発必要** | **Lambda実装**<br>- DynamoDB PutItem<br>- 部門情報付与<br>- タイムスタンプ記録 |
| 7.7 | トークン消費量記録<br>部門別カウント | **DynamoDB** or **CloudWatch Logs** | DynamoDB: TokenUsage テーブル<br>- PK: department-YYYY-MM<br>- Attr: totalTokens<br>or<br>CloudWatch Logs<br>- Log Group: /aws/lambda/PredictStream<br>- Metric Filter で集計 | ✅ **開発必要** | **Lambda実装**<br>- トークン数カウント<br>- DynamoDB 更新<br>or<br>- CloudWatch Logs 出力<br>- Metric Filter 定義（CDK） |

### データ保存状態（このステップ後）

```json
[DynamoDB: ChatHistory]
├─ userId: "sato@isuzu.co.jp"
├─ chatId: "chat-20251211-001"
├─ department: "RAG"
├─ messages: [
│   {
│     "role": "user",
│     "content": "2024年度の売上予測を教えて",
│     "timestamp": "2025-12-11T10:30:00Z"
│   },
│   {
│     "role": "assistant",
│     "content": "2024年度の売上予測は前年比15%増の見込みです...",
│     "timestamp": "2025-12-11T10:30:05Z",
│     "sources": ["sales_forecast_2024.pdf"]
│   }
│ ]
└─ createdAt: "2025-12-11T10:30:00Z"

[DynamoDB: TokenUsage]
├─ departmentMonth: "RAG-2025-12"
├─ totalInputTokens: 1500
├─ totalOutputTokens: 800
└─ lastUpdated: "2025-12-11T10:30:05Z"

[CloudWatch Logs]
└─ /aws/lambda/PredictStream
    └─ Log Entry:
        {
          "timestamp": "2025-12-11T10:30:05Z",
          "userId": "sato@isuzu.co.jp",
          "department": "RAG",
          "inputTokens": 1500,
          "outputTokens": 800,
          "latency": 5200
        }
```

---

## 8. SI開発要素のまとめ

### 開発が必要な要素（SI責任範囲）

| カテゴリ | 開発要素 | 使用技術 | 工数見積 |
|---------|---------|---------|---------|
| **管理画面UI** | システム管理者画面<br>- 部門管理者登録<br>- グループ管理<br>- ユーザー一覧 | React, TypeScript, AWS SDK | 20人日 |
| **管理画面UI** | 部門管理者画面<br>- ユーザー登録<br>- ユーザー一覧<br>- 権限管理 | React, TypeScript, AWS SDK | 15人日 |
| **フロントエンド** | RAGチャット画面<br>- メッセージ入力/表示<br>- SSE ストリーミング受信<br>- 認証状態管理 | React, TypeScript, EventSource | 25人日 |
| **認証・認可** | Cognito 設定<br>- User Pool 作成<br>- Groups 定義<br>- IAM Role マッピング<br>- SAML 連携設定 | CDK (TypeScript), Cognito, IAM | 10人日 |
| **Lambda** | Lambda Authorizer<br>- JWT 検証<br>- 部門情報抽出 | Node.js/Python, AWS SDK | 5人日 |
| **Lambda** | PredictStream<br>- RAG検索<br>- 部門フィルタ<br>- Bedrock API 呼び出し<br>- SSE ストリーミング | Python, Bedrock SDK | 15人日 |
| **Lambda** | GetChatHistory<br>- DynamoDB Query<br>- 部門フィルタ | Python, AWS SDK | 3人日 |
| **Lambda** | Post Authentication Trigger<br>- 保留グループ自動付与 | Node.js, Cognito SDK, DynamoDB | 5人日 |
| **データベース** | Aurora PostgreSQL スキーマ<br>- bedrock_kb テーブル<br>- department 列<br>- インデックス | CDK, SQL, pgvector | 5人日 |
| **データベース** | DynamoDB テーブル設計<br>- ChatHistory<br>- TokenUsage<br>- PendingAssignments | CDK, DynamoDB | 3人日 |
| **インフラ** | CDK スタック全体<br>- VPC, ALB, ECS<br>- API Gateway<br>- CloudFront<br>- S3 | CDK (TypeScript) | 20人日 |
| **モニタリング** | CloudWatch Dashboard<br>- メトリクスフィルター<br>- アラーム設定 | CDK, CloudWatch | 10人日 |
| **カスタムパイプライン** | Step Functions パイプライン<br>- Textract 処理<br>- チャンキング<br>- Embeddings<br>- Aurora INSERT | Step Functions, Lambda, Python | 30人日 |

**合計開発工数**: **約 166人日**

---

### AWSネイティブ機能（開発不要）

| カテゴリ | 機能 | 使用サービス |
|---------|-----|------------|
| **認証** | SAML 2.0 認証 | Cognito User Pool + Entra ID |
| **認証** | JWT 発行・検証 | Cognito User Pool |
| **認証** | OAuth 2.0 Authorization Code Flow | Cognito User Pool |
| **認可** | IAM Role 一時認証情報発行 | Cognito Identity Pool |
| **認可** | グループベース権限管理 | Cognito Groups + IAM |
| **ストレージ** | ユーザー・グループ管理 | Cognito User Pool (内部DB) |
| **AI** | ベクトル検索 | Bedrock Knowledge Base + Aurora pgvector |
| **AI** | LLM推論 | Bedrock (Claude 3.5 Sonnet) |
| **AI** | Embeddings 生成 | Bedrock (Titan Embeddings v2) |
| **ネットワーク** | HTTPS 終端 | CloudFront / ALB |
| **ネットワーク** | WAF | AWS WAF |
| **ログ** | アクセスログ | CloudWatch Logs |

---

## 9. データフロー図（全体）

```
[Entra ID (Microsoft)]
    ↓ SAML 2.0
[Cognito User Pool]  ← ユーザー・グループの保存場所
    ├─ Users (8,000名)
    │   └─ sato@isuzu.co.jp
    │       └─ Groups: ["User-RAG"]
    └─ Groups (200個)
        └─ User-RAG
            ├─ RoleArn: arn:aws:iam::xxx:role/UserRole
            └─ Members: [sato@...]
    ↓ JWT 発行
[React SPA] (ブラウザ)
    ├─ LocalStorage: idToken (cognito:groups: ["User-RAG"])
    └─ API呼び出し: Authorization: Bearer {JWT}
    ↓
[API Gateway]
    └─ Lambda Authorizer → JWT検証 + 部門抽出
    ↓
[Lambda: PredictStream]
    ├─ 部門情報: "RAG" (Authorizer Context から)
    └─ Bedrock Knowledge Base Retrieve API
        └─ filter: { equals: { key: "department", value: "RAG" } }
    ↓
[Aurora PostgreSQL (pgvector)]
    └─ bedrock_kb テーブル
        └─ WHERE department = 'RAG'
        └─ ORDER BY embedding <=> query_vector
    ↓ 検索結果
[Lambda: PredictStream]
    ├─ プロンプト構築
    └─ Bedrock InvokeModelWithResponseStream
    ↓
[Claude 3.5 Sonnet]
    └─ ストリーミング回答生成
    ↓ SSE
[React SPA] (ブラウザ)
    └─ リアルタイム表示
    ↓ 完了後
[DynamoDB]
    ├─ ChatHistory テーブル (部門情報付き)
    └─ TokenUsage テーブル (部門別集計)
```

---

## 10. 顧客側の責任範囲

| 項目 | 内容 | 担当 |
|------|------|------|
| **Entra ID 設定** | SAML Identity Provider 設定<br>セキュリティグループ管理 | いすゞ自動車 IT部門 |
| **Entra ID ユーザー招待** | 新規ユーザーを Entra ID グループに招待 | いすゞ自動車 部門管理者 |
| **システム管理者の初期設定** | 最初のシステム管理者アカウント作成 | SI（初期セットアップ時） |
| **運用ポリシー決定** | 部門別クォータ設定<br>利用ガイドライン策定 | いすゞ自動車 IT部門 |
| **監視・アラート対応** | CloudWatch アラート受信<br>エスカレーション判断 | いすゞ自動車 IT部門 + SI運用保守 |

---

## 11. セキュリティ考慮事項

| 項目 | 実装方法 | 使用サービス | 開発要否 |
|------|---------|------------|---------|
| **通信暗号化** | HTTPS (TLS 1.3) | CloudFront, ALB | ❌ AWS標準 |
| **認証** | SAML 2.0 + MFA | Entra ID, Cognito | ❌ 設定のみ |
| **認可** | JWT + IAM Role | Cognito, IAM | ✅ CDK実装 |
| **データ分離** | 部門フィルタ (department 列) | Aurora PostgreSQL | ✅ Lambda実装 |
| **ログ監査** | CloudWatch Logs | CloudWatch | ❌ AWS標準 |
| **暗号化（保存時）** | AES-256 | Aurora, S3, DynamoDB | ❌ AWS標準 |
| **暗号化（転送時）** | TLS 1.3 | CloudFront, ALB | ❌ AWS標準 |
| **WAF** | SQLi, XSS 防御 | AWS WAF | ✅ CDK実装 |

---

## 12. 参考リンク

- [Cognito User Pool Groups](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html)
- [Cognito Identity Pool Role Mapping](https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html)
- [Bedrock Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Aurora PostgreSQL with pgvector](https://aws.amazon.com/about-aws/whats-new/2023/07/amazon-aurora-postgresql-pgvector-vector-storage-similarity-search/)

---

**作成者**: SI開発チーム
**最終更新**: 2025-12-11
**バージョン**: 1.0
