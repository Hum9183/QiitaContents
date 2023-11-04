---
title: 【Maya】OpenMayaAnim.MFnSkinClusterの使い方【Python API 2.0】
tags:
  - maya
  - OpenMaya
  - Python
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
- この記事は**PythonAPI2.0**のMFnSkinClusterクラスの使い方をまとめたものです。
- PythonAPI1.0やAPI(C++)の使い方は載っていません

# 環境
Autodesk Maya 2024.1
(API2.0が動く環境であればどの環境でも問題ありません)
<font color="DeepPink">**TODO**</font>: 細かいバージョンを載せる

# MFnSkinClusterとは
skinClusterからウェイト情報を取得したり設定することが出来るクラスです。

[【Maya Python API 2.0 Reference】OpenMayaAnim.MFnSkinCluster](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_anim_1_1_m_fn_skin_cluster_html)

# なぜOpenMayaなのか
maya.cmdsのスキンウェイト情報を操作出来るコマンド「**skinCluster**」と比べて、
処理速度が圧倒的に速いためです。

[【Maya Python Command Reference】skinCluster](http://me.autodesk.jp/wam/maya/docs/Maya2009/CommandsPython/skinCluster.html)

# モジュールのインポート
```import_open_maya_2.py
import maya.api.OpenMaya as om2
import maya.api.OpenMayaAnim as oma2
```
使用するためにはmaya.api.OpenMayaAnimというモジュールをインポートします。
エイリアスをoma2としておくのがおすすめです。
maya.api.OpenMayaも一緒に使うことが多いので一緒にインポートしておきます。

# 大まかな手順
1. MFnSkinClusterをインスタンス化する
2. スキンウェイトをGetする
3. スキンウェイトをSetする

# 1. MFnSkinClusterをインスタンス化する
わりとややこしいので、サンプルコードを見ながら解説していきます。

以下は「選択しているバインド済メッシュからMFnSkinClusterインスタンスを生成する」例です。
```python
# -*- coding: utf-8 -*-
from maya import cmds, mel
import maya.api.OpenMaya as om2
import maya.api.OpenMayaAnim as oma2

def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0) # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()

    # skincluster
    skincluster_node = get_skincluster_node(mesh_dag_path.node())
    if skincluster_node is None:
        return False
    skincluster_fn = oma2.MFnSkinCluster(skincluster_node)

    print(type(skincluster_fn))

def get_skincluster_node(mesh_node):
    # type: (om2.MObject) -> om2.MObject
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream)
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None

if __name__ == '__main__':
    main()
```
一つずつ解説していきます。

### 1.1 イニシャライザの確認
MFnSkinClusterのイニシャライザのシグネチャは以下です。
```python
def __init__(self, object):
    # type: (om2.MObject) -> None
    ...
```
MFnSkinClusterの生成にはMObject型のインスタンスが必要になります。

MFnSkinClusterに行き着くまでに色々な型を経由するのですが、
大まかな流れは以下です。
- バインドされたメッシュのインスタンス(MDagPath型)を生成
↓
- スキンクラスターノードのインスタンス(MObject)を生成
↓
- スキンクラスターノードのインスタンス(MFnSkinCluster型)を生成

ちなみにPythonAPIのイニシャライザのシグネチャはリファレンスに載っていないため、
C++のリファレンスを参照してください。
[【Maya API C++ Reference】OpenMayaAnim.MFnSkinCluster](https://download.autodesk.com/us/maya/2009help/API/class_m_fn_skin_cluster.html)

### 1.2 選択項目を取得
```python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
```
MDagPathを生成する前段階として、現在の選択項目を取得します。
OpenMayaでは、MGlobalクラスの**getActiveSelectionList**メソッドを使用します。
これはmaya.cmdsでいうところのcmds.lsにあたります。
返り値の型はlist[str]ではなくMSelectionList型です。

[【Maya Python API 2.0 Reference】OpenMaya.MGlobal](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_global_html)
[【Maya Python API 2.0 Reference】OpenMaya.MSelectionList](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_selection_list_html)

### 1.3 MDagPathを取得
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
+   dag_path = active_sel_list.getDagPath(0) # type: om2.MDagPath
```
MSelectionListクラスの**getDagPath**メソッドを使い、選択項目のMDagPathを取得します。
引数は取り出したい選択項目のIndexです（listの添字だと思ってください）
今回は1つ目の選択項目から取り出したいため、0を渡します。

### 1.4 メッシュのMDagPathを取得
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
+   mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
```
ビューポート等からメッシュを選択した場合、meshノードではなくtransformノードを選択しています。
MDagPathクラスの**extendToShape**メソッドでシェイプ(メッシュ)に変換します。

[【Maya Python API 2.0 Reference】OpenMaya.MDagPath](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_dag_path_html)

### 1.5 skinClusterノードを取得(その1)
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
    mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
+   skincluster_node = get_skincluster_node(mesh_dag_path.node()) # type: om2.MObject

+ def get_skincluster_node(mesh_node):
+   ...
```
メッシュからスキンクラスターノードを探します。
ノードからノードを探す処理は、OpenMayaだとやや長い処理が必要になるため、関数を用意します。
ちなみにmaya.cmdsであれば、cmds.listHistoryで簡単に取得できます。

[【Maya Python Command Reference】listHistory](http://me.autodesk.jp/wam/maya/docs/Maya2010/CommandsPython/listHistory.html)

### 1.6 skinClusterノードを取得(その2)
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
    mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node()) # type: om2.MObject

def get_skincluster_node(mesh_node):
+   dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream)
+   while not dg_iterator.isDone():
+       m_object = dg_iterator.currentNode()
+       if m_object.hasFn(om2.MFn.kSkinClusterFilter):
+           return m_object
+       dg_iterator.next()
+   return None
```
ノードを探す処理は**MItDependencyGraph**クラスを使用します。
ここではskinClusterノードを探したいため、イニシャライザの第2引数に**MFn.kSkinClusterFilter**を渡しています。

[【Maya Python API 2.0 Reference】OpenMaya.MItDependencyGraph](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_it_dependency_graph_html)
[【Maya Python API 2.0 Reference】OpenMaya.MFn](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_fn_html)

### 1.7 MFnSkinClusterを生成
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
    mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node()) # type: om2.MObject
+   skincluster_fn = oma2.MFnSkinCluster(skincluster_node) # type: oma2.MFnSkinCluster

def get_skincluster_node(mesh_node):
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream)
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None
```
ここでやっと**MFnSkinCluster**が出てきました。
先程取得したskinClusterノードをイニシャライザに渡し、MFnSkinClusterをインスタンス化します。

[【Maya Python API 2.0 Reference】OpenMayaAnim.MFnSkinCluster](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_anim_1_1_m_fn_skin_cluster_html)

# 2. スキンウェイトをGetする
インスタンスが用意できたので早速スキンウェイトをGetしてみます。
GetするにはMFnSkinClusterクラスの**getWeights**メソッドを使用します。

getWeightメソッドのシグネチャは以下です(オーバーロードが3つ)
```diff_python
def getWeights(shape, components):
    # type: (om2.MDagPath, om2.MObject) -> (om2.MDoubleArray, int)
    ...
def getWeights(shape, components, influence):
    # type: (om2.MDagPath, om2.MObject, int) -> om2.MDoubleArray
    ...
def getWeights(shape, components, influences):
    # type: (om2.MDagPath, om2.MObject, om2.MIntArray) -> om2.MDoubleArray
    ...
```
## 2.1 引数解説
1つずつ引数を確認していきます。

### 2.1.1 shape
>リファレンスより
>* shape (MDagPath) - the object being deformed by the skinCluster

とあるので、要はバインドされたメッシュのことです。
```python
active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
dag_path = active_sel_list.getDagPath(0) # type: om2.MDagPath
mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
```
上記の`mesh_dag_path`をそのまま使えばOKです。

### 2.1.2 components
>リファレンスより
>* components (MObject) - components to return weights for

少々説明が少ないですが、要はコンポーネント(頂点やエッジ)のことです。

ここではコンポーネントの取得方法を2つ紹介します。
- 選択しているコンポーネントを取得する
- すべてのコンポーネントを取得する

##### 選択しているコンポーネントを取得する
```python
active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
dag_path, m_object_component = active_sel_list.getComponent(0) # type: (om2.MDagPath, om2.MObject)
```
MSelectionListクラスの**getComponent**メソッドで選択中のコンポーネントを取得できます。
ちなみに返り値はタプルになっていて、0番目がMDagPath、1番目がMObjectになっています。
必要なのは1番目のMObjectのみですが、0番目のMDagPathも大抵どこかの処理で使うので、
同時に取得してしまうことが多いです。

##### すべてのコンポーネントを指定する
```python
single_idx_comp_fn = om2.MFnSingleIndexedComponent() # type: om2.MFnSingleIndexedComponent
vert_comp = single_idx_comp_fn.create(om2.MFn.kMeshVertComponent) # type: om2.MObject
```
MFnSingleIndexedComponentクラスの**create**メソッドで生成します。
上記の例だと頂点コンポーネントを取得していますが、エッジでもフェースでも問題ありません
<font color="DeepPink">**TODO**</font>: 本当に問題ないか確認する

[【Maya Python API 2.0 Reference】OpenMaya.MFnSingleIndexedComponent](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_fn_single_indexed_component_html)

### 2.1.3 influence
>リファレンスより
>* influence (int) - index of the single influence to return weights for

骨のインデックスです。
特定の骨のスキンウェイトを取得したい場合に使用することになります。

取得方法はユースケースによるところが大きいので、
汎用的に使えそうなメソッドを2つ紹介します
- MFnSkinCluster.influenceObjects()
- MFnSkinCluster.indexForInfluenceObject()

##### MFnSkinCluster.influenceObjects()
```python
influences = skincluster_fn.influenceObjects() # type: om2.MDagPathArray
```
MFnSkinClusterクラスの**influenceObjects**メソッドです。
スキンクラスタに紐づいている骨をMDagPathArray型(MDagPathのlistのようなもの)で受け取ります。

参考程度ですが、骨の名前を確認したい場合は以下のような書き方をします
```python
def print_influence_names(skincluster_fn):
    # type: (oma2.MFnSkinCluster) -> None
    infl_dag_path_array = skincluster_fn.influenceObjects() # type: om2.MDagPathArray
    for infl_dag_path in infl_dag_path_array:
        print(infl_dag_path.fullPathName()) # ロングネーム
        print(infl_dag_path.__str__()) # ショートネーム
```

##### MFnSkinCluster.indexForInfluenceObject()
```python
index = skincluster_fn.indexForInfluenceObject(influence_dag_path) # type: int
```
MFnSkinClusterクラスの**indexForInfluenceObject**メソッドです。
MDagPath型の骨を渡し、その骨のindexを受け取ります。

### 2.1.4 influences
>リファレンスより
>* influences (MIntArray) - indices of multiple influences to return weights for

influenceが複数に対応した版です。

ちなみにMIntArraryはlistにキャストできます。
```python
def to_int_list(m_int_array):
    # type: (om2.MIntArray) -> list[int]
    return list(m_int_array)

def to_m_int_array(int_list):
    # type: (list[int]) -> om2.MIntArray
    return om2.MIntArray(int_list)
```
Json等にシリアライズする際に必要になるかもしれません。

[【Maya Python API 2.0 Reference】OpenMaya.MIntArray](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_int_array_html)

## 2.2 使ってみる
実際に使ってみます。

なお共通処理として、下記関数はすでに定義済みとします。
```python
def create_skincluster_fn():
    # type: () -> (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0) # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node()) # type: om2.MObject
    skincluster_fn = oma2.MFnSkinCluster(skincluster_node) # type: om2.MFnSkinCluster
    return skincluster_fn, mesh_dag_path, m_object_component

def get_skincluster_node(mesh_node):
    # type: (om2.MObject) -> om2.MObject
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream)
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None

def create_vertex_component():
    # type: () -> om2.MObject
    single_idx_comp_fn = om2.MFnSingleIndexedComponent() # type: om2.MFnSingleIndexedComponent
    return single_idx_comp_fn.create(om2.MFn.kMeshVertComponent) # type: om2.MObject

def get_influence_index_by_name(skincluster_fn, influence_name):
    # type: (oma2.MFnSkinCluster, str) -> int
    influences = skincluster_fn.influenceObjects() # type: om2.MDagPathArray
    for influence in influences:
        if influence.__str__() == influence_name:
            return skincluster_fn.indexForInfluenceObject(influence) # type: int
    return None
```

### 2.2.1 getWeights(shape, components)
```スキンウェイトを取得する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn()
    skin_Weights = skincluster_fn.getWeights(mesh_dag_path, m_object_component)
    print(skin_Weights)
```

ちなみにすべてのコンポーネントを取得する場合は下記になります。
```スキンウェイトを取得する(すべてのコンポーネント).py
def main():
    skincluster_fn, mesh_dag_path, _ = create_skincluster_fn()
    skin_Weights = skincluster_fn.getWeights(mesh_dag_path, create_vertex_component())
    print(skin_Weights)
```

### 2.2.2 getWeights(shape, components, influence)
```名前がHeadの骨のスキンウェイトを取得する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn()
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')
    head_skin_Weights = skincluster_fn.getWeights(mesh_dag_path, m_object_component, head_infl_index)
    print(head_skin_Weights)
```

### 2.2.3 getWeights(shape, components, influences)
```名前がHeadとNeckの骨のスキンウェイトを取得する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn()
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')
    neck_infl_index = get_influence_index_by_name(skincluster_fn, 'Neck')
    indices = om2.MIntArray([head_infl_index, neck_infl_index])
    head_neck_skin_Weights = skincluster_fn.getWeights(mesh_dag_path, m_object_component, indices)
    print(head_neck_skin_Weights)
```

## 2.3 取得したスキンウェイトデータの読み方
getWeightsメソッドで取得したスキンウェイトデータはMDoubleArray型です。
少し癖があるので、読み方を説明します。

頂点が4つのメッシュとそれに骨を3つバインドしたデータで解説します。

![MFnSkinCluster01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/ebba1ab1-b281-e770-e75f-eb14ff0fcdd4.png)

![MFnSkinCluster02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/34e6cb54-e44a-4a6b-7e52-073604a3cd14.png)

## 2.3.1 getWeights(shape, components)
返り値は(MDoubleArray, int)です。

#### MDoubleArray
- 頂点番号0と3にSpineのウェイト
- 頂点番号1にNeckのウェイト
- 頂点番号2にHeadのウェイト
を振ったもので確認してみます。

```python
[
    1.0, 0.0, 0.0,
    0.0, 1.0, 0.0,
    0.0, 0.0, 1.0,
    1.0, 0.0, 0.0
]

[
    vtx0のidx0, vtx0のidx1, vtx0のidx2,
    vtx1のidx0, vtx1のidx1, vtx1のidx2,
    vtx2のidx0, vtx2のidx1, vtx2のidx2,
    vtx3のidx0, vtx3のidx1, vtx3のidx2
]
```
頂点1つのウェイトを骨のインデックス順に羅列されています。
すべての骨のインデックスを羅列し終えたら、次の頂点の羅列が始まります。

#### int
```
3
```
インフルエンス数が返ってきます。

## 2.3.2 getWeights(shape, components, influence)
