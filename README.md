# myacts
[![日本語](https://img.shields.io/badge/lang-日本語-red)](README.md)
[![English](https://img.shields.io/badge/lang-English-blue)](README.en.md)

指定日の Google Workspace アクティビティ（Chat・Calendar・Gmail・Drive）と Redmine チケットを JSONL で返す Go CLI ツールです。

## 機能

- Google Chat・Calendar・Gmail・Drive・Redmine のアクティビティを JSONL で取得
- フィールド絞り込み、期間指定に対応
- 氏名解決（メールアドレス → 表示名）結果を SQLite にキャッシュして API コール数を削減
- グループアドレスの展開（Cloud Identity Groups API、入れ子グループを深さ無制限で展開）
- エージェント向けスキル定義（SKILL.md）をバイナリに同梱し、`myacts skill` でスキルディレクトリへ導入

## インストール

### 動作要件

- Windows 10/11（x64）
- 初回に `myacts auth` で Google OAuth 認証が必要

### 手順

#### リリースの ZIP から

[Releases](https://github.com/aviscaerulea/myacts-releases/releases) から ZIP をダウンロードします。任意のディレクトリに展開します。

#### Scoop から

```bash
scoop bucket add aviscaerulea https://github.com/aviscaerulea/scoop-bucket
scoop install myacts
```

#### アンインストール

`myacts uninstall` が `%APPDATA%\myacts`（認証トークン、設定）、`%LOCALAPPDATA%\myacts`（キャッシュ、ログ）、スキルディレクトリを確認なしで削除し、削除したパスを表示します。その後、ZIP で展開した場合は展開先を削除し、Scoop の場合は `scoop uninstall myacts` を実行します。

## 使い方

```bash
# 初回認証
myacts auth

# 認証状態確認
myacts auth --status

# 個別取得
myacts chat     --date 2026-03-15
myacts calendar --date 2026-03-15
myacts mail     --date 2026-03-15
myacts drive    --date 2026-03-15

# フィールド絞り込み（calendar は members/sender 省略で高速化）
myacts calendar --date 2026-03-15 --fields content,datetime,permalink

# 期間指定
myacts chat --date "2026-03-15 09:00" --end "2026-03-15 18:00"

# 氏名解決
myacts member --emails a@example.com,b@example.com

# 出力スキーマの確認（メディアを指定しなければ全メディア分をまとめて表示）
myacts --schema
myacts chat --schema

# バージョン表示
myacts version

# エージェント向けスキル定義（SKILL.md）を導入・更新
# %USERPROFILE%\.claude と %USERPROFILE%\.agents のうち存在するすべての skills\myacts\ に書き込み、書き込んだパスを表示
myacts skill
```

スキル定義が未導入か、アプリより古いとき、`version`、`skill`、`uninstall` 以外のコマンドは実行時に `myacts skill` の実行を標準エラー出力で案内します。

### グループアドレス展開

`myacts member` は、氏名解決できないアドレスをグループとみなし、Cloud Identity Groups API でメンバーを展開し、入れ子のグループも深さの上限なく再帰的に展開します。
グループのメンバー閲覧権限がない場合は、そのアドレスを展開せずアドレス文字列のまま返します。

この機能には、OAuth スコープへの `cloud-identity.groups.readonly` の追加が必要です。既存ユーザは `myacts auth` を再実行するとスコープを更新できます。

### キャッシュ管理

```bash
# キャッシュ一覧表示（期限切れ含む）
myacts cache list

# キャッシュエントリの手動修正（表示名の誤りを修正する場合など）
myacts cache set <table> <key> <value>

# キャッシュ全削除
myacts cache clear
```

### ログ

myacts は実行ログを `%LOCALAPPDATA%\myacts\logs\myacts-YYYY-MM-DD.log` に保存します。
`--verbose` フラグを指定すると DEBUG レベルのログも出力します（デフォルト：INFO 以上）。
myacts は起動時に 30 日超の古いログファイルを自動で削除します。

## 設定

設定ファイル `myacts.toml` は、実行ファイルと同じディレクトリ、次に `%APPDATA%\myacts\` の順で探します。
`[redmine]` に接続先を設定すると Redmine チケットを取得できます。`[cache]` でキャッシュ DB の保存先と保存日数を変更できます。

## 制限事項

- 認証できるのは、配布元が指定した Google Workspace 組織のメンバーのみだ  
  組織外のアカウントでは `myacts auth` が失敗します。

- Redmine チケットの取得には `myacts.toml` への接続先設定が必要だ  
  未設定の場合、`redmine` コマンドは使えません。

- グループのメンバー閲覧権限がない場合、そのグループアドレスは展開せずアドレス文字列のまま返す

## ビルド方法（開発者向け）

ビルドして配布バイナリを作成するための手順です。エンドユーザは `myacts auth` を実行して Google 認証するだけでよく、このセクションの作業は不要です。

### 前提

- Go 1.26.1 以上
- GCP プロジェクトで OAuth 2.0 クライアント ID（デスクトップアプリ）を作成済み
- GCP プロジェクトで以下の API を有効化済み
  - Google Chat API
  - Google Calendar API
  - Gmail API
  - Google Drive API
  - People API
  - Cloud Identity API

### ビルド

プロジェクトルートに `.env` ファイルを作成します。ファイルにクレデンシャルを設定します。

```
GOOGLE_CLIENT_ID=your_client_id
GOOGLE_CLIENT_SECRET=your_client_secret
```

```bash
task build
```

ビルド成果物は `out/myacts.exe` です。
