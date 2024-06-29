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
start.py(scriptEditorから起動する用)

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
start.pyはscriptEditorで実行する文です。

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
たとえば上記のようにscriptEditorの一番上のスコープで直接インスタンス化&show()を実行すればすぐに消えません。
(しかしGlobal空間を汚すことになるのであまり褒められた書き方ではありません)

しかしながら後述するMayaのMainWindowとParent化することによっても消えなくなるので、
あまり気にしなくてもよいです。

# 2. ボタンを追加する

# 参考
https://tommy-on.hatenablog.com/entry/2019/04/14/231938
