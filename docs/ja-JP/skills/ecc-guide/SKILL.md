---
name: ecc-guide
description: "「ECC のどの部分を X に使えばいいか？」という質問を、正確なサーフェス（skill、コマンド、agent、hook、ルール、MCP コネクタ、インストールプロファイル）へルーティングします。記憶ではなくリポジトリの実状態を読んで回答し、1 画面で答え・正典パス・検証コマンドを返します。TRIGGER: ECC に何が含まれるか、あるコンポーネントの場所、タスクに合うサーフェス、ECC のインストール/リセット/移行/アンインストール方法、hook やコネクタの挙動理由、コマンド・skills・agents・hooks・ルールの関係を尋ねられたとき。DO NOT TRIGGER: 作業そのものの実行を求められたとき（該当コンポーネントを呼ぶ）、実行順と停止条件を伴う複数コマンドのパイプラインが必要なとき（ecc-recipes）、対話型インストールウィザードが必要なとき（configure-ecc）。"
argument-hint: "<トピック | find: クエリ | コンポーネント名 | 空=メニュー>"
origin: community
metadata:
  version: "2.0.0"
  surface-baseline: "2027"
---

# ECC Guide

Everything Claude Code のナビゲーション層。曖昧な意図を、ひとつの具体的なサーフェス、
その正典ファイル、そして答えを裏づけるコマンドへ変換します。

**契約:** 助言と読み取りのみ。本スキルはサーフェスの特定と説明を担当し、
インストールも設定変更も行わず、名指ししたサーフェスを代わりに実行しません。

## 使用するとき

- 「ECC には何が入っている？」「ECC で X をやるには？」
- skill、コマンド、agent、hook、ルール、MCP コネクタ、インストールプロファイルの検索
- 機能が重なって見える 2 つのサーフェスの選択
- インストールパス、スコープ、重複インストール、リセット、アンインストールの理解
- コマンド・skills・agents・hooks・ルール・MCP の関係の説明
- 「ECC は入っているのに X が出てこない」の切り分け

### 使用しないとき

| 状況 | 委ねる先 |
|---|---|
| 今すぐ作業を実行してほしい | 該当する skill／コマンド |
| 実行順と停止条件つきの複数コマンドパイプラインが欲しい | `ecc-recipes` |
| 対話的にインストール・再設定・スコープ移行したい | `configure-ecc` |
| 下書きプロンプトを書き直してほしい | `prompt-optimizer` |
| ECC サーフェスのトークン／コストを集計したい | `context-budget`、`ecc-tools-cost-audit` |

## 第一原則

**記憶ではなく現在のファイルから答える。** ECC のカタログは毎週動きます。
ハードコードした件数・機能一覧・インストールフラグは、いずれ必ず誤答になります。

そこから導かれる 3 つのルール:

1. 今読んだばかりでない件数を口にしない。
2. ファイルシステムを確認せずにコンポーネントの存在を主張しない。
3. チェックアウトが利用できない場合はそれを明言し、名前を推測せず構造で答える
   （「skill は `skills/<name>/SKILL.md` にあります」）。

## 読み取り予算の段階

質問が要求する段階までしか上げません。ほとんどの質問は T1 で止まります。

| 段階 | 質問の形 | 読み取り | 目標コスト |
|---|---|---|---|
| **T0** | 概念— 「skill とコマンドの違いは？」「hook プロファイルとは？」 | なし | 約 0 トークン |
| **T1** | 存在・場所— 「Rust の reviewer はある？」 | `find` か `rg` を 1 回 | 1k トークン未満 |
| **T2** | 選択— 「うちのリポジトリにはどれが合う？」 | 候補 2〜4 件の frontmatter | 4k トークン未満 |
| **T3** | 全数調査— 「X を全部出して」（明示的に求められたときのみ） | `catalog.js --json` | 10k トークン以上 |

段階順の低コストな探索:

```bash
# T1 — 存在と場所（最速・依存なし）
ls skills/<name>/SKILL.md commands/<name>.md agents/<name>.md 2>/dev/null
rg -l "<query>" skills commands agents rules docs --max-count 1

# T2 — 全文を読まずに候補を絞る
head -6 skills/<name>/SKILL.md            # frontmatter のみ
rg -n "^description:" skills/*/SKILL.md | rg -i "<query>"

# T3 — 実時点の全カタログ（「全部出して」と明示されたときのみ）
node scripts/ci/catalog.js --json
node scripts/install-plan.js --list-profiles
node scripts/install-plan.js --list-components --json
```

効率ルール: 本文より先に frontmatter を読む。読んだ内容はセッション中キャッシュする。
独立した探索は 1 回の呼び出しにまとめる。同一セッションで `catalog.js` を 2 回実行しない。

## インテント・ルータ

何かを読む前に、ユーザーの言葉をサーフェス種別へ対応づけます。

| ユーザーの言い方 | サーフェス種別 | 解決先 | 検証 |
|---|---|---|---|
| 「ワークフロー／手順／X のやり方」 | skill | `skills/<name>/SKILL.md` | `ls skills/<name>/` |
| 「スラッシュコマンド／`/x`」 | コマンド | `commands/<name>.md` | `ls commands/` |
| 「委譲／サブエージェント／並列」 | agent | `agents/<name>.md` | `ls agents/` |
| 「自動で／編集のたびに／止めてほしい」 | hook | `hooks/hooks.json`、`scripts/hooks/` | `cat hooks/README.md` |
| 「常に従う／規約／ポリシー」 | ルール | `rules/` | `ls rules/` |
| 「外部システムに接続したい」 | MCP または skill | `mcp-configs/mcp-servers.json`、`docs/MCP-CONNECTOR-POLICY.md` | MCP 予算を参照 |
| 「インストール／プロファイル／スコープ／アンインストール」 | インストーラ | `manifests/install-*.json`、`README.md` | `node scripts/install-plan.js --list-profiles` |
| 「このリポジトリ用に ECC を設定して」 | オンボーディング | `/project-init` | dry-run プラン |
| 「今の構成は健全／安全か」 | 監査 | `/harness-audit`、`/security-scan` | `npm run harness:audit -- --format text` |

**2 つのサーフェスが該当する場合の優先順位:** skill > agent > コマンド > hook。
skill が主たるワークフロー面、コマンドは保守された互換入口、agent は別コンテキスト
ウィンドウで走らせるべき作業、hook は頼まれなくても発火する必要がある挙動のみ。

## サーフェスモデル

層の違いで混乱しているユーザーには、各 1 文で説明します。

- **Skill** — 関連するときにだけ読み込まれるワークフロー。段階的開示により、
  起動時のみコンテキストを消費します。
- **コマンド** — ユーザーが明示的に打つ入口。呼び出しは決定的で、中身の性質は skill と同種。
- **Agent** — *別の*コンテキストウィンドウで実行する委譲作業。広域検索、独立レビュー、
  メインスレッドを埋め尽くす作業向け。
- **Hook** — ライフサイクルイベント上の決定的な自動化。モデルの協力に関係なく走る、
  強制の層です。
- **ルール** — 常時読み込まれる指針。1 行あたりのコンテキスト税が最も高く、短いほど強い。
- **MCP コネクタ** — セッション状態を持つ外部システム。ツールスキーマは使う・使わないに
  関わらず*すべての*セッションに読み込まれます。

## MCP コネクタ予算（2027 の姿勢）

正典ポリシー: `docs/MCP-CONNECTOR-POLICY.md`。要点:

デフォルト枠を得るには、**普遍的**であり、*かつ* MCP でなければ得られないもの
——保持されたセッション状態、ストリーミング、認証ハンドシェイク、構造化ブラウジング
——を本当に必要とすること。ステートレスな要求／応答は、CLI や REST API を包む skill
であってサーバーではありません。

2027 年時点でも、主要ハーネスの実質的デフォルトは**デフォルトコネクタ 0〜2 個と
ネイティブ組み込み**のままです。ECC が同梱するのは 1 つ（`chrome-devtools`）。
6 つは 2026 年 6 月の監査で skill へ置き換えられ、`mcp-configs/mcp-servers.json` に
オプトインとして残っています。その判断は 2027 年も有効です——ハーネス側のネイティブ
検索・記憶・思考が、これらのサーバーの存在理由をさらに吸収したためです。

新しいコネクタを求められたら、この順で問います:

1. 既存の CLI や REST API でできるか？ → できるなら skill であってサーバーではない。
2. 価値は*保持されるセッション*か、単発呼び出しか？ → 単発なら skill。
3. キーが必要か？ → 普遍性を満たさない。オプトインのみ、デフォルトにはしない。
4. 一度も呼ばないセッションにかかるスキーマ税はどれだけか？

同梱コネクタの無効化は `ECC_DISABLED_MCPS="chrome-devtools"`。

## インストール指針

必ず plan → dry-run → apply の順に。マネージドインストーラが対象をサポートしている
場合、コンポーネントファイルを手でコピーしないこと。

```bash
node scripts/install-plan.js --list-profiles
node scripts/install-plan.js --profile minimal --target claude --json
node scripts/install-apply.js --profile minimal --target claude --dry-run

# プロファイルではなく単一 skill を入れる場合
node scripts/install-plan.js --skills <skill-id> --target claude --json
```

押さえるべきフラグ: `--modules`、`--with`、`--without`、`--family`、`--config`、
`--target`。対象は Claude Code、Codex、Cursor、OpenCode、Kimi、Gemini、CodeBuddy、
JoyCode、Qwen をカバーします。対応状況は記憶で断定せず `--list-components --json` で確認を。

**重複サーフェスの警告:** プラグイン導入と、手動またはプロファイル導入を*重ねる*と、
全コンポーネントが二重になります。意図を先に確認してください。

## 切り分け

この順にトリアージし、症状を説明できた最初の層で止めます。

| 症状 | 最初に見る場所 |
|---|---|
| インストール後にコンポーネントが無い | インストールスコープと対象ディレクトリ — `.claude/`、`.codex/`、`.cursor/`、`.opencode/`、`.gemini/`、`.kimi-code/`、`.codebuddy/`、`.joycode/`、`.qwen/` |
| コンポーネントが二重に見える | プラグイン導入に手動／プロファイル導入が重なっている |
| hook が発火しない | `hooks/hooks.json` の matcher、次に `ECC_HOOK_PROFILE` / `ECC_DISABLED_HOOKS` |
| hook が発火しすぎる | hook プロファイルが `strict`。`standard` か `minimal` へ下げる |
| スクリプトが `Cannot find module` | 依存が未導入 — `npm ci` |
| コネクタのツールが見えない | `ECC_DISABLED_MCPS`、次にハーネス側の MCP 設定 |
| セッションが遅い／コンテキストが足りない | `context-budget`。まずルールとコネクタを削る |

リポジトリの健全性チェック（コストの低い順）:

```bash
npm run harness:audit -- --format text
npm run observability:ready
npm test
```

## 回答フォーマット

まず答えから。1 画面で収め、その後に深掘りを提示します。

```text
<サーフェス> を使ってください。理由: <ひとつ>。
正典ファイル: <path>
検証:         <command>
次の一手:     <具体的な 1 アクション>
```

検索の場合:

```text
最有力の一致:
- <path> — <なぜ重要か>
- <path> — <なぜ重要か>
まずはこれから: <ひとつ>（理由: <理由>）
```

インストールの場合:

```text
検出:     <スタックの根拠>
対象:     <harness>  スコープ: <user|project|local>
プラン:   <profile/modules/skills>
Dry run:  <command>
変更範囲: <paths>
適用前の承認要否: <yes/no>
```

## アンチパターン

- 1 つのパスを聞かれてカタログ全体を吐き出す
- 件数・バージョン・プロファイル名を記憶から引用する
- skill 優先の経路があるのに、退役したコマンド入口を勧める
- インストーラが対象を扱えるのに手動 `cp` を指示する
- T1 の質問に T3 で応じる
- 1 種類だけ聞かれたのにサーフェス 6 種すべてを説明する
- 4 つの予算質問を通さずに新しい MCP コネクタを勧める

## 関連サーフェス

| 目的 | サーフェス |
|---|---|
| 実行順と停止条件つきのコマンドパイプライン | `ecc-recipes` |
| 対話的なインストール／再設定／スコープ移行 | `configure-ecc` |
| 対象リポジトリのスタックを踏まえたオンボーディング | `/project-init` |
| 決定的な準備状況スコアカード | `/harness-audit` |
| skill の品質レビュー | `/skill-health` |
| ローカル git 履歴から skill を生成 | `/skill-create` |
| 設定のセキュリティレビュー | `/security-scan` |
| トークン／コスト集計 | `context-budget`、`ecc-tools-cost-audit` |
