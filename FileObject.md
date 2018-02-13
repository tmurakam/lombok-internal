FileObject
==========

Lombok は class file の書き換え(AST変換)を行うようになっているが、
これを実行するため javax.tools.JavaFileObject を wrap してクラスファイル
書き込み動作に割り込むようになっている。

実装は lombok.java.apt.LombokFileObjects にある。
クラス図は uml/LombokClassDiagram.asta に置いた。

JDK内のクラス構造
=================

JavaFileManager と JavaFileObject
----------------------------------

javax.tools に JavaFileManager, JavaFileObject という interface がある。
(この２つは公開クラス)

JavaFileObject が実際にファイル操作をするための interface。
JavaFileManager は JavaFileObject を生成したりするための interface である。

javac
-----

javac の実装は com.sun.tools.javac パッケージにある。こちらは JDK の内部
クラスである。

JavaFileManager の実装は BaseFileManager。
JavaFileObject の実装は BaseFileObject または PathFileObject である。

Lombokの実装
============

LombokFileObject
-----------------

JavaFileObject を wrap するための interface が LombokFileObject である。
実際にファイル操作に割り込むのは、これを実装した InterceptingJavaFileObject である。

javac の JavaFileObject 実装からこちらに処理を delegate するために、
別に Wrapper クラスが用意されている。これには以下の3種類がある。

* Javac6BaseFileObjectWrapper
* Javac7BaseFileObjectWrapper
* Javac9BaseFileObjectWrapper

Javac のバージョンごとに実装が異なっている。6, 7 については BaseFileObject を
(ただし、6 と 7 はパッケージが違うので実質別クラス)、
9 については PathFileObject を継承している。
この3クラスの実装は単純で、単に LombokFileObject (実態は InterceptingJavaFileObject)
に全メソッドを delegate するだけである。

なぜ Wrapper クラスが必要かというと、Javac 側で JavaFileObject を上記
BaseFileObject や PathFileObject に cast しており、
型をあわせないとエラーになってしまうからである。

LombokFileObjects
-----------------

ここからやっと本番。このクラスで JavaFileManager の wrap などの処理を行う。

### LombokFileObjects.Compiler interface

JavaFileObject を wrap して LombokFileObject を返すためのクラス。
wrap() メソッドでこれを実行する。

これも Javac 6, 7, 9 ごとに異なるクラス実装となっている。
それぞれ上述の JavacXBaseFileObjectWrapper を返す実装になっている。

### Java9Compiler

Javac9 に対応する Compiler 実装は Java9Compiler であるが、これだけ処理が複雑である。
これは、ベースクラスである PathFileManager が BaseFileManager を要求するためである。

生成時に渡されるクラスは JavaFileManager なので、BaseFileManager に置換しなければ
ならない。JavaFileManager の実態が BaseFileManager ならそのままキャストすれば良いが、
IntelliJ などから呼び出される場合は全く違うクラスなので、キャストができない。
このため、BaseFileManager を継承した FileManagerWrapper クラスを新しく用意し、
これを使ってさらに delegate を行うようにしている。
(java9 対応で村上が追加したコードである)

### LombokFileObjects.getCompiler()

適切な Compiler を取得するためのメソッドがこれである。

実装としては、引数に渡された JavaFileManager の実装クラス名などを見て振り分けを
行うようになっている。

ここに渡されてくる実装クラス名は、Netbeans, Error Prone Compiler などで異なる
場合があるので、KNOWN_JAVA9_FILE_MANAGERS に既知のクラス名が格納されている。

将来、JDK の実装が変わったらここの実装も変えていく必要がある。









