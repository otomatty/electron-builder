# electron-updaterでMacの自動更新が動作しない問題の調査レポート

## 問題の概要

electron-builderとelectron-updaterを使用した自動更新機能において、WindowsとLinuxでは正常に動作するが、Macでは自動更新が実行されない問題について調査しました。

## 主な原因と解決策

### 1. コード署名の問題 (最も重要)

**問題:**
- Macでは、自動更新を行うためにアプリケーションとupdateファイル（.zip）の両方が適切にコード署名されている必要があります
- electron-updaterのMacUpdaterクラスは内部的にSquirrel.Macを使用しており、署名されていないファイルは更新を拒否します

**確認方法:**
```bash
# アプリケーションの署名状態を確認
codesign -dv --verbose=4 /path/to/your/app.app

# 署名の検証
spctl -a -vvv /path/to/your/app.app
```

**解決策:**
```json
// package.json - electron-builder設定
{
  "build": {
    "mac": {
      "hardenedRuntime": true,
      "gatekeeperAssess": false,
      "entitlements": "build/entitlements.mac.plist",
      "entitlementsInherit": "build/entitlements.mac.plist",
      "identity": "Developer ID Application: Your Name (TEAM_ID)"
    },
    "dmg": {
      "sign": false  // DMGは署名しない（更新には関係なし）
    }
  }
}
```

### 2. Notarization（公証）の問題

**問題:**
- macOS 10.15 Catalina以降では、Apple開発者アカウントによる公証（Notarization）が必要
- 公証されていないアプリは Gatekeeper によってブロックされる可能性があります

**解決策:**
```json
// package.json - 公証設定
{
  "build": {
    "mac": {
      "notarize": {
        "teamId": "YOUR_TEAM_ID"
      }
    }
  }
}
```

**環境変数の設定:**
```bash
# Apple ID認証情報
APPLE_ID=your-apple-id@example.com
APPLE_APP_SPECIFIC_PASSWORD=your-app-specific-password
```

### 3. 証明書の種類の問題

**問題:**
- 自動更新には「Developer ID Application」証明書が必要
- Mac App Store用の証明書では自動更新は動作しません

**確認方法:**
```bash
# キーチェーンで利用可能な証明書を確認
security find-identity -v -p codesigning
```

**必要な証明書:**
- Developer ID Application: Your Name (TEAM_ID)

### 4. Entitlements（権限設定）の問題

**問題:**
- Hardened Runtimeを有効にした場合、適切なentitlementsが必要

**entitlements.mac.plist 例:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.network.server</key>
    <true/>
</dict>
</plist>
```

### 5. ファイル形式の問題

**問題:**
- MacUpdaterは.zipファイルのみを処理します
- .dmgファイルでは自動更新は動作しません

**確認点:**
```javascript
// publish設定で.zipファイルが生成されることを確認
{
  "build": {
    "mac": {
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

### 6. セキュリティ設定の問題

**問題:**
- Gatekeeperの設定により更新が阻止される場合があります

**一時的な回避策:**
```json
{
  "build": {
    "mac": {
      "gatekeeperAssess": false
    }
  }
}
```

## デバッグ方法

### 1. ログの確認
```javascript
// main.js
const { autoUpdater } = require('electron-updater');
const log = require('electron-log');

autoUpdater.logger = log;
autoUpdater.logger.transports.file.level = 'info';

autoUpdater.on('error', (error) => {
  log.error('Update error:', error);
});

autoUpdater.on('update-available', (info) => {
  log.info('Update available:', info);
});

autoUpdater.on('update-not-available', (info) => {
  log.info('Update not available:', info);
});
```

### 2. 手動でのファイル確認
```bash
# 更新ファイルの署名確認
codesign -dv /path/to/update.zip

# アプリケーションの権限確認
codesign -d --entitlements - /path/to/app.app
```

## 推奨される完全な設定例

```json
{
  "build": {
    "appId": "com.example.myapp",
    "mac": {
      "category": "public.app-category.utilities",
      "hardenedRuntime": true,
      "gatekeeperAssess": false,
      "entitlements": "build/entitlements.mac.plist",
      "entitlementsInherit": "build/entitlements.mac.plist",
      "notarize": {
        "teamId": "YOUR_TEAM_ID"
      },
      "target": [
        {
          "target": "zip",
          "arch": ["x64", "arm64"]
        }
      ]
    },
    "publish": {
      "provider": "github",
      "owner": "your-username",
      "repo": "your-repo"
    }
  }
}
```

## 最近の既知の問題

CHANGELOGの調査により、以下の問題が確認されました：

- **v6.2.1**: Mac用の差分ダウンロード機能が一時的に無効化された
- **v6.0.0**: macOS用のキャッシュディレクトリの問題が修正された

これらの問題がある場合は、最新版のelectron-updaterにアップデートすることを推奨します。

## まとめ

Macでの自動更新が動作しない最も一般的な原因は：

1. **適切なコード署名の不備** - Developer ID Application証明書での署名が必要
2. **公証（Notarization）の不備** - macOS 10.15以降で必須
3. **権限設定（Entitlements）の不備** - Hardened Runtime使用時に必要
4. **ファイル形式の問題** - .zipファイルが必要（.dmgは不可）

これらを順次確認・修正することで、Macでの自動更新機能を正常に動作させることができます。