---
title: "【Mac 実機不要】GitHub Actions で iOS アプリをビルドする"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "ios", "xcode", "mac"]
published: false
---

## はじめに

GitHub Actions は、GitHub 上でワークフローを自動化するための機能です。
この記事では、GitHub Actions を使って Mac を持っていなくても iOS アプリをビルドする方法を紹介します。

:::message
App Store Connect にアップロードするため、Apple Developer Program の登録が必要です。
:::

## 1. リポジトリの準備

まずは、iOS アプリのリポジトリーを用意します。
この記事では、以下のようなディレクトリー構造を想定しています。

```
.
├── .github
│   └── workflows
│       └── build.yml
├── MyApp
│   ├── MyApp.xcodeproj
│   └── ...
└── ...
```

## 2. GitHub Actions の設定

リポジトリーのルートディレクトリーに `.github/workflows/build.yml` を作成し、以下の内容を記述します。

```yaml:build.yml
name: Build

on:
  push

jobs:          
  build:
    name: Build
    runs-on: macos-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build Project
        run: |
          xcodebuild build -project MyApp/MyApp.xcodeproj \
                           -scheme MainApp \
                           -sdk iphoneos \
                           -configuration Release \
                           CODE_SIGNING_ALLOWED=NO \

      - name: Archive Project
        run: |                        
          xcodebuild archive -project MyApp/MyApp.xcodeproj \
                             -scheme MainApp \
                             -sdk iphoneos \
                             -configuration Release \
                             -archivePath MainApp.xcarchive \
                             CODE_SIGNING_ALLOWED=NO

      - name: Create ExportOptions.plist
        run: |
          echo '${{ secrets.EXPORT_OPTIONS }}' > ExportOptions.plist

      - name: Create Private Key
        run: |
          mkdir private_keys
          echo -n '${{ secrets.APPLE_API_KEY_BASE64 }}' | base64 --decode > ./private_keys/AuthKey_${{ secrets.APPLE_API_ISSUER_ID }}.p8

      - name: Export IPA
        run: |   
          xcodebuild -exportArchive \
                     -archivePath MainApp.xcarchive \
                     -exportOptionsPlist ExportOptions.plist \
                     -exportPath app.ipa \
                     -allowProvisioningUpdates \
                     -authenticationKeyPath `pwd`/private_keys/AuthKey_${{ secrets.APPLE_API_ISSUER_ID }}.p8 \
                     -authenticationKeyID ${{ secrets.APPLE_API_KEY_ID }} \
                     -authenticationKeyIssuerID ${{ secrets.APPLE_API_ISSUER_ID }}

      - name: Upload IPA to App Store Connect
        run: |
          xcrun altool --upload-app -f app.ipa/MainApp.ipa \
                       -u ${{ secrets.APPLE_ID }} \
                       -p ${{ secrets.APP_SPECIFIC_PASSWORD }} \
                       --type ios
```

この設定では、`xcodebuild` コマンドを使って iOS アプリをビルドし、`xcrun altool` コマンドを使って App Store Connect にアップロードしています。

重要なポイントは以下の通りです。

- Xcode プロジェクトのパスは `MyApp/MyApp.xcodeproj`、スキーム名は `MainApp` としています。適宜変更してください。
- Xcode のプロジェクトで自動署名を有効にしてください。
- `xcodebuild build`、`xcodebuild archive` するときには、署名を無効にするために `CODE_SIGNING_ALLOWED=NO` を指定しています。
- Private Key のパスは内部で暗黙的に参照されるため、変更しないでください。
- `-allowProvisioningUpdates` を指定しているので、プロビジョニングファイルを必要としません。
また、プロビジョニングファイルを 1 年ごとに更新する必要がありません。

## 3. シークレットの設定

GitHub のリポジトリの設定画面から、以下のシークレットを設定します。

- `EXPORT_OPTIONS`: `ExportOptions.plist` の内容
- `APPLE_API_KEY_BASE64`: App Store Connect API のキーの内容を Base64 エンコードしたもの
- `APPLE_API_ISSUER_ID`: App Store Connect API の Issuer ID
- `APPLE_API_KEY_ID`: App Store Connect API のキー ID
- `APPLE_ID`: Apple ID
- `APP_SPECIFIC_PASSWORD`: Apple ID のアプリ用パスワード

### `ExportOptions.plist` の作成

`EXPORT_OPTIONS` は、以下のような内容です。
`YOUR_TEAM_ID` は、Apple Developer Program の [Membership](https://developer.apple.com/account/) から確認できます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
</dict>
</plist>
```

### App Store Connect API のキーの作成

App Store Connect API のキーは、[App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api) から作成できます。
名前は任意ですが、`App Manager` または `Admin` のアクセスが必要です。

`APPLE_API_KEY_BASE64` は、キーの内容です。
一度しかダウンロードできないため、注意してください。
Base64 エンコードしてシークレットに設定します。

#### Base64 エンコードの方法

以下のコマンドで Base64 エンコードできます。

```ps:Windows (PowerShell)
[Convert]::ToBase64String([System.IO.File]::ReadAllBytes("AuthKey_YOUR_ISSUER_ID.p8"))
```

```sh:Linux / macOS
base64 -i AuthKey_YOUR_ISSUER_ID.p8
``

`APPLE_API_ISSUER_ID` は、キーの発行者 ID です。
すべてのキーに共通です。

`APPLE_API_KEY_ID` は、キー ID です。

### アプリ用パスワードの設定

`APPLE_ID` は、Apple ID（メールアドレス）です。
Apple Developer Program の登録に使用した Apple ID を設定してください。

`APP_SPECIFIC_PASSWORD` は、Apple ID の [アプリ用パスワード](https://appleid.apple.com/account/manage) から作成したアプリ用パスワードです。

## 4. 実行

これで準備完了です。

GitHub に push すると、GitHub Actions が自動的に実行されます。
ビルドが成功すると、App Store Connect にアプリがアップロードされます。
App Store Connect から審査不要の内部テスト用リリースを作成することができます。
お手持ちの iPhone でテストしてみてください！

## うまくいかないときは

ワークフローが失敗する場合は、以下の点を確認してください。

- Xcode プロジェクトのパスやスキーム名は正しいか。
`xcodebuild -list -project MyApp/MyApp.xcodeproj` でスキーム名を確認できます。
- Xcode プロジェクトで自動署名は有効か。
- シークレットはすべて設定されているか。
- `APPLE_API_KEY_BASE64` は Base64 エンコードされているか。
- App Store Connect API のキーの権限は正しいか。
- app.ipa ディレクトリーに `MainApp.ipa` が作成されているか。
プロジェクトによってはほかの名前になることがあります。
`ls -l app.ipa` で確認できます。

## おわりに

この記事では、GitHub Actions を使って Mac を持っていなくても iOS アプリをビルドする方法を紹介しました。
ぜひお試しください。
