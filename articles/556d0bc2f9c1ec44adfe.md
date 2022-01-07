---
title: "Rust製Fuzzy Finderのskimを自作CLIツールに組み込んでみる"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust"]
published: true
---

Rust で書かれた`skim`では Go で書かれている`fzf`のように曖昧検索ができます。`cargo install skim`でインストールし`sk`コマンドで利用できるようになります。

https://github.com/lotabout/skim

この `skim` はライブラリとしても利用できます。今回は自作のブックマーク管理 CLI ツールに `skim` を組み込んでみました。

## 作ったもの

こちらです。

https://github.com/urawa72/rbm

自分がよく閲覧する Web の記事のリンクをローカルで管理できるCLIツールです。ブックマークは`~/rbm-bookmarks.toml`というファイルに保存されます。
はてブとかブラウザのブックマークでも良いのですが、自分はVimを使って作業することが多いためいちいちブラウザに行かずにターミナルからさくっと開きたい一心で作りました。

[crate.io](https://crates.io/crates/rbm)にも登録しているので`cargo install rbm`でインストールできます。

`rbm add`コマンドで新規ブックマークを追加します。

```bash
$ rbm add
Title> Google
URL> https://google.com
Tag> Google,Search
```

`rbm list`で登録したブックマークの一覧を表示し、曖昧検索で絞りこんで選択したブックマークをブラウザで開くことができます。ここで`skim`をライブラリとして使用しています。

## ライブラリとして利用している部分のコード

ブックマーク管理 CLI ツールで`skim`を用いて曖昧検索をしている部分のコードは下記となります。
（CLI ツール自体の作りや toml ファイルに保存する部分については割愛します。詳細は GitHub をご確認ください。）

```rust
use crate::bookmark::Bookmark;
use skim::prelude::*;
use skim::{Skim, SkimItemReceiver, SkimItemSender};
use std::{process::Command, sync::Arc};

impl SkimItem for Bookmark {
	  // 曖昧検索に使われる値
    fn text(&self) -> Cow<str> {
        Cow::Borrowed(&self.inner)
    }

    // 絞り込んで選択した際に表示される値
    fn output(&self) -> Cow<str> {
        Cow::Borrowed(&self.url)
    }
}

/// Search bookmark with fuzzy finder.
pub fn finder(bookmarks: Vec<Bookmark>) {
	  // 曖昧検索の起動時の画面高さを指定・Enterキーで選択する
    let options = SkimOptionsBuilder::default()
        .height(Some("50%"))
        .multi(true)
        .bind(vec!["Enter:accept"])
        .build()
        .unwrap();

    let (tx_item, rx_item): (SkimItemSender, SkimItemReceiver) = unbounded();
    for bookmark in bookmarks {
        let _ = tx_item.send(Arc::new(bookmark));
    }
    drop(tx_item);

    let selected_items = Skim::run_with(&options, Some(rx_item))
        .map(|out| match out.final_key {
            Key::Enter => out.selected_items,
            _ => Vec::new(),
        })
        .unwrap_or_else(Vec::new);

    for item in selected_items.iter() {
        let url = item.output();
        // 曖昧検索の結果出力されるurlをopenコマンドで開く
				// Macだけで動く
        Command::new("open")
            .arg(url.as_ref())
            .output()
            .expect("Faild to execute process");
    }
}
```

このコードは`skim`のリポジトリの`examples`内にある[custom_item.rs](https://github.com/lotabout/skim/blob/master/examples/custom_item.rs)と[custom_keybinding_actions.rs](https://github.com/lotabout/skim/blob/master/examples/custom_keybinding_actions.rs)のコードを参考にしています。

正直内部の仕組みを理解して使いこなせているわけではありませんが、割と簡単に自作 CLI ツールに曖昧検索を組み込むことができます。

## ちなみに

今回作ったブックマーク管理アプリは Microsoft が公開している Rust の勉強教材（[コマンドラインの to-do リスト プログラムを作成する](https://docs.microsoft.com/ja-jp/learn/modules/rust-create-command-line-program/)）を改造して作りました。

この教材は私のような Rust 勉強中の者にとっては非常にためになります。
