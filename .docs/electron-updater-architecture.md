# electron-updater アーキテクチャドキュメント

## 概要

electron-updaterは、Electronアプリケーションの自動更新機能を提供するライブラリです。このドキュメントでは、electron-updaterが外部プロジェクトにおいてpackage.jsonを読み込み、アップデート設定を生成・処理する仕組みについて詳しく説明します。

## アーキテクチャ概要

```
外部プロジェクト
├── package.json (repository情報)
├── electron-builder設定
└── ビルド時
    ├── electron-builderがapp-update.yml生成
    └── 実行時にelectron-updaterが読み込み
```

## 1. 設定の読み込み順序

electron-builderは以下の優先順位でpublish設定を決定します：

### 1.1 設定の優先順位
1. **ターゲット固有設定** (`build.nsis.publish`)
2. **プラットフォーム固有設定** (`build.win.publish`, `build.mac.publish`, `build.linux.publish`)
3. **グローバル設定** (`build.publish`)
4. **環境変数からの自動検出** (`GITHUB_TOKEN`, `GH_TOKEN`等)
5. **package.jsonのrepository情報からの自動生成** ← 重要！

### 1.2 package.jsonからの自動生成条件
- 明示的なpublish設定が存在しない場合
- `package.json`に`repository`フィールドが存在する場合
- repositoryのtypeが`github`の場合

## 2. 主要な処理フロー

### 2.1 ビルド時の処理 (electron-builder)

```typescript
// packages/app-builder-lib/src/publish/PublishManager.ts:275-299
export async function getPublishConfigsForUpdateInfo(
  packager: PlatformPackager<any>,
  publishConfigs: Array<PublishConfiguration> | null,
  arch: Arch | null
): Promise<Array<PublishConfiguration> | null> {
  if (publishConfigs.length === 0) {
    // 明示的なpublish設定がない場合
    const repositoryInfo = await packager.info.repositoryInfo
    if (repositoryInfo != null && repositoryInfo.type === "github") {
      // package.jsonのrepository情報からGitHub設定を自動生成
      const resolvedPublishConfig = await getResolvedPublishConfig(
        packager, packager.info, 
        { provider: repositoryInfo.type }, 
        arch, false
      )
      return [resolvedPublishConfig]
    }
  }
  return publishConfigs
}
```

### 2.2 repository情報の解析

```typescript
// packages/app-builder-lib/src/util/repositoryInfo.ts:8-32
export function getRepositoryInfo(
  projectDir: string, 
  metadata?: Metadata, 
  devMetadata?: Metadata | null
): Promise<SourceRepositoryInfo | null> {
  return _getInfo(
    projectDir, 
    (devMetadata?.repository) || (metadata?.repository)
  )
}

async function _getInfo(
  projectDir: string, 
  repo?: RepositoryInfo | string | null
): Promise<SourceRepositoryInfo | null> {
  if (repo != null) {
    // package.jsonのrepository.urlを解析
    return parseRepositoryUrl(typeof repo === "string" ? repo : repo.url)
  }
  
  // CI環境変数からも取得可能
  const slug = process.env.TRAVIS_REPO_SLUG || process.env.APPVEYOR_REPO_NAME
  if (slug != null) {
    const [user, project] = slug.split("/")
    return { user, project }
  }
  
  // .git/configからも取得可能
  const url = await getGitUrlFromGitConfig(projectDir)
  return url == null ? null : parseRepositoryUrl(url)
}
```

### 2.3 publish設定の詳細解決

```typescript
// packages/app-builder-lib/src/publish/PublishManager.ts:556-596
async function getResolvedPublishConfig(...): Promise<PublishConfiguration | null> {
  // owner/projectが不足している場合
  if (!owner || !project) {
    const info = await packager.repositoryInfo  // package.jsonから取得
    if (info != null) {
      if (!owner) owner = info.user      // GitHubのowner
      if (!project) project = info.project  // GitHubのrepo名
    }
  }
  
  // 最終的なGitHub設定を返す
  return { owner, repo: project, ...options } as GithubOptions
}
```

## 3. app-update.yml生成処理

### 3.1 生成タイミング
electron-builderは各プラットフォームのビルド後に`app-update.yml`を生成します：

```typescript
// packages/app-builder-lib/src/publish/PublishManager.ts:111-126
packager.onAfterPack(async event => {
  const publishConfig = await getAppUpdatePublishConfiguration(
    packager, event.arch, this.isPublish
  )
  if (publishConfig != null) {
    await writeFile(
      path.join(packager.getResourcesDir(event.appOutDir), "app-update.yml"), 
      serializeToYaml(publishConfig)
    )
  }
})
```

### 3.2 生成される app-update.yml の例

```yaml
# package.json の repository が以下の場合:
# "repository": {
#   "type": "git",
#   "url": "https://github.com/username/my-app.git"
# }

provider: github
owner: username
repo: my-app
updaterCacheDirName: my-app-updater
```

## 4. 実行時の処理 (electron-updater)

### 4.1 設定ファイルの読み込み

electron-updaterは実行時に以下のファイルから設定を読み込みます：

- **プロダクション**: `process.resourcesPath/app-update.yml`
- **開発環境**: `app.getAppPath()/dev-app-update.yml`

### 4.2 アップデート処理フロー

```typescript
// packages/electron-updater/src/AppUpdater.ts (概念的なフロー)
class AppUpdater {
  async checkForUpdates(): Promise<UpdateCheckResult> {
    // 1. app-update.ymlから設定を読み込み
    const publishConfig = await this.loadUpdateConfig()
    
    // 2. 設定に基づいてアップデート情報を取得
    const updateInfo = await this.getUpdateInfo(publishConfig)
    
    // 3. 現在のバージョンと比較
    if (this.isUpdateAvailable(updateInfo)) {
      return { updateInfo, downloadPromise: this.downloadUpdate(updateInfo) }
    }
    
    return { updateInfo: null }
  }
}
```

## 5. 具体的な設定例

### 5.1 最小限の設定 (package.json)

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/my-electron-app.git"
  },
  "build": {
    // publish設定は不要 - repositoryから自動生成される
  }
}
```

### 5.2 明示的な設定 (package.json)

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "build": {
    "publish": {
      "provider": "github",
      "owner": "username",
      "repo": "my-electron-app"
    }
  }
}
```

### 5.3 プラットフォーム固有の設定

```json
{
  "build": {
    "win": {
      "publish": {
        "provider": "github",
        "owner": "username",
        "repo": "my-electron-app"
      }
    },
    "mac": {
      "publish": {
        "provider": "s3",
        "bucket": "my-updates-bucket"
      }
    }
  }
}
```

## 6. 重要なファイルとその役割

### 6.1 electron-builder側
- `packages/app-builder-lib/src/publish/PublishManager.ts`
  - publish設定の解決とapp-update.yml生成
  - repository情報からの自動設定生成
- `packages/app-builder-lib/src/util/repositoryInfo.ts`
  - package.jsonのrepository情報解析
  - Git設定やCI環境変数からの情報取得

### 6.2 electron-updater側
- `packages/electron-updater/src/AppUpdater.ts`
  - メインのアップデート処理クラス
  - app-update.ymlの読み込みと処理
- `packages/electron-updater/src/providers/`
  - 各プロバイダー（GitHub、S3等）の実装

## 7. デバッグとトラブルシューティング

### 7.1 設定確認方法

```bash
# ビルド時のログでrepository情報を確認
DEBUG=electron-builder:publish npm run build

# 生成されたapp-update.ymlを確認
cat dist/win-unpacked/resources/app-update.yml
```

### 7.2 よくある問題

1. **repository情報が不正**
   - package.jsonのrepository.urlが正しく設定されているか確認
   - URLの形式が正しいか確認（https://github.com/owner/repo.git）

2. **publish設定の競合**
   - 複数のpublish設定が存在する場合、優先順位を確認
   - 意図しない設定が適用されていないか確認

3. **app-update.ymlが生成されない**
   - 対象プラットフォームでアップデートがサポートされているか確認
   - ビルドターゲットが適切か確認（NSIS、DMG、AppImage等）

## 8. まとめ

electron-updaterのアーキテクチャは以下の特徴を持ちます：

1. **設定の階層化**: ターゲット → プラットフォーム → グローバル → 自動検出の順序
2. **自動設定生成**: package.jsonのrepository情報から自動的にGitHub設定を生成
3. **ビルド時生成**: electron-builderがapp-update.ymlを生成
4. **実行時読み込み**: electron-updaterがapp-update.ymlを読み込んでアップデート処理

この仕組みにより、最小限の設定でも強力な自動更新機能を実現できます。 