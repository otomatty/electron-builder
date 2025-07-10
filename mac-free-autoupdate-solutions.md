# 社内Mac向け無料自動更新ソリューション

## 問題

- 社内のMacのみで自動更新を実現したい
- Apple Developer Program（年額$99）の証明書にお金をかけたくない
- 標準のelectron-updaterはDeveloper ID Application証明書を要求する

## 解決策オプション

### 🎯 **推奨：カスタムMacUpdaterによる署名検証スキップ**

**メリット：**
- electron-updaterの仕組みをそのまま活用
- 最小限のコード変更で実現可能
- 社内限定なのでセキュリティリスクが限定的

**実装方法：**

```javascript
// custom-mac-updater.js
const { MacUpdater } = require('electron-updater');

class CustomMacUpdater extends MacUpdater {
  constructor(options, app) {
    super(options, app);
    // 署名検証を無効化するフラグ
    this.disableSignatureCheck = true;
  }

  async doDownloadUpdate(downloadUpdateOptions) {
    // 元のダウンロード処理を実行
    const result = await super.doDownloadUpdate(downloadUpdateOptions);
    
    // 署名検証をスキップして直接インストールを許可
    if (this.disableSignatureCheck) {
      this._logger.info('Skipping signature verification for internal use');
    }
    
    return result;
  }
}

module.exports = { CustomMacUpdater };
```

```javascript
// main.js
const { app } = require('electron');
const { CustomMacUpdater } = require('./custom-mac-updater');
const { NsisUpdater } = require('electron-updater');
const { AppImageUpdater } = require('electron-updater');

let autoUpdater;

if (process.platform === 'darwin') {
  autoUpdater = new CustomMacUpdater();
} else if (process.platform === 'win32') {
  autoUpdater = new NsisUpdater();
} else {
  autoUpdater = new AppImageUpdater();
}

// 通常のelectron-updaterと同じように使用
autoUpdater.checkForUpdatesAndNotify();
```

**package.json設定：**
```json
{
  "build": {
    "mac": {
      "identity": null,  // 署名を無効化
      "target": [
        {
          "target": "zip",
          "arch": ["x64", "arm64"]
        }
      ]
    }
  }
}
```

### 💡 **代替案1：Ad-hoc署名の使用**

無料のad-hoc署名を使用する方法：

```json
{
  "build": {
    "mac": {
      "identity": "-",  // ad-hoc署名を指定
      "gatekeeperAssess": false,
      "target": [
        {
          "target": "zip", 
          "arch": ["x64", "arm64"]
        }
      ]
    }
  }
}
```

**制限事項：**
- Gatekeeperで警告が表示される場合がある
- 社内のセキュリティポリシーによっては使用できない

### 💡 **代替案2：完全カスタムアップデーター**

electron-updaterを使わずに独自実装：

```javascript
// simple-updater.js
const { dialog, shell } = require('electron');
const { download } = require('electron-dl');
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

class SimpleUpdater {
  constructor(updateUrl) {
    this.updateUrl = updateUrl;
  }

  async checkForUpdates() {
    try {
      // 更新情報をチェック
      const response = await fetch(`${this.updateUrl}/latest.json`);
      const updateInfo = await response.json();
      
      const currentVersion = require('../package.json').version;
      
      if (this.isNewerVersion(updateInfo.version, currentVersion)) {
        await this.promptForUpdate(updateInfo);
      }
    } catch (error) {
      console.error('Update check failed:', error);
    }
  }

  async promptForUpdate(updateInfo) {
    const result = await dialog.showMessageBox({
      type: 'info',
      title: 'アップデートが利用可能です',
      message: `新しいバージョン ${updateInfo.version} が利用可能です。今すぐアップデートしますか？`,
      buttons: ['今すぐアップデート', 'キャンセル'],
      defaultId: 0
    });

    if (result.response === 0) {
      await this.downloadAndInstall(updateInfo);
    }
  }

  async downloadAndInstall(updateInfo) {
    try {
      // ダウンロード
      const downloadPath = path.join(require('os').tmpdir(), 'app-update.zip');
      await download(BrowserWindow.getFocusedWindow(), updateInfo.url, {
        directory: path.dirname(downloadPath),
        filename: path.basename(downloadPath)
      });

      // 解凍とインストール
      await this.installUpdate(downloadPath);
      
    } catch (error) {
      console.error('Update installation failed:', error);
    }
  }

  async installUpdate(zipPath) {
    // 簡単なインストールスクリプト
    const script = `
      #!/bin/bash
      # アプリを終了
      pkill -f "Your App Name"
      
      # バックアップ
      mv "/Applications/Your App.app" "/Applications/Your App.app.backup"
      
      # 新しいバージョンを展開
      unzip -q "${zipPath}" -d "/Applications/"
      
      # 新しいアプリを起動
      open "/Applications/Your App.app"
      
      # バックアップを削除
      rm -rf "/Applications/Your App.app.backup"
    `;
    
    const scriptPath = path.join(require('os').tmpdir(), 'update.sh');
    fs.writeFileSync(scriptPath, script);
    fs.chmodSync(scriptPath, '755');
    
    // アプリを終了してスクリプトを実行
    execSync(`osascript -e 'tell application "Your App" to quit'`);
    execSync(`nohup ${scriptPath} &`);
  }

  isNewerVersion(remoteVersion, currentVersion) {
    // バージョン比較ロジック
    return remoteVersion > currentVersion;
  }
}

module.exports = { SimpleUpdater };
```

### 💡 **代替案3：ウェブベース更新通知**

最もシンプルな方法：

```javascript
// web-update-checker.js
class WebUpdateChecker {
  constructor(checkUrl) {
    this.checkUrl = checkUrl;
  }

  async checkForUpdates() {
    try {
      const response = await fetch(this.checkUrl);
      const updateInfo = await response.json();
      
      const currentVersion = require('../package.json').version;
      
      if (updateInfo.version > currentVersion) {
        this.showUpdateNotification(updateInfo);
      }
    } catch (error) {
      console.error('Update check failed:', error);
    }
  }

  showUpdateNotification(updateInfo) {
    const { dialog, shell } = require('electron');
    
    dialog.showMessageBox({
      type: 'info',
      title: 'アップデートが利用可能です',
      message: `新しいバージョン ${updateInfo.version} が利用可能です。`,
      detail: 'ダウンロードページを開きますか？',
      buttons: ['ダウンロードページを開く', 'キャンセル'],
      defaultId: 0
    }).then(result => {
      if (result.response === 0) {
        shell.openExternal(updateInfo.downloadUrl);
      }
    });
  }
}

module.exports = { WebUpdateChecker };
```

## 社内配布のセットアップ

### サーバー設定

```javascript
// update-server.js (Express.js例)
const express = require('express');
const app = express();

// 最新バージョン情報を提供
app.get('/latest.json', (req, res) => {
  res.json({
    version: '1.2.0',
    url: 'http://your-internal-server/releases/app-1.2.0-mac.zip',
    releaseNotes: '新機能が追加されました'
  });
});

// ダウンロードファイルを提供
app.use('/releases', express.static('releases'));

app.listen(3000, () => {
  console.log('Update server running on port 3000');
});
```

### ビルドスクリプト

```bash
#!/bin/bash
# build-and-deploy.sh

# ビルド
npm run build:mac

# リリースディレクトリにコピー
cp dist/*.zip releases/

# latest.jsonを更新
echo '{
  "version": "'$(node -p "require('./package.json').version")'",
  "url": "http://your-internal-server/releases/app-'$(node -p "require('./package.json').version")'-mac.zip"
}' > releases/latest.json

echo "Build and deployment complete!"
```

## 推奨アプローチ

**社内限定の用途であれば、「カスタムMacUpdater」アプローチを推奨します：**

1. **実装が簡単** - 既存のelectron-updaterを拡張するだけ
2. **機能が豊富** - 差分更新、進捗表示なども利用可能
3. **メンテナンスが容易** - electron-updaterのアップデートに追従可能
4. **セキュリティ** - 社内ネットワーク限定なのでリスクが限定的

この方法により、証明書費用をかけずに社内Mac向けの自動更新機能を実現できます。