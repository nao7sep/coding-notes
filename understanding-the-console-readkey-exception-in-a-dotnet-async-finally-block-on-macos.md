<!--
    作成日: 2025年9月16日
    更新日: 2025年9月16日
    英語タイトル: Understanding the Console.ReadKey Exception in a .NET async finally block on macOS
-->

# macOSの.NET非同期finallyブロックにおけるConsole.ReadKeyの例外についての考察

「次に進むには○○を押してください」のところでは、Console.ReadLine を使う。

async Task Main メソッドの finally ブロック内の次のコードが、
Windows では動き、Mac では InvalidOperationException を投げた。

```
Console.Write("何かキーを押して終了します...");
Console.ReadKey(intercept: true);
Console.WriteLine();
```

.NET のコードには、次のようにある。
https://source.dot.net/#System.Console/System/ConsolePal.Unix.cs,bc570a7f9c052c81

```
public static ConsoleKeyInfo ReadKey(bool intercept)
{
    if (Console.IsInputRedirected)
    {
        // We could leverage Console.Read() here however
        // windows fails when stdin is redirected.
        throw new InvalidOperationException(SR.InvalidOperation_ConsoleReadKeyOnFile);
    }

    ConsoleKeyInfo keyInfo = StdInReader.ReadKey(intercept);
    return keyInfo;
}
```

ReadKey は、ReadLine や Read とは仕組みが異なるらしい。
前者は、raw モードで、キーが押された瞬間にその情報がバッファを通らず直接プログラムに送られる。
後者は、cooked/canonical mode（行入力モード）で、Enter が押されるまで入力がバッファリングされる。

後者は、入力が物理キーボードであることが前提。
コマンドラインの < や | など、物理キーボードからの直接の信号でないときには、InvalidOperationException が投げられる。
これは、たとえば *** が代わりに表示されるパスワード入力のところに外部プログラムから無数の文字列が送り込まれてセキュリティーが破られるなどを防ぐためだろう。
物理キーからの直接の信号でないと入力がアプリに届かないようにするのは、デメリットはさほどなく、センシティブな入力を扱うところでは好都合だ。

一方、その特殊な仕様により、物理キーボードからデータを直接的かつ瞬間的にもらいたいわけでないところでは使いにくい。
Windows でこれまで問題がなかったが、Mac で例外が飛んだので、今後は1文字の入力（y/n など）でも ReadLine を第一に考える。

Mac で IsInputRedirected が true になった原因は、今も分からない。
AI に聞けば、「finally も Main の一部であり、Main のあとに実行されるものでない」
「だから、Mac では Main が終わればコンソールが切断される仕様になっているとか、
それで finally では ReadKey が動かないとかを原因と考えることはできない」を旨とする回答を得た。

そのうち、以下を言いだした。

> 閉店作業中の店員に「すみません、今から店内のレイアウトを変えてもいいですか？」と頼むのと、「忘れ物がないか待っていてもいいですか？」と頼むのでは、対応のしやすさが全く違いますよね。
>
> finallyブロックが実行される「閉店間際」のタイミングでは、ランタイムやOSはすでにシャッターを下ろし始めています。そのような状態で「コンソールの設定を変更してくれ」という複雑な要求 (ReadKey) を送ると、OSやランタイムから「申し訳ありません。もう店じまいなので、そのような要求にはお応えできません」と断られてしまう可能性が高まるのです。これが InvalidOperationException の正体です。
>
> 一方で、ReadLineはただ静かに入力を待つだけの受動的な操作です。シャッターが完全に閉まる前であれば、この単純な要求は通りやすいのです。

「finally は Main の一部であるが、finally では IsInputRedirected が true になる」ということに強引に説明をつければ、このくらいしか選択肢がない。
「不可能を消していけば、最後に残ったものが」的な理屈だが、仕方ない。

とりあえず、「Mac の finally では ReadKey が落ちることがある」
「低レベルな処理が必要でないなら、そもそも ReadLine でいい」を結論とする。
