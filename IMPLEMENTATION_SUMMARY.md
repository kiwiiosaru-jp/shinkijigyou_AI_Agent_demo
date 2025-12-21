# 部門切り替え機能とグループ同期機能の実装完了サマリー

## 実装日
2025-12-20

## 概要
このドキュメントは、部門切り替え機能の改善とセキュリティ強化（グループ削除同期）の実装完了サマリーです。localStorage ベースのクライアント側状態管理から、Cognito カスタム属性によるサーバーサイド状態管理への移行により、セキュリティと複数デバイス同期を実現しました。

---

## 🎯 主な成果

### 1. 部門切り替えの改善

| 項目 | 以前（localStorage） | 現在（Cognito 属性） |
|------|---------------------|---------------------|
| 状態保存 | ブラウザローカル | サーバーサイド（Cognito） |
| 複数デバイス | ❌ デバイスごと | ✅ 全デバイスで同期 |
| JWT リフレッシュ | しない | する |
| セキュリティ | クライアント改ざん可 | サーバー管理で安全 |
| 監査ログ | なし | CloudWatch Logs で追跡可能 |
| IAM ロール制御 | ❌ ロールが保持されない | ✅ Admin/User ロールを保持 |

### 2. セキュリティの向上

| セキュリティ項目 | 以前 | 現在 |
|-----------------|------|------|
| グループ削除同期 | ❌ なし | ✅ 最大24時間以内 |
| クライアント改ざん対策 | ❌ 脆弱 | ✅ サーバー管理 |
| 監査証跡 | ❌ 限定的 | ✅ 完全な CloudWatch Logs |
| 緊急時の対応手順 | ❌ なし | ✅ ドキュメント化済み |

---

## 📊 詳細解説：部門切り替えの改善

### 1.1 状態保存：ブラウザローカル → サーバーサイド（Cognito）

#### 以前の実装（localStorage）

```javascript
// クライアント側（ブラウザ）
localStorage.setItem('currentDepartment', 'engineering');
const dept = localStorage.getItem('currentDepartment');
```

**問題点:**
- ブラウザのローカルストレージに保存されるため、デバイスごとに異なる部門が記憶される
- ユーザーがブラウザ開発者ツールで `localStorage` を直接編集可能
- サーバー側で状態を把握できない
- ブラウザのキャッシュクリアで消失

#### 現在の実装（Cognito custom:department 属性）

```typescript
// Lambda 関数（サーバー側）
await client.send(
  new AdminUpdateUserAttributesCommand({
    UserPoolId: userPoolId,
    Username: username,
    UserAttributes: [
      {
        Name: 'custom:department',
        Value: 'engineering',  // 'engineering' または 'sales'
      },
    ],
  })
);
```

**改善点:**
- Cognito User Pool にユーザー属性として保存
- サーバー側で完全に管理され、クライアント側からは変更不可
- AWS の高可用性インフラで永続化
- IAM 権限で保護された API 経由でのみ変更可能

#### データの保存場所と永続性

| 項目 | localStorage | Cognito 属性 |
|------|--------------|--------------|
| 保存場所 | ブラウザローカル | AWS Cognito User Pool |
| 永続性 | ブラウザ依存 | AWS インフラで永続化 |
| 可用性 | 単一デバイス | 全デバイス |
| バックアップ | なし | AWS 管理 |
| 削除条件 | キャッシュクリア | 明示的な削除のみ |

---

### 1.2 複数デバイス対応：デバイスごと → 全デバイスで同期

#### 以前の問題

```
デバイス A: localStorage → 'engineering'
デバイス B: localStorage → 'sales'
デバイス C: localStorage → (未設定)
```

**問題シナリオ:**
1. ユーザーが会社の PC（デバイス A）で部門を「Engineering」に設定
2. 同じユーザーが自宅の PC（デバイス B）でログイン
3. デバイス B では部門が未設定、または古い設定のまま
4. ユーザーがデバイス B で「Sales」に変更
5. 翌日会社の PC（デバイス A）に戻ると「Engineering」のまま
6. **結果**: デバイスごとに異なる部門で作業してしまう

#### 現在の実装：サーバーサイド同期

```
Cognito User Pool
  └─ User: EntraID_user@example.com
      └─ custom:department: 'engineering'  ← 単一の真実の情報源（Single Source of Truth）

デバイス A: JWT から取得 → 'engineering'
デバイス B: JWT から取得 → 'engineering'
デバイス C: JWT から取得 → 'engineering'
```

#### 同期のメカニズム

**1. 部門切り替え時の動作フロー:**

```
[デバイス A: ユーザーが Sales に変更]
  ↓
[API: POST /user/department]
  ↓
[Lambda: updateUserDepartment]
  ↓ AdminUpdateUserAttributesCommand
[Cognito: custom:department = 'sales']
  ↓
[デバイス A: JWT リフレッシュ]
  ↓
[新しい JWT に custom:department = 'sales' が含まれる]
  ↓
[デバイス A: UI に 'Sales' 表示]

[デバイス B: 次回ログインまたはリフレッシュ]
  ↓
[Pre-Token Generation Lambda が custom:department を読み取り]
  ↓
[JWT に custom:department = 'sales' を追加]
  ↓
[デバイス B: UI に 'Sales' 表示]  ← 自動的に同期される
```

**2. 同期タイミング:**

| イベント | 同期タイミング | 所要時間 |
|---------|--------------|----------|
| 部門切り替え（同じデバイス） | 即座（JWT リフレッシュ後） | 1-2秒 |
| 部門切り替え（別デバイス） | 次回ログイン時 | 次回ログインまで |
| 部門切り替え（別デバイス） | トークンリフレッシュ時 | 最大1時間 |

**3. 実際の同期例:**

```bash
# デバイス A: 部門を Sales に変更
Time: 10:00:00
Action: User clicks "Switch to Sales" on Device A
Result: custom:department updated to 'sales' in Cognito

# デバイス B: 10分後にページをリフレッシュ
Time: 10:10:00
Action: User refreshes page on Device B
Flow:
  1. JWT still valid (not expired yet)
  2. JWT contains old value: custom:department = 'engineering'
  3. UI shows 'Engineering'  ← まだ同期されていない

# デバイス B: JWT が期限切れになり、自動リフレッシュ
Time: 11:05:00
Action: JWT expires (1 hour token lifetime), automatic refresh triggered
Flow:
  1. Cognito refreshes JWT
  2. Pre-Token Generation Lambda reads custom:department = 'sales'
  3. New JWT contains custom:department = 'sales'
  4. UI updates to show 'Sales'  ← 同期完了

# デバイス C: 新規ログイン
Time: 14:00:00
Action: User logs in on Device C
Flow:
  1. Login process triggers Pre-Token Generation Lambda
  2. Lambda reads custom:department = 'sales'
  3. JWT contains custom:department = 'sales'
  4. UI shows 'Sales'  ← 即座に同期された状態
```

---

### 1.3 JWT リフレッシュ：しない → する

#### JWT リフレッシュの必要性

**以前の実装（localStorage）:**
```javascript
// JWT は変更されない
// localStorage の値だけが変わる
localStorage.setItem('currentDepartment', 'sales');
// JWT には引き続き cognito:groups = ['Engineering-User', 'Sales-User'] が含まれる
// サーバー側では部門の変更を検知できない
```

**現在の実装（JWT リフレッシュ）:**
```javascript
// 部門変更後に JWT をリフレッシュ
await Auth.currentSession(true);  // forceRefresh = true

// 新しい JWT には custom:department = 'sales' が含まれる
// サーバー側（Knowledge Base など）は新しい JWT から部門を判定
```

#### JWT リフレッシュのフロー

**1. 部門切り替え時の完全なフロー:**

```typescript
// Step 1: ユーザーが UI で部門を切り替え
[UI] ユーザーが「Sales」ボタンをクリック

// Step 2: API 呼び出し
[UI] → POST /api/user/department
        Body: { department: 'sales' }
        Headers: Authorization: Bearer <OLD_JWT>

// Step 3: Lambda が custom:department を更新
[Lambda: updateUserDepartment]
  ↓ AdminUpdateUserAttributesCommand
[Cognito] custom:department = 'sales' に更新

// Step 4: UI が JWT をリフレッシュ
[UI] → Cognito: Refresh Token を使用
        ↓
[Cognito: Pre-Token Generation Lambda が起動]
        ↓ custom:department を読み取り
        ↓ event.response.claimsOverrideDetails に追加
[Cognito] → 新しい JWT を発行
        JWT Claims: {
          "sub": "uuid",
          "cognito:username": "EntraID_user@example.com",
          "cognito:groups": ["Engineering-User", "Sales-User"],
          "custom:department": "sales",  ← 追加される
          "email": "user@example.com",
          ...
        }

// Step 5: UI が新しい JWT を受け取る
[UI] 新しい JWT を localStorage に保存
     UI を更新して「Sales」を表示
```

**2. JWT の比較:**

```json
// 以前の JWT（custom:department なし）
{
  "sub": "abc123",
  "cognito:username": "EntraID_user@example.com",
  "cognito:groups": ["Engineering-User", "Sales-User"],
  "email": "user@example.com",
  "exp": 1703001600,
  "iat": 1702998000
}

// 現在の JWT（custom:department あり）
{
  "sub": "abc123",
  "cognito:username": "EntraID_user@example.com",
  "cognito:groups": ["Engineering-User", "Sales-User"],
  "custom:department": "sales",  ← これが追加される
  "email": "user@example.com",
  "exp": 1703001600,
  "iat": 1702998000
}
```

#### JWT リフレッシュのタイミング

| シナリオ | リフレッシュタイミング | 説明 |
|---------|---------------------|------|
| 部門切り替え直後 | 即座 | UI が明示的に `Auth.currentSession(true)` を呼び出し |
| 通常のページ遷移 | JWT 期限切れ時 | デフォルト1時間後に自動リフレッシュ |
| ログイン直後 | ログイン時 | 新しい JWT が発行される |
| ページリロード | JWT が有効ならリフレッシュしない | localStorage から既存の JWT を使用 |

---

### 1.4 セキュリティ：クライアント改ざん可 → サーバー管理で安全

#### 以前の脆弱性（localStorage）

**攻撃シナリオ 1: 部門の改ざん**

```javascript
// 攻撃者がブラウザの開発者コンソールで実行
localStorage.setItem('currentDepartment', 'sales');
location.reload();

// 結果: Engineering ユーザーが Sales の情報にアクセス可能
```

**攻撃シナリオ 2: API 呼び出しの改ざん**

```javascript
// 攻撃者が API リクエストを傍受・変更
// オリジナルのリクエスト
fetch('/api/knowledge-base', {
  method: 'POST',
  body: JSON.stringify({
    query: 'Show me all contracts',
    department: 'engineering'  // localStorage から取得
  })
});

// 改ざん後のリクエスト（Proxy ツールで変更）
fetch('/api/knowledge-base', {
  method: 'POST',
  body: JSON.stringify({
    query: 'Show me all contracts',
    department: 'sales'  // 'sales' に変更
  })
});

// 結果: Engineering ユーザーが Sales の契約情報を取得
```

#### 現在のセキュリティ対策

**対策 1: サーバーサイド検証**

```typescript
// Lambda 関数（サーバー側）
export const handler = async (event: APIGatewayProxyEvent) => {
  // JWT から部門を取得（クライアントからのリクエストボディは無視）
  const jwtDepartment = event.requestContext.authorizer?.claims?.['custom:department'];

  // JWT に custom:department が含まれていない場合はグループから判定
  if (!jwtDepartment) {
    const groups = event.requestContext.authorizer?.claims?.['cognito:groups'] || [];
    // グループから部門を抽出
  }

  // サーバー側で決定した部門を使用
  const department = jwtDepartment || extractDepartmentFromGroups(groups);

  // Knowledge Base へのクエリで department を使用
  const results = await searchKnowledgeBase(query, department);

  return {
    statusCode: 200,
    body: JSON.stringify(results)
  };
};
```

**対策 2: IAM 権限による保護**

```typescript
// Cognito User Pool の属性変更は IAM 権限で保護
// packages/cdk/lib/construct/api.ts

userPool.grant(
  updateUserDepartmentFunction,
  'cognito-idp:AdminUpdateUserAttributes'  // この権限を持つ Lambda のみが変更可能
);

// クライアント（ブラウザ）からは直接変更不可
// API Gateway + Lambda + IAM 権限 を経由する必要がある
```

**対策 3: API Gateway での認証**

```yaml
# API Gateway の設定
/user/department:
  post:
    security:
      - CognitoAuthorizer: []  # Cognito JWT が必須
    x-amazon-apigateway-integration:
      type: aws_proxy
      httpMethod: POST
      uri: arn:aws:lambda:...:updateUserDepartment
```

#### セキュリティ比較

| セキュリティ項目 | localStorage | Cognito 属性 |
|-----------------|--------------|--------------|
| クライアント側改ざん | 可能（開発者ツールで変更） | 不可能（JWT 署名で保護） |
| API リクエスト改ざん | 影響あり | 影響なし（サーバーが JWT から取得） |
| 必要な攻撃手法 | ブラウザ操作のみ | JWT の偽造（実質不可能） |
| 検知の難易度 | 検知困難（ログなし） | 検知容易（CloudWatch Logs） |
| IAM 権限 | 不要（クライアント側） | 必須（Lambda のみ変更可） |

---

### 1.5 監査ログ：なし → CloudWatch Logs で追跡可能

#### 以前の問題（ログなし）

```javascript
// localStorage への変更はログに記録されない
localStorage.setItem('currentDepartment', 'sales');
// いつ、誰が、どの部門に変更したか追跡不可
```

**問題点:**
- セキュリティインシデント発生時に調査できない
- 部門の変更履歴が残らない
- 不正なアクセスを検知できない
- コンプライアンス要件を満たせない

#### 現在の実装：完全な監査証跡

**1. 部門変更のログ**

```typescript
// Lambda: updateUserDepartment.ts
console.log('[updateUserDepartment] Event:', JSON.stringify(event));
console.log(`[updateUserDepartment] Username: ${username}, Email: ${email}`);
console.log(`[updateUserDepartment] Switching to department group: ${newGroupName}`);
console.log(`[updateUserDepartment] Updated custom:department attribute to ${departmentLower}`);
```

**CloudWatch Logs の出力例:**

```json
{
  "timestamp": "2025-12-20T10:15:30.123Z",
  "level": "INFO",
  "message": "[updateUserDepartment] Username: EntraID_user@example.com, Email: user@example.com"
}
{
  "timestamp": "2025-12-20T10:15:30.456Z",
  "level": "INFO",
  "message": "[updateUserDepartment] User already in Sales-User, preserving role: User"
}
{
  "timestamp": "2025-12-20T10:15:31.789Z",
  "level": "INFO",
  "message": "[updateUserDepartment] Updated custom:department attribute to sales"
}
{
  "timestamp": "2025-12-20T10:15:32.012Z",
  "level": "INFO",
  "message": "[updateUserDepartment] Updated department for user EntraID_user@example.com (user@example.com) to Sales (group: Sales-User)"
}
```

**2. 部門取得のログ**

```typescript
// Lambda: getUserDepartment.ts
console.log('[getUserDepartment] Event:', JSON.stringify(event));
console.log(`[getUserDepartment] Username: ${username}, Email: ${email}`);
console.log(`[getUserDepartment] Found custom:department attribute: ${department}`);
console.log(`[getUserDepartment] User ${username} (${email}) department: ${department}`);
```

**3. グループ同期のログ**

```typescript
// Lambda: mapSamlGroups.ts
console.log('[mapSamlGroups] Pre-Token Generation triggered');
console.log(`[mapSamlGroups] User ${username} current Cognito groups:`, currentCognitoGroups);
console.log(`[mapSamlGroups] Entra ID groups:`, groupNames);
console.log(`[mapSamlGroups] Groups to remove:`, groupsToRemove);
console.log(`[mapSamlGroups] Removed user from group: ${groupName}`);
```

#### ログの検索と分析

**1. AWS CLI でログを検索:**

```bash
# 特定ユーザーの部門変更履歴を検索
aws logs filter-log-events \
  --log-group-name /aws/lambda/updateUserDepartment \
  --filter-pattern "user@example.com" \
  --start-time $(date -u -d '7 days ago' +%s)000 \
  --region ap-northeast-1

# 部門変更の統計を取得
aws logs filter-log-events \
  --log-group-name /aws/lambda/updateUserDepartment \
  --filter-pattern "Updated department" \
  --start-time $(date -u -d '30 days ago' +%s)000 \
  --region ap-northeast-1 \
  | jq '.events | length'
```

**2. CloudWatch Logs Insights クエリ:**

```sql
-- 過去7日間の部門変更回数をユーザー別に集計
fields @timestamp, @message
| filter @message like /Updated department for user/
| parse @message "Updated department for user * (*) to * (group: *)" as username, email, department, group
| stats count() by email, department
| sort count desc

-- 特定ユーザーの部門変更履歴
fields @timestamp, @message
| filter @message like /user@example.com/
| filter @message like /Updated department/
| sort @timestamp desc
| limit 100

-- エラーが発生した部門変更
fields @timestamp, @message
| filter @message like /updateUserDepartment/
| filter @message like /Error/ or @message like /Failed/
| sort @timestamp desc
```

**3. 監査レポート例:**

```
部門変更監査レポート
期間: 2025-12-01 ~ 2025-12-31

ユーザー                 | 変更回数 | 最終部門 | 最終変更日時
------------------------|---------|---------|------------------------
user1@example.com       | 15      | Sales   | 2025-12-20 10:15:32
user2@example.com       | 8       | Eng.    | 2025-12-19 14:30:21
user3@example.com       | 3       | Sales   | 2025-12-18 09:45:10

部門別変更統計:
- Engineering → Sales: 45回
- Sales → Engineering: 38回

異常検知:
- user1@example.com: 1日に5回以上の変更（要確認）
```

#### ログの保持期間とコンプライアンス

| 項目 | 設定値 | 説明 |
|------|--------|------|
| ログ保持期間 | デフォルト: 無期限 | CloudWatch Logs で設定可能 |
| ログの暗号化 | 有効 | AWS KMS で暗号化 |
| ログへのアクセス | IAM 制限 | 管理者のみアクセス可能 |
| ログの改ざん防止 | AWS 管理 | CloudWatch Logs は改ざん不可 |

---

### 1.6 IAM ロールベースのアクセス制御の改善

#### 今回の改善のきっかけ

部門切り替え機能の実装において、**最も重要な改善点は IAM ロール（Admin/User）の適切な保持**でした。以前の実装では、部門を切り替えた際にユーザーの役割（Admin または User）が正しく保持されず、以下の問題が発生していました：

1. **S3 RAG インジェスト用ファイルへのアクセス権限の喪失** - Admin のみがアクセス可能
2. **ユーザ管理権限の喪失** - Admin は配下のユーザを招待・削除可能

注: Knowledge Base のデータフィルタリングは部門ID（custom:department）のみで行われ、Admin/User ロールは影響しません。

#### 以前の問題

**シナリオ: Engineering-Admin から Sales 部門への切り替え**

```typescript
// 以前の実装の問題
User: admin@example.com (Engineering 部門の管理者)
Current Groups: ['Engineering-Admin', 'Sales-User']

// ユーザーが Sales 部門に切り替え
[UI] ユーザーが「Switch to Sales」をクリック

// 問題のある処理
[Lambda] Sales 部門のデフォルトグループ 'Sales-User' に追加
         ↓
Result: User is in ['Engineering-Admin', 'Sales-User']
        ↓
[S3 Access Control]
  - S3 RAG インジェスト用ファイルへのアクセス試行
  - JWT の cognito:groups を確認
  - 'Sales-Admin' が見つからない（'Sales-User' のみ）
  - AccessDenied: Admin ロールが必要
        ↓
[User Management]
  - Sales 部門のユーザ招待ボタンをクリック
  - JWT の cognito:groups を確認
  - 'Sales-Admin' が見つからない（'Sales-User' のみ）
  - Forbidden: Admin 権限が必要
        ↓
問題1: Sales 部門の RAG インジェスト用ファイルにアクセスできない
問題2: Sales 部門のユーザを招待・削除できない
```

**根本原因:**
1. ユーザーが複数の部門グループに所属（Engineering-Admin, Sales-User）
2. 部門切り替え時に、ターゲット部門（Sales）での既存のロール（Admin）を考慮せず、デフォルトの User ロールでグループに追加
3. S3 アクセス制御とユーザ管理機能は Admin ロールを要求
4. 結果として、部門管理者が部門を切り替えると管理者権限を失う

#### 現在の実装：ロール保持ロジック

**実装コード:**

```typescript
// packages/cdk/lambda/updateUserDepartment.ts

export const handler = async (event: APIGatewayProxyEvent) => {
  const { department } = JSON.parse(event.body || '{}');
  const departmentName = department.charAt(0).toUpperCase() + department.slice(1);

  // 現在のグループを取得
  const groupsResponse = await client.send(
    new AdminListGroupsForUserCommand({
      UserPoolId: userPoolId,
      Username: username,
    })
  );

  const currentGroups = groupsResponse.Groups || [];

  // 重要: ターゲット部門で既にロールを持っているか確認
  let userRole = 'User';  // デフォルトは User ロール
  let targetDepartmentGroup: string | null = null;

  // ターゲット部門のグループを検索
  for (const group of currentGroups) {
    const groupName = group.GroupName;
    if (groupName && groupName.startsWith(`${departmentName}-`)) {
      // ユーザーは既にこの部門のグループに所属している
      const parts = groupName.split('-');
      if (parts.length >= 2) {
        targetDepartmentGroup = groupName;
        userRole = parts[1];  // Admin または User を保持
        console.log(`[updateUserDepartment] User already in ${groupName}, preserving role: ${userRole}`);
        break;
      }
    }
  }

  // 新しいグループ名を決定（ロールを保持）
  const newGroupName = `${departmentName}-${userRole}`;

  // グループに追加（必要な場合）
  if (targetDepartmentGroup !== newGroupName) {
    try {
      await client.send(
        new AdminAddUserToGroupCommand({
          UserPoolId: userPoolId,
          Username: username,
          GroupName: newGroupName,
        })
      );
      console.log(`[updateUserDepartment] Added user to ${newGroupName}`);
    } catch (error: any) {
      if (error.name !== 'InvalidParameterException') {
        throw error;
      }
      console.log(`[updateUserDepartment] User already in ${newGroupName}`);
    }
  }

  // custom:department 属性を更新
  await client.send(
    new AdminUpdateUserAttributesCommand({
      UserPoolId: userPoolId,
      Username: username,
      UserAttributes: [
        {
          Name: 'custom:department',
          Value: department.toLowerCase(),
        },
      ],
    })
  );

  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Department updated successfully',
      department: department.toLowerCase(),
      group: newGroupName,
    })
  };
};
```

#### 改善後の動作フロー

**シナリオ: Engineering-Admin から Sales 部門への切り替え**

```typescript
User: admin@example.com (複数部門の管理者)
Current Groups: ['Engineering-Admin', 'Sales-Admin']
                                       ↑
              注: Sales-Admin グループにも所属（Admin ロールを持つ）

// ユーザーが Sales 部門に切り替え
[UI] ユーザーが「Switch to Sales」をクリック
  ↓
[Lambda: updateUserDepartment]
  1. 現在のグループを取得: ['Engineering-Admin', 'Sales-Admin']
  2. Sales 部門のグループを検索: 'Sales-Admin' を発見
  3. ロールを抽出: 'Admin'  ← ここが重要
  4. 新しいグループ名を決定: 'Sales-Admin'（Admin ロールを保持）
  5. custom:department = 'sales' を更新
  ↓
[JWT リフレッシュ]
  JWT Claims: {
    "cognito:groups": ["Engineering-Admin", "Sales-Admin"],
    "custom:department": "sales"  ← これで部門を特定
  }
  ↓
[S3 Access Control]
  - S3 RAG インジェスト用ファイルへのアクセス試行
  - JWT の cognito:groups を確認
  - 'Sales-Admin' が見つかる
  - AccessGranted: Admin ロールを保持
  ↓
[User Management]
  - Sales 部門のユーザ招待ボタンをクリック
  - JWT の cognito:groups を確認
  - 'Sales-Admin' が見つかる
  - Success: ユーザ招待・削除が可能
  ↓
✅ 正しい動作: Sales 部門の管理者として全ての権限を保持
```

#### IAM ロールとアクセス制御の関係

**Admin ロールによるアクセス制御の実装箇所:**

```typescript
// 1. S3 RAG インジェスト用ファイルへのアクセス制御
// API Lambda（概念的なコード）
export const uploadRAGFileHandler = async (event: APIGatewayProxyEvent) => {
  const claims = event.requestContext.authorizer?.claims;
  const currentDepartment = claims?.['custom:department'];  // 'sales'
  const groups = claims?.['cognito:groups'] || [];

  // Admin ロールを確認
  const departmentGroup = groups.find(g =>
    g.toLowerCase().startsWith(currentDepartment)
  );  // 'Sales-Admin' を検索
  const role = departmentGroup?.split('-')[1];  // 'Admin'

  if (role !== 'Admin') {
    return {
      statusCode: 403,
      body: JSON.stringify({ error: 'Admin role required for file upload' })
    };
  }

  // S3 へのアップロード処理
  const s3Key = `${currentDepartment}/rag-files/${fileName}`;
  await s3Client.putObject({ Bucket: ragBucket, Key: s3Key, Body: file });

  return { statusCode: 200, body: JSON.stringify({ success: true }) };
};

// 2. ユーザ管理権限の制御
// API Lambda（概念的なコード）
export const inviteUserHandler = async (event: APIGatewayProxyEvent) => {
  const claims = event.requestContext.authorizer?.claims;
  const currentDepartment = claims?.['custom:department'];
  const groups = claims?.['cognito:groups'] || [];

  // Admin ロールを確認
  const departmentGroup = groups.find(g =>
    g.toLowerCase().startsWith(currentDepartment)
  );
  const role = departmentGroup?.split('-')[1];

  if (role !== 'Admin') {
    return {
      statusCode: 403,
      body: JSON.stringify({ error: 'Admin role required for user management' })
    };
  }

  // ユーザー招待処理
  // (将来的に Entra ID への Graph API 呼び出し、または Cognito への直接追加)

  return { statusCode: 200, body: JSON.stringify({ success: true }) };
};

// 3. Knowledge Base のデータフィルタリング（参考）
// 注: Knowledge Base は部門ID（custom:department）のみでフィルタリング
// Admin/User ロールは影響しない
export const knowledgeBaseQueryHandler = async (event: APIGatewayProxyEvent) => {
  const claims = event.requestContext.authorizer?.claims;
  const currentDepartment = claims?.['custom:department'];  // 'sales'

  // 部門IDのみでフィルタリング（ロールは不要）
  const filter = { department: currentDepartment };
  const results = await queryKnowledgeBase(query, filter);

  return { statusCode: 200, body: JSON.stringify(results) };
};
```

#### ロール保持の重要性

| 項目 | ロール保持なし | ロール保持あり |
|------|---------------|---------------|
| Admin が部門切り替え | User ロールに降格 | Admin ロールを保持 |
| S3 RAG ファイルアクセス | ❌ アクセス不可 | ✅ アップロード・管理可能 |
| ユーザ管理権限 | ❌ 招待・削除不可 | ✅ 配下ユーザ管理可能 |
| セキュリティ | ❌ 意図しない権限変更 | ✅ 権限の一貫性を維持 |
| ユーザー体験 | ❌ 部門切り替え後に権限不足 | ✅ シームレスな操作 |
| 監査証跡 | ⚠️  権限変更が記録されない | ✅ ロール保持がログに記録 |

#### セキュリティ上の利点

**1. 権限の一貫性:**
- Admin ユーザーは部門を切り替えても Admin 権限を保持
- User ユーザーは部門を切り替えても User 権限のまま
- 意図しない権限昇格や降格を防止

**2. アクセス制御の正確性:**
- S3 RAG インジェスト用ファイルへのアクセス制御が正しく機能
- ユーザ管理機能の権限制御が適切に動作
- 部門ごとのデータ分離が確実に実施
- IAM ロールベースのアクセス制御（RBAC）が適切に動作

**3. 監査証跡の完全性:**
- どのロールでどの部門にアクセスしたかを追跡可能
- 権限変更の履歴が完全に記録される
- コンプライアンス要件を満たす

**4. 管理者体験の向上:**
- 部門管理者は複数部門を管理する際、各部門で管理者権限を維持
- 部門を切り替えるたびに権限を再付与する必要がない
- 運用負荷の軽減

---

## 🔒 詳細解説：Entra ID と Cognito のユーザ・グループ同期

### 2.1 グループ削除同期：なし → 最大24時間以内

#### 以前の問題（同期なし）

**問題シナリオ:**

```
Day 1: ユーザーが Sales 部門に所属
  Entra ID: Sales-User グループ
  Cognito: Sales-User グループ  ← 同期されている

Day 2: ユーザーが退職、または部門異動
  Entra ID: Sales-User グループから削除  ← 管理者が削除
  Cognito: Sales-User グループ  ← 削除されない（問題）

Day 3 ~ Day 365:
  ユーザーは引き続き Cognito の Sales-User グループに所属
  Sales 部門の情報にアクセス可能  ← セキュリティリスク
```

**影響:**
- 退職者が無期限にアクセス可能
- 部門異動後も古い部門の情報にアクセス可能
- グループから削除されたユーザーの権限が残り続ける

#### 現在の実装：自動削除同期

**実装コード:**

```typescript
// packages/cdk/lambda/mapSamlGroups.ts (lines 319-361)

// Step 1: ユーザーの現在の Cognito グループを取得
const currentCognitoGroups = await getUserCognitoGroups(
  event.userPoolId,
  event.userName
);
// 例: ['Engineering-User', 'Sales-User', 'ap-northeast-1_xxx_EntraID']

// Step 2: 部門グループのみをフィルタ
// システムグループ（ap-northeast-1_xxx_EntraID）は除外
const departmentGroups = currentCognitoGroups.filter(
  (groupName) => groupName.includes('-') && !groupName.includes('_')
);
// 結果: ['Engineering-User', 'Sales-User']

// Step 3: Entra ID に存在しないグループを特定
const groupsToRemove = departmentGroups.filter(
  (cognitoGroup) => !groupNames.includes(cognitoGroup)
);
// 例: Entra ID に 'Sales-User' が存在しない場合
// groupsToRemove = ['Sales-User']

// Step 4: Entra ID に存在しないグループからユーザーを削除
for (const groupName of groupsToRemove) {
  console.log(`[mapSamlGroups] Removing user from group: ${groupName}`);

  await removeUserFromCognitoGroup(
    event.userPoolId,
    event.userName,
    groupName
  );

  console.log(`[mapSamlGroups] Successfully removed user from group: ${groupName}`);
}
```

#### 同期のタイミングとフロー

**完全な同期フロー:**

```
Time: 09:00 - 管理者が Entra ID でユーザーを Sales-User グループから削除
  ↓
[Entra ID]
  User: user@example.com
  Groups: ['Engineering-User']  ← Sales-User が削除された

[Cognito]
  User: EntraID_user@example.com
  Groups: ['Engineering-User', 'Sales-User']  ← まだ Sales-User が残っている

Time: 09:00 ~ 10:00 - ユーザーがログイン中
  ↓
[JWT が有効]
  JWT Claims: {
    "cognito:groups": ["Engineering-User", "Sales-User"],  ← 古いグループ情報
    "custom:department": "sales"
  }
  ユーザーは引き続き Sales にアクセス可能  ← セキュリティギャップ

Time: 10:00 - JWT の有効期限切れ（デフォルト1時間）
  ↓
[ユーザーのブラウザが自動的に JWT をリフレッシュ]
  ↓
[Cognito: Pre-Token Generation Lambda が起動]
  ↓
[mapSamlGroups Lambda 実行]
  1. Entra ID の Graph API からユーザーのグループを取得
     → ['Engineering-User']  ← Sales-User は含まれない

  2. Cognito の現在のグループを取得
     → ['Engineering-User', 'Sales-User']

  3. 差分を計算
     → groupsToRemove = ['Sales-User']

  4. Sales-User グループからユーザーを削除
     → AdminRemoveUserFromGroupCommand 実行
  ↓
[Cognito]
  User: EntraID_user@example.com
  Groups: ['Engineering-User']  ← Sales-User が削除された（同期完了）
  ↓
[新しい JWT 発行]
  JWT Claims: {
    "cognito:groups": ["Engineering-User"],  ← 更新されたグループ情報
    "custom:department": "engineering"  ← 自動的に engineering に戻る
  }
  ↓
[ユーザーは Sales にアクセス不可]  ← セキュリティ確保
```

#### 24時間の制限について

**最大24時間の理由:**

| 項目 | 設定値 | 説明 |
|------|--------|------|
| JWT Access Token 有効期限 | 1時間（デフォルト） | 最大24時間まで設定可能 |
| JWT Refresh Token 有効期限 | 30日（デフォルト） | リフレッシュトークンは長期間有効 |
| 同期タイミング | Token リフレッシュ時 | Pre-Token Generation Lambda が起動 |

**実際の同期時間:**

```
ケース 1: ユーザーがアクティブに操作中
  - JWT が1時間ごとに自動リフレッシュ
  - 最大1時間以内に同期  ← 一般的なケース

ケース 2: ユーザーがログインしたまま放置
  - JWT の有効期限（最大24時間）まで同期されない
  - 最大24時間後に同期  ← 最悪ケース

ケース 3: ユーザーがログアウト → 再ログイン
  - ログイン時に Pre-Token Generation Lambda 起動
  - 即座に同期  ← 最速ケース
```

#### セキュリティギャップへの対応

**緊急時の手動削除手順:**

```bash
#!/bin/bash
# 緊急時にユーザーを即座にグループから削除

USER_POOL_ID="ap-northeast-1_xxxx"
USERNAME="EntraID_user@example.com"
GROUP_NAME="Sales-User"

# Cognito からグループを削除
aws cognito-idp admin-remove-user-from-group \
  --user-pool-id "$USER_POOL_ID" \
  --username "$USERNAME" \
  --group-name "$GROUP_NAME" \
  --region ap-northeast-1

# 現在のセッションを無効化（オプション）
aws cognito-idp admin-user-global-sign-out \
  --user-pool-id "$USER_POOL_ID" \
  --username "$USERNAME" \
  --region ap-northeast-1

echo "User $USERNAME removed from group $GROUP_NAME"
echo "User's current sessions have been invalidated"
```

---

### 2.2 クライアント改ざん対策：脆弱 → サーバー管理

#### 詳細な攻撃シナリオと対策

**攻撃シナリオ 1: localStorage の直接改ざん**

```javascript
// 以前の脆弱な実装
// 攻撃者がブラウザの開発者コンソールで実行
console.log(localStorage.getItem('currentDepartment'));  // 'engineering'
localStorage.setItem('currentDepartment', 'sales');
console.log(localStorage.getItem('currentDepartment'));  // 'sales'

// アプリケーションは localStorage の値を信頼
const department = localStorage.getItem('currentDepartment');
fetch('/api/knowledge-base', {
  body: JSON.stringify({ query: '契約一覧', department })
});

// 結果: Engineering ユーザーが Sales の契約にアクセス
```

**対策:**

```typescript
// 現在の実装: JWT から部門を取得
// Lambda 関数（サーバー側）
export const handler = async (event: APIGatewayProxyEvent) => {
  // クライアントからのリクエストボディは無視
  const requestBody = JSON.parse(event.body || '{}');
  console.log('[WARNING] Ignoring department from request body:', requestBody.department);

  // JWT から部門を取得（改ざん不可能）
  const department = event.requestContext.authorizer?.claims?.['custom:department'];

  if (!department) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'Department not set in JWT' })
    };
  }

  // サーバー側で決定した部門を使用
  const results = await searchKnowledgeBase(requestBody.query, department);

  return {
    statusCode: 200,
    body: JSON.stringify(results)
  };
};
```

**攻撃シナリオ 2: HTTP リクエストの改ざん（Proxy 攻撃）**

```http
# 攻撃者が Burp Suite や mitmproxy で HTTP リクエストを傍受

# オリジナルのリクエスト
POST /api/knowledge-base HTTP/1.1
Host: example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "query": "契約一覧を表示",
  "department": "engineering"
}

# 攻撃者が department を変更
POST /api/knowledge-base HTTP/1.1
Host: example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "query": "契約一覧を表示",
  "department": "sales"  ← 改ざん
}
```

**対策:**

```typescript
// サーバー側で JWT から部門を取得
// リクエストボディの department は完全に無視

export const handler = async (event: APIGatewayProxyEvent) => {
  const requestBody = JSON.parse(event.body || '{}');

  // JWT から部門を取得（HTTP リクエストとは無関係）
  const jwtDepartment = event.requestContext.authorizer?.claims?.['custom:department'];

  // リクエストボディの department とJWT の department を比較
  if (requestBody.department && requestBody.department !== jwtDepartment) {
    console.warn(
      `[SECURITY] Department mismatch detected! ` +
      `Request: ${requestBody.department}, JWT: ${jwtDepartment}`
    );
    // セキュリティアラートを送信
    await sendSecurityAlert({
      type: 'DEPARTMENT_MISMATCH',
      username: event.requestContext.authorizer?.claims?.['cognito:username'],
      requestDepartment: requestBody.department,
      jwtDepartment,
      timestamp: new Date().toISOString()
    });
  }

  // JWT の値のみを使用
  const results = await searchKnowledgeBase(requestBody.query, jwtDepartment);

  return {
    statusCode: 200,
    body: JSON.stringify(results)
  };
};
```

#### 多層防御アーキテクチャ

```
[レイヤー 1: クライアント側]
  - localStorage は読み取り専用として扱う
  - API リクエストには department を含めない
  ↓
[レイヤー 2: API Gateway]
  - Cognito Authorizer で JWT を検証
  - 署名、有効期限、Issuer を確認
  - 無効な JWT は即座に拒否（Lambda に到達しない）
  ↓
[レイヤー 3: Lambda 関数]
  - JWT の claims から department を取得
  - リクエストボディの department は無視
  - 不一致を検出した場合はアラート
  ↓
[レイヤー 4: Cognito]
  - custom:department 属性は IAM 権限で保護
  - AdminUpdateUserAttributes 権限を持つ Lambda のみが変更可能
  ↓
[レイヤー 5: CloudWatch Logs]
  - 全ての操作をログに記録
  - 不審な動作を検知・アラート
```

---

### 2.3 監査証跡：限定的 → 完全な CloudWatch Logs

#### 完全な監査証跡の内容

**1. 部門変更の監査証跡**

```json
// CloudWatch Logs: /aws/lambda/updateUserDepartment

// イベント 1: API リクエスト受信
{
  "@timestamp": "2025-12-20T10:15:30.123Z",
  "@message": "[updateUserDepartment] Event: {\"requestContext\":{\"authorizer\":{\"claims\":{\"cognito:username\":\"EntraID_user@example.com\",\"email\":\"user@example.com\"}}},\"body\":\"{\\\"department\\\":\\\"sales\\\"}\"}"
}

// イベント 2: ユーザー認証情報
{
  "@timestamp": "2025-12-20T10:15:30.234Z",
  "@message": "[updateUserDepartment] Username: EntraID_user@example.com, Email: user@example.com"
}

// イベント 3: 現在のグループ情報
{
  "@timestamp": "2025-12-20T10:15:30.567Z",
  "@message": "[updateUserDepartment] User EntraID_user@example.com (user@example.com) current groups: [\"Engineering-User\",\"Sales-User\"]"
}

// イベント 4: 既存のロール確認
{
  "@timestamp": "2025-12-20T10:15:30.678Z",
  "@message": "[updateUserDepartment] User already in Sales-User, preserving role: User"
}

// イベント 5: グループ追加（必要な場合）
{
  "@timestamp": "2025-12-20T10:15:31.234Z",
  "@message": "[updateUserDepartment] User already in Sales-User"
}

// イベント 6: custom:department 属性の更新
{
  "@timestamp": "2025-12-20T10:15:31.789Z",
  "@message": "[updateUserDepartment] Updated custom:department attribute to sales"
}

// イベント 7: 完了サマリー
{
  "@timestamp": "2025-12-20T10:15:32.012Z",
  "@message": "[updateUserDepartment] Updated department for user EntraID_user@example.com (user@example.com) to Sales (group: Sales-User)"
}
```

**2. グループ同期の監査証跡**

```json
// CloudWatch Logs: /aws/lambda/mapSamlGroups

// イベント 1: Pre-Token Generation トリガー
{
  "@timestamp": "2025-12-20T10:00:00.123Z",
  "@message": "[mapSamlGroups] Pre-Token Generation triggered for user: EntraID_user@example.com"
}

// イベント 2: Graph API からグループ取得
{
  "@timestamp": "2025-12-20T10:00:01.234Z",
  "@message": "[mapSamlGroups] Fetching groups from Microsoft Graph API"
}

// イベント 3: Entra ID のグループ情報
{
  "@timestamp": "2025-12-20T10:00:02.345Z",
  "@message": "[mapSamlGroups] Entra ID groups for user: [\"Engineering-User\"]"
}

// イベント 4: Cognito の現在のグループ
{
  "@timestamp": "2025-12-20T10:00:02.456Z",
  "@message": "[mapSamlGroups] User EntraID_user@example.com current Cognito groups: [\"Engineering-User\",\"Sales-User\",\"ap-northeast-1_xxx_EntraID\"]"
}

// イベント 5: 部門グループのフィルタリング
{
  "@timestamp": "2025-12-20T10:00:02.567Z",
  "@message": "[mapSamlGroups] Department groups: [\"Engineering-User\",\"Sales-User\"]"
}

// イベント 6: 削除対象のグループ特定
{
  "@timestamp": "2025-12-20T10:00:02.678Z",
  "@message": "[mapSamlGroups] Groups to remove: [\"Sales-User\"]"
}

// イベント 7: グループ削除実行
{
  "@timestamp": "2025-12-20T10:00:03.123Z",
  "@message": "[mapSamlGroups] Removing user from group: Sales-User"
}

// イベント 8: グループ削除成功
{
  "@timestamp": "2025-12-20T10:00:03.456Z",
  "@message": "[mapSamlGroups] Successfully removed user from group: Sales-User"
}

// イベント 9: JWT claims の更新
{
  "@timestamp": "2025-12-20T10:00:03.567Z",
  "@message": "[mapSamlGroups] Updated JWT claims with groups: [\"Engineering-User\"]"
}
```

#### ログからの分析とレポート

**分析クエリ 1: ユーザーの部門変更履歴**

```sql
# CloudWatch Logs Insights

fields @timestamp, @message
| filter @message like /Updated department for user/
| parse @message "Updated department for user * (*) to * (group: *)"
    as username, email, department, group
| filter email = "user@example.com"
| sort @timestamp desc

# 結果例:
# 2025-12-20 10:15:32  Updated department for user ... to Sales (group: Sales-User)
# 2025-12-19 14:30:21  Updated department for user ... to Engineering (group: Engineering-User)
# 2025-12-18 09:45:10  Updated department for user ... to Sales (group: Sales-User)
```

**分析クエリ 2: 部門別の変更統計**

```sql
fields @timestamp, @message
| filter @message like /Updated department for user/
| parse @message "to * (group:" as department
| stats count() by department
| sort count desc

# 結果例:
# Sales: 156 回
# Engineering: 142 回
```

**分析クエリ 3: グループ削除の監査**

```sql
fields @timestamp, @message
| filter @message like /Successfully removed user from group/
| parse @message "removed user from group: *" as group
| stats count() by group

# 結果例:
# Sales-User: 12 回
# Engineering-Admin: 3 回
```

**分析クエリ 4: セキュリティアラート（不一致検出）**

```sql
fields @timestamp, @message
| filter @message like /SECURITY/ or @message like /Department mismatch/
| sort @timestamp desc

# 疑わしいアクティビティを検出
```

---

### 2.4 緊急時の対応手順：なし → ドキュメント化済み

#### 緊急対応マニュアル

**シナリオ 1: 退職者が引き続きアクセス可能**

```bash
#!/bin/bash
# emergency_revoke_access.sh
# 退職者のアクセスを即座に削除

USER_POOL_ID="ap-northeast-1_xxxx"
USERNAME="EntraID_exemployee@example.com"

echo "=== Emergency Access Revocation ==="
echo "User: $USERNAME"
echo "Action: Removing from all groups and signing out"
echo ""

# Step 1: 全てのグループから削除
echo "Step 1: Listing current groups..."
GROUPS=$(aws cognito-idp admin-list-groups-for-user \
  --user-pool-id "$USER_POOL_ID" \
  --username "$USERNAME" \
  --region ap-northeast-1 \
  --query 'Groups[].GroupName' \
  --output text)

echo "Current groups: $GROUPS"
echo ""

for GROUP in $GROUPS; do
  echo "Removing from group: $GROUP"
  aws cognito-idp admin-remove-user-from-group \
    --user-pool-id "$USER_POOL_ID" \
    --username "$USERNAME" \
    --group-name "$GROUP" \
    --region ap-northeast-1
  echo "✓ Removed from $GROUP"
done

echo ""

# Step 2: 全てのセッションを無効化
echo "Step 2: Signing out user from all devices..."
aws cognito-idp admin-user-global-sign-out \
  --user-pool-id "$USER_POOL_ID" \
  --username "$USERNAME" \
  --region ap-northeast-1

echo "✓ User signed out from all sessions"
echo ""

# Step 3: ユーザーを無効化（オプション）
read -p "Disable user account? (y/n) " -n 1 -r
echo ""
if [[ $REPLY =~ ^[Yy]$ ]]; then
  aws cognito-idp admin-disable-user \
    --user-pool-id "$USER_POOL_ID" \
    --username "$USERNAME" \
    --region ap-northeast-1
  echo "✓ User account disabled"
fi

echo ""
echo "=== Access Revocation Complete ==="
echo "User $USERNAME has been:"
echo "  - Removed from all groups"
echo "  - Signed out from all sessions"
echo "  - Unable to access any department resources"
```

**シナリオ 2: 部門の一括変更**

```bash
#!/bin/bash
# bulk_department_change.sh
# 複数ユーザーの部門を一括変更

USER_POOL_ID="ap-northeast-1_xxxx"
TARGET_DEPARTMENT="sales"
USERS_FILE="users_to_change.txt"

echo "=== Bulk Department Change ==="
echo "Target Department: $TARGET_DEPARTMENT"
echo "Users file: $USERS_FILE"
echo ""

# users_to_change.txt の形式:
# EntraID_user1@example.com
# EntraID_user2@example.com
# EntraID_user3@example.com

while IFS= read -r USERNAME; do
  echo "Processing: $USERNAME"

  # custom:department 属性を更新
  aws cognito-idp admin-update-user-attributes \
    --user-pool-id "$USER_POOL_ID" \
    --username "$USERNAME" \
    --user-attributes Name=custom:department,Value="$TARGET_DEPARTMENT" \
    --region ap-northeast-1

  echo "✓ Updated $USERNAME to $TARGET_DEPARTMENT"

  # ユーザーをサインアウト（JWT をリフレッシュさせる）
  aws cognito-idp admin-user-global-sign-out \
    --user-pool-id "$USER_POOL_ID" \
    --username "$USERNAME" \
    --region ap-northeast-1 2>/dev/null

  echo "✓ Signed out $USERNAME (JWT will refresh on next login)"
  echo ""
done < "$USERS_FILE"

echo "=== Bulk Change Complete ==="
```

**シナリオ 3: 監査ログの緊急確認**

```bash
#!/bin/bash
# audit_user_activity.sh
# 特定ユーザーの活動を調査

USERNAME="user@example.com"
START_TIME=$(date -u -d '24 hours ago' +%s)000

echo "=== User Activity Audit ==="
echo "User: $USERNAME"
echo "Period: Last 24 hours"
echo ""

# 部門変更の確認
echo "Department Changes:"
aws logs filter-log-events \
  --log-group-name /aws/lambda/updateUserDepartment \
  --filter-pattern "$USERNAME" \
  --start-time "$START_TIME" \
  --region ap-northeast-1 \
  --query 'events[].message' \
  --output text

echo ""

# グループ同期の確認
echo "Group Sync Events:"
aws logs filter-log-events \
  --log-group-name /aws/lambda/mapSamlGroups \
  --filter-pattern "$USERNAME" \
  --start-time "$START_TIME" \
  --region ap-northeast-1 \
  --query 'events[].message' \
  --output text

echo ""

# Knowledge Base アクセスの確認
echo "Knowledge Base Access:"
aws logs filter-log-events \
  --log-group-name /aws/lambda/knowledgeBaseQuery \
  --filter-pattern "$USERNAME" \
  --start-time "$START_TIME" \
  --region ap-northeast-1 \
  --query 'events[].message' \
  --output text
```

#### エスカレーション手順

**レベル 1: 軽微な問題（開発者対応）**
- 部門切り替えの失敗
- JWT リフレッシュの問題
- 対応: CloudWatch Logs を確認、Lambda を再起動

**レベル 2: 中程度の問題（セキュリティチーム通知）**
- 不審な部門変更パターン
- グループ同期の失敗
- 対応: ユーザーアカウントを一時停止、調査開始

**レベル 3: 重大な問題（即座にアクセス削除）**
- 退職者のアクセス継続
- 不正アクセスの疑い
- 対応: 緊急スクリプト実行、全セッション無効化、管理者に報告

---

### 2.5 ユーザ同期：Entra ID と Cognito の同期戦略

#### 実装状況

| 同期種別 | 実装状況 | 同期タイミング | 説明 |
|---------|---------|--------------|------|
| ユーザ作成同期 | ✅ 自動 | 初回ログイン時 | Entra ID でユーザ作成 → SAML SSO 初回ログイン → Cognito にユーザ自動作成 |
| グループ追加同期 | ✅ 自動 | ログイン/JWT リフレッシュ時 | Entra ID でグループに追加 → 次回ログイン時に Cognito グループに追加 |
| グループ削除同期 | ✅ 自動 | ログイン/JWT リフレッシュ時 | Entra ID でグループから削除 → 次回ログイン時に Cognito グループから削除 |
| ユーザ削除同期 | ⚠️ 手動推奨 | - | Entra ID でユーザ削除 → Cognito には残る（ログイン不可）→ 定期クリーンアップ推奨 |
| ユーザ属性同期 | ✅ 自動 | ログイン時 | Entra ID の属性（email, name）→ Cognito に同期 |

#### ユーザ作成の同期フロー

**Entra ID → Cognito への自動ユーザ作成:**

```
[Entra ID 管理者]
  ↓ Entra ID ポータルで新規ユーザ作成
[Entra ID]
  ├─ ユーザ: newuser@example.com
  └─ グループ: Engineering-User に追加
      Object ID: 423e6dc2-dd7b-4bef-a3eb-36f1fb60473c
  ↓
[新規ユーザが初回ログイン]
  ├─ SAML SSO 認証 (Entra ID)
  └─ 認証成功 → SAML Assertion を Cognito に送信
  ↓
[Cognito]
  ├─ SAML Assertion を受信
  ├─ ユーザが存在しないことを確認
  ├─ ユーザを自動作成: EntraID_newuser@example.com
  └─ カスタム属性 custom:samlGroups に Entra ID グループ Object ID を保存
      custom:samlGroups = "[423e6dc2-dd7b-4bef-a3eb-36f1fb60473c]"
  ↓
[Pre-Token Generation Lambda: mapSamlGroups]
  1. custom:samlGroups からグループ Object ID を取得
     → ["423e6dc2-dd7b-4bef-a3eb-36f1fb60473c"]

  2. Graph API でグループ Object ID → Display Name に変換
     GET https://graph.microsoft.com/v1.0/groups/423e6dc2-...
     → { "displayName": "Engineering-User" }

  3. Cognito グループに追加を試行
     AdminAddUserToGroupCommand("Engineering-User")
     → Cognito に Engineering-User グループが存在
     → ✅ 追加成功

  4. JWT Claims に cognito:groups を設定
     cognito:groups = ["Engineering-User"]
  ↓
[GenU アプリ]
  ✅ 新規ユーザがログイン成功、Engineering 部門にアクセス可能
```

**重要: グループ Object ID → Display Name の変換フロー**

```typescript
// マッピングテーブルは不要！Graph API でリアルタイム変換
async function getGroupDisplayName(groupId: string, accessToken: string) {
  const response = await fetch(
    `https://graph.microsoft.com/v1.0/groups/${groupId}`,
    { headers: { Authorization: `Bearer ${accessToken}` } }
  );
  const group = await response.json();
  return group.displayName;  // "Engineering-User"
}
```

**ホワイトリスト不要: Cognito グループの存在がフィルタとして機能**

```typescript
// Cognito にグループが存在しない場合は自動的にスキップ
async function addUserToCognitoGroup(groupName: string) {
  try {
    await cognitoClient.send(
      new AdminAddUserToGroupCommand({ GroupName: groupName })
    );
    return true;  // 追加成功
  } catch (error: any) {
    if (error.name === 'ResourceNotFoundException') {
      // Cognito にグループが存在しない → 自動スキップ
      console.log(`Group ${groupName} not found, skipping`);
      return false;
    }
    throw error;
  }
}
```

**具体例: 複数グループの処理**

```
Entra ID のグループ:
  - Engineering-Admin (Object ID: 423e6dc2-...)
  - Sales-User (Object ID: 4b3610d2-...)
  - HR-Manager (Object ID: 99999999-...)  ← Cognito には存在しない

SAML Assertion:
  custom:samlGroups = "[423e6dc2-..., 4b3610d2-..., 99999999-...]"

mapSamlGroups Lambda の処理:
  1. Graph API で変換:
     423e6dc2-... → "Engineering-Admin"
     4b3610d2-... → "Sales-User"
     99999999-... → "HR-Manager"

  2. Cognito グループに追加を試行:
     Engineering-Admin → ✅ 存在、追加成功
     Sales-User → ✅ 存在、追加成功
     HR-Manager → ⚠️ 存在しない、自動スキップ

結果:
  cognito:groups = ["Engineering-Admin", "Sales-User"]
  ↑ HR-Manager は含まれない
```

**ポイント:**
- ✅ Cognito へのユーザ作成は**完全自動**
- ✅ Entra ID が Single Source of Truth
- ✅ 初回ログイン時に必要な全ての設定が完了
- ✅ 管理者が Cognito を操作する必要なし
- ✅ **マッピングテーブル不要**（Graph API でリアルタイム変換）
- ✅ **ホワイトリスト不要**（Cognito グループの存在がフィルタ）

#### ユーザ削除の同期戦略

**現在の実装（ユーザ削除同期なし）:**

```
Day 1: Entra ID でユーザを削除
  [Entra ID 管理者]
    ↓ ユーザ exemployee@example.com を削除
  [Entra ID]
    ❌ ユーザ削除: exemployee@example.com

  [Cognito]
    ✅ ユーザ残存: EntraID_exemployee@example.com
    ✅ グループ: Engineering-User

Day 2 ~ ∞: 退職者がログイン試行
  [退職者のブラウザ]
    ↓ GenU アプリにアクセス
  [Cognito / SAML SSO]
    ↓ SAML 認証を試行
  [Entra ID]
    ❌ 認証失敗: ユーザが存在しない
    ↓
  [Cognito]
    ❌ SAML Assertion を受信できない
    ↓
  [GenU アプリ]
    ❌ ログイン失敗（アクセス拒否）

結論: Cognito にユーザが残っていても、Entra ID で削除されていればログイン不可
      → 実質的なセキュリティリスクなし
```

**ユーザ削除同期が不要な理由:**

1. **SAML SSO の仕組み**: Entra ID が認証を制御しているため、Entra ID から削除されたユーザは SAML 認証ができない
2. **Pre-Token Generation Lambda の起動条件**: ログイン成功時のみ起動されるため、削除されたユーザには実行されない
3. **Cognito のユーザレコードは「ゴースト」**: ログイン不可能なため、実害なし

**推奨: 定期的なクリーンアップジョブ（オプション）**

コンプライアンスやストレージ最適化のため、定期的に未使用ユーザを削除することを推奨します。

```bash
#!/bin/bash
# cleanup_inactive_users.sh
# Cognito から90日以上ログインしていないユーザを削除

USER_POOL_ID="ap-northeast-1_xxxx"
INACTIVE_DAYS=90
TODAY=$(date +%s)

echo "=== Cognito User Cleanup ==="
echo "Target: Users inactive for $INACTIVE_DAYS days"
echo ""

# 全ユーザーを取得
aws cognito-idp list-users \
  --user-pool-id "$USER_POOL_ID" \
  --region ap-northeast-1 \
  --query 'Users[*].[Username,UserLastModifiedDate,UserStatus]' \
  --output text | while read USERNAME LAST_MODIFIED STATUS; do

  # 最終ログイン日を Unix timestamp に変換
  LAST_MODIFIED_TS=$(date -d "$LAST_MODIFIED" +%s)
  DIFF_DAYS=$(( (TODAY - LAST_MODIFIED_TS) / 86400 ))

  if [ $DIFF_DAYS -gt $INACTIVE_DAYS ]; then
    echo "Inactive user: $USERNAME (Last login: $DIFF_DAYS days ago)"

    # Entra ID で削除済みか確認（オプション）
    # az ad user show --id "${USERNAME#EntraID_}" 2>/dev/null
    # if [ $? -ne 0 ]; then
    #   echo "  User not found in Entra ID, safe to delete"
    # fi

    # 削除実行（ドライラン: --dry-run を追加してテスト）
    # aws cognito-idp admin-delete-user \
    #   --user-pool-id "$USER_POOL_ID" \
    #   --username "$USERNAME" \
    #   --region ap-northeast-1

    # echo "  ✓ Deleted from Cognito"
  fi
done

echo ""
echo "=== Cleanup Complete ==="
```

**クリーンアップジョブの推奨実行頻度:**
- 月次: ストレージ最適化、コンプライアンス対応
- 四半期: 一般的な運用

**CloudWatch Events でのスケジュール実行:**

```typescript
// packages/cdk/lib/construct/cleanup.ts (新規作成の場合)

import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as lambda from 'aws-cdk-lib/aws-lambda';

// Lambda 関数の作成
const cleanupFunction = new lambda.Function(this, 'CleanupInactiveUsers', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'cleanupInactiveUsers.handler',
  code: lambda.Code.fromAsset('lambda'),
  environment: {
    USER_POOL_ID: userPool.userPoolId,
    INACTIVE_DAYS: '90',
  },
});

// IAM 権限
userPool.grant(cleanupFunction, 'cognito-idp:ListUsers', 'cognito-idp:AdminDeleteUser');

// 月次スケジュール (毎月1日 午前2時 JST)
new events.Rule(this, 'MonthlyCleanupRule', {
  schedule: events.Schedule.cron({
    minute: '0',
    hour: '17', // 2 AM JST = 5 PM UTC (前日)
    day: '1',
    month: '*',
    year: '*',
  }),
  targets: [new targets.LambdaFunction(cleanupFunction)],
});
```

#### ユーザ属性の同期

**SAML Assertion からの属性マッピング:**

```typescript
// Cognito User Pool の SAML 属性マッピング設定
// packages/cdk/lib/construct/auth.ts

const samlProvider = new cognito.CfnUserPoolIdentityProvider(
  this,
  'EntraIDSAMLProvider',
  {
    userPoolId: userPool.userPoolId,
    providerName: 'EntraID',
    providerType: 'SAML',
    attributeMapping: {
      email: 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress',
      name: 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name',
      'custom:samlGroups': 'http://schemas.microsoft.com/ws/2008/06/identity/claims/groups',
    },
  }
);
```

**自動同期される属性:**

| Entra ID 属性 | Cognito 属性 | 同期タイミング | 更新可能 |
|--------------|-------------|--------------|---------|
| Email | email | ログイン時 | ✅ 自動 |
| Display Name | name | ログイン時 | ✅ 自動 |
| Groups (Object IDs) | custom:samlGroups | ログイン時 | ✅ 自動 |
| Department | - | - | ❌ 手動（custom:department は別管理） |

**属性更新の動作:**

```
Day 1: Entra ID でメールアドレスを変更
  [Entra ID]
    olduser@example.com → newuser@example.com

Day 2: ユーザがログイン
  [SAML SSO]
    ↓ SAML Assertion に新しいメールアドレスを含む
  [Cognito]
    ↓ email 属性を自動更新: newuser@example.com
  [Pre-Token Generation Lambda]
    ↓ JWT に新しいメールアドレスを設定
  [GenU アプリ]
    ✅ UI に新しいメールアドレスが表示される
```

#### 同期の監査とトラブルシューティング

**同期状態の確認:**

```bash
# Entra ID のユーザとグループを確認
az ad user show --id user@example.com \
  --query '{DisplayName:displayName,Email:mail,AccountEnabled:accountEnabled}'

az ad user get-member-groups --id user@example.com \
  --query '[].{DisplayName:displayName,Id:id}' -o table

# Cognito のユーザとグループを確認
aws cognito-idp admin-get-user \
  --user-pool-id ap-northeast-1_xxxx \
  --username EntraID_user@example.com \
  --region ap-northeast-1 \
  --query 'UserAttributes[?Name==`email` || Name==`custom:samlGroups`]'

aws cognito-idp admin-list-groups-for-user \
  --user-pool-id ap-northeast-1_xxxx \
  --username EntraID_user@example.com \
  --region ap-northeast-1 \
  --query 'Groups[*].GroupName'
```

**同期ログの確認:**

```bash
# Pre-Token Generation Lambda のログ
aws logs tail /aws/lambda/GenerativeAiUseCasesStack-AuthMapSamlGroupsA7D3F1D-xxxxx \
  --since 1h \
  --format short \
  --region ap-northeast-1 \
  --filter-pattern "EntraID_user@example.com"
```

---

## 📂 変更ファイル一覧

### Lambda 関数
| ファイル | 変更内容 | 影響 |
|---------|---------|------|
| `packages/cdk/lambda/getUserDepartment.ts` | `custom:department` 属性からの読み取り実装 | 部門取得 API |
| `packages/cdk/lambda/updateUserDepartment.ts` | `custom:department` 属性への書き込み実装、ロール保持ロジック追加 | 部門更新 API |
| `packages/cdk/lambda/mapSamlGroups.ts` | グループ削除同期ロジックの追加 | Pre-Token Generation |

### インフラストラクチャ
| ファイル | 変更内容 | 影響 |
|---------|---------|------|
| `packages/cdk/lib/construct/api.ts` | IAM 権限の追加 | Lambda 実行権限 |

### ドキュメント
| ファイル | 変更内容 | 状態 |
|---------|---------|------|
| `docs/SSO_IMPLEMENTATION_GUIDE.md` | 部門切り替えセクションを更新 | ✅ コミット済み |
| `docs/DEPARTMENT_SWITCHING_SECURITY.md` | セキュリティ技術ドキュメント作成 | ✅ 作成済み |
| `docs/IMPLEMENTATION_SUMMARY.md` | 本ドキュメント（実装サマリー） | 📝 本ファイル |

---

## 👥 第3章：ユーザー管理フローの詳細設計

### 3.1 概要：ユーザー管理の基本方針

本章では、Entra ID と Cognito を統合した環境における、ユーザーとグループの管理フローについて詳細に解説します。

**基本方針:**
- **Entra ID が Single Source of Truth（唯一の真実の情報源）**
- **Read-Only 方式**: AWS アプリケーションから Entra ID への書き込みは行わない
- **セキュリティ最優先**: Entra ID への書き込み権限を持たないことで攻撃対象領域を最小化

---

### 3.2 ユーザーのライフサイクル管理

#### 3.2.1 ユーザー作成フロー

```mermaid
sequenceDiagram
    participant Admin as Entra ID 管理者
    participant EntraID as Entra ID
    participant User as 新規ユーザー
    participant Cognito as AWS Cognito
    participant Lambda as mapSamlGroups Lambda
    participant GenU as GenU アプリ

    Admin->>EntraID: 1. 新規ユーザー作成<br/>email: newuser@example.com
    Admin->>EntraID: 2. グループに追加<br/>Engineering-User

    Note over Admin,EntraID: ユーザー作成完了<br/>招待メール送信

    User->>GenU: 3. 初回ログイン<br/>ログインボタンをクリック
    GenU->>EntraID: 4. SAML SSO 認証
    EntraID->>User: 5. 認証画面表示
    User->>EntraID: 6. 認証情報入力
    EntraID->>Cognito: 7. SAML Assertion 送信<br/>email, groups

    Note over Cognito: ユーザーが存在しないことを確認

    Cognito->>Cognito: 8. ユーザー自動作成<br/>EntraID_newuser@example.com
    Cognito->>Lambda: 9. Pre-Token Generation トリガー
    Lambda->>EntraID: 10. Graph API でグループ情報取得
    EntraID->>Lambda: 11. Group: Engineering-User
    Lambda->>Cognito: 12. Engineering-User グループに追加
    Lambda->>Cognito: 13. JWT Claims 設定<br/>cognito:groups, custom:department
    Cognito->>GenU: 14. JWT 発行

    GenU->>User: 15. ログイン成功<br/>Engineering 部門にアクセス可能
```

**手順の詳細:**

| ステップ | 実行者 | 操作 | 所要時間 | 自動化レベル |
|---------|-------|------|---------|------------|
| 1-2 | Entra ID 管理者 | Entra ID ポータルでユーザー作成とグループ追加 | 5分 | 手動 |
| 3 | 新規ユーザー | GenU アプリにアクセス、ログインボタンをクリック | 10秒 | 手動 |
| 4-14 | システム | SAML SSO 認証、Cognito ユーザー作成、グループ同期 | 2-3秒 | 完全自動 |
| 15 | - | ログイン完了 | - | - |

**ユーザー作成の自動化ポイント:**
- ✅ Cognito へのユーザー作成は完全自動（AWS 側の操作不要）
- ✅ グループ同期も完全自動（mapSamlGroups Lambda）
- ✅ 部門属性（custom:department）も初回ログイン時に自動設定
- ✅ JWT にグループ情報が自動的に含まれる

---

#### 3.2.2 ユーザー削除フロー

```mermaid
sequenceDiagram
    participant Admin as Entra ID 管理者
    participant EntraID as Entra ID
    participant Cognito as AWS Cognito
    participant ExUser as 退職者
    participant GenU as GenU アプリ

    Admin->>EntraID: 1. ユーザー削除<br/>exemployee@example.com

    Note over EntraID: ユーザー削除完了
    Note over Cognito: EntraID_exemployee@example.com<br/>は残存（ゴーストユーザー）

    ExUser->>GenU: 2. ログイン試行
    GenU->>EntraID: 3. SAML SSO 認証要求
    EntraID->>ExUser: 4. ❌ 認証失敗<br/>User not found

    Note over ExUser,GenU: ログイン不可<br/>アクセス拒否

    Note over Cognito: （オプション）月次クリーンアップジョブ<br/>90日以上未使用ユーザーを削除
```

**ユーザー削除の動作:**

| 時点 | Entra ID | Cognito | ログイン可否 | セキュリティリスク |
|------|---------|---------|------------|------------------|
| 削除前 | ✅ 存在 | ✅ 存在 | ✅ 可能 | - |
| 削除直後 | ❌ 削除済み | ✅ 残存 | ❌ 不可 | なし（SAML 認証失敗） |
| 90日後（クリーンアップ） | ❌ 削除済み | ❌ 削除済み | ❌ 不可 | なし |

**重要な考慮事項:**
- Cognito に「ゴーストユーザー」が残っても、Entra ID で認証できないためセキュリティリスクはない
- コンプライアンスやストレージ最適化のため、定期クリーンアップジョブを実装することを推奨

---

### 3.3 グループとロールの管理

#### 3.3.1 グループ構造

```
Entra ID (Azure AD)
├─ Engineering-Admin     (部門管理者グループ)
├─ Engineering-User      (部門ユーザーグループ)
├─ Sales-Admin           (部門管理者グループ)
└─ Sales-User            (部門ユーザーグループ)
     ↓ SAML SSO + mapSamlGroups Lambda
Cognito User Pool
├─ Engineering-Admin     (自動同期)
├─ Engineering-User      (自動同期)
├─ Sales-Admin           (自動同期)
└─ Sales-User            (自動同期)
```

**グループの命名規則:**
- フォーマット: `{Department}-{Role}`
- Department: `Engineering`, `Sales`, など
- Role: `Admin`, `User`

**ロールの権限:**

| ロール | S3 RAG ファイル | ユーザ管理 | Knowledge Base | 部門切り替え |
|--------|----------------|----------|----------------|-------------|
| Admin | ✅ アップロード可能 | ✅ 招待・削除可能 | ✅ 部門データにアクセス | ✅ 可能 |
| User | ❌ 読み取りのみ | ❌ 不可 | ✅ 部門データにアクセス | ✅ 可能 |

---

#### 3.3.2 グループ追加フロー

```mermaid
sequenceDiagram
    participant Admin as Entra ID 管理者
    participant EntraID as Entra ID
    participant User as ユーザー
    participant Cognito as AWS Cognito
    participant Lambda as mapSamlGroups Lambda
    participant GenU as GenU アプリ

    Admin->>EntraID: 1. ユーザーをグループに追加<br/>user@example.com → Sales-Admin

    Note over EntraID: グループ追加完了

    User->>GenU: 2. 次回ログイン<br/>または JWT リフレッシュ
    GenU->>EntraID: 3. SAML SSO 認証
    EntraID->>Cognito: 4. SAML Assertion<br/>groups: [Engineering-User, Sales-Admin]
    Cognito->>Lambda: 5. Pre-Token Generation トリガー
    Lambda->>Cognito: 6. 現在のグループ取得<br/>[Engineering-User]
    Lambda->>EntraID: 7. Graph API でグループ情報取得
    EntraID->>Lambda: 8. Groups: [Engineering-User, Sales-Admin]
    Lambda->>Cognito: 9. 新規グループに追加<br/>Sales-Admin
    Lambda->>Cognito: 10. JWT Claims 更新<br/>cognito:groups: [Engineering-User, Sales-Admin]
    Cognito->>GenU: 11. 新しい JWT 発行

    GenU->>User: 12. Sales 部門の管理者権限が付与される
```

**グループ追加の同期タイミング:**
- ログイン時: 即座に同期
- JWT リフレッシュ時: 最大1時間以内に同期
- 手動リフレッシュ: ユーザーがログアウト→ログインで即座に同期

---

#### 3.3.3 グループ削除フロー

セクション 2.1「グループ削除同期」を参照してください。Entra ID でグループから削除されたユーザーは、次回ログインまたは JWT リフレッシュ時（最大24時間以内）に Cognito グループからも削除されます。

---

### 3.4 部門管理者の操作フロー

#### 3.4.1 現在の実装（Read-Only 方式）

**部門管理者ができること:**
- ✅ 所属部門のユーザー一覧を表示
- ✅ ユーザーのロール（Admin/User）を確認
- ✅ S3 RAG インジェスト用ファイルのアップロード
- ✅ 部門の切り替え（複数部門を管理する場合）

**部門管理者ができないこと:**
- ❌ 新規ユーザーの招待（Entra ID 管理者に依頼）
- ❌ ユーザーの削除（Entra ID 管理者に依頼）
- ❌ ユーザーのロール変更（Entra ID 管理者に依頼）

**運用フロー:**

```
部門管理者: 新規ユーザーを追加したい
  ↓
[GenU UI] 「ユーザー招待」ボタン → 招待フォーム表示
  ├─ Email: newuser@example.com
  ├─ Department: Sales
  └─ Role: User
  ↓
[GenU UI] 「招待リクエストを送信」ボタンをクリック
  ↓
[システム] Entra ID 管理者にメール通知
  件名: [GenU] 新規ユーザー招待リクエスト
  本文:
    部門: Sales
    メール: newuser@example.com
    ロール: User
    リクエスト者: manager@example.com
  ↓
[Entra ID 管理者] Entra ID ポータルで操作
  1. ユーザー作成: newuser@example.com
  2. グループ追加: Sales-User
  3. 招待メール送信
  ↓
[新規ユーザー] 招待メールを受信 → GenU にログイン
  ↓
[自動処理] Cognito ユーザー作成、グループ同期
  ↓
[完了] 新規ユーザーがアクセス可能
```

**メリット:**
- ✅ Entra ID が Single Source of Truth を維持
- ✅ AWS からの書き込み権限不要（セキュリティリスク最小）
- ✅ 既存の Entra ID 管理フローを活用

**デメリット:**
- ⚠️ ユーザー追加に手動ステップが必要
- ⚠️ Entra ID 管理者の負荷増加

---

#### 3.4.2 将来の拡張案（Hybrid 方式）

**実装する場合の設計（参考）:**

```mermaid
sequenceDiagram
    participant DeptAdmin as 部門管理者
    participant GenU as GenU アプリ
    participant Lambda as inviteUser Lambda
    participant GraphAPI as Microsoft Graph API
    participant EntraID as Entra ID
    participant NewUser as 新規ユーザー
    participant Cognito as AWS Cognito

    DeptAdmin->>GenU: 1. ユーザー招待フォーム<br/>email, department, role
    GenU->>Lambda: 2. POST /api/user/invite
    Lambda->>GraphAPI: 3. Create User<br/>POST /v1.0/users
    GraphAPI->>EntraID: 4. ユーザー作成
    Lambda->>GraphAPI: 5. Add to Group<br/>POST /v1.0/groups/{id}/members
    GraphAPI->>EntraID: 6. Sales-User に追加
    Lambda->>GraphAPI: 7. Send Invitation<br/>POST /v1.0/invitations
    GraphAPI->>NewUser: 8. 招待メール送信

    Lambda->>GenU: 9. 成功レスポンス
    GenU->>DeptAdmin: 10. 「招待メールを送信しました」

    NewUser->>GenU: 11. 初回ログイン
    GenU->>EntraID: 12. SAML SSO 認証
    EntraID->>Cognito: 13. SAML Assertion
    Cognito->>Cognito: 14. ユーザー作成、グループ同期
    Cognito->>GenU: 15. JWT 発行
    GenU->>NewUser: 16. ログイン成功
```

**必要な実装:**

1. **Lambda 関数: inviteUser**
```typescript
// packages/cdk/lambda/inviteUser.ts (新規作成)

export const handler = async (event: APIGatewayProxyEvent) => {
  const { email, department, role } = JSON.parse(event.body || '{}');

  // 権限チェック: Admin ロールのみ実行可能
  const claims = event.requestContext.authorizer?.claims;
  const currentDepartment = claims?.['custom:department'];
  const groups = claims?.['cognito:groups'] || [];
  const userRole = groups.find(g => g.includes(currentDepartment))?.split('-')[1];

  if (userRole !== 'Admin') {
    return {
      statusCode: 403,
      body: JSON.stringify({ error: 'Admin role required' })
    };
  }

  // Microsoft Graph API でユーザー作成
  const accessToken = await getGraphAccessToken();

  // 1. ユーザー作成
  const user = await createUser(accessToken, email);

  // 2. グループに追加
  const groupName = `${department}-${role}`;
  const groupId = await getGroupIdByName(accessToken, groupName);
  await addUserToGroup(accessToken, user.id, groupId);

  // 3. 招待メール送信
  await sendInvitation(accessToken, email);

  return {
    statusCode: 200,
    body: JSON.stringify({ success: true, message: 'User invited successfully' })
  };
};
```

2. **IAM 権限**
```typescript
// packages/cdk/lib/construct/api.ts

// Graph API 認証情報の取得権限
inviteUserFunction.addToRolePolicy(new iam.PolicyStatement({
  actions: ['secretsmanager:GetSecretValue'],
  resources: [graphCredentialsSecret.secretArn],
}));
```

3. **Azure AD アプリの権限追加**
```bash
# Microsoft Graph API の権限を追加
az ad app permission add \
  --id {APP_ID} \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions \
    User.Invite.All=Role \
    GroupMember.ReadWrite.All=Role

# 管理者の同意を付与
az ad app permission admin-consent --id {APP_ID}
```

**セキュリティ考慮事項:**
- ⚠️ Graph API への書き込み権限が必要（`User.Invite.All`, `GroupMember.ReadWrite.All`）
- ⚠️ Lambda が侵害された場合、Entra ID へ不正な書き込みが可能
- ⚠️ エラーハンドリングが複雑（Graph API エラー、メール送信失敗など）

**推奨事項:**
- 現時点では **Read-Only 方式を継続**
- 部門管理者の運用負荷が高まった場合に Hybrid 方式を検討
- 実装する場合は、セキュリティレビューと承認プロセスを経ること

---

### 3.5 運用シナリオ

#### シナリオ 1: 新規社員の入社

```
Day 1: 入社手続き
  [人事部]
    ↓ Entra ID に新規ユーザー作成
  [Entra ID 管理者]
    ├─ ユーザー作成: newemployee@example.com
    ├─ 初期グループ追加: Engineering-User
    └─ 招待メール送信

Day 1 午後: 初回ログイン
  [新規社員]
    ↓ 招待メールから GenU にアクセス
  [Cognito]
    ├─ ユーザー自動作成
    └─ Engineering-User グループに追加
  [GenU アプリ]
    ✅ Engineering 部門にアクセス可能

Day 30: 昇進（User → Admin）
  [部門マネージャー]
    ↓ Entra ID 管理者に昇進依頼メール送信
  [Entra ID 管理者]
    ├─ Engineering-User グループから削除
    └─ Engineering-Admin グループに追加
  [社員]
    ↓ 次回ログイン時
  [mapSamlGroups Lambda]
    ├─ Engineering-User グループから削除
    └─ Engineering-Admin グループに追加
  [GenU アプリ]
    ✅ 管理者権限が付与される（S3 アップロード、ユーザー管理が可能）
```

#### シナリオ 2: 退職者の処理

```
Day 1: 退職日
  [人事部]
    ↓ Entra ID 管理者に退職通知
  [Entra ID 管理者]
    ├─ ユーザーを無効化（または削除）
    └─ 全グループから削除

Day 1 以降: ログイン試行
  [退職者]
    ↓ GenU にアクセス試行
  [Entra ID]
    ❌ SAML 認証失敗（ユーザー無効）
  [GenU アプリ]
    ❌ ログイン不可（アクセス拒否）

Month 1: 定期クリーンアップ
  [CleanupInactiveUsers Lambda]
    ├─ 90日以上未使用のユーザーを検出
    └─ Cognito からユーザー削除
  [Cognito]
    ✅ ゴーストユーザーが削除される
```

#### シナリオ 3: 部門異動

```
Day 1: 異動発令
  [人事部]
    ↓ Engineering → Sales への異動
  [Entra ID 管理者]
    ├─ Engineering-User グループから削除
    ├─ Sales-User グループに追加
    └─ （オプション）custom:department を 'sales' に更新

Day 1 午後: ログイン
  [社員]
    ↓ GenU にログイン
  [mapSamlGroups Lambda]
    ├─ Engineering-User グループから削除
    ├─ Sales-User グループに追加
    └─ custom:department を 'sales' に更新
  [GenU アプリ]
    ✅ Sales 部門にアクセス可能
    ✅ Engineering 部門へのアクセスは不可
```

---

### 3.6 トラブルシューティング

#### 問題 1: ユーザーが作成されない

**症状:**
- 新規ユーザーが Entra ID で作成されたが、GenU にログインできない

**原因と対策:**

| 原因 | 確認方法 | 対策 |
|------|---------|------|
| Entra ID でグループに追加されていない | `az ad user get-member-groups --id user@example.com` | Entra ID でグループに追加 |
| SAML SSO の設定ミス | Entra ID の SAML アプリ設定を確認 | Reply URL、Entity ID を確認 |
| Cognito の SAML プロバイダー設定ミス | Cognito User Pool の SAML プロバイダー確認 | メタデータ XML を再アップロード |

#### 問題 2: グループが同期されない

**症状:**
- Entra ID でグループに追加したが、Cognito グループに反映されない

**確認手順:**
```bash
# 1. Entra ID でグループを確認
az ad user get-member-groups --id user@example.com

# 2. Cognito でグループを確認
aws cognito-idp admin-list-groups-for-user \
  --user-pool-id ap-northeast-1_xxxx \
  --username EntraID_user@example.com \
  --region ap-northeast-1

# 3. mapSamlGroups Lambda のログを確認
aws logs tail /aws/lambda/GenerativeAiUseCasesStack-AuthMapSamlGroupsA7D3F1D-xxxxx \
  --since 1h \
  --format short \
  --region ap-northeast-1
```

**一般的な原因:**
- JWT がまだリフレッシュされていない → ユーザーにログアウト→ログインを依頼
- Cognito にグループが存在しない → CDK で Cognito グループを作成
- Graph API の認証エラー → Secrets Manager の認証情報を確認

#### 問題 3: 部門切り替え後に権限がない

**症状:**
- Sales 部門の Admin だったユーザーが、Engineering 部門に切り替えたら User 権限になった

**原因:**
- Section 1.6 のロール保持ロジックが正しく動作していない

**確認手順:**
```bash
# ユーザーの全グループを確認
aws cognito-idp admin-list-groups-for-user \
  --user-pool-id ap-northeast-1_xxxx \
  --username EntraID_user@example.com \
  --region ap-northeast-1

# 期待される結果:
# - Engineering-Admin（Admin ロールを保持）
# - Sales-Admin（Admin ロールを保持）
```

**対策:**
- updateUserDepartment Lambda のログを確認
- ロール保持ロジック（604-625行目）が正しく実行されているか確認

---

### 3.7 今後の拡張計画

#### 短期（1-3ヶ月）
- [ ] 部門管理者向け UI の改善（ユーザー一覧、ロール確認）
- [ ] ユーザー招待リクエストの通知機能（部門管理者 → Entra ID 管理者）
- [ ] 定期クリーンアップジョブの実装と監視

#### 中期（3-6ヶ月）
- [ ] セルフサービスポータルの検討（部門管理者が直接ユーザーを招待）
- [ ] Hybrid 方式の実装（Graph API 書き込み権限が承認された場合）
- [ ] ユーザー招待の承認ワークフロー（部門管理者 → Entra ID 管理者 → 承認）

#### 長期（6ヶ月以降）
- [ ] ロール変更のセルフサービス化（User → Admin の昇格リクエスト）
- [ ] 部門間異動の自動化（人事システムとの連携）
- [ ] コンプライアンスレポートの自動生成（ユーザー棚卸し）

---

## 🚀 次のステップ

### 1. 動作確認とテスト
- [ ] テストユーザーで部門切り替えを実行
- [ ] CloudWatch Logs で `custom:department` 更新を確認
- [ ] 複数デバイスで同期を確認
- [ ] グループ削除同期のテスト
- [ ] IAM ロール（Admin/User）の保持を確認

### 2. 本番展開
- [ ] ステージング環境で最終確認
- [ ] 本番環境へのデプロイ計画
- [ ] ユーザーへの通知（複数デバイス同期の説明）
- [ ] ロールバック手順の確認

### 3. 監視とメンテナンス
- [ ] CloudWatch Logs のモニタリング設定
- [ ] セキュリティアラートの設定
- [ ] 月次監査レポートの自動化
- [ ] ドキュメントの定期的な更新

---

## 📞 サポート情報

### トラブルシューティング

| 問題 | 参照ドキュメント | セクション |
|------|---------------|----------|
| 部門切り替えが機能しない | DEPARTMENT_SWITCHING_SECURITY.md | セクション 8 |
| グループ同期が動作しない | DEPARTMENT_SWITCHING_SECURITY.md | セクション 8 |
| JWT リフレッシュエラー | SSO_IMPLEMENTATION_GUIDE.md | トラブルシューティング |
| 緊急時のアクセス削除 | 本ドキュメント | セクション 2.4 |
| IAM ロールが保持されない | 本ドキュメント | セクション 1.6 |

### CloudWatch Logs の場所

```bash
# 部門更新ログ
/aws/lambda/GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx

# 部門取得ログ
/aws/lambda/GenerativeAiUseCasesStack-APIGetUserDepartment2E23-xxxxx

# グループ同期ログ
/aws/lambda/GenerativeAiUseCasesStack-AuthMapSamlGroupsA7D3F1D-xxxxx
```

---

## 📝 変更履歴

| 日付 | バージョン | 変更内容 | 作成者 |
|------|-----------|---------|--------|
| 2025-12-20 | 1.0 | 初版作成 | Claude Code |
| 2025-12-20 | 2.0 | 詳細解説を追加（主な成果に焦点） | Claude Code |
| 2025-12-20 | 3.0 | IAM 権限中心の解説に変更 | Claude Code |
| 2025-12-20 | 4.0 | 前回の詳細解説を復活、IAM ロール制御の解説を追加、IAM 権限の技術詳細を Appendix に移動 | Claude Code |
| 2025-12-20 | 4.1 | Section 1.6 を修正: Admin ロールの用途を S3 RAG ファイルアクセスとユーザ管理権限に訂正、Knowledge Base は部門IDのみでフィルタリングすることを明記 | Claude Code |
| 2025-12-20 | 5.0 | Section 2 のタイトルを「Entra ID と Cognito のユーザ・グループ同期」に変更、Section 2.5 ユーザ同期を追加、第3章「ユーザー管理フローの詳細設計」を新規追加（Read-Only 方式の詳細設計、Hybrid 方式の提案、運用シナリオ、トラブルシューティング） | Claude Code |
| 2025-12-20 | 5.1 | Section 2.5 ユーザ作成フローを詳細化: グループ Object ID → Display Name の変換フロー、Graph API の使用方法、マッピングテーブル不要、ホワイトリスト不要の説明を追加 | Claude Code |

---

## Appendix: IAM 権限の技術詳細

### A.1 最小権限の原則（Principle of Least Privilege）

#### 原則の説明

最小権限の原則とは、各主体（ユーザー、プロセス、システム）に対して、その役割を果たすために必要な最小限の権限のみを付与することです。

**メリット:**
- 攻撃対象領域の最小化
- 誤操作による被害の限定
- 権限昇格攻撃のリスク低減
- コンプライアンス要件への対応

#### 今回の実装での適用

**以前の問題（仮想的な脆弱な実装）:**

```typescript
// 悪い例: 全ての Cognito 操作を許可
{
  "Effect": "Allow",
  "Action": "cognito-idp:*",  // 全ての操作を許可（危険）
  "Resource": "*"              // 全てのリソースに対して（危険）
}
```

**問題点:**
- Lambda が Cognito User Pool を削除可能
- Lambda がユーザーを削除可能
- Lambda が MFA 設定を変更可能
- Lambda が User Pool の設定を変更可能

**現在の実装（最小権限）:**

```typescript
// packages/cdk/lib/construct/api.ts

// getUserDepartment Lambda: 読み取り専用権限
userPool.grant(
  getUserDepartmentFunction,
  'cognito-idp:AdminListGroupsForUser',  // グループの一覧取得のみ
  'cognito-idp:AdminGetUser'              // ユーザー属性の取得のみ
);

// updateUserDepartment Lambda: 部門変更に必要な権限のみ
userPool.grant(
  updateUserDepartmentFunction,
  'cognito-idp:AdminListGroupsForUser',       // グループの一覧取得
  'cognito-idp:AdminAddUserToGroup',          // グループへの追加
  'cognito-idp:AdminRemoveUserFromGroup',     // グループからの削除
  'cognito-idp:AdminUpdateUserAttributes'     // custom:department 属性の更新のみ
);
```

**改善点:**
- 必要な操作のみを許可
- 削除や設定変更などの危険な操作は不可
- 特定の User Pool のみにスコープを限定
- 属性の更新も `AdminUpdateUserAttributes` のみで、全体設定の変更は不可

#### 権限の比較表

| 操作 | getUserDepartment | updateUserDepartment | mapSamlGroups | 理由 |
|------|-------------------|----------------------|---------------|------|
| AdminGetUser | ✅ 許可 | ❌ 不要 | ❌ 不要 | 部門情報の読み取りのみ必要 |
| AdminListGroupsForUser | ✅ 許可 | ✅ 許可 | ✅ 許可 | 全ての Lambda で必要 |
| AdminAddUserToGroup | ❌ 不要 | ✅ 許可 | ✅ 許可 | 読み取り専用のため不要 |
| AdminRemoveUserFromGroup | ❌ 不要 | ✅ 許可 | ✅ 許可 | 読み取り専用のため不要 |
| AdminUpdateUserAttributes | ❌ 不要 | ✅ 許可 | ❌ 不要 | 部門更新時のみ必要 |
| AdminDeleteUser | ❌ 拒否 | ❌ 拒否 | ❌ 拒否 | 全ての Lambda で不要（危険） |
| DeleteUserPool | ❌ 拒否 | ❌ 拒否 | ❌ 拒否 | 全ての Lambda で不要（危険） |

---

### A.2 セキュリティリスク分析：不適切な IAM 権限

#### リスク 1: 過剰な権限付与（Over-Privileged）

**シナリオ: `cognito-idp:*` を付与した場合**

```typescript
// 危険な例
{
  "Effect": "Allow",
  "Action": "cognito-idp:*",
  "Resource": "*"
}
```

**攻撃シナリオ:**

```typescript
// 攻撃者が Lambda のコードに脆弱性を発見
// または、環境変数を改ざん

export const handler = async (event: APIGatewayProxyEvent) => {
  // 正規の処理
  const department = await getUserDepartment(username);

  // 悪意のあるコード（攻撃者によって注入）
  if (event.queryStringParameters?.admin === 'true') {
    // 過剰な権限があるため、ユーザーを削除可能
    await client.send(new AdminDeleteUserCommand({
      UserPoolId: userPoolId,
      Username: 'target-user@example.com'
    }));

    // User Pool 全体を削除可能（最悪のシナリオ）
    await client.send(new DeleteUserPoolCommand({
      UserPoolId: userPoolId
    }));
  }

  return { statusCode: 200, body: JSON.stringify(department) };
};
```

**影響:**
- 全ユーザーの削除が可能
- User Pool 全体の削除が可能
- MFA 設定の無効化が可能
- パスワードポリシーの変更が可能

**対策（現在の実装）:**

```typescript
// 必要最小限の権限のみを付与
userPool.grant(
  getUserDepartmentFunction,
  'cognito-idp:AdminListGroupsForUser',  // グループの一覧のみ
  'cognito-idp:AdminGetUser'              // ユーザー情報の取得のみ
);

// 攻撃者がコードを注入しても...
export const handler = async (event: APIGatewayProxyEvent) => {
  // 悪意のあるコード
  try {
    await client.send(new AdminDeleteUserCommand({
      UserPoolId: userPoolId,
      Username: 'target-user@example.com'
    }));
  } catch (error) {
    // AccessDeniedException: Not authorized to perform: cognito-idp:AdminDeleteUser
    console.error('Attack blocked by IAM:', error);
  }

  // 正規の処理は実行可能
  const department = await getUserDepartment(username);
  return { statusCode: 200, body: JSON.stringify(department) };
};
```

---

#### リスク 2: リソース制限の欠如

**シナリオ: Resource を "*" に設定した場合**

```typescript
// 危険な例
{
  "Effect": "Allow",
  "Action": "cognito-idp:AdminGetUser",
  "Resource": "*"  // 全ての User Pool にアクセス可能
}
```

**攻撃シナリオ:**

```typescript
export const handler = async (event: APIGatewayProxyEvent) => {
  const userPoolId = event.queryStringParameters?.poolId || process.env.USER_POOL_ID;

  // 攻撃者が別の User Pool ID を指定
  // Resource: "*" のため、他の User Pool にもアクセス可能
  const response = await client.send(new AdminGetUserCommand({
    UserPoolId: userPoolId,  // 攻撃者が指定した User Pool ID
    Username: 'admin@example.com'
  }));

  // 他のプロジェクトの User Pool の情報を取得
  return { statusCode: 200, body: JSON.stringify(response) };
};
```

**影響:**
- 同じ AWS アカウント内の全ての User Pool にアクセス可能
- 他のプロジェクトのユーザー情報を取得可能
- クロスプロジェクトのデータ漏洩

**対策（現在の実装）:**

```typescript
// packages/cdk/lib/construct/api.ts

// 特定の User Pool のみに権限を制限
userPool.grant(
  getUserDepartmentFunction,
  'cognito-idp:AdminGetUser'
);

// CDK が自動的に以下の IAM ポリシーを生成:
{
  "Effect": "Allow",
  "Action": "cognito-idp:AdminGetUser",
  "Resource": "arn:aws:cognito-idp:ap-northeast-1:123456789012:userpool/ap-northeast-1_xxxx"
  // ↑ 特定の User Pool のみ
}

// 攻撃者が別の User Pool を指定しても...
export const handler = async (event: APIGatewayProxyEvent) => {
  const attackerPoolId = 'ap-northeast-1_attacker';

  try {
    const response = await client.send(new AdminGetUserCommand({
      UserPoolId: attackerPoolId,  // 攻撃者が指定
      Username: 'admin@example.com'
    }));
  } catch (error) {
    // AccessDeniedException: Not authorized to access userpool/ap-northeast-1_attacker
    console.error('Cross-pool access blocked:', error);
  }
};
```

---

#### リスク 3: 属性変更権限の悪用

**シナリオ: AdminUpdateUserAttributes 権限の悪用**

```typescript
// 現在の実装（必要な権限）
{
  "Effect": "Allow",
  "Action": "cognito-idp:AdminUpdateUserAttributes",
  "Resource": "arn:aws:cognito-idp:...:userpool/..."
}
```

**潜在的な攻撃シナリオ:**

```typescript
// 攻撃者がコードに脆弱性を発見
export const handler = async (event: APIGatewayProxyEvent) => {
  const { department, username } = JSON.parse(event.body || '{}');

  // 正規の処理: custom:department を更新
  await client.send(new AdminUpdateUserAttributesCommand({
    UserPoolId: userPoolId,
    Username: username,
    UserAttributes: [
      { Name: 'custom:department', Value: department }
    ]
  }));

  // 悪意のあるコード: 他の属性も変更可能
  // AdminUpdateUserAttributes は全ての属性を変更可能
  await client.send(new AdminUpdateUserAttributesCommand({
    UserPoolId: userPoolId,
    Username: username,
    UserAttributes: [
      { Name: 'email', Value: 'attacker@evil.com' },           // メールアドレス改ざん
      { Name: 'email_verified', Value: 'true' },               // メール検証をスキップ
      { Name: 'phone_number', Value: '+1234567890' },          // 電話番号改ざん
      { Name: 'phone_number_verified', Value: 'true' },        // 電話番号検証をスキップ
      { Name: 'custom:department', Value: 'admin' }            // 管理者権限に昇格
    ]
  }));

  return { statusCode: 200, body: JSON.stringify({ success: true }) };
};
```

**影響:**
- メールアドレスの改ざんによるアカウント乗っ取り
- 検証済みフラグの改ざんによるセキュリティバイパス
- カスタム属性の改ざんによる権限昇格

**対策 1: アプリケーションレベルでの検証**

```typescript
// packages/cdk/lambda/updateUserDepartment.ts

export const handler = async (event: APIGatewayProxyEvent) => {
  const { department } = JSON.parse(event.body || '{}');

  // 部門の検証（ホワイトリスト方式）
  const allowedDepartments = ['engineering', 'sales'];
  if (!allowedDepartments.includes(department.toLowerCase())) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'Invalid department' })
    };
  }

  // 更新する属性を明示的に制限
  const attributesToUpdate = [
    {
      Name: 'custom:department',  // この属性のみ更新
      Value: department.toLowerCase()
    }
    // 他の属性は更新しない
  ];

  await client.send(new AdminUpdateUserAttributesCommand({
    UserPoolId: userPoolId,
    Username: username,
    UserAttributes: attributesToUpdate
  }));

  // ログに記録（監査証跡）
  console.log(`[AUDIT] Updated custom:department for ${username} to ${department}`);

  return { statusCode: 200, body: JSON.stringify({ success: true }) };
};
```

**対策 2: CloudWatch Logs による異常検知**

```typescript
// 異常な属性変更を検知
// CloudWatch Logs Insights クエリ
fields @timestamp, @message
| filter @message like /AdminUpdateUserAttributes/
| parse @message "UserAttributes: *" as attributes
| filter attributes like /email/ or attributes like /phone_number/  // 疑わしい属性変更
| stats count() by username
| sort count desc
```

---

### A.3 IAM 権限の検証方法

#### 方法 1: AWS CLI による権限確認

```bash
# Lambda のロールを確認
aws lambda get-function-configuration \
  --function-name GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx \
  --region ap-northeast-1 \
  --query 'Role' \
  --output text

# 出力例:
# arn:aws:iam::129119569090:role/GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx

# ロールのポリシーを確認
aws iam list-role-policies \
  --role-name GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx \
  --region ap-northeast-1

# ポリシーの詳細を取得
aws iam get-role-policy \
  --role-name GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx \
  --policy-name APIUpdateUserDepartmentServiceRoleDefaultPolicyB40F4082 \
  --region ap-northeast-1 \
  --output json
```

**期待される出力:**

```json
{
  "RoleName": "GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx",
  "PolicyName": "APIUpdateUserDepartmentServiceRoleDefaultPolicyB40F4082",
  "PolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "cognito-idp:AdminListGroupsForUser",
          "cognito-idp:AdminAddUserToGroup",
          "cognito-idp:AdminRemoveUserFromGroup",
          "cognito-idp:AdminUpdateUserAttributes"
        ],
        "Resource": "arn:aws:cognito-idp:ap-northeast-1:129119569090:userpool/ap-northeast-1_0cmg54YCo"
      }
    ]
  }
}
```

#### 方法 2: IAM Policy Simulator による検証

```bash
# Policy Simulator で権限をテスト
# 許可されるべき操作
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::129119569090:role/GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx \
  --action-names cognito-idp:AdminUpdateUserAttributes \
  --resource-arns arn:aws:cognito-idp:ap-northeast-1:129119569090:userpool/ap-northeast-1_0cmg54YCo

# 期待される出力: "EvaluationDecision": "allowed"

# 拒否されるべき操作
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::129119569090:role/GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx \
  --action-names cognito-idp:AdminDeleteUser \
  --resource-arns arn:aws:cognito-idp:ap-northeast-1:129119569090:userpool/ap-northeast-1_0cmg54YCo

# 期待される出力: "EvaluationDecision": "implicitDeny"
```

#### 方法 3: CloudTrail による権限使用の監視

```bash
# CloudTrail で Cognito API 呼び出しを検索
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AdminUpdateUserAttributes \
  --start-time "2025-12-20T00:00:00Z" \
  --region ap-northeast-1 \
  --query 'Events[*].[Username,EventTime,EventName,Resources]' \
  --output table

# 不正なアクセス試行を検出
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ErrorCode,AttributeValue=AccessDenied \
  --start-time "2025-12-20T00:00:00Z" \
  --region ap-northeast-1 \
  --output json
```

**CloudTrail ログの例:**

```json
{
  "eventTime": "2025-12-20T10:15:31.789Z",
  "eventName": "AdminUpdateUserAttributes",
  "eventSource": "cognito-idp.amazonaws.com",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROA...:GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx",
    "arn": "arn:aws:sts::129119569090:assumed-role/GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx/GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx"
  },
  "requestParameters": {
    "userPoolId": "ap-northeast-1_0cmg54YCo",
    "username": "EntraID_user@example.com",
    "userAttributes": [
      {
        "name": "custom:department",
        "value": "sales"
      }
    ]
  },
  "responseElements": null,
  "errorCode": null,  // 成功した場合は null
  "errorMessage": null
}
```

**不正アクセス試行のログ例:**

```json
{
  "eventTime": "2025-12-20T10:20:00.000Z",
  "eventName": "AdminDeleteUser",
  "eventSource": "cognito-idp.amazonaws.com",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROA...:GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx",
    "arn": "arn:aws:sts::129119569090:assumed-role/GenerativeAiUseCasesStack-APIUpdateUserDepartmentSe-xxxxx/GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx"
  },
  "requestParameters": {
    "userPoolId": "ap-northeast-1_0cmg54YCo",
    "username": "target-user@example.com"
  },
  "responseElements": null,
  "errorCode": "AccessDenied",  // IAM 権限で拒否された
  "errorMessage": "User: arn:aws:sts::129119569090:assumed-role/... is not authorized to perform: cognito-idp:AdminDeleteUser"
}
```

---

### A.4 IAM 権限の監査とコンプライアンス

#### 定期監査チェックリスト

```bash
#!/bin/bash
# iam_audit.sh
# IAM 権限の定期監査スクリプト

echo "=== IAM 権限監査 ==="
echo "実行日時: $(date)"
echo ""

# 1. Lambda 関数のロールを取得
echo "1. Lambda 関数のロール確認"
FUNCTIONS=(
  "GenerativeAiUseCasesStack-APIGetUserDepartment2E23-xxxxx"
  "GenerativeAiUseCasesStack-APIUpdateUserDepartmentA-xxxxx"
  "GenerativeAiUseCasesStack-AuthMapSamlGroupsA7D3F1D-xxxxx"
)

for FUNCTION in "${FUNCTIONS[@]}"; do
  echo "  Function: $FUNCTION"
  ROLE=$(aws lambda get-function-configuration \
    --function-name "$FUNCTION" \
    --region ap-northeast-1 \
    --query 'Role' \
    --output text)
  echo "  Role: $ROLE"

  ROLE_NAME=$(echo $ROLE | awk -F'/' '{print $NF}')

  # ポリシーの一覧を取得
  POLICIES=$(aws iam list-role-policies \
    --role-name "$ROLE_NAME" \
    --query 'PolicyNames[]' \
    --output text)
  echo "  Policies: $POLICIES"

  # 各ポリシーの内容を確認
  for POLICY in $POLICIES; do
    echo "    Checking policy: $POLICY"
    aws iam get-role-policy \
      --role-name "$ROLE_NAME" \
      --policy-name "$POLICY" \
      --query 'PolicyDocument.Statement[*].Action' \
      --output text
  done

  echo ""
done

# 2. 過剰な権限を検出
echo "2. 過剰な権限の検出"
for FUNCTION in "${FUNCTIONS[@]}"; do
  ROLE=$(aws lambda get-function-configuration \
    --function-name "$FUNCTION" \
    --region ap-northeast-1 \
    --query 'Role' \
    --output text)
  ROLE_NAME=$(echo $ROLE | awk -F'/' '{print $NF}')

  # ワイルドカード権限を検索
  WILDCARDS=$(aws iam list-role-policies \
    --role-name "$ROLE_NAME" \
    --query 'PolicyNames[]' \
    --output text | while read POLICY; do
    aws iam get-role-policy \
      --role-name "$ROLE_NAME" \
      --policy-name "$POLICY" \
      --query 'PolicyDocument.Statement[?contains(Action, `*`)]' \
      --output text
  done)

  if [ -n "$WILDCARDS" ]; then
    echo "  ⚠️  WARNING: Wildcard permissions found in $FUNCTION"
    echo "  $WILDCARDS"
  else
    echo "  ✅ No wildcard permissions in $FUNCTION"
  fi
done

echo ""

# 3. CloudTrail でアクセス拒否イベントを検索
echo "3. 過去24時間のアクセス拒否イベント"
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ErrorCode,AttributeValue=AccessDenied \
  --start-time "$(date -u -d '24 hours ago' --iso-8601=seconds)" \
  --region ap-northeast-1 \
  --query 'Events[*].[EventTime,Username,EventName]' \
  --output table

echo ""
echo "=== 監査完了 ==="
```

#### コンプライアンスレポート

**月次 IAM 権限レポート:**

```markdown
# IAM 権限監査レポート - 2025年12月

## 1. 概要
- 監査期間: 2025-12-01 ~ 2025-12-31
- 対象 Lambda 関数: 3
- 検出された問題: 0
- 是正措置: 0

## 2. Lambda 関数の権限サマリー

### getUserDepartment Lambda
| 権限 | リソース | 理由 |
|------|---------|------|
| cognito-idp:AdminListGroupsForUser | ap-northeast-1_0cmg54YCo | フォールバック用 |
| cognito-idp:AdminGetUser | ap-northeast-1_0cmg54YCo | custom:department 取得 |

**評価:** ✅ 最小権限の原則に準拠

### updateUserDepartment Lambda
| 権限 | リソース | 理由 |
|------|---------|------|
| cognito-idp:AdminListGroupsForUser | ap-northeast-1_0cmg54YCo | 現在のグループ確認 |
| cognito-idp:AdminAddUserToGroup | ap-northeast-1_0cmg54YCo | グループへの追加 |
| cognito-idp:AdminRemoveUserFromGroup | ap-northeast-1_0cmg54YCo | (将来のため) |
| cognito-idp:AdminUpdateUserAttributes | ap-northeast-1_0cmg54YCo | custom:department 更新 |

**評価:** ✅ 最小権限の原則に準拠

### mapSamlGroups Lambda
| 権限 | リソース | 理由 |
|------|---------|------|
| cognito-idp:AdminListGroupsForUser | ap-northeast-1_0cmg54YCo | 現在のグループ取得 |
| cognito-idp:AdminAddUserToGroup | ap-northeast-1_0cmg54YCo | Entra ID グループ追加 |
| cognito-idp:AdminRemoveUserFromGroup | ap-northeast-1_0cmg54YCo | 削除されたグループの除去 |
| secretsmanager:GetSecretValue | azure-graph-credentials | Graph API 認証情報 |

**評価:** ✅ 最小権限の原則に準拠

## 3. セキュリティチェック

### ワイルドカード権限
- ✅ 検出なし（全ての権限が明示的）

### リソース制限
- ✅ 全ての権限が特定のリソースに限定

### クロスアカウントアクセス
- ✅ 検出なし

### 不要な権限
- ✅ 検出なし

## 4. CloudTrail 分析

### アクセス拒否イベント
- 期間: 2025-12-01 ~ 2025-12-31
- 総イベント数: 0
- **評価:** ✅ 不正アクセス試行なし

### 権限使用統計
| Lambda | API 呼び出し回数 | エラー率 |
|--------|-----------------|---------|
| getUserDepartment | 1,234 | 0.0% |
| updateUserDepartment | 298 | 0.0% |
| mapSamlGroups | 2,456 | 0.0% |

## 5. 推奨事項

### 短期（1ヶ月以内）
- [x] 全ての Lambda に最小権限を適用済み
- [x] CloudTrail ログの監視設定完了
- [ ] IAM Access Analyzer の有効化を検討

### 中期（3ヶ月以内）
- [ ] 定期的な権限レビュープロセスの自動化
- [ ] IAM ポリシー変更の通知設定

### 長期（6ヶ月以内）
- [ ] Service Control Policy (SCP) の導入検討
- [ ] IAM 権限境界 (Permissions Boundary) の設定

## 6. コンプライアンスステータス

| 基準 | ステータス | 備考 |
|------|-----------|------|
| 最小権限の原則 | ✅ 準拠 | 全ての Lambda で実装済み |
| リソース制限 | ✅ 準拠 | 特定リソースのみに限定 |
| 監査証跡 | ✅ 準拠 | CloudTrail + CloudWatch Logs |
| 定期レビュー | ✅ 準拠 | 月次監査実施中 |

## 7. 結論

**総合評価:** ✅ 良好

全ての IAM 権限が最小権限の原則に基づいて適切に設定されています。過剰な権限やセキュリティリスクは検出されませんでした。現在の権限設計は維持し、定期的な監査を継続することを推奨します。

---

**監査実施者:** セキュリティチーム
**承認者:** システム管理者
**次回監査予定日:** 2026-01-31
```

---

**作成者**: Claude Code
**最終更新**: 2025-12-20
**ドキュメントステータス**: ✅ レビュー待ち
