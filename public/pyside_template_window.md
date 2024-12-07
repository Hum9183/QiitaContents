---
title: 【Maya】ウィンドウの雛形をつくってみた【PySide】
tags:
  - Python
  - PySide
  - maya
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
TODO: 時系列的にDevメニューが有るとおかしいGifがあるので修正する

# はじめに
- この記事はMayaツールのウィンドウの雛形をつくってみたものです
- PySideを使用します
- **Mayaの標準的なウィンドウ(NodeEditorやScriptEditorなど)と同じように動作することを目指します**

# 注意
- 本記事内のソースコードは尺の都合上、一部(import文や焦点があたっていないメソッドなど)を省略することがあります
- Python2系以前(Maya2020以前)はサポートしません

# 対象読者
- PySideのことはよくわからないけど、とりあえずウィンドウをつくってみたい人
- PySideのウィンドウについて理屈から理解したい人
- reloadできるウィンドウをつくりたい人
- Dockableなウィンドウをつくりたい人
- Restoreできるウィンドウをつくりたい人

# 環境
- Autodesk Maya 2022.5.1 (Python 3.7.7)
- Autodesk Maya 2025.1 (Python 3.11.4)

# ディレクトリ構成
最終的な構成は以下になります

pyside_template_window/
├__init__.py
├restart.py
├restore.py
├run.py
└template_window.py
start.py(ScriptEditorから起動する用)

# 本記事のソースコード
本記事では理屈での理解を促すためにわざと遠回りしたり、
詰まったところも含めて解説します(要は長いです)
先に完成品を見たい方はこちらをご覧ください。
https://github.com/Hum9183/pyside_template_window

# 0. PySideとは
`Qt(キュート)`というC++で書かれたフレームワークのPythonバインディング(Pythonで使えるようにしたもの)です。

なおQtはGUI以外にもいろいろな機能があるのですが、
ことMayaのツール作成で使う場合においてはシンプルに`GUIライブラリ`と捉えても問題ないと思います。

またQtのPythonバインディングには`PyQt`というものもあるのですが、
こちらは別物ですので注意が必要です(別物というほど別物ではないのですがライセンスが異なるので同一視してはいけません)

ちなみにMayaのUIはQtで動いています。

# 1. PySideでウィンドウを出す
まずはPySideで普通のウィンドウを出してみます
```template_window.py
try:
    from PySide6.QtWidgets import QMainWindow
except ImportError:
    from PySide2.QtWidgets import QMainWindow

class TemplateWindow(QMainWindow):
    pass
```
普通のウィンドウを作る場合は`QMainWindow`を継承したクラスを作ります。
現在サポートされているMayaのPySideはPySide2とPySide6があります。
今回は両対応するためにtry-except文で実装しています。
```run.py
from .template_window import TemplateWindow

def start() -> None:
    window = TemplateWindow()
    window.show()
```
後でreloadにも対応することを見越して、
インスタンス生成処理はrun.pyというファイルに分けておきます。
```start.py
def start_pyside_template_window():
    from pyside_template_window import run
    run.start()

if __name__ == '__main__':
    start_pyside_template_window()
```
start.pyはScriptEditorで実行するソースコードです。

それでは実行してみます

![01.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/81eb32fb-081f-ea26-a499-a0f73f2d7436.gif)

一瞬ウィンドウが表示されるものの、
すぐに消えてしまいました。

対策としてrun.pyに`sys.exit()`を追加します。
```diff_python: run.py
+import sys
from .template_window import TemplateWindow

def start() -> None:
    window = TemplateWindow()
    window.show()
+   sys.exit()
```
実行してみます。

![02.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/4c548383-5d48-df8f-ffb9-f82663091411.gif)

今度は消えなくなりました。

## 1.1 sys.exit()が必要な理由
いろいろと調べたのですが正直分かりませんでした。
有識者の方、コメントお待ちしております。

ちなみにスコープが絡んでいるところまではなんとなく掴んでいます。
```start.py
from pyside_template_window.template_window import TemplateWindow

window = TemplateWindow()
window.show()
```
たとえば上記のようにScriptEditorの一番上のスコープで直接インスタンス化してshow()を実行すればすぐに消えません。
(しかしGlobal空間を汚すことになるのであまり褒められた書き方ではありません)

しかしながら後述するMayaのMainWindowと親子付けすることによっても消えなくなるので、
あまり気にしなくてもよいです。

# 2. ボタンを追加する
ただのウィンドウでは味気ないのでボタンを追加します。
QtではボタンなどのGUIの部品を`Widget`と呼びます。
このWidgetを追加することによって好みのGUIを作成していくことになります。

ボタンは`QPushButton`というクラスを使います。
QPushButtonのインスタンスをsetCentralWidget()することでボタンをセットすることができます。
```diff_python: template_window.py
try:
+   from PySide6.QtWidgets import QMainWindow, QPushButton
except ImportError:
+   from PySide2.QtWidgets import QMainWindow, QPushButton

class TemplateWindow(QMainWindow):
-   pass
+   def init_gui(self) -> None:
+       push_button = QPushButton('PUSH ME', self)
+       self.setCentralWidget(push_button)
```

run.pyではinit_gui()を呼んでおきます。

```diff_python: run.py
import sys
from .template_window import TemplateWindow

def start() -> None:
    window = TemplateWindow()
+   window.init_gui()
    window.show()
    sys.exit()
```
実行すると、

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/1f0ee91f-b8fd-239c-53c1-f8e9cd4cb039.png)

うまくいっているようです。

## 2.1 ボタンにスロットを接続する
現在のボタンは押しても何も起きないのでスロットを接続します。
スロットというのはオブジェクトに変更(シグナル)があった場合に実行される関数のことです。
コールバック関数と言ったほうが馴染み深い人もいるかもしれません。

https://www.qt.io/ja-jp/blog/2010/06/17/signals-and-slots

今回は`Hello, World!がprintされるスロット`を接続します。
```diff_python: template_window.py
try:
    from PySide6.QtWidgets import QMainWindow, QPushButton
except ImportError:
    from PySide2.QtWidgets import QMainWindow, QPushButton

class TemplateWindow(QMainWindow):
    def init_gui(self) -> None:
        push_button = QPushButton('PUSH ME', self)
+       push_button.clicked.connect(lambda *arg: self.__print_hello_world())
        self.setCentralWidget(push_button)

+   def __print_hello_world(self) -> None:
+       print('Hello, World!')
```
`push_button.clicked`というのがシグナルです。`ボタンをクリックしたとき`ということですね。
シグナルにはほかにも様々な種類があるため、いろいろなことを引き金にして関数を呼び出すことができます。

`lambda *arg: self.__print_hello_world()`というのがスロットです。
lambdaを使っている理由は引数がある関数を使えるようにするためです。
今回は引数がない関数ですが、lambdaを付けたり消したりするとバグの元なので、私はどんなときでもlambdaを使っています。

またlambda以外の方法として`functools.partial()`を使う方法もあります。

https://docs.python.org/ja/3.5/library/asyncio-eventloop.html#calls

さてシグナルとスロットの説明はほどほどに、実際に動かしてみます。

![03.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/5274923a-5050-99b3-b728-79259230fab4.gif)

大丈夫そうですね。

しかしちょっと待ってください。
Mayaの別の場所をクリックすると、ウィンドウが消えてしまいました。

![04.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/137a6cd4-9b3d-f7dd-22ee-c4ce38d9af51.gif)

厳密に言えば消えたわけではなく、Mayaのウィンドウの後ろに行ってしまっただけです。
しかしこのままではとても使いづらいツールになってしまいます。

## 2.2 ウィンドウが後ろに行ってしまう原因
冒頭で軽く触れましたが、**MayaのUIはQtで動いています**。

Qtのウィンドウが2つある状況を想像してみてほしいのですが、
片方がアクティブになった場合、もう片方のウィンドウが後ろに隠れてしまうという挙動は自然に思えます。

これを回避する場合、「片方のウィンドウをもう片方のウィンドウに親子付けする」という方法が考えられます。
ということでMayaのMainWindowに親子付けします。

# 3. MayaのMainWindowに親子付けする
実はMayaのMainWindowに親子付けするための便利なクラスはAutodeskさんの方で事前に用意してくれているのですが、
まずは理屈を理解するために自前で実装してみましょう。

まず`MayaのMainWindow`の取り方ですが、
```get_maya_main_window.py
from maya import OpenMayaUI as omui
maya_main_window_ptr = omui.MQtUtil.mainWindow()
```
で取ってくることができます。
ちなみにこの`maya_main_window_ptr`の型は`SwigPyObject`です。
built-inの型になっており、VSCodeやPyCharmのシンタックスハイライトは効きません。
TODO: ↑本当か調べる

何者なのか調べるために`SwigPyObject`の__dict__をprintしてみます。
```SwigPyObject.__dict__.py
{
    '__repr__': <slot wrapper '__repr__' of 'SwigPyObject' objects>,
    '__getattribute__': <slot wrapper '__getattribute__' of 'SwigPyObject' objects>,
    '__lt__': <slot wrapper '__lt__' of 'SwigPyObject' objects>,
    '__le__': <slot wrapper '__le__' of 'SwigPyObject' objects>,
    '__eq__': <slot wrapper '__eq__' of 'SwigPyObject' objects>,
    '__ne__': <slot wrapper '__ne__' of 'SwigPyObject' objects>,
    '__gt__': <slot wrapper '__gt__' of 'SwigPyObject' objects>,
    '__ge__': <slot wrapper '__ge__' of 'SwigPyObject' objects>,
    '__int__': <slot wrapper '__int__' of 'SwigPyObject' objects>,
    'disown': <method 'disown' of 'SwigPyObject' objects>,
    'acquire': <method 'acquire' of 'SwigPyObject' objects>,
    'own': <method 'own' of 'SwigPyObject' objects>,
    'append': <method 'append' of 'SwigPyObject' objects>,
    'next': <method 'next' of 'SwigPyObject' objects>,
    '__doc__': 'Swig object carries a C/C++ instance pointer',
    '__hash__': None
}
```
> '\_\_doc__': 'Swig object carries a C/C++ instance pointer'

とあるのでC/C++のポインタを持つクラスのようですね。
気になるポインタの取得方法ですが、__int__が実装されているので、intにキャストすることで取得できるようです。
```print_maya_main_window_ptr.py
from maya import OpenMayaUI as omui
maya_main_window_ptr = omui.MQtUtil.mainWindow()
print(int(maya_main_window_ptr))
```
```output.txt
2424978466992
```
PySideではポインタをそのまま扱うことができないため、`PySideで扱える型のインスタンス`に変換する必要があります。
変換には`shiboken`というライブラリの`wrapInstance()`を使います。
```diff_python: get_maya_main_window_qwidget.py
from maya import OpenMayaUI as omui
try:
    from PySide6.QtWidgets import QMainWindow
    from shiboken6 import wrapInstance
except ImportError:
    from PySide2.QtWidgets import QMainWindow
    from shiboken2 import wrapInstance

maya_main_window_ptr = omui.MQtUtil.mainWindow()
maya_main_window: QMainWindow = wrapInstance(int(maya_main_window_ptr), QMainWindow)
```
wrapInstance()の第一引数には`インスタンスに変換したいポインタ`を、第二引数には`変換後の型`を渡します。

ということでMayaのMainWindowが取れたので早速親子付けしてみます。
どうやって親子付けするかですが、これはシンプルに`QMainWindow`のイニシャライザで渡します。
```diff_python: template_window.py
+from maya import OpenMayaUI as omui
try:
    from PySide6.QtWidgets import QMainWindow, QPushButton
+   from shiboken6 import wrapInstance
except ImportError:
    from PySide2.QtWidgets import QMainWindow, QPushButton
+   from shiboken2 import wrapInstance

class TemplateWindow(QMainWindow):
+   def __init__(self) -> None:
+       maya_main_window_ptr = omui.MQtUtil.mainWindow()
+       maya_main_window = wrapInstance(int(maya_main_window_ptr), QMainWindow)
+       super().__init__(parent=maya_main_window)

    def init_gui(self) -> None:
        ...

    def __print_hello_world(self) -> None:
        ...
```
実行します。

![06.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e932353f-7494-c592-a81f-9bae5e259978.gif)

Mayaの別の場所をクリックしてもウィンドウが後ろに行かなくなりました。

## 3.1 sys.exit()を削除する
実はMayaのMainWindowに親子付けした時点でrun.pyのsys.exit()は必要なくなります。
```diff_python: run.py
-import sys
from .template_window import TemplateWindow

def start() -> None:
    window = TemplateWindow()
    window.init_gui()
    window.show()
-   sys.exit()
```

## 3.2 MayaQWidgetBaseMixinを継承する
さて自前で実装して理解が深まったところで、
あらかじめ言っていた通りAutodeskさんが用意してくれているクラスがあるので、そちらに置き換えましょう。

先ほどは__init__()内で自前で親子付けしましたが、
その代わりに`MayaQWidgetBaseMixin`を継承することでshow()を呼んだ際に親子付けをしてくれます。

```diff_python: template_window.py
-from maya import OpenMayaUI as omui
+from maya.app.general.mayaMixin import MayaQWidgetBaseMixin

...(import PySide etc.)

-class TemplateWindow(QMainWindow):
+class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
    def __init__(self) -> None:
-       maya_main_window_ptr = omui.MQtUtil.mainWindow()
-       maya_main_window = wrapInstance(int(maya_main_window_ptr), QMainWindow)
-       super().__init__(parent=maya_main_window)
+       super().__init__()
```
1つ注意しなければならない点として`MayaQWidgetBaseMixin`の**継承の記載順**があります。
今回は`MayaQWidgetBaseMixin` -> `QMainWindow`の順番で記載しましたが、
これを逆にすると親子付けがされません。
```diff_python: template_window.py
-class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
+class TemplateWindow(QMainWindow, MayaQWidgetBaseMixin):
```

![07.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/f07dfc3f-6746-0119-b0d4-5aa22609cc6d.gif)

この問題を菱形継承問題といいます。

https://ja.wikipedia.org/wiki/%E8%8F%B1%E5%BD%A2%E7%B6%99%E6%89%BF%E5%95%8F%E9%A1%8C

現在`TemplateWindow`は`QMainWindow`と`MayaQWidgetBaseMixin`の２つを継承しています。
これは**多重継承**というものです。

この状況で`super().__init__()`を呼んだ場合、`QMainWindow`と`MayaQWidgetBaseMixin`
のどちらの`__init__()`が呼ばれるのでしょうか？

この結果はプログラミング言語によって異なるのですが、
Pythonの場合はC3というアルゴリズムを使用しています(Python2.3以降)

https://www.python.org/download/releases/2.3/mro/

またこの順番のことをMRO(Method Resolution Order)といいます。
MROは実際のPythonで確認することができます。

```method_resolution_order_sample.py
from maya.app.general.mayaMixin import MayaQWidgetBaseMixin
from PySide2.QtWidgets import QMainWindow, QPushButton

class TemplateWindowA(MayaQWidgetBaseMixin, QMainWindow):
    ...

class TemplateWindowB(QMainWindow, MayaQWidgetBaseMixin):
    ...

print(TemplateWindowA.__mro__)
print(TemplateWindowB.__mro__)
```
```output.txt
(
    <class '__main__.TemplateWindowA'>,
    <class 'maya.app.general.mayaMixin.MayaQWidgetBaseMixin'>,
    <class 'PySide2.QtWidgets.QMainWindow'>,
    <class 'PySide2.QtWidgets.QWidget'>,
    <class 'PySide2.QtCore.QObject'>,
    <class 'PySide2.QtGui.QPaintDevice'>,
    <class 'Shiboken.Object'>,
    <class 'object'>
)
(
    <class '__main__.TemplateWindowB'>,
    <class 'PySide2.QtWidgets.QMainWindow'>,
    <class 'PySide2.QtWidgets.QWidget'>,
    <class 'PySide2.QtCore.QObject'>,
    <class 'PySide2.QtGui.QPaintDevice'>,
    <class 'Shiboken.Object'>,
    <class 'maya.app.general.mayaMixin.MayaQWidgetBaseMixin'>,
    <class 'object'>
)
```
TemplateWindowAは2番目に`MayaQWidgetBaseMixin`が来ていますが、
TemplateWindowBは7番目に`MayaQWidgetBaseMixin`が来ています。

これはつまりTemplateWindowBでは__init__()やshow()が呼ばれたときに、
`MayaQWidgetBaseMixin`のメソッドではなく`QMainWindow`のメソッドが呼ばれることを意味しています。

というわけで継承の記載順には気をつける必要があります。
また継承の記載順はMROを決定する要素の一部に過ぎないということも忘れてはいけません。

## 3.3 イニシャライザの仮引数を整える
現在のTemplateWindowのイニシャライザは仮引数がありませんが、
`MayaQWidgetBaseMixin`のイニシャライザには仮引数があります。
```mayaMixin.py
class MayaQWidgetBaseMixin(object):
    def __init__(self, parent=None, *args, **kwargs):
        super(MayaQWidgetBaseMixin, self).__init__(parent=parent, *args, **kwargs)
        self._initForMaya(parent=parent)
```
TemplateWindowで仮引数を埋め立ててしまうのはあまりお行儀が良い書き方ではありません。
TemplateWindowのイニシャライザでも仮引数も渡せるようにしておきましょう。
```diff_python: template_window.py
class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
-   def __init__(self) -> None:
-       super().__init__()
+   def __init__(self, parent=None, *args, **kwargs) -> None:
+       super().__init__(parent=parent, *args, **kwargs)
```

# 4. 2つ以上起動できないようにする
実は今の状態だと起動するたびに新規のウィンドウが生成されます。

![09.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/7e1f3a67-4ba3-cf18-6f56-b8a7a2941cda.gif)

スタンダードなツールは多重で起動はできないものだと思います。

トラブルになりかねないので直していきます。

## 4.1 ウィンドウに名前を設定する
多重起動を防ぐ場合、方法はいろいろあると思いますが、
シンプルなのは**すでにウィンドウが存在するか**をチェックする方法です。

Mayaでは`omui.MQtUtil.findControl()`を使うことで一意のウィンドウを取得することができます。
しかしこの関数の引数には`ウィンドウの名前`を渡す必要があります。
`TemplateWindow`にはなんだかんだで名前を設定していませんでした。
名前はウィンドウを識別する上でかなり重要な要素になるので、しっかりと設定していきます。

https://help.autodesk.com/view/MAYADEV/2025/JPN/?guid=Maya_DEVHELP_Maya_Python_API_Working_with_PySide_in_Maya_html

```diff_python: template_window.py
class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
+   name = 'PysideTemplate'

    def __init__(self, parent=None, *args, **kwargs) -> None:
        super().__init__(parent=parent, *args, **kwargs)

+   def init(self) -> None:
+       self.setObjectName(TemplateWindow.name)

    def init_gui(self) -> None:
        ...

    def __print_hello_world(self) -> None:
        ...
```
```diff_python: run.py
def start() -> None:
    window = TemplateWindow()
+   window.init()
    window.init_gui()
    window.show()
```
`init()`というメソッドを用意し、`setObjectName()`を実行するようにしました。
これでウィンドウの名前を設定できました。

## 4.2 すでにウィンドウが存在するかを判定する
`start()`で`omui.MQtUtil.findControl()`を使ってウィンドウを取得します。
返り値の型は`QMainWindow`などではなく`SwigPyObject`なので、
無難に`if ptr is None:`で存在するかどうかを判定しましょう。
```diff_python: run.py
+from maya import OpenMayaUI as omui
from .template_window import TemplateWindow

def start() -> None:
+   # 現在のMaya内に存在するTemplateWindowのポインタを取得する
+   ptr = omui.MQtUtil.findControl(TemplateWindow.name)
+   if ptr is None:  # ない場合
+       print(f'{TemplateWindow.name}が存在しないため生成します')
        window = TemplateWindow()
        window.init()
        window.init_gui()
        window.show()
+   else: # ある場合
+       print(f'すでに{TemplateWindow.name}が存在しています')
```
実行してみます。

![10.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/7a6f5945-b015-ac38-2d62-418556f566b7.gif)

うまくいっているようですね。

## 4.3 あるけどない場合
一度ウィンドウを閉じたあとでもう一度起動すると、
すでにウィンドウが存在すると言われてしまいます。

![11.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e23b81e5-a6e1-276a-3926-0b0d080d52ea.gif)

これはどちらかというとQtの仕様の話になるのですが、
Qtは**ウィンドウを閉じた際にインスタンスは破棄されません。非表示になっているだけです。**
なのでそれも加味した処理を作る必要があります。

```diff_python: run.py
def start() -> None:
    # 現在のMaya内に存在するTemplateWindowのポインタを取得する
    ptr = omui.MQtUtil.findControl(TemplateWindow.name)
    if ptr is None:  # ない場合
        print(f'{TemplateWindow.name}が存在しないため生成します')
        window = TemplateWindow()
        window.init()
        window.init_gui()
        window.show()
    else:  # ある場合
        print(f'すでに{TemplateWindow.name}が存在しています')
+       if window.isVisible():
+           print('すでに表示されています')
+       else:
+           print('再表示します')
+           window.setVisible(True)
```
ポインタが存在する場合は`isVisible()`を呼び、
Falseの場合は`setVisible(True)`を呼び再表示します。

![12.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/cdd5ebae-1da7-5139-be39-06a342010731.gif)

TODO: この時点のソースコードだと再フォーカスがうまく効かないかも、
何をしたら効くようになるのか調べる

## 4.4 リファクタ
TemplateWindowインスタンスに関する処理は今後もよく使うので関数化してしまいましょう。
```diff_python: run.py
def start() -> None:
    # 現在のMaya内に存在するTemplateWindowのポインタを取得する
    ptr = omui.MQtUtil.findControl(TemplateWindow.name)
    if ptr is None:  # ない場合
        print(f'{TemplateWindow.name}が存在しないため生成します')
-       window = TemplateWindow()
-       window.init()
-       window.init_gui()
+       window = __create_window()
        window.show()
    else:  # ある場合
        print(f'すでに{TemplateWindow.name}が存在しています')
        if window.isVisible():
            print('すでに表示されています')
        else:
            print('再表示します')
            window.setVisible(True)

+def __create_window() -> TemplateWindow:
+   window = TemplateWindow()
+   window.init()
+   window.init_gui()
+   return window
```

# 4.5 タイトルをつける
いまさら感がありますが、今のウィンドウにはタイトルがついていません。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/270fa527-6d66-1522-09d8-a246254924b8.png)

Maya-20xxという無機質なタイトルが設定されています。

お好みのタイトルを付けてしまいましょう。
```diff_python: template_window.py
class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
    name = 'PysideTemplate'
+   title = 'PySide Template'
    def init(self):
        self.setObjectName(TemplateWindow.name)
+       self.setWindowTitle(TemplateWindow.title)
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/f73fda34-0667-93a5-c0de-49ccf0f39f47.png)

# 5. ドッキングできるようにする
Mayaの標準的なウィンドウはほかのGUIとドッキングをすることができます。

![13.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/2d345d15-5d01-db25-705e-c14a7da8f562.gif)

もちろん現在のTemplateWindowではドッキングできません。

ドッキングできるようにするためには以下の2つのことを行う必要があります。
- MayaQWidgetDockableMixinを継承する
- show()のdockableフラグをTrueにする

## 5.1 MayaQWidgetDockableMixinを継承する
`MayaQWidgetDockableMixin`とは、
`MayaQWidgetBaseMixin`にドッキング機能がついたものです。
クラス名が長くて紛らわしいですが名前としては文字列の`Base`が`Dockable`に変わっただけです。
クラスとしては`MayaQWidgetBaseMixin`を継承してできています。

```diff_python: mayaMixin.py
class MayaQWidgetDockableMixin(MayaQWidgetBaseMixin):
    ...
```

それでは、
`MayaQWidgetBaseMixin`を継承していたところを
`MayaQWidgetDockableMixin`に置き換えます。

```diff_python: template_window.py
-from maya.app.general.mayaMixin import MayaQWidgetBaseMixin
+from maya.app.general.mayaMixin import MayaQWidgetDockableMixin

-class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
+class TemplateWindow(MayaQWidgetDockableMixin, QMainWindow):
    ...
```

## 5.2 show()のdockableフラグをTrueにする
`MayaQWidgetDockableMixin`を継承することでshow()に様々なフラグが渡せるようになります。
`dockableフラグ`はデフォルトがNoneなので明示的にTrueを渡します。

やり方ですがrun.pyのstart()のshow()を書き換えると下記のようになります。
```diff_python: run.py
def start() -> None:
    # 現在のMaya内に存在するTemplateWindowのポインタを取得する
    ptr = omui.MQtUtil.findControl(TemplateWindow.name)
    if ptr is None:  # ない場合
        print(f'{TemplateWindow.name}が存在しないため生成します')
        window = __create_window()
-       window.show()
+       window.show(dockable=True)
```
これでも悪くはないのですが、**フラグをどう指定するかなどの細かい情報はrun.py側が気にすることではないので**、今回はtemplate_window.pyの中で指定します。

template_window.pyの中でshow()を呼ぶことはないので、方法としてはshow()をオーバーライドすることで実現します。

```diff_python: template_window.py
class TemplateWindow(MayaQWidgetDockableMixin, QMainWindow):
+   def show(self): # オーバーライド
+       super().show(dockable=True)
```

![14.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/c2bd9c51-d169-1c53-e469-5110f25a3eea.gif)

やや不格好ではありますがドッキングすることができました。

ちなみにドッキングの副産物としてウィンドウサイズと位置を記憶するようになります。
(Mayaを落とすとリセットされます。Mayaを落としても記憶させるには後述するworkSpaceControlというものを使う必要があります)
↑現時点でも内部でworkSpaceControlは使っていると思う。保存されてないだけだと思うから、mayaMixinをいじって検証してみる

![15.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/a526c798-a2ce-bc5c-3118-257d3bda59a4.gif)

# 6. Restoreできるようにする
そもそもRestoreとはなにかですが、
Mayaの起動時に**前回のウィンドウの配置情報を復元すること**です。

みなさんは日頃からMayaを使いやすいようにウィンドウの位置や大きさ、幅を変えていたりすると思いますが、
あれがMayaの起動時に毎回復元されているのはまさにRestoreの機能になります。
もちろん現在のTemplateWindowはRestoreされません。

Restoreできるようにするには以下の2つのことを行う必要があります。
- Restore用の関数を用意する
- show()のuiScriptフラグにRestore用の関数を渡す

今回は説明の都合上、
- Restore用の関数のガワだけつくる
- show()のuiScriptフラグにRestore用の関数を渡す
- Restore用の関数の実装をつくる

という流れで説明します

## 6.1 Restore用の関数とは
Restore用の関数がいつ、なんのためにで呼ばれるものなのかを説明すると、
**Mayaの起動時に呼ばれて**、
**GUIを再構築するため**のものになります。

まさにRestore用の関数というわけですね。

## 6.2 Restore関数のガワだけつくる
```diff_python: run.py
+def restore() -> None:
+   pass
```
run.pyにとりあえずガワだけ用意しておきます。

## 6.3 show()のuiScriptフラグにRestore用の関数を渡す
uiScriptフラグにはスクリプトの文字列そのものを渡す必要があります。
文字列を直打ちして書いても良いのですが少々スマートさに欠けるので、
今回は`inspectモジュール`の`getsource()`という関数を使います。
これは関数などを引数として渡すと文字列にして返してくれるというものです。
```diff_python: template_window.py
+import inspect
+from . import run
def show(self): # オーバーライド
+   restore_script = inspect.getsource(run.restore)
-   super().show(dockable=True)
+   super().show(dockable=True, uiScript=restore_script)
```

## 6.4 一度Mayaを落として起動し直してみる
さてここでMayaを再起動してなにが起きるか確認してみましょう。
当然ですがrestoreの確認なのでTemplateWindowを残したままMayaを落としてください。

そしてMayaを起動すると、

![{0C01B6A6-6B68-4A27-9C29-6D67C093F33B}.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e36c1270-ac67-ed1b-ba89-43c0286d3750.png)

TemplateWindowが復元されていますね。
しかしGUIの中身は再構築されていないようです。
本来であればrestore()に処理が書いてあるためうまく再構築されます。

さてrestore()の処理を書く前にrestore情報の保存先を見ていきます。
こちらを把握しておくことでrestore()になにを書けばよいかが見えてきます。

## 6.5 restore情報の保存先
restore情報はGeneral.jsonに保存されています。

このパスはMayaを起動した言語によって異なります。

```日本語の場合.txt
C:\Users\<ユーザーネーム>\Documents\maya\<Mayaのバージョン>\ja_JP\prefs\workspaces\General.json
```

```英語の場合.txt
C:\Users\<ユーザーネーム>\Documents\maya\<Mayaのバージョン>\prefs\workspaces\General.json
```

General.jsonというのはMayaのワークスペースの情報を記録したものです。
Mayaの右上のほうにある「ワークスペース」という部分ですね。

![16.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/0879e480-510b-fee1-0c4b-d1176ccc72f6.gif)

https://help.autodesk.com/view/MAYACRE/JPN/?guid=GUID-0384C282-3CA1-4587-9775-F7164D3F6980

Mayaを最初に起動したときは「一般(英語だとGeneral)」になっていると思います。

ワークスペースを編集している方はjsonの名前がGeneralではなく任意の名前になっていたりします。
またワークスペースは複数持てますので、restore情報が保存されるのはMayaを落としたときのワークスペースだけになります。

## 6.6 General.jsonを確認する
長いので抜粋しますが重要なのはこのあたりです。
```
{
    "mainWindowPanel": false,
    "posX": 731,
    "posY": 503,
    "splitter": {
        "orientation": "horizontal",
        "children": [
            {
                "tabWidget": {
                    "selectedIndex": 0,
                    "controlWidth": 342,
                    "controlHeight": 144,
                    "collapsed": false,
                    "controls": [
                        {
                            "objectName": "PysideTemplateWorkspaceControl",
                            "title": "PySide Template",
                            "uiScript": "python(\"def restore() -> None:\\n    pass\\n\");",
                            "retain": true,
                            "deleteLater": true,
                            "loadImmediately": true,
                            "checkPlugins": false,
                            "tabDirection": 0,
                            "closed": false,
                            "widthProperty": "free",
                            "heightProperty": "free",
                            "controlWidth": 342,
                            "controlHeight": 144
                        }
                    ]
                }
            }
        ]
    }
},
```
`"objectName": "PysideTemplateWorkspaceControl"`とありますね。
この部分で間違いなさそうです。`WorkspaceControl`というのはあとで解説します。

`"uiScript": "python(\"def restore() -> None:\\n    pass\\n\");"`とありますね。
これは先ほど`show()`のuiScriptフラグで渡した文字列そのものです。
MELに変換されているのとバックスラッシュばかりで見辛いですが内容的には、
```
def restore() -> None:
   pass
```
と同じですね。

## 6.7 restoreの流れを整理する
1. Mayaを落とす
2. General.jsonにTemplateWindowの情報(uiScriptなど)が保存される
3. Mayaを起動する
4. MayaがGeneral.jsonを読みに行く
5. TemplateWindowのガワだけ作られる TODO: workspaceControlが生成されているということ？ちゃんと調べる
6. uiScriptが実行される(GUIが再構築される)

という流れになります。

## 6.8 Restore用の関数の実装をつくる
ではようやくGUIを再構築する処理を書いていきます。
```diff_python: run.py
def restore() -> None:
-   pass
+   # WARNING: 破棄されないようにクラス変数に保存しておく
+   TemplateWindow.restored_instance = __create_window()
+   # PysideTemplate
+   ptr = omui.MQtUtil.findControl(TemplateWindow.name)
+   # PysideTemplateWorkspaceControl
+   restored_control = omui.MQtUtil.getCurrentParent()
+   # 親子付け処理
+   omui.MQtUtil.addWidgetToMayaLayout(int(ptr), int(restored_control))
```
またTemplateWindowにresotored_instanceというクラス変数を用意しておきます。
```diff_python: template_window.py
class TemplateWindow(mayaMixin.MayaQWidgetDockableMixin, QMainWindow):
+   restored_instance = None
```
重要なところなので一つずつ解説します

### 6.8.1 インスタンスを生成する
```diff_python: run.py
def restore() -> None:
+   # WARNING: 破棄されないようにクラス変数に保存しておく
+   TemplateWindow.restored_instance = __create_window()
```
まずは`__create_window()`を呼んでTemplateWindowのインスタンスを生成します。
ここで注意なのは生成したインスタンスはローカル変数に入れてはいけないということです。
restore()が呼ばれているときはMayaの起動中のため、ローカル変数に入れただけではスコープ外になった瞬間に破棄されてしまいます。
なのでクラス変数やグローバル変数などの寿命が長い変数に入れることで即座に破棄されることを防ぐ必要があります。
```diff_python: template_window.py
class TemplateWindow(mayaMixin.MayaQWidgetDockableMixin, QMainWindow):
+   restored_instance = None
```
今回はTemplateWindowのクラス変数に入れています。

### 6.8.2 インスタンスのポインタを取得する
```diff_python: run.py
def restore() -> None:
    # WARNING: 破棄されないようにクラス変数に保存しておく
    TemplateWindow.restored_instance = __create_window()
+   # PysideTemplate
+   ptr = omui.MQtUtil.findControl(TemplateWindow.name)
```
`omui.MQtUtil.findControl()`を使って`TemplateWindow.restored_instance`に入っているインスタンスのポインタを取得します。
引数は名前なので`TemplateWindow.name`を渡します(実際の文字列としては`'PysideTemplate'`ですね)

### 6.8.3 ワークスペースコントロールを取得する
TODO: ワークスペースコントロールの説明がガバガバすぎるのでもう少し正確に書く
```diff_python: run.py
def restore() -> None:
    # WARNING: 破棄されないようにクラス変数に保存しておく
    TemplateWindow.restored_instance = __create_window()
    # PysideTemplate
    ptr = omui.MQtUtil.findControl(TemplateWindow.name)
+   # PysideTemplateWorkspaceControl
+   restored_control = omui.MQtUtil.getCurrentParent()
```
`omui.MQtUtil.getCurrentParent()`を使うと現在の親、
つまりGeneral.jsonを呼んでいるときのオブジェクトが呼ばれます。
General.jsonを見たときに、
`"objectName": "PysideTemplateWorkspaceControl"`というものが出てきましたがまさにこれのことです。

### 6.8.4 ワークスペースコントロールに親子付けする
```diff_python: run.py
def restore() -> None:
    # WARNING: 破棄されないようにクラス変数に保存しておく
    TemplateWindow.restored_instance = __create_window()
    # PysideTemplate
    ptr = omui.MQtUtil.findControl(TemplateWindow.name)
    # PysideTemplateWorkspaceControl
    restored_control = omui.MQtUtil.getCurrentParent()
+   # 親子付け処理
+   omui.MQtUtil.addWidgetToMayaLayout(int(ptr), int(restored_control))
```
`omui.MQtUtil.addWidgetToMayaLayout()`でワークスペースコントロールとPysideTemplateを親子付けします。

階層構造は以下です。
MayaMainWindow
└workspaceControl(Layout)
　└QWidget(PysideTemplate)

## 6.9 Mayaを落として起動し直してみる(再)
restore()の処理が書けたので早速Mayaを再起動してみましょう。

![{B5B83881-6082-49C6-94FB-6E6BEB9151D4}.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/b916dca6-2b34-ad54-e01f-6d197e8b0785.png)

無事復元されているようです。

## 6.10 restore()の取り回しを改善する
このままでも十分良いのですが、
今のuiScriptは処理が全文General.jsonに書き込まれてしまっています。
今General.jsonのuiScriptに何が書き込まれているかを意識するのは面倒なため、
uiScriptに渡すのは`run.restore()を呼ぶ処理`にしてしましましょう。

具体的にはrestore.pyというファイルを新規作成します。
```diff_python: restore.py
def restore_pyside_template_window():
    from pyside_template_window import run
    run.restore()

if __name__ == '__main__':
    restore_pyside_template_window()
```
そしてuiScriptではこのファイル(モジュール)を渡すように変更します。
```diff_python: template_window.py
-from . import run
+from . import restore
def show(self):
-   restore_script = inspect.getsource(run.restore)
+   restore_script = inspect.getsource(restore)
    super().show(dockable=True, uiScript=restore_script)
```
![{9126105D-2141-430A-8215-AD525CB2FC66}.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/e027ed6d-8b57-e3aa-8a53-1d6b2f26d39c.png)

問題ないようですね。
こちらのほうが管理もしやすいですしトラブルも起きにくいと思います。

## 6.x workSpaceControlについて
しっかり重点的に書きたい
.parent()などがどうなっているかもしっかり見せていく
restore()関数を一度素で呼んでみて見せたらかなりわかりやすい気がする

## 6.x retainについて


# 7. reloadできるようにする
Maya上でPythonツールを開発する際、
ソースコードを変更するたびにMayaを立ち上げるのは大変です。
これを解消するためには`reload()`を使うことになります。

maya.cmdsを用いた開発で馴染みのある方も多いと思いますが、
PySideでも使います。

## 7.1 reloadタイミング
よくあるのはウィンドウの起動時に必ずreloadをするというアプローチです。
**ですが本記事ではMayaの標準的なウィンドウと同挙動のウィンドウを目指します。**
なので別途reloadボタンを用意し、そのスロットでreloadするような挙動を実装します。

## 7.2 reloadボタンを実装する
reloadボタンは`MenuBar`というクラスを使って実装していきます。
```diff_python: template_window.py
try:
-   from PySide6.QtWidgets import QMainWindow, QPushButton
+   from PySide6.QtGui import QAction
+   from PySide6.QtWidgets import QMainWindow, QMenu, QPushButton
except ImportError:
-   from PySide2.QtWidgets import QMainWindow, QPushButton
+   from PySide2.QtWidgets import QAction, QMainWindow, QMenu, QPushButton

class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
    def init_gui(self) -> None:
+       # メニューバー
+       menu_bar = self.menuBar()
+       dev_menu = menu_bar.addMenu("Dev")
+       restart_action = QAction('Restart', self)
+       restart_action.triggered.connect(lambda *arg: self.__restart_dummy())
+       dev_menu.addAction(restart_action)

+       # ボタン
        push_button = QPushButton('PUSH ME', self)
        push_button.clicked.connect(lambda *arg: self.__print_hello_world())
        self.setCentralWidget(push_button)

+   def __restart_dummy(self) -> None:
+       print('Restart!')

```
`QAction`はPySide2では`QtWidgets`にありますが、PySide6では`QtGui`にありますので気をつけてください。

実行すると下図のようになります。
![08.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/a7deb285-225d-be20-f5e9-17c3acd930bf.gif)

## 7.3 reloadボタンのスロットを実装する
実際にreload()するスロットを実装していきます。
リロードはウィンドウを一度閉じて再び開く必要があるためTemplateWindow側に処理を持つのは少し都合が悪いです。
なので処理はrun.pyにrestart()という関数として実装します。

```diff_python: run.py
+from maya import cmds

+def restart() -> None:
+   if cmds.workspaceControl(TemplateWindow.workspace_control, q=True, exists=True):
+        # すでに存在しているWindowは削除する
+        cmds.deleteUI(TemplateWindow.workspace_control, control=True)
+
+   window = __create_window()
+   window.show()
```

```diff_python: template_window.py
+from . import restart

class TemplateWindow(MayaQWidgetDockableMixin, QMainWindow):
    def init_gui(self) -> None:
        menu_bar = self.menuBar()
        dev_menu = menu_bar.addMenu("Dev")
        restart_action = QAction('Restart', self)
-       restart_action.triggered.connect(lambda *arg: self.__restart_dummy())
+       restart_action.triggered.connect(lambda *arg: restart())
        dev_menu.addAction(restart_action)
```

## 7.3 reloadの実装が最後になった理由
realodの実装にはworkspaceControlが深く関わっています。
DockableやRestoreの実装が終わってからのほうが大変都合が良かったので、
最後の実装になっています。

# 注意事項
これらの実装は互いに深く影響し合っているため、
ある特定の部分(例えばreload)だけを実装してみてもおそらくうまくいきません。
本記事ではその関係性まで深くは解説しませんが、そういう関係性があるということはご理解いただけると幸いです

# TODO:
- objectName()とmaya.OpenMayaUI.MQtUtil.findControl()の絡みがあるので、MayaQWidgetBaseMixinを継承するのはもう少しあとでいいかもしれない
- workspaceControlは自前実装解説を入れたほうがいいかもしれない


# メモ
>ウィジェットを使用し、maya.OpenMayaUI.MQtUtil.findControl() からルックアップできるようにするには、ウィジェットに一意の objectName() が必要です。
https://help.autodesk.com/view/MAYADEV/2025/JPN/?guid=Maya_DEVHELP_Maya_Python_API_Working_with_PySide_in_Maya_html

http://leavebehind.iobb.net/wordpress/2016/12/14/mac%E7%89%88mayapyside%E3%81%A7%E4%BD%9C%E6%88%90%E3%81%97%E3%81%9F%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6%E3%81%8C%E3%83%A1%E3%82%A4%E3%83%B3%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6/

# 参考
https://tommy-on.hatenablog.com/entry/2019/04/14/231938

https://help.autodesk.com/view/MAYADEV/2025/JPN/?guid=Maya_DEVHELP_Maya_Python_API_Writing_Workspace_controls_html

https://qiita.com/sporty/items/a26ea7e4691437a6e8c8
