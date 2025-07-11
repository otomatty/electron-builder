# electron-updaterにおけるprivateリポジトリの制限事項

## 概要

このドキュメントは、electron-updaterでprivateGitHubリポジトリを使用した自動更新を試みる際の根本的な制限事項とセキュリティ上の懸念について説明します。

## 問題の本質

- ユーザーがprivateリポジトリへのアクセス権を持っている場合、electron-updaterで自動更新が可能か？
    - **端的に言うと：安全には実現できません。**

## プロバイダー選択ロジック（privateとpublicの判定）

electron-updaterは設定に基づいて適切なプロバイダー(publicなのかprivateなのかなど)を自動選択します。この判定は **packages/electron-updater/src/providerFactory.ts:32-39** で行われます：

```typescript
// packages/electron-updater/src/providerFactory.ts:32-39
case "github": {
  const githubOptions = data as GithubOptions
  const token = (githubOptions.private ? process.env["GH_TOKEN"] || process.env["GITHUB_TOKEN"] : null) || githubOptions.token
  if (token == null) {
    return new GitHubProvider(githubOptions, updater, runtimeOptions)  // public
  } else {
    return new PrivateGitHubProvider(githubOptions, updater, token, runtimeOptions)  // private
  }
}
```

### 詳細な判定フロー

判定の核心は **packages/electron-updater/src/providerFactory.ts:34行目** の複雑な条件式です：

```typescript
const token = (githubOptions.private ? process.env["GH_TOKEN"] || process.env["GITHUB_TOKEN"] : null) || githubOptions.token
```

この式は以下の優先順位で評価されます：

#### ステップ1: privateフラグの確認
```typescript
// packages/builder-util-runtime/src/publishOptions.ts:122-124
/**
 * Whether to use private github auto-update provider if `GH_TOKEN` environment variable is defined.
 */
readonly private?: boolean | null
```

#### ステップ2: トークン取得の優先順位

**`githubOptions.private` が `true` の場合：**
1. 環境変数 `GH_TOKEN` をチェック
2. 見つからない場合、環境変数 `GITHUB_TOKEN` をチェック  
3. どちらも見つからない場合、`null` を返す

**`githubOptions.private` が `false` または未設定の場合：**
1. `null` を返す

#### ステップ3: フォールバックチェック
上記で `null` が返された場合、`githubOptions.token` の値を使用

#### ステップ4: 最終判定
- `token` が `null` → **GitHubProvider (public)**
- `token` が存在 → **PrivateGitHubProvider (private)**

### 判定パターンの実例

| `private` | `GH_TOKEN` | `GITHUB_TOKEN` | `token` | 結果 |
|-----------|------------|----------------|---------|------|
| `true` | `"ghp_xxx"` | - | - | **Private** |
| `true` | - | `"ghp_yyy"` | - | **Private** |
| `true` | - | - | `"ghp_zzz"` | **Private** |
| `true` | - | - | - | **Public** |
| `false` | `"ghp_xxx"` | - | - | **Public** |
| `false` | - | - | `"ghp_zzz"` | **Private** |
| 未設定 | `"ghp_xxx"` | - | - | **Public** |
| 未設定 | - | - | `"ghp_zzz"` | **Private** |

**重要**: `private: true` が設定されていても、環境変数にトークンが存在しない場合は **publicプロバイダー** が選択されます。

## privateリポジトリの設定方法

次にどのようにprivateリポジトリとして設定するのかについて解説します。

`githubOptions.private`は、electron-builderの設定ファイルで設定します。以下の場所で設定可能です：

### 設定ファイルの探索順序と配置場所

electron-builderは以下の順序で設定ファイルを探索・読み込みます：

#### 1. 設定ファイルの探索場所

**プロジェクトディレクトリ**は以下の方法で決定されます：
- `--projectDir` オプションで指定
- 未指定の場合、現在の作業ディレクトリ（`process.cwd()`）を使用

**設定ファイルの探索は以下の順序で行われます：**

1. **コマンドライン引数**: `--config <path>` で指定されたファイル
2. **プロジェクトディレクトリ内の設定ファイル**（以下の順序）:
   - `electron-builder.yml`
   - `electron-builder.yaml`
   - `electron-builder.json`
   - `electron-builder.json5`
   - `electron-builder.js`
   - `electron-builder.cjs`
   - `electron-builder.mjs`
   - `electron-builder.ts`
   - `electron-builder.toml`
3. **package.json内の`build`フィールド**

#### 2. 設定ファイルの対応形式

```typescript
// packages/app-builder-lib/src/util/config/load.ts で実装
```

- **JSON/JSON5**: `.json`, `.json5`
- **YAML**: `.yml`, `.yaml`
- **JavaScript**: `.js`, `.cjs`, `.mjs` (CommonJS/ESM対応)
- **TypeScript**: `.ts` (config-file-tsを使用)
- **TOML**: `.toml` (tomlパッケージが必要)

#### 3. プロジェクト構造による設定ファイルの配置

**標準的な構造**:
```
my-electron-app/
├── package.json           # "build"フィールドまたは
├── electron-builder.yml   # 専用設定ファイル
├── src/
│   └── main.js
└── dist/
```

**2つのpackage.json構造**:
```
my-electron-app/
├── package.json           # 開発用依存関係
├── electron-builder.yml   # ビルド設定
├── app/
│   ├── package.json       # アプリケーション依存関係
│   └── main.js
└── dist/
```

**重要**: 設定ファイルは常に**プロジェクトルート**に配置し、アプリケーションディレクトリ内には配置しないでください。

### 設定ファイルでの指定方法

#### package.jsonでの設定

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "main": "src/main.js",
  "build": {
    "appId": "com.example.myapp",
    "publish": {
      "provider": "github",
      "owner": "your-username",
      "repo": "your-repo",
      "private": true
    }
  },
  "devDependencies": {
    "electron": "^latest",
    "electron-builder": "^latest"
  }
}
```

#### electron-builder.ymlでの設定

```yaml
appId: com.example.myapp
publish:
  provider: github
  owner: your-username
  repo: your-repo
  private: true
```

#### electron-builder.jsでの設定（ESM対応）

```javascript
// electron-builder.js
export default {
  appId: 'com.example.myapp',
  publish: {
    provider: 'github',
    owner: 'your-username',
    repo: 'your-repo',
    private: true
  }
}
```

#### electron-builder.tsでの設定

```typescript
// electron-builder.ts
import { Configuration } from 'electron-builder';

const config: Configuration = {
  appId: 'com.example.myapp',
  publish: {
    provider: 'github',
    owner: 'your-username',
    repo: 'your-repo',
    private: true
  }
};

export default config;
```

#### 複数のpublishプロバイダーを設定する場合

**JSON設定例**:
```json
{
  "build": {
    "publish": [
      {
        "provider": "github",
        "owner": "your-username", 
        "repo": "your-repo",
        "private": true
      },
      {
        "provider": "s3",
        "bucket": "my-bucket",
        "region": "us-east-1"
      }
    ]
  }
}
```

**YAML設定例**:
```yaml
publish:
  - provider: github
    owner: your-username
    repo: your-repo
    private: true
  - provider: s3
    bucket: my-bucket
    region: us-east-1
```

### 複数プロバイダーの優先順位と動作

#### 1. 自動更新での優先順位

**重要**: `electron-updater`の自動更新機能は、**配列の最初のプロバイダーのみ**を使用します。

```typescript
// packages/builder-util-runtime/src/publishOptions.ts:28-35
/**
 * Whether to publish auto update info files.
 *
 * Auto update relies only on the first provider in the list (you can specify several publishers).
 * Thus, probably, there`s no need to upload the metadata files for the other configured providers. But by default will be uploaded.
 *
 * @default true
 */
readonly publishAutoUpdate?: boolean
```

#### 2. 公式仕様

electron-builderの公式ドキュメントでは以下のように明記されています：

> "Auto update relies only on the first provider in the list (you can specify several publishers)."

#### 3. 実例での動作説明

以下の設定例での動作：

```json
{
  "build": {
    "publish": [
      {
        "provider": "github",
        "owner": "your-username",
        "repo": "your-repo",
        "private": true
      },
      {
        "provider": "s3",
        "bucket": "backup-bucket"
      }
    ]
  }
}
```

**動作結果**:
- **アーティファクト配布**: 両方のプロバイダーにアップロード
- **自動更新情報**: **GitHubのみ**が使用される
- **`latest.yml`の生成**: 両方のプロバイダーに対して生成（デフォルト）

#### 4. `publishAutoUpdate`による制御

特定のプロバイダーで自動更新情報の生成を無効化：

```json
{
  "build": {
    "publish": [
      {
        "provider": "github",
        "owner": "your-username",
        "repo": "your-repo",
        "private": true
      },
      {
        "provider": "s3",
        "bucket": "backup-bucket",
        "publishAutoUpdate": false
      }
    ]
  }
}
```

この設定では：
- S3には`latest.yml`が生成されない
- GitHubには`latest.yml`が生成される
- 自動更新はGitHubのみを使用

#### 5. 実行時の優先順位決定

electron-builderは以下の順序で実際に使用するプロバイダーを決定します：

```typescript
// packages/app-builder-lib/src/publish/PublishManager.ts:431-468
async function resolvePublishConfigurations() {
  // 1. 明示的な設定がある場合はそれを使用
  if (publishers != null) {
    return publishers
  }
  
  // 2. 環境変数による自動検出
  if (process.env.GH_TOKEN || process.env.GITHUB_TOKEN) {
    return [{ provider: "github" }]
  }
  if (process.env.KEYGEN_TOKEN) {
    return [{ provider: "keygen" }]
  }
  if (process.env.BITBUCKET_TOKEN) {
    return [{ provider: "bitbucket" }]
  }
  
  // 3. デフォルト（空配列）
  return []
}
```

### 設定の優先順位まとめ

1. **設定ファイルの優先順位**: CLI引数 > 専用設定ファイル > package.json
2. **プロバイダーの優先順位**: 配列の最初が自動更新に使用
3. **自動検出の優先順位**: 明示設定 > 環境変数 > デフォルト

### privateフィールドの定義

このフィールドは以下で定義されています：

```typescript
// packages/builder-util-runtime/src/publishOptions.ts:116-118
/**
 * Whether to use private github auto-update provider if `GH_TOKEN` environment variable is defined. See [Private GitHub Update Repo](./auto-update.md#private-github-update-repo).
 */
readonly private?: boolean | null
```

## GitHubのAtomフィードとは

プロバイダーの実装詳細を理解するために、まずGitHubのAtomフィードについて簡単に説明します。

**Atomフィード**は、Webサイトの更新情報を配信するためのXMLベースの標準フォーマットです。GitHubは各リポジトリのリリース情報をAtomフィード形式で提供しています。

### GitHubリリースのAtomフィード

アクセス方法：
```
https://github.com/{owner}/{repo}/releases.atom
```

例：
```
https://github.com/electron/electron/releases.atom
```

### 実際のAtomフィード内容（簡略版）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Release notes from electron</title>
  <link href="https://github.com/electron/electron/releases"/>
  
  <entry>
    <title>v28.1.0</title>
    <link href="https://github.com/electron/electron/releases/tag/v28.1.0"/>
    <updated>2024-01-15T10:30:00Z</updated>
  </entry>
  
  <entry>
    <title>v28.0.0</title>
    <link href="https://github.com/electron/electron/releases/tag/v28.0.0"/>
    <updated>2024-01-10T15:20:00Z</updated>
  </entry>
</feed>
```

### publicリポジトリでの利点

- **認証不要**: 誰でもアクセス可能
- **軽量**: 必要最小限の情報のみ
- **GitHub API制限を回避**: レート制限の対象外

### privateリポジトリでの制限

- **認証が必要**: ログインしていないとアクセス不可
- **GitHubのAtomフィードは認証をサポートしていない**
- **結果として、privateリポジトリではAtomフィードが使用不可**

## 各プロバイダーの実装詳細

Githubに関しては以下の2つのProviderを使用して自動更新を実行するようになっています。
1. GitHubProvider（public用）
2. PrivateGitHubProvider（private用）

### publicリポジトリでの動作 (GitHubProvider)

publicリポジトリの場合、electron-updaterは以下の方法を使用します：

1. **アクセス方法**: GitHubのpublicAtomフィード (`https://github.com/owner/repo/releases.atom`)
2. **認証**: 不要
3. **セキュリティ**: 安全 - 認証情報の露出なし
4. **制限**: publicリポジトリでのみ動作

```typescript
// packages/electron-updater/src/providers/GitHubProvider.ts:60-66
const feedXml: string = (await this.httpRequest(
  newUrlFromBase(`${this.basePath}.atom`, this.baseUrl),
  {
    accept: "application/xml, application/atom+xml, text/xml, */*",
  },
  cancellationToken
))!
```

### privateリポジトリでの動作 (PrivateGitHubProvider)

privateリポジトリの場合、electron-updaterは以下が必要です：

1. **アクセス方法**: GitHub APIへの直接アクセス (`api.github.com/repos/owner/repo/releases`)
2. **認証**: 以下のいずれかによるアクセストークンが必要
   - 環境変数: `GH_TOKEN` または `GITHUB_TOKEN`
   - 設定: `githubOptions.token`
3. **セキュリティ**: **重大なセキュリティリスク** - トークンをアプリケーションに埋め込む必要
4. **制限**: クライアント配布において根本的に安全でない

```typescript
// packages/electron-updater/src/providers/PrivateGitHubProvider.ts:58-62
private configureHeaders(accept: string) {
  return {
    accept,
    authorization: `token ${this.token}`,
  }
}
```

## privateリポジトリが安全に動作しない理由

### 1. ユーザーのアクセス権は無関係

**重要な誤解**: 多くの開発者は、エンドユーザーがprivateリポジトリへのアクセス権を持っていれば、electron-updaterが自動的にアクセスできるはずだと考えています。

**現実**: electron-updaterはクライアントアプリケーション内で動作し、ユーザーのGitHubセッションや認証情報について一切の知識を持ちません。ユーザーのGitHubアクセス権を活用することはできません。

### 2. トークンセキュリティ問題

privateリポジトリにアクセスするため、electron-updaterはGitHubアクセストークンを必要とします。これにより解決不可能なセキュリティ問題が発生します：

- **クライアントサイドトークンは公開情報**: クライアントアプリケーションに埋め込まれたトークンは実質的に公開情報
- **バイナリ解析**: 攻撃者は配布されたバイナリからトークンを抽出可能
- **安全な保存の不可能性**: クライアントアプリケーションは秘密情報を安全に保存できない
- **スコープの拡大**: トークンは多くの場合、必要以上の権限を持つ

### 3. 公式警告

electron-updaterの公式ドキュメントではこの問題について明示的に警告しています：

```typescript
// packages/builder-util-runtime/src/publishOptions.ts:116-118
/**
 * The access token to support auto-update from private github repositories. 
 * Never specify it in the configuration files. Only for [setFeedURL](./auto-update.md#appupdatersetfeedurloptions).
 */
readonly token?: string | null
```

## セキュリティへの影響

### 攻撃ベクトル

1. **トークン抽出**: 攻撃者はクライアントアプリケーションをリバースエンジニアリングして埋め込まれたトークンを抽出
2. **リポジトリアクセス**: 抽出したトークンを使用してprivateリポジトリにアクセス
3. **権限昇格**: トークンがリポジトリアクセス以上の権限を持つ場合がある
4. **横展開**: 侵害されたトークンが他のリソースへのアクセスに使用される可能性

### 攻撃シナリオ例

```bash
# 攻撃者がクライアントバイナリからトークンを抽出
strings your-app.exe | grep -i "ghp_"

# 抽出したトークンを使用してprivateリポジトリにアクセス
curl -H "Authorization: token ghp_extracted_token" \
  https://api.github.com/repos/your-org/private-repo
```

## 推奨解決策

### 1. publicリポジトリの使用（推奨）

**メリット:**
- セキュリティリスクなし
- 設定が簡単
- 追加インフラ不要
- そのまま動作

**デメリット:**
- ソースコードが公開される
- リリースバイナリが公開される

**実装例:**
```javascript
// package.json
{
  "build": {
    "publish": {
      "provider": "github",
      "owner": "your-username",
      "repo": "your-public-repo"
    }
  }
}
```

### 2. カスタムアップデートサーバーの構築

**メリット:**
- プライバシーを維持
- 認証を完全制御
- カスタムビジネスロジックの実装可能
- サーバーサイドでの安全なトークン処理

**デメリット:**
- 追加インフラが必要
- 実装が複雑
- メンテナンス負荷

**実装例:**
```javascript
// package.json
{
  "build": {
    "publish": {
      "provider": "generic",
      "url": "https://your-update-server.com/",
      "channel": "latest"
    }
  }
}
```

サーバーサイド認証の例:
```javascript
// あなたのアップデートサーバー
app.get('/latest.yml', authenticateUser, (req, res) => {
  // サーバーが認証を安全に処理
  // 認証されたユーザーのみに更新情報を返す
  res.sendFile(path.join(__dirname, 'latest.yml'));
});
```

### 3. カスタムプロバイダーの実装

**メリット:**
- 最大限の柔軟性
- カスタム認証フローの実装可能
- 既存システムとの統合

**デメリット:**
- 大幅な開発工数が必要
- セキュリティを正しく実装する必要
- 複雑なメンテナンス

**実装例:**
```typescript
// カスタムプロバイダーの例
class CustomUpdateProvider extends Provider<UpdateInfo> {
  async getLatestVersion(): Promise<UpdateInfo> {
    // カスタムロジックの実装
    // サーバーサイド認証の使用可能
    const response = await this.httpRequest(
      this.customEndpoint,
      { /* カスタムヘッダー */ }
    );
    return parseUpdateInfo(response);
  }
}
```

## ベストプラクティス

### publicリポジトリ使用時
1. リポジトリに機密データを含めない
2. ビルド時の秘密情報には環境変数を使用
3. リリース専用のpublicリポジトリの使用を検討

### privateソリューション使用時
1. クライアントアプリケーションに秘密情報を絶対に埋め込まない
2. 適切なサーバーサイド認証を実装
3. すべての通信でHTTPSを使用
4. アクセストークンの定期的なローテーション
5. 不正アクセスの監視
6. レート制限の実装

## まとめ

electron-updaterはpublicリポジトリからの自動更新について優れたサポートを提供しますが、privateリポジトリを安全に使用するには大幅な追加インフラとセキュリティ考慮が必要です。最も簡単で安全なアプローチは、メインの開発リポジトリはprivateのままでも、リリース用にはpublicリポジトリを使用することです。

privateな更新配布が絶対に必要な組織では、追加の複雑さはありますが、適切な認証を備えたカスタムアップデートサーバーの実装が推奨されるアプローチです。

## 参考資料

- [electron-updater ドキュメント](https://www.electron.build/auto-update)
- [GitHub API 認証](https://docs.github.com/en/rest/authentication)
- [Electron セキュリティガイドライン](https://www.electronjs.org/docs/latest/tutorial/security) 