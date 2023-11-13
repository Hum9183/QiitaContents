---
title: 【Maya】OpenMayaAnim.MFnSkinClusterの使い方【Python API 2.0】
tags:
  - Python
  - maya
  - OpenMaya
private: true
updated_at: '2023-11-13T22:15:46+09:00'
id: 5705d8963e138ac8be35
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
- この記事は**PythonAPI2.0**のMFnSkinClusterクラスの主要な使い方をまとめたものです。
- MFnSkinClusterクラスを網羅的に解説したものではありません。
- PythonAPI1.0やC++APIの使い方は載っていません。

# 確認環境
Maya 2024.1
Maya 2020.4
Maya 2018.7
(PythonAPI2.0が動く環境であればどの環境でも問題ないと思います)

# MFnSkinClusterとは
skinClusterからウェイト情報を取得したり設定することが出来るクラスです。

[【Maya Python API 2.0 Reference】OpenMayaAnim.MFnSkinCluster](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_anim_1_1_m_fn_skin_cluster_html)

# なぜOpenMayaなのか
処理速度が圧倒的に速いためです。
maya.cmdsにもスキンウェイト情報を操作出来るコマンド「**skinPercent**」がありますが、
処理速度が遅いため、実用的なツール作成にはやや不向きです。

[【Maya Python Command Reference】skinPercent](http://me.autodesk.jp/wam/maya/docs/Maya2010/CommandsPython/skinPercent.html)

# モジュールのインポート
```import_open_maya_2.py
import maya.api.OpenMaya as om2
import maya.api.OpenMayaAnim as oma2
```
使用するためには**maya.api.OpenMayaAnim**というモジュールをインポートします。
エイリアスをoma2としておくのがおすすめです。
maya.api.OpenMayaも一緒に使うことが多いので一緒にインポートしておきます。

# 大項目
本記事では大きく分けて、
1. MFnSkinClusterのインスタンス化
2. スキンウェイトの取得
3. スキンウェイトの設定

の3つについて解説します。

# 1. MFnSkinClusterのインスタンス化
わりとややこしいので、サンプルコードを見ながら解説していきます。

下記は「選択しているバインド済メッシュからMFnSkinClusterインスタンスを生成する」例です。
```python
# -*- coding: utf-8 -*-
import maya.api.OpenMaya as om2
import maya.api.OpenMayaAnim as oma2

def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node())   # type: om2.MDagPath
    skincluster_fn = oma2.MFnSkinCluster(skincluster_node)          # type: oma2.MFnSkinCluster
    print(skincluster_fn)

def get_skincluster_node(mesh_node):
    # type: (om2.MObject) -> om2.MObject
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream) # type: om2.MItDependencyGraph
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None

if __name__ == '__main__':
    create_skincluster_fn()
```
一つずつ解説していきます。

## 1.1 イニシャライザの確認
インスタンス化する前に、まずはイニシャライザを確認します。
MFnSkinClusterのイニシャライザのシグネチャは以下です。
```python
def __init__(self, object):
    # type: (om2.MObject) -> None
    ...
```
MFnSkinClusterの生成にはMObject型のインスタンスが必要になります。
そのMObject型のインスタンスを取得するためにはさらに色々な型を経由します。

ざっくりまとめると下記のような流れになります。
- バインドされたメッシュのインスタンス(MDagPath型)を取得
↓
- スキンクラスターノードのインスタンス(MObject)を取得
↓
- スキンクラスターノードのインスタンス(MFnSkinCluster型)を生成

ちなみにPythonAPIのイニシャライザのシグネチャはリファレンスに載っていないため、
C++のリファレンスを参照してください。
[【Maya API C++ Reference】OpenMayaAnim.MFnSkinCluster](https://download.autodesk.com/us/maya/2009help/API/class_m_fn_skin_cluster.html)

## 1.2 選択項目を取得
```python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
```
メッシュのMDagPathを取得する前段階として、現在の選択項目を取得します。
OpenMayaでは、MGlobalクラスの**getActiveSelectionList**メソッドを使用します。
これはmaya.cmdsでいうところのcmds.lsにあたります。
返り値の型はlist[str]ではなくMSelectionList型です。

[【Maya Python API 2.0 Reference】OpenMaya.MGlobal](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_global_html)
[【Maya Python API 2.0 Reference】OpenMaya.MSelectionList](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_selection_list_html)

## 1.3 MDagPathを取得
```diff_python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
+   dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
```
MSelectionListクラスの**getComponent**メソッドを使い、選択項目のMDagPathを取得します。
引数は取り出したい選択項目のIndexです（listの添字だと思ってください）
今回は1つ目の選択項目から取り出したいため、0を渡します。

同時にMObject型のcomponentも取得していますが、こちらは[後で](#212-components)解説します。

ちなみに純粋にMDagPathだけを取得したい場合は、**getDagPath**メソッドを使います。
```diff_python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()  # type: om2.MSelectionList
+   dag_path = active_sel_list.getDagPath(0)                # type: om2.MDagPath
```

componentは後々必要になることがあるので、
ユースケースで使い分けてください。

## 1.4 メッシュのMDagPathを取得
```diff_python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
+   mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
```
ビューポート等からメッシュを選択した場合、meshノードではなくtransformノードを選択しています。
なのでMDagPathクラスの**extendToShape**メソッドを使い、シェイプ(メッシュ)に変換します。

[【Maya Python API 2.0 Reference】OpenMaya.MDagPath](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_dag_path_html)

## 1.5 skinClusterノードを取得(その1)
```diff_python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
+   skincluster_node = get_skincluster_node(mesh_dag_path.node())   # type: om2.MObject

+ def get_skincluster_node(mesh_node):
+   ...
```
メッシュからスキンクラスターノードを探します。
ノードからノードを探す処理は、OpenMayaだとやや長い処理が必要になるため、関数を用意します。
ちなみにmaya.cmdsであれば、cmds.listHistoryで簡単に取得できます。

[【Maya Python Command Reference】listHistory](http://me.autodesk.jp/wam/maya/docs/Maya2010/CommandsPython/listHistory.html)

## 1.6 skinClusterノードを取得(その2)
```diff_python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node())   # type: om2.MObject

def get_skincluster_node(mesh_node):
+   dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream) # type: om2.MItDependencyGraph
+   while not dg_iterator.isDone():
+       m_object = dg_iterator.currentNode()
+       if m_object.hasFn(om2.MFn.kSkinClusterFilter):
+           return m_object
+       dg_iterator.next()
+   return None
```
ノードを探す処理は**MItDependencyGraph**クラスを使用します。
ここではskinClusterノードを探したいため、イニシャライザの第2引数に**MFn.kSkinClusterFilter**を渡します。

[【Maya Python API 2.0 Reference】OpenMaya.MItDependencyGraph](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_it_dependency_graph_html)
[【Maya Python API 2.0 Reference】OpenMaya.MFn](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_fn_html)

## 1.7 MFnSkinClusterを生成
```diff_python
def create_skincluster_fn():
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node())   # type: om2.MObject
+   skincluster_fn = oma2.MFnSkinCluster(skincluster_node)          # type: oma2.MFnSkinCluster

def get_skincluster_node(mesh_node):
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream) # type: om2.MItDependencyGraph
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None
```
やっと**MFnSkinCluster**が出てきました。
先程取得したskinClusterノードをイニシャライザに渡し、MFnSkinClusterをインスタンス化します。

[【Maya Python API 2.0 Reference】OpenMayaAnim.MFnSkinCluster](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_anim_1_1_m_fn_skin_cluster_html)

# 2. スキンウェイトの取得
インスタンスが用意できたので早速スキンウェイトを取得してみます。
取得するにはMFnSkinClusterクラスの**getWeights**メソッドを使用します。

getWeightsメソッドのシグネチャは以下です(オーバーロードが3つ)
```diff_python
def getWeights(shape, components):
    # type: (om2.MDagPath, om2.MObject) -> (om2.MDoubleArray, int) # Python2系は(om2.MDoubleArray, long)
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

要はバインドされたメッシュのことです。

```python
active_sel_list = om2.MGlobal.getActiveSelectionList()  # type: om2.MSelectionList
dag_path = active_sel_list.getDagPath(0)                # type: om2.MDagPath
mesh_dag_path = dag_path.extendToShape()                # type: om2.MDagPath
```
上記の`mesh_dag_path`をそのまま使えばOKです。

### 2.1.2 components
>リファレンスより
>* components (MObject) - components to return weights for

少々説明が少ないですが、コンポーネント(頂点やエッジ)のことです。

ここでコンポーネントの取得方法を2つ紹介します。
- 選択しているコンポーネントを取得する
- すべてのコンポーネントを指定する

##### 選択しているコンポーネントを取得する
```python
active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
```
MSelectionListクラスの**getComponent**メソッドで選択中のコンポーネントを取得できます。
ちなみに返り値はタプルになっていて、0番目がMDagPath、1番目がMObjectになっています。
必要なのは1番目のMObjectのみですが、0番目のMDagPathも大抵どこかの処理で使うので、
同時に取得してしまうことが多いです。

##### すべてのコンポーネントを指定する
```python
single_idx_comp_fn = om2.MFnSingleIndexedComponent()                # type: om2.MFnSingleIndexedComponent
vert_comp = single_idx_comp_fn.create(om2.MFn.kMeshVertComponent)   # type: om2.MObject
```
MFnSingleIndexedComponentクラスの**create**メソッドで生成します。

ちなみに上記の例だと頂点(kMeshVertComponent)を指定していますが、
エッジ(kMeshEdgeComponent)やフェース(kMeshPolygonComponent)でも問題ありません。



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
index = skincluster_fn.indexForInfluenceObject(influence_dag_path) # type: int # Python2系はlong
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
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node())   # type: om2.MObject
    skincluster_fn = oma2.MFnSkinCluster(skincluster_node)          # type: om2.MFnSkinCluster
    return skincluster_fn, mesh_dag_path, m_object_component

def get_skincluster_node(mesh_node):
    # type: (om2.MObject) -> om2.MObject
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream) # type: om2.MItDependencyGraph
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None

def create_vertex_component():
    # type: () -> om2.MObject
    single_idx_comp_fn = om2.MFnSingleIndexedComponent()            # type: om2.MFnSingleIndexedComponent
    return single_idx_comp_fn.create(om2.MFn.kMeshVertComponent)    # type: om2.MObject

def get_influence_index_by_name(skincluster_fn, influence_name):
    # type: (oma2.MFnSkinCluster, str) -> int
    influences = skincluster_fn.influenceObjects()                      # type: om2.MDagPathArray
    for influence in influences:
        if influence.__str__() == influence_name:
            return skincluster_fn.indexForInfluenceObject(influence)    # type: int # Python2系はlong
    return None
```

### 2.2.1 getWeights(shape, components)
```スキンウェイトを取得する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn() # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    skin_Weights = skincluster_fn.getWeights(mesh_dag_path, m_object_component) # type: (om2.MDoubleArray, int) # Python2系は(om2.MDoubleArray, long)
    print(skin_Weights)
```

すべてのコンポーネントを取得する場合は下記になります。
(ちなみにgetWeightsに限ってはオブジェクト選択状態ですべてのコンポーネントを取得できるので、上記の記法でもすべてのコンポーネントを取得できます)
```スキンウェイトを取得する(すべてのコンポーネント).py
def main():
    skincluster_fn, mesh_dag_path, _ = create_skincluster_fn()                          # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    skin_Weights = skincluster_fn.getWeights(mesh_dag_path, create_vertex_component())  # type: (om2.MDoubleArray, int) # Python2系は(om2.MDoubleArray, long)
    print(skin_Weights)
```

### 2.2.2 getWeights(shape, components, influence)
```Headのスキンウェイトを取得する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn()                         # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')                               # type: int # Python2系はlong
    head_skin_Weights = skincluster_fn.getWeights(mesh_dag_path, m_object_component, head_infl_index)   # type: (om2.MDoubleArray, int) # Python2系は(om2.MDoubleArray, long)
    print(head_skin_Weights)
```

### 2.2.3 getWeights(shape, components, influences)
```NeckとHeadのスキンウェイトを取得する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn()                         # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    neck_infl_index = get_influence_index_by_name(skincluster_fn, 'Neck')                               # type: int # Python2系はlong
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')                               # type: int # Python2系はlong
    infl_indices = om2.MIntArray([neck_infl_index, head_infl_index])                                    # type: om2.MIntArray
    neck_head_skin_Weights = skincluster_fn.getWeights(mesh_dag_path, m_object_component, infl_indices) # type: (om2.MDoubleArray, int) # Python2系は(om2.MDoubleArray, long)
    print(neck_head_skin_Weights)
```

## 2.3 取得したスキンウェイトデータの読み方
getWeightsメソッドで取得したスキンウェイトデータはMDoubleArray型です。

サンプルとして、骨を3つバインドしたメッシュで解説します。

スキンウェイトは、
- 頂点番号0と3 => Spineのウェイトが1.0
- 頂点番号1 => Neckのウェイトが1.0
- 頂点番号2 => Headのウェイトが1.0

とします。

なお骨のインデックスは、
- Spine => 0
- Neck => 1
- Head  => 2

です。


![MFnSkinCluster01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/ebba1ab1-b281-e770-e75f-eb14ff0fcdd4.png)

![MFnSkinCluster02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/34e6cb54-e44a-4a6b-7e52-073604a3cd14.png)

### 2.3.1 getWeights(shape, components)
返り値の型は(MDoubleArray, int)のタプルです。
(Python2系は(MDoubleArray, long))

#### MDoubleArray

```python
[
    1.0, 0.0, 0.0, # 頂点番号0
    0.0, 1.0, 0.0, # 頂点番号1
    0.0, 0.0, 1.0, # 頂点番号2
    1.0, 0.0, 0.0  # 頂点番号3
]

[
    vtx0のidx0, vtx0のidx1, vtx0のidx2,
    vtx1のidx0, vtx1のidx1, vtx1のidx2,
    vtx2のidx0, vtx2のidx1, vtx2のidx2,
    vtx3のidx0, vtx3のidx1, vtx3のidx2
]
```

頂点1つのウェイトが骨のインデックス順に列挙されています。
すべての骨のインデックスを羅列し終えたら、次の頂点の列挙が始まります。

Componentを指定している場合は、指定したComponentのみ列挙されます

```頂点番号0と2のみを指定した場合.py
[
    1.0, 0.0, 0.0, # 頂点番号0
    0.0, 0.0, 1.0  # 頂点番号2
]

[
    vtx0のidx0, vtx0のidx1, vtx0のidx2,
    vtx2のidx0, vtx2のidx1, vtx2のidx2
]
```


#### int(Python2系はlong)
```
3
```
インフルエンス数が返ってきます。
今回の場合は「Spine、Neck、Head」の3つなので3です。

### 2.3.2 getWeights(shape, components, influence)
返り値の型はMDoubleArrayです。
引数でinfluenceを指定しているので、int(インフルエンス数)は返ってきません。

#### MDoubleArray
基本的に他のオーバーロードと同じですが、
influenceを指定しているので、指定した骨のウェイトのみが列挙されます。

```Head骨のウェイトのみ.py
[
    0.0, # 頂点番号0
    0.0, # 頂点番号1
    1.0, # 頂点番号2
    0.0  # 頂点番号3
]

[
    vtx0のidx2,
    vtx1のidx2,
    vtx2のidx2,
    vtx3のidx2
]
```

### 2.3.3 getWeights(shape, components, influences)
返り値の型はMDoubleArrayです。

#### MDoubleArray
2.3.2とほぼ同じです。
influenceが複数になります。

```Neck骨とHead骨のウェイトのみ.py
[
    0.0, 0.0, # 頂点番号0
    1.0, 0.0, # 頂点番号1
    0.0, 1.0, # 頂点番号2
    0.0, 0.0  # 頂点番号3
]

[
    vtx0のidx1, vtx0のidx2,
    vtx1のidx1, vtx1のidx2,
    vtx2のidx1, vtx2のidx2,
    vtx3のidx1, vtx3のidx2
]
```

# 3. スキンウェイトの設定
取得が分かったので、次は設定をしてみます。
設定するにはMFnSkinClusterクラスの**setWeights**メソッドを使用します。

setWeightsメソッドのシグネチャは以下です(オーバーロードが2つ)

```python
def setWeights(shape, components, influence, weight, normalize=True, returnOldWeights=False):
    # type: (om2.MDagPath, om2.MObject, int, float, bool, bool) -> None or om2.MDoubleArray
    ...
def setWeights(shape, components, influences, weights, normalize=True, returnOldWeights=False):
    # type: (om2.MDagPath, om2.MObject, MIntArray, MDoubleArray, bool, bool) -> None or om2.MDoubleArray
    ...
```

## 3.1 引数解説
1つずつ引数を確認していきます。

### 3.1.1 shape
>* shape (MDagPath) - object being deformed by the skinCluster

バインドされたメッシュです。getWeightsに同じ。

### 3.1.2 components
> * components (MObject) - the components to set weights on

コンポーネントです。getWeightsに同じ。

### 3.1.3 influence
> * influence (int) - physical index of a single influence object

単体の骨のインデックスです。getWeightsに同じ。
上のオーバーロードで使用します。

### 3.1.4 influences
> * influences (MIntArray) - physical indices of several influence objects.

複数の骨のインデックスです。getWeightsに同じ。
下のオーバーロードで使用します。

### 3.1.5 weight
> * weight (float) - single weight to be applied to all components.

設定するスキンウェイトの値です。
上のオーバーロードで使用するので、influenceと同時に使用します。
つまり単一のinfluenceに設定したいのウェイト値になります。

### 3.1.6 weights
> * weights (MDoubleArray) - weights to be used with several influence objects.

設定するスキンウェイトの値です。
下のオーバーロードで使用するので、influencesと同時に使用します。
つまり複数のinfluencesに設定したいウェイト値になります。

### 3.1.7 normalize
> * normalize (bool) - if True, normalize weights on other influence objects

Trueの場合、スキンウェイトのノーマライズ(合計値が1になるようにする処理)がかかります。
デフォルトでTrueです。

しかしskinClusterノード内のnomalizeWeightsの値により挙動が左右されるようなので、
正確なところは不明です(有識者の方コメントお待ちしてます)

### 3.1.8 returnOldWeights
> * returnOldWeights(bool) - if True, return the old weights, otherwise return None

Trueの場合、setWeightsする前のスキンウェイト値が返り値として返ってきます。
デフォルトでFalseです。

## 3.2 使ってみる
実際に使ってみます。

なお共通処理として、下記関数はすでに定義済みとします(getWeightsのときのものとまったく同じです)
```python
def create_skincluster_fn():
    # type: () -> (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    active_sel_list = om2.MGlobal.getActiveSelectionList()          # type: om2.MSelectionList
    dag_path, m_object_component = active_sel_list.getComponent(0)  # type: (om2.MDagPath, om2.MObject)
    mesh_dag_path = dag_path.extendToShape()                        # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node())   # type: om2.MObject
    skincluster_fn = oma2.MFnSkinCluster(skincluster_node)          # type: om2.MFnSkinCluster
    return skincluster_fn, mesh_dag_path, m_object_component

def get_skincluster_node(mesh_node):
    # type: (om2.MObject) -> om2.MObject
    dg_iterator = om2.MItDependencyGraph(mesh_node, om2.MFn.kSkinClusterFilter, om2.MItDependencyGraph.kUpstream) # type: om2.MItDependencyGraph
    while not dg_iterator.isDone():
        m_object = dg_iterator.currentNode()
        if m_object.hasFn(om2.MFn.kSkinClusterFilter):
            return m_object
        dg_iterator.next()
    return None

def create_vertex_component():
    # type: () -> om2.MObject
    single_idx_comp_fn = om2.MFnSingleIndexedComponent()            # type: om2.MFnSingleIndexedComponent
    return single_idx_comp_fn.create(om2.MFn.kMeshVertComponent)    # type: om2.MObject

def get_influence_index_by_name(skincluster_fn, influence_name):
    # type: (oma2.MFnSkinCluster, str) -> int
    influences = skincluster_fn.influenceObjects()                      # type: om2.MDagPathArray
    for influence in influences:
        if influence.__str__() == influence_name:
            return skincluster_fn.indexForInfluenceObject(influence)    # type: int # Python2系はlong
    return None
```

### 3.2.1 setWeights(shape, components, influence, weight, normalize=True, returnOldWeights=False)
getWeightsと異なり、コンポーネントを選択していないと設定されないので注意です。

```Headのスキンウェイトを1.0に設定する.py
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn() # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')       # type: int # Python2系はlong
    skincluster_fn.setWeights(mesh_dag_path, m_object_component, head_infl_index, 1.0)
```

すべてのコンポーネントに設定する場合は下記になります。
```Headのスキンウェイトを1に設定する(すべてのコンポーネント).py
def main():
    skincluster_fn, mesh_dag_path, _ = create_skincluster_fn()              # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')   # type: int # Python2系はlong
    skincluster_fn.setWeights(mesh_dag_path, create_vertex_component(), head_infl_index, 1.0)
```

setWeightsは、オブジェクト選択状態ではすべての頂点を選択した判定にはなりません。しっかりとコンポーネントを選択している必要があります。
なので、すべてのコンポーネントのスキンウェイトを設定したい場合は、
MFnSingleIndexedComponentで生成したMObjectを使うことをおすすめします。

### 3.2.2 setWeights(shape, components, influences, weights, normalize=True, returnOldWeights=False)
下記は**頂点番号順に、Head、Head、Neck、Neckにそれぞれ1.0を設定する**例です。
すべてのコンポーネントを選択して実行してください。
```python
def main():
    skincluster_fn, mesh_dag_path, m_object_component = create_skincluster_fn() # type: (om2.MFnSkinCluster, om2.MDagPath, om2.MObject)
    neck_infl_index = get_influence_index_by_name(skincluster_fn, 'Neck')       # type: int # Python2系はlong
    head_infl_index = get_influence_index_by_name(skincluster_fn, 'Head')       # type: int # Python2系はlong
    infl_indices = om2.MIntArray([neck_infl_index, head_infl_index])            # type: om2.MIntArray
    weights_to_set = om2.MDoubleArray(
        [
            0.0, 1.0, # 頂点番号0
            0.0, 1.0, # 頂点番号1
            1.0, 0.0, # 頂点番号2
            1.0, 0.0  # 頂点番号3
        ]
    ) # type: om2.MDoubleArray
    skincluster_fn.setWeights(mesh_dag_path, m_object_component, infl_indices, weights_to_set)
```

このときに渡すweightsの要素数は**component * influences**となります。
この要素数やcomponentやinfluencesに過不足がある場合、切り詰め処理になります。

例えばcomponentが不足の場合、
実例をあげると、上記例で**頂点番号3を選択して実行した場合**、
weightsの要素数に対して、componentが不足しているため、
頂点番号3にNeckが0.0、Headが1.0に設定される、という挙動になります。

# 4. おわりに
MFnSkinClusterのすべてのメソッドを紹介したわけではありませんが、
基本的なスキンウェイトの取得・設定に関しては、紹介した項目だけで概ねカバーできると思います。

スキンウェイトデータが一次元の配列だったりと、少し癖がありますが、
maya.cmdsとは比べ物にならないくらい爆速で動作するので、ぜひ使ってみてください。
