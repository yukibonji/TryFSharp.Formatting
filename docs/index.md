# FSharp.Formattingを試す

[FSharp.Formatting][link01] はTomas Petricek氏らによって開発されている
F#製のドキュメント生成ツールです。

このツールを使うと、既存のコードにコメントを追加するだけで
コードを含んだ形のドキュメントを生成することができます。
[doxygen][link02] や [Sandcastle Help File Builder][link03]といった類のツールと同じ役割を果たすものだといえます。

F#界隈のコミュニティでは [FSharp.Formatting][link01] と
[FAKE][link04] の組み合わせでよく使用されている感じがしますが、
今回は [FAKE][link04] ではなくMSBuildでドキュメントを生成させてみようと思っています。

ドキュメント生成には最低限以下のファイル/ディレクトリが関連します：

* .nuget/
 * NuGet.Config
 * NuGet.exe
 * NuGet.targets
* contents/
* docs/
* templates/
 * template.cshtml
* build.proj
* generate.fsx
* packages.config

## ファイルとディレクトリの概要

### .nuget

.nugetディレクトリはVisual Studio上でソリューションを設定して、
NuGetパッケージの復元を有効化した際に生成されるものと全く同じものです。

### contents

contentsディレクトリにはドキュメントに付随する追加ファイルを置く予定です。

### docs

docsディレクトリにはドキュメント生成の対象になるファイル( `*.md`, `*.fsx` )を置きます。
ちなみに拡張子 `.fs` のファイルは対象に **ならない** ので注意が必要です。

### templates

templatesディレクトリにはすべてのドキュメントで共通して使用されるテンプレートファイルを置きます。
テンプレートファイルはRazor記法で記述します。

### build.proj

MSBuildのターゲットとなるXMLファイルです。

### generate.fsx

[FSharp.Formatting][link01] を使ってドキュメントを生成するコードが書かれたF# スクリプトファイルです。
プロジェクト毎、言語毎に用意されることになります。

### packages.config

NuGetを使って[FSharp.Formatting][link01] とその関連パッケージを取得/復元するためのファイルです。

## その他のディレクトリ

### output

生成されたドキュメントファイルが置かれます。

### packages

NuGetで取得した[FSharp.Formatting][link01] および関連パッケージが配置されます。

## build.projの作成

まずMSBuildのターゲットファイルを作成します。
このファイルがサポートするビルドターゲットは以下の通りとします：

* Build (デフォルト)
* Clean
* Rebuild

その他にも補助的なターゲットを用意しますが、基本的には上の3つだけが使われることになる予定です。

ファイルの内容は [GitHub上][link05] で参照してください。

MSBuildはPropertyGroup要素やItemGroup要素の下に任意の名前のプロパティやアイテムを定義できるので、
プロジェクトのディレクトリ構成に沿って設定することになります。

`$(MSBuildThisFileDirectory)` はこのファイルが置かれたディレクトリを指すプロパティです。
その他の既知のプロパティについては [MSDN][link06] を参照してください。

デフォルトのビルドターゲットは `<Project>`ノードの `DefaultTargets` 属性で指定しています。
ターゲットを複数指定する場合は `;` でターゲット名を区切ります。

`Build` で行いたいことは要約すると以下の通りです：

* NuGetを使って関連パッケージを取得する
* ビルド用のfsxファイルを `fsi.exe` で実行してドキュメントを生成する

また `Clean` では単に生成されたドキュメントをフォルダごと削除、
`Rebuild` では `Clean` した後に `Build` するだけです。

### NuGetでのパッケージ取得

    nuget.exe restore packages.config -PackagesDirectory packages

というコマンドを実行して、[packages.config][link07] に書かれたパッケージを `packages` ディレクトリに配置するだけです。
[packages.configの中][link07] では今のところ [FSharp.Formatting][link01] とその依存パッケージだけを列挙してあります。

### ドキュメント生成スクリプトの実行

    fsi.exe --exec generate.fsx

実際のところはこれだけです。
スクリプト `generate.fsx` の中で定数定義で条件分岐させている都合上、実際には以下のように ``--define`` も指定します：

    fsi.exe --exec --define:$(Configuration) generate.fsx

`$(Configuration)` はファイルの上の方で以下のように定義しています：

    <Configuration Condition=" '$(Configuration)' == '' ">RELEASE</Configuration>

MSBuildでは実行時に任意のプロパティを指定できるわけなので、
ここでは `Configuration` にまだ値が設定されていなければ値を `RELEASE` に設定しています。

なので、この定義は以下のようにしてMSBuildが実行されるとスキップされます：

    MSBuild.exe build.proj /p:Configuration=DEBUG

## ドキュメント生成スクリプト

[`generate.fsx`][link08] の内容は [FSharp.Formatting][link01] のREADMEでも言及されていますが、
[FSharp.ProjectScaffoldのgenerate.fsx][link09] とほぼ同じです。

処理の中心は [Literate.ProcessDirectoryを呼び出す][link10] ところです。
この関数を呼び出すときに、引数 `replacements` で指定した値ペアがテンプレート内から参照できるようになっているようです。

## テンプレートファイル

[テンプレートファイル][link11] で最低限必要なのは [`RenderBody`の呼び出し][link11] です。
たぶん。

また、先ほどビルドスクリプト中で `replacements` に指定した値は `@Properties['名前']` の形式で参照できます。

## ビルドの実行

Visual Studio開発者用コンソールを起動した後、作業ディレクトリで以下のコマンドを実行するだけです：

    MSBuild.exe build.proj

ドキュメントを強制的に再作成したい場合には `Rebuild` ターゲットを指定します：

    MSBuild.exe build.proj /t:Rebuild

生成されたドキュメントを削除する場合は `Clean` ターゲットを指定します：

    MSBuild.exe build.proj /t:Clean

生成されたドキュメントをローカルでチェックしたい場合には `Configuration` の値を変更します：

    MSBuild.exe build.proj /p:Configuration=DEBUG

> 実際は `RELEASE` 以外の値であれば何でもいいです。

## 続く？

fsx内で特別な形式のコメント( `omit` とか `hide` とか `include` とか `define` とか)を埋め込むと
コードを隠したり、後方にあるコードをドキュメントの前のほうで表示させたり出来るんですが、
その話については後で更新したりしなかったりします。

[link01]: http://tpetricek.github.io/FSharp.Formatting/ "FSharp.Formatting"
[link02]: http://www.doxygen.org/index.html "doxygen"
[link03]: http://shfb.codeplex.com/ "Sandcastle Help File Builder"
[link04]: http://fsharp.github.io/FAKE/ "FAKE"
[link05]: https://github.com/yukitos/TryFSharp.Formatting/blob/master/build.proj "build.proj"
[link06]: http://msdn.microsoft.com/ja-jp/library/ms164309.aspx "MSBuildの予約済みおよび既知のプロパティ"
[link07]: https://github.com/yukitos/TryFSharp.Formatting/blob/master/packages.config "packages.config"
[link08]: https://github.com/yukitos/TryFSharp.Formatting/blob/master/generate.fsx "generate.fsx"
[link09]: https://github.com/fsprojects/FSharp.ProjectScaffold/blob/master/docs/tools/generate.fsx "FSharp.ProjectScaffoldのgenerate.fsx"
[link10]: https://github.com/yukitos/TryFSharp.Formatting/blob/master/generate.fsx#L46-L48 "Literate.ProcessDirectory"
[link11]: https://github.com/yukitos/TryFSharp.Formatting/blob/master/templates/template.cshtml#L24 "RenderBody()"
