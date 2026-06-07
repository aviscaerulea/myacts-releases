# myacts

指定日の Google Workspace アクティビティ（Chat・Calendar・Gmail・Drive）と Redmine チケットを JSON で返す Go CLI ツール。

## 機能

- Google Chat・Calendar・Gmail・Drive・Redmine のアクティビティを JSON で取得
- 全メディアの一括取得、フィールド絞り込み、期間指定に対応
- Claude Code・Gemini CLI などから利用できる MCP サーバモード
- 氏名解決（メールアドレス → 表示名）結果を SQLite にキャッシュして API コール数を削減
- グループアドレスの展開（GAS 版 member API との連携）

## 動作要件

- Windows 10/11（x64）
- 初回に `myacts auth` で Google OAuth 認証が必要

## インストール方法

### Scoop

```bash
scoop bucket add aviscaerulea https://github.com/aviscaerulea/scoop-bucket
scoop install myacts
```

### 手動

[Releases](https://github.com/aviscaerulea/myacts/releases) から zip をダウンロードして任意のディレクトリに展開する。

## 使用方法

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

# 全メディア一括取得
myacts all --date 2026-03-15

# フィールド絞り込み（calendar は members/sender 省略で高速化）
myacts calendar --date 2026-03-15 --fields content,datetime,permalink

# 期間指定
myacts chat --date "2026-03-15 09:00" --end "2026-03-15 18:00"

# 氏名解決
myacts member --emails a@example.com,b@example.com
```

### MCP サーバ

Claude Code 等の MCP クライアントからツールとして呼び出せる。

```bash
myacts mcp
```

stdio トランスポート（JSON-RPC）で動作し、以下の 9 ツールを公開する。

| ツール名 | 説明 |
|----------|------|
| `myacts_chat` | Chat 送信メッセージの取得 |
| `myacts_calendar` | Calendar イベントの取得 |
| `myacts_mail` | Gmail 送信メールの取得 |
| `myacts_drive` | Drive 更新ファイルの取得 |
| `myacts_redmine` | Redmine チケットの取得 |
| `myacts_all` | 全メディアの一括取得 |
| `myacts_schema` | 全メディアの出力 JSON スキーマの取得 |
| `myacts_cache_set` | キャッシュエントリの値を更新 |
| `myacts_member` | メールアドレスから氏名への解決 |

事前に `myacts auth` で認証を済ませておくこと。未認証の場合、起動時にブラウザ認証フローが走る。

#### Claude Code

```bash
# 全プロジェクト共通で登録（推奨）
claude mcp add --scope user myacts -- /path/to/myacts.exe mcp

# 特定プロジェクト固有で登録（.mcp.json に保存）
claude mcp add --scope project myacts -- /path/to/myacts.exe mcp
```

#### Gemini CLI（`~/.gemini/settings.json`）

```json
{
  "mcpServers": {
    "myacts": {
      "command": "/path/to/myacts.exe",
      "args": ["mcp"]
    }
  }
}
```

### グループアドレス展開

グループアドレスの展開には GAS 版の `media=member` API が必要。
`myacts.toml` に以下を設定する。

```toml
[gas]
member_url = "https://script.google.com/macros/s/{DEPLOY_ID}/exec"
member_token = "YOUR_GAS_API_TOKEN"
```

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

実行ログは実行ファイルと同ディレクトリの `logs/myacts-YYYY-MM-DD.log` に保存される。
`--verbose` フラグを指定すると DEBUG レベルのログも出力する（デフォルト：INFO 以上）。
30 日超の古いログファイルは起動時に自動削除される。

## ビルド方法（開発者向け）

ビルドして配布バイナリを作成するための手順。エンドユーザは `myacts auth` を実行して Google 認証するだけでよく、このセクションの作業は不要。

### 前提

- Go 1.26 以上
- GCP プロジェクトで OAuth 2.0 クライアント ID（デスクトップアプリ）を作成済み
- GCP プロジェクトで以下の API を有効化済み
  - Google Chat API
  - Google Calendar API
  - Gmail API
  - Google Drive API
  - People API

### ビルド

プロジェクトルートに `.env` ファイルを作成してクレデンシャルを設定する。

```
GOOGLE_CLIENT_ID=your_client_id
GOOGLE_CLIENT_SECRET=your_client_secret
```

```bash
task build
```

ビルド成果物は `out/myacts.exe`。

## 技術仕様

- Go 1.26
- cobra（CLI フレームワーク）
- golang.org/x/oauth2（OAuth PKCE 認証）
- google.golang.org/api（Google API SDK）
- BurntSushi/toml（設定ファイル）
- SQLite（キャッシュ）

詳細な仕様は [spec.md](spec.md) を参照。

