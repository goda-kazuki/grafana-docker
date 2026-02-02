# Grafana Docker + AWS Identity Center 認証

Grafana を Docker で起動し、AWS Identity Center（Cognito 経由）で SSO 認証を行う構成です。

## アーキテクチャ

```
┌─────────────┐     ┌─────────────────┐     ┌──────────────────┐
│   ユーザー   │────▶│    Grafana      │────▶│  Cognito User    │
│             │     │   (Docker)      │     │     Pool         │
│             │◀────│  OAuth2/OIDC    │◀────│                  │
└─────────────┘     └─────────────────┘     └────────┬─────────┘
                                                     │ SAML 2.0
                                                     ▼
                                            ┌──────────────────┐
                                            │  AWS Identity    │
                                            │     Center       │
                                            └──────────────────┘
```

## 前提条件

- Docker / Docker Compose がインストール済み
- AWS CLI が設定済み
- AWS Identity Center が有効化済み
- 適切な IAM 権限（Cognito、CloudFormation）

---

## 構築手順

### Step 1: CloudFormation スタックをデプロイ

Cognito User Pool を作成します。

```bash
aws cloudformation deploy \
  --template-file cognito-identity-center.yaml \
  --stack-name grafana-cognito-dev \
  --region ap-northeast-1 \
  --profile <your-profile>
```

### Step 2: スタックの出力値を取得

```bash
aws cloudformation describe-stacks \
  --stack-name grafana-cognito-dev \
  --region ap-northeast-1 \
  --profile <your-profile> \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
  --output table
```

以下の値をメモしておきます：

| 出力キー                | 用途                                       |
| ----------------------- | ------------------------------------------ |
| `SAMLAcsURL`            | Identity Center SAML アプリの ACS URL      |
| `SAMLAudienceURI`       | Identity Center SAML アプリの Audience URI |
| `UserPoolId`            | Cognito User Pool ID                       |
| `UserPoolClientId`      | Grafana の Client ID                       |
| `AuthorizationEndpoint` | Grafana の Auth URL                        |
| `TokenEndpoint`         | Grafana の Token URL                       |
| `UserInfoEndpoint`      | Grafana の API URL                         |

---

### Step 3: Identity Center で SAML アプリケーションを作成

1. **AWS コンソール** → **IAM Identity Center** → **アプリケーション**

2. **アプリケーションを追加** → **カスタム SAML 2.0 アプリケーションをセットアップする**

3. **表示名**: `Grafana` を入力

4. **IAM Identity Center メタデータ** セクションで **IAM Identity Center SAML メタデータファイル** をダウンロード（後で使用）

5. **アプリケーションメタデータ** を手動入力：
   - **アプリケーション ACS URL**: `<SAMLAcsURL の値>`
   - **アプリケーション SAML 対象者**: `<SAMLAudienceURI の値>`

6. **送信** をクリック

7. **アプリケーション開始 URL（オプション）** を編集して以下を設定（Access Portal からの起動用）：
   ```
   http://localhost:3000/login/generic_oauth
   ```

---

### Step 4: Identity Center で属性マッピングを設定

1. 作成したアプリケーション → **アクション** → **属性マッピングを編集**

2. 以下の属性を設定：

   | アプリケーションのユーザー属性                                       | マッピング       | 形式           |
   | -------------------------------------------------------------------- | ---------------- | -------------- |
   | `Subject`                                                            | `${user:email}`  | `emailAddress` |
   | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` | `${user:email}`  | `unspecified`  |
   | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name`         | `${user:name}`   | `unspecified`  |
   | `http://schemas.xmlsoap.org/claims/Group`                            | `${user:groups}` | `unspecified`  |

3. **変更を保存**

---

### Step 5: Identity Center でユーザー/グループを割り当て

1. アプリケーション → **ユーザーとグループを割り当てる**

2. Grafana にアクセスさせたいユーザーまたはグループを選択

3. **割り当て** をクリック

---

### Step 6: Cognito に SAML IdP を追加

1. **AWS コンソール** → **Amazon Cognito** → **ユーザープール** → `grafana-userpool-dev`

2. **認証** → **ソーシャルプロバイダーと外部プロバイダー** → **ID プロバイダーを追加**

3. **SAML** を選択

4. 設定：
   - **プロバイダー名**: `IdentityCenter`（半角英数字のみ）
   - **メタデータドキュメント**: Step 3 でダウンロードした XML ファイルをアップロード

5. **属性マッピング**:

   | SAML 属性                                                            | ユーザープール属性 |
   | -------------------------------------------------------------------- | ------------------ |
   | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` | `email`            |
   | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name`         | `name`             |
   | `http://schemas.xmlsoap.org/claims/Group`                            | `custom:groups`    |

6. **ID プロバイダーを追加**

---

### Step 7: Cognito アプリクライアントに IdP を追加

1. **アプリケーション** → **アプリケーションクライアント** → `grafana-client-dev`

2. **マネージドログインページの設定** セクションの **編集** をクリック

3. **ID プロバイダー** で `IdentityCenter` にチェック ✅

4. **変更を保存**

---

### Step 8: Client Secret を取得

```bash
aws cognito-idp describe-user-pool-client \
  --user-pool-id <UserPoolId> \
  --client-id <UserPoolClientId> \
  --region ap-northeast-1 \
  --profile <your-profile> \
  --query 'UserPoolClient.ClientSecret' \
  --output text
```

---

### Step 9: docker-compose.yml を更新

取得した値で `docker-compose.yml` を更新します。

必要な環境変数：

```yaml
environment:
  # OAuth 設定
  - GF_AUTH_GENERIC_OAUTH_CLIENT_ID=<UserPoolClientId>
  - GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=<ClientSecret>
  - GF_AUTH_GENERIC_OAUTH_AUTH_URL=<AuthorizationEndpoint>
  - GF_AUTH_GENERIC_OAUTH_TOKEN_URL=<TokenEndpoint>
  - GF_AUTH_GENERIC_OAUTH_API_URL=<UserInfoEndpoint>
  
  # ロールマッピング（グループ UUID を指定）
  - 'GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH="custom:groups" == ''<admin-group-uuid>'' && ''Admin'' || ''Viewer'''
```

---

### Step 10: Grafana を起動

```bash
docker-compose up -d
```

---

### Step 11: 動作確認

1. ブラウザで http://localhost:3000 にアクセス

2. **Sign in with AWS Identity Center** をクリック

3. Identity Center でログイン

4. Grafana にリダイレクトされ、ログイン完了

---

## ロールマッピング

Identity Center のグループ UUID に基づいて Grafana のロールを自動割り当てします。

### グループ UUID の確認方法

**Identity Center** → **グループ** → 対象グループを開き、**グループ ID** を確認。

### 設定例

```yaml
# tte-ems-grafana-admin グループ (UUID: 77349ad8-10c1-706a-910d-2e98d9ec06fb) → Admin
# それ以外 → Viewer
- 'GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH="custom:groups" == ''77349ad8-10c1-706a-910d-2e98d9ec06fb'' && ''Admin'' || ''Viewer'''
```

---

## Access Portal からの起動

Identity Center のアプリ設定で **アプリケーション開始 URL** を設定することで、Access Portal から直接 Grafana を開けます：

```
http://localhost:3000/login/generic_oauth
```

※ IdP-initiated SSO は Cognito でサポートされていないため、この URL は SP-initiated フローを開始します。

---

## ファイル構成

```
grafana-docker/
├── docker-compose.yml              # Grafana Docker 設定
├── cognito-identity-center.yaml    # CloudFormation テンプレート
└── README.md                       # このファイル
```

---

## トラブルシューティング

### Missing saved oauth state

**原因**: Cookie の問題

**解決策**: 以下の環境変数を設定
```yaml
- GF_SERVER_ROOT_URL=http://localhost:3000
- GF_SECURITY_COOKIE_SAMESITE=lax
- GF_SECURITY_COOKIE_SECURE=false
```

### User sync failed

**原因**: Grafana の既存データと不整合

**解決策**: ボリュームを削除して再起動
```bash
docker-compose down -v
docker-compose up -d
```

### Invalid samlResponse or relayState

**原因**: IdP-initiated SSO を試行（Cognito は非対応）

**解決策**: Grafana から開始（SP-initiated）でログイン

### Admin ロールが適用されない

**原因**: JMESPath 構文エラー、またはグループ UUID の不一致

**解決策**: 
1. グループ UUID が正しいか確認
2. docker-compose.yml の構文を確認（シングルクォートとダブルクォートの使い分け）

---

## クリーンアップ

```bash
# Grafana を停止
docker-compose down -v

# CloudFormation スタックを削除
aws cloudformation delete-stack \
  --stack-name grafana-cognito-dev \
  --region ap-northeast-1 \
  --profile <your-profile>
```

---

## 制限事項

| 機能              | 対応状況                      |
| ----------------- | ----------------------------- |
| SP-initiated SSO  | ✅                            |
| IdP-initiated SSO | ❌（Cognito の制限）          |
| ロールマッピング  | ✅                            |
| チーム自動同期    | ❌（Grafana Enterprise 限定） |