<!--
title:   Pythonのクラス変数とインスタンス変数の区別ついてますか？これでもう間違えない！
tags:    Instance,Python,class,variable
id:      afab2410adec60854b38
private: false
-->
突然ですが、次のコードは何を出力するでしょうか？
`python
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

ayumi.kokugo = 90
Exam.kokugo = -1
print(Exam.kokugo, ayumi.kokugo, genta.kokugo)
`-1 90 -1`と答えた方は本記事は読まなくても大丈夫です（が良ければ神の目線でご覧ください）。
そうでない方はぜひ読んで行って下さい。


## Pythonのクラス変数とインスタンス変数
Pythonは好きな言語の1つですが、Pythonでよく分からないと言われることにクラス変数とインスタンス変数があります。
この2つが合わさった挙動は直感に反する場合があり、新たなPython開発者を混乱させることがあります。

この記事ではPythonのクラス変数とインスタンス変数で混乱してしまう内容について具体例を通して確認し、「クラスやインスタンスが持つ変数の定義はこう書こう」というのを簡単に例示してみます。

## 想定外の値変化
ある小学校で、テストの結果を管理するプログラムを作りたいとします。
次のサンプルコードを見てください。

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

print(ayumi.kokugo, genta.kokugo) # -> 0 0
ayumi.kokugo = 90
print(ayumi.kokugo, genta.kokugo) # -> 90 0
genta.kokugo = 30
print(ayumi.kokugo, genta.kokugo) # -> 90 30
```

`Exam`は小テストの点数を管理するクラスです。ここでは国語の点数だけ定義しています。
国語の点数をセットする前は0が表示され、セットした後はセットした値が出力されます。何も問題ないように見えます。
ではちょっと変なことをしてみましょう。

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

print(ayumi.kokugo, genta.kokugo) # -> 0 0
ayumi.kokugo = 90
print(ayumi.kokugo, genta.kokugo) # -> 90 0  ...①
Exam.kokugo = -1 # +++
print(ayumi.kokugo, genta.kokugo) # -> 90 -1 ...②
genta.kokugo = 30
print(ayumi.kokugo, genta.kokugo) # -> 90 30 ...③

```

国語のデフォルト値を途中で変えてみました。
②のprint文を見てください。元太くんの国語は`-1`が出力されています。しかし元太くんの国語には何も代入していません。
ここに違和感を覚える方は多いのではないでしょうか。`90 0`が表示されるはずだと思ってしまいます。
何が起こっているのでしょうか？

## インスタンス生成時の変数はどうなっているのか
コードの最初から、1つ1つ追っていきたいと思います。
まずインスタンスを作った直後のメモリ状態のイメージを見てみましょう（以降メモリイメージと呼びます）。

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()
```

この状態では、Examクラスやayumi, gentaインスタンスのメモリイメージはどういう状態でしょうか。
下図に近い状態を想像するのではないでしょうか。

![memory1_1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/f6d265a2-ce64-bc76-863b-5d69a62881cf.png)

残念ながらこれは誤りです。
`ayumi`, `genta`インスタンスを生成した段階では`ayumi.kokugo`と`genta.kokugo`というインスタンス変数はありません。
![memory1_2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/e0ffe360-68f7-a2c5-8964-f3850c4d5d91.png)

では、インスタンス生成直後の`ayumi.kokugo`, `genta.kokugo`は何故アクセスできるのでしょうか？また、何を指しているのでしょうか？
実はこの時点では、この2つの変数名はクラス変数である`Exam.kokugo`と同じメモリを指しています。
この時点で`ayumi.kokugo`はインスタンス変数にアクセスするような記述ですが、クラス変数にアクセスしています。


![memory1_3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/f4d2cb42-5828-0e6b-cf1e-dc06d1ecd3c4.png)

試しに`Exam.kokugo`, `ayumi.kokugo`, `genta.kokugo`のIDを見てみます。確かに、同じIDが出力されます。

注) ID出力について *[サンプルコードにおける補足](https://qiita.com/ugu/items/afab2410adec60854b38#%E8%A3%9C%E8%B6%B3-%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%B3%E3%83%BC%E3%83%89%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8Bid%E7%A2%BA%E8%AA%8D)* を本記事の最後に載せています。

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
 # -> 4347660720 4347660720 4347660720
```

この3つは、同じ場所を指しているようです。
では、それを踏まえて`ayumi.kokugo`に90点を代入してみましょう。
分かりやすいように`Exam.kokugo`も一緒に出力します。
`python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
ayumi.kokugo = 90 # +++
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> 0 90 0
`

出力について変に思うかもしれません。
「同じ場所を示すのならば、`ayumi.kokugo = 90`を実行すると`Exam.kokugo`, `genta.kokugo`も90になって、`90 90 90`になるはずじゃないか？」

## 代入文でインスタンス変数が作られる
実は、**Pythonでは`ayumi.kokugo = 90`のようにインスタンス変数への代入文の中で、初めてインスタンス変数が作られます。** そして、以降`ayumi.kokugo`はそのインスタンス変数を指します。
ここが分かりにくいポイントです。

ではIDを出して確認してみましょう。

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
 # -> 4312050096 4312050096 4312050096
ayumi.kokugo = 90
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> 0 90 0
print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
 # -> 4312050096 4312052976 4312050096
```

`ayumi.kokugo`のIDが変わっていますね。
メモリイメージは次のようになります。

![memory1_4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/950c5ed2-8c0e-529b-121e-d2d6c0749fda.png)

ayumiに新しくメモリ領域が確保され、`ayumi.kokugo`はそちらを指します。
ここでクラス変数`Exam.kokugo`とは別の場所を指す`ayumi.kokugo`というインスタンス変数が作られるのですね。

では続けて、`Exam.kokugo`に`-1`を代入してみましょう。
これまでの話を踏まえると、何を出力されるのかイメージしやすいかと思います。
（読みづらいのでIDの出力はここでやめます）

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

ayumi.kokugo = 90
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> 0 90 0
Exam.kokugo = -1 # +++
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> -1 90 -1
```

`Exam.kokugo = -1`を追加しました。`Exam.kokugo`が`-1`を出力するのは当然として、`genta.kokugo`は同じ場所を指しているので、`genta.kokugo`も`-1`を出力します。
メモリイメージを見てみましょう。

![memory1_5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/ec6906fd-285c-9b36-b7bf-d20dd5c09da8.png)

では元太くんの点数も設定してみます。
これは歩美ちゃんの時と同じです。

```python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

ayumi.kokugo = 90
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> 0 90 0
Exam.kokugo = -1
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> -1 90 -1
genta.kokugo = 30
print(Exam.kokugo, ayumi.kokugo, genta.kokugo) # -> -1, 90 30
```

メモリイメージを見てみます。

![memory1_6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/94019dde-5b9a-94fb-e66d-5af949031e44.png)

新しくインスタンス変数のメモリが確保され、`genta.kokugo`はそちらを指すようになりましたね。

## Class直下の記述はクラス変数
改めて何が分かりにくいのかを考えてみます。
前提として、Pythonでは`class`直下に記述した変数はクラス変数となります。

```python
class Exam:
    kokugo = 0
```
kokugoはクラス変数です。C++やJavaでは、いわゆるクラス変数を定義したい場合`static`をつけますが、Pythonでは付けません。class直下の変数定義は全てクラス変数です。
これについては「Pythonではそうなんだ」と受け取るだけかと思います。

では何が分かりにくくさせるのかというと、まずPythonではインスタンス変数参照の記述でクラス変数を参照できます。
```python
class Exam:
    kokugo = 0

ayumi = Exam()
print(ayumi.kokugo) # -> 0 インスタンス変数風だが、クラス変数への参照
```

ただ、これはJavaやRubyでも同じです。
これに加えPythonでは、**代入文でインスタンス変数が生成される言語仕様**というのがあります。したがって`ayumi.kokugo = 90`をすると、さっきまで`ayumi.kokugo`はクラス変数を指していたのに以後`ayumi.kokugo`はインスタンス変数を指すようになります。この挙動が混乱を招いてしまっているのだと思います。

改めて、この記事の最初のコードを見てみます。
`python:test.py
class Exam:
    kokugo = 0

ayumi = Exam()
genta = Exam()

print(ayumi.kokugo, genta.kokugo) # クラス変数参照　クラス変数参照
ayumi.kokugo = 90
print(ayumi.kokugo, genta.kokugo) # インスタンス変数参照　クラス変数参照
genta.kokugo = 30
print(ayumi.kokugo, genta.kokugo) # インスタンス変数参照 インスタンス変数参照
`
問題ないように見えていた挙動でしたが、代入文によりクラス変数からインスタンス変数へ参照が変わっています。多くの場合、これは意図したものではないと思います。


## インスタンス変数は`__init__`へ
では例題の歩美ちゃん元太くんの小テスト結果のようなことをしたければどうすればいいのでしょうか。
繰り返しになりますが、代入文でインスタンス変数が生成されます。
インスタンス変数を定義したい場合は、インスタンス生成時に実行される`__init__`で代入文を記述しましょう。

```python
class Exam:
    def __init__(self):
        self.kokugo = 0

ayumi = Exam()
print(ayumi.kokugo) # -> 0 ayumi.kokugoはインスタンス変数
```

パイソニスタには見慣れた記述ですが、改めて見るとなるほどと思います。
`Exam()`でインスタンス生成時に`__init__`が実行されるので、`self.kokugo = 0`でインスタンス変数`kokugo`が生成され、これを`ayumi`に代入するので`ayumi.kokugo`が`ayumi`インスタンスの`kokugo`変数になるわけですね。

下記のように`dataclass`を用いても良いです。
ここでは細かい内容は省きますが、`dataclass`は上記コードの`__init__`を生成してくれるシンタックスシュガーです。

```python
from dataclasses import dataclass

@dataclass
class Exam:
    kokugo = 0

ayumi = Exam()
print(ayumi.kokugo) # -> 0 ayumi.kokugoはインスタンス変数
```
より直感に近い記述になりますね。

## まとめ
Pythonのクラス変数参照について、分かりにくさの原因となる挙動を具体例を通して確認し、なぜ分かりにくいのかのポイントをまとめました。
Pythonのクラス変数とインスタンス変数について理解が深まれば幸いです。


## 補足） サンプルコードにおけるID確認
Pythonでは数値リテラルは最適化され、値が同じだと同じIDを返します。
```python
a = 0
b = 0
print(id(a), id(b)) # -> 4340976048 4340976048

a = 999999
b = 999999
print(id(a), id(b)) # -> 4487693488 4487693488
```
なので、サンプルコードで`Exam.kokugo`, `ayumi.kokugo`, `genta.kokugo`のIDを確認してますが、同じ値では同じIDになるため、インスタンス変数とクラス変数の参照の切り替わりを確認する方法としては実は不適です。

分かりやすさのため記事上では数値リテラルにしてますが、ご自分で正しく参照の変更を確認するには`kokugo`をクラスでWrapしてあげてください。
```python
class Score:
    def __init__(self, num: int):
        self.num = num

class Exam:
    kokugo = Score(0)

ayumi = Exam()
genta = Exam()
print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
# -> 4454425072 4454425072 4454425072
ayumi.kokugo = Score(90)
print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
# -> 4454425072 4454639264 4454425072
genta.kokugo = Score(90)
print(id(Exam.kokugo), id(ayumi.kokugo), id(genta.kokugo))
# -> 4454425072 4454639264 4454639504
```

## 参考
- Python3 公式ドキュメント
   - 9.3.5. クラスとインスタンス変数

https://docs.python.org/ja/3/tutorial/classes.html#class-and-instance-variables