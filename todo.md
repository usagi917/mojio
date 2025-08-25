# todo.md — ローカル文字起こし & 会議録整形アプリ（Minutes Maker）チェックリスト

> 目的: **常にビルド可能・テスト可能**な状態で、Electron(React+MUI)+Node(ffmpeg)+OpenAI(Whisper/GPT) による議事録生成アプリを安全に完成させる。  
> 使い方: 各項目を上から順番にチェック。各ブロック末尾の **検証コマンド** を実行して緑になることを確認。

---

## 0. 事前準備 / Preflight

- [ ] **開発環境**
  - [ ] Node.js 20+ / pnpm インストール
  - [ ] Git 初期化（main ブランチ、`.gitignore` 追加）
  - [ ] OS: Windows/macOS での動作確認用 VM or 実機を用意
- [ ] **セキュリティ/プライバシー方針の合意**
  - [ ] OpenAI への送信データは学習に使われない旨を README に明記
  - [ ] ログは個人情報を含めない（デバッグレベル切替可）
- [ ] **API キー**
  - [ ] OpenAI API キー発行
  - [ ] 後続で keytar に安全保存（平文ファイルに書かない）
- [ ] **メディア**
  - [ ] 検証用ダミー音声（5分/30分/2時間）を用意（権利クリア）
- [ ] **CI（任意）**
  - [ ] GitHub Actions などで Lint/Test/E2E/Build の雛形

**検証コマンド**
```bash
node -v && pnpm -v
```

---

## M1. プロジェクト基盤（Node20+TS+Electron+React+Vite / Vitest / Playwright）

- [ ] T1: pnpm + TypeScript + ESLint + Prettier 設定（strict / no-explicit-any 警告）
- [ ] T2: Electron main / preload / renderer(React) の最小起動（Hello）
- [ ] T3: Vitest（単体）/ Playwright（E2E）セットアップ（サンプル1件ずつ）
- [ ] T4: electron-builder で dev/build スクリプト追加

**検証コマンド**
```bash
pnpm install
pnpm dev               # ウィンドウに "Hello" 表示
pnpm test              # Vitest 緑
pnpm e2e               # Playwright 緑
pnpm build             # 配布物生成
```

---

## M2. UI スケルトン（MUI v5 / MD3）

- [ ] T5: MUI 導入・レイアウト（左:ドロップ+進捗 / 右:プレビュー / 下:保存）
- [ ] T6: ドロップゾーン（.wav/.mp3/.m4a 検証、アクセシビリティ label/role）
- [ ] T7: 状態管理（Zustand）導入とレンダラの基本状態

**受け入れ基準**
- [ ] 起動時に「Minutes Maker」タイトル、ドロップ領域、プレビュー、保存ボタンが見える
- [ ] 無効拡張子ドロップでエラーの視覚的フィードバック

**検証コマンド**
```bash
pnpm e2e               # UI 要素存在チェック
```

---

## M3. ffmpeg 分割ラッパ（約5分 / OS一時フォルダ）

- [ ] T8: ffmpeg-static + child_process.spawn で分割関数実装
- [ ] T9: 生成チャンクごとに進捗イベントを発火（index/開始秒/終了秒/パス）
- [ ] T10: セッション JSON にチャンクメタを書き出し（後続処理用）

**受け入れ基準**
- [ ] 総尺 X 秒に対し `ceil(X/300)` チャンクが生成される
- [ ] 不正ファイルで明確なエラー

**検証コマンド**
```bash
pnpm test -t "split"   # 分割数/境界の単体テスト
pnpm dev               # ドロップ→分割完了トースト表示
```

---

## M5. 進捗/ETA 骨格

- [ ] T21: 移動平均ベース ETA クラス（k=10、外れ値上下10%除外）
- [ ] T22: 進捗/ETA を renderer に逐次反映（progress.update / eta.update）

**受け入れ基準**
- [ ] 分割や文字起こしが進むほど ETA が単調減少傾向
- [ ] 停止中は ETA 固定

**検証コマンド**
```bash
pnpm test -t "eta"
```

---

## M4. Whisper クライアント（モック→本番）

- [ ] T11: transcribeChunk（Whisper API ラッパ）— モックで TDD
- [ ] T12: キュー（p-queue 等）+ 指数バックオフ（最大3回、ジッターあり）
- [ ] T13: 429/Retry-After 対応、キューのレジューム

**受け入れ基準**
- [ ] concurrency 設定を守る
- [ ] 失敗時に指数バックオフ、最終的に成功または明確な失敗

**検証コマンド**
```bash
pnpm test -t "whisper"  # 成功/失敗/429 の挙動
pnpm dev                # モックモードで UI に部分結果が流れる
```

---

## M5(続). 結合とプレビュー差分

- [ ] T14: チャンク文字起こしの index 結合（安定ソート）
- [ ] T15: プレビュー差分更新（O(Δ) の軽量パッチ適用、巨大テキスト対応の下地）

**受け入れ基準**
- [ ] 欠落インデックス検出→UI 警告
- [ ] 大きなテキストでもカクつかない（後の仮想化で強化）

**検証コマンド**
```bash
pnpm test -t "combine"
```

---

## M6. 整形（ローカルルールTDD→GPT DI）

- [ ] T16: 整形ルールのユニットテスト 10+（フィラー/相槌削除、言い直し、ですます、文分割、語尾統一）
- [ ] T17: gptFormat(text) 実装（失敗/レート時は localRules フォールバック）
- [ ] T18: 長文は段落分割+オーバーラップ+再結合→最後に軽い全体整形

**受け入れ基準**
- [ ] 意味保全（要約・削除なし）
- [ ] 境界での重複や欠落なし

**検証コマンド**
```bash
pnpm test -t "formatter"
```

---

## M7. DOCX 出力

- [ ] T19: `docx` による Document→Buffer 生成（見出し/段落の最小スタイル）
- [ ] T20: 命名規則 `<original>_minutes.docx`、重複時は `_<n>` 連番、保存ダイアログ

**受け入れ基準**
- [ ] 出力ファイルを Word で開ける
- [ ] 文字化けや改行崩れがない

**検証コマンド**
```bash
pnpm test -t "exporter"
pnpm dev    # 「保存」クリック→ファイル作成確認
```

---

## M8. 制御（Pause/Resume/Cancel）と ETA 連動

- [ ] T22: 一時停止/再開/キャンセルを UI→キューに反映
- [ ] T21: 停止中は ETA 固定、再開で再計算

**検証コマンド**
```bash
pnpm e2e -g "pause resume cancel"
```

---

## M9. 自動保存・復旧

- [ ] T23: 30秒ごとに SessionState を `userData/sessions/<hash>.json` へ保存（debounce/interval）
- [ ] T24: 起動時に復旧検出→続行/破棄ダイアログ→続行で再開

**受け入れ基準**
- [ ] 強制終了→再起動で**30秒以内**の状態まで復元

**検証コマンド**
```bash
pnpm test -t "autosave"
pnpm e2e -g "crash recovery"
```

---

## M10. 設定/セキュリティ（keytar + セキュアIPC）

- [ ] T25: keytar で OpenAI API キー保存/取得（preload 経由）
- [ ] T26: 設定（concurrency / chunkSec / ログレベル）を UI から変更→即時反映
- [ ] セキュア IPC（contextIsolation: true、最小 API 表面）

**検証コマンド**
```bash
pnpm test -t "settings"
pnpm e2e -g "settings ui"
```

---

## M11. E2E シナリオ（モック API）

- [ ] T27: 30分ダミー音声で 分割→ASR→整形→保存 の通し E2E
- [ ] T28: レート制限（429）、一時停止→再開、クラッシュ→復旧の各シナリオ

**検証コマンド**
```bash
pnpm e2e
```

---

## M12. パフォーマンス/巨大テキスト

- [ ] T29: 並列度・チャンク長のチューニング（設定で動的反映）
- [ ] T30: プレビュー仮想化（react-window 等）で 10万文字でも快適

**検証コマンド**
```bash
pnpm test -t "diff patch perf"
pnpm e2e -g "virtualized preview"
```

---

## M13. 本番 API 切替 & 設定 UI 仕上げ

- [ ] Whisper/GPT を実 API に切替（ENV/設定 UI）
- [ ] 低並列（2〜3）で短い音声が通ることを確認
- [ ] 失敗時のユーザ向けエラーメッセージ（後述マッピング）

**検証コマンド**
```bash
# 実 API（短音声）でのスモークテスト
pnpm dev
```

---

## M14. エラーハンドリング & 部分エクスポート

- [ ] errorMap: ネットワーク/レート/ファイル不正/容量不足 を分類→分かりやすい通知文言
- [ ] 途中失敗時も部分エクスポート（整形前/後テキスト）可能

**検証コマンド**
```bash
pnpm test -t "error map"
pnpm e2e -g "partial export"
```

---

## M15. パッケージング（Windows/macOS）

- [ ] electron-builder 設定（Win: nsis、mac: dmg）
- [ ] コード署名はダミー/スキップ構成でビルド確認
- [ ] 出力アーティファクトの存在検査スクリプト

**検証コマンド**
```bash
pnpm build
```

---

## M16. 最終ワイヤリング & クリーンアップ

- [ ] 未使用エクスポートやデッドコードの削除
- [ ] IPC エンドポイントの結線確認
- [ ] README: 使い方/制約/プライバシー/ログ方針/ディレクトリ構成/FAQ 追記
- [ ] 既知の制約（約5分境界の近似など）を明記

**検証コマンド**
```bash
pnpm lint
pnpm test
pnpm e2e
```

---

## 受け入れテスト / Definition of Done（本番モード・短音声）

- [ ] ドロップ→分割→ASR→整形→プレビュー→保存(.docx) が**ノンブロッキング**で完了
- [ ] 進捗バー/ETA が概ね現実的
- [ ] 一時停止/再開/キャンセルが機能
- [ ] 自動保存→クラッシュ復旧が機能
- [ ] API キーが keytar に保存され、平文で残らない
- [ ] 出力ファイル名が `<元ファイル名>_minutes.docx` 規則に一致
- [ ] ログに個人情報や機密テキストを残していない

---

## 付録A: エラーメッセージ対応表（例）

- ネットワーク不良 → 「ネットワークに問題が発生しました。接続を確認して再試行してください。」
- 429（レート制限） → 「混雑のため待機中です。しばらくして自動的に再開します。」
- 不正メディア → 「ファイル形式が不正か破損しています。WAV/MP3/M4A をご使用ください。」
- 容量不足 → 「ディスク容量が不足しています。不要なファイルを削除して再試行してください。」

---

## 付録B: GPT 整形プロンプト雛形（実装内 DI 用）

```text
あなたは公的会議録の校正者です。次のテキストを「です・ます」調の公式な議事録風に整形してください。
ルール:
- フィラー（例: えー, あのー）と相槌（はい, ええ等）を削除
- 言い直しの統合（「…いや、…です」→後者）
- 長文は意味単位で文分割（過度分割禁止）
- 語尾・表現を公式な言い回しで統一
- 内容の要約や削除は禁止（意味保全）
- 出力はテキストのみ、Markdownや装飾は禁止
入力: <TRANSCRIPT_SEGMENT>
出力: <FORMATTED_TEXT>
```

---

## 付録C: ロールバック手順（安全策）

- [ ] 直近コミットにタグを打ってから作業
- [ ] 失敗時は `git restore -s HEAD~1 -SW .` で巻き戻し
- [ ] 依存関係を変更した場合は `pnpm store prune && rm -rf node_modules && pnpm install`

---

## 付録D: 将来拡張（Parking Lot）

- [ ] PDF 出力
- [ ] 話者分離 + 名簿 CSV マッピング
- [ ] 保存前の手動編集機能
- [ ] オフライン文字起こし（whisper.cpp）
