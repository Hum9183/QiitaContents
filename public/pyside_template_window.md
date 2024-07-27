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
# はじめに
- この記事はMayaツールのウィンドウの雛形をつくってみたものです
- PySideを使用します
- **Mayaの標準的なウィンドウ(NodeEditorやScriptEditorなど)と同じように動作することを目指します**

# 注意
- 本記事内のソースコードは尺の都合上、一部(import文や焦点があたっていないメソッドなど)を省略することがあります

# 対象読者
- PySideのことはよくわからないけど、とりあえずウィンドウをつくってみたい人
- PySideのウィンドウについて理屈から理解したい人
- reloadできるウィンドウをつくりたい人
- Dockableなウィンドウをつくりたい人
- Restoreできるウィンドウをつくりたい人

# 環境
- Autodesk Maya 2022.5.1 (Python 3.7.7)
- Autodesk Maya 2025.1 (Python 3.11.4)

# 本記事のソースコード
本記事では理屈での理解を促すためにわざと遠回りしたり、
詰まったところも含めて解説します(要は長いです)
先に完成品を見たい方はこちらをご覧ください。
https://github.com/Hum9183/pyside_template_window

# ディレクトリ構成
最終的な構成は以下になります

pyside_template_window/
├__init__.py
├restart.py
├restore.py
├run.py
└template_window.py
start.py(ScriptEditorから起動する用)

# 0. PySideとは
`Qt(キュート)`というC++で書かれたフレームワークのPythonバインディング(Pythonで使えるようにしたもの)です。

なおQtはGUI以外にもいろいろな機能があるのですが、
ことMayaのツール作成で使う場合においてはシンプルに`GUIライブラリ`と捉えても問題ないと思います。

またQtのPythonバインディングには`PyQt`というものもあるのですが、
こちらは別物ですので注意が必要です(別物というほど別物ではないのですがライセンスが異なるので同一視してはいけません)

ちなみにMayaのUIはQtで動いています。

# 1. PySideでウィンドウを出す
まずはPySideの普通のウィンドウを出してみます
```template_window.py
try:
    from PySide6.QtWidgets import QMainWindow
except ImportError:
    from PySide2.QtWidgets import QMainWindow

class TemplateWindow(QMainWindow):
    pass
```
普通のウィンドウを作る場合はQMainWindowを継承したクラスを作ります。
Maya2025からはPySide6になっているため、importの書き方に気をつけてください。
```run.py
from .template_window import TemplateWindow

def start() -> None:
    window = TemplateWindow()
    window.show()
```
後でreloadにも対応することを見越して、
インスタンス生成処理はファイルを分けておきます。
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

ボタンはQPushButtonというクラスを使います。
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
今は`MayaQWidgetBaseMixin` -> `QMainWindow`の順番で記載しましたが、
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
`TemplateWindow`にはなんだかんだで名前が設定していませんでした。
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
Mayaの標準的なウィンドウではほかのGUIとドッキングをすることができます。

![13.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/2d345d15-5d01-db25-705e-c14a7da8f562.gif)

もちろん現在のTemplateWindowではできません。

# 6. reloadできるようにする
Maya上でPythonツールを開発する際、
ソースコードを変更するたびにMayaを立ち上げるのは大変です。
これを解消するためには`reload()`を使うことになります。

maya.cmdsを用いた開発で馴染みのある方も多いと思いますが、
PySideでも使います。

## 6.1 reloadタイミング
よくあるのはウィンドウの起動時に必ずreloadをするというアプローチです。
**ですが本記事ではMayaの標準的なウィンドウと同挙動のウィンドウを目指します。**
なので別途reloadボタンを用意し、そのスロットでreloadするような挙動を実装します。

## 6.2 reloadボタンを実装する
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

## 6.3 reloadボタンのスロットを実装する
実際にreload()するスロットを実装していきます。
処理はrun.pyに関数として実装します。


# TODO:
objectName()とmaya.OpenMayaUI.MQtUtil.findControl()の絡みがあるので、
MayaQWidgetBaseMixinを継承するのはもう少しあとでいいかもしれない

# メモ
>ウィジェットを使用し、maya.OpenMayaUI.MQtUtil.findControl() からルックアップできるようにするには、ウィジェットに一意の objectName() が必要です。
https://help.autodesk.com/view/MAYADEV/2025/JPN/?guid=Maya_DEVHELP_Maya_Python_API_Working_with_PySide_in_Maya_html

http://leavebehind.iobb.net/wordpress/2016/12/14/mac%E7%89%88mayapyside%E3%81%A7%E4%BD%9C%E6%88%90%E3%81%97%E3%81%9F%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6%E3%81%8C%E3%83%A1%E3%82%A4%E3%83%B3%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6/

# 参考
https://tommy-on.hatenablog.com/entry/2019/04/14/231938
