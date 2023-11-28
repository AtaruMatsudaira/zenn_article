---
title: "YetAnotherHttpHandlerでHTTP/2通信！〜UnityWebRequestとの比較を添えて〜"
emoji: "🐥"
type: "tech"
topics:
  - "csharp"
  - "unity"
  - "network"
  - "http2"
published: false
published_at: "2023-12-01 00:00"
---

# はじめに
都内で大学生をしているmattunです。最近は卒研が佳境を迎えており、中々大変な日々を過ごしています。
この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の1日目です。

# TL;DR
Cysharpさん から [YetAnotherHYetAnotherHttpHandler](https://github.com/Cysharp/YetAnotherHttpHandler)がリリースされました。

詳しくはneueccさんのブログを読んでください。
https://neue.cc/2023/07/28_yetanotherhttphandler.html

YetAnotherHttpHandlerは最新のUnity環境でも問題なく動作するgRPCクライアントとして注目されていますが、今回はUnityWebRequestで行うような通常のHTTPリクエストをYetAnotherHttpHandlerでやって行きたいと思います。

# HTTP/1.1 vs HTTP/2

SEGAさんの講演ですごいわかりやすく取り上げてくれています。
https://learning.unity3d.jp/3330/

以下はGPTによる要約です。

```
HTTP/2は、並列で複数のリクエストとレスポンスを処理できるストリームという機能を持っています。
これにより、HTTP/1.1では1リクエスト処理するのに1RTTの時間がかかるのに対し、HTTP/2では1RTTの時間で複数のリクエストとレスポンスを処理することができます。
また、HTTP/2ではヘッダーの圧縮やサーバープッシュなどの機能も備えており、これらの機能によって通信量を削減することができます。
これらの改善により、HTTP/2はHTTP/1.1よりも高速になります
```

# 動作環境
- Unity2022.3.11f1
- YetAnotherHttpHandler : 
- Editor環境 : iMac Late2019
:::details Mac 詳細
```
$ neofetch
                    'c.          mattun@cpsLabnoiMac-Pro.local
                 ,xNMM.          --------------------------------
               .OMMMMo           OS: macOS 13.2.1 22D68 x86_64
               OMMM0,            Host: iMacPro1,1
     .;loddo:' loolloddol;.      Kernel: 22.3.0
   cKMMMMMMMMMMNWMMMMMMMMMM0:    Uptime: 10 days, 4 hours, 53 mins
 .KMMMMMMMMMMMMMMMMMMMMMMMWd.    Packages: 141 (brew)
 XMMMMMMMMMMMMMMMMMMMMMMMX.      Shell: fish 3.5.1
;MMMMMMMMMMMMMMMMMMMMMMMM:       Resolution: 2560x1440@2x, 1080x1920@2x
:MMMMMMMMMMMMMMMMMMMMMMMM:       DE: Aqua
.MMMMMMMMMMMMMMMMMMMMMMMMX.      WM: Quartz Compositor
 kMMMMMMMMMMMMMMMMMMMMMMMMWd.    WM Theme: Blue (Dark)
 .XMMMMMMMMMMMMMMMMMMMMMMMMMMk   Terminal: WarpTerminal
  .XMMMMMMMMMMMMMMMMMMMMMMMMK.   CPU: Intel Xeon W-2140B (16) @ 3.20GHz
    kMMMMMMMMMMMMMMMMMMMMMMd     GPU: Radeon Pro Vega 56
     ;KMMMMMMMWXXWMMMMMMMk.      Memory: 21575MiB / 32768MiB
       .cooc,.    .,coo:.
```
:::
- Standalone環境 : ThinkPad T14s
:::details Win 詳細
```
        ,.=:!!t3Z3z.,                  mattun@LAPTOP-17L636F3
       :tt:::tt333EE3                  ----------------------
       Et:::ztt33EEEL @Ee.,      ..,   OS: Windows 11 Home x86_64
      ;tt:::tt333EE7 ;EEEEEEttttt33#   Host: LENOVO 21BRCTO1WW
     :Et:::zt333EEQ. $EEEEEttttt33QL   Kernel: 10.0.22621
     it::::tt333EEF @EEEEEEttttt33F    Uptime: 2 days, 14 hours, 4 mins
    ;3=*^```"*4EEV :EEEEEEttttt33@.    Packages: 3 (scoop)
    ,.=::::!t=., ` @EEEEEEtttz33QF     Shell: bash 5.2.12
   ;::::::::zt33)   "4EEEtttji3P*      Resolution: 2240x1400
  :t::::::::tt33.:Z3z..  `` ,..g.      DE: Aero
  i::::::::zt33F AEEEtttt::::ztF       WM: Explorer
 ;:::::::::t33V ;EEEttttt::::t3        WM Theme: Custom
 E::::::::zt33L @EEEtttt::::z3F        Terminal: Windows Terminal
{3=*^```"*4E3) ;EEEtttt:::::tZ`        CPU: 12th Gen Intel i7-1260P (16) @ 2.500GHz
             ` :EEEEtttt::::z7         GPU: Caption
                 "VEzjt:;;z>*`         GPU: Intel(R) Iris(R) Xe Graphics
                                       GPU
                                       Memory: 15019MiB / 16081MiB
```
:::
- iOS環境 : iPhone15 Pro
- Android環境 : Pixel 6

# デモ

今回は研究室のローカルネットワーク上に配置した33.2MBのVRMモデルをダウンロードさせてみたいと思います。

今回はニコニ立体で配布されていた金子卵黄様の「ふらすこ式風きりたんVRM」をお借りします。

https://3d.nicovideo.jp/works/td39036

プロジェクトはGitHubで配布しています。手元で動かしたい方はどうぞ、CloneなりForkなりしてください。
https://github.com/AtaruMatsudaira/AdventCal_http

## ソースコード

```cs
using System.Diagnostics;
using System.Net.Http;
using UnityEngine;
using Cysharp.Net.Http;
using Cysharp.Threading.Tasks;
using TMPro;
using UnityEngine.Networking;
using VRM;

public class HttpCompareSample : MonoBehaviour
{
    [SerializeField] private Transform unityWebRequestModelRoot;
    [SerializeField] private Transform yetAnotherHttpModelRoot;

    [SerializeField] private TMP_Text unityWebRequestLabel;
    [SerializeField] private TMP_Text yetAnotherHttpHandlerLabel;

    private const string URI = "";

    private void Start()
    {
        RunAsync().Forget();
    }

    private async UniTaskVoid RunAsync()
    {
        var sw = new Stopwatch();

        #region UnityWebRequest

        sw.Start();
        var uwrBin = await GetUnityWebAsync();
        sw.Stop();

        GenerateModelAsync(uwrBin, unityWebRequestModelRoot).Forget();
        unityWebRequestLabel.SetText($"Elapsed:{sw.ElapsedMilliseconds}");

        #endregion

        #region YetAnotherHttpHandler

        sw.Restart();
        var yahBin = await GetYetAnotherAsync();
        sw.Stop();

        GenerateModelAsync(yahBin, yetAnotherHttpModelRoot).Forget();
        yetAnotherHttpHandlerLabel.SetText($"Elapsed:{sw.ElapsedMilliseconds}");

        #endregion
    }

    private async UniTask<byte[]> GetUnityWebAsync()
    {
        var client = UnityWebRequest.Get(URI);
        await client.SendWebRequest();

        var bytes = client.downloadHandler.data;

        return bytes;
    }

    private async UniTask<byte[]> GetYetAnotherAsync()
    {
        using var handler = new YetAnotherHttpHandler();
        var httpClient = new HttpClient(handler);

        var bytes = await httpClient.GetByteArrayAsync(URI);

        return bytes;
    }

    private async UniTaskVoid GenerateModelAsync(byte[] bytes, Transform root)
    {
        var model = await VrmUtility.LoadBytesAsync(root.name, bytes);
        model.ShowMeshes();
        model.Root.transform.parent = root;
        model.Root.transform.localPosition = Vector3.zero;
        model.Root.transform.localRotation = Quaternion.Euler(0,180,0);
    }
}
```

# 性能比較

| | UnityWebRequest(ms) | YetAnotherHttpHandler(ms) |
| ---- | ---- | ---- |
| Editor | 1844 | 1433 |
| Standalone(Win) | 3399 | 1231 |
| iOS | 6393 | 1459 |
| Android | 2248 | 1947 |

以下にぞれぞれの端末でのスクリーンショットを掲載します。

- Unity Editor
![](https://storage.googleapis.com/zenn-user-upload/bed405df38b2-20231127.png)

- Standalone(Windows)
![](https://storage.googleapis.com/zenn-user-upload/8dc114edd81f-20231127.png)
	
- iOS
![](https://storage.googleapis.com/zenn-user-upload/ddb64771a3d9-20231127.png)

- Android
![](https://storage.googleapis.com/zenn-user-upload/d85c41d08f77-20231127.png)

# 終わりに

今回はYetAnotherHttpHandlerを触ってみました。
実は私の大学での研究でもUnityWebRequestからYetAnotherHttpHandlerにReplaceしていたりします。
HttpClient自体は.NET標準のHttpClientを使用しているので、使用する上ではそのままMSの公式dドキュメントが参考になるかと思います。
https://learn.microsoft.com/ja-jp/dotnet/fundamentals/networking/http/httpclient

明日は、〇〇さんの[XXXX](YYYY)となります！