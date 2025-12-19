# マルチテナント実装ガイド

## 目次
1. [概要](#概要)
2. [アーキテクチャ設計判断](#アーキテクチャ設計判断)
3. [データモデル](#データモデル)
4. [実装の詳細](#実装の詳細)
5. [実装手順](#実装手順)
6. [コード例](#コード例)
7. [テスト方法](#テスト方法)
8. [トラブルシューティング](#トラブルシューティング)
9. [ベストプラクティス](#ベストプラクティス)

---

## 概要

このドキュメントでは、React Context（TenantProvider）を使用したアプリケーション単位のマルチテナント実装について説明します。

### 目的

- 1つのコードベースで複数の組織・部門向けアプリケーションを提供
- テナント毎に異なるUI、RAG設定、LLM設定、機能セットを動的に適用
- 部門切り替えトグルによる即座のテナント切り替え
- 将来のLLMゲートウェイ拡張に対応した設計

### 主要機能

- **アプリケーション単位のテナント管理**: 1画面 = 1テナント設定
- **動的UI適用**: テナント毎のブランディング（ロゴ、色、テーマ）
- **RAG設定の動的切り替え**: テナント毎のKnowledge Base、フィルタ、検索設定
- **LLM設定の動的切り替え**: テナント毎のシステムプロンプト、temperature、モデル選択
- **機能制御**: テナント毎の有効/無効機能の切り替え
- **リアルタイム切り替え**: 部門トグルによる即座のアプリケーション切り替え
- **LLMゲートウェイ対応**: テナント毎のレート制限、予算管理、モデル制限

### 関連ドキュメント

- [SSO実装ガイド](./SSO_IMPLEMENTATION_GUIDE.md): Azure Entra ID SAML認証との連携

---

## アーキテクチャ設計判断

### アプローチの比較

マルチテナント実装には2つのアプローチがあります：

#### アプローチA: アプリケーション単位（推奨）✅

```
┌─────────────────────────────────────────────────────────────┐
│ TenantProvider (React Context)                              │
│ currentTenant = { tenantId, uiConfig, ragConfig, ... }      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ App Component                                        │  │
│  │  ├─ ChatPage     ← useTenant()で設定取得             │  │
│  │  ├─ RAGPage      ← useTenant()で設定取得             │  │
│  │  ├─ ImageGenPage ← useTenant()で設定取得             │  │
│  │  └─ SettingsPage ← useTenant()で設定取得             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

特徴:
- 1画面 = 1テナント
- グローバルステート（React Context）で一元管理
- 全コンポーネントが同じテナント設定を参照
```

**メリット**:
- ✅ 実装がシンプル（約150行のContext）
- ✅ コード重複なし
- ✅ 保守性が高い（変更箇所が1箇所）
- ✅ パフォーマンス良好（API呼び出し1回）
- ✅ テナント切り替えが簡単
- ✅ ユーザー体験の一貫性

**デメリット**:
- ❌ 1画面で複数テナント同時表示は不可

#### アプローチB: コンポーネント単位

```
┌─────────────────────────────────────────────────────────────┐
│ App Component                                               │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ChatPage                                             │  │
│  │   <ChatComponent tenantId="tenant-engineering" />    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ DashboardPage (複数テナント同時表示)                  │  │
│  │   <ChatWidget tenantId="tenant-engineering" />       │  │
│  │   <ChatWidget tenantId="tenant-sales" />             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

特徴:
- 1画面に複数テナントが共存可能
- 各コンポーネントが個別にテナント設定を保持
```

**メリット**:
- ✅ 1画面で複数テナント同時表示が可能

**デメリット**:
- ❌ 実装が複雑（約400行以上）
- ❌ コード重複が多い
- ❌ 保守性が低い（各コンポーネントで修正）
- ❌ パフォーマンス低下（コンポーネント毎にAPI呼び出し）
- ❌ テナント切り替えが複雑
- ❌ ユーザー体験の一貫性が低い

### 設計判断: アプローチA（アプリケーション単位）を採用

**理由**:

1. **実装の簡潔性**: コード量が1/3以下、保守性が高い
2. **ユーザー体験**: 1画面 = 1テナント設定で一貫性のある体験
3. **パフォーマンス**: API呼び出し1回、メモリ効率が良い
4. **拡張性**: LLMゲートウェイ実装が容易
5. **テスト容易性**: テスト対象が1箇所（TenantProvider）

**例外ケース**:
将来的にダッシュボードで複数テナント同時表示が必要な場合は、ハイブリッドアプローチ（デフォルトはアプリケーション単位、特定コンポーネントのみコンポーネント単位）を検討。

---

## データモデル

### DynamoDB TenantMaster テーブル

```typescript
{
  // Partition Key
  tenantId: "tenant-engineering",

  // 基本情報
  tenantName: "技術部門ポータル",
  departmentId: "engineering",
  enabled: true,

  // アプリケーション設定（1つのオブジェクトとして管理）
  config: {
    // ========================================
    // UI設定
    // ========================================
    ui: {
      branding: {
        logoUrl: "https://cdn.example.com/engineering-logo.png",
        primaryColor: "#1E3A8A",        // 青色（技術部門）
        secondaryColor: "#3B82F6",
        companyName: "株式会社〇〇 技術本部"
      },
      features: {
        chatEnabled: true,
        ragEnabled: true,
        imageGenerationEnabled: false,
        agentEnabled: true,
        customFeatures: ["code-review", "architecture-design"]
      },
      layout: {
        sidebarPosition: "left",
        theme: "dark",
        language: "ja",
        customCss: "https://cdn.example.com/tenant-engineering.css"
      },
      navigation: {
        customMenuItems: [
          {
            label: "技術ドキュメント",
            icon: "PiBook",
            to: "/tech-docs",
            requiredRole: "engineering"
          }
        ]
      }
    },

    // ========================================
    // RAG設定
    // ========================================
    rag: {
      knowledgeBaseIds: [
        "kb-engineering-docs",
        "kb-technical-specs"
      ],
      defaultFilters: {
        department: "engineering",
        documentType: ["technical", "architecture"]
      },
      vectorSearchConfig: {
        numberOfResults: 10,
        searchType: "HYBRID"
      }
    },

    // ========================================
    // LLM設定
    // ========================================
    llm: {
      systemPromptTemplate: `あなたは技術部門向けのAIアシスタントです。
コードレビュー、アーキテクチャ設計、技術ドキュメント作成を支援します。
回答は技術的に正確で、ベストプラクティスに従ってください。`,

      ragPromptTemplate: `以下の技術ドキュメントを参照して、質問に答えてください。

参照ドキュメント:
{context}

質問: {question}

技術的に正確で、具体的なコード例を含めて回答してください。`,

      temperature: 0.3,  // 技術的な回答は低めのtemperature
      maxTokens: 4096,
      defaultModel: "claude-3-5-sonnet",
      stopSequences: []
    },

    // ========================================
    // LLMゲートウェイ設定（将来拡張）
    // ========================================
    gateway: {
      allowedModels: [
        "claude-3-5-sonnet",
        "gpt-4o"
      ],
      defaultModel: "claude-3-5-sonnet",
      rateLimitPerHour: 1000,
      rateLimitPerDay: 10000,
      maxTokensPerRequest: 100000,
      budgetLimitPerMonth: 100000,
      piiFilteringEnabled: true,
      auditLogEnabled: true,
      allowedDomains: ["*.example.com"],
      blockedKeywords: ["password", "secret"]
    }
  },

  // アクセス制御
  allowedDepartments: ["engineering", "devops"],
  allowedUserEmails: [
    "s.abe@churadata.okinawa",
    "*@engineering.example.com"
  ],

  // メタデータ
  createdAt: "2025-01-15T10:00:00Z",
  updatedAt: "2025-01-15T10:00:00Z",
  createdBy: "admin@example.com",
  updatedBy: "admin@example.com"
}
```

### 営業部門の設定例（比較）

```typescript
{
  tenantId: "tenant-sales",
  tenantName: "営業部門ポータル",
  departmentId: "sales",
  enabled: true,

  config: {
    ui: {
      branding: {
        logoUrl: "https://cdn.example.com/sales-logo.png",
        primaryColor: "#16A34A",        // 緑色（営業部門）
        secondaryColor: "#22C55E",
        companyName: "株式会社〇〇 営業本部"
      },
      features: {
        chatEnabled: true,
        ragEnabled: true,
        imageGenerationEnabled: true,  // 営業資料作成用
        agentEnabled: false,
        customFeatures: ["customer-proposal", "sales-analytics"]
      },
      theme: "light"
    },

    rag: {
      knowledgeBaseIds: [
        "kb-sales-docs",
        "kb-customer-data"
      ],
      defaultFilters: {
        department: "sales",
        documentType: ["proposal", "case-study"]
      }
    },

    llm: {
      systemPromptTemplate: `あなたは営業支援AIアシスタントです。
顧客提案、営業戦略、プレゼンテーション作成を支援します。
ビジネス価値を明確に伝え、説得力のある回答を心がけてください。`,

      temperature: 0.7,  // 創造性重視
      defaultModel: "gpt-4o"
    },

    gateway: {
      allowedModels: ["gpt-4o", "dall-e-3"],
      rateLimitPerHour: 500,
      budgetLimitPerMonth: 50000
    }
  },

  allowedDepartments: ["sales", "marketing"]
}
```

### テーブル定義（CDK）

```typescript
// packages/cdk/lib/construct/database.ts
const tenantMasterTable = new Table(this, 'TenantMaster', {
  tableName: 'TenantMaster',
  partitionKey: {
    name: 'tenantId',
    type: AttributeType.STRING,
  },
  billingMode: BillingMode.PAY_PER_REQUEST,
  encryption: TableEncryption.AWS_MANAGED,
  pointInTimeRecovery: true,
  removalPolicy: RemovalPolicy.RETAIN,
});

// GSI: departmentId でクエリ可能にする
tenantMasterTable.addGlobalSecondaryIndex({
  indexName: 'DepartmentIdIndex',
  partitionKey: {
    name: 'departmentId',
    type: AttributeType.STRING,
  },
  projectionType: ProjectionType.ALL,
});
```

---

## 実装の詳細

### 実装フロー

```
┌─────────────────────────────────────────────────────────────┐
│ 1. ユーザーログイン (Cognito + Entra ID SAML)                │
│    → JWT token with custom:samlGroups="Engineering,Sales"    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. ユーザープロファイル取得                                    │
│    GET /api/user/profile                                      │
│    → {                                                        │
│         userId: "s.abe@churadata.okinawa",                   │
│         departments: ["engineering", "sales"],               │
│         tenants: [                                            │
│           {                                                   │
│             tenantId: "tenant-engineering",                  │
│             tenantName: "技術部門ポータル",                    │
│             departmentId: "engineering"                      │
│           },                                                  │
│           {                                                   │
│             tenantId: "tenant-sales",                        │
│             tenantName: "営業部門ポータル",                    │
│             departmentId: "sales"                            │
│           }                                                   │
│         ],                                                    │
│         defaultTenantId: "tenant-engineering"                │
│       }                                                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. テナント設定の完全取得（初回）                              │
│    GET /api/tenant/tenant-engineering/config                  │
│    → TenantMaster 全設定 (UI + RAG + LLM + Gateway)          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. TenantProvider初期化                                       │
│    - currentTenantステートに設定保存                           │
│    - UIブランディング適用（CSS変数、テーマ）                    │
│    - RAG/LLM設定をlocalStorageに保存（API呼び出し時に使用）    │
│    - ロゴ、タイトル、ファビコン設定                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. アプリケーション表示（技術部門ポータル）                     │
│    - 青色テーマ                                               │
│    - 技術部門ロゴ                                             │
│    - コードレビュー、アーキテクチャ設計機能                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. ユーザーが部門切り替えトグルをクリック                       │
│    「技術部門」→「営業部門」                                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. テナント切り替え処理                                        │
│    POST /api/tenant/tenant-sales/validate (アクセス権限検証)  │
│    GET /api/tenant/tenant-sales/config (新設定取得)          │
│    - localStorageを更新                                       │
│    - アプリケーション全体を再初期化（window.location.reload） │
│    ※ JWT tokenはリフレッシュ不要                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. アプリケーション表示（営業部門ポータル）                     │
│    - 緑色テーマ                                               │
│    - 営業部門ロゴ                                             │
│    - 顧客提案、営業分析機能                                    │
│    - 異なるKnowledge Base、システムプロンプト                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. RAGクエリ（営業部門設定で実行）                             │
│    - Knowledge Base: kb-sales-docs                            │
│    - Filter: { department: "sales" }                          │
│    - System Prompt: "営業支援AIアシスタント..."                │
│    - Temperature: 0.7                                         │
│    - Model: gpt-4o                                            │
└─────────────────────────────────────────────────────────────┘
```

### コンポーネント構成

```
packages/web/src/
├── contexts/
│   └── TenantContext.tsx              ← テナント管理の中核
├── components/
│   ├── TenantSwitcher.tsx             ← テナント切り替えUI
│   └── UserInfo.tsx                   ← ユーザー情報表示
├── hooks/
│   ├── useTenant.ts                   ← Context re-export
│   ├── useChat.ts                     ← テナント設定を利用
│   └── useRag.ts                      ← テナント設定を利用
└── pages/
    ├── ChatPage.tsx                   ← useTenant()を呼ぶ
    ├── RAGPage.tsx                    ← useTenant()を呼ぶ
    └── App.tsx                        ← TenantProviderで全体をラップ

packages/cdk/lambda/
├── getUserProfile.ts                  ← ユーザーの所属テナント取得
├── getTenantConfig.ts                 ← テナント設定取得
├── validateTenantAccess.ts            ← アクセス権限検証
├── predict.ts                         ← テナント設定を使用（修正）
└── predictStream.ts                   ← テナント設定を使用（修正）
```

---

## 実装手順

### フェーズ1: DynamoDB テーブル作成

#### ステップ1: CDKでTenantMasterテーブル定義

```typescript
// packages/cdk/lib/construct/database.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import {
  Table,
  AttributeType,
  BillingMode,
  TableEncryption,
  ProjectionType,
} from 'aws-cdk-lib/aws-dynamodb';

export interface DatabaseProps {
  readonly envName: string;
}

export class Database extends Construct {
  public readonly tenantMasterTable: Table;

  constructor(scope: Construct, id: string, props: DatabaseProps) {
    super(scope, id);

    // TenantMaster テーブル
    this.tenantMasterTable = new Table(this, 'TenantMaster', {
      tableName: `TenantMaster-${props.envName}`,
      partitionKey: {
        name: 'tenantId',
        type: AttributeType.STRING,
      },
      billingMode: BillingMode.PAY_PER_REQUEST,
      encryption: TableEncryption.AWS_MANAGED,
      pointInTimeRecovery: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // GSI: departmentId でクエリ
    this.tenantMasterTable.addGlobalSecondaryIndex({
      indexName: 'DepartmentIdIndex',
      partitionKey: {
        name: 'departmentId',
        type: AttributeType.STRING,
      },
      projectionType: ProjectionType.ALL,
    });

    // Outputs
    new cdk.CfnOutput(this, 'TenantMasterTableName', {
      value: this.tenantMasterTable.tableName,
      description: 'TenantMaster DynamoDB Table Name',
    });

    new cdk.CfnOutput(this, 'TenantMasterTableArn', {
      value: this.tenantMasterTable.tableArn,
      description: 'TenantMaster DynamoDB Table ARN',
    });
  }
}
```

#### ステップ2: メインスタックに統合

```typescript
// packages/cdk/lib/generative-ai-use-cases-stack.ts
import { Database } from './construct/database';

export class GenerativeAiUseCasesStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Database construct
    const database = new Database(this, 'Database', {
      envName: process.env.ENVIRONMENT_NAME || 'dev',
    });

    // Pass to other constructs that need it
    const api = new Api(this, 'Api', {
      // ... other props
      tenantMasterTable: database.tenantMasterTable,
    });
  }
}
```

#### ステップ3: 初期データ投入

```bash
# packages/cdk/scripts/seed-tenant-data.sh
#!/bin/bash

TABLE_NAME="TenantMaster-dev2"

# 技術部門テナント
aws dynamodb put-item \
  --table-name "$TABLE_NAME" \
  --item file://seed-data/tenant-engineering.json

# 営業部門テナント
aws dynamodb put-item \
  --table-name "$TABLE_NAME" \
  --item file://seed-data/tenant-sales.json
```

```json
// packages/cdk/scripts/seed-data/tenant-engineering.json
{
  "tenantId": { "S": "tenant-engineering" },
  "tenantName": { "S": "技術部門ポータル" },
  "departmentId": { "S": "engineering" },
  "enabled": { "BOOL": true },
  "config": {
    "M": {
      "ui": {
        "M": {
          "branding": {
            "M": {
              "logoUrl": { "S": "https://cdn.example.com/engineering-logo.png" },
              "primaryColor": { "S": "#1E3A8A" },
              "secondaryColor": { "S": "#3B82F6" },
              "companyName": { "S": "株式会社〇〇 技術本部" }
            }
          },
          "features": {
            "M": {
              "chatEnabled": { "BOOL": true },
              "ragEnabled": { "BOOL": true },
              "imageGenerationEnabled": { "BOOL": false },
              "agentEnabled": { "BOOL": true }
            }
          },
          "layout": {
            "M": {
              "theme": { "S": "dark" },
              "language": { "S": "ja" }
            }
          }
        }
      },
      "rag": {
        "M": {
          "knowledgeBaseIds": {
            "L": [
              { "S": "kb-engineering-docs" }
            ]
          },
          "defaultFilters": {
            "M": {
              "department": { "S": "engineering" }
            }
          },
          "vectorSearchConfig": {
            "M": {
              "numberOfResults": { "N": "10" },
              "searchType": { "S": "HYBRID" }
            }
          }
        }
      },
      "llm": {
        "M": {
          "systemPromptTemplate": {
            "S": "あなたは技術部門向けのAIアシスタントです。\nコードレビュー、アーキテクチャ設計、技術ドキュメント作成を支援します。\n回答は技術的に正確で、ベストプラクティスに従ってください。"
          },
          "temperature": { "N": "0.3" },
          "maxTokens": { "N": "4096" },
          "defaultModel": { "S": "claude-3-5-sonnet" }
        }
      },
      "gateway": {
        "M": {
          "allowedModels": {
            "L": [
              { "S": "claude-3-5-sonnet" },
              { "S": "gpt-4o" }
            ]
          },
          "rateLimitPerHour": { "N": "1000" },
          "maxTokensPerRequest": { "N": "100000" },
          "budgetLimitPerMonth": { "N": "100000" }
        }
      }
    }
  },
  "allowedDepartments": {
    "L": [
      { "S": "engineering" },
      { "S": "devops" }
    ]
  },
  "createdAt": { "S": "2025-01-19T10:00:00Z" },
  "updatedAt": { "S": "2025-01-19T10:00:00Z" }
}
```

### フェーズ2: バックエンドAPI実装

#### Lambda関数1: getUserProfile

```typescript
// packages/cdk/lambda/getUserProfile.ts
import {
  DynamoDBClient,
  QueryCommand,
  GetCommand,
} from '@aws-sdk/client-dynamodb';
import { unmarshall } from '@aws-sdk/util-dynamodb';
import type {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
} from 'aws-lambda';

const dynamodb = new DynamoDBClient({});
const TENANT_TABLE_NAME = process.env.TENANT_TABLE_NAME!;

interface TenantInfo {
  tenantId: string;
  tenantName: string;
  departmentId: string;
}

// JWT tokenからユーザーの部門を取得
const getUserDepartments = (claims: any): string[] => {
  const samlGroups = claims['custom:samlGroups'];
  if (typeof samlGroups === 'string' && samlGroups.length > 0) {
    return samlGroups
      .split(',')
      .map((g: string) => g.trim().toLowerCase())
      .filter((g: string) => g.length > 0);
  }
  return [];
};

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  console.log('[getUserProfile] Event:', JSON.stringify(event, null, 2));

  try {
    // ユーザー認証情報を取得
    const claims = event.requestContext.authorizer?.claims;
    if (!claims) {
      return {
        statusCode: 401,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Unauthorized' }),
      };
    }

    const userEmail = claims.email;
    const userDepartments = getUserDepartments(claims);

    console.log('[getUserProfile] User email:', userEmail);
    console.log('[getUserProfile] User departments:', userDepartments);

    // ユーザーがアクセス可能なテナントを検索
    const availableTenants: TenantInfo[] = [];

    for (const department of userDepartments) {
      // DepartmentIdIndex を使用してクエリ
      const queryResult = await dynamodb.send(
        new QueryCommand({
          TableName: TENANT_TABLE_NAME,
          IndexName: 'DepartmentIdIndex',
          KeyConditionExpression: 'departmentId = :departmentId',
          ExpressionAttributeValues: {
            ':departmentId': { S: department },
            ':enabled': { BOOL: true },
          },
          FilterExpression: 'enabled = :enabled',
        })
      );

      if (queryResult.Items) {
        for (const item of queryResult.Items) {
          const tenant = unmarshall(item);
          availableTenants.push({
            tenantId: tenant.tenantId,
            tenantName: tenant.tenantName,
            departmentId: tenant.departmentId,
          });
        }
      }
    }

    // デフォルトテナントを決定（engineeringを優先）
    let defaultTenantId = availableTenants[0]?.tenantId;
    const engineeringTenant = availableTenants.find(
      (t) => t.departmentId === 'engineering'
    );
    if (engineeringTenant) {
      defaultTenantId = engineeringTenant.tenantId;
    }

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        userId: userEmail,
        departments: userDepartments,
        tenants: availableTenants,
        defaultTenantId: defaultTenantId,
      }),
    };
  } catch (error) {
    console.error('[getUserProfile] Error:', error);
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        error: 'Internal server error',
        message: error instanceof Error ? error.message : 'Unknown error',
      }),
    };
  }
};
```

#### Lambda関数2: getTenantConfig

```typescript
// packages/cdk/lambda/getTenantConfig.ts
import { DynamoDBClient, GetCommand } from '@aws-sdk/client-dynamodb';
import { unmarshall } from '@aws-sdk/util-dynamodb';
import type {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
} from 'aws-lambda';

const dynamodb = new DynamoDBClient({});
const TENANT_TABLE_NAME = process.env.TENANT_TABLE_NAME!;

// JWT tokenからユーザーの部門を取得
const getUserDepartments = (claims: any): string[] => {
  const samlGroups = claims['custom:samlGroups'];
  if (typeof samlGroups === 'string' && samlGroups.length > 0) {
    return samlGroups
      .split(',')
      .map((g: string) => g.trim().toLowerCase())
      .filter((g: string) => g.length > 0);
  }
  return [];
};

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  console.log('[getTenantConfig] Event:', JSON.stringify(event, null, 2));

  try {
    const tenantId = event.pathParameters?.tenantId;
    if (!tenantId) {
      return {
        statusCode: 400,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'tenantId is required' }),
      };
    }

    // ユーザー認証情報を取得
    const claims = event.requestContext.authorizer?.claims;
    if (!claims) {
      return {
        statusCode: 401,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Unauthorized' }),
      };
    }

    const userDepartments = getUserDepartments(claims);
    console.log('[getTenantConfig] User departments:', userDepartments);

    // テナント設定を取得
    const getResult = await dynamodb.send(
      new GetCommand({
        TableName: TENANT_TABLE_NAME,
        Key: {
          tenantId: { S: tenantId },
        },
      })
    );

    if (!getResult.Item) {
      return {
        statusCode: 404,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Tenant not found' }),
      };
    }

    const tenant = unmarshall(getResult.Item);

    // テナントが有効かチェック
    if (!tenant.enabled) {
      return {
        statusCode: 403,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Tenant is disabled' }),
      };
    }

    // ユーザーがこのテナントにアクセス権限があるかチェック
    const allowedDepartments = tenant.allowedDepartments || [];
    const hasAccess = userDepartments.some((dept: string) =>
      allowedDepartments.includes(dept)
    );

    if (!hasAccess) {
      console.log(
        '[getTenantConfig] Access denied. User departments:',
        userDepartments,
        'Allowed:',
        allowedDepartments
      );
      return {
        statusCode: 403,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Access denied to this tenant' }),
      };
    }

    // テナント設定を返却
    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(tenant),
    };
  } catch (error) {
    console.error('[getTenantConfig] Error:', error);
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        error: 'Internal server error',
        message: error instanceof Error ? error.message : 'Unknown error',
      }),
    };
  }
};
```

#### Lambda関数3: validateTenantAccess

```typescript
// packages/cdk/lambda/validateTenantAccess.ts
import { DynamoDBClient, GetCommand } from '@aws-sdk/client-dynamodb';
import { unmarshall } from '@aws-sdk/util-dynamodb';
import type {
  APIGatewayProxyEvent,
  APIGatewayProxyResult,
} from 'aws-lambda';

const dynamodb = new DynamoDBClient({});
const TENANT_TABLE_NAME = process.env.TENANT_TABLE_NAME!;

const getUserDepartments = (claims: any): string[] => {
  const samlGroups = claims['custom:samlGroups'];
  if (typeof samlGroups === 'string' && samlGroups.length > 0) {
    return samlGroups
      .split(',')
      .map((g: string) => g.trim().toLowerCase())
      .filter((g: string) => g.length > 0);
  }
  return [];
};

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  console.log('[validateTenantAccess] Event:', JSON.stringify(event, null, 2));

  try {
    const tenantId = event.pathParameters?.tenantId;
    if (!tenantId) {
      return {
        statusCode: 400,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'tenantId is required' }),
      };
    }

    const claims = event.requestContext.authorizer?.claims;
    if (!claims) {
      return {
        statusCode: 401,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Unauthorized' }),
      };
    }

    const userDepartments = getUserDepartments(claims);

    // テナント設定を取得
    const getResult = await dynamodb.send(
      new GetCommand({
        TableName: TENANT_TABLE_NAME,
        Key: {
          tenantId: { S: tenantId },
        },
      })
    );

    if (!getResult.Item) {
      return {
        statusCode: 404,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'Tenant not found' }),
      };
    }

    const tenant = unmarshall(getResult.Item);

    // アクセス権限チェック
    const allowedDepartments = tenant.allowedDepartments || [];
    const hasAccess = userDepartments.some((dept: string) =>
      allowedDepartments.includes(dept)
    );

    if (!hasAccess) {
      return {
        statusCode: 403,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          error: 'Access denied to this tenant',
          userDepartments,
          allowedDepartments,
        }),
      };
    }

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: 'Access granted',
        tenantId,
      }),
    };
  } catch (error) {
    console.error('[validateTenantAccess] Error:', error);
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        error: 'Internal server error',
        message: error instanceof Error ? error.message : 'Unknown error',
      }),
    };
  }
};
```

#### API Gateway エンドポイント定義

```typescript
// packages/cdk/lib/construct/api.ts
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import * as path from 'path';

export class Api extends Construct {
  constructor(scope: Construct, id: string, props: ApiProps) {
    super(scope, id);

    // Lambda functions
    const getUserProfileFunction = new NodejsFunction(
      this,
      'GetUserProfile',
      {
        runtime: LAMBDA_RUNTIME_NODEJS,
        entry: path.join(__dirname, '../../lambda/getUserProfile.ts'),
        environment: {
          TENANT_TABLE_NAME: props.tenantMasterTable.tableName,
        },
      }
    );

    const getTenantConfigFunction = new NodejsFunction(
      this,
      'GetTenantConfig',
      {
        runtime: LAMBDA_RUNTIME_NODEJS,
        entry: path.join(__dirname, '../../lambda/getTenantConfig.ts'),
        environment: {
          TENANT_TABLE_NAME: props.tenantMasterTable.tableName,
        },
      }
    );

    const validateTenantAccessFunction = new NodejsFunction(
      this,
      'ValidateTenantAccess',
      {
        runtime: LAMBDA_RUNTIME_NODEJS,
        entry: path.join(__dirname, '../../lambda/validateTenantAccess.ts'),
        environment: {
          TENANT_TABLE_NAME: props.tenantMasterTable.tableName,
        },
      }
    );

    // Grant DynamoDB permissions
    props.tenantMasterTable.grantReadData(getUserProfileFunction);
    props.tenantMasterTable.grantReadData(getTenantConfigFunction);
    props.tenantMasterTable.grantReadData(validateTenantAccessFunction);

    // API Gateway routes
    const userResource = api.root.addResource('user');
    userResource
      .addResource('profile')
      .addMethod('GET', new LambdaIntegration(getUserProfileFunction), {
        authorizer: cognitoAuthorizer,
      });

    const tenantResource = api.root.addResource('tenant');
    const tenantIdResource = tenantResource.addResource('{tenantId}');

    tenantIdResource
      .addResource('config')
      .addMethod('GET', new LambdaIntegration(getTenantConfigFunction), {
        authorizer: cognitoAuthorizer,
      });

    tenantIdResource
      .addResource('validate')
      .addMethod('POST', new LambdaIntegration(validateTenantAccessFunction), {
        authorizer: cognitoAuthorizer,
      });
  }
}
```

### フェーズ3: フロントエンド実装

#### TenantContext実装

```typescript
// packages/web/src/contexts/TenantContext.tsx
import React, { useState, useEffect, useContext, createContext } from 'react';
import { fetchAuthSession } from 'aws-amplify/auth';

interface TenantInfo {
  tenantId: string;
  tenantName: string;
  departmentId: string;
}

interface TenantConfig {
  tenantId: string;
  tenantName: string;
  departmentId: string;
  enabled: boolean;
  config: {
    ui: {
      branding: {
        logoUrl: string;
        primaryColor: string;
        secondaryColor: string;
        companyName: string;
      };
      features: {
        chatEnabled: boolean;
        ragEnabled: boolean;
        imageGenerationEnabled: boolean;
        agentEnabled: boolean;
      };
      layout: {
        theme: 'light' | 'dark';
        language: string;
      };
    };
    rag: {
      knowledgeBaseIds: string[];
      defaultFilters: Record<string, any>;
      vectorSearchConfig: {
        numberOfResults: number;
        searchType: 'SEMANTIC' | 'HYBRID';
      };
    };
    llm: {
      systemPromptTemplate: string;
      ragPromptTemplate: string;
      temperature: number;
      maxTokens: number;
      defaultModel: string;
    };
    gateway: {
      allowedModels: string[];
      rateLimitPerHour: number;
      maxTokensPerRequest: number;
    };
  };
}

interface TenantContextType {
  currentTenant: TenantConfig | null;
  availableTenants: TenantInfo[];
  switchTenant: (tenantId: string) => Promise<void>;
  isFeatureEnabled: (featureName: string) => boolean;
  loading: boolean;
}

const TenantContext = createContext<TenantContextType | undefined>(undefined);

export const TenantProvider: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const [currentTenant, setCurrentTenant] = useState<TenantConfig | null>(null);
  const [availableTenants, setAvailableTenants] = useState<TenantInfo[]>([]);
  const [loading, setLoading] = useState(true);

  // UIテーマを適用
  const applyTheme = (uiConfig: TenantConfig['config']['ui']) => {
    const root = document.documentElement;

    // CSS変数を設定
    root.style.setProperty(
      '--tenant-primary-color',
      uiConfig.branding.primaryColor
    );
    root.style.setProperty(
      '--tenant-secondary-color',
      uiConfig.branding.secondaryColor
    );

    // テーマクラスを設定
    root.setAttribute('data-theme', uiConfig.layout.theme);

    // ドキュメントタイトルを設定
    document.title = uiConfig.branding.companyName;

    // ファビコンを設定
    const favicon = document.querySelector("link[rel*='icon']") as HTMLLinkElement;
    if (favicon && uiConfig.branding.logoUrl) {
      favicon.href = uiConfig.branding.logoUrl;
    }

    console.log('[TenantProvider] Theme applied:', uiConfig.branding);
  };

  // テナント初期化
  const initializeTenant = async (tenantId: string) => {
    console.log(`[TenantProvider] Initializing tenant: ${tenantId}`);

    try {
      // 認証セッションを取得
      const session = await fetchAuthSession();
      const idToken = session.tokens?.idToken;

      if (!idToken) {
        throw new Error('Not authenticated');
      }

      // テナント設定を取得
      const response = await fetch(
        `${import.meta.env.VITE_APP_API_ENDPOINT}/tenant/${tenantId}/config`,
        {
          headers: {
            Authorization: `Bearer ${idToken.toString()}`,
          },
        }
      );

      if (!response.ok) {
        throw new Error(`Failed to fetch tenant config: ${response.statusText}`);
      }

      const tenantConfig: TenantConfig = await response.json();
      console.log('[TenantProvider] Tenant config loaded:', tenantConfig);

      // UIテーマを適用
      applyTheme(tenantConfig.config.ui);

      // グローバルステートに設定を保存
      setCurrentTenant(tenantConfig);
      localStorage.setItem('selected_tenant', tenantId);

      // RAG/LLM設定をlocalStorageに保存（API呼び出し時に使用）
      localStorage.setItem(
        'tenant_rag_config',
        JSON.stringify(tenantConfig.config.rag)
      );
      localStorage.setItem(
        'tenant_llm_config',
        JSON.stringify(tenantConfig.config.llm)
      );

      console.log('[TenantProvider] Tenant initialization complete');
    } catch (error) {
      console.error('[TenantProvider] Failed to initialize tenant:', error);
      throw error;
    }
  };

  // 初回ロード時
  useEffect(() => {
    const initializeOnLoad = async () => {
      setLoading(true);

      try {
        // 認証セッションを取得
        const session = await fetchAuthSession();
        const idToken = session.tokens?.idToken;

        if (!idToken) {
          console.log('[TenantProvider] Not authenticated yet');
          setLoading(false);
          return;
        }

        // ユーザープロファイルを取得
        const profileResponse = await fetch(
          `${import.meta.env.VITE_APP_API_ENDPOINT}/user/profile`,
          {
            headers: {
              Authorization: `Bearer ${idToken.toString()}`,
            },
          }
        );

        if (!profileResponse.ok) {
          throw new Error('Failed to fetch user profile');
        }

        const profile = await profileResponse.json();
        setAvailableTenants(profile.tenants);

        // 保存されているテナントIDを確認
        const savedTenantId = localStorage.getItem('selected_tenant');

        // 使用するテナントを決定
        let tenantIdToUse: string;
        if (
          savedTenantId &&
          profile.tenants.some((t: TenantInfo) => t.tenantId === savedTenantId)
        ) {
          tenantIdToUse = savedTenantId;
        } else {
          tenantIdToUse = profile.defaultTenantId;
        }

        // テナントを初期化
        await initializeTenant(tenantIdToUse);
      } catch (error) {
        console.error('[TenantProvider] Failed to initialize on load:', error);
      } finally {
        setLoading(false);
      }
    };

    initializeOnLoad();
  }, []);

  // テナント切り替え
  const switchTenant = async (newTenantId: string) => {
    console.log(`[TenantProvider] Switching tenant to: ${newTenantId}`);

    if (currentTenant && newTenantId === currentTenant.tenantId) {
      console.log('[TenantProvider] Already on this tenant');
      return;
    }

    setLoading(true);

    try {
      // 認証セッションを取得
      const session = await fetchAuthSession();
      const idToken = session.tokens?.idToken;

      if (!idToken) {
        throw new Error('Not authenticated');
      }

      // アクセス権限を検証
      const validateResponse = await fetch(
        `${import.meta.env.VITE_APP_API_ENDPOINT}/tenant/${newTenantId}/validate`,
        {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${idToken.toString()}`,
          },
        }
      );

      if (!validateResponse.ok) {
        throw new Error('Access denied to this tenant');
      }

      // テナントを再初期化
      await initializeTenant(newTenantId);

      // アプリケーション全体をリロードして新しいテナント設定を完全に反映
      window.location.reload();
    } catch (error) {
      console.error('[TenantProvider] Failed to switch tenant:', error);
      alert('テナントの切り替えに失敗しました: ' + (error as Error).message);
    } finally {
      setLoading(false);
    }
  };

  // 機能有効チェック
  const isFeatureEnabled = (featureName: string): boolean => {
    if (!currentTenant) return false;
    return currentTenant.config.ui.features[featureName] === true;
  };

  return (
    <TenantContext.Provider
      value={{
        currentTenant,
        availableTenants,
        switchTenant,
        isFeatureEnabled,
        loading,
      }}
    >
      {children}
    </TenantContext.Provider>
  );
};

export const useTenant = () => {
  const context = useContext(TenantContext);
  if (!context) {
    throw new Error('useTenant must be used within TenantProvider');
  }
  return context;
};
```

#### TenantSwitcher UI実装

```typescript
// packages/web/src/components/TenantSwitcher.tsx
import React, { useState } from 'react';
import { useTenant } from '../contexts/TenantContext';
import { PiBuildings, PiCaretDown, PiCheck } from 'react-icons/pi';

const TenantSwitcher: React.FC = () => {
  const { currentTenant, availableTenants, switchTenant, loading } = useTenant();
  const [isOpen, setIsOpen] = useState(false);

  if (!currentTenant || availableTenants.length <= 1) {
    return null;
  }

  const handleSwitch = async (tenantId: string) => {
    if (tenantId === currentTenant.tenantId) {
      setIsOpen(false);
      return;
    }

    setIsOpen(false);
    await switchTenant(tenantId);
  };

  return (
    <div className="relative">
      {/* トグルボタン */}
      <button
        onClick={() => setIsOpen(!isOpen)}
        disabled={loading}
        className="flex items-center gap-2 px-3 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-lg hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 transition-colors"
        style={{
          borderColor: currentTenant.config.ui.branding.primaryColor,
        }}
      >
        <PiBuildings className="text-lg" />
        <span className="hidden sm:inline">{currentTenant.tenantName}</span>
        <PiCaretDown
          className={`transition-transform ${isOpen ? 'rotate-180' : ''}`}
        />
      </button>

      {/* ドロップダウンメニュー */}
      {isOpen && (
        <>
          {/* 背景オーバーレイ */}
          <div className="fixed inset-0 z-10" onClick={() => setIsOpen(false)} />

          {/* メニュー */}
          <div className="absolute right-0 z-20 mt-2 w-64 origin-top-right rounded-lg bg-white shadow-lg ring-1 ring-black ring-opacity-5">
            <div className="py-1">
              <div className="px-4 py-2 text-xs font-semibold text-gray-500 uppercase">
                テナント選択
              </div>
              {availableTenants.map((tenant) => {
                const isActive = tenant.tenantId === currentTenant.tenantId;

                return (
                  <button
                    key={tenant.tenantId}
                    onClick={() => handleSwitch(tenant.tenantId)}
                    className={`flex w-full items-center justify-between px-4 py-3 text-sm hover:bg-gray-100 transition-colors ${
                      isActive ? 'bg-gray-50' : ''
                    }`}
                  >
                    <div className="flex flex-col items-start">
                      <span className="font-medium text-gray-900">
                        {tenant.tenantName}
                      </span>
                      <span className="text-xs text-gray-500">
                        {tenant.departmentId}
                      </span>
                    </div>

                    {isActive && (
                      <PiCheck
                        className="text-lg"
                        style={{
                          color: currentTenant.config.ui.branding.primaryColor,
                        }}
                      />
                    )}
                  </button>
                );
              })}
            </div>
          </div>
        </>
      )}
    </div>
  );
};

export default TenantSwitcher;
```

#### App.tsx統合

```typescript
// packages/web/src/App.tsx
import React from 'react';
import { Outlet } from 'react-router-dom';
import { TenantProvider, useTenant } from './contexts/TenantContext';
import TenantSwitcher from './components/TenantSwitcher';
import UserInfo from './components/UserInfo';
import Drawer, { ItemProps } from './components/Drawer';
import useDrawer from './hooks/useDrawer';
import { useTranslation } from 'react-i18next';
import { PiHouse, PiList, PiX } from 'react-icons/pi';

const AppContent: React.FC = () => {
  const { t } = useTranslation();
  const { switchOpen: switchDrawer, opened: isOpenDrawer } = useDrawer();
  const { currentTenant, isFeatureEnabled, loading } = useTenant();

  // テナント設定読み込み中
  if (loading || !currentTenant) {
    return (
      <div className="flex h-screen w-screen items-center justify-center">
        <div className="text-center">
          <div className="mb-4 text-lg font-medium">
            テナント設定を読み込み中...
          </div>
          <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900 mx-auto"></div>
        </div>
      </div>
    );
  }

  // メニュー項目を動的に構築
  const items: ItemProps[] = [
    {
      label: t('navigation.home'),
      to: '/',
      icon: <PiHouse />,
      display: 'usecase' as const,
    },
    isFeatureEnabled('chatEnabled')
      ? {
          label: t('navigation.chat'),
          to: '/chat',
          icon: <PiChatsCircle />,
          display: 'usecase' as const,
        }
      : null,
    isFeatureEnabled('ragEnabled')
      ? {
          label: currentTenant.tenantName,
          to: '/rag-aurora-knowledge-base',
          icon: <PiChatCircleText />,
          display: 'usecase' as const,
        }
      : null,
    // ... その他の機能
  ].flatMap((i) => (i !== null ? [i] : []));

  return (
    <div
      className="screen:w-screen screen:h-screen overflow-x-hidden overflow-y-scroll"
      style={
        {
          '--tenant-primary': currentTenant.config.ui.branding.primaryColor,
          '--tenant-secondary': currentTenant.config.ui.branding.secondaryColor,
        } as React.CSSProperties
      }
    >
      <main className="flex-1">
        <header className="bg-aws-squid-ink visible flex h-12 w-full items-center justify-between text-lg text-white lg:invisible lg:h-0 print:hidden">
          <div className="flex w-10 items-center justify-start">
            <button
              className="focus:ring-aws-sky mr-2 rounded-full p-2 hover:opacity-50"
              onClick={() => switchDrawer()}
            >
              <PiList />
            </button>
          </div>

          {/* テナント名とロゴ */}
          <div className="flex items-center gap-2">
            <img
              src={currentTenant.config.ui.branding.logoUrl}
              alt="Logo"
              className="h-8 w-8 object-contain"
            />
            <span className="hidden sm:inline">
              {currentTenant.config.ui.branding.companyName}
            </span>
          </div>

          <div className="mr-3 flex items-center gap-3">
            {/* テナント切り替えトグル */}
            <TenantSwitcher />

            {/* ユーザー情報 */}
            <UserInfo />
          </div>
        </header>

        <div
          className={`fixed -left-64 top-0 z-50 transition-all lg:left-0 lg:z-0 ${
            isOpenDrawer ? 'left-0' : '-left-64'
          }`}
        >
          <Drawer items={items} />
        </div>

        <div className="text-aws-font-color lg:ml-64">
          <Outlet />
        </div>
      </main>
    </div>
  );
};

const App: React.FC = () => {
  return (
    <TenantProvider>
      <AppContent />
    </TenantProvider>
  );
};

export default App;
```

#### useChatフックの修正

```typescript
// packages/web/src/hooks/useChat.ts（修正部分のみ）
import { useTenant } from '../contexts/TenantContext';

const useChat = () => {
  const { currentTenant } = useTenant();

  const postChat = async (message: string) => {
    if (!currentTenant) {
      throw new Error('Tenant not initialized');
    }

    // テナントIDを含めてリクエスト
    const response = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message,
        extraData: {
          tenantId: currentTenant.tenantId,
          departmentId: currentTenant.departmentId,
        },
      }),
    });

    return response;
  };

  return { postChat };
};
```

### フェーズ4: RAG Lambda修正

```typescript
// packages/cdk/lambda/predict.ts（修正部分のみ）
import { DynamoDBClient, GetCommand } from '@aws-sdk/client-dynamodb';
import { unmarshall } from '@aws-sdk/util-dynamodb';

const dynamodb = new DynamoDBClient({});
const TENANT_TABLE_NAME = process.env.TENANT_TABLE_NAME!;

// テナント設定を取得
const getTenantConfig = async (tenantId: string): Promise<any> => {
  const result = await dynamodb.send(
    new GetCommand({
      TableName: TENANT_TABLE_NAME,
      Key: { tenantId: { S: tenantId } },
    })
  );

  if (!result.Item) {
    throw new Error(`Tenant not found: ${tenantId}`);
  }

  return unmarshall(result.Item);
};

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const body = JSON.parse(event.body || '{}');
  const message = body.message;
  const extraData = body.extraData || {};

  // テナントIDを取得
  const tenantId = extraData.tenantId;
  if (!tenantId) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'tenantId is required' }),
    };
  }

  // テナント設定を取得
  const tenantConfig = await getTenantConfig(tenantId);

  // テナント固有のRAG設定を使用
  const ragConfig = tenantConfig.config.rag;
  const ragResults = await retrieveKnowledgeBase({
    knowledgeBaseId: ragConfig.knowledgeBaseIds[0],
    retrievalQuery: { text: message },
    retrievalConfiguration: {
      vectorSearchConfiguration: {
        numberOfResults: ragConfig.vectorSearchConfig.numberOfResults,
        filter: {
          equals: {
            key: 'department',
            value: tenantConfig.departmentId,
          },
        },
      },
    },
  });

  // テナント固有のLLM設定を使用
  const llmConfig = tenantConfig.config.llm;
  const systemPrompt = llmConfig.systemPromptTemplate;

  const ragPrompt = llmConfig.ragPromptTemplate
    .replace('{context}', formatContext(ragResults))
    .replace('{question}', message);

  // テナント固有のモデル・パラメータで推論
  const response = await invokeModel({
    modelId: llmConfig.defaultModel,
    messages: [{ role: 'user', content: ragPrompt }],
    system: systemPrompt,
    inferenceConfig: {
      temperature: llmConfig.temperature,
      maxTokens: llmConfig.maxTokens,
    },
  });

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      answer: response.content[0].text,
      tenantId: tenantId,
      model: llmConfig.defaultModel,
    }),
  };
};
```

---

## コード例

完全なコード例は上記の実装手順に含まれています。

---

## テスト方法

### 1. ローカル開発環境でのテスト

#### DynamoDBテーブル確認

```bash
# テーブルが作成されているか確認
aws dynamodb describe-table \
  --table-name TenantMaster-dev2 \
  --query "Table.{TableName:TableName,Status:TableStatus}" \
  --output json

# テナントデータを確認
aws dynamodb scan \
  --table-name TenantMaster-dev2 \
  --query "Items[*].{TenantId:tenantId.S,TenantName:tenantName.S,Enabled:enabled.BOOL}" \
  --output table
```

#### API エンドポイントテスト

```bash
# 1. ログイン後、ID tokenを取得
ID_TOKEN="<Cognitoから取得したID Token>"

# 2. ユーザープロファイル取得
curl -X GET \
  "https://your-api-endpoint.execute-api.ap-northeast-1.amazonaws.com/api/user/profile" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  | jq .

# 期待される出力:
# {
#   "userId": "s.abe@churadata.okinawa",
#   "departments": ["engineering", "sales"],
#   "tenants": [
#     {
#       "tenantId": "tenant-engineering",
#       "tenantName": "技術部門ポータル",
#       "departmentId": "engineering"
#     },
#     {
#       "tenantId": "tenant-sales",
#       "tenantName": "営業部門ポータル",
#       "departmentId": "sales"
#     }
#   ],
#   "defaultTenantId": "tenant-engineering"
# }

# 3. テナント設定取得
curl -X GET \
  "https://your-api-endpoint.execute-api.ap-northeast-1.amazonaws.com/api/tenant/tenant-engineering/config" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  | jq '.config.ui.branding'

# 4. テナントアクセス検証
curl -X POST \
  "https://your-api-endpoint.execute-api.ap-northeast-1.amazonaws.com/api/tenant/tenant-sales/validate" \
  -H "Authorization: Bearer ${ID_TOKEN}"
```

### 2. フロントエンドテスト

#### テストシナリオ1: 初回ログイン

1. アプリケーションにアクセス
2. EntraIDでログイン
3. **期待される動作**:
   - デフォルトテナント（engineering優先）でアプリ初期化
   - ブランディング（色、ロゴ）が適用される
   - テナント切り替えトグルが表示される
   - ユーザーメールが表示される

#### テストシナリオ2: テナント切り替え

1. テナント切り替えトグルをクリック
2. 別のテナント（例: 営業部門）を選択
3. **期待される動作**:
   - アプリケーションがリロードされる
   - 新しいテナントのブランディングが適用される
   - 異なるKnowledge Baseが使用される
   - 異なるシステムプロンプトで動作する

#### テストシナリオ3: RAGクエリ

1. テナント: 技術部門
2. 質問: 「技術ドキュメントについて教えて」
3. **期待される動作**:
   - Knowledge Base: kb-engineering-docs
   - Filter: department = engineering
   - System Prompt: 技術部門向けプロンプト
   - Temperature: 0.3

4. テナント切り替え: 営業部門
5. 質問: 「営業戦略について教えて」
6. **期待される動作**:
   - Knowledge Base: kb-sales-docs
   - Filter: department = sales
   - System Prompt: 営業部門向けプロンプト
   - Temperature: 0.7

### 3. 自動テスト

```typescript
// packages/web/src/contexts/__tests__/TenantContext.test.tsx
import { renderHook, act, waitFor } from '@testing-library/react';
import { TenantProvider, useTenant } from '../TenantContext';

describe('TenantContext', () => {
  it('should initialize with default tenant', async () => {
    const wrapper = ({ children }) => <TenantProvider>{children}</TenantProvider>;

    const { result } = renderHook(() => useTenant(), { wrapper });

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.currentTenant).not.toBeNull();
    expect(result.current.currentTenant?.tenantId).toBe('tenant-engineering');
  });

  it('should switch tenant successfully', async () => {
    const wrapper = ({ children }) => <TenantProvider>{children}</TenantProvider>;

    const { result } = renderHook(() => useTenant(), { wrapper });

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    await act(async () => {
      await result.current.switchTenant('tenant-sales');
    });

    expect(result.current.currentTenant?.tenantId).toBe('tenant-sales');
  });

  it('should check feature flags correctly', async () => {
    const wrapper = ({ children }) => <TenantProvider>{children}</TenantProvider>;

    const { result } = renderHook(() => useTenant(), { wrapper });

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.isFeatureEnabled('chatEnabled')).toBe(true);
    expect(result.current.isFeatureEnabled('imageGenerationEnabled')).toBe(false);
  });
});
```

---

## トラブルシューティング

### 問題1: テナント設定が取得できない

**症状**:
- API呼び出しで404エラー
- Console: "Tenant not found"

**原因と対策**:
```bash
# 1. テーブルにデータが存在するか確認
aws dynamodb get-item \
  --table-name TenantMaster-dev2 \
  --key '{"tenantId":{"S":"tenant-engineering"}}' \
  | jq .

# 2. データがない場合はシードデータ投入
cd packages/cdk/scripts
chmod +x seed-tenant-data.sh
./seed-tenant-data.sh

# 3. Lambda関数の環境変数を確認
aws lambda get-function-configuration \
  --function-name GetTenantConfig \
  --query "Environment.Variables.TENANT_TABLE_NAME" \
  --output text
```

### 問題2: アクセス権限エラー

**症状**:
- API呼び出しで403エラー
- "Access denied to this tenant"

**原因と対策**:
```bash
# 1. ユーザーのJWT tokenを確認
# ID tokenの内容をデコード: https://jwt.io/
# custom:samlGroups に "Engineering" または "Sales" が含まれているか確認

# 2. テナントのallowedDepartmentsを確認
aws dynamodb get-item \
  --table-name TenantMaster-dev2 \
  --key '{"tenantId":{"S":"tenant-engineering"}}' \
  --query "Item.allowedDepartments" \
  --output json

# 3. allowedDepartmentsを修正
aws dynamodb update-item \
  --table-name TenantMaster-dev2 \
  --key '{"tenantId":{"S":"tenant-engineering"}}' \
  --update-expression "SET allowedDepartments = :depts" \
  --expression-attribute-values '{":depts":{"L":[{"S":"engineering"},{"S":"devops"}]}}'
```

### 問題3: UIテーマが適用されない

**症状**:
- テナント切り替え後も色が変わらない
- ロゴが表示されない

**原因と対策**:
```typescript
// 1. ブラウザのDevToolsでCSS変数を確認
console.log(
  getComputedStyle(document.documentElement).getPropertyValue('--tenant-primary-color')
);

// 2. TenantContextのapplyTheme関数にログを追加
const applyTheme = (uiConfig: TenantConfig['config']['ui']) => {
  console.log('[applyTheme] Applying theme:', uiConfig.branding);
  // ... rest of function
};

// 3. localStorage確認
console.log('Selected tenant:', localStorage.getItem('selected_tenant'));
console.log('Tenant RAG config:', localStorage.getItem('tenant_rag_config'));
```

### 問題4: テナント切り替え後にRAG設定が古いまま

**症状**:
- テナント切り替え後も古いKnowledge Baseが使われる

**原因と対策**:
```typescript
// 1. predictLambdaでテナント設定取得を確認
console.log('[predict] Request extraData:', body.extraData);
console.log('[predict] Tenant config loaded:', tenantConfig);

// 2. localStorageの古いデータをクリア
localStorage.removeItem('tenant_rag_config');
localStorage.removeItem('tenant_llm_config');

// 3. ページをリロード
window.location.reload();

// 4. または、switchTenant内でリロードを確認
const switchTenant = async (newTenantId: string) => {
  // ...
  await initializeTenant(newTenantId);

  // 必ずリロード
  window.location.reload();
};
```

### 問題5: Lambda関数のDynamoDBアクセスエラー

**症状**:
- CloudWatch Logsに "AccessDeniedException"

**原因と対策**:
```typescript
// 1. Lambda関数のIAMロールを確認
aws lambda get-function \
  --function-name GetTenantConfig \
  --query "Configuration.Role" \
  --output text

# 2. IAMロールのポリシーを確認
aws iam list-attached-role-policies \
  --role-name <RoleName>

# 3. CDKでDynamoDB権限を付与
props.tenantMasterTable.grantReadData(getTenantConfigFunction);
props.tenantMasterTable.grantReadData(getUserProfileFunction);
props.tenantMasterTable.grantReadData(validateTenantAccessFunction);

# 4. 再デプロイ
npm run cdk:deploy
```

---

## ベストプラクティス

### 1. テナント設定の管理

#### 推奨: DynamoDB + 管理UI

```typescript
// 将来実装: テナント管理画面
const TenantAdminPage: React.FC = () => {
  const [tenants, setTenants] = useState<TenantConfig[]>([]);

  const updateTenant = async (tenantId: string, config: Partial<TenantConfig>) => {
    await fetch(`/api/admin/tenant/${tenantId}`, {
      method: 'PUT',
      body: JSON.stringify(config),
    });
  };

  return (
    <div>
      <h1>テナント管理</h1>
      {tenants.map((tenant) => (
        <TenantConfigEditor
          key={tenant.tenantId}
          tenant={tenant}
          onSave={(config) => updateTenant(tenant.tenantId, config)}
        />
      ))}
    </div>
  );
};
```

#### 非推奨: ハードコード

```typescript
// ❌ 悪い例: 設定をコードにハードコード
const TENANT_CONFIG = {
  'tenant-engineering': {
    primaryColor: '#1E3A8A',
    // ...
  },
};

// ✅ 良い例: DynamoDBから動的に取得
const tenantConfig = await getTenantConfig(tenantId);
```

### 2. パフォーマンス最適化

#### キャッシュ戦略

```typescript
// TenantContextでキャッシュを実装
const tenantConfigCache = new Map<string, TenantConfig>();

const initializeTenant = async (tenantId: string) => {
  // キャッシュチェック
  if (tenantConfigCache.has(tenantId)) {
    const cached = tenantConfigCache.get(tenantId)!;
    setCurrentTenant(cached);
    applyTheme(cached.config.ui);
    return;
  }

  // APIから取得
  const config = await fetch(`/api/tenant/${tenantId}/config`).then(r => r.json());

  // キャッシュに保存
  tenantConfigCache.set(tenantId, config);

  setCurrentTenant(config);
  applyTheme(config.ui);
};
```

### 3. セキュリティ

#### JWT token検証

```typescript
// Lambda関数で必ずJWT tokenを検証
export const handler = async (event: APIGatewayProxyEvent) => {
  // Cognito Authorizerが自動検証するため、claimsは信頼できる
  const claims = event.requestContext.authorizer?.claims;

  if (!claims) {
    return {
      statusCode: 401,
      body: JSON.stringify({ error: 'Unauthorized' }),
    };
  }

  // ユーザーの部門を取得
  const userDepartments = getUserDepartments(claims);

  // テナントへのアクセス権限を検証
  const hasAccess = tenant.allowedDepartments.some(dept =>
    userDepartments.includes(dept)
  );

  if (!hasAccess) {
    return {
      statusCode: 403,
      body: JSON.stringify({ error: 'Access denied' }),
    };
  }

  // 処理続行...
};
```

#### フロントエンドでの検証

```typescript
// フロントエンドでも基本的な検証を実施
const useTenant = () => {
  // ...

  const switchTenant = async (newTenantId: string) => {
    // まずバックエンドでアクセス権限を検証
    const validateResponse = await fetch(`/api/tenant/${newTenantId}/validate`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${idToken}`,
      },
    });

    if (!validateResponse.ok) {
      throw new Error('Access denied');
    }

    // 検証通過後にテナント切り替え
    await initializeTenant(newTenantId);
  };

  // ...
};
```

### 4. モニタリング

#### CloudWatch Logs Insights クエリ

```sql
-- テナント別アクセス頻度
fields @timestamp, tenantId, userEmail, action
| filter action = "tenant_switch"
| stats count() by tenantId
| sort count desc

-- テナント別エラー率
fields @timestamp, tenantId, error
| filter error != ""
| stats count() by tenantId, error
| sort count desc

-- テナント別レスポンス時間
fields @timestamp, tenantId, @duration
| filter action = "get_tenant_config"
| stats avg(@duration), max(@duration), min(@duration) by tenantId
```

#### X-Ray トレーシング

```typescript
// Lambda関数でX-Rayトレーシングを有効化
export const handler = async (event: APIGatewayProxyEvent) => {
  const segment = AWSXRay.getSegment();
  const subsegment = segment?.addNewSubsegment('getTenantConfig');

  subsegment?.addAnnotation('tenantId', tenantId);
  subsegment?.addMetadata('userDepartments', userDepartments);

  try {
    const config = await getTenantConfig(tenantId);
    subsegment?.close();
    return { statusCode: 200, body: JSON.stringify(config) };
  } catch (error) {
    subsegment?.addError(error);
    subsegment?.close();
    throw error;
  }
};
```

### 5. デプロイ戦略

#### Blue/Greenデプロイメント

```bash
# 1. 新しいテナント設定をステージング環境で検証
aws dynamodb put-item \
  --table-name TenantMaster-staging \
  --item file://tenant-engineering-v2.json

# 2. ステージング環境でテスト
npm run test:e2e -- --env=staging

# 3. 本番環境にデプロイ
aws dynamodb put-item \
  --table-name TenantMaster-prod \
  --item file://tenant-engineering-v2.json

# 4. 問題があれば即座にロールバック
aws dynamodb put-item \
  --table-name TenantMaster-prod \
  --item file://tenant-engineering-v1-backup.json
```

---

## まとめ

このマルチテナント実装により、以下が実現されます：

✅ **1つのコードベースで複数のテナント向けアプリケーションを提供**
- テナント毎に異なるUI、RAG設定、LLM設定
- 部門切り替えトグルによる即座の切り替え

✅ **シンプルで保守性の高い実装**
- React Context（約150行）で全体を管理
- 新規ページ追加時の追加コードは0行

✅ **セキュリティとパフォーマンス**
- JWT token ベースのアクセス制御
- API呼び出し1回でテナント設定を取得
- キャッシュ戦略によるパフォーマンス最適化

✅ **将来のLLMゲートウェイ拡張に対応**
- テナント毎のモデル制限、レート制限、予算管理
- DynamoDB設計により動的な設定変更が可能

この実装により、エンタープライズグレードのマルチテナントAIアプリケーションを効率的に構築・運用できます。
