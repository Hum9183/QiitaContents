---
title: '【Maya】reload()したらsuper(class, self)でTypeErrorが出た話【Python】'
tags:
  - Python
  - maya
private: false
updated_at: '2024-06-22T12:29:50+09:00'
id: b6cb9abffa225619710a
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
- MayaPython上の話です

# 環境
- Autodesk Maya 2020.4 (Python 2.7.11)
- Autodesk Maya 2023.3 (Python 3.9.7)

# サンプルプロジェクト
https://github.com/Hum9183/super_test

# 経緯
super(class, self)を使ったスクリプトでreload()を行うと、
> TypeError: super(type, obj): obj must be an instance or subtype of type

というようなエラーが出てしまう。

# 結論
クラスのidが破綻しないようにreload()する順番に気をつける。

以下詳しく解説します。

# なにが起きているのか
まずreload()によってクラスにどんな変化が起こるかを理解する必要があります。

下図のディレクトリ構成を例にして説明します。
>super_test/
├__init__.py
├darkdragon.py
├dragon.py
└main.py
start.py(scriptEditorから起動する用)

まずdragon.pyにDragonクラスを定義したとします。
(実装内容自体に特に意味はありません)
```dragon.py
class Dragon(object):
    def __init__(self, hp, atk):
        self.__hp = hp
        self.__atk = atk

    def attack(self):
        print(u'攻撃!')
```

attack()を呼び出してみます。
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
ここで1つreload()における重要な要素を確認します。
それは**id**です。

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
つまり**全く同じクラスだったとしても別物**として扱われるのです。

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

# idを確認する
```get_class.py
hoge = Hoge()
print(hoge.__class__)
```
インスタンス.\_\_class__でそのインスタンスのクラスを取得することができます。
これを活用してDarkDragonのイニシャライザにprintを仕込んでみます。
```darkdragon.py
from .dragon import Dragon

class DarkDragon(Dragon):
    def __init__(self, hp, atk):
        print(id(self.__class__), 'class of self')
        print(id(DarkDragon), 'in DarkDragon class')
        super(DarkDragon, self).__init__(hp, atk)
```
実行してみます
```output.txt
2321255725456 class of self
2321255697136 in DarkDragon class
```
予想通りではありますが残念なことにidが異なるようです。

先に答えを言ってしまうと、
reload()の順番を逆にすることでidは同じになります。
darkdragon -> main の順にreload()するということですね。
```diff_python: start.py
from importlib import reload
from super_test import darkdragon, main

+reload(darkdragon)
reload(main)
-reload(darkdragon)
main.main()
```
実行すると
```output.txt
2321255716016 class of self
2321255716016 in DarkDragon class
攻撃!
```

# なにがどのタイミングでreloadされているのか
この問題を理解するにはreload()の挙動を理解する必要があります。

importlib.realod()はモジュールを引数にして、そのモジュールをリロードします。
しかし**そのモジュールがインポートしているモジュール**はリロードされません。
つまり**再帰的にリロードするわけではない**のです。

```start.py
from importlib import reload
from super_test import darkdragon, main

reload(main) # mainはリロードされる。mainがimportしているモジュール(darkdragonなど)はリロードされない
reload(darkdragon)
main.main()
```
つまりreload(main)時点では古いdarkdragonが読み込まれることになります。
そして次にreload(darkdragon)することでdarkdragonモジュールはリロードされます。

Pythonのモジュールインポートは自身のモジュールに取り込むという仕様になっており、
モジュール同士が参照で繋がっているわけではありません。
**darkdragonモジュールをリロードして新しくしても、
main内のdarkdragonモジュールは古いままです。**

ですのでこれを解決する場合、
**先にdarkdragonをリロードしてしまい、後でmainをリロードします。
こうすればmainのリロード時にmain内のdarkdragonは最新のものが取り込まれます。**

```start.py
from importlib import reload
from super_test import darkdragon, main

reload(darkdragon) # darkdragonがリロードされる
reload(main) # mainはリロードされる。mainがimportしているモジュール(darkdragonなど)はリロードされないが、現時点でのdarkdragonはリロード済みなので、結果的に新しいdarkdragonをインポートすることになる
main.main()
```

# おわりに
いかがでしたか？
Pythonのモジュールインポートまわりは色々と複雑なので、
興味がある方はぜひ調べてください！

またこのような問題を根本的に解決するために、
再起reloadを自作するのも面白いかもしれませんね。

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
Python3系のみのサポートでよい場合は、引数が無いsuper()を使いましょう。
それが一番楽です。

# おまけ2
引数が無いsuper()はクラス内のメソッドからしか使うことができません。
なのでリスト内包表記などのスコープが別になってしまう記法では引数が無いsuper()は使えません。
今回の問題と同じErrorが出ます。

しかし逆に言えば引数ありのsuper()はクラス外のスコープからも呼べるということですね。

# 参考
https://docs.python.org/ja/3.5/library/functions.html?highlight=type#super
https://qiita.com/risuoku/items/a6bda20e0004660a65de
https://qiita.com/Mt_Snow/items/6af7f94295dc572598a1
