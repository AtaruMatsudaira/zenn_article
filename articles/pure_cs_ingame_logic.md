---
title: "PureなC#でインゲームロジックを書いてUnityやBlazorで使い回してみる"
emoji: "🦔"
type: "tech"
topics: ["unity", "csharp"]
published: true
published_at: "2024-12-15 20:00"
---

:::message
[この記事はQualiArtsアドベントカレンダー2024](https://qiita.com/advent-calendar/2024/qualiarts) 15日目の記事です
:::

# この記事の目的

PureなC#でインゲームロジックを記述して、Unityや他のプラットフォームで叩いてみます。

このような事をする目的としては以下が考えられます。

- 他プラットフォームに移植を考えている
- サーバーサイドでインゲームロジックを動作させたい
    - クライアント改ざんの検知など
- ロジックがUnityなどのエンジンに依存するのを防ぐ

# 構成

今回は、PureなC#で書いたロジックをUnityとBlazorWASMで動かしてみます。

Blazorの選定理由ですが、BlazorでもASP.NETでもコンソールアプリでも大枠は一緒なので、見た目を提供できる事でしょうか。

筆者がBlazor大好きマンというわけではないです。

では、実際にやって行きましょう。

# PureなC#ロジック

まず、PureなC#ロジックを書いていきます。

こちらは、クラスライブラリとして作成します。

また、動作確認として、NUnitを使ってテストを書いていきます。


```bash
$ mkdir PureLogic
$ cd PureLogic
$ dotnet new sln
$ dotnet new classlib -n PureLogic --framework netstandard2.1
$ mkdir PureLogic/Sources
$ touch PureLogic/Sources/Fortune.cs
$ touch PureLogic/Sources/package.json
$ dotnet new nunit -n PureLogic.Tests
$ dotnet sln add PureLogic/PureLogic.csproj
$ dotnet sln add PureLogic.Tests/PureLogic.Tests.csproj
$ cd PureLogic.Tests
$ dotnet add reference ../PureLogic/PureLogic.csproj 
```

ここまで作業したら以下のディレクトリ構成になっているはずです。

```
$ tree .
.
├── PureLogic
    ├── Class1.cs
    ├── Sources
        ├── Fortune.cs
        └── package.json
    ├── PureLogic.csproj
    └── obj
        ├── PureLogic.csproj.nuget.dgspec.json
        ├── PureLogic.csproj.nuget.g.props
        ├── PureLogic.csproj.nuget.g.targets
        ├── project.assets.json
            └── project.nuget.cache
├── PureLogic.Tests
    ├── PureLogic.Tests.csproj
    ├── UnitTest1.cs
    └── obj
        ├── PureLogic.Tests.csproj.nuget.dgspec.json
        ├── PureLogic.Tests.csproj.nuget.g.props
        ├── PureLogic.Tests.csproj.nuget.g.targets
        ├── project.assets.json
        └── project.nuget.cache
└── PureLogic.sln
```

ここまでで、クラスライブラリのプロジェクトと、それのテストを行うNUnitのプロジェクトを作成しました。

次に、PureLogicプロジェクトにロジックを書いていきます。

PureLogicプロジェクトにあるFortune.csに以下のようなロジックを書いていきます。

```cs
using System;

namespace Shared
{
    public static class FortuneLogic
    {
        public static int GetLuckyNumber(DateTime date)
        {
            return date.Day + date.Month + date.Year;
        }

        public static Fortune GetFortune(int luckyNumber)
        {
            if (luckyNumber % 2 == 0)
            {
                return Fortune.Good;
            }
            else if (luckyNumber % 3 == 0)
            {
                return Fortune.Bad;
            }
            else
            {
                return Fortune.Neutral;
            }
        }
    }

    public enum Fortune
    {
        Good,
        Bad,
        Neutral
    }
}
```

アドカレで大規模なロジックを書くのは難しいので、簡単なロジックにしています。

次に、テストを書いていきます。

PureLogic.TestsプロジェクトのUnitTest1.csに以下のようなテストを書いていきます。

```cs
using Shared;

namespace ClassLibTest;

public class Tests
{
    [Test]
    public void Test()
    {
        var now = new DateTime(2024, 12, 15);
        var luckyNumber = FortuneLogic.GetLuckyNumber(now);
        var fortune = FortuneLogic.GetFortune(luckyNumber);
        
        // 2024 + 12 + 15 = 2051
        Assert.That(luckyNumber, Is.EqualTo(2051));
        
        // 2051 % 2 == 1 && 2051 % 3 == 2 => Neutral
        Assert.That(fortune, Is.EqualTo(Fortune.Neutral));
    }
}
```

テストコードがかけたらテストプロジェクトのディレクトリでテストを実行してみましょう。

```bash
$ dotnet test

復元が完了しました (0.4 秒)
  PureLogic 成功しました (0.6 秒) → /Users/mattun/dev/CSharpShared/PureLogic/PureLogic/bin/Debug/net9.0/PureLogic.dll
  PureLogic.Tests 成功しました (0.5 秒) → bin/Debug/net9.0/PureLogic.Tests.dll
NUnit Adapter *******: Test execution started
Running all tests in /Users/mattun/dev/CSharpShared/PureLogic/PureLogic.Tests/bin/Debug/net9.0/PureLogic.Tests.dll
   NUnit3TestExecutor discovered 1 of 1 NUnit test cases using Current Discovery mode, Non-Explicit run
NUnit Adapter *******: Test execution complete
  PureLogic.Tests テスト 成功しました (2.5 秒)

テスト概要: 合計: 1, 失敗数: 0, 成功数: 1, スキップ済み数: 0, 期間: 2.5 秒
4.8 秒後に 成功しました をビルド
```

ここまでで、PureなC#ロジックを書いて、テストを書いて、テストを実行することができました。

ではここから他のプラットフォームでこのロジックを動かしてみましょう。

# Unityクライアント

まず、先ほど書いたPureLogicのプロジェクトのpackage.jsonに以下のような内容を追加します。

```json
{
  "name": "com.mattun.logic",
  "version": "0.0.1",
  "displayName": "Adcale2024Logic",
  "unity": "2020.3",
  "author": {
    "name": "mattun",
    "email": "hogehoge.fuga@mail.com",
    "url": "https://hoge.com"
  }
}
```

上記は、Unityのパッケージとして認識されるためのパッケージマニフェストです。
詳しくは[Unityのドキュメント](https://docs.unity3d.com/ja/2019.4/Manual/upm-manifestPkg.html)を参照してください。


Unityのプロジェクトを作成しましょう。
今回はUnity6000.0.27f1でやってみました。

プロジェクトを作成したら、PackageManagerを開き、"+"ボタンを押して、"Install package from disk..."を選択し、

![/images/pure_csharp_logic_2
.png](/images/pure_csharp_logic_2.png)

先ほど編集したpackage.jsonを選択します。

![/images/pure_csharp_ligic_3](/images/pure_csharp_logic_3.png)

ロードが終わったら、下記の画像のようにPureLogicがインストールされていることを確認してください。

![/images/pure_csharp_logic_4.png](/images/pure_csharp_logic_4.png)

この状態ではまだPureLogicのクラスを使うことができません。
Fortuneクラスが属するcsprojがまだ生成されておらず、Unityが認識できないためです。

そこで、PureLogicプロジェクトのSourcesディレクトリにasmdefファイルを作成しましょう。
パッケージのフォルダで右クリックするとそのまま作成できます。
asmdefの設定はお好みで。この記事では特に何もしなくても問題ないです。

![/images/pure_csharp_logic_5.png](/images/pure_csharp_logic_5.png)

ここまででPureLogic側の準備が完了しました。

次に、Unity側でPureLogicを使ってみましょう。

クライアント側でもasmdefファイルを作成し、PureLogicを参照するようにしましょう。

![/images/pure_csharp_logic_7.png](/images/pure_csharp_logic_7.png)

そして、適当なスクリプトを作成し、PureLogicを使ってみましょう。
少々雑ですが、このようなスクリプトを作成してみました。

```cs
using UnityEngine;
using System;
using Shared;

public class UnityDemo : MonoBehaviour
{
    [RuntimeInitializeOnLoadMethod()]
    static void Init()
    {
        var date = DateTime.Now;
        var luckyNumber = FortuneLogic.GetLuckyNumber(date);
        Debug.Log($"Your lucky number is {luckyNumber}");

        var fortune = FortuneLogic.GetFortune(luckyNumber);
        Debug.Log($"Your fortune is: {fortune}");
    }
}
``` 

実行してみて、正しくログが出たら成功です。

![/images/pure_csharp_logic_8.png](/images/pure_csharp_logic_8.png)

# Blazorクライアント

次にBlazorで動作させてみましょう。
Unityではcsprojは基本的に開発者が関知できないものなので、asmdefを使ってUnityに認識させました。
しかし、一般的な.NETプロジェクトではcsproj単位で参照を行います。
Blazorでも同様にcsprojを参照することで、PureLogicを使うことができます。

では、実際にBlazorプロジェクトを作成しましょう。

```bash
$ dotnet new blazor -o BlazorClient
$ cd BlazorClient
$ dotnet add reference ../PureLogic/PureLogic/PureLogic.csproj 
```

以上で、Blazorのプロジェクトを作成し、PureLogicのプロジェクトを参照しました。
一般的な.NETプロジェクトではこのように参照を行います。

ここで、Blazorのcsprojファイルを見てみましょう。

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <ItemGroup>
    <ProjectReference Include="..\PureLogic\PureLogic\PureLogic.csproj" />
  </ItemGroup>

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

重要なのは下記です。
このように、ディレクトリを指定することでローカルのプロジェクトを参照することができます。

```xml
  <ItemGroup>
    <ProjectReference Include="..\PureLogic\PureLogic\PureLogic.csproj" />
  </ItemGroup>
```

では、Blazorのコードを書いていきましょう。

`1Components/Pages/Home.razor`を下記に書き換えます。

```razor
@page "/"
@using Shared;

<PageTitle>Home</PageTitle>


Now : @DateTime.Now

<br>

Lucky Number : @FortuneLogic.GetLuckyNumber(DateTime.Now);

<br>

Fortune : @FortuneLogic.GetFortune(FortuneLogic.GetLuckyNumber(DateTime.Now));
```

書き換えたら実行しましょう。プロジェクトがあるディレクトリで`dotnet run`を実行するとできます。

画像のように、ロジックが動作していれば成功です。

![/images/pure_csharp_logic_9.png](/images/pure_csharp_logic_9.png)

# 最後に

ここまでで、PureなC#ロジックを書いて、UnityとBlazorで動かすことができました。
本当はiOSネイティブプラグイン周りの話をする予定でしたが、検証に時間が掛かってしまい今回は断念しました。
アドカレは余裕を持ってやるモノですね。

この記事は以上となります。ありがとうございました！
明日は@ra-menIAさんの記事です。お楽しみに！
