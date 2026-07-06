# voice-coach Phase 0 スパイク仕様（実車検証用デモ）

## 目的
運転中音声コーチングアプリの技術前提を実車で検証するための最小デモ。
検証対象：①Android の音声認識ビープ音の頻度と許容度 ②走行ノイズ下の認識精度 ③Bluetooth(A2DP) 経由の TTS 聴取性 ④Wake Lock の持続。
**Claude API は使わない**（エコー応答のみ）。ネットワークは音声認識(Google)のみ使用。

## 成果物
`index.html` 単一ファイル（外部依存ゼロ・ビルド不要・全て inline の HTML+CSS+JS）。
GitHub Pages (HTTPS) で配信する前提。日本語 UI。加えて `manifest.webmanifest`（ホーム画面追加用・最小限）と `.nojekyll`（空ファイル）。Service Worker は作らない。

## 機能要件

### 1. 会話ループ（半二重）
- 大きな「開始」ボタン（画面下半分・タップ 1 回）→ Wake Lock 取得 → 挨拶を TTS（「こんにちは。マイクテストを始めます。何か話しかけてください。」）→ listening 開始
- 状態遷移: idle → listening（認識中）→ speaking（TTS 再生中）→ listening → …
- **半二重厳守**: speaking 中は SpeechRecognition を必ず stop し、TTS の `onend` 後に認識を再開する（自己エコー防止）
- エコー応答: 認識確定テキスト T に対し「『T』と聞こえました。」＋ローテーションで次のいずれかを続けて発話：「続けてどうぞ。」「もう少し詳しく教えてください。」「他にはありますか。」（コーチング対話のリズムを模擬）
- 音声コマンド: 認識テキストが「終了」「しゅうりょう」「おわり」を含んだら「テストを終了します。お疲れさまでした。」と発話してループ停止・idle に戻る

### 2. STT（音声認識）
- `webkitSpeechRecognition || SpeechRecognition`、`lang='ja-JP'`、`interimResults=true`、`continuous=true`
- `onend` で（speaking 中でも停止指示中でもなければ）自動 `start()` する再起動ループ。`start()` が InvalidStateError を投げた場合は 250ms 後にリトライ
- `onerror`: `no-speech`/`aborted` は静かに再起動。`network` はログして 1 秒後再起動。`not-allowed` は画面に権限エラーを大きく表示
- interim（途中経過）テキストも画面に薄色で表示

### 3. TTS（読み上げ）
- `speechSynthesis`、日本語 voice を自動選択（`ja` を含む voice 優先）
- **文単位分割発話**: 「。」「？」「！」で分割し、順次 `SpeechSynthesisUtterance` をキューに投入（長文途切れバグ対策）
- 最後の utterance の `onend` で listening を再開。保険として発話開始から 15 秒のタイムアウトでも再開

### 4. Wake Lock
- `navigator.wakeLock.request('screen')`。`visibilitychange` で visible に戻ったら再取得。非対応ブラウザでは警告表示

### 5. UI（運転中に読ませない設計）
- 画面全体の背景色で状態を示す: idle=濃灰、listening=緑、speaking=橙、エラー=赤。黒基調・低輝度前提のダークデザイン
- 中央に最終認識テキストを特大フォント表示（停車時確認用）、その下に interim を薄く
- 「開始」「停止」の 2 大ボタンのみ（停止はループ中のみ表示・画面下部全幅）
- 画面上部に小さくセッション統計: 認識回数 / 再起動回数（≒ビープ推定回数）/ エラー回数 / 経過時間

### 6. 診断ログ（実車後のレビュー用）
- 全イベント（start/result/end/restart/error/speak 開始・終了、Wake Lock 取得/喪失）をタイムスタンプ付きで記録
- localStorage に自動保存（キー `voicecoach:log`、直近 1000 件）。折りたたみ式ログパネル＋「ログをコピー」ボタン（`navigator.clipboard`）＋「ログ消去」ボタン
- 認識結果は confidence 値も記録

### 7. manifest.webmanifest
- name「Voice Coach 実車テスト」、display: standalone、背景 #111、theme #111。アイコンは SVG の data URI か省略可（省略ならその旨コメント）

## 制約
- 外部 CDN・ライブラリ禁止。フレームワーク禁止（素の JS）
- Android Chrome を主対象（デスクトップ Chrome でも動作確認できること）
- コード中のコメントは簡潔な日本語で、各セクションの役割がわかる程度

## 受け入れ確認（Codex 自身が行うこと）
- `node --check` 相当の構文検証が通ること（script 部分を抽出して node で検証してよい）
- 状態遷移に「speaking 中に認識が動く」穴がないことをコードレビューで確認
