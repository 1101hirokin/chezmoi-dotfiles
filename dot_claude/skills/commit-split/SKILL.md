---
name: commit-split
description: |
  現在の未コミット変更（staged / unstaged）を論理的な単位に分割してコミットする。
  複数の関心事が混在した変更を整理し、レビューしやすい単位でコミット履歴を構築する。
  トリガー: "/commit-split", "変更をコミットして", "コミットに分割して", "コミットを整理して"
when_to_use: |
  変更が複数の論理的関心事を跨いでいるとき。新機能・バグ修正・ドキュメント・リファクタリングが
  混在している場合。大量のファイルを有意な単位でコミット履歴として残したいとき。
argument-hint: "[--dry-run | --auto]"
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *) Bash(git diff *) Bash(git log *)
---

# Commit Split

未コミット変更を論理的な単位でコミットに分割する。

## 引数

- `--dry-run`: コミット計画を提示するのみ。実際のコミットは行わない
- `--auto`: 計画の確認をスキップして即実行する

引数なしの場合: 計画を提示し、ユーザーの承認後に実行する。

## 現在の変更

### Git Status
!`git status --short`

### Staged diff
!`git diff --staged --stat`

### Unstaged diff (全ファイル)
!`git diff --stat HEAD`

## 手順

### Step 1: 変更の分析

上記の diff から変更を以下のカテゴリに分類する:

| カテゴリ | Conventional Commits type | 例 |
|---|---|---|
| 新機能・新サービス | `feat` | 新しいエンドポイント、新サービス追加 |
| バグ修正・セキュリティ修正 | `fix` | エラー正規化、脆弱性対応 |
| リファクタリング | `refactor` | インターフェース変更、コード整理 |
| テスト追加・修正 | `test` | 単体テスト、統合テスト |
| DB スキーマ・マイグレーション | `fix` or `feat` | スキーマ変更、インデックス追加 |
| ドキュメント・issue ファイル | `docs` | issues/, README.md, CLAUDE.md |
| 設定・ビルド・依存関係 | `chore` | go.work, go.mod, Makefile |
| OpenAPI / proto | `feat` or `fix` | spec 変更 |

### Step 2: コミット計画の構築

分類した変更をコミット単位にグループ化する。グループ化の原則:
- **1 コミット = 1 つの論理的変更**: 異なる関心事を混在させない
- **実装 + 直接対応テスト = 同一コミット**: バグ修正やfeatureに直接対応するテストは実装と同一コミットに含める。独立したテストスイート追加は別コミット
- **順序**: 後のコミットが前のコミットに依存する順序で並べる（例: 実装 → ドキュメント）
- **サービス境界を尊重**: 異なるサービスの変更は原則別コミット
- **セキュリティ修正は個別に**: セキュリティ修正は審査しやすいよう単独コミットにする
- **ビルド依存の変更は同一コミットに**: 関数シグネチャ変更と全呼び出し箇所の更新は同一コミット（中間状態がビルドエラーになるため）

コミットメッセージ形式:
```
<type>(<scope>): <short description>

<optional body: why, not what>
```

例:
```
fix(app-instance-auth-api): normalize internal error messages to prevent leakage
feat(edge-instance-console-gateway): implement HTTP/JSON BFF with auth middleware
docs(issues): record security audit findings and resolution status
```

### Step 3: 計画の提示

以下の形式でコミット計画を提示する:

```
COMMIT PLAN (N commits):

[1] fix(scope): message
    Files:
      - path/to/file.go
      - path/to/other.go

[2] feat(scope): message
    Files:
      - ...

TOTAL: X files staged across N commits
```

`--dry-run` の場合はここで終了する。

### Step 4: 確認 (--auto でない場合)

計画を提示した後、ユーザーに確認を求める:
```
上記の計画でコミットしますか？ (y/n/edit)
```

ユーザーが `n` を選んだ場合は中止する。`edit` の場合はグループ化を調整する。

### Step 5: コミット実行

各コミットを順番に作成する:

```bash
# ファイルを名前で指定してステージ
git add <file1> <file2> ...

# ステージ内容を確認（意図しないファイルが含まれていないか）
git diff --staged --stat

# コミット作成（HEREDOC 形式）
git commit -m "$(cat <<'EOF'
<type>(<scope>): <description>
EOF
)"
```

### Step 6: 完了サマリー

```
COMMITTED (N commits):

abc1234 fix(app-instance-auth-api): normalize internal error messages
def5678 feat(edge-instance-console-gateway): implement HTTP/JSON BFF
ghi9012 docs(issues): record security audit findings

Use `git log --oneline -N` to verify.
```

## 制約（絶対に守ること）

- `git add -A` や `git add .` は使わない。原則ファイルを **名前で指定**する
  - 例外: 新規追加ディレクトリ配下が全て新規ファイルかつ単一の論理変更の場合のみ `git add <dir>/` 可
- `--no-verify` は絶対に使わない
- コミットメッセージは **英語** で書く（Conventional Commits は英語が標準）
- secrets（`.env`、credentials 含むファイル）を検出したら **警告して除外**し、コミットしない
- コミットメッセージは Conventional Commits 形式に従う
- コミット後に `git diff --staged` が空であることを確認してから次に進む
- 1 つのコミットが失敗したら即座に停止してユーザーに報告する
