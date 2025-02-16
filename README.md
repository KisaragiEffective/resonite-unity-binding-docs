# Resonite to Unity binding documentations

このドキュメントは暫定的なものなので、わからなかったら適宜ググるなどしてください。

## ライセンス
CC0-1.0

間違っていても上記ライセンスの定めにより許容される最大限の範囲で免責されます。

## Unity とは？

1. Resonite が現時点で内部的に採用しているレンダリングエンジンです。
2. より一般には、FrooxEngine と同様のゲームエンジンです。

## 用語のマッピング

話を簡潔にするために、先に用語のマッピングを定義しておきます。

* Slot -> GameObject
* Attach Component -> Add Component
* Context Menu -> Expression Menu

逆に、同じ用語もあります。
* Shader
* Material
* Mesh
* Bone
* Texture2D - 2次元のテクスチャのこと
* Texture3D - 3次元のテクスチャのこと
* RenderTexture - 描画の宛先とすることで動的に内容を変えることができるテクスチャのこと
* UV

## データ構造の違い

Resonite と Unity はデータ構造が異なります。前提となる知識のため読み飛ばさないようにしてください。

### テクスチャ

例えば StaticTexture2D コンポーネントは皆さん毎日のようにお使いかと思いますが、 Unity においては PNG 形式や JPEG 形式の画像を D&D すると勝手に Texture2D として認識されます。

また、 RenderTexture は独立したファイルとして管理されています。

### マテリアル

例えば XiexeToonMaterial は Resonite においてコンポーネントとして存在しますが、 Unity においては単なるマテリアルとして独立したファイルで保存されます。各 GameObject が持つレンダラーはそれを参照するだけです。

### シェーダー

マテリアルは内部的にシェーダーを持っていますが、シェーダーも独立したファイルとして存在します。現状の Resonite とは異なり、自分で好きなシェーダーを書くこともできます。

以下に、シェーダーのマッピングの概略を示します:

* PBS -> Standard
* XiexeToon -> XiexeToon, lilToon, ...
* FurMaterial -> lilToon w/ Fur
* ...

## VRChat と Unity の関係性

ご存知の通り、 VRChat は Unity を使用して様々な表現を可能にしています。しかし、その表現のエンコード方法はResoniteに慣れていると少々奇特に感じることでしょう。

ここからは、VRChat の アバターSDK を中心にフォーカスします。

## アバターがアバターとしてみなされる要件

VRChat Avatar Descriptor というコンポーネントをつけることです。
Animator というコンポーネントがついていると思いますが、それも**ほとんどの場合で必須**です。**消さないように**してください。
Animator が何をするのかは下で解説するので、今はそういうものだと思っておいてください。

参考: <https://creators.vrchat.com/avatars/creating-your-first-avatar/>

## 変数

VRChatのアバターにおいて 変数は Animator コンポーネントの Animator Controller が持つ Parameters (をラップしたデータ構造) で表現されます。

### 変数のデータ型

具体的には、以下のデータ型しかやりくりできません。

* Int
* Bool
* Float

### 変数の値域

* Int: 0-255
* Bool: True or False
* Float: ローカル: Float32と同様 / リモート: 0-1、1/255 精度

### 変数の最大個数

VRChat では変数の最大個数がビット数によって決定されます。
それぞれのデータ型ごとの消費ビット数は以下のとおりです:

* Int: 8ビット
* Bool: 1ビット
* Float: 8ビット

上限となるビット数は、
* 同期されるパラメーター: 256ビット
* 同期されないパラメーター: 8192ビット

です。

## どのようにボーンを認識しているのか

話を簡単にするためにヒューマノイドなアバターのみ説明します。人間の形をしていたら大体ヒューマノイドです。

1. モデルを Unity にインポートするときは Unity により命名及び階層のマッチングが試みられ、 [Avatar][UnityEngine.Avatar] (※ユーザーが操作するアバターとは別) というデータ構造で保存されます。
2. モデルを使う場合、上記の [Avatar][UnityEngine.Avatar] というデータ構造からヒューマノイドの構造に対応するボーンを引っ張ってきます。

[Avatar][UnityEngine.Avatar] というデータ構造は Head Proxy などのプロキシや VRIK コンポーネントの代わりになります。

また、 CenteredRoot という名前のGameObjectは存在しません。大体の場合、 Armature という名前のGameObjectの下にあるGameObjectがボーンを表現しています。使用しているアバターによっては Armature という名前ではないかもしれませんが、構造はほとんど同じです。

[UnityEngine.Avatar]: https://docs.unity3d.com/ja/2022.3/Manual/class-Avatar.html

## どのようにオブジェクトのオンオフを切り替えているのか

昨今では MA ObjectToggle や LI Prop という便利なコンポーネントがありますが、基礎を理解するためにはこのセクションは避けては通れない項目です。絶対に避けては通れない項目なので、読み飛ばさないようにしてください。

Unity は モデルをインポートするときに Animator というコンポーネントを Add Component します。この Animator というコンポーネントは VRChat において **非常に重要** であり、これなしにオブジェクトのオンオフを切り替えることはできません。

さっきアタッチした VRC Avatar Descriptor の中に「Base」「Action」「FX」という項目があると思います。FXは [Foreign eXchange](https://ja.wikipedia.org/wiki/%E5%A4%96%E5%9B%BD%E7%82%BA%E6%9B%BF%E5%B8%82%E5%A0%B4) のことではありません。

基本的にオブジェクトのオンオフは FX レイヤーで行うという[約束][VRC.Avatar.PlayableLayers]になっています。FX レイヤーの中にオンオフを切り替えるためのアニメーションを実装するため、以下の手順を踏みます。
1. (初回のみ) Animator Controller を作って
2. 補完が効くように Animator コンポーネントに作った Animator Controller を割り当てて
3. Animator Controller の中にオンのステートとオフのステートを作って
4. bool のパラメーターを追加して
5. VRChat Avatar DescriptorのParametersにパラメーターの名前を追加して
6. それぞれのステートに紐づくAnimation Clip で GameObject の Is Active を割り当てて
7. 無限ループするようにアニメーションの編集画面で右側のキーフレームを破壊して
8. Animation Clip をステートにアサインして
9. ステートを遷移で結合して
10. 遷移条件に bool のパラメーターを追加して
11. 即時切り替わるようにするために遷移のtransition time を 0 にして
12. Animator コンポーネントに作った Animator Controller の割当を解除する

ProtoFlux に比べるとめちゃめちゃ面倒ですがそういうものなのでなれてください。なぜならこれしか制御する方法がないからです。

……ですが、先程も述べたようにあくまでそれが基礎であり、トラブルシューティングで必要な知識というだけで、実用上は覚える必要はありません。
最近では[MA ObjectToggle][dev.nadena.modular_avatar.ObjectToggle] や [LI Prop][lilxyzw.lilycalinventory.LIProp] という便利なコンポーネントで以上のことが抽象化されているため、そちらを使うようにしましょう。

[dev.nadena.modular_avatar.ObjectToggle]: https://modular-avatar.nadena.dev/ja/docs/tutorials/object_toggle
[lilxyzw.lilycalinventory.LIProp]: https://lilxyzw.github.io/lilycalInventory/ja/docs/components/prop.html
[VRC.Avatar.PlayableLayers]: https://creators.vrchat.com/avatars/playable-layers/

## どのようにオブジェクトのパラメーターを切り替えているのか

基本的には上と同じですが、「GameObject の Is Active」という部分が他の適切なものに切り替わります。
網羅することは不可能なので自分で色々見てみてください。

## どのようにオブジェクトのパラメーターを補完しているのか

基本的には上と同じですが、Animation Clip を作る段を BlendTree という親戚に置き換える必要があります。

パラメーターの数により、作るべきバリアントが異なります。

1. 1つ: [BlendTree 1D][UnityEngine.BlendTree.Dim1]
2. 2つ: [BlendTree 2D][UnityEngine.BlendTree.Dim2]
3. 3つ以上: [DirectBlendTree][UnityEngine.BlendTree.Direct]

頑張ってください。

上級者向けのアドバイス: あなたが Write Defaults をオフにする信仰をしていたとしても、 DBT は Write Defaults をオンにしてください。

[UnityEngine.BlendTree.Dim1]: https://docs.unity3d.com/ja/2022.3/Manual/BlendTree-1DBlending.html
[UnityEngine.BlendTree.Dim2]: https://docs.unity3d.com/ja/2022.3/Manual/BlendTree-2DBlending.html
[UnityEngine.BlendTree.Direct]: https://docs.unity3d.com/ja/2022.3/Manual/BlendTree-DirectBlending.html

## どのように変数へ代入しているのか

様々な手法があります。

1. [Expression Menu](https://creators.vrchat.com/avatars/expression-menu-and-controls) から
2. [VRC Contact Receiver](https://creators.vrchat.com/avatars/avatar-dynamics/contacts/#receiver) から
3. [VRC Avatar Parameter Driver](https://creators.vrchat.com/avatars/state-behaviors/#avatar-parameter-driver) から
4. \[黒魔術\] [AAP](https://vrc.school/docs/Other/AAPs/) から









