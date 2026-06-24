# 看護小規模多機能型居宅介護 加算管理ダッシュボード

RMシステムから出力したCSVをアップロードするだけで、各事業所の介護保険加算の算定状況・看護体制強化加算の可否・平均介護度の推移をリアルタイムに把握できるWebアプリです。Google Sheetsをデータベースとして使用し、複数事業所のデータをクラウドで共有管理します。

---

## 関連URL一覧

| 用途 | URL |
|------|-----|
| **本番（GitHub Pages）** | https://yst-nakamura.github.io/-/ |
| **予備（Netlify）** | https://imaginative-sprinkles-74ecb5.netlify.app/ |
| **GAS Web App** | https://script.google.com/macros/s/AKfycbw5vg_DYk2oMifFJmkPng6ms8-9BbN-JykrVbqYGu9MBJsDU73c58cNVX8Q3Gfb0no6/exec |
| **ソースコード（GitHub）** | https://github.com/yst-Nakamura/- |

> **パスワード:** yst0001  
> GAS URLはアプリ画面の「⚙️ GAS設定」から変更可能

---

## ファイル構成

```
kangosmall-deploy/
├── index.html          ← アプリ本体（HTML/CSS/JS すべて1ファイル）
├── docs/
│   └── manual.md       ← スタッフ向け操作マニュアル
├── README.md           ← このファイル
├── CHANGELOG.md        ← 変更履歴・既知問題
├── requirements.md     ← 要件・背景
├── design.md           ← 設計概要
├── tasks.md            ← タスク管理
└── CLAUDE.md           ← AI引き継ぎ用コンテキスト
```

ローカル作業ファイル（Gitで管理しない）:
```
Documents/
├── 加算管理kangosmall_dashboard.html  ← ローカル編集用（index.htmlと同一内容）
└── kangosmall_gas.js                  ← GASバックエンドのソース
```

---

## 環境構築（ローカル開発）

特別な環境構築は不要です。`index.html` をブラウザで直接開くだけで動作します。

```
# リポジトリをクローン
git clone https://github.com/yst-Nakamura/-.git kangosmall-deploy
cd kangosmall-deploy

# ブラウザで開く（Windowsの場合）
start index.html
```

ただし、GAS連携（データの保存・読み込み）はGAS URLが設定されている場合のみ動作します。

---

## デプロイ方法

### GitHub Pages（本番）

```bash
# 修正後は index.html を編集して push するだけ
git add index.html
git commit -m "fix: 修正内容を記述"
git push origin main
```

→ 1〜2分後に https://yst-nakamura.github.io/-/ へ自動反映されます。

### Netlify（予備）

Netlify は GitHub リポジトリと連携済みです。`main` ブランチへの push で自動デプロイされます。手動操作は不要です。

### GASの再デプロイ（バックエンド変更時のみ）

1. Google Apps Script エディタを開く
2. `kangosmall_gas.js` の内容をコピーして貼り付け
3. 「デプロイ」→「新しいデプロイを管理」→「新しいデプロイ」
4. 種類: ウェブアプリ、アクセス: 全員
5. 発行後に表示されるURLを `index.html` の `GAS_URL` 定数に貼り付けて push

---

## 担当者・連絡先

| 役割 | 担当 |
|------|------|
| 開発・保守 | 中村 裕貴（yutaka.nakamura.y@sashiite.com） |
| GASアカウント管理 | 同上（Googleアカウントでのログインが必要） |
| GitHub管理 | yst-Nakamura アカウント |
