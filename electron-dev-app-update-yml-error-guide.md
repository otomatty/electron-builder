# Electrondev環境でのdev-app-update.ymlエラー解決ガイド

## 問題の概要

Electronアプリケーションをdev環境でビルドまたは実行したときに、macOSで「`dev-app-update.yml`がない」というエラーが発生する問題について解説します。

## エラーの詳細

### 典型的なエラーメッセージ
```
Error: Cannot find file dev-app-update.yml
Unable to find dev-app-update.yml in the root directory
Dev app update configuration not found
```

### エラーが発生する状況
- 開発環境でElectronアプリを実行している場合
- `electron-updater`を使用した自動更新機能をテストしている場合
- パッケージ化されていないアプリケーションで更新チェックを実行している場合

## dev-app-update.ymlとは

### 目的と役割
`dev-app-update.yml`は、**開発環境で自動更新機能をテストするため**に必要な設定ファイルです。

### 本番環境との違い
- **本番環境**: `app-update.yml`ファイルがElectronアプリのリソース内に自動生成される
- **開発環境**: パッケージ化されていないため、手動で`dev-app-update.yml`を作成する必要がある

### electron-builderドキュメントでの記載
公式ドキュメントによると：
> Note that in order to develop/test UI/UX of updating without packaging the application you need to have a file named `dev-app-update.yml` in the root of your project, which matches your `publish` setting from electron-builder config (but in yaml format).

## 解決方法

### 1. dev-app-update.ymlファイルの作成

プロジェクトのルートディレクトリに`dev-app-update.yml`ファイルを作成します。

#### GitHubリリース用の設定例
```yaml
provider: github
owner: your-username
repo: your-repository-name
private: false
```

#### Generic Server用の設定例
```yaml
provider: generic
url: https://your-update-server.com/updates/
channel: latest
```

#### S3用の設定例
```yaml
provider: s3
bucket: your-update-bucket
region: us-west-2
path: updates
```

### 2. forceDevUpdateConfigフラグの設定

最新バージョンのelectron-updaterでは、開発モードを明示的に有効にする必要があります。

```javascript
// main.js または main.ts
const { autoUpdater } = require('electron-updater');

// 開発環境でのみ設定
if (process.env.NODE_ENV === 'development') {
  autoUpdater.forceDevUpdateConfig = true;
}
```

### 3. package.jsonのpublish設定との整合性確認

`dev-app-update.yml`の内容は、`package.json`の`build.publish`設定と一致させる必要があります。

#### package.jsonの例
```json
{
  "build": {
    "publish": {
      "provider": "github",
      "owner": "your-username",
      "repo": "your-repository-name"
    }
  }
}
```

#### 対応するdev-app-update.yml
```yaml
provider: github
owner: your-username
repo: your-repository-name
```

## 完全な実装例

### ディレクトリ構造
```
your-electron-app/
├── dev-app-update.yml    # 開発環境用の設定ファイル
├── package.json
├── src/
│   └── main.js
└── ...
```

### main.jsの実装例
```javascript
const { app, BrowserWindow } = require('electron');
const { autoUpdater } = require('electron-updater');

class AppUpdater {
  constructor() {
    // 開発環境での設定
    if (process.env.NODE_ENV === 'development') {
      autoUpdater.forceDevUpdateConfig = true;
      console.log('Development mode: Using dev-app-update.yml');
    }
    
    // ログ設定
    autoUpdater.logger = require('electron-log');
    autoUpdater.logger.transports.file.level = 'info';
    
    // 更新チェック
    autoUpdater.checkForUpdatesAndNotify();
  }
}

app.whenReady().then(() => {
  createWindow();
  new AppUpdater();
});

function createWindow() {
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      nodeIntegration: true,
      contextIsolation: false
    }
  });

  mainWindow.loadFile('index.html');
}
```

### dev-app-update.ymlの完全な設定例
```yaml
# GitHub Releases用
provider: github
owner: your-username
repo: your-repository-name
private: false
token: ${env.GH_TOKEN}  # オプション：プライベートリポジトリの場合

# リリースチャンネル
channel: latest

# 更新サーバーURL（Genericプロバイダの場合）
# url: https://your-update-server.com/

# 追加オプション
updaterCacheDirName: your-app-updater
requestHeaders:
  # カスタムヘッダーがある場合
```

## 注意事項とベストプラクティス

### 1. 本番環境では不要
`dev-app-update.yml`は開発環境専用のファイルです。本番ビルドには含めないでください。

### 2. .gitignoreへの追加を検討
機密情報（トークンなど）が含まれる場合は、`.gitignore`に追加することを検討してください。

```gitignore
# 開発環境の設定ファイル（必要に応じて）
dev-app-update.yml
```

### 3. 環境変数の使用
認証トークンなどは環境変数を使用して管理してください。

```yaml
provider: github
owner: your-username
repo: your-repository-name
token: ${env.GH_TOKEN}
```

### 4. パッケージ化されたアプリでのテスト推奨
開発環境でのテストは限定的です。実際の更新機能は、パッケージ化されたアプリケーションでテストすることが推奨されます。

## トラブルシューティング

### 1. ファイルが見つからない場合
- `dev-app-update.yml`がプロジェクトのルートディレクトリに配置されているか確認
- ファイル名に誤字がないか確認（`dev-app-update.yml`）

### 2. 設定が反映されない場合
- `autoUpdater.forceDevUpdateConfig = true`が設定されているか確認
- package.jsonの`publish`設定とdev-app-update.ymlの内容が一致しているか確認

### 3. 認証エラーの場合
- GitHubトークンが正しく設定されているか確認
- プライベートリポジトリの場合は`private: true`を設定

### 4. macOS固有の問題
- macOSではコード署名が必要な場合があります
- 開発環境では署名なしでもテスト可能ですが、本番環境では必須です

## 関連ドキュメント

- [electron-builder公式ドキュメント](https://www.electron.build/auto-update)
- [electron-updater GitHub](https://github.com/electron-userland/electron-updater)
- [Electron公式更新ガイド](https://www.electronjs.org/docs/latest/tutorial/tutorial-publishing-updating)

## まとめ

`dev-app-update.yml`エラーは、開発環境で自動更新機能をテストする際に必要な設定ファイルが不足していることが原因です。適切な設定ファイルの作成と`forceDevUpdateConfig`フラグの設定により解決できます。ただし、実際の更新機能の検証は、パッケージ化されたアプリケーションで行うことが重要です。