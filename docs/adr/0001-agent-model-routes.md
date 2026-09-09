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

Copilot 経路には、さらに **インテグレータ単位のモデル allowlist** がある。クライアントが送る `Copilot-Integration-Id` によって使えるモデルが変わる。opencode は `copilot-language-server` として振る舞うため、以下の実測結果になった。

| モデル | 結果 |
|---|---|
| `claude-opus-4.7` / `claude-opus-4.5` / `claude-sonnet-4.5` / `claude-haiku-4.5` | 安定して成功（4/4） |
| `claude-opus-4.8` / `claude-sonnet-5` | 不安定。`model_not_available_for_integrator` で断続的に失敗（3回中1回成功） |
| `claude-opus-5` | 常に 403。`x-endpoint-integration-forbidden: tpm:claude-opus-5:clientID:copilot_language_server`（ToS ベースのクライアント別制限） |
| `claude-fable-5.1` | クライアント側が未対応 |

`Copilot-Integration-Id` を `vscode-chat` に差し替えると上記すべてが 200 で通ることも実測で確認済み。つまり制限はヘッダ一つで越えられる。

## 決定

1. **主経路は Bedrock**。長文脈（200K超）、最新世代モデル（Opus 5 等）、明示的なプロンプトキャッシュ、Batch / Files API、Anthropic 製サーバーツールが必要な作業は Bedrock 経路を使う。
2. **副経路として Copilot を使う**。既存 seat を活かした日常的なコーディング作業に限る。
3. **Copilot 経路の既定モデルは、インテグレータ allowlist に載っている安定モデルから選ぶ**。2026-09-09 時点では `claude-opus-4.7`。不安定なモデル（`claude-opus-4.8`、`claude-sonnet-5`）を既定にしない。
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
