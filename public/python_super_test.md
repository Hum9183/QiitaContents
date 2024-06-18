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




# そもそもとして
Python3系のみのサポートでよい場合は、引数が必要ない方のsuper()を使いましょう。
それが一番楽です。
