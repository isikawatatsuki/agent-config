# ADR 0001: CLI エージェントから社内 LLM を呼ぶ経路とモデル制約

- ステータス: Accepted
- 決定日: 2026-09-09
- 関連: [approval-levels.md](../approval-levels.md), [loop-engineering.md](../loop-engineering.md)

## コンテキスト

CLI で動くコーディングエージェント（Claude Code / opencode など）から LLM を呼ぶ経路が複数ある。どの経路をどう使うかが未定義だと、使えないモデルを既定にしたり、組織のアクセス制御を意図せず回避したりする。

利用可能な経路と、2026-09-09 時点で実測した状態:

| 経路 | 認証 | 状態 |
|---|---|---|
| Amazon Bedrock（AIDD アカウント） | AWS SSO プロファイル | 利用可。`jp.` / `global.` 接頭辞のモデル ID のみ有効 |
| GitHub Copilot Business seat | エディタプラグインの OAuth アプリ | 利用可。`chat_enabled: true`、`sku: copilot_for_business_seat_quota` |
| GitHub Copilot CLI | Copilot CLI 独自のポリシー | **利用不可**。`403 unauthorized: not authorized to use this Copilot feature`（組織ポリシーで CLI が無効） |

Copilot 経路には、さらに **クライアント単位のモデル allowlist** がある。クライアントが送る `Copilot-Integration-Id` ヘッダで使えるモデルが変わり、ヘッダを送らない場合は GitHub が OAuth アプリからクライアントを推定する（エディタプラグインのアプリは `copilot_language_server` として扱われる）。

opencode は `Copilot-Integration-Id` を送らず、OAuth トークン（`ghu_`）をそのまま Bearer に載せる。この形を raw HTTP で再現して各モデル 6 回試した結果:

| モデル | 成功率 | 失敗時のエラー |
|---|---|---|
| `claude-opus-4.7` / `claude-haiku-4.5` / `gpt-5.5` | **6/6** | — |
| `claude-opus-4.8` | 4/6 | 400 `model_not_available_for_integrator` |
| `claude-sonnet-5` | 3/6 | 400 `model_not_available_for_integrator` |
| `gpt-6-astra` | 3/6 | 403 ToS（`x-endpoint-integration-forbidden: tpm:...:clientID:copilot_language_server`） |
| `claude-opus-5` | **0/6** | 403 ToS（同上。200 が返る場合も `choices: []` で本文が空） |

重要なのは、**この不安定さが GitHub 側で発生している**こと。完全に同一のリクエストを連続で投げても 200 / 400 / 403 が混ざるため、クライアント実装の問題ではなく、GitHub 側のクライアント別制限の適用が一貫していない。

`Copilot-Integration-Id: vscode-chat`（VS Code Copilot Chat の識別子）を明示すると、上記すべてが安定して 200 で通る。つまり制限はヘッダ一つで越えられる。

なお `gpt-6-astra` / `gpt-5.5` / `gpt-5.6-*` / `gpt-5.3-codex` は `supported_endpoints` が `/responses` のみで、`/chat/completions` には存在しない（`unsupported_api_for_model`）。クライアント側が Responses API に対応していないとそもそも呼べない。

## 決定

1. **主経路は Bedrock**。長文脈（200K超）、最新世代モデル（Opus 5 等）、明示的なプロンプトキャッシュ、Batch / Files API、Anthropic 製サーバーツールが必要な作業は Bedrock 経路を使う。
2. **副経路として Copilot を使う**。既存 seat を活かした日常的なコーディング作業に限る。
3. **Copilot 経路の既定モデルは、実測で成功率 100% のモデルから選ぶ**。2026-09-09 時点では `claude-opus-4.7`（代替: `claude-haiku-4.5`、GPT 系が必要なら `gpt-5.5`）。成功率が 100% でないモデル（`claude-opus-4.8` 4/6、`claude-sonnet-5` 3/6、`gpt-6-astra` 3/6）を既定にしない。モデルを追加・変更するときは連続 6 回以上試して成功率を確認する。
4. **`Copilot-Integration-Id` を偽装してモデル制限を越えることは禁止（L3 相当）**。GitHub が ToS を根拠に明示的に拒否しているクライアント制限であり、技術的に可能でもアクセス制御の回避に当たる。必要なら組織管理者に正規の解放を依頼する。
5. **認証トークンの取り違えを防ぐ**。Copilot 用トークン（会社アカウント）を `GITHUB_TOKEN` で渡す場合、`GH_TOKEN` に `gh` 自身のトークン（個人アカウント）を明示的に入れ直し、セッション内の `gh` / `glab` が別アカウントにすり替わらないようにする。
6. **承認レベルは経路に依存しない**。[approval-levels.md](../approval-levels.md) の L0〜L3 をエージェント側の permission 機構にマップする（L0→allow / L2→ask / L3→deny）。permission をバイパスする起動オプション（`--dangerously-skip-permissions` 等、およびそれを付与するランチャの yolo モード）は使わない。

## 帰結

**得られるもの**

- 既存 Copilot seat を追加コストなしでコーディングエージェントに使える
- どのモデルが使えるかの判断基準が固定され、モデル選択の試行錯誤が不要になる
- 承認レベルの定義が経路をまたいで一貫する

**受け入れる制約**

- Copilot 経路ではプロンプト 200K / コンテキスト 264K / 最大出力 64K が上限
- Copilot 経路では明示的プロンプトキャッシュ・Batch API・Anthropic 製サーバーツールが使えない
- Copilot 経路のリクエストは GitHub / Microsoft のプロキシを通る。データ処理主体が Anthropic 直・Bedrock と異なる
- モデルの可用性は GitHub 側と組織管理者のポリシーで一方的に変わりうる。allowlist は定期的に再確認が必要（`model_not_available_for_integrator` のエラー本文が現在の allowlist を返す）
- 制限の適用が GitHub 側で一貫していないため、成功率 100% だったモデルが後日不安定になる可能性がある。ループ運用ではモデル起因の失敗を「リトライで解決しない失敗」として扱い、[loop-engineering.md](../loop-engineering.md) の失敗記録に残す

## クライアント側（opencode 1.2.6）で別途確認した不具合

GitHub 側の制限とは切り分けて記録する。これらは経路の問題ではなくクライアント実装の問題:

- `Copilot-Integration-Id` を送らない。そのため GitHub の推定に委ねる形になり、上記の不安定さをそのまま受ける
- Responses API 専用モデルを `/chat/completions` に投げるケースがある（`gpt-6-astra` で `unsupported_api_for_model` を実測）
- タイトル生成用の小モデル（`claude-haiku-4.5`）に `reasoning_effort: "low"` を送って毎回失敗している（`reasoning_effort "low" was provided, but model claude-haiku-4.5 ...`）。本体の応答には影響しない

## 検討して却下した選択肢

| 選択肢 | 却下理由 |
|---|---|
| `Copilot-Integration-Id` を `vscode-chat` に差し替えて全モデルを解放する | GitHub が ToS を根拠に拒否しているクライアント制限の回避に当たる。決定 4 の通り禁止 |
| Copilot CLI の組織承認を待ってから CLI 系エージェントを整備する | 承認時期が読めない。opencode 経路が今すぐ使えるため待つ必要がない |
| Bedrock 一本に統一する | 既に支払っている Copilot seat が遊ぶ。日常的なコーディング作業には Copilot 経路で足りる |
| 不安定なモデル（`claude-opus-4.8`）を既定にしてリトライで凌ぐ | 3回に1回しか通らず、ループ運用で失敗が積み上がる |

## 再確認の手順

allowlist の変化は次で確認できる（allowlist 外のモデルを指定すると、エラー本文に現在の allowlist が列挙される）:

```sh
TOK=$(python3 -c "import json;d=json.load(open('$HOME/.config/github-copilot/apps.json'));print(list(d.values())[0]['oauth_token'])")
CT=$(curl -s -H "Authorization: token $TOK" -H "Editor-Version: vscode/1.99" \
  https://api.github.com/copilot_internal/v2/token | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")
curl -s -H "Authorization: Bearer $CT" -H "Content-Type: application/json" \
  -H "Editor-Version: vscode/1.99" -H "Copilot-Integration-Id: copilot-language-server" \
  -d '{"model":"__probe__","messages":[{"role":"user","content":"hi"}],"max_tokens":16}' \
  https://api.githubcopilot.com/chat/completions
```
