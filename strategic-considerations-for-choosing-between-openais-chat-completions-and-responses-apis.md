<!--
    作成日: 2025年9月19日
    更新日: 2025年9月19日
    英語タイトル: Strategic Considerations for Choosing Between OpenAI's Chat Completions and Responses APIs
-->

# OpenAIのChat Completions APIとResponses APIの戦略的な使い分け

少し前から OpenAI は、Chat Completions から Responses API への移行を勧めてきている。
Responses API に「API」が入っているのは、そのサブセットとして、
Responses の CRUD、Conversations の CRUD、おまけに Streaming events とやらもあるからか。

Chat Completions に比べての Responses API の違いは、ザッと見た限りでは以下の認識。

1. 会話ログの管理が API 頼みになるのは長短あり
2. 機能がちょっと多いが、自分がすぐに使うものは Chat Completions にも揃っている
3. パラメーターが多少整理されていて、「なんでここにこれが」的なのが少なく、コーディングが楽そう

まず3だが、元々マルチモーダルでなかったものをあとからそうした Chat Completions だと、
model と同じ階層に audio や web_search_options があって、
「揃ってもいない要素が、そのときに思い付いたところに追加された」感があって、
ドメインモデルや DTO をつくるときに、どことどこを共通化してよいのか分かりにくい。
Responses API でそこが改善されたようなのは、自分としては嬉しい。

ただ、1が地味に困る。

「多数のメッセージの中から、次の回に必要なものだけを選択して送る」とか、
「永遠に続くチャットルームにおいて、今の話題と（コサイン類似度的に）関係のある直近のメッセージのみ送る」とかを実装したければ、
conversation 中の item の更新や抜き差しができないようである Responses API では、conversation づくりからになる。

具体的には、https://api.openai.com/v1/conversations で conversation をつくり、その id を取得し、
https://api.openai.com/v1/conversations/{conversation_id}/items で「1回20個まで」という制限において items を追加し、
同じ id を指定して https://api.openai.com/v1/responses に最後の user message（に相当するもの）を送り、
その回答は自動的に conversation に入ってくれるので、それがイマイチでチャットログとして残さないなら、
https://api.openai.com/v1/responses/{response_id} によりリモートのコンテキストから消す必要がある。

ローカルで会話ログを残すにおいて、ベクトルデータをつくっておき、多言語だから英語版をつくっておき、
読みやすくまとめ直したものを各言語でつくっておき、コンテキスト圧縮用のサマリーをつくっておくといったことをして、
ユーザーにはその人の言語で表示し、AI には次の回に必要なものだけを選んで送るようなシステムなら、
何を送っても stateless 的に処理して答えだけ返してくれる Chat Completions の柔軟性がありがたい。

もっとも、上記のようなことをしたければ、Responses API では、
role も何もつかない input に「以下が今回お渡しする会話ログです」的にひとまとめにして入れてもよいのかもしれない。
Whether to store the generated model response for later retrieval via API とのことである store を false にして、
それでも conversation がつくられるとすれば、そこを消す API もあるので、消してしまうことで、stateless 的なことは、こちらでもできる。
「会話データを送るなら、Chat Completions では messages に、Responses API では Conversations に、構造的に入っていなければならない」というのが、
ただの先入観というのか、律儀にやりすぎなのかもしれないとは思う。

会話ログが OpenAI 側にあることには、ユーザーにとっても明確なメリットがある。
自分が OpenAI 側なら、「こちらにある分を、事前に AI にとって都合のいい形にしておく」という処理を行う。
そこに id だけ指定してもらって次のリクエストが届く方が、処理を最適化しやすい。
もちろん、messages が全て届くのでも、「さっきはこうだった」というのがあって、そこまでの中間データがキャッシュされているなら、
メッセージのうち、どこからどこまでが一致しているかを調べ、「使えるなら中間データを使う」ということが可能。
ただ、それを「キャッシュヒット」と言い、その「率」を扱っていることから、
送る側のコーディングが甘いと、100％理想的な中間データ再利用は難しいのだろう。
つまりは、中間データはあっても、そこに確実にヒットさせることはできなくての再計算のもどかしさがあるのだろう。
OpenAI 側で conversations を管理するなら、CRUD のタイミングで確定的な中間データをつくれる。
そういった理由によるのか、Migrate to the Responses API のページには、
Lower costs: Results in lower costs due to improved cache utilization
(40% to 80% improvement when compared to Chat Completions in internal tests) とある。

Chat Completions に依存するアプリがすでに多数あるだろうし、「OpenAI 方式」としてデファクトっぽくなっているものでもある。
OpenAI が Chat Completions を完全にオフにすれば、API base の変更だけで移行しうるほかの API に客が流れる。
だから Chat Completions は、すでに提供されている機能で不足がない案件においては、今後も使ってよさそう。

自分なりの基準としては次のように考えておく。

- AI 機能が Chat Completions のもので足りて、messages をいじるかもしれず、stateless でやりたい → Chat Completions
- それらに該当しなかったり、派生開発が長く続きそうだったり（新しいモデルが出たら使いたい）、コストを最適化したかったり → Responses API

https://platform.openai.com/docs/api-reference/responses
https://platform.openai.com/docs/api-reference/conversations
https://platform.openai.com/docs/api-reference/chat
https://platform.openai.com/docs/guides/migrate-to-responses
