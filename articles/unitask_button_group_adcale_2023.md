---
title: "UniTaskでUIイベントを直感的に扱ってみる ~ボタンのグループ化~"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "csharp"
  - "unity"
published: true
published_at: "2023-12-014 00:00"
---

# はじめに
都内で大学生をしているmattunです。最近は毎日特茶を飲んでいますが痩せれた気がしません。
この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の14日目です。

# TL;DR

下記のように、ボタンをグループ化して排他制御します

![](/images/adcal_2023_unitask_ui_group_1.gif)

# ボタンをグループ化する

従来ではボタンに対して直接、イベントを購読していました。
今回はグループのインスタンスに対してボタンと処理を紐付けたいと思います。

グループのクラスは以下の通りです。

```cs
using System;
using System.Linq;
using System.Collections.Generic;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine.UI;

namespace UI
{
    public class ButtonGroup : IDisposable
    {
        private readonly Dictionary<string, (Button btn, Func<CancellationToken, UniTask> eventAction)> _eventMap =
            new(4);

        private readonly CancellationTokenSource _cts = new();

        private Func<CancellationToken, UniTask> _preAsyncEvent;
        private Func<CancellationToken, UniTask> _postAsyncEvent;

        public void AddButton(Button button, Func<CancellationToken, UniTask> eventAction, string key = default)
        {
            key ??= Guid.NewGuid().ToString();

            _eventMap.Add(key, (button, eventAction));
        }

        public void SetPreAsyncEvent(Func<CancellationToken, UniTask> preClickEvent)
        {
            _preAsyncEvent = preClickEvent;
        }

        public void SetPostAsyncEvent(Func<CancellationToken, UniTask> postClickEvent)
        {
            _postAsyncEvent = postClickEvent;
        }

        public void Remove(string key)
        {
            _eventMap.Remove(key);
        }

        public async UniTask RunAsync(CancellationToken ct)
        {
            var clickHandlers = _eventMap
                .Values
                .Select(x => (x.btn, x.eventAction))
                .ToList();

            while (!ct.IsCancellationRequested)
            {
                using var clickCts = CancellationTokenSource.CreateLinkedTokenSource(ct, _cts.Token);

                var clickedIndex = await UniTask.WhenAny(
                    clickHandlers
                        .Select(h => h.btn.OnClickAsync(clickCts.Token)));

                if (_preAsyncEvent != null)
                {
                    await _preAsyncEvent(ct);
                }

                await clickHandlers[clickedIndex].eventAction(clickCts.Token);

                if (_postAsyncEvent != null)
                {
                    await _postAsyncEvent(ct);
                }
            }
        }

        public void Dispose()
        {
            _eventMap.Clear();
            _cts.Cancel();
            _cts.Dispose();
        }
    }
}
```

以下で詳しく解説していきます。

## ボタンの登録

```cs
        public void AddButton(Button button, Func<CancellationToken, UniTask> eventAction, string key = default)
        {
            key ??= Guid.NewGuid().ToString();

            _eventMap.Add(key, (button, eventAction));
        }
```

登録時は ``Button`` と``Func<CancellationToken,UniTask>``、キーの文字列をグループに渡しています。
キーはボタンの購読を解除するのに使用しています。
また、キーの指定がない際はGuidを生成しています。

## イベントハンドリング

まず、それぞれのボタンのクリックのUniTaskのコレクションをWhenAnyでawaitすることで、どのボタンがクリックされたかIndexで取得しています。

```cs
var clickedIndex = await UniTask.WhenAny(clickHandlers
    .Select(h => h.btn.OnClickAsync(clickCts.Token)));
```

その後、Indexに基づいて登録された非同期処理を実行しています。

```cs
await clickHandlers[clickedIndex].eventAction(clickCts.Token);
```

注意点としては、CancellationTokenSource/CancellationTokenをクリックされるたびに``CancellationTokenSource.CreateLinkedTokenSource``で作成しています。
``using var`` で定義しているのでスコープが外れた際、今回の場合は1回のwhileループが終了する際に、CancellationSourceがDisposeされています。
そのため、WhenAnyで待たれなくなった非同期処理も1ループが終了する際にはCancellされています。

では実際にこのButtonGroupを使用してみましょう。

# イベントを購読する

冒頭の紹介したように動作するサンプルです。

![](/images/adcal_2023_unitask_ui_group_1.gif)

それぞれのボタンクリック時の演出中はボタンをタップしても演出が再生されないようになっています。

```cs
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using TMPro;
using UI;
using UnityEngine;
using UnityEngine.UI;

namespace Samples
{
    public class ButtonGroupSample : MonoBehaviour
    {
        [SerializeField] private Button button1;
        [SerializeField] private Button button2;
        [SerializeField] private Button button3;

        [SerializeField] private TMP_Text label1;
        [SerializeField] private TMP_Text label2;
        [SerializeField] private TMP_Text label3;

        [SerializeField] private GameObject processingPanel;

        private ButtonGroup _buttonGroup;

        private void Start()
        {
            _buttonGroup = new ButtonGroup();

            _buttonGroup.AddButton(button1, Button1EventAsync);
            _buttonGroup.AddButton(button2, Button2EventAsync);
            _buttonGroup.AddButton(button3, async ct =>
            {
                label3.SetText("size change");
                for (int i = 0; i < 10; i++)
                {
                    label3.fontSize += 1;
                    await UniTask.Delay(TimeSpan.FromSeconds(0.5), cancellationToken: ct);
                }

                label3.fontSize = 72;
                label3.SetText("Finish");
            });

            _buttonGroup.SetPreAsyncEvent(async _ => processingPanel.SetActive(true));
            _buttonGroup.SetPostAsyncEvent(async _ => processingPanel.SetActive(false));

            _buttonGroup.RunAsync(destroyCancellationToken).Forget();
        }

        private async UniTask Button1EventAsync(CancellationToken ct)
        {
            for (int i = 3; i > 0; i--)
            {
                label1.SetText($"Wait {i} seconds");
                await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: ct);
            }

            label1.SetText("Finish");
        }

        private async UniTask Button2EventAsync(CancellationToken ct)
        {
            float time = 0;
            label2.SetText("Color Change");

            while (time < 3)
            {
                label2.color = Color.HSVToRGB(time / 3f, 1, 1);
                await UniTask.Yield(PlayerLoopTiming.Update, ct);
                time += Time.deltaTime;
            }

            label2.color = Color.black;
            label2.SetText("Finish");
        }

        private void OnDestroy()
        {
            _buttonGroup.Dispose();
        }
    }
}
```

登録部分に関してはメンバの関数を渡してもラムダ式を渡しても問題ないようにしています。

```cs
            _buttonGroup.AddButton(button1, Button1EventAsync);
            _buttonGroup.AddButton(button2, Button2EventAsync);
            _buttonGroup.AddButton(button3, async ct =>
            {
                ... // 省略
            }
```

また今回はボタンクリックの非同期処理が実行される前と後に共通で実行して欲しい処理も設定できます。

今回は前後でパネルを表示させることにしました。

```cs
            _buttonGroup.SetPreAsyncEvent(async _ => processingPanel.SetActive(true));
            _buttonGroup.SetPostAsyncEvent(async _ => processingPanel.SetActive(false));
```

# おわりに

今回紹介したものはあくまで一例となります。
改善案とかがありましたら下記リンクのリポジトリでPRを投げていただけると励みになります。

これで[サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の14日目の記事は終わりになります.
ありがとうございました。