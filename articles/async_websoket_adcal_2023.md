---
title: "ナウでヤングな若者言葉をWebSocketがしゃべれるようにする"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","unity","websocket"]
published: true
published_at: "2023-12-03 00:00"
---

# はじめに

都内で大学生をしているmattunです。アドカレと卒論で執筆スパイラルに陥っています。
この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の3日目です。

# TL;DR

async/awaitでUniTaskAsyncEnumerableなWebSocketライブラリを自作したので、使用方法とサンプルの解説をします。

https://github.com/AtaruMatsudaira/AsyncWebSocket

# 背景

研究でWebSocketをUnityで扱うことになりました。
async/awaitでUniTaskAsyncEnumerableなのがナウでヤングだと感じたのですが、さすがにWebSocketのライブラリは若者言葉を使えません。

(ナウでヤングな若者言葉はとりすーぷさんの記事で丁寧に解説されています。)
https://qiita.com/toRisouP/items/8f66fd952eaffeaf3107#unitaskasyncenumerableiunitaskasyncenumerable

そこで、NativeWebSocketというライブラリがナウでヤングな若者言葉を話せるようにラップしてあげることにしました。
そのため、WebSocketの根幹的な部分はまんまNativeWebSocketのまんまです。

# インストール

manifest.jsonのdependeciesに下記の項目を追加しましょう。
追加されるのは今回作成した私のライブラリの他に、NativeWebSocketとUniTaskとなります。
```
{
  "dependencies": {
    "com.cysharp.unitask": "https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask",
    "com.endel.nativewebsocket": "https://github.com/endel/NativeWebSocket.git#upm",
    "com.mattun.asyncwebsocket" : "https://github.com/AtaruMatsudaira/AsyncWebSocket.git?path=/Assets/Plugins/AsyncWebSocket",
    ...
  }
}
```

本当はこっちで配布してるpackage.jsonのdependeciesに書いた内容で勝手にインストールして欲しいんですけど、なんでupmはそうならないんですかね...?
何かいい方法があったら教えてください。

# サンプル

10回ボタンがクリックされるまで、
- ボタンがクリックされたら test とメッセージをWebSocketで送信する
- Messageを受信したらConsoleのログに出力する

サンプルは PackageManager の Samples から Import できます。
![Alt text](/images/adcal_2023_ws_sample.png)

```cs
using System.Threading;
using AsyncWebSocket;
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using UnityEngine;
using UnityEngine.UI;

public class SampleConnection : MonoBehaviour
{
    [SerializeField] private Button button;
    [SerializeField] private string uri;

    WebSocketAsyncTrigger _webSocketAsyncTrigger;

    private void Start()
    {
        RunAsync().Forget();
    }

    private async UniTask RunAsync()
    {
        using (CancellationTokenSource cts = CancellationTokenSource.CreateLinkedTokenSource(this.GetCancellationTokenOnDestroy()))
        {
            _webSocketAsyncTrigger = WebSocketAsyncTrigger.GetOrCreate(uri, cts.Token);
       
            _webSocketAsyncTrigger.OnReceivedAsyncEnumerable(cts.Token).Subscribe(data =>
            {
                string msg = System.Text.Encoding.UTF8.GetString(data);
                Debug.Log(msg);
            }).AddTo(cts.Token);

            _webSocketAsyncTrigger.OnErrorAsyncEnumerable(cts.Token)
                .Subscribe(Debug.LogError)
                .AddTo(cts.Token);
     
            await button.onClick.OnInvokeAsAsyncEnumerable(cts.Token).Take(10).ForEachAwaitAsync(async _ =>
            {
                await _webSocketAsyncTrigger.PublishAsync("test", cts.Token);
            }, cts.Token);
     
        }
    }

    private void OnDestroy()
    {
        _webSocketAsyncTrigger?.Dispose();
    }
}
```

WebSocketAsyncTriggerからは以下の IUniTaskEnumerable が生えています
- OnOpenedAsyncEnumerable : WebsocketのOpen時
- OnReceivedAsyncEnumerable : メッセージを受信したタイミング
- OnErrorAsyncEnumerable : エラーを受信したタイミング
- OnClosedAsyncEnumerable : WebSocket時

今回のサンプルだと、

```cs
_webSocketAsyncTrigger.OnReceivedAsyncEnumerable(cts.Token).Subscribe(data =>
            {
                string msg = System.Text.Encoding.UTF8.GetString(data);
                Debug.Log(msg);
            }).AddTo(cts.Token);
```

でメッセージを受け取るたびにログ出力しています。

また、メッセージを送る際は

WebSocketAsyncTriggerからPublisAsyncが生えています。
こちらは string と byte[] の双方に対応しています。

今回のサンプルでは

```cs
await button.onClick.OnInvokeAsAsyncEnumerable(cts.Token).Take(10).ForEachAwaitAsync(async _ =>
            {
                await _webSocketAsyncTrigger.PublishAsync("test", cts.Token);
            }, cts.Token);
```

でボタンのonClickイベントが発火されるたびに、test を送っています。

# 終わりに

この手のasync/awaitでUniTaskAsyncEnumerableなナウでヤングな若者言葉をプロジェクトを作成するたびに書いてるので、いい加減オラオラUnityUIライブラリを作成して使い回したいと思います。

これでこの記事は以上となります。
明日は[@harashima318さん](https://qiita.com/harashima318)の[ゲーム開発シーンにおけるDDDについて書く予定]()です！お楽しみに！