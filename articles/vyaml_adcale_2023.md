---
title: "VYamlでUnityYamlと和解せよ"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity","csharp","yaml","dotnet"]
published: false
---

# はじめに

都内で大学生をしているmattunです。二郎系ラーメンを食ったあとにこの記事を書いているのでお腹が痛いです。

この記事は [サイバーエージェント24卒内定者 Advent Calendar](https://qiita.com/advent-calendar/2023/ca24engineer)の15日目の記事です。

# TL;DR

VYamlってのを使うとUnityの変なYamlとも和解できるよ！

# UnityとYaml

実はUnityの内部で結構使われています。
よくあるのだと、PrefabなどのアセットやScene、ProjectSettingがあげられます。


# UnityのYamlって何が変なの？

例えば、以下のPrefabがあったとします。

![Alt text](/images/adcal_2023_yvaml_1.png)

そのPrefabをテキストエディタで開くと以下のようになります。

```
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



https://github.com/hadashiA/VYaml


# VYamlでPrefabを解析してみる

動作環境は以下の通りです。

```
Unity : 2022.3.11f1
.NET : 7.0.101
```

また、今回作成したものは下記のリポジトリで公開しています。
https://github.com/AtaruMatsudaira/vyaml_handson

## Prefabのタグチェック

## TMP_Textのフォントサイズチェック

# おわりに
