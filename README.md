# claude-ops-portfolio

Claude Code（Anthropic の CLI（コマンドライン）ツール）を自律運用するために個人で構築した、コストゲートとリスクゲート付きのハーネス（実行基盤）です。
製造業の現場で働きながら、業務外で EC（電子商取引）代行業務向けの受注・顧客・決済管理基盤（CRM＋通知）を安全に AI へ実装委託するために作りました。
2026 年 4 月から現在まで、日常の開発・運用作業のすべてをこの基盤の上で回しています。
コードは自分で書きません。実装は AI（Claude Code）に委譲し、要件定義・アーキテクチャ設計・コードレビュー・動作検証・本番運用を担当しています。
本リポジトリには、実運用中の環境から再利用可能な設計部分だけを抜粋しています（実データは含みません）。

## テクニカルサポート / カスタマーサクセスの仕事にどう繋がるか

- **AI が生成したコードの品質と安全を担保するゲートを自分で設計・運用している**（`hooks/pretooluse-guard.py` / `pretooluse-cost-gate.py` / `pretooluse-external-send-gate.py`）。「AI の出力を鵜呑みにせず検証してから通す」という姿勢は、問い合わせ対応で AI ツールの回答を検証してユーザーに返す仕事とそのまま重なります。
- **実インシデントを 原因特定 → 恒久ルール化 → hook 化 という順で改修したサイクルがある**（後述、2026-07-09 の事故）。障害の報告を受けて原因を切り分け、再発防止策として仕組みに落とし込むプロセスは、障害切り分けとエスカレーション業務の基本と同じです。
- **8 社のベンダー API を一元管理し、仕様変更に追従し続けている**（`scripts/ask.py`、1,228 行）。モデル ID は半年放置すると 1〜3 世代古くなることを実測し、棚卸し時に各社の API（`/v1/models`）を実際に叩いて実在するモデル ID を確認・追従する運用にしています。複数ベンダーの API 変更に追従し続けるメンテナンス習慣は、SaaS 製品のサポート業務にそのまま活きます。
- **AI サブエージェントに「ゴール・制約・検証方法・報告フォーマット」を明示した指示書を書いている**（`agents/*.md`）。相手（AI でも人でも）に伝わる形で仕様や手順を文書化する力は、マニュアル作成やエスカレーション先への引き継ぎに直結します。
- **「何を自分で判断し、何を人に確認すべきか」の線引きを自分で設計した**（`CLAUDE.md.example` の「自律デフォルト＋金とリスクだけ人間ゲート」）。通常操作は確認なしで進め、課金や破壊的操作だけ人間の承認を挟む、という基準そのものが、問い合わせ対応でのエスカレーション基準の設計と同じ思考です。

## アーキテクチャ

```
利用者
  │
  ▼
Claude Code（メインセッション）
  │
  ├─ 振り分け層  scripts/ask.py（1,228 行）
  │    8社の直API（Anthropic / OpenAI / Google / xAI / DeepSeek / Moonshot / Zhipu 等）
  │    + ローカル Ollama を alias 一発で呼び分け
  │
  ├─ 委譲層      agents/*.md + skills/*
  │    codex-implementer（実装をOpenAI直APIへ）
  │    external-researcher（速報/長文要約を外部APIへ）
  │    inventory-curator（定期棚卸しの自動集計）
  │    skills/ui-flow・skills/video-b（定型作業の手順をスキル化）
  │
  ├─ 安全層      hooks/*.py（6本、PreToolUse / SessionEnd フック）
  │    cost-gate / guard / heredoc-gate / external-send-gate
  │    課金・破壊操作・機密送信をコマンド実行前に検知しゲート
  │
  └─ 記憶層（本リポジトリには含まない）
       point-in-time な memory ファイル群。個人の実データのため非公開
```

## 5 分でわかる各層

- **振り分け層**（`scripts/ask.py`）: 用途と機密度に応じて、中間業者を挟まずに 8 社の直 API（Anthropic / OpenAI / Google / xAI / DeepSeek / Moonshot / Zhipu 等）とローカル Ollama を alias ひとつで呼び分ける。モデル ID は陳腐化するため、各社 `/v1/models` を実叩きして追従する運用も併せ持つ。
- **委譲層**（`agents/*.md`、`skills/*`）: 各サブエージェントに役割・ツール権限・エラー時フォールバックを明示した Markdown 指示書を持たせる。実装は codex-implementer 経由で OpenAI 直 API へ、速報調査や長文要約は external-researcher 経由で用途別に外部委譲する。
- **安全層**（`hooks/*.py`）: `pretooluse-guard.py` が破壊的操作・シークレット直書きを正規表現で検知し exit 2 でブロック、`pretooluse-cost-gate.py` が課金操作の前だけ人間承認を挟み、`pretooluse-external-send-gate.py` が「機密データ × 外部送信」の AND 条件のときだけ確認を挟む。
- **記憶層**（本リポジトリには含まない）: point-in-time な memory ファイル群。`CLAUDE.md.example` に「記憶は書いた時点のスナップショットであり現在の状態ではない」という前提を明記し、断定の根拠にしないルールを敷いている。

## 実インシデント起点の改修例

2026-07-09、`git commit -m "$(cat <<'EOF' ... EOF)"` のような複数行ヒアドキュメントが Claude Code セッション内のツール呼び出しのフレーミングを壊し、壊れた構文が再発する事故が発生した。原因を切り分けた結果、Bash ツール内のヒアドキュメントそのものが既知のトリガーだったため、恒久対策として hooks/pretooluse-heredoc-gate.py を新設。ヒアドキュメントを一律ブロックし、複数行データは「複数の `-m` フラグ」「Write ツールで一時ファイル化」「`printf` の行単位出力」に置き換える運用ルールへ落とし込んだ。事故 → 恒久ルール化 → 基盤（hooks）への反映、という改修サイクルの一例。

## 設計思想

- **自律デフォルト + 金とリスクだけ人間ゲート**: 読取・編集・テストは確認なしで進め、課金や破壊的操作だけ承認を求める。hooks/pretooluse-cost-gate.py の `permissionDecision: "ask"` が settings.json の自動実行設定より優先される点を利用している。
- **機密はローカルLLM強制**: 顧客データや個人情報を含むコマンドが外部APIへ送られようとした場合のみ承認を要求する AND 条件設計。hooks/pretooluse-external-send-gate.py。
- **破壊操作は hook と permissions の二重ブロック**: SQL の DROP / 条件無し DELETE、API キーの直書き、`git push -f` 等は正規表現ベースの hook が settings.json の permissions.deny より厳格に弾く。hooks/pretooluse-guard.py。
- **memory は point-in-time**: 記憶ファイルは常に「その時点のスナップショット」であり生きた現在状態ではない、という前提を CLAUDE.md.example の設計思想部分に明記し、断定の根拠にしないルールを敷いている。
- **モデルIDは陳腐化する**: 半年放置すると全プロバイダのモデルIDが1〜3世代古くなるため、棚卸し時に各社の `/v1/models` を実叩きして実在IDを確認する運用にしている。scripts/ask.py。

## セットアップ

環境変数（値は各自のキーを設定。名前のみ）:

- `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GEMINI_API_KEY` / `XAI_API_KEY` / `DEEPSEEK_API_KEY` / `MOONSHOT_API_KEY` / `ZAI_API_KEY`
- ローカル推論を使う場合は Ollama（`127.0.0.1:11434`）が別途必要

Windows 注意点:

- Ollama 等ローカルサーバーへの接続は `localhost` ではなく `127.0.0.1` を明示すること（`localhost` 解決で数秒のスコールが発生するケースがある）
- コンソール既定エンコーディングが cp932 のため、hook の標準エラー出力は ASCII のみに統一している（日本語は文字化けし、Claude が自己修正する際の読み取りに失敗する）

## 免責

このリポジトリには実運用データ（memory ファイル群、コスト台帳、settings.json の実値）は一切含まれません。パス・金額・アカウント情報等のサンプル値はすべてプレースホルダです。

## 作者について

製造業の現場で機械オペレーターとして働きながら、業務外でこの基盤を構築・運用しています。
IT 業界の実務経験はありません。
