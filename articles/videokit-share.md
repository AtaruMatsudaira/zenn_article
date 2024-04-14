---
title: "Unityでカメラロールにアクセスしてみる"
emoji: "📷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity","csharp","android","ios"]
published: true
---

# はじめに

この度、新社会人になった(?) mattun です。
今回は NatMLから出ている VideoKit を使って Unity でカメラロールから画像を選択したり、逆に画像を保存してみようと思います。
また、おまけとして画像のSNS共有もやります。よろしくお願いします。

# 動作確認環境

- Unity : 2022.3.16f1
- VideoKit : [0.0.17](https://github.com/natmlx/videokit)
- Android : Pixel6 (Android 14)
- iOS : iPhone15 (iOS 17.4)

# 導入方法

## パッケージの導入

UPMで入れていきます。
`manifest.json` の `scopedRegistries` と `dependencies`ß の項目に次の2つを追記してください。

```json
{
  "scopedRegistries": [
    {
      "name": "NatML",
      "url": "https://registry.npmjs.com",
      "scopes": [
        "ai.natml",
        "ai.fxn"
      ]
    }
  ],
  "dependencies": {
    "ai.natml.videokit": "0.0.17",
  }
}
```

できたら Unity に戻り、ロードが終わればパッケージの追加が完了しました。

## APIキーの取得

次にAPIキーの取得を行っていきます。

下記の NatML のトップページにアクセスし、ログインしてください

<https://hub.natml.ai/>

ログインしたら下記リンクから開発者向けコンソールへ行きましょう。

<https://hub.natml.ai/account/developers>

右上野 `+ Create` ボタンをクリックすることでAPIキーが取得できます。
取得に当たって、クレジットカードなどの支払い情報は特に要求されません。

![alt text](/images/videokit_share_1.png)

## Project　Settingの変更

先ほど取得したAPIキーを設定すると共に、iOSで必須となるカメラロールのアクセス周りの権限の設定をやって行きましょう。

まず、 Project Settings => Videokit へ行き、下記の画像のようにAPIキーの設定とフォトライブラリーへアクセスする際の説明文書を入力しましょう。

![alt text](/images/videokit_share_2.png)

次に、 Project Settings => Player Settings => iOS => cameraUsageDescription にて カメラへアクセスする際の説明文言を入力しましょう。

![alt text](/images/videokit_share_3.png)

本記事で実装する機能では特にカメラを利用することはないですが、Videokitに含まれるほかの機能にてカメラを参照する物があり、ビルド時に `cameraUsageDescription` の項目が空だとビルドが止まってしまうためです。

最後に、Project Settings => Playe Settings =? Android => AndroidMinSdkVersion　にて、 `Android 7.0 'Nougat' (API level 24)` 以上を選択しましょう。
VideoKit側の Mimimum APIが24となっているためです。

![alt text](/images/videokit_share_4.png)

## 機能の実装

今回はこんなデモを実装してみました。
(画像は SNS Shareボタンタップ時の画面です)

<https://youtube.com/shorts/6g1bz5sl-Vs>

![alt text](/images/videokit_share_5.png)

先にコード全体を貼っておきます。

```cs
using UnityEngine;
using UnityEngine.UI;
using VideoKit.Assets;

public class ShareSample : MonoBehaviour
{
    [SerializeField] private RawImage image;
    [SerializeField] private Button uploadBtn;
    [SerializeField] private Button saveBtn;
    [SerializeField] private Button snsBtn;

    private Texture2D _uploadTexture;

    private Texture2D UploadTexture
    {
        get => _uploadTexture;
        set
        {
            _uploadTexture = value;
            if (_uploadTexture == null) return;

            image.texture = _uploadTexture;
        }
    }

    private void Start()
    {
        uploadBtn.onClick.AddListener(async () =>
        {
            var asset = await MediaAsset.FromCameraRoll<ImageAsset>() as ImageAsset;
            UploadTexture = await asset.ToTexture();
        });

        saveBtn.onClick.AddListener(async () =>
        {
            ImageAsset asset = await MediaAsset.FromTexture(UploadTexture);
            bool saved = await asset.SaveToCameraRoll();
        });

        snsBtn.onClick.AddListener(async () =>
        {
            ImageAsset asset = await MediaAsset.FromTexture(UploadTexture);
            var appId = await asset.Share();
        });
    }
}
```

それぞれの機能の実装を下記にまとめます。

## 画像のアップロード

```cs
var asset = await MediaAsset.FromCameraRoll<ImageAsset>() as ImageAsset;
UploadTexture = await asset.ToTexture();
```

## 画像の保存

```cs
ImageAsset asset = await MediaAsset.FromTexture(UploadTexture);
bool saved = await asset.SaveToCameraRoll();
```

## 画像のSNS共有

```cs
ImageAsset asset = await MediaAsset.FromTexture(UploadTexture);
var appId = await asset.Share();
```

# 終わりに

以上で本記事は終了となります。
意外と簡単にカメラロールへアクセスできたのではないでしょうか？
NatMLは共有機能を[NatShare](https://github.com/natmlx/natshare)というライブラリに分けてリリースしていました。
しかし、昨年のタイミングでML機能を搭載したVideoKitに統合してしまいました。
NatShareの時の方が取り回しが楽だったのですが、使えない物は仕方なしということでVideoKitを使っていきましょう！
ありがとうございました！