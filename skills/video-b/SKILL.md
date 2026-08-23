---
name: video-b
description: YouTube URL・動画ファイル(.mp4/.mkv等)・「この動画なに？」系の話題が出たら必ず読み込むスキル。動画B処理(映像OCR+Claude整理)の即応提案フォーマット、ask_video.py のコマンド・コスト・制約を提供。動画の要約・画面内コード抽出・チャプター指定解析・字幕処理の依頼全般で発火。
---

# 動画B処理パイプライン

ユーザーは技術詳細を覚えていない前提。動画 URL / ファイル / 動画話題が出たら、まず yt-dlp でタイトル・長さを取得して即このフォーマットで提案する:

```
タイトル: <yt-dlp で取得>  長さ: <duration>
B処理しますか？
- A: 字幕→Gemini要約（30秒、安い、喋り中心向け）
- B: 映像OCR + Claude整理（数分、コード・設定画面が取れる）
- C: 特定章だけB（時刻指定）
```

## コマンド（`~/.claude/scripts/ask_video.py`）

```bash
# YouTube 指定範囲を OCR モード（要約禁止、画面文字を生で吐く → 後段で Claude が整理）
python ~/.claude/scripts/ask_video.py "https://www.youtube.com/watch?v=XXX" \
  --start 5:51:04 --end 5:58:09 \
  --mode ocr --quality 720 \
  --out result.md

# 長尺は自動分割（5分刻み x 並列1で安定）
python ~/.claude/scripts/ask_video.py URL --start 2:27:29 --end 2:58:27 \
  --mode ocr --chunk-minutes 5 --quality 720 --workers 1 \
  --out result.md

# ローカルファイル
python ~/.claude/scripts/ask_video.py /path/to/clip.mp4 --prompt "..."
```

- `--mode summary`（既定）: 日本語要約（A方式）
- `--mode ocr`: 画面文字を一字一句そのまま抽出（B方式、Claude が後整理）

## コスト・制約（重要）

- **Gemini 2.5 Flash 無料枠 = 20 req/日**（2026-05 実測、[[reference_gemini_25_free_tier_quota]]）。多用日は枯渇するので計画的に
- 動画は **10分以内**で品質ピーク、30分超は要約モード化する
- `--workers 2+` は yt-dlp 衝突 + RPM 超過で失敗率上昇 → **`--workers 1` 推奨**
- 13時間超の動画は Gemini が拒否（最大2-6h）→ クリップ DL 必須
- 内部フロー: yt-dlp+ffmpeg 範囲DL → Gemini Files API upload → poll → generateContent（503/429 自動リトライ5回）

## 出力先・知識置き場

- 解析結果・ノートの統合先: `~/.claude/video-notes/`（関連話題では Glob/Read で参照）
- 詳細リファレンス: memory [[reference_ask_video_tool]]
