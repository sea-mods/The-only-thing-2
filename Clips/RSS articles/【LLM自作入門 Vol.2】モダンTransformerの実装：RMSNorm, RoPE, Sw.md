---
title: "【LLM自作入門 Vol.2】モダンTransformerの実装：RMSNorm, RoPE, SwiGLUを使用した実装方法"
date: 2026年1月16日
tags: [saved]
source: "Zennの「NLP」のフィード"
link: https://zenn.dev/lluminai_tech/articles/f488b0843efda3
author: ""
feedTitle: "Zennの「NLP」のフィード"
guid: "https://zenn.dev/lluminai_tech/articles/f488b0843efda3"
---

# 【LLM自作入門 Vol.2】モダンTransformerの実装：RMSNorm, RoPE, SwiGLUを使用した実装方法

はじめに
----

ルミナイR&Dチームの宮脇彰梧です。

「LLMをゼロから作る」連載の第2回です。  
前回（Vol.1）は、BPEトークナイザとデータパイプラインを構築し、テキストをモデルが理解できる数値列に変換する準備を整えました。

今回は、いよいよ**Transformerモデル本体**を実装します。  
ただし、教科書的な「Original Transformer (2017)」ではなく、Llama 3 や Mistral などのSOTAモデルで採用されている「モダンなアーキテクチャ」をPyTorchで構築します。

### この記事で学べること

*   **理論**：なぜ最近のLLMは LayerNorm ではなく RMSNorm を使うのか？
*   **実装**：回転位置埋め込み（**RoPE**）の数式とコード
*   **実装**：活性化関数 **SwiGLU** を用いたFeedForward層
*   **統合**：Decoder-only Transformerのフルスクラッチ構築

### 結論

現代のLLMは、学習の安定性と推論性能を高めるために、初期のTransformerから「正規化の位置（Pre-Norm）」「位置情報の扱い（RoPE）」「活性化関数（SwiGLU）」の3点を大きく進化させています。

1\. なぜこのテーマを選んだのか
-----------------

「Transformerの実装」という記事は世の中に溢れていますが、その多くは `nn.Transformer` を呼ぶだけか、GPT-2（2019年）の構成を解説したものです。

しかし、私が研究や実務で触れる最新モデル（Llama, Gemma等）は、内部構造が微妙に異なります。この「微妙な違い」こそが、学習の安定性やコンテキスト長の拡張（Scaling）に直結しています。  
今回は、現代のLLM開発者が知っておくべき「モダンな標準構成」を、数式とともにコードに落とし込みます。

### 📋 ひと目でわかる比較表

比較項目

👴 従来 (GPT-2, Original)

🤖 モダン (Llama, Gemma)

進化のメリット

**1\. 正規化**

**LayerNorm**

**RMSNorm**

計算が速い＆学習が安定する

**2\. 位置情報**

**絶対位置** (Absolute)

**RoPE** (回転位置)

長い文章でもボケない（外挿性能UP）

**3\. 活性化関数**

**ReLU / GELU**

**SwiGLU**

表現力が高く、賢くなりやすい

**(構造)**

**Post-Norm**

**Pre-Norm**

深い層でも学習が壊れにくい

2\. 関連調査
--------

今回の実装のベースとなる、現代LLMのデファクトスタンダードを確立した論文を分析します。

### 📘 Paper: Llama 2: Open Foundation and Fine-Tuned Chat Models

*   **Origin**: arXiv:2307.09288 (2023)
*   **Authors**: Hugo Touvron et al. (Meta AI)
*   **概要**: Llama 2の開発詳細を記したテクニカルレポート。アーキテクチャとして「RMSNorm (Pre-Norm)」「SwiGLU」「RoPE」「GQA (Grouped-Query Attention)」を採用し、オープンソースLLMの標準を決定づけた。

#### 📊 評価表（Peer Review Metrics）

指標

評価

コメント

**新規性**

★★★☆☆

個々の技術（RoPEやSwiGLU）は既知だが、それらを最適に組み合わせた「レシピ」の確立に価値がある。

**実用性**

★★★★★

産業界における標準アーキテクチャとなっており、実装上の知見として極めて重要。

**再現性**

★★★★★

詳細なハイパーパラメータと訓練手順が開示されている。

**技術深度**

★★★★☆

スケーリング則に基づいた設計と、安全性評価のプロセスが詳細。

**妥当性**

★★★★★

数兆トークンの学習において安定してLossが下がることを実証済み。

#### 🧾 筆者のコメント

> **Strengths:**  
> 初期のGPT-3スタイルから脱却し、学習効率と推論速度のトレードオフを最適化した構成。特に **RoPE (Rotary Positional Embeddings)** の採用により、外挿性能（学習時より長い文脈への対応）が理論的に向上している点が重要。
> 
> **Implication for us:**  
> 我々が作るLLMも、Llama 2の構成（RMSNorm, SwiGLU, RoPE）に準拠することで、学習の安定性を担保しつつ、FlashAttentionなどの高速化カーネルの恩恵を受けやすくする。

3\. 実装・検証（Dev：やってみた）
--------------------

それでは、`torch.nn.Module` を継承して各コンポーネントを作っていきます。  
最終的に `LlamaModel` クラスとして組み上げます。

### 3.1 設定クラス (Config)

まず、モデルの設計図となるハイパーパラメータを定義します。

    from dataclasses import dataclass
    import torch
    import torch.nn as nn
    import torch.nn.functional as F
    import math
    
    @dataclass
    class ModelArgs:
        dim: int = 512          # 埋め込み次元 (d_model)
        n_layers: int = 8       # レイヤー数
        n_heads: int = 8        # Attentionヘッド数
        vocab_size: int = 32000 # 語彙数 (Vol.1で作成したtokenizerに合わせる)
        multiple_of: int = 256  # SwiGLUの隠れ層次元の調整用
        max_seq_len: int = 512  # 最大コンテキスト長
        dropout: float = 0.1
        
        # 計算用プロパティ
        @property
        def head_dim(self):
            return self.dim // self.n_heads
    

今回は学習実験用なので「ミニサイズ」の設定ですが、この数字を増やせばそのまま7B/70Bクラスになります。

### 3.2 RMSNorm (Root Mean Square Layer Normalization)

通常のLayerNormは平均と分散を使いますが、RMSNormは**平均を引かず（センタリングなし）、二乗平均平方根だけで正規化**します。計算量が少なく、学習が安定しやすいとされています。

\\bar{a}\_i = \\frac{a\_i}{\\text{RMS}(a)} g\_i, \\quad \\text{where } \\text{RMS}(a) = \\sqrt{\\frac{1}{n} \\sum\_{i=1}^{n} a\_i^2 + \\epsilon}

    class RMSNorm(torch.nn.Module):
        def __init__(self, dim: int, eps: float = 1e-6):
            super().__init__()
            self.eps = eps
            self.weight = nn.Parameter(torch.ones(dim))
    
        def _norm(self, x):
            # x: (Batch, Seq, Dim)
            return x * torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + self.eps)
    
        def forward(self, x):
            output = self._norm(x.float()).type_as(x)
            return output * self.weight
    

### 3.3 RoPE (Rotary Positional Embeddings)

ここが最大の難所です。  
絶対位置埋め込み（学習可能なベクトルを足す）ではなく、クエリとキーのベクトルを「回転」させることで相対的な位置関係を注入します。

\\begin{pmatrix} q\_0 \\\\ q\_1 \\end{pmatrix} \\rightarrow \\begin{pmatrix} \\cos m\\theta & -\\sin m\\theta \\\\ \\sin m\\theta & \\cos m\\theta \\end{pmatrix} \\begin{pmatrix} q\_0 \\\\ q\_1 \\end{pmatrix}

    def precompute_freqs_cis(dim: int, end: int, theta: float = 10000.0):
        # 回転角度の事前計算 (複素数平面で考えると楽)
        freqs = 1.0 / (theta ** (torch.arange(0, dim, 2)[: (dim // 2)].float() / dim))
        t = torch.arange(end, device=freqs.device)  # type: ignore
        freqs = torch.outer(t, freqs).float()  # (Seq, Dim/2)
        freqs_cis = torch.polar(torch.ones_like(freqs), freqs)  # complex64
        return freqs_cis
    
    def apply_rotary_emb(xq, xk, freqs_cis):
        # xq, xk: (Batch, Seq, Head, HeadDim) -> 複素数化して回転
        xq_ = torch.view_as_complex(xq.float().reshape(*xq.shape[:-1], -1, 2))
        xk_ = torch.view_as_complex(xk.float().reshape(*xk.shape[:-1], -1, 2))
        
        # broadcastingのためにshapeを合わせる
        freqs_cis = freqs_cis[:xq.shape[1]].view(1, xq.shape[1], 1, -1)
        
        # 回転適用
        xq_out = torch.view_as_real(xq_ * freqs_cis).flatten(3)
        xk_out = torch.view_as_real(xk_ * freqs_cis).flatten(3)
        return xq_out.type_as(xq), xk_out.type_as(xk)
    

### 3.4 Attention Mechanism (Causal Self-Attention)

PyTorch 2.0以降の `F.scaled_dot_product_attention` を活用しつつ、RoPEを適用するフローを作ります。

    class Attention(nn.Module):
        def __init__(self, args: ModelArgs):
            super().__init__()
            self.n_heads = args.n_heads
            self.head_dim = args.head_dim
            
            self.wq = nn.Linear(args.dim, args.n_heads * self.head_dim, bias=False)
            self.wk = nn.Linear(args.dim, args.n_heads * self.head_dim, bias=False)
            self.wv = nn.Linear(args.dim, args.n_heads * self.head_dim, bias=False)
            self.wo = nn.Linear(args.n_heads * self.head_dim, args.dim, bias=False)
    
        def forward(self, x, freqs_cis):
            # x: (Batch, Seq, Dim)
            bsz, seqlen, _ = x.shape
            
            # Q, K, V の射影 & Head分割
            xq = self.wq(x).view(bsz, seqlen, self.n_heads, self.head_dim)
            xk = self.wk(x).view(bsz, seqlen, self.n_heads, self.head_dim)
            xv = self.wv(x).view(bsz, seqlen, self.n_heads, self.head_dim)
    
            # RoPE の適用 (QとKを回転させる)
            xq, xk = apply_rotary_emb(xq, xk, freqs_cis)
    
            # Flash Attention (is_causal=True で因果マスク自動適用)
            output = F.scaled_dot_product_attention(
                xq.transpose(1, 2), # (B, H, S, D)
                xk.transpose(1, 2),
                xv.transpose(1, 2),
                is_causal=True
            )
            
            output = output.transpose(1, 2).contiguous().view(bsz, seqlen, -1)
            return self.wo(output)
    

### 3.5 FeedForward (SwiGLU)

ReLUではなく、SiLU（Swish）とGating機構を組み合わせたものです。パラメータ数は通常のFFNより増えますが、性能が良いです。

    class FeedForward(nn.Module):
        def __init__(self, args: ModelArgs):
            super().__init__()
            # 隠れ層のサイズ計算 (Llamaの仕様：2/3 * 4d 程度にしてmultiple_of倍数にする)
            hidden_dim = 4 * args.dim
            hidden_dim = int(2 * hidden_dim / 3)
            hidden_dim = args.multiple_of * ((hidden_dim + args.multiple_of - 1) // args.multiple_of)
    
            self.w1 = nn.Linear(args.dim, hidden_dim, bias=False) # Gate
            self.w2 = nn.Linear(hidden_dim, args.dim, bias=False) # Down
            self.w3 = nn.Linear(args.dim, hidden_dim, bias=False) # Up
    
        def forward(self, x):
            # SwiGLU: w2( SiLU(w1(x)) * w3(x) )
            return self.w2(F.silu(self.w1(x)) * self.w3(x))
    

### 3.6 Transformer Block & Model

これらを積み上げます。  
重要なのは **Pre-Norm**（Attention/FFNの**前**にNormをかける）である点です。オリジナルのTransformer（Post-Norm）より深い層でも学習が安定します。

    class TransformerBlock(nn.Module):
        def __init__(self, args: ModelArgs):
            super().__init__()
            self.attention_norm = RMSNorm(args.dim)
            self.attention = Attention(args)
            self.ffn_norm = RMSNorm(args.dim)
            self.feed_forward = FeedForward(args)
    
        def forward(self, x, freqs_cis):
            # Residual Connection (Pre-Norm)
            h = x + self.attention(self.attention_norm(x), freqs_cis)
            out = h + self.feed_forward(self.ffn_norm(h))
            return out
    
    class Transformer(nn.Module):
        def __init__(self, args: ModelArgs):
            super().__init__()
            self.args = args
            self.tok_embeddings = nn.Embedding(args.vocab_size, args.dim)
            self.layers = nn.ModuleList([TransformerBlock(args) for _ in range(args.n_layers)])
            self.norm = RMSNorm(args.dim) # Final Norm
            self.output = nn.Linear(args.dim, args.vocab_size, bias=False)
            
            # RoPEテーブルの事前計算
            self.freqs_cis = precompute_freqs_cis(self.args.dim // self.args.n_heads, self.args.max_seq_len * 2)
    
        def forward(self, idx):
            # idx: (Batch, Seq)
            bsz, seqlen = idx.shape
            x = self.tok_embeddings(idx)
            
            # RoPEテーブルをデバイスへ
            freqs_cis = self.freqs_cis[:seqlen].to(x.device)
    
            for layer in self.layers:
                x = layer(x, freqs_cis)
                
            x = self.norm(x)
            logits = self.output(x)
            return logits
    

4\. 筆者の考察
---------

### 4.1 RoPEの美しさと実装の落とし穴

RoPEは「回転」という幾何学的な操作で位置情報を埋め込む非常に美しい手法ですが、実装時には `view_as_complex` や `reshape` の次元操作でバグりやすいです。  
特に、**Head Dim（ヘッドごとの次元数）の半分だけ回転させる**実装や、今回の実装のように**複素数**を使うパターンなど、ライブラリによって流儀が異なります。PyTorchの複素数サポートが充実してきたので、`torch.polar` を使うのが最も可読性が高いと私は感じています。

### 4.2 なぜBiasを抜くのか

今回の実装では `bias=False` を多用しています（Linear層など）。  
LlamaやPaLMなどの最近のモデルは、バイアス項を削除する傾向にあります。これは、LayerNorm（またはRMSNorm）が中心化やスケーリングを行うため、線形層のバイアスが冗長になり、学習の安定性を損なう場合があるからです。  
**引き算の美学**がモダンLLMアーキテクチャのトレンドと言えます。

5\. まとめ
-------

今回は、現代的なLLM（Llamaスタイル）のアーキテクチャをフルスクラッチで実装しました。

1.  **RMSNorm**: 計算効率と安定性を重視。
2.  **RoPE**: 相対位置情報を回転で表現し、長い文脈への対応力を向上。
3.  **SwiGLU**: 表現力の高い活性化関数を採用。
4.  **Pre-Norm**: Residual Blockの入力側で正規化を行い、深層学習を安定化。

これで「データ（Vol.1）」と「モデル（Vol.2）」が揃いました。  
次回Vol.3では、これらを組み合わせて「学習ループ（Training Loop）」を実装し、実際にLossが下がっていく様子（モデルが言語を学習する瞬間）を観測します。

**次の記事**

* * *

執筆：宮脇 彰梧（ルミナイ株式会社 / Lluminai）

* * *

【現在採用強化中です！】

*   AIエンジニア
*   PM/PdM
*   戦略投資コンサルタント

▼代表とのカジュアル面談URL

[Source](https://zenn.dev/lluminai_tech/articles/f488b0843efda3)