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

## CSVフォーマットの重要仕様

RMシステムのSEI_RM CSV（Shift-JIS）：

```
ヘッダー行: 1B20,事業所コード,YYYYMM（年月が6桁）
データ行:   ...,YYYY,MM,...（年と月が別列 = col[2]とcol[3]）
```

→ 月フィルターは `col[2] + col[3].padStart(2,'0')` で6桁を構成して比較すること（`col[3]`だけでは2桁になりlength===6が常にfalse）

短期看護小規模の除外: `itemName.includes('短期')` で判定。baseCodeが null になる利用者を filteredUsers でフィルタ。

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

| action | 処理 |
|--------|------|
| `getAllData` | 全シートのデータをJSONで返す |
| `saveMonthlyData` | facilityId+ymの既存行削除→新データappend |
| `deleteMonth` | facilityId+ymの行削除 |
| `saveTerminalEntry` | ターミナルケア加算の登録 |
| `deleteTerminalEntry` | ターミナルケア加算の削除 |
| `saveIshaValue` | 医師指示割合の保存 |
| `saveKatsudan` | 喀痰吸引届出状況の保存 |
| `createFacility` | 事業所の新規作成 |
| `renameFacility` | 事業所名変更 |

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
            baseCode: string | null,  // 短期は null
            items: [{ name, tanka, kaisu }]
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
2. **GASデータのキャッシュ**: parseCSVはクライアント側処理。修正版コードがデプロイされていないとGASに古いデータが蓄積される。修正後は対象月を削除→再アップロードでGASデータを上書きできる
3. **型変換**: GASから返ってくる `registered` は文字列の場合があるため `=== true || === 'true'` の両方をチェックすること

## デプロイコマンド

```powershell
cd "C:\Users\024886-nakamura.024886-4313-10\Documents\kangosmall-deploy"
git add index.html
git commit -m "fix: 説明"
git push origin main
```
