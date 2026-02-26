---
title: "音楽生成モデル「ACE-Step-1.5」を試す"
source: "https://zenn.dev/kun432/scraps/8503a288b1b3d4"
author:
  - "[[Zenn]]"
published: 21日前にクローズ
created: 2026-02-26
description: "kun432さんのスクラップ"
tags:
  - "clippings"
---
GitHubレポジトリ

GitHubのREADMEから抜粋。翻訳はPLaMo翻訳。

> ## ACE-Step 1.5
> 
> **オープンソース音楽生成の可能性をさらに押し広げる**
> 
> ![](https://storage.googleapis.com/zenn-user-upload/0558170919b1-20260204.png)  
> *referred from [https://github.com/ace-step/ACE-Step-1.5](https://github.com/ace-step/ACE-Step-1.5)*

> ## 📝 概要
> 
> 🚀 本プロジェクトでは、商用レベルの音楽生成性能を一般消費者向けハードウェアで実現する、極めて効率的なオープンソース音楽基盤モデル「ACE-Step v1.5」を紹介します。一般的な評価指標において、ACE-Step v1.5は大多数の商用音楽モデルを凌駕する品質を達成しつつ、驚異的な処理速度を実現しています。A100 GPUでは1曲あたり2秒未満、RTX 3090では10秒未満という超高速処理が可能です。モデルはローカル環境で動作し、VRAM使用量も4GB未満と軽量です。さらに、軽量なパーソナライゼーション機能を備えており、ユーザーはわずか数曲からLoRAモデルを学習させることで、自身の音楽スタイルを反映させることができます。
> 
> 🌉 その中核となるのは、言語モデル（LM）が万能型プランナーとして機能する革新的なハイブリッドアーキテクチャです。単純なユーザー入力を、短いループから10分に及ぶ楽曲まで、あらゆる規模の楽曲構成に拡張可能であり、Chain-of-Thought手法を用いてメタデータ、歌詞、キャプションを生成することで、Diffusion Transformer（DiT）の生成プロセスを導くことができます。⚡ 特筆すべきは、この整合性がモデルの内部メカニズムのみに依存する内在的強化学習によって実現されている点です。これにより、外部報酬モデルや人間の嗜好に起因するバイアスを完全に排除しています。🎚️
> 
> 🔮 標準的な楽曲生成機能に加え、ACE-Step v1.5は精密なスタイル制御と多様な編集機能を統合しています。カバー曲生成、リペイント、ボーカルからBGMへの変換など、多彩な編集が可能でありながら、50以上の言語にわたってプロンプトへの厳密な準拠を維持しています。これにより、音楽アーティスト、プロデューサー、コンテンツクリエイターのクリエイティブワークフローにシームレスに統合できる強力なツール群が実現します。🎸
> 
> ## ✨ 主な機能
> 
> ![](https://storage.googleapis.com/zenn-user-upload/d6475422b427-20260204.png)  
> *referred from [https://github.com/ace-step/ACE-Step-1.5](https://github.com/ace-step/ACE-Step-1.5)*
> 
> ### ⚡ パフォーマンス
> 
> - ✅ **超高速生成** — A100環境でフル楽曲1曲あたり2秒未満、RTX 3090では10秒未満で生成可能（A100環境では思考モードと拡散ステップ数に応じて0.5秒～10秒の範囲で変動）
> - ✅ **柔軟な再生時間設定** — 10秒から10分（600秒）までの音声生成に対応
> - ✅ **一括生成機能** — 最大8曲を同時に生成可能
> 
> ### 🎵 生成品質
> 
> - ✅ **商用レベルの出力品質** — ほとんどの商用音楽モデルを凌駕する品質を実現（Suno v4.5とSuno v5の中間レベル）
> - ✅ **豊富なスタイル表現** — 1,000種類以上の楽器とスタイルをサポートし、音色の詳細な描写が可能
> - ✅ **多言語対応の歌詞生成** — 50以上の言語に対応し、歌詞プロンプトによる楽曲構成とスタイルの精密な制御が可能
> 
> ### 🎛️ 汎用性と制御機能
> 
> | 機能 | 説明 |
> | --- | --- |
> | ✅ 参照音声入力 | 生成スタイルをガイドするための参照音声を使用可能 |
> | ✅ カバー曲生成 | 既存の音声データからカバーバージョンを作成 |
> | ✅ リペイント＆編集 | 特定部分の音声を編集・再生成する機能 |
> | ✅ トラック分離 | 音声データを個別のステムに分離 |
> | ✅ マルチトラック生成 | Suno Studioの「レイヤー追加」機能のように複数トラックを追加可能 |
> | ✅ Vocal2BGM | ボーカルトラックに自動的に伴奏を生成 |
> | ✅ メタデータ制御 | 再生時間、BPM、調/スケール、拍子記号などを制御 |
> | ✅ 簡易モード | 簡潔な説明からフル楽曲を生成 |
> | ✅ クエリ書き換え | タグや歌詞を自動拡張するAuto LM機能 |
> | ✅ 音声理解 | 音声データからBPM、調/スケール、拍子記号、キャプションを抽出 |
> | ✅ LRC生成 | 生成楽曲に合わせて自動的に歌詞のタイムスタンプを生成 |
> | ✅ LoRA学習 | ワンクリックで注釈付けとGradio上での学習が可能。RTX 3090（12GB VRAM）環境で3090では8曲を1時間で学習 |
> | ✅ 品質評価 | 生成音声の品質を自動評価 |

> ### 利用可能なモデル一覧
> 
> | モデル名 | HuggingFaceリポジトリ | 説明 |
> | --- | --- | --- |
> | **メインモデル** | [ACE-Step/Ace-Step1.5](https://huggingface.co/ACE-Step/Ace-Step1.5) | vae、Qwen3-Embedding-0.6B、acestep-v15-turbo、acestep-5Hz-lm-1.7Bを含むコアコンポーネント |
> | acestep-5Hz-lm-0.6B | [ACE-Step/acestep-5Hz-lm-0.6B](https://huggingface.co/ACE-Step/acestep-5Hz-lm-0.6B) | 軽量言語モデル（パラメータ数0.6B） |
> | acestep-5Hz-lm-4B | [ACE-Step/acestep-5Hz-lm-4B](https://huggingface.co/ACE-Step/acestep-5Hz-lm-4B) | 大規模言語モデル（パラメータ数4B） |
> | acestep-v15-base | [ACE-Step/acestep-v15-base](https://huggingface.co/ACE-Step/acestep-v15-base) | DiTベースモデル |
> | acestep-v15-sft | [ACE-Step/acestep-v15-sft](https://huggingface.co/ACE-Step/acestep-v15-sft) | SFT適用済みDiTモデル |
> | acestep-v15-turbo-shift1 | [ACE-Step/acestep-v15-turbo-shift1](https://huggingface.co/ACE-Step/acestep-v15-turbo-shift1) | turbo版DiTモデル（シフト1設定） |
> | acestep-v15-turbo-shift3 | [ACE-Step/acestep-v15-turbo-shift3](https://huggingface.co/ACE-Step/acestep-v15-turbo-shift3) | turbo版DiTモデル（シフト3設定） |
> | acestep-v15-turbo-continuous | [ACE-Step/acestep-v15-turbo-continuous](https://huggingface.co/ACE-Step/acestep-v15-turbo-continuous) | turbo版DiTモデル（連続シフト1-5設定） |
> 
> ### 💡 どのモデルを選べばいい？
> 
> ACE-StepはGPUのVRAM容量に応じて自動的に最適化されます。以下に簡単な推奨ガイドをご紹介します:
> 
> | GPUのVRAM容量 | 推奨言語モデル | 備考 |
> | --- | --- | --- |
> | **6GB以下** | なし（DiTのみ） | メモリ節約のためデフォルトで言語モデルは無効化 |
> | **6-12GB** | `acestep-5Hz-lm-0.6B` | 軽量ながらバランスの良いモデル |
> | **12-16GB** | `acestep-5Hz-lm-1.7B` | より高品質な出力が可能 |
> | **16GB以上** | `acestep-5Hz-lm-4B` | 最高品質で、音声理解能力も優れている |
> 
> > 📖 **GPU互換性に関する詳細情報** （処理時間制限、バッチサイズ、メモリ最適化方法など）については、GPU互換性ガイドをご覧ください： [英語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en/GPU_COMPATIBILITY.md) | [中国語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/zh/GPU_COMPATIBILITY.md) | [日本語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/ja/GPU_COMPATIBILITY.md)
> 
> ## 🚀 使用方法
> 
> ACE-Stepには複数の利用方法があります：
> 
> | 方法 | 説明 | ドキュメント |
> | --- | --- | --- |
> | 🖥️ **Gradio Web UI** | 音楽生成用のインタラクティブなウェブインターフェース | [Gradioガイド](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en/GRADIO_GUIDE.md) |
> | 🐍 **Python API** | プログラムから利用できる統合用API | [推論API](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en/INFERENCE.md) |
> | 🌐 **REST API** | サービス向けのHTTPベース非同期API | [REST API](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en/API.md) |
> 
> **📚 ドキュメントは以下の言語で利用可能です：** [英語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en) | [中国語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/zh) | [日本語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/ja) \*\* |
> 
> ## 📖 チュートリアル
> 
> **🎯 必読：** ACE-Step 1.5の設計思想と使用方法に関する包括的なガイド。
> 
> | 言語 | リンク |
> | --- | --- |
> | 🇺🇸 英語 | [英語チュートリアル](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en/Tutorial.md) |
> | 🇨🇳 中文 | [中国語](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/zh/Tutorial.md) |
> | 🇯🇵 日本語 | [日本語チュートリアル](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/ja/Tutorial.md) |
> 
> このチュートリアルでは以下の内容を網羅しています：
> 
> - 概念モデルと設計思想
> - モデルアーキテクチャと選択方法
> - 入力制御（テキストおよび音声入力）
> - 推論時のハイパーパラメータ設定
> - ランダム要素と最適化戦略
> 
> ## 🔨 学習
> 
> Gradio UIの **LoRA学習** タブからワンクリックで学習を開始できます。詳細な手順については [Gradioガイド - LoRA学習](https://github.com/ace-step/ACE-Step-1.5/blob/main/docs/en/GRADIO_GUIDE.md#lora-training) をご覧ください。
> 
> ## 🏗️ アーキテクチャ
> 
> ![](https://storage.googleapis.com/zenn-user-upload/c7360471c37d-20260204.png)  
> *referred from [https://github.com/ace-step/ACE-Step-1.5](https://github.com/ace-step/ACE-Step-1.5)*
> 
> ## 🦁 モデルZoo
> 
> ![](https://storage.googleapis.com/zenn-user-upload/537971283416-20260204.png)  
> *referred from [https://github.com/ace-step/ACE-Step-1.5](https://github.com/ace-step/ACE-Step-1.5)*
> 
> ### DiTモデル
> 
> | DiTモデル | Pre-Training | SFT | RL | CFG | Step | 音声参照 | テキスト→音楽 | カバー画像 | 再塗装 | 抽出 | レゴ | 完全モデル | 品質 | 多様性 | ファインチューニング容易性 | Hugging Face |
> | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
> | `acestep-v15-base` | ✅ | ❌ | ❌ | ✅ | 50 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 中 | 高 | 容易 | [リンク](https://huggingface.co/ACE-Step/acestep-v15-base) |
> | `acestep-v15-sft` | ✅ | ✅ | ❌ | ✅ | 50 | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | 高 | 中 | 容易 | [リンク](https://huggingface.co/ACE-Step/acestep-v15-sft) |
> | `acestep-v15-turbo` | ✅ | ✅ | ❌ | ❌ | 8 | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | 非常に高い | 中 | 中 | [リンク](https://huggingface.co/ACE-Step/Ace-Step1.5) |
> | `acestep-v15-turbo-rl` | ✅ | ✅ | ✅ | ❌ | 8 | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | 非常に高い | 中 | 中 | 近日公開予定 |
> 
> ### 言語モデル
> 
> | 言語モデル | 事前学習元 | Pre-Training | SFT | RL | CoTメタ | クエリ書き換え | 音声理解 | 作曲能力 | メロディコピー | Hugging Face |
> | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
> | `acestep-5Hz-lm-0.6B` | Qwen3-0.6B | ✅ | ✅ | ✅ | ✅ | ✅ | 中 | 中 | 弱 | ✅ |
> | `acestep-5Hz-lm-1.7B` | Qwen3-1.7B | ✅ | ✅ | ✅ | ✅ | ✅ | 中 | 中 | 中 | ✅ |
> | `acestep-5Hz-lm-4B` | Qwen3-4B | ✅ | ✅ | ✅ | ✅ | ✅ | 強 | 強 | 強 | ✅ |
> 
> ## 📜 ライセンスおよび免責事項
> 
> 本プロジェクトは [MITライセンス](https://github.com/ace-step/ACE-Step-1.5/blob/main/LICENSE) の下で提供されています。
> 
> ACE-Stepは多様なジャンルのオリジナル音楽生成を可能にし、クリエイティブ制作、教育、エンターテインメント分野での活用が期待されます。本モデルはポジティブで芸術的な用途を支援するよう設計されていますが、スタイルの類似性による意図しない著作権侵害、文化的要素の不適切な融合、有害コンテンツ生成への悪用といった潜在的リスクが存在することを認識しています。責任ある利用を確保するため、ユーザーの皆様には生成作品のオリジナリティを確認すること、AIの関与を明確に開示すること、および保護されたスタイルや素材を改変する場合には適切な許可を取得することを推奨します。ACE-Stepをご利用いただく際には、これら原則を遵守し、芸術的な誠実性、文化的多様性、および法的遵守を尊重することに同意いただくものとします。著者は、本モデルの著作権侵害、文化的配慮の欠如、有害コンテンツ生成など、いかなる誤用についても責任を負いかねますのでご了承ください。
> 
> 🔔 重要なお知らせ  
> ACE-Stepプロジェクトの公式ウェブサイトは、当社のGitHub Pagesサイトのみです。  
> その他のウェブサイトは運営しておりません。  
> 🚫 偽のドメインとして以下のものが確認されています（ただしこれらに限定されません）：  
> ac\*\*p.com、a\*\*p.org、a\*\*\*c.org  
> ⚠️ ご注意ください。これらのサイトへの訪問、信頼、または支払いは一切行わないようお願いいたします。

インストールは以下の方法がある

1. Windows: ポータブルパッケージをダウンロード・展開して実行するだけ（要CUDA-12.8）
2. レポジトリをクローンしてuvでセットアップ（要uv）

どちらもとても簡単になっているように思える。今回は、ローカルのUbuntu-22.04（RTX4090）で試すので後者。

レポジトリクローン

```
git clone https://github.com/ACE-Step/ACE-Step-1.5 && cd ACE-Step-1.5
```

依存パッケージをインストール。uv syncでらくちん。

```
uv sync
```

ではAceStepを動かしていく。以下の3つの方法がある。

- GradioのWeb UI
- REST APIサーバ
- Python API

今回は推奨となっているGradioのWeb UIでまず試してみる。

で、ドキュメントには「モデルは初回実行時に自動でダウンロードされる」とあるのだが、WebUIのインタフェースがわかりにくくて、起動後もちょっとよくわからない。そこで手動でダウンロードする方法が用意されているので、先にダウンロードしてしまう。以下のどちらかを実行すれば良い。

```
# 実行に必要なものだけダウンロード
uv run acestep-download

# オプションも含めたすべてのモデルをダウンロード
uv run acestep-download --all
```

今回は前者を実行した。これで `./checkpoints` ディレクトリが作成され、モデルがダウンロードされる。

出力

```
checkpoints/
├── Qwen3-Embedding-0.6B
│   ├── added_tokens.json
│   ├── chat_template.jinja
│   ├── config.json
│   ├── merges.txt
│   ├── model.safetensors
│   ├── special_tokens_map.json
│   ├── tokenizer.json
│   ├── tokenizer_config.json
│   └── vocab.json
├── README.md
├── acestep-5Hz-lm-1.7B
│   ├── added_tokens.json
│   ├── chat_template.jinja
│   ├── config.json
│   ├── merges.txt
│   ├── model.safetensors
│   ├── special_tokens_map.json
│   ├── tokenizer.json
│   ├── tokenizer_config.json
│   └── vocab.json
├── acestep-v15-turbo
│   ├── config.json
│   ├── configuration_acestep_v15.py
│   ├── model.safetensors
│   ├── modeling_acestep_v15_turbo.py
│   └── silence_latent.pt
├── config.json
└── vae
    ├── config.json
    └── diffusion_pytorch_model.safetensors

4 directories, 27 files
```

次にGradioのWeb UIを起動するが、デフォルトだとアクセスできるのは localhost のみとなっている。以下のオプションがある。

> | オプション | デフォルト値 | 説明 |
> | --- | --- | --- |
> | `--port` | 7860 | サーバーポート番号 |
> | `--server-name` | 127.0.0.1 | サーバーのIPアドレス（ネットワークアクセス可能にするには `0.0.0.0` を使用） |
> | `--share` | false | Gradioの公開リンクを作成 |
> | `--language` | en | UIの表示言語: `en` （英語）、 `zh` （中国語）、 `ja` （日本語） |
> | `--init_service` | false | 起動時にモデルを自動初期化 |
> | `--config_path` | auto | DiTモデルを指定（例: `acestep-v15-turbo`, `acestep-v15-turbo-shift3` ） |
> | `--lm_model_path` | auto | 言語モデルを指定（例: `acestep-5Hz-lm-0.6B`, `acestep-5Hz-lm-1.7B` ） |
> | `--offload_to_cpu` | auto | CPUへのオフロード処理（VRAM容量が16GB未満の場合に自動有効化） |
> | `--enable-api` | false | Gradio UIと並行してREST APIエンドポイントを有効化 |
> | `--api-key` | none | APIエンドポイント認証用のAPIキー |
> | `--auth-username` | none | Gradio認証用のユーザー名 |
> | `--auth-password` | none | Gradio認証用のパスワード |

自分のUbuntuサーバはLAN内のサーバなので、 `--server-name` のオプションが必要。あとはUIを日本語にした。

```
uv run acestep \
    --server-name 0.0.0.0 \
    --language ja
```

ブラウザで7860番ポートにアクセス。ちょっとメニューが多い・・・以下は推論部分のインタフェース。この下にLoRA学習用のインタフェースもあるのだが、そこは割愛。

![](https://storage.googleapis.com/zenn-user-upload/52f5329762ce-20260204.png)

設定も多いのだが、まずはシンプルにテキストから曲を生成するのをやってみる。上から順に見ていく。

まず一番上の「サービス設定」。ここはモデルをロードや、量子化設定など、初期化プロセスを行う箇所。既に最低限必要なモデルはダウンロード済みなので、メインモデルパスなどに値が入っていることを確認して、「サービスを初期化」をクリック。

![](https://storage.googleapis.com/zenn-user-upload/7b090c1ecbf6-20260204.png)

モデルがロードされて、VRAM消費が9GBぐらいとなった。

出力

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        On  |   00000000:01:00.0  On |                  Off |
|  0%   46C    P8             12W /  450W |    9044MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```

次に、生成したい曲の設定を行う。「曲の説明」にテキストで入力。今回は「日本の女性アイドルグループの曲」とした。「サンプル作成」をクリック。

![](https://storage.googleapis.com/zenn-user-upload/581a3dd4b44a-20260204.png)

すると曲のスタイルや歌詞などが生成される。なぜかロシア語で生成されている。

![](https://storage.googleapis.com/zenn-user-upload/753078064b22-20260204.png)

ここで設定を変えて、再度「サンプル作成」をクリックして再生成することもできるし、直接修正することもできる。ボーカル言語を日本語にして、少し説明を追加して、再生すると以下のようになった。

![](https://storage.googleapis.com/zenn-user-upload/468d6c90fa8a-20260204.png)

では曲を生成する。「音楽を生成」をクリック。

![](https://storage.googleapis.com/zenn-user-upload/02897c1216b6-20260204.png)

だいたい30秒ほどで生成された。2パターン生成されているのがわかる。再生ボタンをクリックして確認できる。

![](https://storage.googleapis.com/zenn-user-upload/58417e430492-20260204.png)

実際に生成されたものは以下。

日本語もいける、ただちょっと怪しいところもあるかな。

このスクラップは21日前にクローズされました