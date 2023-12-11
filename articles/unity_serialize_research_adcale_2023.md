---
title: "Unityのシリアライズフォーマットについて考えてみる"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity","csharp","json","messagepack"]
published: false
---
# はじめに

都内で大学生をしているmattunです。執筆時点で今年はアドカレを7枠取っていることに気がつき驚愕しています。卒論もやばいのに一体何をしているのでしょう。

この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)のX日目です。

# この記事でやること

Unityにおけるオブジェクトのシリアライズについて考えていきます

# シリアライズとは

まず、シリアライズとは何のことでしょうか、ChatGPTに聞いてみました。

> オブジェクトのシリアライズは、プログラム内で使用されるオブジェクトの状態やデータ構造を、バイト列やテキストなどの形式に変換するプロセスです。シリアライズされたデータは通常、ファイルに保存したり、ネットワークを介して他のプログラムと共有したりするために使用されます。逆のプロセス、つまりシリアライズされたデータからオブジェクトの状態を復元することを「デシリアライズ」と呼びます。
> オブジェクトのシリアライズは、異なるプログラムやプラットフォーム間でデータを共有するためによく使用されます。また、オブジェクトの永続化（データを保存して後で再利用すること）や、分散コンピューティング環境でのデータ交換などでも役立ちます。
> プログラミング言語やフレームワークには、オブジェクトをシリアライズするための組み込みのメソッドやライブラリが提供されていることがあります。例えば、JavaではSerializableインターフェースを実装することでオブジェクトをシリアライズできます。Pythonではpickleモジュールが一般的に使用され、JSON形式を利用することもあります。


ざっくり言うと、シリアライズとはオブジェクト(インスタンス)をバイト列やテキストの形式に変換することを指しています。
逆にバイト列やテキストからオブジェクトに復元することをデシリアライズと言います。
ゲーム開発における主な用途だとサーバとのやり取りやセーブデータの形式があげられます。

そして、テキストやバイト列のフォーマットがシリアライズフォーマットとなる訳です。

ここからUnityで使用可能なシリアライズフォーマットについて解説して行きます。

# JSON

JSONとはJavaScriptでのオブジェクトの記法を用いたフォーマットとなっています。
世界で最も使用者が多いシリアライズフォーマットなのではないでしょうか。

記法は以下の通りとなります。

```json
{
    "name" : "",
    "level" : 10,
    "weapons " : [
        {
            "name" : "sowrd",
            "power" : 100
        },
        {
            "name" : "stick",
            "power" : 60
        }
    ]
}
```

Unityで使用する際にはUnity標準で使用できるJsonUtilityが鉄板となっています。

しかしJsonUtilityは
```cs
[Serializable]
public class User
{
    public string Name {get; set;}
    public float Level {get; set;}
}
```
のようなプロパティには対応していません。

以下のようにメンバをただの変数として定義する必要があります。
```cs
[Serializable]
public class User
{
    public string name;
    public float level;
}
```

このようにかゆいところに手が届かないため、JsonUtility以外の使用方法についても解説したいと思います。

## System.Text.Json

System.Text.Json は名前空間からわかる通り、.NET標準のJSONモジュールです。

特徴としてはSourceGeneratorというC#コンパイラの機能を用いて各シリアライズクラスに対し、専用のシリアライズ/デシリアライズコードを生成することで高速な処理を実現している点が上げられます。
下記記事で詳しく取り上げられています。

https://devblogs.microsoft.com/dotnet/try-the-new-system-text-json-source-generator/

詳しい解説や導入方法、JsonUtilityとSystem.Text.Jsonの性能比較についてはは下記の記事で解説されています。

https://blog.yucchiy.com/2022/12/csharp-advent-calendar-system-text-json-unity/#%E3%83%91%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%B3%E3%82%B9%E6%AF%94%E8%BC%83

## Newtonsoft.Json

ちょっとややこしいことになっています。


## Utf8Json (非推奨)
neueccさんせいのJsonシリアライザです。性能はとてもよいのですがパブリックアーカイブとなっており、これから新規で採用することはないです。
少なくともIL2CPP環境では確実に動作しません。
今までありがとうございました。

https://github.com/neuecc/Utf8Json

# MessagePack

MessagePackはJSONの置換として注目されているバイト列を用いたフォーマットです。

https://msgpack.org/ja.html

公式サイトからわかるとおり、多くの言語でMessagePack向けのライブラリが作成されています。
Unityで利用可能な MessagePack for C# は GitHub上でもstar数が5000以上でかつ頻繁にアップデートが来ているので十分信頼できるライブラリとなっています。

https://github.com/MessagePack-CSharp/MessagePack-CSharp

また、作成者はUniTaskやUniRx、ZStringなどを開発したCysharpの代表をされている neueccさんです。



# MemoryPack

