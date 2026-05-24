---
title: "OpenCVとスレッディングで実現する、遅延の少ないリアルタイム動画解析パイプラインの構築"
emoji: "📘"
type: "tech"
topics: ["OpenCV", "Threading", "リアルタイム処理", "動画分析", "パイプライン構築"]
published: true
---

今回は、**Building a Real-Time Video Analytics Pipeline with OpenCV and Threading** という記事を読み、リアルタイムな動画処理におけるパフォーマンス改善の手法が非常に実践的だと感じたので、その内容を整理してご紹介します。

カメラ映像をリアルタイムで解析しようとすると、処理の重さによって映像がカクついたり、遅延が積み重なったりすることがよくあります。こうした課題を「スレッディング（多重スレッド化）」という手法でどのように解決できるのか、私なりにまとめてみました。

元記事は、とっても良い記事です。時時間の動画を python で処理するときスレッドで処理しないとフレームを落としますよね。スレッドを使って書くにはどうすれば良いのかを説明しています。参考になります。

---

## なぜ動画解析にスレッディングが必要なのか

一般的な動画処理プログラムでは、1つのスレッド（メイン処理）の中で「フレームの読み込み」と「フレームの加工・解析」を順番に行います。

しかし、この構成には大きな弱点があります。たとえば30fps（1秒間に30フレーム）の動画を処理する場合、1フレームあたり約33ミリ秒以内にすべての処理を終える必要があります。もし解析処理に40ミリ秒かかってしまうと、次のフレームの読み込みが遅れ、結果として表示がカクついてしまいます。

### シーケンシャル処理とスレッド処理の比較

以下の図は、処理の流れをイメージしたものです。

```mermaid
flowchart TD
    subgraph "シーケンシャル処理（逐次処理）"
        A[フレーム読込] --> B[解析・表示]
        B --> C[次のフレーム読込]
        C --> D[解析・表示]
    end

    subgraph "スレッド処理（並列処理）"
        direction LR
        P[スレッドA: フレーム読込] -- キューへ投入 --> Q[(キュー)]
        Q -- 取り出し --> R[スレッドB: 解析・表示]
    end
```

このように、読み込みと解析を切り離す（デカップリングする）ことで、読み込み側は解析の完了を待たずに次のフレームを準備できるようになります。

---

## パイプラインの設計

今回の手法では、**プロデューサー・コンシューマー（生成者・消費者）パターン**を採用します。

1.  **プロデューサー・スレッド**: カメラやビデオファイルからフレームを高速に読み込み、共有のキュー（Queue）に保存します。
2.  **コンシューマー・スレッド**: キューから最新のフレームを取り出し、画像解析（フィルタリングや物体検出など）を行って結果を表示します。

### 実装のポイント
Pythonで実装する場合、標準ライブラリの `threading` モジュールと、スレッド間で安全にデータをやり取りできる `queue.Queue` を使うのが一般的です。

```python
import cv2
import threading
import queue
import time

class VideoStream:
    def __init__(self, src=0):
        self.stream = cv2.VideoCapture(src)
        self.queue = queue.Queue(maxsize=128)
        self.stopped = False

    def start(self):
        # フレーム読み込み専用のスレッドを開始
        t = threading.Thread(target=self.update, args=())
        t.daemon = True
        t.start()
        return self

    def update(self):
        while True:
            if self.stopped:
                return
            
            # キューが一杯でない場合に読み込み
            if not self.queue.full():
                ret, frame = self.stream.read()
                if not ret:
                    self.stop()
                    return
                self.queue.put(frame)

    def read(self):
        # キューから最新のフレームを取り出す
        return self.queue.get()

    def stop(self):
        self.stopped = True
```

このようにクラス化しておくと、メインの処理側では `read()` を呼ぶだけでよくなり、読み込みの待ち時間を意識せずに済みます。

---

## 逐次処理とスレッド処理の比較

実際にこれらを比較してみると、以下のような違いが出てくるかと思います。

| 項目 | 逐次処理 (Sequential) | スレッド処理 (Threaded) |
| :--- | :--- | :--- |
| **FPS（フレームレート）** | 解析処理の重さに依存して低下しやすい | 安定しやすく、上限が高まる |
| **遅延 (Latency)** | 処理が詰まると古いフレームが溜まる | 最新のフレームを優先して取得しやすい |
| **CPU利用率** | 1つのコアに負荷が集中する | 読み込みと解析で負荷を分散できる |
| **実装の複雑さ** | シンプル | 競合状態やメモリ管理への配慮が必要 |

実際に試してみると分かりますが、たとえ単純なグレースケール変換であっても、高解像度のWebカメラなどではスレッド化した方が滑らかに動くことが多いです。

---

## 実務で注意すべき点

この構成を導入する際、いくつか気をつけておくべきポイントがあると感じました。

1.  **キューのサイズ管理**:
    キューを無限に大きくしてしまうと、解析が遅い場合にメモリを大量に消費してしまいます。`maxsize` を設定し、古いフレームを捨てる（ドロップさせる）仕組みが必要になるかもしれません。
2.  **PythonのGIL（グローバルインタプリタロック）**:
    Pythonのスレッドは、実はCPU負荷の高い計算を同時に行うのには向いていません（GILの影響があるため）。ただし、今回のような「カメラからの読み込み待ち（I/O待ち）」が発生するケースでは、スレッディングは非常に効果的です。
3.  **リソースの解放**:
    スレッドが予期せず残ってしまうと、アプリケーションが終了しなくなることがあります。`daemon=True` を設定したり、終了フラグを適切に管理したりするのが安全かと思います。

---

## まとめ

リアルタイム動画解析において、OpenCVの標準的な `read()` メソッドをそのままループの中で使うのは、実はかなり贅沢な時間の使い方をしている可能性があります。

今回紹介したようにスレッドを分けることで、入力と処理の「待ち」をうまく隠蔽し、システム全体のパフォーマンスを底上げできるはずです。まずはシンプルなキューの実装から始めて、解析パイプラインのボトルネックを探ってみるのが良いのではないでしょうか。

「たかがスレッド、されどスレッド」といったところで、少しの工夫でユーザー体験が大きく変わる部分だと改めて感じました。

## 参照記事

- [Building a Real-Time Video Analytics Pipeline with OpenCV and Threading](https://medium.com/@sohail_saifi/building-a-real-time-video-analytics-pipeline-with-opencv-and-threading-d6f841c7b56b)
- [I Solved a Race Condition in Front of My Interviewer — And Got the Offer Instantly](https://medium.com/@koteshavula/i-solved-a-race-condition-in-front-of-my-interviewer-and-got-the-offer-instantly-fcf4e0997a6f)
- [Go Just Killed Rust’s Only Advantage (And Nobody’s Talking About It)](https://medium.com/@kanishks772/go-just-killed-rusts-only-advantage-and-nobody-s-talking-about-it-0d5fc550f355)
- [The Thread Pool Configuration That Prevents Deadlocks](https://medium.com/@sohail_saifi/the-thread-pool-configuration-that-prevents-deadlocks-e1435b04ef50)
- [Python’s GIL Problem Is Real. And It Cost Us $100K](https://medium.com/@devrimozcay/pythons-gil-problem-is-real-and-it-cost-us-100k-dbbd05239b98)
- [Project Loom Killed Half Our Microservices: What Java Architects Need to Unlearn](https://medium.com/@yashbatra11111/project-loom-killed-half-our-microservices-what-java-architects-need-to-unlearn-3d6bf01814bf)

---

詳しくは[こちら](https://microarchitectures.jp/blog/low-latency-real-time-video-analysis-opencv-threading/)をご覧ください。