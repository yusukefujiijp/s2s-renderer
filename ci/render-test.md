# CI: render-test (概念設計 / artifact v4 前提)

## 1. 概要
- 目的: `scenes.json` からサーバ側で安定して短尺（10秒）動画を生成するためのCI概念。
- DoD: 10秒×360p×15fps の MP4（H.264）を生成し、GitHub Actions(v4) の artifact としてアップロード／DLできること。
- 暫定閾値: 映像落ち < 5%、音ズレ < 80ms

## 2. トリガ
- push (特定ブランチ: `chore/poc-static-frames`) または workflow_dispatch（手動実行）

## 3. 実行環境（概念）
- runner: ubuntu-latest
- タイムアウト: 初期 ≤ 10 分（必要に応じて延長）
- 必要ソフト: Node.js（レンダ操作用）/ Chromium（ヘッドレス）/ ffmpeg（libx264）

## 4. ジョブ構成（高レベル）
1. checkout
2. setup-node (if needed)
3. install deps (npm ci / yarn)
4. render (headless chromium): produce image frames or recorded raw media
5. encode (ffmpeg + libx264): frames/raw → mp4 (baseline, compatible with iOS)
6. capture metrics: render time, frame count, dropped frames, A/V offset (ms)
7. upload-artifact (v4): video.mp4 + metrics.json + logs

## 5. 成果物命名規約
- run-<commit>-<res>-<fps>-<secs>.mp4
  例: run-b22e7f9-360p-15fps-10s.mp4

## 6. ログ / メトリクス（必須）
- render_start_timestamp, render_end_timestamp (ms)
- frame_total, frame_expected, frames_dropped, drop_rate (%)
- audio_present (bool), av_offset_ms (signed integer)
- ffmpeg_encode_time_ms, ffmpeg_return_code

（これらを metrics.json として artifact に含める）

## 7. 受け入れ基準
- artifact が存在すること（non-zero size）
- drop_rate < 5%
- abs(av_offset_ms) < 80
- logs にエラーが無いこと

## 8. 保持方針 / ストレージ
- artifact 保存期間: 初期 7 日（運用で調整）
- 高解像度テストは短期間のみ保存（容量管理）

## 9. フォールバック / 代替
- mp4 出力失敗時: webm 出力（VP9）を試行し、Runbook に変換手順（webm→mp4 via ffmpeg）を追加
- Chromium レンダ失敗時: スタティック画像出力→ffmpeg で合成（最小PoC）

## 10. 次アクション（実行計画）
1. この文書を main に PR で入れる（今）  
2. `chore/poc-static-frames` ブランチを作成して最小PoC（静止画像数枚→ffmpegで10秒MP4）を実装  
3. CI をトリガして artifact がアップされることを検証（3回連続成功を目標）

## 11. 補足（運用上の注意）
- artifact は v4 を前提に記載。v3は既に非推奨/廃止のため使用しないこと。
- すべての実験は `exp/...` タグで記録すること（今回運用済）。