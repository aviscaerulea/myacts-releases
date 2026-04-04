# myacts

指定日の Google Workspace アクティビティ（Chat・Calendar・Gmail・Drive）を JSON で返す Windows CLI ツール。

各ユーザーが自分の PC で OAuth 認証して実行するため、マルチユーザー対応かつ実行時間制限なし。
MCP サーバとしても動作し、Claude Code や Gemini CLI からツールとして直接呼び出せる。

## ダウンロード

[Releases ページ](https://github.com/aviscaerulea/myacts-releases/releases)から最新の zip をダウンロードして任意のフォルダに展開する。

| ファイル | 説明 |
|---|---|
| `myacts.exe` | 実行ファイル |
| `myacts.toml` | 設定ファイルテンプレート |

## 動作要件

- Windows 10/11 (x64)
- GCP プロジェクトで OAuth 2.0 クライアント ID（デスクトップアプリ）を作成済み
- 以下の Google API を有効化済み
  - Google Chat API
  - Google Calendar API
  - Gmail API
  - Google Drive API
  - People API

## セットアップ

### 1. 認証

```
myacts auth
```

ブラウザが開いて Google 認証画面が表示される。認証後、トークンが `%APPDATA%\myacts\token.json` に保存される。

### 2. 設定（任意）

Redmine 連携やグループアドレス展開が必要な場合は `myacts.toml` を編集する。

## 使い方

```
# 個別取得
myacts chat     --date 2026-03-15
myacts calendar --date 2026-03-15
myacts mail     --date 2026-03-15
myacts drive    --date 2026-03-15

# 全メディア一括取得
myacts all --date 2026-03-15

# 期間指定
myacts chat --date "2026-03-15 09:00" --end "2026-03-15 18:00"

# フィールド絞り込み
myacts calendar --date 2026-03-15 --fields content,datetime,permalink

# 認証状態確認
myacts auth --status
```

## MCP サーバ

```
myacts mcp
```

stdio トランスポート（JSON-RPC）で動作し、以下の 9 ツールを公開する。

| ツール名 | 説明 |
|---|---|
| `myacts_chat` | Chat 送信メッセージの取得 |
| `myacts_calendar` | Calendar イベントの取得 |
| `myacts_mail` | Gmail 送信メールの取得 |
| `myacts_drive` | Drive 更新ファイルの取得 |
| `myacts_redmine` | Redmine チケットの取得 |
| `myacts_all` | 全メディアの一括取得 |
| `myacts_schema` | 全メディアの出力 JSON スキーマの取得 |
| `myacts_cache_set` | キャッシュエントリの値を更新 |
| `myacts_member` | メールアドレスから氏名への解決 |

### Claude Code への登録

事前に `myacts auth` で認証を済ませておくこと。

```bash
# 全プロジェクト共通で登録（推奨）
claude mcp add --scope user myacts -- C:\path\to\myacts.exe mcp

# 特定プロジェクト固有で登録
claude mcp add --scope project myacts -- C:\path\to\myacts.exe mcp
```

### Gemini CLI への登録（`~/.gemini/settings.json`）

```json
{
  "mcpServers": {
    "myacts": {
      "command": "C:\\path\\to\\myacts.exe",
      "args": ["mcp"]
    }
  }
}
```

## キャッシュ

Google People API による名前解決結果を SQLite にキャッシュして API コール数を削減する。

- `myacts.db`（実行ファイルと同ディレクトリ）に保存
- TTL はデフォルト 7 日（`myacts.toml` の `[cache] ttl_days` で変更可能）

```bash
myacts cache list   # 一覧表示
myacts cache set <table> <key> <value>   # 手動修正
myacts cache clear  # 全削除
```

## ログ

実行ログは実行ファイルと同ディレクトリの `logs/myacts-YYYY-MM-DD.log` に保存される。
`--verbose` フラグを指定すると DEBUG レベルのログも出力する。30 日超の古いログは起動時に自動削除される。
