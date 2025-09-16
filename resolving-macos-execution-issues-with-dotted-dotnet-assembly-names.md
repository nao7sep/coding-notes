<!--
    作成日: 2025年9月16日
    更新日: 2025年9月16日
    英語タイトル: Resolving macOS Execution Issues with Dotted .NET Assembly Names
-->

# ドットを含む.NETアセンブリ名のMacにおける実行問題の解決

.csproj ファイルで AssemblyName を指定しない。

AssemblyName は、指定があれば、バイナリーのファイル名に使われる。
なければ、プロジェクト名が使われる。

Kotoban.DataManager を Windows でビルドすれば、Kotoban.DataManager.exe になる。
当然動く。

Mac だと、Kotoban.DataManager となり、これが動かなかった。
ファイルは生成されたが、実行ファイルのアイコン（黒いコンソール）でなく、ダブルクリックで起動しなかった。
file コマンドで見たところ、Mach-O 64-bit executable arm64 と表示された。
ということは、実行ファイルとして正しく出力されている。
ドットを消したところ、白色だったアイコンが上記のようになり、ダブルクリックで起動できた。

AI に聞けば、「Mac の実行ファイルにドットが入ってはいけないとの情報はない」と。
しかし、Windows と異なり、.exe のような目立つ拡張子が「おれは実行ファイルかもしれないから、ヘッダーを見てケロ～」と言ってこない。
ドットが入っていれば、「.DataManager」が拡張子らしく見えてしまう状況において、Finder が中身を見てくれないのかもしれない。
chmod をやっても効果なし。
そもそも chmod なしでも、ドットさえ消せば実行ファイルになっていた。

AssemblyName として Kotoban-DataManager を指定すれば、
ダブルクリック可能な実行ファイルが黒いアイコンでつくられた。
Google のガイドラインにもあるように、ファイル区切りにはハイフンが適する。
https://developers.google.com/search/docs/crawling-indexing/url-structure?hl=ja
今後、アセンブリー名のドットを全てハイフンにすることを検討した。

しかし、Microsoft.Extensions.Logging.dll や Kotoban.DataManager.deps.json といったファイルがそこら中にある。
.NET 界隈では、「所属」や「構造」には名前空間的にドットを使うのが、長いものに巻かれにいく無難な設計だ。
そこで自分だけ Microsoft-Extensions-Logging.dll 的になる応急処置をとってはいけない。

Mac でも、Kotoban.DataManager からドットのみ取り除けば、ほかのファイルがそのままでも動いた。
昔の App.config のように実行ファイル名と紐づけされるものが今のところ見つかっていない。
今後見つかれば、スクリプトファイルを書いて dotnet コマンドを使っての起動にしても、手間は小さい。

よって、AssemblyName を指定せず、アセンブリー名の「所属」や「構造」には今後もドットを使い、
Mac でバイナリーを配置するときにはファイル名のドットをハイフンに置き換え、
それだけでは動かなくなれば、ドットの置換をやめてスクリプトで起動する。
