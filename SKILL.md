---
name: fantasy-atlas
description: 空想上・歴史上の世界地図(架空の国、ある帝国の最大版図、勢力図など)を色分けして作りたい時に使うスキル。「〇〇帝国の地図を作って」「この設定の世界地図を塗って」「ここを併合した後の地図」のような依頼、または既存の lovingthenhk-cpu/fantasy-atlas サイトの更新・再デプロイを頼まれた時に使う。国単位・州/省単位・自由な形のどれでも塗れるテンプレートが既に GitHub Pages にデプロイ済みなので、ゼロから地図を作るのではなく必ずこのテンプレートを編集して使う。
---

# fantasy-atlas

lovingthenhk-cpu/fantasy-atlas という既存リポジトリ(GitHub Pages で公開済み)を
テンプレートとして使い、世界地図に色を塗って再デプロイするためのスキル。

公開URL: https://lovingthenhk-cpu.github.io/fantasy-atlas/

新しくHTMLやSVGを書き起こす必要はない。`index.html` 冒頭の「設定ゾーン」だけを
書き換えて push すれば、その内容が地図に反映される。

## 前提: 認証

github-pages-deploy スキル(`/mnt/skills/user/github-pages-deploy/SKILL.md`)に書かれている
認証復元コマンドと同じトークンを使う。このファイルにはトークンを直接書かない
(publicリポジトリの secret scanning に弾かれるため)。認証手順は必ず
github-pages-deploy スキルの方を参照すること。

## 手順

### 1. 既存リポジトリを clone する(新規作成しない)

```bash
rm -rf /home/claude/fantasy-atlas
git clone https://${GH_TOKEN}@github.com/lovingthenhk-cpu/fantasy-atlas.git /home/claude/fantasy-atlas
cd /home/claude/fantasy-atlas
ls
# index.html / regions.topo.json / region-index.json / country-index.json
```

### 2. 塗りたい地域の ID を調べる

- **国単位**で塗りたい場合 → `country-index.json` で国名から `adm0`(ISO 3166-1 alpha-3)を引く
- **州・省単位**で塗りたい場合 → `region-index.json` で `id`(だいたい ISO 3166-2、例: `US-CA`)を引く
  - `region-index.json` は約4,400件(Natural Earth admin-1、全世界カバー)なので `jq`/`grep` で絞り込む

```bash
# 国名で adm0 コードを調べる例
jq '.[] | select(.country | test("United Kingdom|India|Canada|Australia"; "i"))' country-index.json

# 州・省名で id を調べる例
jq '.[] | select(.region | test("California"; "i"))' region-index.json
```

見つからない/表記ゆれが不安な場合は `.country` や `.region` の部分一致で検索し直す。
adm0 は Natural Earth 由来で一部の係争地域などは通常のISO3と異なることがあるので、
不安なら1件 jq で引いてから目視確認すること。

### 3. `index.html` の設定ゾーンを編集する

`<script>` タグの先頭、`▼▼▼ 設定ゾーン` 〜 `▲▲▲ 設定ゾーンここまで` の間だけを編集する。
それより下の描画ロジックは通常触らない。

```js
const MAP_TITLE    = "大英帝国";                      // 見出し
const MAP_SUBTITLE = "最大版図(1920年ごろ)";           // 副題

const REGION_COLORS = {
  // 州・省単位で塗る。個別指定はCOUNTRY_COLORSより優先される
  // "US-CA": "#8b2e2e",
};

const COUNTRY_COLORS = {
  // 国単位でまとめて塗る(キーはadm0 = ISO3166-1 alpha-3)
  "GBR": "#8b2e2e", "IND": "#8b2e2e", "CAN": "#8b2e2e", "AUS": "#8b2e2e",
  "NZL": "#8b2e2e", "ZAF": "#8b2e2e", "EGY": "#8b2e2e", "NGA": "#8b2e2e"
};

const FREEHAND_SHAPES = [
  // 行政境界を無視して自由な形(空想の国境線など)。points は [経度, 緯度] の配列
  // { color: "#5b4b8a", label: "幻の王国", points: [[139.0,36.0],[141.0,36.5],[140.5,38.0]] }
];

const LEGEND_LABELS = {
  // 色 → 凡例に出すラベル名
  "#8b2e2e": "大英帝国 最大版図(1920年)"
};
```

編集は str_replace で該当ブロックだけをピンポイントに書き換える。CDN読み込み部分
(`<script src="https://cdn.jsdelivr.net/...">`)やそれ以降のロジックは触らない。

### 4. デプロイ(push するだけ。Pages設定は既に有効)

```bash
cd /home/claude/fantasy-atlas
git add -A
git commit -m "update map"
git push origin master
```

新しいリポジトリを作る必要は無い。Pages は既に有効化されているので、push すれば
数十秒〜数分で https://lovingthenhk-cpu.github.io/fantasy-atlas/ に反映される。
反映確認は `curl` で 200 が返るか、または少し待って再取得する。

### 5. ユーザーに結果を伝える

公開URLをそのまま案内する。スクリーンショットで確認したい場合は Playwright 等が
使えるならレンダリングして見た目を確認してから伝えるとよい(証明書エラーが出る
サンドボックス環境では `ignore_https_errors=True` を使う)。

## 注意点

- **UIはビューアー専用**(クリックで塗る等の機能は無い)。塗り分けは必ずこの設定ゾーンの編集で行う。
- 州・省の網羅性は Natural Earth 10m admin-1 ベースなので、非常に細かい行政区画(市区町村など)は無い。
  それより細かい粒度が要る場合は `FREEHAND_SHAPES` で座標指定して自由に塗る。
- `REGION_COLORS` に指定がある州は `COUNTRY_COLORS` の指定より優先される
  (例: 国全体は塗るが一部の州だけ別色、という表現が可能)。
- 大きめの `regions.topo.json`(約1.4MB)を毎回全部読み込んで確認する必要は無い。
  地域IDの特定は `region-index.json` / `country-index.json` の grep/jq で十分。
- リポジトリ名は `fantasy-atlas` 固定。複数の地図を同時に作りたい等の要望が
  出た場合はユーザーに相談してから別リポジトリ名にするかブランチを分けるか決める。
