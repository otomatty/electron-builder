# electron-updaterでMacの自動更新が動作しない問題の調査レポート

## 問題の概要

electron-builderとelectron-updaterを使用した自動更新機能において、WindowsとLinuxでは正常に動作するが、Macでは自動更新が実行されない問題について調査しました。

## 主な原因と解決策

### 1. ファイル形式の問題（最も基本的な要件）

**問題の根拠:**
MacUpdaterは.zipファイルのみを処理し、.dmgファイルでは自動更新は動作しません。

**ソースコード根拠:**
```javascript
// packages/electron-updater/src/MacUpdater.ts:85-94
const zipFileInfo = findFile(files, "zip", ["pkg", "dmg"])

if (zipFileInfo == null) {
  throw newError(`ZIP file not provided: ${safeStringifyJson(files)}`, "ERR_UPDATER_ZIP_FILE_NOT_FOUND")
}
```

`findFile`関数は第3引数で "pkg" と "dmg" を除外対象として指定しており、zipファイルのみを検索対象としています。

**解決策:**
```json
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

### 2. Squirrel.Macの要件とコード署名

**問題の根拠:**
MacUpdaterは内部的にSquirrel.Macを使用しており、署名されていないファイルは更新を拒否します。

**ソースコード根拠:**
```javascript
// packages/electron-updater/src/MacUpdater.ts:16-17
export class MacUpdater extends AppUpdater {
  private readonly nativeUpdater: AutoUpdater = require("electron").autoUpdater
```

```javascript
// packages/electron-updater/src/MacUpdater.ts:144-145
this.debug(`Creating proxy server for native Squirrel.Mac (${logContext})`)
```

Electronの`autoUpdater`はSquirrel.Macのラッパーであり、プロキシサーバーのコメントでも「native Squirrel.Mac」と明記されています。

**コード署名の要件:**
```javascript
// packages/app-builder-lib/src/macPackager.ts:267-278
const certificateTypes = getCertificateTypes(isMas, isDevelopment)

let identity = null
for (const certificateType of certificateTypes) {
  identity = await findIdentity(certificateType, qualifier, keychainFile)
  if (identity != null) {
    break
  }
}
```

**解決策:**
```json
{
  "build": {
    "mac": {
      "hardenedRuntime": true,
      "gatekeeperAssess": false,
      "entitlements": "build/entitlements.mac.plist",
      "entitlementsInherit": "build/entitlements.mac.plist",
      "identity": "Developer ID Application: Your Name (TEAM_ID)"
    }
  }
}
```

### 3. Entitlements（権限設定）の要件

**ソースコード根拠:**
```xml
<!-- packages/app-builder-lib/templates/entitlements.mac.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <!-- https://github.com/electron/electron-notarize#prerequisites -->
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <!-- https://github.com/electron-userland/electron-builder/issues/3940 -->
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
  </dict>
</plist>
```

このテンプレートは`com.apple.security.cs.allow-jit`などの必要な権限を含んでいます。

### 4. Notarization（公証）の要件

**ソースコード根拠:**
```javascript
// packages/app-builder-lib/src/macPackager.ts:382-384
if (!isMas) {
  await this.notarizeIfProvided(appPath)
}
```

MAS（Mac App Store）ビルドでない場合は、notarization処理が実行されます。

**解決策:**
```json
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

### 5. 署名の検証方法

**確認コマンド:**
```bash
# packages/app-builder-lib/src/codeSign/macCodeSign.ts:220-265で実装されている処理
/usr/bin/security find-identity -v -p codesigning
```

## 実際の問題と修正履歴

### 既知の問題（GitHubイシュー/プルリクエスト）

**1. サーバー接続エラー（修正済み）**
- **問題:** [PR #6743](https://github.com/electron-userland/electron-builder/pull/6743)で修正
- **原因:** `quitAndInstall`でサーバーが早期にクローズされる
- **修正内容:** サーバークローズのタイミングを適切に調整

**2. 差分更新の問題**
- **実装:** [PR #7709](https://github.com/electron-userland/electron-builder/pull/7709)でmacOS差分更新を実装
- **取り消し:** [PR #8091](https://github.com/electron-userland/electron-builder/pull/8091)で問題のため取り消し
- **再実装:** [PR #8095](https://github.com/electron-userland/electron-builder/pull/8095)で再実装

**3. キャッシュの問題**
- **修正:** [PR #8541](https://github.com/electron-userland/electron-builder/pull/8541)で`ENOENT`エラーを修正
- **改善:** [PR #8623](https://github.com/electron-userland/electron-builder/pull/8623)で非同期ファイルコピーに変更

### 証明書の種類の要件

**必要な証明書:**
```javascript
// packages/app-builder-lib/src/codeSign/macCodeSign.ts:15
export const appleCertificatePrefixes = [
  "Developer ID Application:", 
  "Developer ID Installer:", 
  "3rd Party Mac Developer Application:", 
  "3rd Party Mac Developer Installer:"
]
```

自動更新には「Developer ID Application」証明書が必要です。

## デバッグ方法

### 1. ログの確認
```javascript
// packages/electron-updater/src/MacUpdater.ts:27-33
this.nativeUpdater.on("error", it => {
  this._logger.warn(it)
  this.emit("error", it)
})
```

### 2. 差分ダウンロードの確認
```javascript
// packages/electron-updater/src/MacUpdater.ts:99-108
const canDifferentialDownload = () => {
  if (!pathExistsSync(cachedUpdateFilePath)) {
    log.info("Unable to locate previous update.zip for differential download (is this first install?), falling back to full download")
    return false
  }
  return !downloadUpdateOptions.disableDifferentialDownload
}
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

## まとめ

Macでの自動更新が動作しない最も一般的な原因は：

1. **ファイル形式の問題** - ZIPファイルが必要（DMGは不可）
2. **適切なコード署名の不備** - Developer ID Application証明書での署名が必要
3. **公証（Notarization）の不備** - macOS 10.15以降で必須
4. **権限設定（Entitlements）の不備** - Hardened Runtime使用時に必要
5. **Squirrel.Macの要件** - 内部的にSquirrel.Macを使用するため、その要件を満たす必要

これらの要件は全てelectron-builderのソースコード上で実装・検証されており、順次確認・修正することで、Macでの自動更新機能を正常に動作させることができます。

## 参考リンク

- [electron-builder公式ドキュメント](https://www.electron.build/auto-update)
- [Squirrel.Mac](https://github.com/Squirrel/Squirrel.Mac)
- [electron-notarize](https://github.com/electron/electron-notarize)
- [Apple Developer Documentation](https://developer.apple.com/documentation/security/hardened_runtime)