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
start.pyはScriptEditorで実行する文です。

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

# 1.1 sys.exit()が必要な理由
いろいろと調べたのですが正直分かりませんでした。
有識者の方、コメントお待ちしております。

ちなみにスコープが絡んでいるところまではなんとなく掴んでいます。
```start.py
from pyside_template_window.template_window import TemplateWindow

window = TemplateWindow()
window.show()
```
たとえば上記のようにScriptEditorの一番上のスコープで直接インスタンス化&show()を実行すればすぐに消えません。
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

# 2.1 ボタンにスロットを接続する
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
ほかにも様々な種類のシグナルがあるため、いろいろなことを引き金にして関数を呼び出すことができます。

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
しかしこれだととても使いづらいツールになってしまいます。

# 2.2 ウィンドウが後ろに行ってしまう原因
冒頭で軽く触れましたが、**MayaのUIはQtで動いています**。

Qtのウィンドウが2つある状況を想像してみてほしいのですが、
片方がアクティブになった場合、もう片方のウィンドウが後ろに隠れてしまうという挙動は自然に思えます。

これを回避する場合、「片方のウィンドウをもう片方のウィンドウに親子付けする」という方法が考えられます。
ということでMayaのMainWindowに親子付けします。

# 3. MayaのMainWindowに親子付けする
実はMayaのMainWindowに親子付けするための便利なクラスをAutodeskさんの方で用意してくれているのですが、
ひとまずは理屈を理解するために自前で実装してみましょう。

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
気になるポインタの取得方法ですが、__int__を持っているので、intにキャストすることで取得できるようです。
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
wrapInstance()の第一引数には`インスタンスに変換したいポインタ`を、第二引数には`変換する型`を渡します。

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
Mayaをクリックしてもウィンドウが後ろに行かなくなりました。

# 3.1 sys.exit()を削除する
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

# 3.2 MayaQWidgetBaseMixinを継承する
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

    def init_gui(self) -> None:
        ...

    def __print_hello_world(self) -> None:
        ...
```
1つ注意点として`MayaQWidgetBaseMixin`の継承の記載順があります。
今は`MayaQWidgetBaseMixin` -> `QMainWindow`の順番で記載しましたが、
これを逆にすると意図した動きになりません。
```diff_python: template_window.py
-class TemplateWindow(MayaQWidgetBaseMixin, QMainWindow):
+class TemplateWindow(QMainWindow, MayaQWidgetBaseMixin):
```
![07.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121056/f07dfc3f-6746-0119-b0d4-5aa22609cc6d.gif)

この問題は菱形継承問題といいます。

https://ja.wikipedia.org/wiki/%E8%8F%B1%E5%BD%A2%E7%B6%99%E6%89%BF%E5%95%8F%E9%A1%8C

現在`TemplateWindow`は`QMainWindow`と`MayaQWidgetBaseMixin`の２つを継承しています。
これを**多重継承**といいます。

この状況で`super().__init__()`を呼んだ場合、`QMainWindow`と`MayaQWidgetBaseMixin`
のどちらの`__init__()`が呼ばれるのでしょうか？

この結果はプログラミング言語によって異なるのですが、
Pythonの場合は、


# TODO:
objectName()とmaya.OpenMayaUI.MQtUtil.findControl()の絡みがあるので、
MayaQWidgetBaseMixinを継承するのはもう少しあとでいいかもしれない



# メモ
>ウィジェットを使用し、maya.OpenMayaUI.MQtUtil.findControl() からルックアップできるようにするには、ウィジェットに一意の objectName() が必要です。
https://help.autodesk.com/view/MAYADEV/2025/JPN/?guid=Maya_DEVHELP_Maya_Python_API_Working_with_PySide_in_Maya_html

http://leavebehind.iobb.net/wordpress/2016/12/14/mac%E7%89%88mayapyside%E3%81%A7%E4%BD%9C%E6%88%90%E3%81%97%E3%81%9F%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6%E3%81%8C%E3%83%A1%E3%82%A4%E3%83%B3%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6/

# 参考
https://tommy-on.hatenablog.com/entry/2019/04/14/231938
