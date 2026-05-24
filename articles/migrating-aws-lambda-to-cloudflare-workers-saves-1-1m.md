---
title: "AWS Lambda から Cloudflare Workers への移行で年間 110 万ドルを削減した話：コスト構造の違いを整理する"
emoji: "📘"
type: "tech"
topics: ["Cloudflare_Workers", "AWS_Lambda", "サーバーレス", "コスト最適化", "システム設計"]
published: true
---

今回は、サーバーレス環境の運用コストについて書かれた **He Was a Lambda Fanboy… Until the Math Showed Cloudflare Workers Saved Us $1.1M a Year** という記事を読み、インフラ構成の選択がビジネスに与えるインパクトが予想以上に大きかったので、その内容を自分なりに整理してみたいと思います。

サーバーレスアーキテクチャを検討する際、まず選択肢に挙がるのは AWS Lambda かもしれません。しかし、サービスの規模が拡大するにつれて、当初は気にならなかった「隠れたコスト」が大きな負担になることがあります。

Serverless 構成はあんまりやらないんですが、Lambda より Cloudflare Workers 安いですからねぇ。

---

### Lambda ファンが直面した「コストの壁」

元記事の筆者は、もともと AWS Lambda の熱烈な支持者でした。AWS のエコシステムとの親和性や、スケーラビリティの高さは確かに魅力的です。しかし、大量のリクエストを処理するシステムにおいて、以下の 3 点が課題として浮き彫りになりました。

1.  **API Gateway の高額な料金**: Lambda 自体よりも、その手前に置く API Gateway のリクエスト単価が無視できないレベルに達すること。
2.  **実行時間の課金単位**: メモリ使用量と実行時間の積（GB-秒）で課金されますが、リクエスト数が増えるとこれが雪だるま式に膨らみます。
3.  **コールドスタートの遅延**: 実行環境を起動する際のオーバーヘッドを解消するために「プロビジョニングされた同時実行」などを使うと、さらにコストが加算されます。

これらを「大きなトラックを毎回エンジンから始動して荷物を運ぶ」ようなイメージだとすると、リクエストごとにトラックを呼び出すのは、効率の面で少し課題があるかもしれません。

### Cloudflare Workers が低コストな理由

対して Cloudflare Workers は、Google Chrome でも使われている「V8 Isolates」という技術を活用しています。これは、OS レベルの仮想化（コンテナなど）ではなく、ブラウザのタブを分けるような軽量な仕組みで実行環境を分離するものです。

以下の図は、両者の実行環境の立ち上がりの違いを簡略化したものです。

```mermaid
flowchart TD
    subgraph AWS_Lambda[AWS Lambda / Container Base]
        A[リクエスト] --> B[Firecracker VMM 起動]
        B --> C[ランタイムロード]
        C --> D[コード実行]
    end

    subgraph CF_Workers[Cloudflare Workers / V8 Isolates]
        E[リクエスト] --> F[既存プロセス内の Isolate 作成]
        F --> G[コード実行]
    end
```

この「軽量さ」が、結果としてコストの差に直結しています。Cloudflare Workers はコールドスタートがほぼゼロであり、かつリクエストあたりの単価が AWS Lambda よりも低く設定されている傾向にあります。

### 具体的な比較表

実際にどのような違いがあるのか、主要な項目を表にまとめてみました。

| 項目 | AWS Lambda (+ API Gateway) | Cloudflare Workers |
| :--- | :--- | :--- |
| **主な課金要素** | リクエスト数 + GB-秒 + API Gateway | リクエスト数 + CPU 時間 |
| **起動速度** | 数百ms 〜 数秒（コールドスタート時） | 5ms 以下 |
| **実行場所** | 特定のリージョン | 世界各地のエッジロケーション |
| **スケーリング** | コンテナの増減による制御 | V8 Isolates による超軽量並列処理 |
| **コスト（大規模時）** | 比較的高価になりやすい | 非常に抑えやすい |

たとえば、単純な API 処理を行う場合、Lambda では API Gateway の通信費が支配的になることがありますが、Workers ではそれらがパッケージ化されているため、構成がシンプルになります。

### 110 万ドルの節約を実現した計算

元記事では、詳細な数学的アプローチによってコストが算出されました。特に、エッジ（利用者に近い場所）で処理を完結させることで、オリジンサーバーへのトラフィックを減らし、データ転送量を抑えられた点も大きいようです。

もし、Node.js のような環境で動作するコードであれば、Cloudflare Workers への移行はそれほど難しくありません。以下は、簡単なリクエスト処理の例です。

```javascript
// Cloudflare Workers の基本的なハンドラ
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // エッジで直接レスポンスを返すことで、バックエンドの負荷を軽減
    if (url.pathname === '/api/health') {
      return new Response(JSON.stringify({ status: 'ok' }), {
        headers: { 'content-type': 'application/json' },
      });
    }

    // 必要に応じてメインの処理へ
    return new Response("Hello from the Edge!");
  },
};
```

このようなシンプルなコードが、世界中のエッジロケーションで低遅延かつ低コストで動作します。

### まとめ：適材適所のインフラ選定

今回の事例は「AWS がダメで Cloudflare が良い」という単純な話ではありません。AWS には強固なデータベース群や豊富なマネージドサービスがあり、それらが必要なケースも多々あるかと思います。

しかし、もし現在のプロジェクトで「API のリクエスト数が膨大で、コストが直線的に伸び続けている」という状況であれば、一度立ち止まって計算してみる価値はあるかもしれません。

「なんとなく慣れているから」という理由で Lambda を選び続けるのではなく、実際の数値を元に、V8 Isolates のような新しい技術を選択肢に入れてみると、驚くほどの最適化ができる可能性があると感じました。インフラ構成を「数学的」に見直してみることが、エンジニアとしての重要な役割の一つなのかもしれませんね。

## 参照記事

- [He Was a Lambda Fanboy… Until the Math Showed Cloudflare Workers Saved Us $1.1M a Year](https://medium.com/@cachecowboy/he-was-a-lambda-fanboy-until-the-math-showed-cloudflare-workers-saved-us-1-1m-a-year-3b5dac83235e)
- [Architecture Patterns That Actually Scale in 2025 (Most Teams Need Only These Three)](https://medium.com/@anurag.ydv36/architecture-patterns-that-actually-scale-in-2025-most-teams-need-only-these-three-02a1bd6084bf)
- [I Rewrote jQuery in Assembly for Artistic Reasons](https://medium.com/@sohail_saifi/i-rewrote-jquery-in-assembly-for-artistic-reasons-9df842969e1f)
- [If You Don’t Know These 12 System Design Basics, You’re Not a Real Software Engineer](https://medium.com/@kanishks772/if-you-dont-know-these-12-system-design-basics-you-re-not-a-real-software-engineer-28043fbb09bc)
- [The 9 Hidden C Features Nobody Told You About (But Every Senior Dev Uses)](https://medium.com/@mahadrajpoot911/the-9-hidden-c-features-nobody-told-you-about-but-every-senior-dev-uses-e699a05e0d7e)
- [21 OpenClaw Automations Nobody Talks About — Because the Obvious Ones Already Broke the Internet](https://medium.com/@rentierdigital/21-openclaw-automations-nobody-talks-about-because-the-obvious-ones-already-broke-the-internet-3f881b9e0018)

---

詳しくは[こちら](https://microarchitectures.jp/blog/aws-lambda-to-cloudflare-workers-migration-saving-1-1m/)をご覧ください。