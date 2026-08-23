# claude-ops-portfolio

Claude Code (Anthropic の CLI ツール) を自律運用するために個人で構築した、コストゲートとリスクゲート付きのハーネス（実行基盤）です。2026年4月から実運用中の環境から、再利用可能な設計部分だけを抜粋しています。

## アーキテクチャ

```
利用者
  │
  ▼
Claude Code（メインセッション）
  │
  ├─ 振り分け層  scripts/ask.py
  │    8社の直API（Anthropic / OpenAI / Google / xAI / DeepSeek / Moonshot / Zhipu 等）
  │    + ローカル Ollama を alias 一発で呼び分け
  │
  ├─ 委譲層      agents/*.md
  │    codex-implementer（実装をOpenAI直APIへ）
  │    external-researcher（速報/長文要約を外部APIへ）
  │    inventory-curator（定期棚卸しの自動集計）
  │
  ├─ 安全層      hooks/*.py（PreToolUse フック）
  │    cost-gate / guard / heredoc-gate / external-send-gate
  │    課金・破壊操作・機密送信をコマンド実行前に検知しゲート
  │
  └─ 記憶層（本リポジトリには含まない）
       point-in-time な memory ファイル群。個人の実データのため非公開
```

## 設計思想

- **自律デフォルト + 金とリスクだけ人間ゲート**: 読取・編集・テストは確認なしで進め、課金や破壊的操作だけ承認を求める。hooks/pretooluse-cost-gate.py の `permissionDecision: "ask"` が settings.json の自動実行設定より優先される点を利用している。
- **機密はローカルLLM強制**: 顧客データや個人情報を含むコマンドが外部APIへ送られようとした場合のみ承認を要求する AND 条件設計。hooks/pretooluse-external-send-gate.py。
- **破壊操作は hook と permissions の二重ブロック**: SQL の DROP / 条件無し DELETE、API キーの直書き、`git push -f` 等は正規表現ベースの hook が settings.json の permissions.deny より厳格に弾く。hooks/pretooluse-guard.py。
- **memory は point-in-time**: 記憶ファイルは常に「その時点のスナップショット」であり生きた現在状態ではない、という前提を CLAUDE.md.example の設計思想部分に明記し、断定の根拠にしないルールを敷いている。
- **モデルIDは陳腐化する**: 半年放置すると全プロバイダのモデルIDが1〜3世代古くなるため、棚卸し時に各社の `/v1/models` を実叩きして実在IDを確認する運用にしている。scripts/ask.py。

## 実インシデント起点の改修例

2026-07-09、`git commit -m "$(cat <<'EOF' ... EOF)"` のような複数行ヒアドキュメントが Claude Code セッション内のツール呼び出しのフレーミングを壊し、壊れた構文が再発する事故が発生した。原因を切り分けた結果、Bash ツール内のヒアドキュメントそのものが既知のトリガーだったため、恒久対策として hooks/pretooluse-heredoc-gate.py を新設。ヒアドキュメントを一律ブロックし、複数行データは「複数の `-m` フラグ」「Write ツールで一時ファイル化」「`printf` の行単位出力」に置き換える運用ルールへ落とし込んだ。事故 → 恒久ルール化 → 基盤（hooks）への反映、という改修サイクルの一例。

## セットアップ

環境変数（値は各自のキーを設定。名前のみ）:

- `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GEMINI_API_KEY` / `XAI_API_KEY` / `DEEPSEEK_API_KEY` / `MOONSHOT_API_KEY` / `ZAI_API_KEY`
- ローカル推論を使う場合は Ollama（`127.0.0.1:11434`）が別途必要

Windows 注意点:

- Ollama 等ローカルサーバーへの接続は `localhost` ではなく `127.0.0.1` を明示すること（`localhost` 解決で数秒のスコールが発生するケースがある）
- コンソール既定エンコーディングが cp932 のため、hook の標準エラー出力は ASCII のみに統一している（日本語は文字化けし、Claude が自己修正する際の読み取りに失敗する）

## 免責

このリポジトリには実運用データ（memory ファイル群、コスト台帳、settings.json の実値）は一切含まれません。パス・金額・アカウント情報等のサンプル値はすべてプレースホルダです。
