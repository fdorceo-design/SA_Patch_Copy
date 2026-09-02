# SA Patch Copy

Spitfire Audio系音源のパッチ名をクリックだけで完成済みトラック名としてコピーし、DAW(Studio One等)へ手動で貼り付けるためのローカルHTMLツールです。タイピングを極力なくすことが目的です。

**公開ページ:** https://fdorceo-design.github.io/SA_Patch_Copy/

## 使い方

1. 上記ページを開く(またはリポジトリの `index.html` をローカルで開く)
2. 左側でライブラリを選択
3. 右側のパッチ名をクリックすると `LibraryPrefix - PatchName` の形式でクリップボードへコピーされる
4. DAWへ手動で貼り付け

## 特徴

- 完全ローカル、通信不要(GitHub Pagesでの公開時も外部通信なし)
- 外部CDN不要
- 表示設定・折り畳み状態はlocalStorageに保存
- 公式データ(公式サイト / マニュアル / アプリ内階層)をパッチ名・分類・並び順の正本とし、推測では作成しない

## 開発状況

Spitfire Audio / Originals / SA Recordingsの全所有ライブラリを対象に、公式情報源から1ライブラリずつパッチデータを整備中です。Kontaktとの統合は今後対応予定です。
