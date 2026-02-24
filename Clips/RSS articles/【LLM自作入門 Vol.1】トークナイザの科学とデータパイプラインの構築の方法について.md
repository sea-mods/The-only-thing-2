---
title: "【LLM自作入門 Vol.1】トークナイザの科学とデータパイプラインの構築の方法について"
date: 2026年1月14日
tags: [saved]
source: "Zennの「NLP」のフィード"
link: https://zenn.dev/lluminai_tech/articles/ca54564106414a
author: ""
feedTitle: "Zennの「NLP」のフィード"
guid: "https://zenn.dev/lluminai_tech/articles/ca54564106414a"
---

# 【LLM自作入門 Vol.1】トークナイザの科学とデータパイプラインの構築の方法について

はじめに
----

ルミナイR&Dチームの宮脇彰梧です。  
現在はマルチモーダルAIの研究を行う大学院生として、  
生成AIやAIエージェントの技術を実践的に探求しています。

今回から**LLM（大規模言語モデル）をゼロから作る**という連載シリーズを始めます。既存の `Trainer` や `AutoModel` を使えば一瞬ですが、それでは「なぜ動くのか」「どこで性能が決まるのか」というブラックボックスが残ったままです。

研究者の視点（Why）とエンジニアの視点（How）を行き来しながら、PyTorchの生コードでLLMを構築していきます。

**Vol.1のテーマは「データとトークナイザ」です。**

### この記事で学べること

*   **理論**：Subword（BPE）がなぜ現代LLMの標準なのか
*   **実装**：独自のデータセットでBPEトークナイザを学習させる方法
*   **実装**：PyTorchによる事前学習用 `Dataset` / `DataLoader` の設計

### 構成

1.  背景：なぜトークナイズが重要か
2.  Research：BPEの起源と理論（論文査読）
3.  Dev：トークナイザの学習とパイプライン実装
4.  筆者の考察
5.  まとめ

### 結論

LLMの学習は、モデルにデータを流し込む前の**データの断片化（トークナイズ）の設計時点で、圧縮効率と学習難易度が決定づけられます。**

1\. 背景：なぜトークナイズが重要か
-------------------

LLMの開発において、モデルアーキテクチャ（Transformerの層数やヘッド数）の議論は華やかですが、実は「テキストをどう数値化するか」という前処理の時点で、モデルの性能上限の大部分が決まってしまいます。

LLMの本質は、テキストデータの「圧縮」と「予測」です。もしトークナイザの設計が不適切で、1つの単語を無駄に細かく分割しすぎてしまえば、シーケンス長（文脈の長さ）が不必要に伸び、モデルは「意味」ではなく「綴り」を学習することに計算リソースを浪費してしまいます。逆に、分割が粗すぎれば未知語に対応できません。

「最小の計算量で、最大の意味情報をモデルに伝える」。この最適解を見つけるために、現代のLLMで標準的に使われている Byte-Pair Encoding (BPE) の仕組みを解剖し、実装に落とし込みます。

2\. Research：BPEの起源と理論（論文査読）
----------------------------

今回は、Subword（単語の一部）を用いたトークナイズ手法の起源となった記念碑的論文を分析します。

### 📘 Paper: Neural Machine Translation of Rare Words with Subword Units

*   **Origin**: arXiv:1508.07909 (ACL 2016)
*   **Authors**: Rico Sennrich, Barry Haddow, Alexandra Birch
*   **概要**: 未知語（Rare Words）問題を解決するために、頻出する文字のペアを結合して「サブワード」単位を作成するByte Pair Encoding (BPE) をニューラル機械翻訳に導入した論文。

#### 📊 評価表（Peer Review Metrics）

指標

評価

コメント

**新規性**

★★★★★

従来の単語ベース辞書か文字ベースかの二項対立に対し、統計的に最適な「中間の粒度」を自動獲得する手法を提示。

**実用性**

★★★★★

現代のほぼ全てのLLM（GPT, Llama等）の基礎となっており、計算コストも低い。

**再現性**

★★★★★

アルゴリズムが単純であり、再実装が極めて容易。

**技術深度**

★★★★☆

情報理論的な圧縮の観点に基づいた堅実な設計。

**妥当性**

★★★★★

翻訳タスクにおける未知語処理能力の向上を定量的に実証している。

#### 🧾 筆者コメント

> **Strengths:**  
> 固定された辞書を持たず、データ中の頻度統計のみから語彙を構築するため、あらゆる言語やドメインに柔軟に適応できる点が革命的である。特に「未知語（Unknown Token）」を理論上ゼロにできる（文字単位まで分解できるため）点は、生成モデルにおいて致命的な重要性を持つ。
> 
> **Weaknesses:**  
> 初期提案ではバイト単位ではなく文字単位であったため、漢字などの大規模文字種言語では語彙爆発の懸念があった（後にBBPEなどで解消）。
> 
> **Implication for us:**  
> 我々が作るLLMも、このBPE（またはその派生）を採用することで、日本語のような膠着語（単語の境界が曖昧な言語）を効率的に処理可能になる。

3\. Dev：トークナイザの学習とパイプライン実装
--------------------------

では、実際に「日本語テキストデータ」を用意し、「独自のトークナイザ」を学習させ、PyTorchで読み込める形にするまでを実装します。

### 環境

*   Python 3.10
*   PyTorch 2.1+
*   `tokenizers` (Hugging Face)
*   `datasets` (Hugging Face)

### 3.1 データの準備

ここではデモとして、Hugging Faceの日本語Wikipediaデータセットの一部を使用します。

    from datasets import load_dataset
    import tqdm
    
    # 1. データセットのロード（ストリーミングモードでメモリ節約）
    print("データセットに接続中...")
    dataset = load_dataset("izumi-lab/wikipedia-ja-20230720", split="train", streaming=True)
    
    # 2. 保存処理
    CORPUS_FILE = "wiki_ja_subset.txt"
    print(f"ダウンロードと保存を開始します: {CORPUS_FILE}")
    
    with open(CORPUS_FILE, "w", encoding="utf-8") as f:
        # 最初の1万件だけを取得
        for i, data in enumerate(tqdm.tqdm(dataset)):
            # 改行を除去して1行にする
            text = data["text"].replace("\n", "")
            
            # 短すぎる記事は除外（100文字以上のみ保存）
            if len(text) > 100:
                f.write(text + "\n")
            
            # 10,000件でストップ
            if i >= 10000:
                break
    
    print("✅ データセット作成完了！")
    
    

### 3.2 BPEトークナイザの学習

GPT-2やRobertaと同様の、バイトレベルBPE（Byte-Level BPE）トークナイザを学習させます。  
`tokenizers` ライブラリを使うと非常に高速です。

    from tokenizers import Tokenizer
    from tokenizers.models import BPE
    from tokenizers.trainers import BpeTrainer
    from tokenizers.pre_tokenizers import ByteLevel
    from tokenizers.decoders import ByteLevel as ByteLevelDecoder
    
    # 1. モデルの初期化（BPE）
    tokenizer = Tokenizer(BPE())
    
    # 2. Pre-tokenizerの設定（バイトレベルで分割）
    # GPT-2などと同様、スペースも含めて処理する設定
    tokenizer.pre_tokenizer = ByteLevel(add_prefix_space=False)
    
    # 3. Decoderの設定（IDから文字列に戻す用）
    tokenizer.decoder = ByteLevelDecoder()
    
    # 4. Trainerの設定
    # vocab_size: 語彙数（日本語LLMでは32000〜64000程度が一般的）
    # min_frequency: 登場回数がこれ以下のサブワードは作らない
    vocab_size = 32000
    trainer = BpeTrainer(
        vocab_size=vocab_size,
        min_frequency=2,
        special_tokens=["<|endoftext|>", "<|pad|>"], # 特殊トークンの定義
        show_progress=True
    )
    
    # 5. 学習実行
    tokenizer.train([CORPUS_FILE], trainer)
    
    # 6. 保存
    tokenizer.save("custom_tokenizer.json")
    print(f"トークナイザ学習完了。Vocab Size: {tokenizer.get_vocab_size()}")
    

**実行結果:**

    トークナイザ学習完了。Vocab Size: 32000
    

これで、このWikipediaデータセットの統計情報に基づいた、専用の辞書が完成しました。

### 3.3 動作確認

学習したトークナイザがどのように日本語を分割するか見てみましょう。

    test_sentence = "生成AIの技術は日進月歩で進化しています。"
    encoded = tokenizer.encode(test_sentence)
    
    print(f"Original: {test_sentence}")
    print(f"Tokens:   {encoded.tokens}")
    print(f"IDs:      {encoded.ids}")
    

**出力例:**

    Original: 生成AIの技術は日進月歩で進化しています。
    Tokens:   ['çĶŁæĪĲ', 'AI', 'ãģ®æĬĢè¡ĵ', 'ãģ¯', 'æĹ¥', 'éĢ²', 'æľĪ', 'æŃ©', 'ãģ§', 'éĢ²åĮĸ', 'ãģĹãģ¦ãģĦ', 'ãģ¾ãģĻ', 'ãĢĤ']
    IDs:      [4916, 8585, 4832, 209, 287, 770, 355, 2293, 216, 8767, 1914, 4052, 212]
    

１つずつIDをもとの文字に戻してみましょう。

    # 1つずつのIDを元の文字に戻して表示する検証コード
    print(f"元の文: {test_sentence}\n")
    print("--- トークンごとの分割内訳 ---")
    
    for t_id in encoded.ids:
        # IDを1つだけデコードする
        word = tokenizer.decode([t_id])
        print(f"ID: {t_id:5d} | トークン: {word}")
    

**出力例：**

    元の文: 生成AIの技術は日進月歩で進化しています。
    
    --- トークンごとの分割内訳 ---
    ID:  4916 | トークン: 生成
    ID:  8585 | トークン: AI
    ID:  4832 | トークン: の技術
    ID:   209 | トークン: は
    ID:   287 | トークン: 日
    ID:   770 | トークン: 進
    ID:   355 | トークン: 月
    ID:  2293 | トークン: 歩
    ID:   216 | トークン: で
    ID:  8767 | トークン: 進化
    ID:  1914 | トークン: してい
    ID:  4052 | トークン: ます
    ID:   212 | トークン: 。
    

頻出語（「生成」「進化」）は1トークンで、少し珍しい熟語（「日進月歩」など）は文字単位や短いサブワードに分割されていることがわかります。これがBPEの効果です。

### 3.4 PyTorch Datasetの実装

事前学習では、大量のテキストを固定長（Context Length）で切り出してモデルに入力します。  
ここでは、**Next Token Prediction（次の単語予測）** 用のデータセットクラスを作成します。

    import torch
    from torch.utils.data import Dataset
    
    class LLMPretrainDataset(Dataset):
        def __init__(self, txt_file, tokenizer, max_length=512):
            self.tokenizer = tokenizer
            self.max_length = max_length
            self.input_ids = []
    
            # 全テキストを読み込んでトークナイズ
            with open(txt_file, "r", encoding="utf-8") as f:
                text = f.read()
            
            # 一気にエンコード
            tokens = tokenizer.encode(text).ids
            
            # max_lengthごとに分割（簡易的なスライディングウィンドウなしの実装）
            # strideをつける場合は step = max_length (オーバーラップなし)
            for i in range(0, len(tokens) - max_length, max_length):
                self.input_ids.append(tokens[i : i + max_length])
                
        def __len__(self):
            return len(self.input_ids)
        
        def __getitem__(self, idx):
            # input:  x_1, x_2, ..., x_T
            # target: x_2, x_3, ..., x_{T+1} (1つ右にずらす)
            chunk = self.input_ids[idx]
            
            x = torch.tensor(chunk, dtype=torch.long)
            y = torch.tensor(chunk, dtype=torch.long) # 実際はずらしてLoss計算時に処理するが、ここではデータとしては同じものを返すのが通例
            
            return x, y
    
    # 動作確認
    dataset = LLMPretrainDataset(CORPUS_FILE, tokenizer, max_length=128)
    print(f"総サンプル数: {len(dataset)}")
    
    x, y = dataset[0]
    print(f"Input shape: {x.shape}")
    

**出力例（イメージ）:**

    総サンプル数: 143540
    Input shape: torch.Size([128])
    

これで、`DataLoader` に渡せば学習ループを回せる準備が整いました。

4\. 筆者の考察
---------

### 4.1 語彙数と圧縮率のトレードオフ

BPEの語彙数（Vocab Size）の設定は、モデル設計の最初の大きな分岐点です。

*   **語彙数が小さい場合**：
    *   1単語あたりのトークン数が増える（「ルミナイ」→「ル」「ミ」「ナ」「イ」）。
    *   シークエンス長が伸びるため、Attentionの計算量（O(N^2)）が増大する。
    *   Embedding層のパラメータ数は減る。
*   **語彙数が大きい場合**：
    *   1トークンで表現できる情報量が増える（「ルミナイ」→「ルミナイ」）。
    *   コンテキストウィンドウ内に多くの情報を詰め込める。
    *   ただし、Embedding層が巨大になり（例：vocab 10万 × dim 4096）、メモリを圧迫する。

近年のLlama 3（vocab size 128k）やGeminiなどは語彙数を大きく取る傾向にあります。これは、計算リソースの増大に伴い、**「埋め込み層が大きくても、文脈圧縮率を高めて推論効率（1トークンあたりの情報量）を上げたい」** という意図が見えます。

### 4.2 独自のトークナイザを作る意義

既存のトークナイザをそのまま使うことも可能ですが、日本語特化モデルを作る場合、**「日本語の圧縮効率」** が極めて悪くなることがあります（ひらがなが1文字1トークンに分解される等）。  
自社データセットの分布に合わせてBPEを再学習させることで、同じコンテキスト長でも、扱える実質的なテキスト量は1.5倍〜2倍になることもあります。これは推論コスト直結の重要な最適化ポイントです。

5\. まとめ
-------

今回は「LLMをゼロから作る」の第一歩として、データの数値化（トークナイズ）を行いました。

1.  **BPE（Byte-Pair Encoding）** は、データ統計に基づいて最適な「サブワード」を発見するアルゴリズムである。
2.  **`tokenizers` ライブラリ** を使えば、独自データセットに適合した高速なトークナイザを簡単に作成できる。
3.  **PyTorch Dataset** では、トークン列を固定長（Chunk）に切り出し、モデルに供給するパイプラインを構築する。

次回は、このデータを入力として受け取る**Transformerアーキテクチャ**を、`nn.Module` を継承してゼロから実装します。Self-Attentionの数式をコードに落とし込んでいきましょう。

**次の記事**

* * *

執筆：宮脇 彰梧（ルミナイ株式会社 / Lluminai）

* * *

【現在採用強化中です！】

*   AIエンジニア
*   PM/PdM
*   戦略投資コンサルタント

▼代表とのカジュアル面談URL

[Source](https://zenn.dev/lluminai_tech/articles/ca54564106414a)