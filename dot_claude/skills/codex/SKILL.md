---
name: codex
description: |
  Codex CLI（OpenAI）を使用してコードの実装・レビュー・分析を行う。
  トリガー: "codex", "codexと相談", "codexに聞いて", "コードレビュー", "レビューして", "実装して", "codexに実装させて"
  使用場面: (1) コード実装（ファイル生成・変更）、(2) コードレビュー、(3) 設計の相談、(4) バグ調査、(5) 解消困難な問題の調査
---

# Codex

Codex CLI を使用してコードの実装・レビュー・分析を行うスキル。

## モード選択（必須）

タスクの種類に応じてサンドボックスモードを選ぶ：

| モード | フラグ | 使う場面 |
|---|---|---|
| **実装** | `--sandbox workspace-write` | ファイルを作成・変更する（実装、リファクタリング） |
| **分析** | `--sandbox read-only` | ファイルを読むだけ（レビュー、バグ調査、設計相談） |

## コマンド形式

### 実装モード（ファイルを書く）
```
codex exec --sandbox workspace-write --cd <project_dir> "<request>" < /dev/null
```
- `< /dev/null` は **必須**（stdin を閉じないと Codex がブロックする）

### 分析モード（読み取り専用）
```
codex exec --sandbox read-only --cd <project_dir> "<request>" < /dev/null
```

## プロンプトのルール

末尾に必ず含める：
> 「確認や質問は不要です。具体的な提案・修正案まで自主的に出力してください。」

### 実装プロンプトの書き方（重要）

**コードをプロンプトに含めてはいけない。** Codex はプロジェクトを自分で読める。

良い例（仕様ファイルを参照させる）:
```
services/edge-instance-console-gateway/ の Step 2 を実装してください。
仕様は services/edge-instance-console-gateway/README.md を参照してください。
確認や質問は不要です。具体的な実装まで自主的に出力してください。
```

悪い例（コードを丸ごと貼る）:
```
以下のコードを実装してください:
func Auth(...) { ... 全文 ... }  ← NG: 高トークン消費、Codex の自律実装力を無駄にする
```

## 実行手順

1. ユーザーの意図が「実装」か「分析」かを判定する
2. 対象プロジェクトのディレクトリを特定する
3. **実装**: 仕様ファイル（README, CLAUDE.md, プラン等）のパスを参照させる短いプロンプトを作る
4. **分析**: レビュー対象・観点を明記したプロンプトを作る
5. 対応するモードのコマンドを `< /dev/null` 付きで実行する
6. 結果をユーザーに報告する

## 使用例

### 実装（新機能）
```
codex exec --sandbox workspace-write --cd /path/to/project \
  "services/foo/ の Step 2 を実装してください。仕様は services/foo/README.md を参照してください。確認や質問は不要です。" \
  < /dev/null
```

### コードレビュー
```
codex exec --sandbox read-only --cd /path/to/project \
  "services/bar/ の最近の変更をレビューしてください。DDD 層分離・セキュリティ・エラーハンドリングの観点で確認してください。確認や質問は不要です。具体的な問題点と修正案まで出力してください。" \
  < /dev/null
```

### バグ調査（原因特定のみ）
```
codex exec --sandbox read-only --cd /path/to/project \
  "認証処理でエラーが発生する原因を調査してください。確認や質問は不要です。原因の特定と具体的な修正案まで出力してください。" \
  < /dev/null
```

### バグ修正（ファイル変更を伴う）
```
codex exec --sandbox workspace-write --cd /path/to/project \
  "services/foo/internal/usecase/bar.go のバグを修正してください。〈バグの説明を自然言語で〉。ファイルを読んで問題箇所を特定し修正してください。確認や質問は不要です。" \
  < /dev/null
```
- コードをプロンプトに貼らない。ファイルパスを示して Codex 自身に読ませる。
