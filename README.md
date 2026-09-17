# agent-config

Loop Engineering: Agent configuration and automation patterns

## Contents

### skills/
- [pr-babysitter.md](skills/pr-babysitter.md) — PR監視ループ（CI失敗検知・レビュー催促・コンフリクト検知・ステイルPR管理）

### docs/
- [loop-engineering.md](docs/loop-engineering.md) — ループエンジニアリングのルールセット（基本ルール・ループ実行・失敗記録・完了条件）
- [approval-levels.md](docs/approval-levels.md) — エージェントの承認レベル定義（L0自動実行〜L3禁止、アクション別分類）
- [agent-cli-cheatsheet.txt](docs/agent-cli-cheatsheet.txt) — herdr / Claude Code / OpenCode / Codex の操作と用途別プロンプト集

herdr 内のどのペインからでも `Ctrl+/` でチートシートを開くには、`~/.config/herdr/config.toml` に次を追加する。

```toml
[[keys.command]]
key = "ctrl+/"
type = "popup"
command = "LESS='-R -X' less ~/agent-config/docs/agent-cli-cheatsheet.txt"
width = "92%"
height = "90%"
```

設定後に `herdr config check` で検証し、起動中なら `herdr server reload-config` で反映する。ポップアップは `q` で閉じる。

### templates/
- [loop-engineering-instructions.md](templates/loop-engineering-instructions.md) — CLAUDE.md / Project Instructions にそのまま貼り付けて使えるテンプレート
- [ISSUE_TEMPLATE/](templates/ISSUE_TEMPLATE/) — Issue テンプレート
