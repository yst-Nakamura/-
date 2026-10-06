# CLAUDE.md — AI引き継ぎ用コンテキスト

このファイルはAIコーディングツール（Claude Code等）が次のセッションでコンテキストを回復するためのドキュメントです。

---

## プロジェクト概要

看護小規模多機能型居宅介護向けの加算管理ダッシュボード。RMシステムCSVをアップロードするだけで、看護体制強化加算の算定可否を自動判定するWebアプリ。

## 技術スタック

- **フロントエンド**: 単一HTMLファイル（`index.html`）。CSS・JSすべてインライン。外部ライブラリなし
- **バックエンド**: Google Apps Script（GAS）Web App。REST APIとして機能
- **DB**: Google Sheets（シート名: facilities / monthlyData / terminalEntries / ishaValues / katsudanData）
- **ホスティング**: GitHub Pages（メイン）/ Netlify（予備）

## 重要なURL・キー

```
GAS URL: https://script.google.com/macros/s/AKfycbw5vg_DYk2oMifFJmkPng6ms8-9BbN-JykrVbqYGu9MBJsDU73c58cNVX8Q3Gfb0no6/exec
GitHub Pages: https://yst-nakamura.github.io/-/
Netlify: https://imaginative-sprinkles-74ecb5.netlify.app/
パスワードハッシュ: 2b68c603cd53027a45cd35c50e9d75b936cd2628ae0a3144acbbc291ad618275 (yst0001)
```

## ファイル配置

```
Documents/
├── 加算管理kangosmall_dashboard.html  ← ローカル編集用（index.htmlと同一内容を維持）
├── kangosmall_gas.js                  ← GASソース
└── kangosmall-deploy/                 ← Gitリポジトリ
    └── index.html                     ← デプロイ用（こちらを push する）
```

**修正手順**: `加算管理kangosmall_dashboard.html` を編集 → `kangosmall-deploy/index.html` に同じ修正を適用 → `git push origin main`

**絶対ルール（ユーザー指定）**: コード変更（バグ修正・機能追加・UI修正を問わず）時は、**両ファイルの `CHANGELOG_DATA` 配列の先頭に新バージョンのエントリを必ず追加**する。バージョンは直前をインクリメント（例: v1.6.4 → v1.6.5）、dateは当日、labelは「新機能」「改善」「バグ修正」から選ぶ。CHANGELOG_DATA は各ファイルの末尾付近（3300行前後）にある。デプロイ前の全チェック項目は deploy-checklist スキルを参照。

## ファイル内セクションの探し方

3,300行の単一ファイル。全文Readせず、以下で位置を特定して部分Readする:

- CSS/HTMLセクション: `===== セクション名 =====` 形式のコメントをGrep（1〜590行）
- JS関数: `function 関数名` をGrep（588行目の `<script>` 以降）
- 修正ログ: `CHANGELOG_DATA` をGrep（末尾付近）

## CSVフォーマットの重要仕様

RMシステムのSEI_RM CSV（Shift-JIS）：

```
ヘッダー行: 1B20,事業所コード,YYYYMM（年月が6桁）
データ行:   ...,YYYY,MM,...（年と月が別列 = col[2]とcol[3]）
```

→ 月フィルターは `col[2] + col[3].padStart(2,'0')` で6桁を構成して比較すること（`col[3]`だけでは2桁になりlength===6が常にfalse）

短期看護小規模の除外: `svcType=37`（小多機は34/35）の行をスキップし、`baseCode` が null の利用者を `filteredUsers` で除外する。

**GASへ保存するのは必ず `filteredUsers`**。v1.6.22 まで除外前の `users` を保存していたため、取込直後は正しく見えても再読み込みで短期利用者が復活していた（短期混入が何度直しても再発した真因）。古い保存データ向けに `_applyRawData` の読み込み時にも除外している（v1.6.24）。

**`parseCSV` の集計ロジックを変えたら `PARSER_VERSION` を必ず上げる**。保存データに刻まれた版数が古い直近3ヶ月の月に「要再取込」マークが出る。

日割請求: 同一利用者が複数行に分かれるため、加算は `u.items.some(i => i.name === itemName)` で重複チェック。

## 主要関数一覧

| 関数 | 役割 |
|------|------|
| `parseCSV(text, ym)` | CSVをパースしてmonthDataを構築 |
| `getLevel(baseCode)` | 基本コードから介護度（1〜5）を取得。全角・半角両対応 |
| `evalReqs(threeM, termValid, ishaVal, katsudanRegistered)` | 看護体制強化加算Ⅰ・Ⅱの判定 |
| `calcThreeMonths(targetYm, facId)` | 指定月算定のための前3ヶ月集計 |
| `checkTerminal(ym, facId)` | ターミナルケア加算の有効判定（12ヶ月） |
| `saveKatsudan(ym, registeredRaw)` | 喀痰吸引届出状況の保存。boolean/string両方を受け付ける |
| `showAvgLevelHistory(facId)` | 平均介護度推移モーダルの表示 |
| `gasPost(payload)` | GASへのPOSTリクエスト共通関数 |
| `loadAllData()` | 起動時のGASからの全データ読み込み |

## GAS側アクション一覧

**`kangosmall_gas.js` の `doPost` の switch が正。名前を推測で書かないこと**（以前この表が誤っていて、それを元に書かれた D1 同期が一部一度も動いていなかった）。

| action | 処理 |
|--------|------|
| （GET `doGet`） | `getAllData()` で全シートのデータをJSONで返す |
| `addFacility` | 事業所の新規作成。IDはGASが採番し戻り値 `{ success, id, name }` で返す |
| `saveServiceType` | サービス種別変更 |
| `renameFacility` | 事業所名変更 |
| `deleteFacility` | 事業所と関連データを削除 |
| `saveMonthlyData` | facilityId+ymの既存行削除→新データappend（再アップロードで上書きされる） |
| `deleteMonth` | facilityId+ymの行削除 |
| `saveTerminal` | ターミナルケア実績を事業所単位で丸ごと置き換え（`entries` 配列） |
| `saveIsha` | 医師指示割合の保存 |
| `saveKatsudan` | 喀痰吸引届出状況の保存（職員名は `staffJson` 文字列） |

D1（Cloudflare Worker `shift-worker`、シフト作成アプリと共用）へは `gasPost` 後に `d1KangoSync` で非同期コピーする。`saveServiceType` / `deleteFacility` は Worker に受け口が無く未同期。D1 の `kango_monthly_users` には金額・区分・取込版数の列が無いため、D1 から復元した表示は不完全（画面上部に注意表示が出る）。

## データ構造（メモリ）

```javascript
facilityData = {
  [facId]: {
    monthData: {
      [ym]: {    // e.g. "202601"
        ym,
        label,
        users: {
          [userId]: {
            name: string,
            baseCode: string | null,  // 短期は null（保存時に除外済み）
            items: [{ name, tanka, kaisu }],
            totalRec, insuranceTotal, userSvcType,
            _pv: number               // 取込時のパーサ版数（PARSER_VERSION）
          }
        }
      }
    },
    terminalEntries: [{ ym, label }],
    manualIsha: { [ym]: number },
    katsudan: { registered: true|false|null, staff: string[] }
  }
}
```

## 既知の問題・地雷

1. **再請求CSV**: ターミナルケア加算が再請求月に記録される。手動で正しい月を入力することで回避
2. **GASデータは過去の取り込み結果がそのまま残る**: parseCSVはクライアント側処理のため、パース修正をデプロイしても**保存済みの月データは自動では直らない**。該当月を**再アップロードするだけで上書きされる**（`saveMonthlyData` が facilityId+ym の既存行を削除してから書き込むため、事前の月削除は不要）。
   - 短期利用者の混入 → v1.6.22 で読み込み時にも除外するため、表示上は再取込なしで是正される
   - **過去月の混入** → 保存データに月の情報が残らないため**自動検出は不可能**。再アップロードでしか直らない（旧版の月フィルタバグ `col[3]` のみで6桁判定していた時期の取り込みが原因）
   - 見分け方: 登録利用者数がRM側の実数より多い／身に覚えのない利用者が明細に出ている
3. **型変換**: GASから返ってくる `registered` は文字列の場合があるため `=== true || === 'true'` の両方をチェックすること
4. **スプレッドシートの日付自動変換**: 「2026年7月」「2026-07」は日付に変換され、世界標準時の `"2026-06-30T15:00:00.000Z"` で返る（日本時間では7月1日）。年月は必ず `toYm()` で正規化し、ラベルは保存値を使わず `ymLabel(ym)` で作り直す
5. **前3ヶ月の取り込み漏れ**: 欠けている月がある場合、`calcThreeMonths` の `gapMonths` に入り、`evalReqs` と詳細画面は可・不可とも「要確認」にする。残りの月だけで判定すると要件未達のまま請求するおそれがあるため

## デプロイコマンド

```powershell
cd "C:\Users\024886-nakamura.024886-4313-10\Documents\kangosmall-deploy"
git add index.html
git commit -m "fix: 説明"
git push origin main
```
