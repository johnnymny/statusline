# statusline (customized)

[usedhonda/statusline](https://github.com/usedhonda/statusline) をベースにカスタマイズしたClaude Code用ステータスライン。

## 変更点

### Windows対応: ターミナル高さ検出の修正
`get_terminal_height()` のデフォルト値を4→24に変更。Windows環境ではtput/tmux/isattyが全て失敗し、デフォルト4に落ちる→常に1行ミニマルモードになる問題を修正。

### Compact行: autocompact buffer対応 + claudex連携
compaction閾値の計算を `context_size * 0.8` から `context_size - 33K` に変更。1Mコンテキスト（opus[1m]等）で閾値が800Kではなく967Kと正しく表示される。autocompact bufferは33K固定（Claude Code v2.1.21以降）。

さらに、`ANTHROPIC_BASE_URL` が `localhost` / `127.0.0.1` の proxy を指している場合は claudex/custom-provider runtime とみなし、Claude Code 本体の `context_window.used_percentage` が当てにならない環境でも Compact 行を表示できるようにした。

- `~/.claude/claudex-usage/port-<port>.jsonl` の直近 usage を Compact の分子に使う
- 分母は `.codex` session log の `model_context_window - 33K` を優先し、取れなければ Claude Code 側の閾値へ fallback
- `compact_boundary` 以降の usage だけを見るので、手動 `/compact` 後の状態にも追従する

### カラースキーム: 暗め（dim）に変更
ANSI dim属性（`\033[2;xxm`）を使用し、全体的に目に優しい暗めの配色に変更。

### Session行: runtime-aware 使用率表示
元のSession行（5時間ブロックの経過時間）を、Usage APIベースの表示に置き換えた。

- **通常の Claude runtime**: Claude.ai Usage API（`five_hour.utilization` / `seven_day`）を主表示
- **claudex runtime**: Codex の 5時間 / 週間使用率を主表示し、Claude 側は副表示に回す
- Claude 副表示は 5時間が **0% でも非表示にしない**（正しい 0% をそのまま出す）

**表示例（通常 runtime）:**
```
Compact: ██████▒▒▒▒▒▒▒▒▒▒▒▒▒▒ [30%] 50.0K/160.0K
Session: ██████▒▒▒▒▒▒▒▒▒▒▒▒▒▒ [34%] resets 03:59 (wk95% 6d8h)  GLM:█▒▒▒15%(wk9%)  Codex:█▒▒▒37%(wk12%)
```

**表示例（claudex runtime）:**
```
Compact: ████████▒▒▒▒▒▒▒▒▒▒▒▒ [48%] 109.3K/225.4K
Session: ███▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒ [12%] resets 14:10 (wk4% 6d12h)  Claude:▒▒▒▒0%(wk95% 6d8h)  GLM:█▒▒▒15%(wk9%)
```

- **Session 主表示**: 現在の runtime における主要な Usage source
- **Claude / GLM / Codex 副表示**: 外部サービスの使用率（5時間 + 週間）

### 外部サービス使用率の統合表示 (GLM / Codex / Claude副表示)
Session行にZ.AI (GLM) と OpenAI (Codex) の使用率を並列表示。claudex runtime では Claude Usage も副表示に加わる。`~/.claude/statusline-services.json` で設定。

- サービスごとに5時間バー(width=4) + 数値% + 週間%を表示
- 設定ファイルなし → サービス表示スキップ（graceful degradation）
- 各サービスのauth失敗 → そのサービスだけスキップ
- 60秒キャッシュ（TTL設定可能）

## セットアップ

### 1. 依存パッケージ

```bash
pip install curl_cffi
```

`curl_cffi` はTLSフィンガープリントをChromeに偽装し、Cloudflareのbot対策を回避してClaude.ai APIにアクセスする。これにより `cf_clearance` Cookieやheadlessブラウザが不要になる。

### 2. statusline.pyの配置

```bash
cp statusline.py ~/.claude/statusline.py
```

### 3. Claude Codeの設定

`~/.claude/settings.json` に追加:

```json
{
  "statusLine": {
    "type": "command",
    "command": "python -X utf8 C:\\Users\\<username>\\.claude\\statusline.py"
  }
}
```

macOS/Linuxの場合:
```json
{
  "statusLine": {
    "type": "command",
    "command": "python3 ~/.claude/statusline.py"
  }
}
```

### 4. Usage API設定

`~/.claude/claude-usage.json` を作成:

```json
{
    "org_id": "<your-org-id>",
    "session_key": "<your-session-key>",
    "cache_ttl_seconds": 60
}
```

#### org_id の取得
1. ブラウザで https://claude.ai/settings/usage を開く
2. DevTools (F12) → Network タブ
3. `/usage` リクエストのURLから `organizations/<org_id>/usage` の部分をコピー

#### session_key の取得
1. 同じ `/usage` リクエストのResponse Headersを確認
2. `set-cookie: sessionKey=sk-ant-sid01-...` の値をコピー

**注意事項:**
- `session_key` は約1ヶ月有効。期限切れになったらブラウザから再取得
- APIレスポンスで新しい `sessionKey` が返された場合、自動的に設定ファイルを更新する
- APIは60秒間キャッシュされる（`cache_ttl_seconds` で変更可能）
- API取得に失敗した場合、前回のキャッシュ値を表示する

### 5. 外部サービス使用率 (任意)

`~/.claude/statusline-services.json` を作成:

```json
{
  "glm": {
    "keys_file": "/path/to/llm/keys.json",
    "key_name": "zai"
  },
  "codex": {
    "auth_file": "/path/to/.codex/auth.json"
  },
  "cache_ttl_seconds": 60
}
```

- **glm**: Z.AI (GLM) の使用率。`keys_file` は [llm](https://github.com/simonw/llm) の鍵ファイル、`key_name` はその中のキー名
- **codex**: OpenAI Codex の使用率。`auth_file` は Codex CLI の認証ファイル（`access_token` を含むJSON）。claudex runtime では、この値が Session 行の主表示ソースになる
- 不要なサービスはキーごと省略可能
- ファイル自体が存在しなければ全サービスをスキップ

### 6. 表示設定

`statusline.py` 冒頭の設定で表示行を選択:

```python
SHOW_LINE1 = False  # モデル名・gitブランチ・ディレクトリ・メッセージ数
SHOW_LINE2 = True   # Compact: コンテキストウィンドウ使用率
SHOW_LINE3 = True   # Session: runtime-aware 使用率（Claude / Codex）
SHOW_LINE4 = False  # Burn: トークン消費レート（スパークライン）
```

## ベースリポジトリ

- [usedhonda/statusline](https://github.com/usedhonda/statusline)
