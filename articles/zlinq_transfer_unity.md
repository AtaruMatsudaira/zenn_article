---
title: "ZLinq for Unity移行ガイド"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity", "csharp", "cysharp", "zlinq"]
published: false
---

# はじめに

この記事では、既にある程度開発が進んだ Unity プロジェクトに対して、ZLinq を導入した際に行った対応についてまとめます。
私自身、ZLinq の使い方を探っている状況なので、マサカリ大歓迎です。

# 環境

移行した環境は下記の通りです。

- Unity 6000.0.50f1
- C# 11 (CsprojModifier を使って LangVersion を引き上げてます)
- ZLinq 1.4.10
- ZLinq.DropInGenerator 1.4.10

自身が管理する全てのアセンブリに対して、下記ソースを配置しています。

```cs
global using ZLinq;

[assembly: ZLinqDropInAttribute("", DropInGenerateTypes.Everything)]
```

これにより、既存の LINQ を使用していた箇所が全て DropInGenerator に置き換わります。

この時点でコンパイルエラーが大量に出ている状態でした。

また、ZLinq の [README](https://github.com/Cysharp/ZLinq?tab=readme-ov-file#unity)を参考にプロジェクト側で `ValueEnumerable` の拡張メソッドとして `IEnumerable<T>` を返す `AsEnumerable` も実装しています。


# 問題発生項目とその対処

下記で対応を列挙していきます。

## `IEnumerable<T>` が使えなくなる

ZLinq は独自の ValueEnumerable 構造体を基底に持っているため、`IEnumerable<T>` と直接的な互換性はありません。
そのため、 明示的に `IEnumerable<T>` を使用している箇所は軒並みコンパイルエラーとなってしまいます。

### メンバとして `IEnumerable<T>` を保持していた場合

愚直にやるのなら、 `ValueEnumerable` に置き換えるのが良さそうに思えるかも知れません。

実は、`ValueEnumerable` は構造体であり、型引数で `IValueEnumerator<T>` を実装した型（以下、`TValueEnumerator`）を要求しているため、共変性もありません。
そのため、 `ValueEnumerator` の具象型を指定する必要があります。
ただ、この `TValueEnumerator` は使用されているオペレータとその組み合わせを元に型が変わるため、個人的には具象型を指定するのは現実的ではないように感じました。

:::details オペレータのサンプルコード

```cs
ValueEnumerable<FromRange, int> source = ValueEnumerable.Range(0, 10);

ValueEnumerable<Where<FromRange, int>, int> where = source.Where(x => x % 2 == 0);

ValueEnumerable<RangeSelect<int>, int> select = source.Select(x => x * 2);

ValueEnumerable<WhereSelect<FromRange, int, int>, int> whereSelect = source
    .Where(x => x % 2 == 0)
    .Select(x => x * 2);
```

:::

また、ZLinq 導入以前に、`IEnumerable<T>` として保持すべきケースは遅延処理を行わせたいなど、限られていると感じます。

そのため、

- 遅延処理をさせたい場合
  - `IReadOnlyList<T>` の getter プロパティを定義し、プロパティ内でシーケンスを評価し、`ToArray()` して返す
- その他
  - 代入する箇所で `ToArray()` を行い、`IReadOnlyList<T>` で保持

の対応を行いました。

### 自身が定義したメソッドなどで `IEnumerable<T>` を 返していた場合

こちらも、ほぼ上記と同様に、`IReadOnlyList<T>` で返すようにしています。

また、ZLinq には `PooledArray` という `IDisposable` を実装した `System.Buffers.ArrayPool` のラッパーが実装されています。
Dispose されると、`ArrayPool` に返却されるためコストが低いです。
返したコレクションの寿命が明確な場合はこちらの `PooledArray` で返すようにしています。

### 使用してるライブラリなどの API 側で `IEnumerable<T>` を要求している場合

使用頻度の低いものに関しては、上記で定義した `AsEnumerable` でそのまま渡すようにしています。

使用頻度の高いものは、`ValueEnumerable` で受け取る拡張メソッドや asmref + partial クラスをプロジェクト側で定義し、
その中で `AsEnumerable()` や `ToArray()` をして API 側に渡しています。

:::details UniTask の partial クラスのサンプルコード

```cs
namespace Cysharp.Threading.Tasks
{
    public partial struct UniTask
    {
        public static UniTask<T[]> WhenAll<TEnumerator, T>(ValueEnumerable<TEnumerator, UniTask<T>> valueEnumerable)
            where TEnumerator : struct, IValueEnumerator<UniTask<T>>
        {
            return WhenAll(valueEnumerable.AsEnumerable());
        }

        public static UniTask WhenAll<TEnumerator>(ValueEnumerable<TEnumerator, UniTask> valueEnumerable)
            where TEnumerator : struct, IValueEnumerator<UniTask>
        {
            return WhenAll(valueEnumerable.ToArray());
        }
    }
}
```

:::

:::details R3 の拡張メソッドのサンプルコード

```cs
public static Observable<T> Merge<T, TEnumerator>(this ValueEnumerable<TEnumerator, Observable<T>> source)
    where TEnumerator : struct, IValueEnumerator<Observable<T>>
{
    return source.AsEnumerable().Merge();
}
```

:::

# 最後に

対応は以上となります。
ぶっちゃけ、`ToArray()` しまくってますが、これはこれでアリなのか自分でもあまりよく分かっていません。

本題とはそれてしまいますが、ZLinq を導入してみて、最新の .NET 仕様のオペレータを利用できるのが個人的に良かったと感じました。
例えば、今回導入したプロジェクトでは Nullable を導入しているのですが、
既存の .NET Standard 2.1 の LINQ の `FirstOrDefault` だと Null を返す可能性があるのに `T` で返していました。
ZLinq は最新の .NET よろしく、 `T?` で返してくれます。
これにより、よりnull安全性を高めることができました。

重ねてになりますが、私自身手探りで ZLinq の使い方を模索している最中なのでマサカリ大歓迎です！
Zenn のコメントでも Twitter の引用 RT でも何でも受け付けてます！よろしくお願いします！