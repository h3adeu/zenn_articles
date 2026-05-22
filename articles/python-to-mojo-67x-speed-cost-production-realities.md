---
title: "PythonをMojoにリプレースして得た「67倍速」の代償：本番投入で直面した現実"
emoji: "📘"
type: "tech"
topics: ["Mojo", "システムプログラミング", "ハードウェア最適化", "Python移行", "メモリ管理"]
published: true
---

今回は、**I Replaced Python with Mojo and Got a 67x Speedup. Then Production Happened.** という記事を参考に、新しいプログラミング言語を実際のシステムに導入した際に起こりうる「ベンチマークの先にある現実」についてまとめてみます。

性能改善のニュースでよく目にする「◯◯倍の高速化」という言葉。エンジニアなら誰しも心が躍りますが、その裏側には意外な落とし穴が隠れていることもあるようです。

mojoネタと言う事で共有です。😅

---

## なぜMojoを選んだのか

もともとのシステムは、Pythonで構築されたリアルタイム推論のデータパイプラインでした。NumPyやカスタムトークナイザーを使い、1時間あたり約80万件のレコードを処理していたといいます。

決して遅いわけではありません。しかし、システムの規模が大きくなるにつれて、わずかな遅延がインフラコストを押し上げる要因になっていました。「ハードウェアの問題だ」という声もありましたが、ボトルネックはソフトウェア側、特にPythonの処理能力にあると考えたわけです。

そこで注目したのが、Chris Lattner氏が発表した新しい言語「Mojo」でした。

### 移行のステップ

いきなりすべてを書き換えるのではなく、最も負荷が高かった「トークナイザー」の部分だけをMojoで書き直すという戦略をとりました。

```mermaid
graph TD
    A[入力データ] --> B[Python製パイプライン]
    B --> C{トークナイザー}
    C -- 以前 -- > D[Python実装]
    C -- 変更後 -- > E[Mojo実装]
    D --> F[推論処理]
    E --> F[推論処理]
    F --> G[結果出力]
    
    style E fill:#f96,stroke:#333,stroke-width:2px
```

## ベンチマークの結果と「見出し」のカラクリ

書き換えの結果、トークナイザー単体の性能は**67倍**という数字を叩き出しました。これだけ聞くと魔法のように思えますが、システム全体で見ると話は少し変わってきます。

| 評価対象 | Python (従来) | Mojo (リプレース後) | 改善率 |
| :--- | :---: | :---: | :---: |
| トークナイザー単体 | 1x | 67x | 6,700% |
| パイプライン全体 (End-to-End) | 1x | 18x | 1,800% |

全体で18倍というのも十分すぎるほどの結果ですが、記事のタイトルにある「67倍」というのは、あくまで最も効果が出た部分的なモジュールの話です。実際のシステムでは、入出力や他の処理が介在するため、一部がどれだけ速くなっても全体が同じ割合で速くなるわけではない、という点には注意が必要です。

## 本番環境で突きつけられた現実

「これならいける」と確信して本番に投入した3週間後、チームは午前2時にデバッグ作業に追われることになります。ベンチマークの記事では語られない、3つの大きな壁にぶつかったのです。

### 1. 「未完成」という壁

Mojoは所有権モデルやコンパイル時のメタプログラミングなど、Pythonの弱点を補う素晴らしい設計を持っています。しかし、まだ言語として成熟しきっていない部分があります。

たとえば、以下のようなエラーに遭遇したとします。

```text
error: 'StringRef' is not implicitly constructible from value of type 'String'
```

Pythonに慣れたエンジニアからすれば、「文字列を扱いたいだけなのに、なぜ型変換でこんなに苦労するのか」と困惑してしまうでしょう。ドキュメントにないエラーの解決に数時間を費やすことも珍しくありません。

### 2. Pythonとの相互運用性は「魔法」ではない

Mojoの強みは、既存のPythonライブラリをそのまま呼び出せることです。

```python
from python import Python

def call_pandas():
    pd = Python.import_module("pandas")
    df = pd.read_csv("data.csv")
    return df
```

このように、PandasなどをMojoから使うこと自体は簡単です。しかし、Mojo独自の構造体（struct）をPython側に渡したり、逆にPythonのオブジェクトをMojoの厳格な型として扱おうとすると、データの「翻訳」コストが発生します。この橋渡しが複雑になると、せっかくの高速化が相殺されてしまうこともあるのです。

### 3. デバッグの難易度

Pythonであれば、エラーが起きてもスタックトレースを読めば大抵のことはわかります。しかし、Mojoのようにハードウェアに近い最適化を行う言語では、メモリ関連の深い問題が発生した際、Stack Overflowにも答えが載っていないような未知の領域に踏み込んでしまうことがあります。

## 結局、何が重要だったのか

このリプレースを通じて得られた教訓は、**「言語の速度よりも、アーキテクチャのほうが重要である」**という点です。

新しい言語を導入して局所的なスピードを追い求めるよりも、まずは今のシステムで無駄なデータの往復はないか、メモリ管理は適切か、といった基本的な設計を見直すほうが、結果としてメンテナンス性の高いシステムにつながります。

Mojoは将来性の高い、非常に興味深い言語であることは間違いありません。しかし、本番環境に導入するのであれば、その「若さ」ゆえの苦労をチーム全体で引き受ける覚悟が必要だということですね。

---

## 参照記事

- [I Replaced Python with Mojo and Got a 67x Speedup. Then Production Happened.](https://medium.com/@suryawanshiaditya159/i-replaced-python-with-mojo-and-got-a-67x-speedup-then-production-happened-22b0d3456350)
- [I Rewrote jQuery in Assembly for Artistic Reasons](https://medium.com/@sohail_saifi/i-rewrote-jquery-in-assembly-for-artistic-reasons-9df842969e1f)
- [How I Made Postgres Fly on 2 Cores and 4GB RAM — 1.2 Billion Rows in 0.7 Seconds](https://medium.com/@kakamber07/how-i-made-postgres-fly-on-2-cores-and-4gb-ram-1-2-billion-rows-in-0-7-seconds-1a0e01c49571)
- [The 9 Hidden C Features Nobody Told You About (But Every Senior Dev Uses)](https://medium.com/@mahadrajpoot911/the-9-hidden-c-features-nobody-told-you-about-but-every-senior-dev-uses-e699a05e0d7e)
- [7 Underused Rust Features Every Senior Developer Knows](https://medium.com/@Krishnajlathi/7-underused-rust-features-every-senior-developer-knows-7b8bb8da684f)
- [Our Docker Containers Kept Dying. It Took 3 Days to Find Why.](https://medium.com/@codexlab/our-docker-containers-kept-dying-it-took-3-days-to-find-why-488992450546)

---

詳しくは[こちら](https://microarchitectures.jp/blog/python-to-mojo-67x-speed-cost-production-reality/)をご覧ください。