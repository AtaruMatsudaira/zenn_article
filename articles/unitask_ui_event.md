---
title: "UniTaskでUIイベントを直感的に扱ってみる ~連打対応~"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity","csharp"]
published: true
published_at: "2023-12-13 00:00"
---

# はじめに

都内で大学生をしているmattunです。松屋のうまトマチキンを食べました。美味しかったです。
この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の13日目です。

# TL;DR

UniTaskを使用し、ボタンがクリックされた際に発生する非同期処理が重複しないように管理します。

# IUniTaskAsyncEnumerableとは

UniTaskAsyncEnumerableは、C# 8.0のIAsyncEnumerableをUniTaskで実装したもので、複数の非同期処理を一括で扱うことができる機能です。

大雑把に言うと、イベントを非同期のストリームにしてLinqライクなメソッドチェーンで管理する感じです。

Observable（IObservable<T>）に似ていますが、UniTaskAsyncEnumerableはPull型の非同期処理を扱う仕組みであり、ObservableはPush型で非同期処理を扱っているため対照的です。
以上のことから、UniTaskAsyncEnumerableを使用する際の大きな利点は、非同期処理の実行タイミングを受信側でコントロールできる点にあります。

詳しくは下記のリンクにて説明されています。

https://qiita.com/toRisouP/items/8f66fd952eaffeaf3107#%E6%96%B0%E6%A9%9F%E8%83%BD

# Buttonのイベントを IUniTaskAsyncEnumerable で制御してみる

では実際にButtonのイベントを制御してみましょう。
今回はボタンをクリックすると5秒間をカウントするタイマーが作動します。

ソースコードは以下の通りです。
(今回はCancellationTokenを省略しています)

```cs
using System;
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

namespace Samples
{
    public class ButtonThrottlerSample : MonoBehaviour
    {
        [SerializeField] private Button buttonA;
        [SerializeField] private Button buttonB;

        [SerializeField] private TMP_Text timerLabelA;
        [SerializeField] private TMP_Text timerLabelB;

        private void Start()
        {
            buttonA.onClick.AddListener(() => TimerRunAsync(timerLabelA).Forget());

            buttonB.OnClickAsAsyncEnumerable().SubscribeAwait(async _ => await TimerRunAsync(timerLabelB));
        }

        private async UniTask TimerRunAsync(TMP_Text timerLabel)
        {
            DateTime endTime = DateTime.Now.AddSeconds(5);

            while (DateTime.Now < endTime)
            {
                timerLabel.SetText((endTime - DateTime.Now).TotalSeconds.ToString("F2"));
                await UniTask.Yield(PlayerLoopTiming.Update);
            }

            timerLabel.SetText("Finish");
        }
    }
}
```

こんな感じに動作します。

![Alter](/images/adcal_2023_unitask_ui_event_1.gif)

A、Bのそれぞれの挙動を見てみましょう。

## 従来の``onClick.AddListerer``による実装 (Aボタン)

```cs
buttonA.onClick.AddListener(() => TimerRunAsync(timerLabelA).Forget());
```

Aはタイマーの非同期処理が動いている間でもクリックイベントが発火してしまいます。
タイマーの非同期処理をForgetで呼んでいるので、終了を検知できていませんね。

##  ``IUniTaskAsyncEnumerable`` を使った実装 (Bボタン)

```cs
buttonB
    .OnClickAsAsyncEnumerable()
    .SubscribeAwait(async _ => await TimerRunAsync(timerLabelB));
```

BはSubscribeAwaitでタイマーの非同期処理をawaitしています。
そのため、重複してタイマーの非同期処理が実行されません。

今回のサンプルではSubscribeAwaitを使用しました。
そのほかにも、ForEachAsyncやForEachAwaitAsyncなどの選択肢もあります。
状況に応じて適切に使い分けましょう。

# おわりに

この記事ではボタンを使用しました。
そのほかにも、InputFieldやDropdown、Sliderなどの他のUIコンポーネントでも``IUniTaskAsyncEnumerable`` 系統の拡張メソッドが生えているので活用しましょう。

また、今回のサンプルは下記リポジトリの ``Assets/Scenes/ButtonThrottlerSample.unity`` に格納されています。
CloneなりForkなりお好きにどうぞ。

https://github.com/AtaruMatsudaira/UniTaskUISamples

これで [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の13日目の記事は以上となります！

明日の投稿もご期待ください！！！