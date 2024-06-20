---
title: 【Maya】reload()したらsuper(class, self)でTypeErrorが出た話【Python】
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
- MayaPython上の話です

# 環境
- Autodesk Maya 2020.4 (Python 2.7.11)
- Autodesk Maya 2023.3 (Python 3.9.7)

# 経緯
super(class, self)を使ったスクリプトでreload()を行うと、
> TypeError: super(type, obj): obj must be an instance or subtype of type

というようなエラーが出てしまう。

# 結論
クラスのidが破綻しないようにreload()する順番に気をつける

以下詳しく解説します。

# なにが起きているのか
まずreload()によってクラスにどんな変化が起こるかを理解する必要があります。

下図のディレクトリ構成を例にして説明します。
>super_test/
├__init__.py
├darkdragon.py
├dragon.py
├main.py
└util.py
start.py(scriptEditorから起動する用)

まずdragon.pyにDragonクラスを定義したとします。
```dragon.py
class Dragon(object):
    def __init__(self, hp, atk):
        self.__hp = hp
        self.__atk = atk

    def attack(self):
        print(u'攻撃!')
```

実際にattack()を呼び出してみます。
```main.py
from .dragon import Dragon

def main():
    dragon = Dragon(130, 80)
    dragon.attack()
```
```start.py
from super_test import main

main.main()
```
```output.txt
攻撃!
```
ここまでは問題ありませんね。
ではDragonクラスをreloadしてみましょう。
```start.py
from importlib import reload
from super_test import dragon, main

reload(dragon)
main.main()
```
```output.txt
攻撃!
```
問題ありませんね。
ここで1つreloadにおける重要な要素を確認します。
それはidです。

# id
main.pyに**Dragonクラスのidをプリントする**一文を追加します。
```diff_python: main.py
from .dragon import Dragon

def main():
+   print(id(Dragon), 'in main function')
    dragon = Dragon(130, 80)
    dragon.attack()
```
実行してみます。
```output.txt
2993528039936 in main function
攻撃!
```
13桁の数字が出力されました。
これがidです。
もう一度実行してみましょう。
```output.txt
2993528028608 in main function
攻撃!
```
>2993528039936
2993528028608

数字が微妙に変わりましたね。
idというのはクラスの情報が格納されているメモリ番号です。
リロードするたびに違う番号が割り当てられます。
要は**全く同じクラスだったとしても別物**として扱われるのです。

ためしにこんな実験をしてみましょう。
```start.py
from importlib import reload
from super_test import dragon

print(id(dragon.Dragon), 'first import')
reload(dragon)
print(id(dragon.Dragon), 'second import')
```
実行すると
```output.txt
2993528055984 first import
2993528056928 second import
```
やはりidが変わっています。

# super(class, self)を使う
さてここからが本題です。
まずはDragonクラスを継承してDarkDragonクラスをつくります。
```darkdragon.py
from .dragon import Dragon

class DarkDragon(Dragon):
    def __init__(self, hp, atk):
        super(DarkDragon, self).__init__(hp, atk)
```
インスタンス化するオブジェクトもDragonからDarkDragonに変更します。
```main.py
from .darkdragon import DarkDragon

def main():
    dark_dragon = DarkDragon(130, 80)
    dark_dragon.attack()
```
```start.py
from importlib import reload
from super_test import darkdragon, main

reload(darkdragon)
main.main()
```
実行すると
```output.txt
攻撃!
```
無事実行できました。

それではここで**main.pyもreload**してみましょう
```start.py
from importlib import reload
from super_test import darkdragon, main

reload(main)
reload(darkdragon)
main.main()
```
実行すると
```output.txt
# エラー: TypeError: file C:/Users/UserName/Document/maya/scripts/super_test/darkdragon.py line 5: super(type, obj): obj must be an instance or subtype of type
```
エラーが出てしまいました。
これが本題。
冒頭でお話した問題です。

# エラーを読む
>super(type, obj): obj must be an instance or subtype of type

**super(type, obj)のobjは「typeのinstance」または「typeのsubtype」でなければならない**
だそうです。

今回エラーが出ているのは下記の一文ですから、
```python: darkdragon.py line 5
super(DarkDragon, self).__init__(hp, atk)
```

この変数名に合わせるなら、
**super(DarkDragon, self)のselfは「DarkDragonのinstance」または「DarkDragonのsubtype」でなければならない**
となります。
DarkDragonを更に継承したりはしていないので、今回該当するのは前者の「DarkDragonのinstance」です。

でもおかしいですね。selfはちゃんとDarkDragonのinstanceのはずです...


本当にそうでしょうか？
先ほどreload()するとクラスのidが変わってしまうということを確認しました。
printデバッグをしてidがどうなっているか確認してみましょう。


# おまけ
importlib.reload()はをPython2では使えません。
Python2では組み込みのreload()を使う必要があります。
下記のようなutilを作ると便利です。
```util.py
import sys

def custom_reload(module):
    if get_python_major_version() == 2:
        reload(module)      # Python2系
    else:
        import importlib    # Python3系
        importlib.reload(module)

def get_python_major_version():
    ver = sys.version
    major_ver = ver.split('.')[0]
    return int(major_ver)
```


# そもそもとして
Python3系のみのサポートでよい場合は、引数が必要ない方のsuper()を使いましょう。
それが一番楽です。

# おまけ2
引数が必要ない方のsuper()は引数を推論しています。
なので推論できない場合は使えません。
例えばリスト内包表記など。
