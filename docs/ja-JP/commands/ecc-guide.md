---
description: ECC のライブサーフェス（スキル、コマンド、エージェント、フック、ルール、MCP コネクタ、インストールプロファイル）をナビゲートし、正典パスと検証コマンドを返します。
---

# /ecc-guide

Everything Claude Code の会話型マップ。ルーティング、読み取り予算、MCP 予算の
ロジックは `ecc-guide` スキルが持ちます。`skills/ecc-guide/SKILL.md` を読んで
それに従ってください。本ファイルは入口だけを定義します。

## 使い方

```text
/ecc-guide                     # コンパクトなメニュー
/ecc-guide setup | install     # インストールパス、プロファイル、スコープ
/ecc-guide skills | commands | agents | hooks | rules | mcp
/ecc-guide find: <クエリ>       # 全サーフェスを横断検索
/ecc-guide <機能名またはファイル名>  # 単一コンポーネントの特定
```

## 動作ルール

1. 記憶ではなく現在のファイルから答える。件数や機能一覧をハードコードしない。
2. 質問が要求する最小の読み取り段階に留まる（T0 概念 → T3 全カタログ）。
3. 答えから始める: サーフェス、正典パス、検証コマンド、次の 1 アクション。
4. 存在しないコンポーネントを作らない。主張の前にファイルシステムを確認する。
5. 助言のみ。特定と説明を行い、インストールや実行はしない。

## 参照先

| サーフェス | 正典の場所 |
|---|---|
| スキル | `skills/*/SKILL.md` |
| コマンド | `commands/*.md` |
| エージェント | `agents/*.md` |
| フック | `hooks/hooks.json`、`hooks/README.md`、`scripts/hooks/` |
| ルール | `rules/` |
| MCP コネクタ | `mcp-configs/mcp-servers.json`、`docs/MCP-CONNECTOR-POLICY.md` |
| インストールプロファイル | `manifests/install-*.json`、`README.md` |
| ライブカタログ | `node scripts/ci/catalog.js --json` |

## モード

- **引数なし** — コンパクトなメニュー: インストール、スキル選択、コマンドとスキルの違い、
  エージェントと委譲、フックと安全性、MCP 予算、トラブルシューティング。その後、
  次に何をしたいか尋ねる。
- **トピック** — 現在のサーフェスを 3〜6 の箇条書きで要約し、正典ディレクトリと
  検証コマンドを 1 つ示す。要求がない限り網羅列挙はしない。
- **`find: <クエリ>`** — スキル、コマンド、エージェント、ルール、ドキュメントを `rg` で検索。
  サーフェスごとにまとめ、最有力から順に、それぞれ次のアクションを添える。
- **機能名** — まず正確なパス（`skills/<n>/SKILL.md`、`commands/<n>.md`、`agents/<n>.md`）、
  次に `rg`。何をするか、いつ使うか、どのファイルが正典かを説明する。

## 関連

`/project-init` · `/harness-audit` · `/skill-health` · `/skill-create` ·
`/security-scan` · `ecc-recipes`（パイプライン）· `configure-ecc`（インストールウィザード）
