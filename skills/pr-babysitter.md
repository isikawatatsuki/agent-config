# PR Babysitter

PRの状態を定期的に監視し、停滞や問題を検知して対応するループパターン。

## 概要

- **実行間隔**: 5〜15分
- **ロールアウトレベル**: L1 (Report-only) → L2 (Assisted) → L3 (Unattended)
- **対象**: 指定リポジトリのオープンPR

## 監視項目

### CI ステータス
- CI が失敗した PR を検知
- 失敗ログを分析し、原因を分類（テスト失敗 / lint / ビルドエラー / インフラ）
- L2 以上: 自動修正の PR を提案
- L3: 軽微な修正（lint、型エラー等）は自動コミット

### レビュー状態
- レビュー待ちが一定時間（例: 4時間）を超えた PR を検知
- レビュアーへのリマインドメッセージを送信（Discord）
- Approved だがマージされていない PR を通知

### コンフリクト
- マージコンフリクトが発生した PR を検知
- L2 以上: コンフリクト解消の PR を提案

### ステイル PR
- 一定期間（例: 3日）更新のない PR を検知
- 作成者に状況確認のメッセージを送信

## 実行フロー

```
1. オープンPR一覧を取得 (gh pr list)
2. 各PRについて:
   a. CI ステータスを確認 (gh pr checks)
   b. レビュー状態を確認 (gh pr view)
   c. コンフリクト有無を確認
   d. 最終更新日時を確認
3. 問題のあるPRについて:
   a. L1: Discord に状況レポートを送信
   b. L2: 修正提案 or リマインドを送信
   c. L3: 自動修正を実行
```

## 使用コマンド

```bash
# オープンPR一覧
gh pr list --repo <owner>/<repo> --state open --json number,title,updatedAt,reviewDecision,statusCheckRollup

# PR の CI 状態
gh pr checks <number> --repo <owner>/<repo>

# PR の詳細
gh pr view <number> --repo <owner>/<repo> --json mergeable,reviewDecision,statusCheckRollup,updatedAt
```

## レポートフォーマット（Discord 通知）

```
📋 PR Babysitter Report — <repo>

🔴 CI 失敗:
- #123: タイトル — テスト失敗 (2h前)

⏳ レビュー待ち:
- #456: タイトル — 6h待ち

⚠️ コンフリクト:
- #789: タイトル

😴 ステイル (3日以上):
- #101: タイトル — 5日前
```

## 設定パラメータ

| パラメータ | デフォルト | 説明 |
|-----------|-----------|------|
| `interval` | `15m` | 監視間隔 |
| `review_wait_threshold` | `4h` | レビュー待ち警告の閾値 |
| `stale_threshold` | `3d` | ステイル判定の閾値 |
| `auto_fix_level` | `L1` | 自動修正レベル |
| `notify_channel` | — | Discord 通知先チャンネル |
| `target_repos` | — | 監視対象リポジトリ一覧 |

## 段階的ロールアウト

### L1: Report-only（推奨スタート）
- PR の状態を定期レポートするのみ
- 人間が判断してアクションを取る

### L2: Assisted
- CI 失敗の原因分析と修正 PR の自動作成
- レビュー催促メッセージの自動送信
- マージ可能な PR の通知

### L3: Unattended
- lint / フォーマット修正の自動コミット
- Approved + CI 通過の PR を自動マージ（設定で有効化）
- ステイル PR の自動クローズ（設定で有効化）
