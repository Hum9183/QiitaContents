---
title: 【Maya】OpenMaya.MFnSkinClusterの使い方【Python API 2.0】
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
- この記事は**PythonAPI2.0**のMFnSkinClusterクラスの使い方をまとめたものです。
- PythonAPI1.0やAPI(C++)の使い方は載っていません。

# 環境
Autodesk Maya 2024.1
(API2.0が動く環境であればどの環境でも問題ありません)

# MFnSkinClusterとは
skinClusterからウェイト情報を取得したり設定することが出来るクラスです。

# なぜOpenMayaなのか
maya.cmdsの中にもウェイト情報を操作出来るコマンド、**skinCluster**が存在します。
しかし、このコマンドは処理速度が遅いため、実用的なツールを作る際は処理速度が速いOpenMayaを使用します。

# モジュールのインポート
```import_oma2.py
import maya.api.OpenMaya as om2
import maya.api.OpenMayaAnim as oma2
```
使用するためにはmaya.api.OpenMayaAnimというモジュールをインポートします。
エイリアスをoma2としておくのがおすすめです。
maya.api.OpenMayaも一緒に使うことが多いのでまとめてインポートしておきます。

# インスタンス化する
MFnSkinClusterの生成には引数にMObject型のインスタンスが必要になります。
MObjectを生成するにはにMDagPathの型インスタンスが必要になります。

大まかな流れは下記です。
- MDagPath型のインスタンス
↓
- MObject型のインスタンス
↓
- MFnSkinCluster型のインスタンス

以下は「選択しているバインド済メッシュからMFnSkinClusterインスタンスを生成する」例です
```import_oma2.py
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

def get_skincluster_node(mesh_depend_node):
    dg_iterator = om2.MItDependencyGraph(
                    mesh_depend_node,
                    om2.MFn.kSkinClusterFilter,    # NOTE: kInvalidを渡すと無指定になるので全て列挙することもできる
                    om2.MItDependencyGraph.kUpstream)
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

## 選択項目を取得
```python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
```
現在の選択項目を取得します。
maya.cmdsで言うところのcmds.lsにあたります。
返り値の型はlist[str]ではなくMSelectionList型です。

[【Maya Python API 2.0 Reference】OpenMaya.MGlobal](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_global_html)
[【Maya Python API 2.0 Reference】OpenMaya.MSelectionList](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_selection_list_html)

## MDagPathを取得
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
+   dag_path = active_sel_list.getDagPath(0) # type: om2.MDagPath
```
MSelectionListのメソッド**getDagPath**を使い、MDagPathを取得します。
引数は取り出したい選択項目のIndexです（listの添字だと思ってください）
今回は1つ目の選択項目から取り出したいため、0を渡します。

## メッシュのMDagPathを取得
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
+   mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
```
ビューポート等からメッシュを選択した場合、メッシュではなくtransformノードを選択しています。
transformからskinClusterをたどることは出来ないため、一度シェイプ(メッシュ)に変換します。

[【Maya Python API 2.0 Reference】OpenMaya.MDagPath](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_dag_path_html)

## skinClusterノードを取得(その1)
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
    mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
+   skincluster_node = get_skincluster_node(mesh_dag_path.node()) # type: om2.MObject

+ def get_skincluster_node(mesh_node):
+   ...
```
ノードからノードを探す処理は、OpenMayaだとやや長い処理が必要になるため、関数を用意します。
ちなみにmaya.cmdsであれば、cmds.listHistoryで簡単に取得できます。

## skinClusterノードを取得(その2)
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
ここではskinClusterノードを探したいため、第2引数に**MFn.kSkinClusterFilter**を渡しています。

[【Maya Python API 2.0 Reference】OpenMaya.MItDependencyGraph](https://help.autodesk.com/view/MAYAUL/2022/ENU/?guid=Maya_SDK_py_ref_class_open_maya_1_1_m_it_dependency_graph_html)

## MFnSkinClusterを生成
```diff_python
def main():
    active_sel_list = om2.MGlobal.getActiveSelectionList() # type: om2.MSelectionList
    dag_path = active_sel_list.getComponent(0) # type: om2.MDagPath
    mesh_dag_path = dag_path.extendToShape() # type: om2.MDagPath
    skincluster_node = get_skincluster_node(mesh_dag_path.node()) # type: om2.MObject
+   skincluster_fn = oma2.MFnSkinCluster(skincluster_node) # type: om2.MFnSkinCluster

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
先程取得したskinClusterノードを引数に渡し、MFnSkinClusterをインスタンス化します。
