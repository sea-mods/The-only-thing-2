---
title: "【論文紹介】Qwen3 Embedding"
date: 2026年2月3日
tags: [saved]
source: "Zennの「NLP」のフィード"
link: https://zenn.dev/arai0/articles/da8bdf3fb128c3
author: ""
feedTitle: "Zennの「NLP」のフィード"
guid: "https://zenn.dev/arai0/articles/da8bdf3fb128c3"
---

# 【論文紹介】Qwen3 Embedding

今更ですがQwen3 Embeddingがず〜っと強い印象があるので、テクニカルレポートの紹介をしてみます。

**読む論文: [Qwen3 Embedding: Advancing Text Embedding and  
Reranking Through Foundation Models](https://arxiv.org/pdf/2506.05176)**  
※特別な言及のない限り、本記事中の図表は上記の論文からの引用です。

忙しい方向け
------

*   言語モデルQwen3をベースにした高性能なEmbeddingモデル、Rerankingモデルを構築
*   弱教師あり学習->教師あり学習->モデルマージの3段階訓練で高い性能を実現
*   Qwen3にペルソナを付与した多様な訓練データの合成が効果的

モデルの構築
------

### アーキテクチャ

*   単一のテキスト（Query）を埋め込み表現に変換する**Embeddingモデル**と、QueryとDocからなるテキストペアの類似度スコアを計算する**Rerankingモデル**の2種類を構築
    *   Embeddnigモデルも2つのテキストをそれぞれエンコードしてコサイン類似度を計算することで、テキストペアの類似度をはかることを意図して構築されているが、2文を同時に見ながら計算できるためRerankingモデルの方が高精度かつ高コスト

![](https://zenn.dev/zenn-user-upload/251bd7ca9225-20260203.png)  
_それぞれのモデルの推論イメージ_

#### （１）Embeddingモデル

*   `{Instruction}{Query}<|endoftext|>`の形式で入力し、`<|endoftext|>`の最終層埋め込みを`{Query}`の表現として利用
    *   Qwen3 Embeddingは指示文を入力できるモデルであり、利用したいタスクの情報などを`{Instruction}`に当てはめる
        *   [AmazonPolarityClassificationでの評価で利用された指示文例](https://github.com/QwenLM/Qwen3-Embedding/blob/main/evaluation/task_prompts.json)：Classify Amazon reviews into positive or negative sentiment

#### （２）Rerankingモデル

*   指示、クエリ、文章を入力とし、文章がクエリと指示に対して合っているか、yesもしくはnoを出力させ、`yesの出力確率/ (yesの出力確率+noの出力確率)`をスコアとして利用

実際のプロンプトテンプレート

      <|im_start|>system
    Judge whether the Document meets the requirements based on the Query and the
    Instruct provided. Note that the answer can only be "yes" or
    "no".<|im_end|>,→,→<|im_start|>user
    <Instruct>: {Instruction}
    <Query>: {Query}
    <Document>: {Document}<|im_end|>
    <|im_start|>assistant
    <think>\n\n</think>\n\n

* * *

### 訓練

*   モデルの訓練は大規模な合成データによる弱教師あり学習、高品質データによる教師あり学習、モデルマージの3段階からなる
    *   Rerankingモデルは弱教師あり学習をskip  
        ![Training pipeline](https://zenn.dev/zenn-user-upload/0b96b8312c3b-20260203.png)  
        _多段階訓練の概要_

#### （１）Embeddingモデル

*   InfoNCEベースの対照学習損失で、クエリと正例の文章の類似度を大きく、その他のペアの類似度を小さくするように訓練を進める

L\_{\\text{embedding}} = -\\frac{1}{N} \\sum\_{i}^{N} \\log \\frac{e^{(s(q\_i, d\_i^+) / \\tau)}}{Z\_i}\\tag{1},

\\begin{aligned} Z\_i &= e^{(s(q\_i, d\_i^+) / \\tau)} + \\sum\_{k}^{K} m\_{ik} e^{(s(q\_i, d\_{i,k}^-) / \\tau)} \\\\ &\\quad + \\sum\_{j \\neq i} m\_{ij} e^{(s(q\_i, q\_j) / \\tau)} + \\sum\_{j \\neq i} m\_{ij} e^{(s(d\_i^+, d\_j) / \\tau)} + \\sum\_{j \\neq i} m\_{ij} e^{(s(q\_i, d\_j) / \\tau)}\\tag{2} \\end{aligned}

*   類似度を小さくするペアとしては、**クエリと負例文章**（hard negative、式(2)第2項）および**クエリとバッチに含まれるその他の文章**（in-batch negatives、第5項）のみを利用するのではなく、**クエリとバッチに含まれるその他のクエリ**（第3項）、**文章とバッチに含まれるその他の文章**（第4項）も利用
    *   ただし、正例ペアよりも0.1以上類似度が大きいペア、もしくは正例文書と全く一緒の負例文書については、偽陰性の可能性が高いため、係数mを0にする（訓練に利用しない）  
        （負例としてクエリxクエリ、文章x文章も使うの、最近よく見る気がする 計算コスト変わらず訓練に使うペアを増やせるからやり得なのかしら…）

#### （2）Rerankingモデル

*   SFT lossで、正例ペアについてはyesを、負例ペアについてはnoを出力するように訓練
    *   言語モデルの訓練と同じ、次単語予測なので詳細割愛

* * *

### 訓練データの合成

*   Qwen3の事前学習コーパスに含まれる多言語な文章をベースに、正例となるクエリをQwen3-32Bで生成
*   生成プロセスはデータの多様性を確保するための構成ステップと、生成ステップからなる
    *   構成ステップでは、生成する際にLLMに与えるペルソナ(!!)と、クエリを生成するための要件（タスクの種類、難易度、クエリの長さ）を決定
        *   プロンプトテンプレート
            
            ![](https://zenn.dev/zenn-user-upload/b587105a5205-20260203.png)
            
        *   ペルソナは、その文章について最も興味を持ちそうなものを[Persona Hub](https://huggingface.co/datasets/proj-persona/PersonaHub)から選択
    *   生成ステップでは、文章及び構成ステップで決定したペルソナ、要件から、クエリを生成
        *   プロンプトテンプレート
            
            ![](https://zenn.dev/zenn-user-upload/231350f754ca-20260203.png)
            
*   約150Mペアを生成し、訓練Stage1 弱教師あり学習ではそのすべてを、Stage2 教師あり学習では品質の高い（コサイン類似度が0.7以上の）12Mペアを利用
    *   Stage2では合成データに加えて既存のデータセット7Mも利用

評価
--

### ベンチマークでの比較実験

![](https://zenn.dev/zenn-user-upload/61324c426fb9-20260203.png)  
_EmbeddingモデルのMTEBでの評価結果_  
![](https://zenn.dev/zenn-user-upload/3bb8915fdbfb-20260203.png)  
_Rerankingモデルの評価結果_

*   とっても良好な結果

2025年6月に発表されてから半年以上経過した現段階でも、[MTEB Leaderboard（Multilingual）](https://huggingface.co/spaces/mteb/leaderboard)でEmbedding 8Bモデルが3位、1B以下に絞ると0.6Bモデルが1位の性能を示していたりと、強モデルシリーズだなという感じ

* * *

### 訓練パイプラインの有効性分析

![](https://zenn.dev/zenn-user-upload/5f7301c632d1-20260203.png)

*   合成データの利用（Stage 1）は効果的
    *   Stage1なしのモデル（表中の2行目）はQwen3-0.6Bと比較して明確に性能が低下
    *   Stage1のみのモデル1行目）は（事前学習しかしていないモデルにしては）強力な性能
*   モデルマージは効果的
    *   モデルマージをせず、データサンプリングでタスクのバランスを調整したモデル（3行目）はQwen3-0.6Bと比較して明確に性能が低下

おわり
---

ペルソナを使った合成データのスケーリングをテキスト埋め込みに導入している論文は初見で、プロンプトテンプレート面白いな〜と眺めていました。この頃の埋め込みモデル作り、訓練データが肝すぎる…！という感覚になっていたので（訓練方法の探索は一段落した感じがあるため）、どこかでつかってみたい…………

[Source](https://zenn.dev/arai0/articles/da8bdf3fb128c3)