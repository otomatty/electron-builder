# electron-updater プロバイダー設定ガイド

## 概要

このドキュメントでは、electron-updaterの各プロバイダーでのダウンロード先の設定方法と、実際のファイルダウンロード先について詳細に解説します。

## 1. GitHub Provider (Public リポジトリ)

### 設定方法

```typescript
// electron-builder.json5 での設定
{
  "publish": {
    "provider": "github",
    "owner": "your-username",
    "repo": "your-repo-name",
    "private": false
  }
}
```

```typescript
// アプリケーションコードでの設定
import { autoUpdater } from 'electron-updater';

autoUpdater.setFeedURL({
  provider: 'github',
  owner: 'your-username',
  repo: 'your-repo-name',
  private: false
});
```

### 実際のダウンロード先

```
https://github.com/{owner}/{repo}/releases/download/{tag}/{fileName}
```

**具体例**：
- Windows: `https://github.com/electron-userland/electron-builder/releases/download/v23.0.8/MyApp-Setup-1.0.0.exe`
- macOS: `https://github.com/electron-userland/electron-builder/releases/download/v23.0.8/MyApp-1.0.0.dmg`
- Linux: `https://github.com/electron-userland/electron-builder/releases/download/v23.0.8/MyApp-1.0.0.AppImage`

#### ⚠️ よくある疑問：`releases/download` パスについて

**Q: この `releases/download` というパスは本当に存在するの？**

**A: はい、これは GitHub の公式 URL パターンです。**

GitHub のリリース機能では以下の URL 構造になっています：
- **リリースページ**: `https://github.com/{owner}/{repo}/releases/tag/{tag}`
- **ダウンロードURL**: `https://github.com/{owner}/{repo}/releases/download/{tag}/{fileName}`

これは GitHub 側が提供している仕様であり、electron-updater が勝手に作ったパスではありません。

**検証方法**：
1. 任意のGitHubリポジトリのリリースページにアクセス
2. アップロードされたファイルを右クリックして「リンクアドレスをコピー」
3. URLを確認すると `releases/download` パスが含まれている

**実際の例**：
- electron-builder: https://github.com/electron-userland/electron-builder/releases/tag/v24.6.4
- ダウンロードリンク: https://github.com/electron-userland/electron-builder/releases/download/v24.6.4/...

### 更新情報の取得先

```
https://github.com/{owner}/{repo}/releases.atom
```

### 必要な手順

1. **GitHub リリースの作成**
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```

2. **リリースページでのファイル添付**
   - GitHub の Releases ページにアクセス
   - "Create a new release" をクリック
   - タグを選択し、リリースノートを記載
   - ビルドしたファイル（.exe、.dmg、.AppImage等）をアップロード

3. **latest.yml の設置**
   ```yaml
   version: 1.0.0
   files:
     - url: MyApp-Setup-1.0.0.exe
       sha512: [SHA512ハッシュ]
       size: 123456789
   path: MyApp-Setup-1.0.0.exe
   sha512: [SHA512ハッシュ]
   releaseDate: '2024-01-15T10:30:00.000Z'
   ```

### チャネル指定

```typescript
// プレリリースの場合
autoUpdater.allowPrerelease = true;
autoUpdater.channel = 'beta';
```

## 2. GitHub Provider (Private リポジトリ)

### 設定方法

```typescript
// electron-builder.json5 での設定
{
  "publish": {
    "provider": "github",
    "owner": "your-username",
    "repo": "your-private-repo",
    "private": true,
    "token": "ghp_your_personal_access_token"
  }
}
```

```typescript
// アプリケーションコードでの設定
import { autoUpdater } from 'electron-updater';

autoUpdater.setFeedURL({
  provider: 'github',
  owner: 'your-username',
  repo: 'your-private-repo',
  private: true,
  token: 'ghp_your_personal_access_token'
});
```

### 実際のダウンロード先

```
https://api.github.com/repos/{owner}/{repo}/releases/assets/{asset_id}
```

**認証ヘッダー**：
```
Authorization: token ghp_your_personal_access_token
```

#### ⚠️ Public リポジトリとの違い

**Private リポジトリでは GitHub API 経由でダウンロードします**：

| 項目 | Public リポジトリ | Private リポジトリ |
|------|------------------|-------------------|
| ダウンロード方式 | 直接ダウンロード | GitHub API 経由 |
| URL パターン | `releases/download/{tag}/{fileName}` | `releases/assets/{asset_id}` |
| 認証 | 不要 | Personal Access Token 必須 |
| リダイレクト | なし | API が実際のダウンロードURLを返す |

**理由**：Private リポジトリのファイルは直接アクセスできないため、GitHub API で認証してからダウンロードURLを取得する必要があります。

### 更新情報の取得先

```
https://api.github.com/repos/{owner}/{repo}/releases
```

### 必要な手順

1. **Personal Access Token の作成**
   - GitHub Settings → Developer settings → Personal access tokens
   - `repo` スコープを有効にする

2. **環境変数での設定**
   ```bash
   export GH_TOKEN=ghp_your_personal_access_token
   ```

3. **electron-builder での自動アップロード**
   ```bash
   npm run build
   npm run publish
   ```

## 3. Generic Provider (カスタムサーバー)

### 設定方法

```typescript
// electron-builder.json5 での設定
{
  "publish": {
    "provider": "generic",
    "url": "https://your-server.com/auto-updater",
    "channel": "latest"
  }
}
```

```typescript
// アプリケーションコードでの設定
import { autoUpdater } from 'electron-updater';

autoUpdater.setFeedURL({
  provider: 'generic',
  url: 'https://your-server.com/auto-updater',
  channel: 'latest'
});
```

### 実際のダウンロード先

```
https://your-server.com/auto-updater/{fileName}
```

### 更新情報の取得先

```
https://your-server.com/auto-updater/latest.yml
```

### サーバー構成例

```
your-server.com/auto-updater/
├── latest.yml              # 更新情報ファイル
├── latest-mac.yml          # macOS用更新情報
├── latest-linux.yml        # Linux用更新情報
├── MyApp-Setup-1.0.0.exe   # Windows インストーラー
├── MyApp-1.0.0.dmg         # macOS DMG
├── MyApp-1.0.0.AppImage    # Linux AppImage
└── MyApp-1.0.0.deb         # Linux DEB
```

### latest.yml の例

```yaml
version: 1.0.0
files:
  - url: MyApp-Setup-1.0.0.exe
    sha512: ba8a1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
    size: 123456789
path: MyApp-Setup-1.0.0.exe
sha512: ba8a1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
releaseDate: '2024-01-15T10:30:00.000Z'
```

### 自動デプロイ設定

```yaml
# .github/workflows/build.yml
name: Build and Deploy
on:
  push:
    tags: ['v*']
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: npm run build
      - name: Deploy to server
        run: |
          rsync -avz dist/ user@your-server.com:/var/www/auto-updater/
```

## 4. Bitbucket Provider

### 設定方法

```typescript
// electron-builder.json5 での設定
{
  "publish": {
    "provider": "bitbucket",
    "owner": "your-username",
    "slug": "your-repo-slug",
    "channel": "latest"
  }
}
```

```typescript
// アプリケーションコードでの設定
import { autoUpdater } from 'electron-updater';

autoUpdater.setFeedURL({
  provider: 'bitbucket',
  owner: 'your-username',
  slug: 'your-repo-slug',
  channel: 'latest'
});
```

### 実際のダウンロード先

```
https://api.bitbucket.org/2.0/repositories/{owner}/{slug}/downloads/{fileName}
```

### 更新情報の取得先

```
https://api.bitbucket.org/2.0/repositories/{owner}/{slug}/downloads/latest.yml
```

### 必要な手順

1. **Bitbucket Downloads への手動アップロード**
   - Bitbucket リポジトリページにアクセス
   - Downloads セクションに移動
   - ファイルを手動でアップロード

2. **API を使用したアップロード**
   ```bash
   curl -X POST \
     -u username:app_password \
     -F files=@MyApp-Setup-1.0.0.exe \
     https://api.bitbucket.org/2.0/repositories/owner/repo/downloads
   ```

## 5. Keygen Provider

### 設定方法

```typescript
// electron-builder.json5 での設定
{
  "publish": {
    "provider": "keygen",
    "account": "your-account-id",
    "product": "your-product-id",
    "channel": "stable"
  }
}
```

```typescript
// アプリケーションコードでの設定
import { autoUpdater } from 'electron-updater';

autoUpdater.setFeedURL({
  provider: 'keygen',
  account: 'your-account-id',
  product: 'your-product-id',
  channel: 'stable'
});
```

### 実際のダウンロード先

```
https://api.keygen.sh/v1/accounts/{account}/artifacts?product={product}
```

### 更新情報の取得先

```
https://api.keygen.sh/v1/accounts/{account}/artifacts/latest.yml?product={product}
```

### 必要な手順

1. **Keygen アカウントの作成**
   - https://keygen.sh でアカウントを作成
   - プロダクトを作成し、プロダクトIDを取得

2. **API を使用したアップロード**
   ```bash
   curl -X POST \
     -H "Authorization: Bearer your-api-token" \
     -F artifact=@MyApp-Setup-1.0.0.exe \
     https://api.keygen.sh/v1/accounts/your-account/artifacts
   ```

## 6. 設定のベストプラクティス

### セキュリティ設定

```typescript
// 署名検証の有効化
autoUpdater.forceDevUpdateConfig = false; // 本番環境では必須
autoUpdater.autoDownload = false; // 手動ダウンロード制御
autoUpdater.autoInstallOnAppQuit = false; // 手動インストール制御
```

### エラーハンドリング

```typescript
autoUpdater.on('error', (error) => {
  console.error('Auto updater error:', error);
  
  // プロバイダー別のエラーハンドリング
  if (error.message.includes('ERR_UPDATER_CHANNEL_FILE_NOT_FOUND')) {
    // チャンネルファイルが見つからない場合の処理
    console.log('Update channel file not found. Falling back to manual update.');
  } else if (error.message.includes('ERR_UPDATER_LATEST_VERSION_NOT_FOUND')) {
    // 最新バージョンが見つからない場合の処理
    console.log('No latest version found. Check your release configuration.');
  }
});
```

### 複数プロバイダーの対応

```typescript
// 環境によってプロバイダーを切り替え
const isProduction = process.env.NODE_ENV === 'production';
const isDevelopment = process.env.NODE_ENV === 'development';

if (isProduction) {
  autoUpdater.setFeedURL({
    provider: 'github',
    owner: 'your-username',
    repo: 'your-repo',
    private: false
  });
} else if (isDevelopment) {
  autoUpdater.setFeedURL({
    provider: 'generic',
    url: 'https://dev-server.com/auto-updater',
    channel: 'beta'
  });
}
```

## 7. よくある疑問とトラブルシューティング

### よくある疑問（FAQ）

#### Q1: プロバイダーのURLパターンは本当に正しいの？

**A1: はい、すべて各サービスの公式仕様に基づいています。**

| プロバイダー | 仕様の根拠 | 検証方法 |
|-------------|------------|----------|
| GitHub Public | GitHub公式のreleases機能 | リリースページでファイルを右クリック→リンクアドレス確認 |
| GitHub Private | GitHub REST API仕様 | `curl -H "Authorization: token YOUR_TOKEN" https://api.github.com/repos/owner/repo/releases` |
| Generic | カスタム実装 | サーバーの実際のファイルパスを確認 |
| Bitbucket | Bitbucket Downloads API | Bitbucket公式ドキュメント参照 |
| Keygen | Keygen API仕様 | Keygen公式ドキュメント参照 |

#### Q2: なぜプロバイダーによってURLパターンが違うの？

**A2: 各サービスのアーキテクチャとセキュリティモデルが異なるためです。**

- **GitHub Public**: CDN経由で直接配信（高速）
- **GitHub Private**: API認証後にダウンロード（セキュア）
- **Generic**: ユーザーのサーバー設計に依存
- **Bitbucket/Keygen**: 独自のAPI設計

#### Q3: ダウンロードURLを直接ブラウザで開いても動作するの？

**A3: プロバイダーによって異なります。**

- **GitHub Public**: ✅ 直接ダウンロード可能
- **GitHub Private**: ❌ 認証トークンが必要
- **Generic**: ✅ サーバー設定による
- **Bitbucket**: ❌ API認証が必要
- **Keygen**: ❌ API認証が必要

#### Q4: ファイル名にスペースが含まれている場合はどうなるの？

**A4: 自動的に `-` に置換されます。**

```typescript
// GitHubProvider の場合
return resolveFiles(updateInfo, this.baseUrl, p => 
  this.getBaseDownloadPath(updateInfo.tag, p.replace(/ /g, "-"))
)
```

例：`My App Setup 1.0.0.exe` → `My-App-Setup-1.0.0.exe`

**コードでの実装確認**：
```typescript
// packages/electron-updater/src/providers/GitHubProvider.ts:185-197
private get basePath(): string {
  return `/${this.options.owner}/${this.options.repo}/releases`
}

private getBaseDownloadPath(tag: string, fileName: string): string {
  return `${this.basePath}/download/${tag}/${fileName}`
}

resolveFiles(updateInfo: GithubUpdateInfo): Array<ResolvedUpdateFileInfo> {
  return resolveFiles(updateInfo, this.baseUrl, p => 
    this.getBaseDownloadPath(updateInfo.tag, p.replace(/ /g, "-"))
  )
}
```

**最終的なURL構築**：
`${this.baseUrl}${this.basePath}/download/${tag}/${fileName.replace(/ /g, "-")}`
= `https://github.com/{owner}/{repo}/releases/download/{tag}/{fileName}`

#### Q5: ダウンロードURLを実際に確認する方法は？

**A5: 各プロバイダー別の確認方法があります。**

##### GitHub Public リポジトリの場合

```bash
# 1. リリース情報を取得
curl -s https://api.github.com/repos/electron-userland/electron-builder/releases/latest

# 2. 実際のダウンロードURLを確認
# レスポンスの assets[].browser_download_url を確認
# 形式: https://github.com/{owner}/{repo}/releases/download/{tag}/{filename}
```

**ブラウザでの確認**：
1. GitHubリポジトリのReleasesページにアクセス
2. 任意のリリースを開く
3. アップロードされたファイルを右クリック
4. 「リンクアドレスをコピー」で実際のURLを確認

##### GitHub Private リポジトリの場合

```bash
# 1. 認証つきでリリース情報を取得
curl -H "Authorization: token YOUR_TOKEN" \
     https://api.github.com/repos/owner/private-repo/releases/latest

# 2. assets[].url を確認
# 形式: https://api.github.com/repos/{owner}/{repo}/releases/assets/{asset_id}

# 3. 実際のダウンロード（リダイレクトされる）
curl -L -H "Authorization: token YOUR_TOKEN" \
     -H "Accept: application/octet-stream" \
     https://api.github.com/repos/owner/repo/releases/assets/12345
```

##### Generic Provider の場合

```bash
# 1. latest.yml の確認
curl https://your-server.com/path/to/latest.yml

# 2. ファイルのダウンロードテスト
curl -I https://your-server.com/path/to/MyApp-Setup-1.0.0.exe
```

**electron-updater のログから確認**：
```typescript
// ログレベルをdebugに設定
autoUpdater.logger = require('electron-log');
autoUpdater.logger.transports.console.level = 'debug';

// ログでダウンロードURLが確認できる
autoUpdater.checkForUpdates();
```

### よくある問題と解決方法

#### 1. ファイルが見つからない (404エラー)
```typescript
// 解決方法: ファイル名とURLの確認
console.log('Expected file URL:', autoUpdater.getFeedURL());
console.log('Expected file name:', 'MyApp-Setup-1.0.0.exe');
```

#### 2. 署名検証エラー
```typescript
// Windows でのコード署名確認
autoUpdater.on('error', (error) => {
  if (error.message.includes('ERR_UPDATER_INVALID_SIGNATURE')) {
    console.log('Invalid signature detected. Check your code signing certificate.');
  }
});
```

#### 3. 権限エラー
```typescript
// 管理者権限での実行
autoUpdater.quitAndInstall(true, true); // (silent, forceRunAfter)
```

### ログとデバッグ

```typescript
import log from 'electron-log';

// ログレベルの設定
log.transports.file.level = 'debug';
autoUpdater.logger = log;

// カスタムログ
autoUpdater.on('checking-for-update', () => {
  log.info('Checking for update...');
});

autoUpdater.on('update-available', (info) => {
  log.info('Update available:', info);
});

autoUpdater.on('download-progress', (progressObj) => {
  log.info(`Download progress: ${progressObj.percent}%`);
});
```

## 8. まとめ

各プロバイダーの特徴：

| プロバイダー | 用途 | 設定難易度 | セキュリティ | 費用 |
|-------------|------|------------|-------------|------|
| GitHub Public | オープンソース | 低 | 中 | 無料 |
| GitHub Private | 商用プロジェクト | 中 | 高 | 有料 |
| Generic | カスタム要件 | 高 | 設定次第 | サーバー費用 |
| Bitbucket | Atlassian環境 | 中 | 中 | 有料 |
| Keygen | ライセンス管理 | 高 | 高 | 有料 |

**推奨順序**：
1. **GitHub Public** - 最も簡単で、オープンソースプロジェクトに最適
2. **GitHub Private** - 商用プロジェクトで、GitHub を既に使用している場合
3. **Generic** - 完全な制御が必要な場合
4. **Bitbucket** - Atlassian エコシステムを使用している場合
5. **Keygen** - 高度なライセンス管理が必要な場合

## 参考資料

- [electron-updater 公式ドキュメント](https://www.electron.build/auto-update)
- [GitHub Releases API](https://docs.github.com/en/rest/releases/releases)
- [Bitbucket Downloads API](https://developer.atlassian.com/bitbucket/api/2/reference/resource/repositories/%7Bworkspace%7D/%7Brepo_slug%7D/downloads)
- [Keygen API ドキュメント](https://keygen.sh/docs/api/) 