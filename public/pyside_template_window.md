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
- reloadできるウィンドウをつくりたい人
- Dockableなウィンドウをつくりたい人
- Restoreできるウィンドウをつくりたい人

# 環境
- Autodesk Maya 2022.5.1 (Python 3.7.7)
- Autodesk Maya 2025.1 (Python 3.11.4)

# 本記事のソースコード
本記事では制作において詰まったところも含めて解説します(要は結構ゆっくり進行します)
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

# 1. PySideでウィンドウを出す
まずはPySideの普通のウィンドウを出してみます
```template_window.py
try:
    from PySide6.QtWidgets import QMainWindow
except ImportError:
    from PySide2.QtWidgets import QMainWindow

class TemplateWindow(QMainWindow):
    def __init__(self):
        super().__init__()
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

しかしながら後述するMayaのMainWindowとParent化することによっても消えなくなるので、
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
    def __init__(self):
        super().__init__()

+   def init(self):
+       push_button = QPushButton('PUSH ME', self)
+       self.setCentralWidget(push_button)
```

run.pyではinit()を呼んでおきます。

```diff_python: run.py
import sys
from .template_window import TemplateWindow

def start() -> None:
    window = TemplateWindow()
+   window.init()
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

今回はボタンを押したら`Hello, World!`がprintされるスロットを接続します。
```diff_python: template_window.py
try:
    from PySide6.QtWidgets import QMainWindow, QPushButton
except ImportError:
    from PySide2.QtWidgets import QMainWindow, QPushButton

class TemplateWindow(QMainWindow):
    def __init__(self):
        super().__init__()

    def init(self):
        push_button = QPushButton('PUSH ME', self)
+       push_button.clicked.connect(lambda *arg: self.__hello_world())
        self.setCentralWidget(push_button)

+   def __hello_world(self):
+       print('Hello, World!')
```
`push_button.clicked`というのがシグナルです。`ボタンをクリックしたとき`ということですね。
ほかにも様々な種類のシグナルがあるため、いろいろなことを引き金にして関数を呼び出すことができます。

`lambda *arg: self.__hello_world()`というのがスロットです。
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

# 3. MayaのMainWindowに親子付けする

TODO: 最初のほうのTemplateWindowクラスのイニシャライザはいらないかも。
無い方が読者は頭に入ってきやすそう

# 参考
https://tommy-on.hatenablog.com/entry/2019/04/14/231938
