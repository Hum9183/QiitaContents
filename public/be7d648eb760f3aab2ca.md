---
title: 【ArtistでもMELが書きたい】Maya MEL入門(9)【便利なコマンドとフラグ】
tags:
  - mel
  - maya
private: false
updated_at: '2023-06-05T23:44:06+09:00'
id: be7d648eb760f3aab2ca
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
この記事は、3DCG業界で働く**Artist向けのMEL入門記事**です。
普段の業務を効率化したいけど、MELの書き方が分からない、調べたけどイマイチ分からなかった、そんな方向けに書きました。
本記事は「便利なコマンドとフラグを知る」をゴールとします。

# 環境
Autodesk Maya 2022.2

# 前回のあらすじと今回へのつなぎ
[前回](https://qiita.com/Hum9183/items/9a1b4489a6d4e7a86092)は`ls`コマンドを使用しました。大変便利で強力なコマンドですが、MELには他にもたくさんの強力なコマンドがあります。今回は新要素の習得というよりはコマンドやフラグの紹介と、使用例の解説をしていきます。なおフラグにはショートネームを使用している箇所があります。

# その前に
出鼻をくじく形にはなってしまいますが、コマンドやフラグを理解するためにはある程度**Mayaの仕組み**の知識を必要とします。
まずは今回紹介するコマンドの理解に必要な知識を簡単に見ていきます。

# メッシュとトランスフォーム
突然ですが、メッシュの構成要素は何だと思いますか？
メッシュなのだからメッシュ1つで構成されていると思うかもしれません。
アウトライナで見ても1つにしか見えません。
![MEL_learning17.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/c29f11cd-a33d-fe03-d3e9-9b4752e3f0f2.png)
ですが、実はメッシュは**メッシュとトランスフォーム**で構成されています。
これは、アウトライナの**ディスプレイ**メニューの「**シェイプ**」にチェックを入れると確認することができます。
![MEL_learning18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/6eebc490-44c0-8f16-1dec-7697070195e1.png)
![MEL_learning19.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/69a94ea8-9e41-6430-a293-f32c8305b15c.png)
なにやら親子階層ができましたね。
これは「シェイプ」がオンになったことによって親子階層ができたのではなく、シェイプがオフのときは**表示が省略されていただけ**です。
早速どういう構造になっているのか見てみましょう。
![MEL_learning20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/729e1c52-0481-aeb4-65c2-3099e7cbc47a.png)
さっきまでメッシュだと思っていた「pCube1」がトランスフォームとして存在しており、その子供には「pCubeShape1」という**メッシュ**が存在しています。
これが、**メッシュはトランスフォームと一緒に構成されている**所以になります。

試しに「pCube1(トランスフォーム)」を選択して、チャネルボックスを見てみましょう。
![MEL_learning21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/41667d2a-5fe9-549f-3729-f15fa079fcde.png)
なんの変哲もないですが、これはメッシュではありません。ただのトランスフォームです。移動や回転・スケール等を司るもので、メッシュとは一切関係がありません。

次に「pCubeShape1(メッシュ)」を選択して、チャネルボックスを見てみましょう。
![MEL_learning22.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e5dc6b7b-ee7f-e3be-17af-1ab851ce9efc.png)
あれ、いつも見ているチャネルボックスとは少し違いますね。
これはトランスフォームのGUI部分が丸ごと存在しないからです。
メッシュは本当にメッシュの情報しか持っておらす、**移動や回転といった情報は持っていない**のです。
つまり、メッシュは**トランスフォームの子供になることによって移動や回転を擬似的に表現している**ということになります。

でもモデリングする際に、それを意識するのはめんどくさいですよね？
なので普段は省略されているのです。

# ノード
ノード。たまに聞きますが、実態はなんなのでしょうか？
少し抽象的な概念になってしまうので、実物を見て確認しましょう。
ウィンドウメニューからノードエディタを開いてください。
![MEL_learning23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/11ecc502-daa0-f4c1-19b0-107a0a21b216.png)
なんらかのオブジェクト(今回はpCube1)を選択した状態で下図のアイコンを押すと、**ノードを見ること**ができます。
![MEL_learning24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/7e9e7d08-0866-5aab-476a-d8b5308a6b42.png)
![MEL_learning25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/a5047403-095a-b38c-6263-f479080fac01.png)
この横長のものがノードです。
ノード上をマウスオーバーすると**ノード名とノードのタイプ**が表示されます。
![MEL_learning26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/7e1ba7ed-9227-91ea-9d29-bdd28066d4bb.png)
pCube1は**トランスフォームノード**のようですね。
![MEL_learning27.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/8d20ae94-d16d-7166-955f-d7906b7c39ac.png)
pCubeShape1は**メッシュノード**のようです。

ここでは深く言及しませんが、ヒストリも一つ一つがノードです。
例えばスキンのバインドをした際にできる「skinCluster」は**スキンクラスターノード**です。

Mayaはノードでできているという表現もあるくらいです。

https://area.autodesk.jp/column/tutorial/maya_atoz/node/

さて少し小難しい話をしたところで、次からはコマンドとフラグの紹介に入っていきます。

# objectType
オブジェクトのタイプを取得するコマンドです。
```
string $selections[] = `ls -sl`;
string $objectType = `objectType $selections[0]`;
print($objectType);
```
pCube1を選択した状態で上記を実行すると、
```
transform
```
がログにprintされます。

当然ですが、pCubeShape1を選択した状態で実行すれば、
```
mesh
```
とprintされます。

# ls -dagObjects
DAGオブジェクトをすべて選択するフラグです。

まずDAGとはなにかですが、簡単に言うと**親子階層を持てるノード**のことです。
たとえばpCube1はトランスフォームノードとメッシュノードから構成されますが、両方ともDAGです。ジョイントもDAGです。
全てのノードがDAGなのかというとそういうわけではありません。スキンクラスターノードは親子関係なんて持てませんよね。

https://knowledge.autodesk.com/ja/support/maya/learn-explore/caas/CloudHelp/cloudhelp/2020/JPN/Maya-Basics/files/GUID-5029CF89-D420-4236-A7CF-884610828B70-htm.html

使用例としては、選択したオブジェクトの階層下のオブジェクトも全て取得したい場合です。
試しに選択階層下の骨をすべて取得してみましょう。
![MEL_learning01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/0d1b1d7c-610b-d382-382b-32f5bc834b49.png)
一番上の階層の骨だけを選び、`dagObjects`フラグを使用すれば、
```
string $selDagObjs[] = `ls -sl -dagObjects`;
print($selDagObjs);
```
![MEL_learning02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/12418db3-ab99-ee0c-e522-77a05f760b28.png)
階層下の骨をすべて取得できます。

# ls -flatten
コンポーネントを一つ一つ取得できるフラグです。

そもそもとしてですが、頂点等のコンポーネントを選択して、
![MEL_learning03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/133ec7b8-2f1f-d65e-8844-d665cda3648a.png)
`ls -sl`コマンドを使うと、
![MEL_learning04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e3aaf3bd-297f-bb3e-b180-9b6093058649.png)
このように頂点番号が連番でまとめられてしまいます。**一つずつ取得したい**場合には不便ですよね。
そこで`flatten`フラグを使用すれば、
```
string $selvertices[] = `ls -sl -flatten`;
print($selvertices);
```
![MEL_learning05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/cb74dae9-7d86-391a-df9c-04d3b09df06c.png)
一つずつ取得できます。

# ls -long
オブジェクトの単品名ではなく、親階層を含めた名前を取得するフラグです。

例えばjoint5の名前を取得する場合、
![MEL_learning06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/20862ea2-cdac-44a6-30eb-20b7f7952964.png)
`long`フラグなしだと、
![MEL_learning07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/c52c2f1f-3bc4-eeb3-d040-919e2023a5e0.png)
オブジェクト名のみしか取得できません。
`long`フラグありだと、
```
string $selObjs[] = `ls -sl -long`;
print($selObjs);
```
![MEL_learning08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/679ed8d8-068c-a3b0-b8f7-4fdc0eadab36.png)
一番上の親からの階層を含めた**フルパス**を取得することができます。

# ls -type
オブジェクトのタイプでソートするフラグです。

そもそも、`ls`コマンドはフラグを指定しない場合は、下図のようにシーン上にある全てのものを取得します。
![MEL_learning11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/f4ebde55-8a22-1016-5022-1a13128d8aa4.png)
`sl`フラグはこの一覧の中から、**選択しているもの**だけに絞るわけです。
`type`フラグはこの一覧の中から、**オブジェクトのタイプ**で絞ることができます。
冒頭でも確認しましたが、オブジェクトのタイプはノードエディタで確認ができます。
例えば、ジョイントのノードをマウスオーバーすると、
![MEL_learning14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/8098e371-88e1-0760-d0e1-d4d4c5e902be.png)
ジョイントのタイプは`joint`だとわかります。

このようなシーン上で、
![MEL_learning12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/6ca0fb29-0d70-c141-facd-64c1cae34364.png)
![MEL_learning13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/12473619-f266-39a6-2430-359a6b5347ce.png)
ジョイントのみを取得したい場合は、`type`フラグで`"joint"`を指定すると、
```
string $allJoints[] = `ls -type "joint"`;
print($allJoints);
```
![MEL_learning15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e2dd761b-b42d-ac58-e1e8-468329f87e0f.png)
タイプが`joint`のオブジェクトのみを取得できます。

# ls -noIntermediate
中間オブジェクトを無視することができるフラグです。

まず**Intermediate**とはなにかですが、**中間オブジェクト**のことです。
デフォームを使用している際に確認することができます。代表的なものだとスキンのバインドですね。
![MEL_learning09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/2473b659-18bc-091d-0ebf-21c74cdb420a.png)
ノードエディタで確認ができます。
![MEL_learning10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/2fb78a52-d106-dcda-e65e-5b3159c864a8.png)
**Orig**というsuffixがついているノードが中間オブジェクトです。

https://knowledge.autodesk.com/ja/support/maya/learn-explore/caas/CloudHelp/cloudhelp/2020/JPN/Maya-CharacterAnimation/files/GUID-B4091023-A1F6-43D0-8866-97A47522ED13-htm.html

`type`フラグを使って`"mesh"`のみを取得しようとすると、
```
string $allMeshes[] = `ls -type "mesh"`;
print($allMeshes);
```
![MEL_learning16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/59d92fe1-796d-c1d4-0782-e98866c46742.png)
純粋な`"mesh"`のほかにも中間オブジェクトがついてきてしまいます。これは中間オブジェクトも`"mesh"`であるからです。
`noIntermediate`フラグを使用すると、
```
string $allMeshes[] = `ls -type "mesh" -noIntermediate`;
print($allMeshes);
```
![MEL_learning17.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/a0bbed9a-7337-de42-c124-92d373fe1d3e.png)
中間オブジェクトが含まれなくなります。

# listHistory
ヒストリをリストで取得するコマンドです。

使用例としては、「スキンのバインド」を行った際に生成される`skinCluster`ノードを取得する場合があります。
```
string $selections[] = `ls -sl`;
string $histories[] = `listHistory $selections[0]`;
print($histories);
```
バインド済みのモデルを選択した状態で、上記を実行してみると、
![MEL_learning28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/f05a53ba-3a7f-4616-a1ce-0ad4bcd78247.png)
```
pCubeShape1
skinCluster1
pCubeShape1Orig
joint1
joint2
joint3
```
なにやらたくさん出てきましたが、これらがヒストリです。
`skinCluster`のみが欲しい場合はここから更に絞り込みます。
実は絞り込みは`ls`コマンドで行うことが可能です。
```
string $sortedHistories[] = `ls -type "skinCluster" $histories`;
```
この書き方で絞り込みができます。タイプを`skinCluster`で絞るということですね。
絞り込み処理を追加して、再度実行してみると、
```
string $selections[] = `ls -sl`;
string $histories[] = `listHistory $selections[0]`;
string $sortedHistories[] = `ls -type "skinCluster" $histories`;
print($sortedHistories);
```
![MEL_learning29.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/c00e9e34-e585-c033-d042-13d661112d43.png)
うまくいきましたね。

# listRelatives
親子関係を取得するコマンドです。

使用例としては、メッシュのトランスフォームから、メッシュ本体を取得する場合があります。
冒頭でも説明した通り、メッシュはトランスフォームの子になっています。
なおかつ、ビューポートでメッシュを選択すると、**トランスフォーム**を選択することになります。

メッシュに対してなにかを行いたいときは、「アウトライナの**シェイプ**をオンにしてから、メッシュ本体を選択して実行する」、なんていうのは少々めんどくさいですよね。

なのでトランスフォームを選択した状態から、子供であるメッシュ本体を取得したいわけです。
```
string $selections[] = `ls -sl`;
string $children[] = `listRelatives $selections[0]`;
print($children);
```
メッシュを選択した状態で上記を実行すると、メッシュ本体を取得できます。

`listRelatives`コマンドには、他にも**親を取得するフラグ**等があり、大変便利です。
気になった場合はコマンドリファレンスを参照してみてください。

# select
選択を操作するコマンドです。

使用例としては至極単純で、オブジェクトを選択する場合等があります。
```
select pCube1;
```
シーン上にpCube1がある状態で、上記を実行すると、pCube1を選択できます。

逆に選択解除したい場合は、
```
select -clear;
```
上記を実行すると、選択を解除できます。

# おわりに
いかがでしたか？
今回紹介したコマンドは他にもたくさんのフラグを持っています。
適切なフラグを使えば更に凝ったことや便利なことが行えるようになります。
適切に扱うにはMayaへの理解も必要なので最初はちょっととっつきにくいかもしれませんが、少しずつ確実に理解して、コマンドやフラグを使いこなしてもらえると嬉しいです。

次回はこちら→[【ArtistでもMELが書きたい】Maya MEL入門(10)](https://qiita.com/Hum9183/items/76fd522ebb84f2f3bca3)

# 参考
https://tech-coyote.com/archives/384

http://cgjishu.net

https://apps.autodesk.com/MAYA/ja/Detail/Index?id=8115150172702393827&os=Win64&appLang=en




