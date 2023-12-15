---
title: "VYamlでUnityYamlと和解せよ"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity","csharp","yaml","dotnet"]
published: true
published_at: "2023-12-15 00:00"
---

# はじめに

都内で大学生をしているmattunです。二郎系ラーメンを食ったあとにこの記事を書いているのでお腹が痛いです。

この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の15日目です。

# TL;DR

VYamlってのを使うとUnityの変なYamlとも和解できるよ！

# UnityとYaml

実はUnityの内部で結構使われています。
よくあるのだと、PrefabなどのアセットやScene、ProjectSettingがあげられます。


# UnityのYamlって何が変なの？

例えば、以下のPrefabがあったとします。

![Alt text](/images/adcal_2023_yvaml_1.png)

そのPrefabをテキストエディタで開くと以下のようになります。

```yaml
%YAML 1.1
%TAG !u! tag:unity3d.com,2011:
--- !u!1 &980850718327413238
GameObject:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  serializedVersion: 6
  m_Component:
  - component: {fileID: 5247112485843355343}
  - component: {fileID: 1132035523716758630}
  - component: {fileID: 7185744008649528604}
  - component: {fileID: 793798646269218595}
  m_Layer: 6
  m_Name: Cube
  m_TagString: FindMe
  m_Icon: {fileID: 0}
  m_NavMeshLayer: 0
  m_StaticEditorFlags: 0
  m_IsActive: 1
--- !u!4 &5247112485843355343
Transform:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_GameObject: {fileID: 980850718327413238}
  serializedVersion: 2
  m_LocalRotation: {x: 0, y: 0, z: 0, w: 1}
  m_LocalPosition: {x: 0, y: 0, z: 0}
  m_LocalScale: {x: 1, y: 1, z: 1}
  m_ConstrainProportionsScale: 0
  m_Children: []
  m_Father: {fileID: 0}
  m_LocalEulerAnglesHint: {x: 0, y: 0, z: 0}
--- !u!33 &1132035523716758630
MeshFilter:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_GameObject: {fileID: 980850718327413238}
  m_Mesh: {fileID: 10202, guid: 0000000000000000e000000000000000, type: 0}
--- !u!23 &7185744008649528604
MeshRenderer:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_GameObject: {fileID: 980850718327413238}
  m_Enabled: 1
  m_CastShadows: 1
  m_ReceiveShadows: 1
  m_DynamicOccludee: 1
  m_StaticShadowCaster: 0
  m_MotionVectors: 1
  m_LightProbeUsage: 1
  m_ReflectionProbeUsage: 1
  m_RayTracingMode: 2
  m_RayTraceProcedural: 0
  m_RenderingLayerMask: 1
  m_RendererPriority: 0
  m_Materials:
  - {fileID: 10303, guid: 0000000000000000f000000000000000, type: 0}
  m_StaticBatchInfo:
    firstSubMesh: 0
    subMeshCount: 0
  m_StaticBatchRoot: {fileID: 0}
  m_ProbeAnchor: {fileID: 0}
  m_LightProbeVolumeOverride: {fileID: 0}
  m_ScaleInLightmap: 1
  m_ReceiveGI: 1
  m_PreserveUVs: 0
  m_IgnoreNormalsForChartDetection: 0
  m_ImportantGI: 0
  m_StitchLightmapSeams: 1
  m_SelectedEditorRenderState: 3
  m_MinimumChartSize: 4
  m_AutoUVMaxDistance: 0.5
  m_AutoUVMaxAngle: 89
  m_LightmapParameters: {fileID: 0}
  m_SortingLayerID: 0
  m_SortingLayer: 0
  m_SortingOrder: 0
  m_AdditionalVertexStreams: {fileID: 0}
--- !u!65 &793798646269218595
BoxCollider:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  m_GameObject: {fileID: 980850718327413238}
  m_Material: {fileID: 0}
  m_IncludeLayers:
    serializedVersion: 2
    m_Bits: 0
  m_ExcludeLayers:
    serializedVersion: 2
    m_Bits: 0
  m_LayerOverridePriority: 0
  m_IsTrigger: 0
  m_ProvidesContacts: 0
  m_Enabled: 1
  serializedVersion: 3
  m_Size: {x: 1, y: 1, z: 1}
  m_Center: {x: 0, y: 0, z: 0}
```

一番変な点は、 ``--- !u!1 &980850718327413238`` ではないでしょうか。
こちらは ``--- !u!{ClassID} &{FileID}`` という規則に従っています。

ClassIDはGameObjectやBox Colliderなどのクラス名となっており、下記のリンクにUnity標準クラスのClassIDの一覧が掲載されています。
https://docs.unity3d.com/ja/current/Manual/ClassIDReference.html

FileIDはそれぞれのアタッチされたコンポーネントごとに一意に振られているIDです。
こちらはオブジェクトの参照関係を示す際に使用されています。

詳しくはUnity公式がかなり丁寧に説明されているので、こちらを一読いただけると理解が深まるんじゃないかと思います。
https://blog.unity.com/ja/engine-platform/understanding-unitys-serialization-language-yaml

# この状態からでもDeserialiseできるYamlParserがあるんですか！？

.NETだと [YamlDotNet](https://github.com/aaubry/YamlDotNet) がポピュラーだと思いますが、UnityYamlはガラパゴスなフォーマットなのでこやつで使うためには工夫が必要です。
([工夫してDeserialiseして可視化までした偉大なる先人の記事](https://qiita.com/satanabe1@github/items/515f206659177883c7f4))

しかし、その状況の中救いの手がVContainerでお馴染みのhadashiAさんより差し伸べられました！
VYamlというYamlParserです！

https://github.com/hadashiA/VYaml

VYamlの特徴としては以下が挙げられます。

- 比較的新しい.NETのAPIを使用して実装されたため高速
  - YamlDotNETと比較して6倍以上も高速
  - アロケーションコストが小さい
- YAML 1.2の仕様をほとんど網羅
- Unity 2021.3以降でも動かせる
  - Unityでの性能を特に重視されている

ざっくりまとめると最強のYaml Parserということです！

では、実際にYaml Parserを使ってみましょう。

# VYamlでPrefabを解析してみる

今回は指定したタグやレイヤが設定されてるPrefabのPathを列挙するCLIアプリを作成してみました。
GitHub ActionsなどのCI環境下でのチェックを想定しています。

動作環境は以下の通りです。

```
Unity : 2022.3.11f1
.NET : 7.0.101
```

以下、ソースコードです。

ConsoleAppやZxについてはこちらの記事で解説しています。
https://zenn.dev/matsuataru/articles/csharp_shell_comedian_adcal_2023

```cs
using Zx;
using static Zx.Env;
using VYaml.Serialization;

await ConsoleApp.RunAsync(args, MainAsync);
    
async Task<int> MainAsync(
    [Option("path")] string projectPath,
    [Option("tag")] string? searchTag = default,
    [Option("layer")] string? searchLayer = default
)
{
    if(searchLayer == default && searchTag == default)
    {
        log("Please specify --tag or --layer");
        return 1;
    }
    
    await $"cd {projectPath}";

    var projectSettingPath = Path.Combine(projectPath, "ProjectSettings", "TagManager.asset");

    var tagManager = YamlSerializer.Deserialize<dynamic>(File.ReadAllBytes(projectSettingPath));

    List<object> layers = tagManager["TagManager"]["layers"];
    
    var prefabFiles = Directory.GetFiles("./Assets", "*.prefab", SearchOption.AllDirectories);

    foreach (var filePath in prefabFiles)
    {
        var file = File.ReadAllBytes(filePath);

        var yaml = YamlSerializer.Deserialize<Dictionary<object, dynamic>>(file);

        if (!yaml.ContainsKey("GameObject"))
        {
            continue;
        }

        var gameObject = yaml["GameObject"];

        if (searchTag != default && gameObject["m_TagString"] == searchTag)
        {
            log($"Find Tag: {filePath}");
        }

        if (searchLayer != default && layers[gameObject["m_Layer"]] == searchLayer)
        {
            log($"Find Layer: {filePath}");
        }
    }

    return 0;
}
```

ここから解説していきます。

まず ``<projectRoot>/ProjectSettings/TagManager.asset`` から各レイヤーの文字列を取得しています。

Deserialzeする際はdynamic型も使用できます。
IDEによる支援を受けたい場合は自身で専用のClassを作成してもいいですが、部分的に ``List<object>`` などを使用するのも効果的です。

```cs
    var projectSettingPath = Path.Combine(projectPath, "ProjectSettings", "TagManager.asset");

    var tagManager = YamlSerializer.Deserialize<dynamic>(File.ReadAllBytes(projectSettingPath));

    List<object> layers = tagManager["TagManager"]["layers"];
```

レイヤーは下記のYamlからわかる通り、数値で保存されています。
そのため、文字列で検索をかけるにあたって事前に各レイヤーの文字列を取得しています。
また、タグに関してはそのまま文字列で保存されているため、TagManagerからの取得は不要となっています。

```yaml
GameObject:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_PrefabAsset: {fileID: 0}
  serializedVersion: 6
  m_Component:
  - component: {fileID: 5247112485843355343}
  - component: {fileID: 1132035523716758630}
  - component: {fileID: 7185744008649528604}
  - component: {fileID: 793798646269218595}
  m_Layer: 6
  m_Name: Cube
  m_TagString: FindMe
  m_Icon: {fileID: 0}
  m_NavMeshLayer: 0
  m_StaticEditorFlags: 0
  m_IsActive: 1
```

次に、 ``<projectRoot>/Assets/`` 内のPrefabを列挙しています。
Unityは意外と拡張子が用途によって使い分けられているので、いい感じにフィルタをかけれたりします。

```cs
    var prefabFiles = Directory.GetFiles("./Assets", "*.prefab", SearchOption.AllDirectories);
```

あとはそれぞれのPrefabごとにDeserializeして中身を見るだけです。

```cs

        var file = File.ReadAllBytes(filePath);

        var yaml = YamlSerializer.Deserialize<Dictionary<object, dynamic>>(file);

        if (!yaml.ContainsKey("GameObject"))
        {
            continue;
        }

        var gameObject = yaml["GameObject"];

        if (searchTag != default && gameObject["m_TagString"] == searchTag)
        {
            log($"Find Tag: {filePath}");
        }

        if (searchLayer != default && layers[gameObject["m_Layer"]] == searchLayer)
        {
            log($"Find Layer: {filePath}");
        }
```

# おわりに

また、今回作成したものは下記のリポジトリで公開しています。
https://github.com/AtaruMatsudaira/vyaml_handson

今回のサンプルはUnity内で動作させませんでしたが、実際にUnity内でも問題なく動作することを確認できています。
エディタ拡張などでも役に立つのではないでしょうか？

この記事をきっかけにVYamlを皆様が活用してくれると幸いです。

以上で[サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の15日目の記事は終わりとなります。
お付き合いいただき、ありがとうございました。
