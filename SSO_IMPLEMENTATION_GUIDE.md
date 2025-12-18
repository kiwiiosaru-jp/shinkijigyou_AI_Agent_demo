# Azure Entra ID SAML SSO Implementation Guide

## 目次
1. [概要](#概要)
2. [アーキテクチャ](#アーキテクチャ)
3. [実装の詳細](#実装の詳細)
4. [Azure Entra ID 設定手順](#azure-entra-id-設定手順)
5. [AWS Cognito 設定](#aws-cognito-設定)
6. [CDK実装](#cdk実装)
7. [フロントエンド実装](#フロントエンド実装)
8. [トラブルシューティング](#トラブルシューティング)

---

## 概要

このドキュメントでは、Azure Entra ID（旧Azure AD）とAWS CognitoのSAML連携による、エンタープライズグレードのSSO（Single Sign-On）実装について説明します。

### 主要機能
- **SAML 2.0 認証**: Azure Entra IDを使用したシングルサインオン
- **セキュリティグループ連携**: Entra IDのセキュリティグループをCognito属性にマッピング
- **動的グループ名変換**: Graph APIを使用したグループObject IDから表示名への変換
- **部門ベースアクセス制御**: RAGナレッジベースへの部門別アクセス制限
- **マルチテナント対応**: localStorage基盤の部門切り替え機能
- **強制再認証**: 毎回Entra IDログイン画面を表示

---

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Browser                              │
├─────────────────────────────────────────────────────────────────┤
│  1. Click "Login" → 2. Redirect to Cognito Hosted UI            │
│  3. Redirect to Entra ID SAML endpoint                           │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Entra ID                                │
├─────────────────────────────────────────────────────────────────┤
│  4. User authenticates with Entra ID credentials                │
│  5. Entra ID returns SAML Response with:                         │
│     - email                                                      │
│     - givenName, surname                                         │
│     - groups (Object IDs: 423e6dc2-..., 4b3610d2-...)           │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────────┐
│                   AWS Cognito User Pool                          │
├─────────────────────────────────────────────────────────────────┤
│  6. SAML Response received                                       │
│  7. Attributes mapped to Cognito attributes:                     │
│     - email → email                                              │
│     - custom:samlGroups → "423e6dc2-...,4b3610d2-..."           │
│                                                                  │
│  8. Pre-Token Generation Lambda Trigger                          │
│     ┌──────────────────────────────────────────────────────┐   │
│     │  mapSamlGroups Lambda                                 │   │
│     │  - Get Azure Graph API access token                   │   │
│     │  - For each group Object ID:                          │   │
│     │    GET /groups/{id} → displayName                     │   │
│     │  - Replace custom:samlGroups with:                    │   │
│     │    "Sales,Engineering"                                │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                  │
│  9. ID Token generated with custom:samlGroups="Sales,Engineering"│
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend Application                          │
├─────────────────────────────────────────────────────────────────┤
│ 10. Store tokens in localStorage                                │
│ 11. Display user email in header (from ID token)                │
│ 12. User selects department → store in localStorage             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────────────┐
│                API Gateway + Lambda Functions                    │
├─────────────────────────────────────────────────────────────────┤
│  [GET /user/department]                                          │
│  - Extract custom:samlGroups from ID token                       │
│  - Return available departments and default                      │
│                                                                  │
│  [POST /user/department]                                         │
│  - Validate user has access to requested department             │
│  - Return success (no Cognito update)                            │
│                                                                  │
│  [POST /predict, /predictStream]                                 │
│  - Extract custom:samlGroups from ID token                       │
│  - Get department from localStorage (via extraData)              │
│  - Apply department filter to Knowledge Base query               │
│  - Filter results by department metadata                         │
└─────────────────────────────────────────────────────────────────┘
```

### データフロー

#### 1. 認証フロー
```
User → Cognito → Entra ID → SAML Response → Cognito
→ Pre-Token Generation Lambda → Graph API
→ ID Token (custom:samlGroups="Sales,Engineering")
```

#### 2. 部門選択フロー
```
User selects department in UI
→ POST /user/department (validation only, no Cognito update)
→ Store in localStorage: 'selected_department' = 'engineering'
→ JWT token remains unchanged (custom:samlGroups="Sales,Engineering")
→ Used in RAG queries as extraData
```

**重要**:
- ✅ JWTトークンは変更されない
- ✅ ページリロードは不要
- ✅ LocalStorageに保存されるのみ
- ✅ RAGクエリ時にextraDataとして送信

#### 3. RAG クエリフロー
```
User submits query
→ Frontend: Get department from localStorage
→ Frontend: Create extraData with department filter
→ POST to predict/predictStream Lambda
→ Lambda: Extract custom:samlGroups from JWT token (default)
→ Lambda: Override with extraData department (if provided)
→ Lambda: Apply filter to Knowledge Base query
→ Knowledge Base: Filter documents by department metadata
→ Return filtered results to user
```

**フィルタ優先順位**:
1. **ExtraData filter** (from localStorage) - 最優先
2. **Dynamic filter** (from JWT token custom:samlGroups) - デフォルト
3. **Hidden static filter** (application-level) - 常に適用

---

## 実装の詳細

### 1. SAML グループ属性マッピング

#### Entra ID → Cognito マッピング
```
http://schemas.microsoft.com/ws/2008/06/identity/claims/groups
↓
custom:samlGroups (カンマ区切り文字列)
```

#### Object ID → 表示名変換
Pre-Token Generation Lambdaで変換:
```typescript
// Input: "423e6dc2-dd7b-4bef-a3eb-36f1fb60473c,4b3610d2-9c9e-4c54-85bc-eebb8749535f"
// Output: "Sales,Engineering"
```

**実装ファイル**: `packages/cdk/lambda/mapSamlGroups.ts`

**処理フロー**:
1. Azure Service Principal認証情報を環境変数から取得
2. Microsoft Graph API アクセストークンを取得
3. 各グループObject IDに対して `GET /groups/{id}` を実行
4. displayName を抽出
5. カンマ区切りで結合して `custom:samlGroups` を上書き

**必要な権限**:
- Service Principal: `Group.Read.All` (Application permission)

---

### 2. 部門ベースアクセス制御

#### データモデル

**DepartmentMaster テーブル (DynamoDB)**:
```typescript
{
  departmentId: 'engineering',      // PK
  cognitoName: 'Engineering',       // Cognito group name (legacy)
  labelJa: '技術部',                // Japanese label
  labelEn: 'Engineering',           // English label
  enabled: 'true',                  // GSI PK
  sortOrder: 1,                     // GSI SK
  createdAt: '2025-01-01T00:00:00Z',
  updatedAt: '2025-01-01T00:00:00Z'
}
```

**GSI: EnabledSortOrderIndex**:
- PK: `enabled`
- SK: `sortOrder`
- 有効な部門をsortOrder順で取得

#### 部門取得 API

**エンドポイント**: `GET /user/department`

**実装**: `packages/cdk/lambda/getUserDepartment.ts`

**ロジック**:
```typescript
// 1. ID トークンから custom:samlGroups を抽出
const samlGroups = claims['custom:samlGroups']; // "Sales,Engineering"

// 2. カンマ区切りで分割
const departments = samlGroups.split(',').map(g => g.trim().toLowerCase());

// 3. デフォルト部門を決定（engineeringを優先）
const defaultDept = departments.includes('engineering') ? 'engineering' : departments[0];

// 4. レスポンス
return {
  department: defaultDept,
  availableDepartments: departments
};
```

#### 部門切り替え API

**エンドポイント**: `POST /user/department`

**リクエスト**:
```json
{
  "department": "engineering"
}
```

**実装**: `packages/cdk/lambda/updateUserDepartment.ts`

**ロジック**:
```typescript
// 1. DynamoDB から部門情報を取得
const departmentItem = await dynamodb.get({
  TableName: DEPARTMENT_MASTER_TABLE,
  Key: { departmentId: 'engineering' }
});

// 2. 部門が有効か確認
if (departmentItem.enabled !== 'true') {
  return 400; // Department disabled
}

// 3. ユーザーがその部門に所属しているか確認
const userDepartments = extractFromToken(claims['custom:samlGroups']);
if (!userDepartments.includes('engineering')) {
  return 403; // Access denied
}

// 4. 検証成功（Cognitoは更新しない）
return {
  message: 'Department selection validated successfully',
  department: 'engineering',
  label: '技術部'
};
```

**重要ポイント**:
- ✅ このAPIはCognitoのユーザー属性を**更新しません**
- ✅ JWTトークンを**リフレッシュしません**
- ✅ 検証のみを行い、フロントエンドがlocalStorageに保存
- ✅ RAGクエリ時にlocalStorageの値をextraDataとして送信
- ✅ ユーザーは複数部門に所属可能（例: Engineering, Sales）
- ✅ UI上で部門を切り替えても、JWTトークンの`custom:samlGroups`は変更されない

---

### 3. RAG Knowledge Base フィルタリング

#### 動的フィルタの適用

**実装**: `packages/common/src/custom/rag-knowledge-base.ts`

**getDynamicFilters関数**:
```typescript
export const getDynamicFilters = (
  idTokenPayload: CognitoIdTokenPayload | undefined
): RetrievalFilter[] => {
  let department: string | null = null;

  // Method 1: SAML ユーザー (custom:samlGroups)
  const samlGroups = idTokenPayload['custom:samlGroups'];
  if (samlGroups) {
    const groupList = samlGroups.split(',').map(g => g.trim());
    // Prefer "engineering" if it exists
    department = groupList.includes('engineering')
      ? 'engineering'
      : groupList[0].toLowerCase();
  }

  // Method 2: ネイティブCognitoユーザー (cognito:groups)
  if (!department) {
    const groups = idTokenPayload['cognito:groups'];
    // Extract from "Engineering-Admin" → "engineering"
    department = extractDepartmentFromGroupName(groups);
  }

  // Apply filter
  return department ? [{
    equals: {
      key: 'department',
      value: department
    }
  }] : [];
};
```

#### UI フィルタオーバーライド

**実装**: `packages/web/src/pages/RagAuroraKnowledgeBasePage.tsx`

**重要**: ユーザーがUI上で部門を切り替えた場合の動作:

1. ✅ **JWTトークンは変更されない**
   - `custom:samlGroups` は "Engineering,Sales" のまま
   - トークンリフレッシュは行われない

2. ✅ **LocalStorageに部門IDを保存**
   - `localStorage.setItem('selected_department', 'engineering')`

3. ✅ **RAGクエリ時にextraDataとして送信**
   - LocalStorageの値がバックエンドに送られる
   - バックエンドでextraDataのフィルタが優先される

ユーザーがUI上で部門を切り替えた場合、localStorageの値を使用してフィルタを上書き:

```typescript
// 1. LocalStorageから部門を取得
const department = localStorage.getItem('selected_department');

// 2. extraDataとしてバックエンドに送信
const extraData: ExtraData[] = [
  {
    type: 'json',
    name: 'departmentFilter',
    source: {
      type: 'inline',
      mediaType: 'application/json',
      data: JSON.stringify({
        equals: {
          key: 'department',
          value: department
        }
      })
    }
  }
];

// 3. バックエンドでextraDataのフィルタを優先適用
```

**バックエンド処理** (`packages/cdk/lambda/utils/bedrockKbApi.ts`):
```typescript
// 1. Dynamic filter (from token)
const dynamicFilters = getDynamicFilters(payload);

// 2. User-defined explicit filter (from extraData)
const userFilters = extractFromExtraData(messages);

// 3. Merge filters (user filter overrides dynamic filter)
const finalFilters = mergeDepartmentFilters(dynamicFilters, userFilters);
```

---

### 4. 強制再認証の実装

**目的**: 毎回Entra IDのログイン画面を表示し、自動ログインを防止

**実装**: `packages/web/src/components/AuthWithSAML.tsx`

```typescript
const signIn = async () => {
  try {
    // 1. 既存のCognitoセッションをクリア
    await signOut({ global: false });
  } catch (error) {
    console.log('No existing session to clear');
  }

  // 2. LocalStorageからすべてのAmplify/Cognito関連データを削除
  Object.keys(localStorage).forEach(key => {
    if (key.startsWith('CognitoIdentityServiceProvider') ||
        key.startsWith('aws-amplify-') ||
        key.includes('amplify')) {
      localStorage.removeItem(key);
    }
  });

  // 3. SessionStorageをクリア
  sessionStorage.clear();

  // 4. クリーンアップ完了を待つ
  await new Promise(resolve => setTimeout(resolve, 100));

  // 5. Entra IDにリダイレクト
  signInWithRedirect({
    provider: {
      custom: 'EntraID'
    }
  });
};
```

**重要ポイント**:
- `signOut({ global: false })`: Cognito User Poolからのみサインアウト（Entra IDセッションは保持）
- LocalStorageクリア: Amplifyのトークンキャッシュを削除
- SessionStorageクリア: 一時認証データを削除

---

## Azure Entra ID 設定手順

### 前提条件
- Azure サブスクリプション
- Global Administrator または Application Administrator 権限
- Azure CLI インストール済み

### 1. Azure Service Principal の作成

Service Principal は Graph API を使用してグループ情報を取得するために必要です。

```bash
# Azure にログイン
az login --tenant 1548244c-78cc-4075-b367-753f7d6096dc

# Service Principal を作成
az ad sp create-for-rbac \
  --name "GenU-GAAB-GraphAPI-Reader" \
  --role Reader \
  --scopes /subscriptions/{subscription-id}

# 出力例:
# {
#   "appId": "12345678-1234-1234-1234-123456789abc",
#   "displayName": "GenU-GAAB-GraphAPI-Reader",
#   "password": "your-client-secret",
#   "tenant": "1548244c-78cc-4075-b367-753f7d6096dc"
# }
```

**重要**: `appId` (Client ID)、`password` (Client Secret)、`tenant` (Tenant ID) を保存してください。

### 2. Graph API 権限の付与

```bash
# Application ID を取得
APP_ID="12345678-1234-1234-1234-123456789abc"

# Microsoft Graph の Resource ID を取得
GRAPH_RESOURCE_ID=$(az ad sp list --query "[?appId=='00000003-0000-0000-c000-000000000000'].id" -o tsv)

# Group.Read.All 権限の ID を取得
GROUP_READ_ALL_ID="5b567255-7703-4780-807c-7be8301ae99b"

# Application Permission を付与
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$APP_ID/appRoleAssignments" \
  --body "{
    \"principalId\": \"$APP_ID\",
    \"resourceId\": \"$GRAPH_RESOURCE_ID\",
    \"appRoleId\": \"$GROUP_READ_ALL_ID\"
  }" \
  --headers "Content-Type=application/json"

# Admin Consent を付与（Global Administrator が実行）
az ad app permission admin-consent --id $APP_ID
```

### 3. SAML エンタープライズアプリケーションの作成

#### 3.1 アプリケーション登録

```bash
# アプリケーションを作成
az ad app create \
  --display-name "GenU GAAB SAML SSO" \
  --identifier-uris "urn:amazon:cognito:sp:ap-northeast-1_u8W3FROzJ" \
  --web-redirect-uris "https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/saml2/idpresponse"
```

#### 3.2 Service Principal の作成

```bash
# Application ID を取得
APP_ID=$(az ad app list --display-name "GenU GAAB SAML SSO" --query "[0].appId" -o tsv)

# Service Principal を作成
az ad sp create --id $APP_ID

# Service Principal Object ID を取得
SP_OBJECT_ID=$(az ad sp list --filter "appId eq '$APP_ID'" --query "[0].id" -o tsv)
```

#### 3.3 SAML 設定

```bash
# SAML Single Sign-On モードを有効化
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$SP_OBJECT_ID" \
  --body '{"preferredSingleSignOnMode":"saml"}' \
  --headers "Content-Type=application/json"

# Application の設定を更新
az rest --method PATCH \
  --uri "https://graph.microsoft.com/v1.0/applications/$APP_OBJECT_ID" \
  --body '{
    "identifierUris": ["urn:amazon:cognito:sp:ap-northeast-1_u8W3FROzJ"],
    "web": {
      "redirectUris": [
        "https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/saml2/idpresponse"
      ],
      "logoutUrl": "https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/logout"
    },
    "groupMembershipClaims": "SecurityGroup"
  }' \
  --headers "Content-Type=application/json"
```

**重要**: `groupMembershipClaims` を `"SecurityGroup"` に設定することで、SAML ResponseにグループObject IDが含まれます。

#### 3.4 SAML トークン属性マッピング

Azure Portal で以下の属性マッピングを設定:
- **email**: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` → `user.mail`
- **givenname**: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname` → `user.givenname`
- **surname**: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname` → `user.surname`
- **groups**: `http://schemas.microsoft.com/ws/2008/06/identity/claims/groups` → `user.groups`

### 4. セキュリティグループの作成

```bash
# Engineering グループを作成
az ad group create \
  --display-name "Engineering" \
  --mail-nickname "engineering" \
  --description "Engineering Department (Microsoft 365 Group for SAML SSO)"

# Sales グループを作成
az ad group create \
  --display-name "Sales" \
  --mail-nickname "sales" \
  --description "Sales Department (Microsoft 365 Group for SAML SSO)"

# グループ Object ID を取得
ENGINEERING_GROUP_ID=$(az ad group list --filter "displayName eq 'Engineering'" --query "[0].id" -o tsv)
SALES_GROUP_ID=$(az ad group list --filter "displayName eq 'Sales'" --query "[0].id" -o tsv)

echo "Engineering Group ID: $ENGINEERING_GROUP_ID"
echo "Sales Group ID: $SALES_GROUP_ID"
```

### 5. ユーザーをグループに追加

```bash
# ユーザー Object ID を取得
USER_ID=$(az ad user show --id "s.abe@churadata.okinawa" --query "id" -o tsv)

# Engineering グループに追加
az ad group member add \
  --group $ENGINEERING_GROUP_ID \
  --member-id $USER_ID

# Sales グループに追加
az ad group member add \
  --group $SALES_GROUP_ID \
  --member-id $USER_ID

# 確認
az ad group member list \
  --group $ENGINEERING_GROUP_ID \
  --query "[].{DisplayName:displayName,Email:mail}" \
  --output table
```

### 6. アプリケーションへのユーザー割り当て

```bash
# Service Principal Object ID を取得
SP_OBJECT_ID=$(az ad sp list --filter "appId eq '$APP_ID'" --query "[0].id" -o tsv)

# ユーザーを割り当て
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$SP_OBJECT_ID/appRoleAssignedTo" \
  --body "{
    \"principalId\": \"$USER_ID\",
    \"resourceId\": \"$SP_OBJECT_ID\",
    \"appRoleId\": \"00000000-0000-0000-0000-000000000000\"
  }" \
  --headers "Content-Type=application/json"

# グループを割り当て（推奨）
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$SP_OBJECT_ID/appRoleAssignedTo" \
  --body "{
    \"principalId\": \"$ENGINEERING_GROUP_ID\",
    \"resourceId\": \"$SP_OBJECT_ID\",
    \"appRoleId\": \"00000000-0000-0000-0000-000000000000\"
  }" \
  --headers "Content-Type=application/json"
```

### 7. SAML メタデータ URL の取得

```bash
TENANT_ID="1548244c-78cc-4075-b367-753f7d6096dc"
APP_ID="ec8ec894-2820-411c-8ef1-04d2dfd3a907"

METADATA_URL="https://login.microsoftonline.com/$TENANT_ID/federationmetadata/2007-06/federationmetadata.xml?appid=$APP_ID"

echo "SAML Metadata URL: $METADATA_URL"
```

このURLをAWS Cognito設定で使用します。

### 8. AWS Secrets Manager に認証情報を保存

```bash
aws secretsmanager create-secret \
  --name generative-ai-use-cases/dev2/azure-graph-credentials \
  --description "Azure Graph API credentials for SAML group mapping" \
  --secret-string '{
    "tenantId": "1548244c-78cc-4075-b367-753f7d6096dc",
    "clientId": "12345678-1234-1234-1234-123456789abc",
    "clientSecret": "your-client-secret-here"
  }' \
  --region ap-northeast-1
```

**セキュリティ注意**:
- Client Secret は安全に管理してください
- 定期的にローテーションしてください
- 最小権限の原則に従ってください

---

## AWS Cognito 設定

### 1. User Pool カスタム属性

以下のカスタム属性を定義:

| 属性名 | 型 | 説明 |
|--------|-----|------|
| `custom:department` | String | ユーザーの部門（レガシー・未使用） |
| `custom:accessLevel` | String | アクセスレベル（未使用） |
| `custom:samlGroups` | String | SAMLグループ（カンマ区切り） |

**CDK定義**:
```typescript
const userPool = new UserPool(this, 'UserPool', {
  customAttributes: {
    department: new StringAttribute({ mutable: true }),
    accessLevel: new StringAttribute({ mutable: true }),
    samlGroups: new StringAttribute({ mutable: true }),
  }
});
```

### 2. SAML Identity Provider

**CDK定義** (`packages/cdk/lib/construct/auth.ts`):
```typescript
const samlProvider = new UserPoolIdentityProviderSaml(this, 'EntraIdSAML', {
  userPool,
  name: 'EntraID',
  metadata: UserPoolIdentityProviderSamlMetadata.url(props.samlMetadataUrl),
  attributeMapping: {
    email: {
      attributeName: 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress',
    },
    givenName: {
      attributeName: 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname',
    },
    familyName: {
      attributeName: 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname',
    },
    'custom:samlGroups': {
      attributeName: 'http://schemas.microsoft.com/ws/2008/06/identity/claims/groups',
    } as any,
  },
});
```

### 3. User Pool Client

```typescript
const client = userPool.addClient('client', {
  idTokenValidity: Duration.days(1),
  supportedIdentityProviders: [
    UserPoolClientIdentityProvider.custom('EntraID'),
  ],
  oAuth: {
    callbackUrls: ['https://d2b6nxic7zvmb4.cloudfront.net'],
    logoutUrls: ['https://d2b6nxic7zvmb4.cloudfront.net'],
    scopes: ['openid', 'email', 'profile'],
    flows: {
      authorizationCodeGrant: true,
    }
  },
});
```

### 4. Cognito Domain

```typescript
const domain = new UserPoolDomain(this, 'UserPoolDomain', {
  userPool,
  cognitoDomain: {
    domainPrefix: 'genu-gaab-sso-dev2',
  },
});
```

**生成されるURL**:
- **Login**: `https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/oauth2/authorize`
- **Logout**: `https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/logout`
- **SAML ACS**: `https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/saml2/idpresponse`

### 5. Pre-Token Generation Lambda Trigger

```typescript
if (props.samlAuthEnabled) {
  const azureGraphSecret = Secret.fromSecretNameV2(
    this,
    'AzureGraphSecret',
    'generative-ai-use-cases/dev2/azure-graph-credentials'
  );

  const mapSamlGroupsFunction = new NodejsFunction(
    this,
    'MapSamlGroups',
    {
      runtime: LAMBDA_RUNTIME_NODEJS,
      entry: path.join(__dirname, '../../lambda/mapSamlGroups.ts'),
      timeout: Duration.seconds(30),
      environment: {
        AZURE_TENANT_ID: azureGraphSecret
          .secretValueFromJson('tenantId')
          .unsafeUnwrap(),
        AZURE_CLIENT_ID: azureGraphSecret
          .secretValueFromJson('clientId')
          .unsafeUnwrap(),
        AZURE_CLIENT_SECRET: azureGraphSecret
          .secretValueFromJson('clientSecret')
          .unsafeUnwrap(),
      },
    }
  );

  azureGraphSecret.grantRead(mapSamlGroupsFunction);

  userPool.addTrigger(
    UserPoolOperation.PRE_TOKEN_GENERATION,
    mapSamlGroupsFunction
  );
}
```

---

## CDK実装

### 現在の設定確認

```bash
cd /Users/shigeru.abe/GAAB/GenU/generative-ai-use-cases/packages/cdk

# cdk.jsonの確認
cat cdk.json
```

### デプロイコマンド

```bash
# Full deployment
npm run cdk:deploy

# Quick deployment (skip asset build)
npm run cdk:deploy:quick

# Specific stack
npx cdk deploy GenerativeAiUseCasesStackdev2
```

### 環境変数設定

**cdk.json** または **環境変数**:
```json
{
  "context": {
    "samlAuthEnabled": "true",
    "samlMetadataUrl": "https://login.microsoftonline.com/1548244c-78cc-4075-b367-753f7d6096dc/federationmetadata/2007-06/federationmetadata.xml?appid=ec8ec894-2820-411c-8ef1-04d2dfd3a907",
    "cognitoDomainPrefix": "genu-gaab-sso-dev2"
  }
}
```

### Lambda Functions

#### 1. mapSamlGroups.ts
- **トリガー**: Pre-Token Generation
- **目的**: グループObject IDをグループ表示名に変換
- **場所**: `packages/cdk/lambda/mapSamlGroups.ts`

#### 2. getUserDepartment.ts
- **エンドポイント**: `GET /user/department`
- **目的**: ユーザーの利用可能部門とデフォルト部門を返す
- **場所**: `packages/cdk/lambda/getUserDepartment.ts`

#### 3. updateUserDepartment.ts
- **エンドポイント**: `POST /user/department`
- **目的**: 部門切り替えリクエストを検証
- **場所**: `packages/cdk/lambda/updateUserDepartment.ts`

#### 4. listDepartments.ts
- **エンドポイント**: `GET /departments/list`
- **目的**: 有効な部門一覧を返す（DepartmentMasterから取得）
- **場所**: `packages/cdk/lambda/listDepartments.ts`

### DynamoDB テーブル

#### DepartmentMaster

**スキーマ**:
```typescript
{
  departmentId: string;      // PK
  cognitoName: string;
  labelJa: string;
  labelEn: string;
  enabled: string;           // GSI PK
  sortOrder: number;         // GSI SK
  createdAt: string;
  updatedAt: string;
}
```

**GSI: EnabledSortOrderIndex**:
- PK: `enabled`
- SK: `sortOrder`

**初期データ**:
```json
[
  {
    "departmentId": "engineering",
    "cognitoName": "Engineering",
    "labelJa": "技術部",
    "labelEn": "Engineering",
    "enabled": "true",
    "sortOrder": 1
  },
  {
    "departmentId": "sales",
    "cognitoName": "Sales",
    "labelJa": "営業部",
    "labelEn": "Sales",
    "enabled": "true",
    "sortOrder": 2
  }
]
```

---

## フロントエンド実装

### 1. 認証コンポーネント

**ファイル**: `packages/web/src/components/AuthWithSAML.tsx`

#### Amplify 設定
```typescript
useEffect(() => {
  if (!isAmplifyConfigured) {
    Amplify.configure({
      Auth: {
        Cognito: {
          userPoolId: import.meta.env.VITE_APP_USER_POOL_ID,
          userPoolClientId: import.meta.env.VITE_APP_USER_POOL_CLIENT_ID,
          identityPoolId: import.meta.env.VITE_APP_IDENTITY_POOL_ID,
          loginWith: {
            oauth: {
              domain: samlCognitoDomainName,
              scopes: ['openid', 'email', 'profile'],
              redirectSignIn: [window.location.origin],
              redirectSignOut: [window.location.origin],
              responseType: 'code',
            },
          },
        },
      },
    });
    isAmplifyConfigured = true;
  }
}, []);
```

#### ログイン処理
```typescript
const signIn = async () => {
  // 1. 既存セッションをクリア
  try {
    await signOut({ global: false });
  } catch (error) {
    console.log('No existing session');
  }

  // 2. LocalStorageクリア
  Object.keys(localStorage).forEach(key => {
    if (key.startsWith('CognitoIdentityServiceProvider') ||
        key.startsWith('aws-amplify-') ||
        key.includes('amplify')) {
      localStorage.removeItem(key);
    }
  });
  sessionStorage.clear();

  // 3. 待機
  await new Promise(resolve => setTimeout(resolve, 100));

  // 4. リダイレクト
  signInWithRedirect({
    provider: {
      custom: 'EntraID',
    },
  });
};
```

### 2. ユーザー情報表示

**ファイル**: `packages/web/src/components/UserInfo.tsx`

```typescript
const UserInfo: React.FC = () => {
  const [userEmail, setUserEmail] = useState<string>('');
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchUserInfo = async () => {
      try {
        const session = await fetchAuthSession();
        const idToken = session.tokens?.idToken;

        if (idToken && idToken.payload) {
          const email = idToken.payload.email as string;
          setUserEmail(email || '');
        }
      } catch (error) {
        console.error('Failed to fetch user info:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchUserInfo();
  }, []);

  if (loading || !userEmail) {
    return null;
  }

  return (
    <div className="flex items-center gap-2 text-sm">
      <PiUser className="text-base" />
      <span className="max-w-[150px] truncate" title={userEmail}>
        {userEmail}
      </span>
    </div>
  );
};
```

### 3. 部門管理フック

**ファイル**: `packages/web/src/hooks/useDepartment.ts`

```typescript
const useDepartment = () => {
  const [department, setDepartmentState] = useState<Department>(null);
  const [availableDepartments, setAvailableDepartments] = useState<string[]>([]);

  // 初期化: APIから部門情報を取得
  useEffect(() => {
    const fetchDepartment = async () => {
      // LocalStorageから取得
      const savedDepartment = localStorage.getItem('selected_department');

      // APIから利用可能部門を取得
      const response = await api.get('/user/department');
      const defaultDept = response.data.department;
      const available = response.data.availableDepartments;

      setAvailableDepartments(available);

      // Saved departmentが有効なら使用、なければdefault
      if (savedDepartment && available.includes(savedDepartment)) {
        setDepartmentState(savedDepartment);
      } else {
        setDepartmentState(defaultDept);
        localStorage.setItem('selected_department', defaultDept);
      }
    };

    fetchDepartment();
  }, []);

  // 部門切り替え
  const setDepartment = async (newDepartment: Department) => {
    // APIで検証
    await api.post('/user/department', {
      department: newDepartment,
    });

    // LocalStorageに保存
    localStorage.setItem('selected_department', newDepartment);

    // 状態更新
    setDepartmentState(newDepartment);
  };

  return {
    department,
    availableDepartments,
    setDepartment,
  };
};
```

### 4. RAG Knowledge Base ページ

**ファイル**: `packages/web/src/pages/RagAuroraKnowledgeBasePage.tsx`

#### 部門選択UI
```typescript
const RagAuroraKnowledgeBasePage: React.FC = () => {
  const { department, setDepartment, loading } = useDepartment();
  const [selectedDepartment, setSelectedDepartment] = useState<Department>(null);

  // 部門変更ハンドラ
  const handleDepartmentChange = async (newDept: Department) => {
    if (!newDept) return;

    try {
      // バックエンドで検証（Cognitoは更新しない）
      await setDepartment(newDept);

      // ローカル状態を更新
      setSelectedDepartment(newDept);

      // 成功メッセージ
      alert(`部門を ${DEPARTMENT_LABELS[newDept]} に切り替えました`);

      // ❌ JWTトークンはリフレッシュしない
      // ❌ ページリロードもしない
      // ✅ LocalStorageに保存された値がRAGクエリで使用される
      // await refreshTokenAndReload(); // 実行しない
    } catch (error) {
      console.error('Failed to switch department:', error);
      alert('部門の切り替えに失敗しました');
    }
  };

  // UIレンダリング
  return (
    <div>
      <Select
        label="部門選択"
        value={selectedDepartment || department}
        onChange={handleDepartmentChange}
        options={[
          { value: 'engineering', label: '技術部' },
          { value: 'sales', label: '営業部' },
        ]}
      />
      {/* RAG Chat UI */}
    </div>
  );
};
```

#### ExtraData送信
```typescript
// LocalStorageから部門を取得
const currentDepartment = localStorage.getItem('selected_department') || department;

// RAGクエリ時にextraDataとして送信
const extraData: ExtraData[] = [
  {
    type: 'json',
    name: 'departmentFilter',
    source: {
      type: 'inline',
      mediaType: 'application/json',
      data: JSON.stringify({
        equals: {
          key: 'department',
          value: currentDepartment
        }
      })
    }
  }
];

// Chatメッセージに含める
await postChat({
  content: userInput,
  extraData: extraData,
});
```

### 環境変数

**.env** または **Vite config**:
```bash
VITE_APP_USER_POOL_ID=ap-northeast-1_u8W3FROzJ
VITE_APP_USER_POOL_CLIENT_ID=5jgopjla8sm9d0uf8vbv5kn0a7
VITE_APP_IDENTITY_POOL_ID=ap-northeast-1:5c78d49a-687b-495d-9cbf-f6efc0829a50
VITE_APP_SAML_COGNITO_DOMAIN_NAME=genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com
VITE_APP_SAML_COGNITO_FEDERATED_IDENTITY_PROVIDER_NAME=EntraID
VITE_APP_API_ENDPOINT=https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/api
```

---

## トラブルシューティング

### 1. redirect_mismatch エラー

**症状**:
```
https://genu-gaab-sso-dev2.auth.ap-northeast-1.amazoncognito.com/error?error=redirect_mismatch
```

**原因**:
- Amplify.configure() が毎回異なるredirectSignInを使用している
- window.location.origin が不安定

**解決方法**:
```typescript
// ❌ Bad: Component body で呼ぶと毎レンダリング時に実行される
Amplify.configure({ ... });

// ✅ Good: useEffect で1回だけ実行
useEffect(() => {
  if (!isAmplifyConfigured) {
    Amplify.configure({ ... });
    isAmplifyConfigured = true;
  }
}, []);
```

### 2. グループが表示されない

**症状**: custom:samlGroups が空、またはObject IDのまま

**確認ポイント**:
1. Entra ID側で `groupMembershipClaims: "SecurityGroup"` が設定されているか
2. ユーザーがセキュリティグループに所属しているか
3. Pre-Token Generation Lambda が正常に動作しているか
4. Graph API の認証情報が正しいか

**デバッグ**:
```bash
# Lambda のログを確認
aws logs tail /aws/lambda/GenU-MapSamlGroups --since 30m --format short

# Cognito User attributes を確認
aws cognito-idp admin-get-user \
  --user-pool-id ap-northeast-1_u8W3FROzJ \
  --username "EntraID_s.abe@churadata.okinawa"
```

### 3. 部門フィルタが効かない

**症状**: 他部門のドキュメントも検索結果に表示される

**確認ポイント**:
1. Knowledge Base のドキュメントメタデータに `department` フィールドがあるか
2. getDynamicFilters が正しいフィルタを返しているか
3. UI で選択した部門が extraData として送信されているか

**デバッグ**:
```typescript
// Backend: bedrockKbApi.ts
console.log('[DEBUG] dynamicFilters:', JSON.stringify(dynamicFilters));
console.log('[DEBUG] userDefinedFilters:', JSON.stringify(userDefinedExplicitFilters));

// Frontend: useDepartment.ts
console.log('[useDepartment] Current department:', department);
console.log('[useDepartment] Available departments:', availableDepartments);
```

### 4. 自動ログインされてしまう

**症状**: Entra ID のログイン画面が表示されず、自動的にログインされる

**原因**:
- Cognito セッションが残っている
- LocalStorage にトークンがキャッシュされている
- Entra ID 側でもセッションが保持されている

**解決方法**:
1. **アプリ側**: signIn() でストレージをクリア（実装済み）
2. **ブラウザ側**: ユーザーがブラウザキャッシュをクリア
3. **Entra ID側**: 完全にログアウトするには `signOut({ global: true })` を使用

### 5. Graph API エラー

**症状**:
```
Failed to get group 423e6dc2-dd7b-4bef-a3eb-36f1fb60473c: 403 Forbidden
```

**原因**:
- Service Principal に Group.Read.All 権限がない
- Admin Consent が付与されていない

**解決方法**:
```bash
# 権限を確認
az ad app permission list --id $APP_ID

# Admin Consent を付与
az ad app permission admin-consent --id $APP_ID
```

### 6. Token 有効期限切れ

**症状**: API リクエストが 401 Unauthorized を返す

**解決方法**:
```typescript
// Amplify が自動的にトークンをリフレッシュ
const session = await fetchAuthSession({ forceRefresh: true });
```

### 7. SAML Metadata 取得エラー

**症状**: CDK デプロイ時に SAML メタデータが取得できない

**確認ポイント**:
- メタデータ URL が正しいか
- Entra ID アプリケーションが SAML モードになっているか
- ネットワークからアクセス可能か

**正しいメタデータURL**:
```
https://login.microsoftonline.com/{TENANT_ID}/federationmetadata/2007-06/federationmetadata.xml?appid={APP_ID}
```

---

## ベストプラクティス

### セキュリティ

1. **最小権限の原則**
   - Service Principal には `Group.Read.All` のみ付与
   - Lambda には必要最小限の権限のみ付与

2. **シークレット管理**
   - Azure Client Secret は AWS Secrets Manager で管理
   - 定期的にローテーション

3. **トークン有効期限**
   - ID Token: 1日（設定済み）
   - Refresh Token: 30日（Cognito デフォルト）

4. **CORS 設定**
   - API Gateway: 必要なオリジンのみ許可
   - CloudFront: 適切な CORS ヘッダー設定

### パフォーマンス

1. **Graph API キャッシング**
   - グループ情報をDynamoDBにキャッシュ（将来の改善案）
   - Pre-Token Generation Lambda のタイムアウトを30秒に設定

2. **フィルタ最適化**
   - Dynamic filter を優先適用
   - 不要なフィルタは削除

### メンテナンス

1. **ログ監視**
   - CloudWatch Logs で Lambda エラーを監視
   - Cognito ログを定期的に確認

2. **ドキュメント更新**
   - グループ追加時は DepartmentMaster テーブルを更新
   - 新しい部門追加時は Knowledge Base メタデータも更新

3. **テスト**
   - 各部門で正しくフィルタリングされるか定期的にテスト
   - 新しいユーザー追加時の動作確認

---

## まとめ

この実装により、以下が実現されました:

✅ **Enterprise SSO**: Azure Entra ID による安全なシングルサインオン
✅ **動的グループマッピング**: Graph API によるグループ名変換
✅ **部門ベースアクセス制御**: RAG Knowledge Base の部門別フィルタリング
✅ **柔軟な部門切り替え**: LocalStorage ベースの高速部門切り替え
   - JWTトークンをリフレッシュせずに部門切り替え
   - ページリロード不要
   - ナレッジベースのフィルタリング専用
✅ **強制再認証**: 毎回 Entra ID ログイン画面を表示
✅ **ユーザー情報表示**: ログインユーザーのメールアドレス表示
✅ **スケーラブル設計**: 新しい部門やグループの追加が容易
✅ **マルチ部門対応**: ユーザーは複数部門に所属可能（例: Engineering, Sales）

### 今後の改善案

1. **グループキャッシング**: DynamoDB にグループ情報をキャッシュして Graph API 呼び出しを削減
2. **管理画面**: DepartmentMaster テーブルを管理する UI の追加
3. **監査ログ**: 部門切り替えの監査ログを記録
4. **複数 Knowledge Base 対応**: 部門ごとに異なる Knowledge Base を使用
5. **ロールベースアクセス**: Admin/User ロールによる機能制限

---

**作成日**: 2025-12-18
**バージョン**: 1.0
**作成者**: Claude (Anthropic)
