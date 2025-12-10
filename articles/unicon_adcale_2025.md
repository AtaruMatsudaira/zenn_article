---
title: "UnityEditorのアイコンを画像に差し替えるOSS「Unicon」を作ってみた"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "Mac", "Windows"]
published: true
---

# 初めに

この記事は [QualiArts アドベントカレンダー 2025](https://qiita.com/advent-calendar/2025/qualiarts)の 10 日目です。

# 前書き

最近、ClaudeCode などの AI エージェントに自走させていると、並列で複数の Unity Editor を立ち上げる機会が多くなりました。

以前から作業プロジェクトと基盤プロジェクトなどの 2 エディタくらいを並列で開くことはあったのですが、AI エージェントを併走させるとなると平気で 3、4 個同時に立ち上げることになり、判別がかなり難しくなったと感じます。

そのため、次の gif のように Unity のアイコンを任意の画像に差し替えられる OSS を作成してみました。マサカリ/PR 大歓迎です！

![image](/images/unicon_adcale_2025/unicon_1.gif)

https://github.com/AtaruMatsudaira/Unicon/

# 使い方

下の項目で実装方法を書いていくのですが、とりあえずアイコンを差し替えたいという方は、[GitHub の README](https://github.com/AtaruMatsudaira/Unicon/blob/main/Packages/com.mattun.unicon/README_ja.md#%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB)を参考にインストールしてください。

OpenUPM と Git 形式に対応しています。

:::details 余談
OpenUPMでの月間インストール数、400ありがとうございます！！
https://openupm.com/packages/com.mattun.unicon/
![](/images/unicon_adcale_2025/open_upm.png)
:::

インストールしたら、Preferences から Unicon の項目を選び、

1. Enable Custom Dock Icon を有効化（赤枠）
2. 任意の項目を設定（緑枠）
   - 画像/OverlayColor/テキストをそれぞれ設定できます
3. Apply Current Settings をクリックすれば反映されます

![image](/images/unicon_adcale_2025/preference.png)

実行時やコンパイル時に一瞬だけ消えることもありますが、少し経てば設定した画像に戻ります。

# 内部実装

## 全体アーキテクチャ

以下の図は、Unity Editor から各プラットフォーム実装までの流れを示したものです。
C#層で抽象化を行い、各プラットフォームネイティブ実装でアイコンを更新しています。

```mermaid
flowchart TD
    UE["Unity Editor"]
    UI["UniconInitializer<br>[InitializeOnLoad]"]
    NM["NativeMethods<br>(プラットフォーム抽象層)"]

    subgraph Platforms["プラットフォーム実装"]
        direction LR
        MAC["macOS Native"]
        WIN["Windows Native"]
        DMY["Dummy"]
    end

    subgraph macOS側["macOS側の実装"]
        SW["Swift Plugin<br>(.bundle)"]
        NSA["NSApplication"]
        DOCK["macOS Dock"]
    end

    subgraph Windows側["Windows側の実装"]
        CPP["C++ Plugin<br>(.dll)"]
        SHELL["Windows Shell API"]
        TASK["Windows Taskbar"]
    end

    UE --> UI --> NM
    NM --> MAC
    NM --> WIN
    NM --> DMY

    MAC -. "P/Invoke" .-> SW
    WIN -. "P/Invoke" .-> CPP

    SW --> NSA --> DOCK
    CPP --> SHELL --> TASK
```

## MacOS での実装

### ネイティブ上でのアイコンの扱い

まず前提として、macOS は標準で提供されている Dock アイコン上でのプログレスバーという機能が特に存在しません。

Unity のコンパイルや Import 時のプログレス表示は、標準の Unity アイコンの上に Progress 表示が入った画像を用意して、それを進捗に合わせて表示していると推察されます。

そのため今回のアプローチとしては、設定種別 (画像/オーバーレイ/テキスト)に関わらず、内容に応じた[NSImage](https://developer.apple.com/documentation/appkit/nsimage)のインスタンスを作成し、[NSApplication.shared.applicationIconImage](https://developer.apple.com/documentation/appkit/nsapplication/applicationiconimage) に設定することとしました。

また、カスタム画像が設定されていない場合、ユーザーが期待するのはデフォルトのアイコンの上にテキストなどが書き足されている状態だと思います。
そのため、デフォルトの画像は `<UnityEditorアプリケーションのパス>/Contents/Resources/UnityAppIcon.icns` を参照するようにしました。

NSApplication.shared.applicationIconImage のドキュメントでは

> Assign an image to this property when you want to **temporarily** change the app icon in the dock app tile. The image you provide is scaled as needed so that it fits in the tile. To restore your app’s original icon, set this property to nil.

とあるので、任意の画像をここに設定すればよさそうですし、リセットしたいときは `nil` を設定すれば良さそうです。

これらを行う関数を Swift で実装して cdecl で公開し、ビルドした bundle ファイルを Unity プロジェクト上に配置して実行したところ、アイコンが変更できることを確認できました。

### Unity 側の画像の扱い

とりあえず、任意の画像を設定できるようになりました。

ただし、

- "temporarily change the app icon in the dock app tile" とあるように、一時的にアイコンを差し替える仕様である
- Unity のプログレス表示も上記と同じ方法で実現している

ということもあり、InitializeOnLoad のタイミングで一度だけアイコンを設定しても、ロードが走るたびにすぐアイコンが元のものに戻ってしまうという問題がありました。

そのため、macOS のエディタ上では

- 1 秒ごと（InitializeOnLoad 時に `EditorApplication.update` に引っかける）
- コンパイル後（[DidReloadScripts](https://docs.unity3d.com/ScriptReference/Callbacks.DidReloadScripts.html) 属性に引っかける）

と、頻繁にアイコンを設定することで Unity 側から上書きされても即座に元のアイコンに戻るようにしました。

## Windows での実装

元々、Unicon は macOS のみのサポートを予定していました。
OSS として v1.0.0 を公開したところ、私と同じくサイバーエージェントグループの Applibot に所属する同期の [焼き鯖くん](https://x.com/yakisabananoka)が Windows 対応の PR を出してくれました。

詳しくは焼き鯖くんのアドカレで投稿があるとのことなので、この記事では簡単に解説します。

### ウインドウのアイコンの扱い

Windows は macOS と異なり、アイコンは各ウィンドウごとに独立しています。
そのため、各ウィンドウに対してそれぞれアイコンを設定する必要があります。

今回のアプローチとしては、Win32 API の [WM_SETICON](https://learn.microsoft.com/ja-jp/windows/win32/winmsg/wm-seticon) メッセージをそれぞれのウィンドウに送信する手法を採用しました。

### タスクバーでのアイコンの扱い

通常 Unity では [AUMID](https://learn.microsoft.com/ja-jp/windows/win32/shell/appids) が未設定のためシステムで定義された AUMID が使用される関係で、半強制的にアプリ固有のアイコンがタスクバーで使用されていました。
そのため、タスクバー上でアイコンをグループ化するために、各Unityごとに独立したAUMIDを設定した上で、任意のアイコンを設定しています

https://github.com/AtaruMatsudaira/Unicon/blob/fd24ba2669d58f1a70e08b2131b91b559e99ce22/Packages/com.mattun.unicon/Editor/Unicon/Internal/WindowsNativeMethods.cs#L133-L137

これにより、同じ Unity Editor でもプロジェクトごとに別のアプリケーションとして認識され、タスクバー上で分離して表示されるようになります。

# 最後に

以上で、QualiArts アドベントカレンダー 2025 の 10 日目の記事を締めたいと思います。

Unicon が皆さんの並列作業の一助となれば幸いです。

ここまで読んでいただき、ありがとうございました！
