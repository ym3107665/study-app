# 公認会計士試験 学習アプリ (study_app.html)

## 概要
- 単一HTMLファイル (~4,507行, ~334KB)
- IndexedDB (version 6), marked.js, KaTeX, PDF.js, JSZip, Google Drive PKCE OAuth
- 動作環境: VS Code Live Server (port 5500), Edge (PC専用)
- 言語: UIテキスト・応答すべて日本語

## 科目構成
6科目: financial(財務会計論) / managerial(管理会計論) / audit(監査論) / law(企業法) / tax(租税法) / business(経営学)

暗記タブ専用追加カテゴリ (SUBJECTSには含めない):
- kanji(漢字) → MEM_EXTRA_SUBJECTS で定義

subTopic を持つ科目:
- audit: 個別論点 / 論点洗出
- law: 要件 / 論点 / 趣旨 / 判例 / 条文番号 (フィルタは「論点等」「条文番号」の2グループ)
- tax: 法人税 / 所得税 / 消費税

## タブ構成
ダッシュボード / 予定 / 復習 / 暗記 / 計算 / ノート / 問題管理 / 成績 / 設定

## IndexedDB ストア
notes, questions, settings, exams, memorize, calc

## 主要 State (S オブジェクト)
view, noteFilter, qFilter, noteTaxSub, qTaxSub,
editingNote, viewingNote, editingQ, viewingQ, editingQCalcPicker,
studySubject, studyQueue, studyIdx, cardFlipped, studyEditing, sessionResults, sessionData,
studyPickerSubject, studyPickerSelected,
expandedTopics, expandedQTopics,
journalCtx, examFilter, viewingExam, examMemoEditing, examSubjectMemoEditing, examImport,
memFilter, memSubFilters{tax,law,audit}, memTopicFilter, memImpFilter, memDueOnly, memTodayOnly,
viewingMem, editingMem, memMemoEditing, memAnswerShownId, memImport, memDragId,
planView, planMonth, planEditingCat, planSpotBulk, planWeekMode, planMonthMode, planSpotsOpen, planManualProgress,
calcFilter, calcTaxSub, calcImpFilter, viewingCalc, editingCalc, calcMemoEditing, calcQPicker, calcImport

## 重要な設計パターン

### 深夜3時境界ルール
- 要復習/今日変更の判定は深夜3時を日付の区切りとする
- 0:00-2:59 のチェックは前日扱い
- 関数: effectiveTodayLocal(), memEffectiveToday(), memBaseDateFromIso(), calcEffectiveYmdFromIso()

### 計算問題の自動カウント
- calc ストアの history [{result, at}] から、その日に履歴追加されたユニーク問題数を自動集計
- getCalcAutoCountForDay(ymd, sid): 指定日の科目別ユニーク問題数
- getCalcAutoCountInMonth(yyyyMM, sid, untilYmd?): 月内の累計
- planComputeDayValue で remaining 計算時に自動カウントを加算
- togglePlanDayCheck では計算カテゴリは planProgress に加算しない (二重計上防止)
- 月進捗バー (月ビュー・週ビュー) でも planProgress.calc + 自動カウントを合算表示

### 予定タブのカスケード計算
- 月目標 → 「残り目標 ÷ 今日以降の残り有効日数」で今日の日割り値を算出
- 今月の未来日は全日同じ値 (今日時点の進捗ベースで均等配分)
- 過去日 / 次月以降は planPerValidDay (月全体の素直な配分)
- 模試日は計算カテゴリのみ Math.floor(perDay/2) で半減

### 復習タブ
- 「全てを一括学習」: 従来通り要復習問題で即開始
- 「選んで学習」: openStudyPicker → 全問題からチェックボックスで選択 → startStudyFromPicker

### 暗記タブの科目別一覧表示
- audit: 論点番号 | 論点
- law: 論点番号 | 種類 | 論点 | ページ
- managerial: 論点番号 | 問 | 論点
- business: 論点番号 | 問 (論点名=説明文は非表示、詳細モーダルで確認)
- kanji: 論点番号 | ひらがな (答え=漢字はタップで開示)
- financial / その他: 論点番号 | 論点 | 問

### 暗記の未着手状態
- recallLevel=null かつ nextDueDate=null → 要復習に含まれない
- CSVインポート時はデフォルトで未着手

### CSVインポート (暗記タブ)
detectCsvFormat でヘッダーから自動判定:
- financial: 問題番号+論点番号
- managerial: 番号
- audit: テキスト
- audit2: 監基法重要度
- law: 範囲+種類
- tax: 税目+論点
- business: 章 (論点名→topicName, 章→topicNumber, 番号→problemNumber, 答え→answer)

## ⚠️ 重要: 編集時の注意事項

### 関数消失チェック (必須)
過去に str_replace の貪欲マッチで renderNotes, buildOrderedKeys, planCatDetailBadge が巻き込み削除された。
編集後は必ず以下で確認:

```bash
node --check study_app.html  # まずscript部分だけ抽出してから
# 主要関数の存在チェック
grep -c "function renderStudy\|function renderMemorize\|function renderCalc\|function renderPlan\|function renderDashboard\|function renderNotes\|function buildOrderedKeys\|function planCatDetailBadge\|function openStudyPicker\|function getCalcAutoCountForDay" study_app.html
```

### 機能消失チェック (必須)
以下のフラグが全て存在することを確認:

```bash
grep -c "MEM_EXTRA_SUBJECTS\|openStudyPicker\|let calcDone = prog.calc\|done += await getCalcAutoCountInMonth\|study-pick-item\|>暗記<" study_app.html
```

期待値: 各1以上

### JS構文チェック
```bash
python3 -c "
import re
with open('study_app.html','r',encoding='utf-8') as f:
    content = f.read()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', content, re.DOTALL)
inline = ''.join(s for s in scripts if s.strip())
with open('/tmp/inline.js','w',encoding='utf-8') as f:
    f.write(inline)
" && node --check /tmp/inline.js
```

## ヘルパー関数リファレンス

### 日付
todayLocal(), ymdLocal(d), nowLocalIso(), memBaseDateFromIso(iso),
memEffectiveToday(), effectiveTodayLocal(), isDue(), planParseYMD(), planYMD(),
planMonthOf(), planWeekMonday(), planMonthLabel(), planAdjustMonth(), planMondaysInMonth()

### データ表示
showSubTopic(rec), esc(s), sName(id), renderMd(t), renderMath(el)

### 暗記
recalcNextDue, memIsDue, memChangedToday, memSubsFor, memHasSubs,
memFilterChoicesFor, getMemSub, setMemSub, memSubLabel, LAW_FILTER_GROUPS,
memLawGroupOf, detectCsvFormat, applyCsvFormat, parseMemorizeCSV, parseCSVRows,
notionLevelToApp, parseNotionDateTime, memTopicHeadNum, memTopicNumCompare, memProblemNumCompare

### 計算
CALC_SUBJECTS, CALC_RESULTS, calcResultLabel, calcResultClass, calcLastUpdated,
calcFormatTimestamp, calcProblemNumCompare, updateCalcField, calcEffectiveYmdFromIso,
getCalcAutoCountForDay, getCalcAutoCountInMonth,
renderCalc, renderCalcDetail, renderCalcEditForm, handleCalcCSVFile,
renderCalcImportPreview, saveCalcImport

### 問題管理
openQViewer, renderQViewer, closeQViewer, editFromQViewer,
toggleQCalcPicker, toggleQCalcRelation, saveQEditorBufferToState, saveQ, deleteQ

### 復習
openStudyPicker, renderStudyPicker, toggleStudyPick, toggleAllStudyPicker,
closeStudyPicker, startStudyFromPicker

### 予定
planIsValidDayForCat, planCountValidDaysInMonth, planCountValidDaysFromDate, planPerValidDay,
planComputeDayValue, planComputeWeekValue,
planGetMonthGoals/planSaveMonthGoals,
planGetWeekOverride/planSaveWeekOverride,
planGetDayOverride/planSaveDayOverride,
planGetProgress/planSetProgress/planAddProgress,
togglePlanDayCheck, commitDayCheckCustom,
planCatDetailBadge, planProgressBarHTML,
listSettingsKeys, delSetting,
collectPlanSettings/applyPlanSettings,
PLAN_CALC_SUBJECTS, PLAN_CATS, PLAN_CAT_LABELS, PLAN_CAT_UNITS

## 予定タブのデータ構造 (settings ストア内)
- planMonthGoals:YYYY-MM → { subjectId: { calc, calcStart?, calcEnd?, calcDays?, theoryPages, ..., lecture, ... } }
- planWeekOverride:YYYY-MM-DD(月曜):subjectId → { calc?, theoryPages?, lecture? }
- planDayOverride:YYYY-MM-DD:subjectId → 同上
- planProgress:YYYY-MM:subjectId → { calc, theoryPages, lecture } 累積完了数
- planSpotTasks → [{id, date, subject, title}]
- planChecks:YYYY-MM-DD → { auto_mem, auto_q, day_<sid>:{committed:{...}}, spot_<id>, initialCounts:{mem,q} }
- subjectExamMemo:<subjectId> → 科目別メモ(Markdown)

## バックアップ
- JSON エクスポート/インポート (version 6, plans/subjectExamMemo/progress 全て含む)
- Google Drive PKCE OAuth バックアップ (version 6)

## 画面幅
暗記/計算/予定タブのみ body.wide-view で 780→1280px に拡張
