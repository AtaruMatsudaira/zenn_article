---
title: "C#でShell芸人を目指す"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: 
  - "csharp"
  - "dotnet"
  - "shell"
published: true
published_at: "2023-12-02 00:00"
---

# はじめに

この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の2日目です。

皆さんのPJにはPython2やPerl,Js,ShellScriptなどで作られた秘伝のタレはありますか？

よくある秘伝のタレ

- CIツール
    - ビルドしたapkやipaをAppCenterなどにPush
- ビルドマシンのブランチお掃除
    - アプリまわり
    - バージョン表記
    - デバッグ用Appのアイコン自動生成
- その他
    - TODO列挙

などなど...割と言語バラバラで作られてることが多いです。
そのため、言語を何かしらに統一したほうがいいと筆者は考えています。

今回の記事を読んでいただいて、選択肢にC#が加えてくれたらうれしいと思います。

# でもC#ってめんどくさくない？

誰かの心の声 : 「C#でCLIツール？でもめんどくさいんでしょ？
Hello World 実行するだけでもこれぐらいかかるし...」

```csharp
using System;

namespace ConsoleApp1
{
		public class Program
		{
			 public static void Main(string[] args)
			 {
					 Console.WriteLine("Hello world");
			 }
		}
}
```

誰かの心の声 : 「 System.Diagnostics.Processだって何か使いづらいし、
オプション引数だってやること多いし...C#でCLIツールなんて
正気とは思えナ…」

**.NET6 (トップレベルステートメント)** と **ProcessX**、**ConsoleAppFramework**を組みあわせることですべてがうまくいきます！！！

# トップレベルステートメントとは

## 概要

.NET6 から導入されたトップレベル(クラスや名前空間よりも外側)に処理を書ける機能です
https://learn.microsoft.com/ja-jp/dotnet/csharp/tutorials/top-level-statements

.NET6以降でコンソールアプリケーションを作成するとProgram.csは以下のようになります。

```csharp
// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, World!");
```

RiderのIL Viewerの(low-level C#)を見ると、ProgramクラスとMainメソッドの中で動いているのがわかるります。

```csharp
using System;
using System.Runtime.CompilerServices;

[CompilerGenerated]
internal class Program
{
    private static void <Main>$(string[] args)
    {
        Console.WriteLine("Hello World");
    }

    public Program()
	{
		base..ctor();
	}
}
```

## 注意点

PythonやJSっぽくかけるので動的言語に見えますが、C#なのでしっかり静的言語です。
Mainメソッドに展開される関係上、処理部分が上、定義部分が下と、分けて書く必要があります。

![error](/images/adcal_2023_error.png)

添付の画像は上部でクラスを定義して、下部でステートメントを記述しているので怒られています。

# ProcessX

## 概要

Cysharp製の外部Processを簡単にたたけるライブラリです。
リリース初期はC#8.0の非同期処理でいい感じに扱えるのを売りにしていました。
(標準のSystem.Diagnostics.ProcessはC#1.0の時代にできたモノなのでAPIが今だと扱いにくい)

```csharp
using Cysharp.Diagnostics; 

await foreach (string item in ProcessX.StartAsync("dotnet --info"))
{
	Console.WriteLine(item);
}
```

https://neue.cc/2020/01/30_590.html

そのうち、google/zx のオマージュでかなりカジュアルにShellを書ける Zx がリリースされました。
StringクラスにGetAwaiterメソッドが追加され、コマンドをawaitするのがとても簡単になりました。

```csharp
using Zx;
using static Zx.Env;
string items = await "dotnet --info";
log(items);
```

https://neue.cc/2021/08/23_602.html

# ConsoleAppFramework

## 概要

Cysharp製の.NET向けのCLIツール向けライブラリです。
CLIツールのオプション周りを数行で書けたりと何かと便利な点が多いです。
昔は [MicroBatchFramework]([https://tech.cygames.co.jp/archives/3241/](https://tech.cygames.co.jp/archives/3241/)) という名前で出していました。(要はリブランディング)

例えば下記のソースだとラムダ式の引数で定義したeがそのままオプションとして指定できます。

```csharp
await ConsoleApp.RunAsync(args, (string e) => Console.WriteLine(e));
```

```bash
$ dotnet run --e 焼肉定食
焼肉定食
```

また、helpや引数名の省略記法なども、Option属性をメソッドの引数につけることで簡単に実装できます。

```csharp
await ConsoleApp.RunAsync(args, MainAsync);

async Task<int> MainAsync([Option("path", "走査するディレクトリ")] string rootPath,[Option("ex", "拡張子")] string extensions = "cs")
{
}
```

```bash
$ dotnet run help
Usage: GitTodo [options...]
Options:
 -path, --root-path <String> 走査するディレクトリ
(Required)
 -ex, --extensions <String> 拡張子 (Default: cs)
Commands:
 help Display help.
 version Display version.
```

実行するメソッドの戻り値が `int` または `Task<int>` の場合、それが終了コードとなります。これは ConsoleAppFrameworkライブラリが提供する機能の一部です。この機能を使用することで、CLIツールの実行結果をエラーコードとして返すことができます。

たとえば、以下のようなメソッドを実行する場合を考えてみましょう。

```csharp
await ConsoleApp.RunAsync(args, MainAsync);

async Task<int> MainAsync(int num)
{
    if (num % 2 == -1)
    {
        return 1;
    }

    return 0;
}

```

この場合、`MainAsync` メソッドの戻り値が奇数の場合は終了コードとして `1` を返し、偶数の場合は `0` を返します。終了コードは、CLIツールの実行結果を表す重要な情報であり、他のプログラムやシステムとの連携に役立ちます。

ConsoleAppFrameworkには、このような終了コードの制御を簡潔に記述するための便利な機能が備わっています。これにより、CLIツールの振る舞いを柔軟に制御することができます。また、終了コードを適切に設定することで、エラーハンドリングや自動化のプロセスにおいて正確な情報を提供することができます。

以上が、`int` または `Task<int>` の戻り値を持つメソッドが終了コードとして扱われる仕組みについての説明です。CLIツールの開発において、終了コードの制御は重要な要素となりますので、適切に活用することをおすすめします。

# サンプル

上記3つを組み合わせて簡単なサンプルを作ってみました。

TODO コメントを残して放置した人をあぶり出します。

流れは

1. .cs ファイルを一覧で取得する
2. TODOコメントが書いてある行だけ抽出する
3. git blame でその行のコミット情報を取得
4. コミット情報からコミットAuthorを取得
5. まとめて取得

```csharp
using System.Text;
using Cysharp.Diagnostics;
using Zx;
using static Zx.Env;

var br = Environment.NewLine;
var sb = new StringBuilder();

await ConsoleApp.RunAsync(args, MainAsync);

async Task<int> MainAsync(
    [Option("path", "走査するディレクトリ")] string rootPath,
    [Option("ex", "拡張子")] string extensions = "cs")
{
    if (File.Exists(rootPath))
    {
        log("pathがファイルでした。ディレクトリを指定してください", ConsoleColor.Red);
        return 1;
    }

    await $"cd {rootPath}";

    var csPaths = await $"find {rootPath} -type f -name '*.{extensions}'";
    foreach (var csPath in csPaths.Split(br))
    {
        string grepResults;
        try
        {
            grepResults = await $"grep -in '[^a-zA-Z]TODO[^a-zA-Z]' '{csPath}'";
        }
        catch (ProcessErrorException)
        {
            continue;
        }

        foreach (var grepResult in grepResults.Split(br))
        {
            var index = grepResult.IndexOf(':');
            var lineNum = grepResult.Substring(0, index);
            var content = grepResult.Substring(index + 1);
            string blameInfos;

            try
            {
                blameInfos = await $"git blame -L {lineNum},+1 {csPath} --porcelain";
            }
            catch (ProcessErrorException)
            {
                continue;
            }

            var author = blameInfos.Split(br).Where(info => info.StartsWith("author")).FirstOrDefault("");
            sb.AppendLine($"{csPath}:{lineNum}:{author}");
            sb.AppendLine(content);
            sb.AppendLine();
        }
    }

    log(sb.ToString(), ConsoleColor.Yellow);
    return 0;
}
```

https://github.com/AtaruMatsudaira/GitTodo

# まとめ


C#を使って開発すると、Linqを活用することができて楽しいです！Linqを使うことで、データのクエリや変換を簡潔に記述することができます。

また、async/awaitを使ってC#を書くと、コードが読みやすくなります。非同期処理をシンプルに表現できるため、コードの流れが明確になります。

さらに、JenkinsでGroovyの処理がベタ書きされている場合、C#に切り出すことでActionsに移行する際に便利です。C#の力を借りて、効率的な移行を実現しましょう。

C#にはWindowsでしか動かないライブラリもありますが、Xamarinで使用されているライブラリは基本的にマルチプラットフォーム対応しています。そのため、Xamarinを活用することで、さまざまなプラットフォームでアプリケーションを動かすことができます。

Cysharpに足を向けて寝れそうにないですね！

明日は、〇〇さんの[XXXX](YYYY)となります！