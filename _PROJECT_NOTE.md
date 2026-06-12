# SAA-C03 学習アプリ — 引き継ぎノート

別のチャットに作業を引き継ぐ場合、このファイルの内容を貼り付ければ続きから作業できます。

---

## 📁 ファイルの場所（編集対象）

すべて `E:\勉強用\SAA\勉強機関アプリ\SAA_study_app\` 配下：

| ファイル | 役割 | 編集 |
|---|---|---|
| `index.html` | アプリ本体（UI・状態管理・全ロジック・CSS・JS全部入り、約8000行超） | ★主に編集 |
| `sw.js` | Service Worker（オフライン用キャッシュ）。変更時は `CACHE_VERSION` を上げる（現在 `saa-v1.3.0`） | 更新時に必須 |
| `manifest.json` | PWAマニフェスト | ほぼ固定 |
| `icon.svg` / `icon-maskable.svg` | アプリアイコン | 固定 |
| `study_data.js` | 教材データ（`const STUDY_DATA`）。28節・用語・要点・確認テスト | 自動生成 |
| `flashcard_data.js` | 単語帳データ（`VOCAB_CARDS`＝434件 / `DIAGRAM_CARDS`＝27図 / `INDEX_ITEMS`＝384語） | 自動生成 |

### データ生成スクリプト（編集元）
- `E:\勉強用\SAA\gen_study_data.py` → `study_data.js` を生成
- `E:\勉強用\SAA\gen_flashcard_data.py` → `flashcard_data.js` を生成（用語はCSVから自動、図解カードはスクリプト内に手書きSVG/HTML）
- 元素材: `E:\勉強用\SAA\{章フォルダ}\要点\02_用語解説.csv`・`03_要点解説.csv`、`演習問題\演習問題.csv`、`キーワード.txt`
- 索引: `E:\勉強用\SAA\索引.csv`（用語,意味,関連章）→ `INDEX_ITEMS` に変換、各語をvocabカードへ自動紐づけ
- 章フォルダは1-1〜5-4の全28節（4・5章も追加済み）
- 参考書スクショ: `E:\勉強用\SAA\SAAスクショ\{章名}\*.png`（図解カード作成の元ネタ）

### 再生成コマンド
```
cd E:\勉強用\SAA && python gen_study_data.py
cd E:\勉強用\SAA && python gen_flashcard_data.py
```

---

## 🚀 デプロイ方法
GitHub Pages に上げてスマホでPWAとして使用。
編集後は **変更したファイル＋`sw.js`（CACHE_VERSION必須インクリメント）** をGitHubに push → スマホで更新バナーから反映。

---

## 🗂 アプリのタブ構成
ヘッダーのタブ: **予定 / 教材 / 単語帳 / デッキ / 統計 / 模試 / 設定**
- 予定（schedule）: 日別タスクのチェックリスト、デッキ、メモ、勉強時間記録
- 教材（study）: 節アコーディオン（キーワード・用語解説・要点・確認テスト4択）
- 単語帳（flash）: 用語/図解フラッシュカード＋クイズ＋図解虫食い＋閲覧モード
- デッキ（decks）: Ping-t等のデッキを5段階反復管理（●●●○○）
- 統計（stats）: 進捗グラフ
- 模試（exam）: 模擬試験スコア記録
- 設定（settings）: バックアップ/エクスポート/インポート/リセット

---

## 🧩 主要な実装ポイント（index.html内）

### 状態管理
- `state` オブジェクトを localStorage に毎回保存（`saveState()`）
- フィールド: `checks, actualHours, gapQuestions, notes, examScores, openDates, deckDone, deckDoneByDate, dayDecks, lastBackup, decksSeeded, quiz, studyOpen, flashSessions, flashGoals`
- リセット/インポートの初期化箇所が2つあるので state フィールド追加時は両方更新する

### スケジュール
- `const SCHEDULE = [...]` に日別データ（date, dow, shift, workTime, targetHours, tasks[], memo）
- `const EXAMS = [...]` に模試一覧

### 単語帳（flash）
- `renderFlash()` ホーム / `renderFlashCard()` クイズ / `renderFlashBrowse()` 閲覧 / `renderFlashDiagramQuiz()` 図解虫食い
- `buildFlashCards(settings)` でカード生成。direction: normal/reverse/random（用語→意味 / 意味→用語 / ランダム）
- 図解虫食いは虫食い率（`window._fClozeRate`、デフォルト50%）で各空欄を個別にランダム隠し
- `ALL_SECS` に対象節リスト（現在 1-1〜3-5）

### ポモドーロ＋全日タイムライン（今回の作業）
- 左下にフローティング 🍅 タイマー（`pomoFloat`）→ タップでパネル（`pomoPanel`）
- `_pomo` オブジェクトでタイマー管理。45分作業→10分休憩（変更可）、終了時ビープ＋バイブ、休憩は自動開始
- パネル内「📋 今日の全日タイムライン」ボタン → `renderDayPlan()`
- `const DAY_PLANS = { "6/3":{...}, "6/4":{...}, "6/5":{...} }`
  - 各日: `label`, `limitH/limitM`（終了リミット）, `blocks[]`
  - blockのtype: `study`(ポモ数), `mock`(模試本番・分指定・ポモなし), `meal`, `break`
- `computeDayTimeline()` が effStart/effEnd を計算（詰めモード対応）
- `_dayActual` = ブロックごとの実際完了時刻、`_dayCompress` = 日ごとの詰めモードON/OFF

---

## ✅ 直近で完了した作業（2026/6/1時点・v1.3.0）

### 集中3日間スケジュール（模試①32%を受けた追い込み）
- 6/3（8:00〜23:00）★集中日①: 4・5章読了＋学習模試A＋弱点①②
- 6/4（7:00〜23:00）★集中日②: 弱点③④＋学習模試B/C＋間違い再演習
- 6/5（7:00〜18:00）★集中日③: 実力模試①＋復習＋単語帳
- 模試タブも mock2=実力模試①(6/5)、mock3=②(6/6)、mock4=③(6/9) に更新

### ポモドーロ＆全日タイムライン
- 45分/10分のポモドーロタイマー（フローティング＋パネル）
- 全日タイムライン（DAY_PLANS）: 現在時刻にNOWバッジ、ブロックごと完了ボタン
- **模試ブロックはポモなし**（本番形式・130分計測、`type:"mock"`）
- **詰めモード**: ブロックを早く終えて「完了✓」を押すと、詰めモードONで以降のブロックを前倒し → 末尾に空き時間（freeMin）が出る。模試が猶予内に終わった時に時間を詰められる

---

## 📌 次にやれそうなこと（未着手アイデア）
- ポモ完了数や `_dayActual` を localStorage 永続化（現在はリロードで消える）
- 6/6以降の集中日以外もDAY_PLANS化
- 4・5章・6/3〜の用語/図解カードの拡充（スクショから手書きSVG追加）
- 単語帳の正答率を節ごとに集計表示

---

## 👤 ユーザー状況メモ（2026/6/12更新）
- AWS SAA-C03 試験日: **6/24**（6/10から延期）
- 模試実績: ①26%（5/30）→②35%→③34%（6/3）。合格ライン72%に対し横ばい
- 弱点（模試3回横断・`E:\勉強用\SAA\模擬試験結果\弱点分析\弱点まとめ.md` 参照）:
  1. コンピュート（模試③で最多11問）2. ネットワーク（3回連続最多級・最も根深い）3. IAM（再悪化）4. DB周辺（全滅）5. S3（改善中）
- 6/12にSCHEDULE改訂（v1.6.0）: 模試④を6/9→6/14に移動（目標45%）、模試⑤を6/20（目標55%）、模試⑥は中止して「間違い見直し優先」に再構成
- 勉強リズム: **45分勉強→10分休憩**。夜勤シフトのため夜勤2日目は勉強NG
- 間違い見直しは `E:\勉強用\SAA\模擬試験結果\チェックリストアプリ.html`、選び分けルールは `弱点分析\弱点克服ガイド.md`
