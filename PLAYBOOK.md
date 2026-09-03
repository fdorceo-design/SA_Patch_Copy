# Patch Copy アプリ 制作手順・注意点(雛形)

SA Patch Copy (Spitfire Audio版) の制作過程で確立した、他の音源メーカー版 (NI版、EastWest版など) を作る際にも使えるプレイブック。

## 1. アプリの基本設計

- **単一HTMLファイル**。ビルド不要、外部CDN不使用、完全ローカル動作。
- UI状態(非表示ライブラリ、折りたたみ状態など)は `localStorage` に保存。
- データは1つの巨大な `const DATA=[...]` 配列(1行)に集約。

### DATAスキーマ

```
{
  section: string,          // 大分類 (例: "Kontakt", "Spitfire Audio", "Originals", "SA Recordings")
  displayName: string,      // ライブラリの表示名
  copyPrefix: string,       // コピー時に使う短縮名 (スペースを除いた製品名など)
  patches: [{name, raw}],   // フラットな全パッチ一覧 (groups/unitPatchesの内容と必ず一致させる)
  groups: null | [{name, unit, patches:[{name,raw}]}],  // 単一製品内のカテゴリ分け
  units: [string],          // 複数ユニット(サブ製品/セクション)がある場合の並び順
  unitPatches: {unitName: [{name,raw}]},  // groupsを使わず直接ユニットにパッチを紐付ける場合 (バンドル製品のサブ製品など)
  sourceType: "official" | "official-manual" | "official-catalog" | "local-scan" | "local-scan-verified",
  catalogOnly?: true         // データ未取得のプレースホルダー。trueなら patches は空配列
}
```

- `groups` と `unitPatches` は併用可能(例: Abbey Road One は主要オーケストラを `groups` で持ち、Grand Brass等のバンドル内サブ製品を `unitPatches` で持つ)。
- **重要な不変条件**: `lib.patches` は常に `groups` の全パッチ + `unitPatches` の全パッチを結合した「正しい合計」と一致させること。ズレるとサイドバーの件数と実表示が食い違うバグになる(詳細は6章)。

## 2. データ収集の優先順位(絶対に守るルール)

1. **公式マニュアルのAppendix**(Articulation List / Techniques・Mics・Mixes 等) — 最も信頼できる一次情報源。
2. **公式製品ページのプリセット一覧** — マニュアルより詳しい場合がある(eDNA Earth, Kinematik, Ambient Guitars, Jupiter by Trevor Hornなど、製品ページに全プリセット名がテキストで載っているケース)。
3. **ユーザー提供のローカルスキャンデータ**(実機の `.nki` 等のファイル名一覧)。これが最も確実な場合もある。
4. **上記いずれにも情報が無ければ「保留」とし、絶対に推測・捏造しない。** カテゴリ名の説明だけ、UIスクリーンショットの一例だけ、という場合はアウト。

### 捏造してしまった実例(教訓)

Woodwind EvolutionsのPDFが大きすぎて `Read` ツールが正しく処理できず、実際には中身を見れていなかったにもかかわらず、他の類似製品(Symphonic Strings Evolutions等)のパターンから「もっともらしい」24項目のリストを提示してしまった。後から気づいて破棄・訂正した。

**教訓**: `Read` ツールでPDFを読んだ際、返答が実際のテキスト内容を含んでいるか毎回確認すること。`[media removed: request limit]` のような表示は「中身を見れていない」サイン。この状態で内容を語ってはいけない。確信が持てない場合は、あらためて別の抽出手段(3章参照)を試すか、正直に「確認できていない」と申告して保留にする。

## 3. PDFマニュアル取得の実践手順

1. 製品ページを `WebFetch` し、「Download user manual」リンクの実URLを聞き出す。相対パス `//www.spitfireaudio.com/cdn/shop/files/....pdf` 形式や `cloudfront.net/p/files/product-manuals/.../....pdf` 形式が多い。
2. そのPDF直リンクを `WebFetch` する。ほぼ必ず「バイナリで読めない」という応答になるが、**副産物としてPDFがローカルに保存される**(保存先パスがエラーメッセージ内に明記される)。
3. 保存されたPDFを `Read` ツールで読む。画像として解析されるが、実際にテキストが返ってくることが多い。
4. **ファイルサイズが大きい(目安15MB超)場合、`Read` が `[media removed: request limit]` を返し中身が見れないことがある。** その場合の代替手順:
   - `curl -sL <url> -o file.pdf` で直接ダウンロード
   - 初回のみ `npm install pdf-parse --no-save` (スクラッチパスに一時インストール)
   - 新しい `pdf-parse` (v2系) は `PDFParse` クラスを使う:
     ```js
     const { PDFParse } = require("pdf-parse");
     const parser = new PDFParse({ data: fs.readFileSync("file.pdf") });
     const result = await parser.getText();  // result.text, result.total(ページ数)
     ```
   - 抽出したテキストを `grep -n -i "appendix"` 等で検索し、該当セクションを `Read` でオフセット指定して確認する。
5. Appendix内に**個別パッチ名の完全な一覧**があるかを必ず確認する。無い場合(カテゴリ説明文だけ、UIスクショの一例だけ等)は保留にする。

## 4. データ構造化のワークフロー

- Node.js の使い捨てスクリプトで DATA 配列を読み書きする(巨大な1行JSONを手編集しない)。
- 典型的な手順:
  1. 抽出した公式データをスクラッチパスに `xxx_data.json` として書き出す(`[{unit, groups:[{name, patches:[...]}]}]` 形式)
  2. `apply_lib.js` のようなヘルパーで、指定ライブラリ名にそのデータを適用し、`sourceType` を設定、`catalogOnly` を解除
  3. 適用後、件数をログ出力して人間の目で確認(公式資料の「◯◯プリセット」という記載と一致するか照合すると精度が上がる)
  4. DATA配列全体を `JSON.parse` して構文検証してからコミット
- 巨大なプリセットリスト(500件超)は `Write` で一括作成、追記が必要なら `Edit` で分割して追加する。

## 5. 分類・重複排除ルール

- **プレイヤー/プラグイン要否の判定**: Kontakt(Player)必須の音源か、メーカー独自プラグイン(native)かをWebSearchで個別に確認。後で全体を監査し、誤分類がないか再チェックする。
- **上位互換の重複排除**: Professional版やSuperset版が存在する場合、下位版(Core版など)は削除する。
- **バンドル内重複の排除**: バンドル製品に含まれるサブ製品が単体でも別枠に存在する場合、重複させない。

## 6. レンダリングバグの注意点(今回発見・修正したもの)

1. **`group.unit` と `units[]` の不一致**: グループの `unit` フィールドの値が `units[]` 配列の要素と完全一致していないと、そのグループは描画時に除外される。サイドバーの件数計算はこのフィルタを通さず全グループを合算するため、**件数は合っているように見えるのに詳細画面が空白**という発見しづらいバグになる。
   - 定期的に以下のようなチェックスクリプトで全ライブラリを走査すること:
     ```js
     DATA.forEach(lib=>{
       if(!lib.groups?.length) return;
       const units = lib.units?.length ? lib.units : [lib.displayName];
       const groupUnits = new Set(lib.groups.map(g=>g.unit||lib.displayName));
       const missing = [...groupUnits].filter(gu=>!units.includes(gu));
       if(missing.length) console.log(lib.displayName, missing);
     });
     ```
2. **サイドバー件数の計算漏れ**: `unitPatches` 形式を使っているライブラリ(バンドル製品など)で、サイドバーの件数計算が `groups` だけを合算し `unitPatches` を見落とすと過小表示になる。両方を合算するよう修正が必要。
3. **孤立ユニット**: `units[]` に名前が列挙されているのに、対応する `groups` エントリも `unitPatches` エントリも存在しないケース。この場合そのユニットは「空欄」にすら見えず、静かに描画されない。以下でチェックできる:
   ```js
   DATA.forEach(lib=>{
     const hasData = lib.groups?.length || (lib.unitPatches && Object.keys(lib.unitPatches).length);
     if(!hasData || !lib.units?.length) return;
     const groupUnitSet = new Set((lib.groups||[]).map(g=>g.unit||lib.displayName));
     const unitPatchSet = new Set(Object.keys(lib.unitPatches||{}));
     const orphans = lib.units.filter(u=>!groupUnitSet.has(u) && !unitPatchSet.has(u));
     if(orphans.length) console.log(lib.displayName, orphans);
   });
   ```
   → ユーザーへ「未所有ライブラリで後回しにしていただけかもしれない」と確認した上で、公式データで埋めるか削除するかを判断する。

## 7. Git運用

- データ変更のたびに `commit` + `push` する(バッチしない。1ライブラリ1コミットが基本)。
- コミットメッセージには**情報源を明記**する(例: 「公式マニュアルのAppendix C」「製品ページのプリセット一覧」「ユーザー提供のローカルスキャン」など)。数字の一致確認(「マニュアル記載の◯◯個と一致」)も書くと後から検証しやすい。

## 8. 進め方のペース

- ユーザーの指示に応じて「小さい順」「大きい順」「N件ずつ」など柔軟にバッチサイズを変える。
- 保留リスト(データ源が無く埋められなかったもの)は都度報告し、進捗を可視化する。
- セッションが長時間に及ぶ場合、時々「残り件数」を集計して見える化すると良い。

## 9. 他メーカー版を作る際のチェックリスト

- [ ] 対象メーカーの製品分類(Kontakt必須/独自プラグイン/DAW内蔵 等)を最初に整理する
- [ ] 命名規則(コピー時のプレフィックス)を決める
- [ ] 大分類(section)をメーカーの実際の製品ラインナップに合わせて設計する
- [ ] マニュアルPDFの入手経路(製品ページのリンクパターン)を早めに把握する
- [ ] 大きいライブラリ・小さいライブラリのサイズ一覧を作り、ユーザーと相談してバッチ順を決める
- [ ] 本チェックリストの6章のバグパターンを、実装直後に一度全ライブラリに対して走らせておく
