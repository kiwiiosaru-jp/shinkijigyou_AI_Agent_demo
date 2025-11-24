# GenU Phase1 システムアーキテクチャ（2025-11-24 時点）

## 概要
- ユーザーは CloudFront 経由で SPA（GenU UI）にアクセスし、Cognito Hosted UI で認証。
- API Gateway (REST) が単一バックエンドエントリーポイント。Lambda でチャット、RAG（Kendra/Knowledge Base）、画像・動画生成、AgentCore 呼び出し、MCP 連携を提供。
- RAG は 2 系統：Amazon Kendra（S3 -> Kendra Index）と Bedrock Knowledge Base（S3 -> Bedrock KB -> OpenSearch Serverless）。
- モデル呼び出しは Bedrock（マルチリージョンモデル＋東京 inference profile for Nova Canvas/Reel/Sonic）。一部 AgentCore は us-west-2。
- ガードレール (Guardrails for Amazon Bedrock) を Converse ルートに適用。
- 監視は CloudWatch（API/Lambda/DynamoDB）と CloudWatch Dashboard Stack。

## アーキテクチャ（Mermaid）
```mermaid
flowchart LR
  subgraph Client
    U[User Browser<br/>CloudFront d1vmznikinaycd.cloudfront.net]
  end

  subgraph Auth
    COG[Cognito Hosted UI<br/>UserPool ap-northeast-1_G9ZG9hncP]
    IDP[(IdP: future SAML/Google/Entra)]
  end

  subgraph Web
    S3WEB[S3 Web Bucket]
    CF[CloudFront]
  end

  subgraph API
    APIGW[API Gateway /api/*]
    Lpred[Lambda PredictStream<br/>chat/RAG/agent routing]
    Lkb[Lambda KB Retrieve]
    Lrag[Lambdas Rag Query/Retrieve]
    Lmisc[Lambda misc: title/flow/optimize/etc]
    DDB[(DynamoDB chats/stats)]
    S3FILE[(S3 File Bucket)]
  end

  subgraph RAG_KB
    KBDS[S3 KB Datasource Bucket<br/>ragknowledgebasestackdev-...]
    KB[Bedrock Knowledge Base BAIYEVEULE]
    AOSS[OpenSearch Serverless<br/>Collection generative-ai-use-cases-jpdev]
  end

  subgraph RAG_Kendra
    KDS[S3 Kendra Datasource Bucket<br/>generativeaiusecasesstack-ragdatasourcebucket...]
    Kendra[Kendra Index generative-ai-use-cases-indexdev]
  end

  subgraph Models
    BR_APN1["Bedrock ap-northeast-1\nClaude 3.5 Sonnet v1\nNova Pro/Lite/Micro\nNova Canvas/Reel/Sonic (AIP)"]
    BR_USE1["Bedrock us-east-1\nClaude 3.7 Sonnet\nNova Premier"]
    BR_USW2["Bedrock us-west-2\nDeepSeek R1"]
    GR["Guardrail 87xulvzb1pln DRAFT"]
  end

  subgraph AgentCore
    AC_USW2[AgentCore runtimes<br/>Generic/AgentBuilder<br/>us-west-2]
  end

  subgraph MCP
    MCPLambda[Lambda URL MCP endpoint]
    MCPSrv[MCP sample servers<br/>AWS docs/CDK/Nova Canvas/time]
  end

  U --> CF --> S3WEB
  U -->|Auth| COG --> IDP
  CF --> APIGW
  APIGW --> Lpred
  APIGW --> Lrag
  APIGW --> Lkb
  APIGW --> Lmisc
  Lpred --> DDB
  Lpred --> S3FILE
  Lpred -->|Chat/RAG| BR_APN1
  Lpred -->|Cross-region| BR_USE1
  Lpred -->|Cross-region| BR_USW2
  Lpred --> GR
  Lpred --> AC_USW2
  Lpred --> MCPLambda --> MCPSrv

  %% KB path
  Lpred -->|KB Retrieve&Generate| KB
  KBDS --> KB
  KB --> AOSS

  %% Kendra path
  Lrag --> Kendra
  KDS --> Kendra

  %% Data ingest
  U -->|Upload docs (future)| S3FILE
  U -->|KB docs| KBDS
  U -->|Kendra docs| KDS
```

## コンポーネント詳細
- フロントエンド: CloudFront + S3 ホスティング。環境変数で RAG(KB/Kendra) 切替、モデル一覧、Agent/MCP、認証情報をバンドル。
- 認証: Cognito User Pool（ドメイン `genu-demo.auth.ap-northeast-1.amazoncognito.com`）。SAML は将来拡張用。ログイン後の idToken を API 呼び出しに使用。
- API Gateway & Lambda:
  - PredictStream (主要チャット/RAG/Agent/MCP ルーター)
  - Rag Query/Retrieve（Kendra）
  - KB Retrieve（Bedrock KB）
  - 画像/動画生成、Flow/タイトル生成、ファイル署名 URL など補助関数
  - DynamoDB にチャット履歴、統計を保存。S3 File Bucket はアップロード/生成物保管。
- RAG (Knowledge Base):
  - S3 KB Datasource Bucket -> Bedrock KB (BAIYEVEULE) -> AOSS Collection `generative-ai-use-cases-jpdev`
  - 埋め込み: Titan Text v2 Binary / Advanced Parsing 有効 / Rerank: amazon.rerank-v1 / Query Decomposition ON
  - 暗黙・隠しメタデータフィルタは無効化（メタデータ無し PDF を除外しないため）。
- RAG (Kendra):
  - S3 Datasource Bucket -> Kendra Index (ja) / Developer Edition
  - 現在サンプルは `QuantumMesh.pdf` のみ。
- モデル:
  - Text: Claude 3.5 Sonnet v1 (東京)、Claude 3.7 Sonnet (us-east-1)、Nova Pro/Lite/Micro (東京)、Nova Premier (us-east-1)、DeepSeek R1 (us-west-2)
  - 画像/動画/音声: Nova Canvas/Reel/Sonic（東京、AIP を自動付与）
- Agent/Flow/MCP:
  - AgentCore runtimes in us-west-2（Generic/AgentBuilder）
  - MCP Lambda URL + サンプル MCP サーバー定義（API キーは後入れ）
  - WebSearch Agent (Brave) をサンプルとして有効
- ガードレール:
  - Guardrail 87xulvzb1pln (DRAFT) を Converse ルートで適用。PII ブロック設定のみ（デフォルト）。
- 監視:
  - CloudWatch Logs（Lambda/APIGW），CloudWatch Dashboard Stack（ap-northeast-1）
  - API/PredictStream のロググループ `/aws/lambda/GenerativeAiUseCasesStack-APIPredictStream44DDBC25-*`

## 主なデータフロー
1. **認証**: ブラウザ → CloudFront → Cognito Hosted UI → idToken 取得 → API 呼び出し時に使用。
2. **通常チャット**: ブラウザ → APIGW → Lambda PredictStream → Bedrock (Converse) → Guardrail → 応答を SSE で返却。
3. **RAG (Knowledge Base)**: ブラウザ → PredictStream (model.type=bedrockKb) → Bedrock RetrieveAndGenerate(KB) → KB → AOSS → 回答＋引用。フィルタは無効化済み。
4. **RAG (Kendra)**: ブラウザ → Rag Query/Retrieve Lambda → Kendra → 回答＋引用。
5. **画像/動画/音声**: ブラウザ → APIGW → Lambda → Bedrock (Nova Canvas/Reel/Sonic with AIP) → 一時 S3 に出力（動画）→ 署名 URL で取得。
6. **Agent/AgentCore**: ブラウザ → PredictStream → AgentCore runtimes (us-west-2) または WebSearch Agent → 結果を返却。
7. **MCP**: ブラウザ → PredictStream → MCP Lambda URL → MCP サーバー（後で API キー設定）。

## 運用メモ / 試行錯誤の反映
- KB でメタデータ無し PDF を除外しないよう、暗黙/隠しフィルタを無効化済み（2025-11-24）。メタデータを活用する場合のみ再有効化すること。
- KB サンプルは `kagutan.pdf` のみ、Kendra サンプルは `QuantumMesh.pdf` のみ。不要 PDF はバケットから削除済み。
- CloudFront: `https://d1vmznikinaycd.cloudfront.net` / Hosted UI: `https://genu-demo.auth.ap-northeast-1.amazoncognito.com/login?...`
- スタック一覧は `artifacts/HANDOVER_20251122.md` を参照。
