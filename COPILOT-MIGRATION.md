# GitHub Copilot Agent → gstack 移行ガイド

対象: C# WinForms 装置制御システム開発者

---

## 概要

| | GitHub Copilot Agent | gstack (Claude Code) |
|---|---|---|
| 動作場所 | VS Code 内インライン / GitHub PR | ターミナル / Claude Code CLI |
| コンテキスト | 開いているファイル・ワークスペース | git リポジトリ全体 + カスタムチェックリスト |
| 装置制御ドメイン知識 | なし（汎用） | あり（カスタマイズ済み） |
| 自律的ワークフロー | Agent モードで一部対応 | スキル単位で完全自動化 |

---

## コマンド対応表

### VS Code Copilot Chat スラッシュコマンド

| Copilot Agent | gstack | 差分・注意点 |
|---|---|---|
| `/fix` | `/review` | gstack は**装置制御固有のチェック**（インターロック・スレッド安全性・リソース管理）を実行した上で自動修正する。Copilot は汎用的な修正のみ。 |
| `/tests` | `/plan-eng-review` のテストレビューセクション | gstack はハードウェアありのテストとなしのテストを分けてテスト計画を生成する。 |
| `/doc` | `/document-release` | gstack は git diff から差分を読み取り README・ARCHITECTURE・CHANGELOG を一括更新する。 |
| `/explain` | Claude Code ネイティブチャット | gstack スキルではなく Claude Code の通常の質問として聞けば良い。 |
| `/workspace` コンテキスト付き質問 | Claude Code ネイティブチャット | `@workspace` に相当するリポジトリ全体の読み込みは Claude Code が自動で行う。 |

### GitHub Copilot for Pull Requests

| Copilot Agent | gstack | 差分・注意点 |
|---|---|---|
| PR 説明の自動生成 | `/ship`（Step 8） | gstack は CHANGELOG・テスト結果・プレランディングレビュー結果をすべて PR 本文に自動挿入する。Copilot は diff の要約のみ。 |
| PR コードレビューコメント | `/review` | gstack はハードウェア安全性・スレッド安全性カテゴリを含む装置制御固有のレビューを行い、修正可能なものは自動で直す。 |
| PR サマリー生成 | `/ship` 実行 → PR 作成まで自動 | gstack は「テスト実行 → レビュー → バージョンバンプ → CHANGELOG → PR 作成」を一括で行う。 |

### GitHub Copilot Agent モード（自律タスク）

| Copilot Agent モード | gstack | 差分・注意点 |
|---|---|---|
| 「このバグを直して PR を作って」 | `/ship` | gstack は**テスト失敗時に止まる**・**インターロックバイパスを検出したら ASK する**など装置制御に適したガードが入っている。 |
| 「このコードをリファクタして」 | Claude Code ネイティブ実行 + `/review` で事後確認 | gstack スキルには単体リファクタのスキルはない。実行後に `/review` でハードウェア安全性チェックを行う。 |
| 「テストを書いて」 | Claude Code ネイティブ実行 + `/plan-eng-review` | テスト設計レビューは `/plan-eng-review` の Section 3 で行える。 |

---

## ワークフロー対応表

### 日常開発フロー

```
Copilot Agent での従来フロー:
  コード書く → Copilot /fix でその場修正 → PR 作成 → Copilot が PR 説明生成

gstack での推奨フロー:
  コード書く → /review で装置制御チェック + 自動修正 → /ship で PR まで一括
```

### 設計・計画フロー

```
Copilot Agent での従来フロー:
  設計をチャットで相談 → コーディング開始

gstack での推奨フロー:
  設計をチャットでまとめる
    → /plan-eng-review で「テスト計画・失敗モード・パフォーマンス」をレビュー
    → /plan-ceo-review でスコープ・アーキテクチャ・デプロイ計画をレビュー（大きな変更時）
    → コーディング開始
```

### コードレビューフロー

```
Copilot Agent での従来フロー:
  PR に Copilot レビューコメントが付く → 手動で対応

gstack での推奨フロー:
  /review を実行
    → AUTO-FIX: マジックナンバー定数化・using ブロック追加・InvokeRequired 追加 など
    → ASK: インターロック変更・ソフトリミット変更・バルブシーケンス変更 など
    → 承認後 /ship で PR へ
```

### ふりかえりフロー

```
Copilot Agent: 該当機能なし

gstack:
  /retro        → 週次コミット分析・テスト比率・セッションパターン・チームハイライト
  /retro 14d    → 2週間分析
  /retro compare → 前週比較
```

---

## gstack にしかない機能

Copilot Agent では対応できないが、gstack（今回の装置制御カスタマイズ済み）で使える機能:

| 機能 | スキル | 説明 |
|---|---|---|
| **装置制御固有レビュー** | `/review` | インターロックバイパス・バルブシーケンス違反・非常停止の安全状態復帰漏れを検出 |
| **C# スレッド安全性チェック** | `/review` | `InvokeRequired` 欠落・`lock` なしデバイスアクセス・`CancellationToken` 未対応を自動修正 |
| **WinForms UI 設計監査** | `/plan-design-review` `/qa-design-review` | HiDPI スケーリング・Anchor/Dock・タブ順・アクセシビリティ・Windows UX ガイドライン適合 |
| **失敗モードレジストリ** | `/plan-ceo-review` `/plan-eng-review` | 全 codepath × 例外クラス × 捕捉状態 × オペレータへの表示 のテーブルを強制生成 |
| **週次エンジニアリング振り返り** | `/retro` | コミット分析・テスト比率トレンド・セッションパターン・チームハイライト |
| **ドキュメント自動更新** | `/document-release` | README・ARCHITECTURE・CONTRIBUTING を diff から自動更新。.NET SDK バージョンと装置設定手順の確認も含む |

---

## Copilot Agent にしかない機能

gstack では対応していないが Copilot Agent が得意なこと:

| 機能 | Copilot | gstack での代替 |
|---|---|---|
| **インライン補完** | ✅ タイプ中にリアルタイム提案 | なし（Claude Code はチャット型） |
| **VS Code 統合 UI** | ✅ エディタ内でそのまま使える | Claude Code はターミナル起動が必要 |
| **GitHub Actions / CI 連携** | ✅ CI 失敗の原因を PR 上で説明 | なし |
| **特定行へのコメント修正提案** | ✅ PR の特定行に差分付きで提案 | `/review` は全体レビュー |

---

## 推奨ハイブリッド運用

```
コーディング中:
  Copilot Agent のインライン補完を使う（タイピング補助として最適）

コミット前:
  /review を実行（装置制御固有チェック + 自動修正）

PR 作成時:
  /ship を実行（テスト → レビュー → バージョン → CHANGELOG → PR まで一括）

設計時（大きな変更）:
  /plan-eng-review または /plan-ceo-review でレビュー

UI 変更時:
  /qa-design-review で HiDPI・アクセシビリティ・Windows UX 確認

週次:
  /retro でテスト比率・コミットパターン・チームハイライトを確認
```

---

## セットアップ確認

gstack が使える状態か確認:

```bash
# Claude Code のインストール確認
claude --version

# gstack スキルの確認
ls ~/.claude/skills/gstack/

# 現在のスキルバージョン確認
cat ~/.claude/skills/gstack/package.json | grep version
```

初回セットアップ（未インストールの場合）:
```bash
cd ~/.claude/skills/gstack && bun install && bun run build
```
