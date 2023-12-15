---
title: "OS関係なくAirDrop的にファイルを共有したい"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode"]
published: false
---

# はじめに

この記事は四工大アドベントカレンダーの16日目の記事です。
四工大の一角をなす電大、またの名をTDUというTokyo Denki University

# やること

AirDrop、便利ですよね。
iPhoneからだと画像を共有するのがメインだと思いますが、Macだとファイルだったら何でもOKなのでzipファイルやshファイルなども遅れちゃいます。

ですが、AirDropはあくまでApple経済圏のみで使えるサービスなので、LinuxやWindowsユーザにデータを共有する際には別の手段をとらねばなりません。

前置きが長くなりましたが、今回はVSCodeの拡張機能としてあるLive Serverをファイル共有ツールとして利用してみたいと思います。

# Live Serverとは

MS公式からリリースされているVSCode向けの拡張機能の一つです。
作業しているディレクトリをローカルネットワーク内で展開することができます。

![Alt text](/images/global_aridrop_2.png)

ホットリロードにデフォルトで対応しているので、HTMLのプレビューなどのWeb開発で主に使用されています。

今回はディレクトリをローカルネットワーク内で展開するという点を利用して、ローカルネットワーク内にこのディレクトリ群を共有していきたいと思います。


# やりかた

VSCodeを入れます
最近だとCLI経由でいれれるので、そっちを使いましょう。
Debian系以外のLinuxを普段使いしてる人は逸般の方だと思われるので説明は致しません。(というかこの記事を読んでいないと思います。)

Windows
```
$ winget install visual-studio-code
```

Mac
```
$ brew install visual-studio-code
```

Debian系Linux
```
$ sudo apt install visual-studio-code
```

VSCodeをぶち込めたら下記のVisualStudio Marketplaceにアクセスしましょう。

https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer

"Install" をクリックするとなんや感やでVSCodeに遷移します。

VSCodeの遷移先でLive Serverのインストール画面がまたでてくるので、またインストールしましょう。

![Alt text](/images/global_airdrop_1.png)

インストールができたら右下に Go Liveなるものができあがっています。
このGo Liveをクリックしましょう。

![Alt text](/images/global_aridrop_3.png)

クリックすると、冒頭のこれになります。

![Alt text](/images/global_aridrop_5.png)

今回はこの画面を同一ネットワーク内にある端末内で出して、ファイルを共有したいと思います。

しかし、デフォルトだとURLが ``localost:5500`` or ``127.0.0.1:5500`` になります。

このURLを共有してもほかの端末では何も表示されません。

そのため、ローカルネットワーク内でのIPがURLになるようにしましょう。

![Alt text](/images/global_aridrop_4.png)

