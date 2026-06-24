# 設計概要

## アーキテクチャ全体図

```
ブラウザ（index.html）
    │
    │ FormData POST / JSON GET
    ▼
GAS Web App（Google Apps Script）
    │
    │ SpreadsheetApp
    ▼
Google Sheets（データベース）
    ├── facilities       ← 事業所マスタ
    ├── monthlyData      ← 月次データ（usersJSONを保存）
    ├── terminalEntries  ← ターミナルケア加算記録
    ├── ishaValues       ← 医師指示割合（月別・事業所別）
    └── katsudanData     ← 喀痰吸引届出状況（事業所別）
```

## 主要な設計判断

### フロントエンド: シングルHTML

外部ライブラリを一切使用しない（CDN依存なし）。グラフはSVGで自前実装。
HTMLファイル1つをコピーするだけで動作するため、メンテナー不在時でも運用継続しやすい。

### バックエンド: GAS Web App

Googleアカウントがあれば無料でRESTライクなAPIを公開できる。
Spreadsheetをそのまま確認・編集できるため、非エンジニアでもデータを直接修正可能。

### データ保存方式

月次データは利用者全員のJSONを1セルに保存（`usersJson`列）。
- メリット: スキーマ変更に強い。列追加不要
- デメリット: 1セル50,000文字上限あり（利用者100名以上で注意）

### CSVパース

RMシステムのSEI_RM CSV（Shift-JIS）をブラウザ側でパース。
`TextDecoder('shift-jis')` でデコード後、独自の `parseCSV()` 関数で処理。

CSVの重要な仕様：
- ヘッダー行: `1B20,事業所コード,YYYYMM` の3列構造
- データ行: 年（col[2]）と月（col[3]）が**別列**に分かれている（`YYYYMM`ではない）
- 日割請求: 同一利用者が複数の請求期間行に分かれて出力される
- 短期看護小規模: `itemName.includes('短期')` で判定し集計から除外

### 看護体制強化加算の判定ロジック

```
evalReqs(threeM, termValid, ishaVal, katsudanRegistered)
    ├── 加算Ⅱ: ア（医師指示≥80%）かつ イ（緊急≥50%）かつ ウ（特別管理≥20%）
    └── 加算Ⅰ: 加算Ⅱ要件 かつ エ（ターミナル1名以上/12ヶ月） かつ オ（喀痰吸引届出済）
```

`calcThreeMonths(targetYm, facId)` が前3ヶ月の利用者データを集計して割合を計算。

### 認証

パスワードをSHA-256でハッシュ化し、クライアント側で比較。
`hash(input) === '2b68c603...'` で判定。セッションはlocalStorageに保存。
※ セキュリティ強度は高くない。機密情報の管理には使用しないこと。

## データフロー

### CSVアップロード時
```
1. FileReaderでShift-JISデコード
2. parseCSV() でユーザーオブジェクトに変換
3. facilityData[facId].monthData[ym] にセット（メモリ）
4. gasPost({action:'saveMonthlyData', ...}) でGASへ送信
5. GASがSheetの既存行を削除→新データをappend
```

### ページ読み込み時
```
1. gasPost({action:'getAllData'}) でGASから全データ取得
2. JSON.parse(row.usersJson) でメモリに復元
3. renderDashboard() で画面描画
```

## ファイル別の役割

| ファイル | 役割 |
|----------|------|
| `index.html` | アプリ全体（HTML + CSS + JS） |
| `kangosmall_gas.js` | GASバックエンド（Google Apps Scriptに貼り付けて使用） |
