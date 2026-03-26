---
title: "自力でaiを作る！！！【python】"
date: 2026年3月7日
tags: [saved]
source: "Zennの「ディープラーニング」のフィード"
link: https://zenn.dev/free_eerf/articles/4606b98b86537e
author: ""
feedTitle: "Zennの「ディープラーニング」のフィード"
guid: "https://zenn.dev/free_eerf/articles/4606b98b86537e"
---

# 自力でaiを作る！！！【python】

はじめに
----

こんにちは  
今回はnumpyを主軸にtorchなどを使わずに自力で気温予測ai（深層学習）を作ったのでそれについて書きます（数式が出ても逃げないでね）

まずai（深層学習）の仕組みとは
----------------

深層学習のaiは基本的に複数の**ニューロン**（変換装置みたいなもの）によって作られています  
↓イメージ図　丸いのがニューロン  
![](https://zenn.dev/zenn-user-upload/a06f1fd96fda-20260301.jpg)  
このニューロンは入力層、隠れ層、出力層の３つ種類に分けられます  
そしてこのニューロンとニューロンの間の線には一つ一つ**重み**と**バイアス**という数値があります（重みとバイアスの初期値はランダムに設定されていることが多い）  
流れとしては  
データを入力層のニューロンに入れる  
↓  
入力層を通ったデータを隠れ層に入れる  
↓  
隠れ層を通ったデータを出力層に入れる  
↓  
それを出力結果として返す  
です

そしてこの各層を通るときに行われるニューロンの計算が

y = wx+b

です  
この式はx（入れたデータ）とw（重み）をかけてそれにb（バイアス）を足したものがy（ニューロンの出力）という意味です  
これがデータがニューロンを通るたび行われているのです

学習
--

でもこれだけじゃ物事を予想するための学習方法が不明ですよね

まずaiの学習（強化学習などの深層学習以外の場合は別）は、データを覚えさせることです  
ここで私たちが答えが明確にある教科の勉強するときどうしてるかを考えます  
勿論答えが明確にあるので、前学んだことを思い出しながら問題を解いて答え合わせをします  
aiも似たような感じです  
さっき説明したニューロンなどを使って計算をして結果を出します  
そして答え合わせをして間違えを学びます  
このようにaiは学ぶのですが、これを計算で行うときに出てくるのが

**損失関数**と**微分**です

aiの学習手順は  
問題をさっき説明したニューロンなどを使って計算  
↓  
結果と答えの差（間違い）の大きさを損失関数を使って計算する  
↓  
その計算結果を微分して傾きがマイナスだったらパラメーターを増やし、プラスだったら減らす

この繰り返しです

でもちょっと最後の手順がわかりにくいので説明すると  
例えば損失関数のグラフが二次関数のような曲線になっているとします  
損失関数は間違えが大きいければ大きいほど大きい値を返すので、当然間違えが少ない、二次関数の一番凹んでいるところ（微分したら傾きが0のところ）を目指すべきです

パラメーターの重みやバイアスはどっちもかけたり足したりするので大きくなったら結果の値も大きくなるので、結果が答えより小さければパラメーターを増やし、大きければ減らす必要があります  
この増やしたほうがいいのか、減らしたほうがいいのかを判断するときに微分が出てきます

微分はある点の傾きを求めることなので、損失関数の結果から微分して、傾きがプラス（右肩上がりのグラフ）だったらそれはパラメーターが行き過ぎて答えを通り過ぎていることになるので減らす、マイナス（右肩下がり）だったらパラメーターが少ないということなので増やす、という判断ができるのです

これを繰り返し、パラメーターを調整し学習をしていくのがai（深層学習）なのです

コードの解説
------

コードの流れは  
import  
↓  
データの読み込み、前処理  
↓  
使う関数を定義  
↓  
モデルのクラスを定義（ニューロンがたくさん入ってるai本体みたいなもの）  
↓  
学習  
↓  
テスト、様々なデータをプロット  
↓  
コマンドラインで気温予測（毎回ctrl+Cするのめんどくさいからコメントアウトされてる）  
です  
順を追って説明しましょう

import
------

    # import
    import re
    from datetime import date
    import pandas as pd
    import numpy as np
    import matplotlib.pyplot as plt
    

ここはただreとdatetimeのdateとpandasとnumpyとpyplotを持って来てるだけですね

データの読み込み、前処理
------------

    # データの読み込み
    learn_df = pd.read_csv("learn_data.csv", encoding="utf-8")
    test_df = pd.read_csv("test_data.csv", encoding="utf-8")
    
    # データの前処理
    # (2192, 6)
    input_data = learn_df[["Date", "MaximumTemperature(C)", "MinimumTemperature(C)", "AverageTemperature(C)", "TotalPrecipitation(mm)", "AverageWindSpeed(m/s)"]].values
    # (2192, 2)
    label_data = learn_df[["MaximumTemperature(C)", "MinimumTemperature(C)"]].values
    # (1097, 6)
    test_input_data = test_df[["Date", "MaximumTemperature(C)", "MinimumTemperature(C)", "AverageTemperature(C)", "TotalPrecipitation(mm)", "AverageWindSpeed(m/s)"]].values
    # (1097, 2)
    test_label_data = test_df[["MaximumTemperature(C)", "MinimumTemperature(C)"]].values
    
    # cosの周期に日付を変換
    def date_to_cos(date_str):
        # 日付文字列から年、月、日を抽出
        match = re.match(r"(\d+)/(\d+)/(\d+)", date_str)
        if match:
            year, month, day = map(int, match.groups())
            first_date = date(year, 1, 1)
            current_date = date(year, month, day)
            day_of_year = (current_date - first_date).days + 1  # 1月1日を1日目とする
            cos_value = (np.cos(2 * np.pi * day_of_year / 365) + 1) / 2  # 正規化して周期性を考慮
            return float(cos_value)  # cos値を返す
        else:
            return float(0.5)  # 日付形式が不正な場合は0.5を返す
        
    # 日付をcos値に変換して入力データを更新
    for i in range(len(input_data)):
        input_data[i][0] = date_to_cos(input_data[i][0])
    for i in range(len(test_input_data)):
        test_input_data[i][0] = date_to_cos(test_input_data[i][0])
    
    # データをfloat64に変換
    input_data = input_data.astype(np.float64)
    test_input_data = test_input_data.astype(np.float64)
    label_data = label_data.astype(np.float64)
    test_label_data = test_label_data.astype(np.float64)
    

最初learn\_dfとtest\_dfの定義で学習用のとテスト用のcsvファイルをdataframeとして読み込んで、  
必要なデータを取り出し入力データと答え（label）に分けています  
次にdate\_to\_cos関数を定義しています  
dateにcos関数を通させる関数です  
なぜdateにcosを通すのかというと、aiは1月1日の1というデータと12月31日の365というデータを全然違うデータとして扱ってしまうから、コサインカーブで滑らかにさせる必要があったからです  
そしてそれをdateに適用させ、その後型を合わせている感じです

使う関数を定義
-------

    # Leaky ReLU
    def leaky_relu(x, alpha=0.01):
        return np.where(x > 0, x, alpha * x)
    
    # 微分
    def leaky_relu_derivative(x, alpha=0.01):
        return np.where(x > 0, 1, alpha)
    
    # MSE（平均二乗誤差）損失関数
    def mse(y_true, y_pred):
        return np.mean((y_true - y_pred) ** 2)
    
    # MSEの微分
    def mse_derivative(y_true, y_pred):
        n = y_true.size
        return 2 * (y_pred - y_true) / n
    

ここはめちゃくちゃ数式が出てきますが逃げないでください  
この４つの関数を解説します

### leaky\_relu(x)

これは通称Leaky ReLU関数で、数式で表すと

\\mathrm{LeakyReLU}(x) = \\begin{cases} x & (x > 0) \\\\ \\alpha x & (x \\le 0) \\end{cases}

です  
0を超えていたらはそのままのx、0以下だったらアルファとかけたxを返すという関数です  
今回の場合はこのアルファは0.01と設定されています  
アルファxのところが0のReLU関数というのがありそれの進化系です  
ReLU関数の問題点として0以下だったとき学習されないというものがありそれの解決策としてできた関数です  
ReLU関数はランプ関数とも呼びます  
のちのち活性化関数として出てきます

### leaky\_relu\_derivative(x)

これはさっきのLeaky ReLU関数の導関数、微分で、数式で表すと

\\frac{d}{dx}\\mathrm{LeakyReLU}(x) = \\begin{cases} 1 & (x > 0) \\\\ \\alpha & (x \\le 0) \\end{cases}

です  
aiの仕組みで出てきたように微分がまたのちのち出てきます  
それに関係してます

### mse(y\_true, y\_pred)

これは損失関数で、数式に表すと

\\mathrm{MSE} = \\frac{1}{n} \\sum\_{i=1}^{n} (y\_i - \\hat{y}\_i)^2

です  
yは答え、yの上に^みたいのがついてるのが予測値です  
答えから予測値を引き、それを二乗することによってマイナスを無くし、大きい誤差はもっと大きくなるので深刻に受け止めて（？）くれます  
しかし大きい誤差をもっと大きくすることは、外れ値（とても数値が低かったり高かったりして参考にすべきではない答え）に影響を受けやすいということなので、注意が必要です  
損失関数は外れ値対策として二乗せずに絶対値でマイナスを外すもの（MAE）や、huber損失（MAEとMSEのハイブリッドみたいな感じ）、クロスエントロピー誤差（分類系に向いてる）など色々あるので調べてみると面白いです

### mse\_derivative(y\_true, y\_pred)

これはさっきのMSEの導関数、微分で、数式に表すと

\\frac{\\partial \\mathrm{MSE}}{\\partial \\hat{y}\_i} = \\frac{2}{n}(\\hat{y}\_i - y\_i)

です  
aiの仕組みで出てきたように微分がまたのちのち出てきます  
それに関係してます

モデルのクラスを定義
----------

    # モデルを定義
    class WeatherPredictor:
        def __init__(self):
            # 重みの初期化
            self.weights_input_hidden = np.random.rand(6, 16) * 0.01 # 入力層から隠れ層への重み
            self.weights_hidden_output = np.random.rand(16, 2) * 0.01 # 隠れ層から出力層への重み
            self.bias_hidden = np.zeros((1,16)) # 隠れ層のバイアス
            self.bias_output = np.zeros((1,2)) # 出力層のバイアス
        
        def forward(self, x):
            # 順伝播
            self.hidden_input = np.dot(x, self.weights_input_hidden) + self.bias_hidden
            self.hidden_output = leaky_relu(self.hidden_input)
            self.final_input = np.dot(self.hidden_output, self.weights_hidden_output) + self.bias_output
            return self.final_input
        
        def backward(self, x, y_true, y_pred, learning_rate):
            # 出力層の誤差
            output_error = mse_derivative(y_true, y_pred)
            output_delta = output_error
    
            # 隠れ層の誤差
            hidden_error = np.dot(output_delta, self.weights_hidden_output.T)
            hidden_delta = hidden_error * leaky_relu_derivative(self.hidden_input)
    
            # 重みとバイアスの更新
            self.weights_hidden_output -= np.dot(self.hidden_output.T, output_delta) * learning_rate
            self.bias_output -= np.sum(output_delta, axis=0, keepdims=True) * learning_rate
            self.weights_input_hidden -= np.dot(x.T, hidden_delta) * learning_rate
            self.bias_hidden -= np.sum(hidden_delta, axis=0, keepdims=True) * learning_rate
    

ここはai本体のクラスを定義してます  
**一番重要なところです**  
まず\_\_init\_\_から見ていきましょう  
weights\_input\_hiddenは入力層から隠れ層の間、weights\_hidden\_outputは隠れ層から出力層の間のニューロンの間の線の重みです  
どっちもrand関数に0.01をかけて大きさを調整して初期値を設定しています  
入力層は日付と最高気温と最低気温と平均気温と降水量と平均風速なので、形は(1,6)です  
なのでweights\_input\_hiddenはそれにドット積できる形でなければいけません  
なので(1,6)とドット積が可能な(6,16)という形になっています  
weights\_hidden\_outputも同じ理由です  
(1,16)の形をした隠れ層に対応するために(16,2)なのです（2は最低気温と最高気温の2つの値を予測するaiなので2）  
バイアスは初期値は0に設定しています  
bias\_hiddenは入力層から隠れ層の間の計算のあとに足されるb（バイアス）なので隠れ層の形の(1,16)と同じ形にしてあります  
bias\_outputは隠れ層から出力層の間の計算のあとに足されるb（バイアス）なので出力層の形の(1,2)と同じ形にしてあります  
※行列の計算がわからなかったら調べてください

次にforward、準伝播です

一行目はhidden\_inputに、隠れ層の`y=wx+b`の計算を代入しています  
そして次です  
reluにhidden\_input入れ計算させhidden\_output（隠れ層の出力）に代入しています  
なぜleaky\_reluを通すのか説明すると  
まずleaky\_reluは活性化関数と言ってニューロンの計算の最後に使われる関数です  
活性化関数はニューロンの中で計算した結果を**どのように出力するか**を決めることができます  
活性化関数の中で役割がわかりやすいのは↓のシグモイド関数というものなんですが、

\\sigma(x) = \\frac{1}{1 + e^{-x}}

このシグモイド関数はxが0のときはyは0.5、xが大きければ大きいほどyは1に近づき、小さければ小さいほどyは0に近づくという特性を持ちます  
そして0~1の間が範囲なので、この特性を活かせる物事は、0か1、はいかいいえ、のような２つに分けられる物事になります  
このように活性化関数ごとには特性があり、適した物事に適した活性化関数を使うことが大事になってきます  
そしてこのLeaky ReLU関数は、計算が早く、勾配消失が起きにくいという特性を持っています  
※勾配消失とは損失関数の微分がほぼ0になってしまいもっと良い最適解がある状況でも学習が進まなくなる現象のことです  
なので隠れ層に向いているため、Leaky ReLU関数を採用しています  
ちなみに活性化関数の他の説明方法では、活性化関数を通すことによって直線で分類されていた物事が曲線で分類できるようになる、のようなものもありました

そして最後、final\_inputに出力層のニューロンの`y=wx+b`の計算が代入され、それを返しています  
※ここでReLU関数に通してしまうと気温がマイナスのとき0が返ってしまうためそのまま返す

最後にbackward、逆伝播です

まず逆伝播とは何かですが、  
aiの説明の図は左から右に入力層、隠れ層、出力層とありました  
しかし逆伝播は逆で、右から左に計算されるのです  
よって出力層、隠れ層、入力層という流れで計算されるのが逆伝播なのです  
でコードの内容ですが  
コードの内容を説明をするのは難しいので、大まかに内容を簡単に言うと  
「出力層、隠れ層、入力層の順番で誤差を逆算してそれを重みとバイアスに反映してる」  
コードです  
そしてその時に、考え方は微分してその傾きからパラメーターの変更することですから、  
微分を使います微分した活性化関数、微分した損失関数を使うのです  
ちなみにlearning\_rateは重みとバイアスに与える影響の大きさを表す**学習率**です  
結構大事です  
これによってはaiの精度に天と地の差がつきます

学習
--

    wp = WeatherPredictor()
    
    # 学習
    losses = []
    # 学習率スケジューラーをするために初期学習率を設定
    learning_rate = 0.00002
    # 学習率の記録
    learning_rates = []
    
    for epoch in range(2192 - 1): # ラベルの+1を考慮して-1 2192 - 1 = 2191
        input_data_i = input_data[epoch].reshape(1, -1) # (1, 6)
        label_data_i = label_data[epoch+1].reshape(1, -1) # (1, 2)
    
        # 順伝播
        output = wp.forward(input_data_i)
        # 誤差逆伝播
        wp.backward(input_data_i, label_data_i, output, learning_rate)
    
        if np.isnan(wp.weights_input_hidden).any() or np.isnan(wp.weights_hidden_output).any():
            print(f"NaN detected at epoch {epoch + 1}")
            print(f"Weights Input-Hidden: {wp.weights_input_hidden}, Weights Hidden-Output: {wp.weights_hidden_output}, Bias Hidden: {wp.bias_hidden}, Bias Output: {wp.bias_output}")
            break
    
        learning_rates.append(learning_rate)
    
        # 100エポックごとに損失を表示
        if (epoch + 1) % 100 == 0:
            loss = mse(label_data_i, output)
            losses.append(loss)
            learning_rate *= 0.7 # 学習率を減衰させる
            print(f"Epoch {epoch + 1}, Loss: {loss}, Output: {output}, Label: {label_data_i}, Input: {input_data_i}")
    

最初wpにさっきのモデルを入れています  
そして学習に使う変数を定義しています  
lossesは損失関数の結果を入れて最後グラフにするためにあります  
learning\_rateは学習率なのですが、学習率というのは学習の終盤につれて小さくなって行くことが理想です  
なぜなら、最初は高速で答えのところに辿り着きたいのに対して、終盤になると小さい誤差の調整をしてほしいからです  
そして変数として定義しておくと学習内で減らすことが簡単なので定義しています  
learning\_ratesは学習率を記録して学習率の減り方を最後グラフにするためにあります

そして次、学習です

for文でepochをデータの数分回しています  
epochは学習の回数のことです  
for文内では最初、学習用の入力データと答えのラベルデータをepochで指定しreshapeで形を調整しています  
その次順伝播、左から右に入力層、隠れ層、出力層の順番で処理し、  
その結果を逆伝播の関数に渡し重みとバイアスを調整させています  
`if np.isnan`のあたりは万が一データにNaNがあったとき場所をすぐ見つけられるようにするためにあります  
そしてlearning\_ratesにlearning\_rateを入れ記録しています  
次に100エポック（epochが100）ごとに損失を表示しています  
mseでlabel\_data\_iとoutputの誤差を計算しそれを記録し、ついでにlearning\_rate（学習率）を減衰させ、エポックやlossなど色々なものを表示させています

テスト
---

    # 結果を表示
    outputs = np.zeros((1097 - 1, 2)) # (1096, 2)
    
    for i in range(1097 - 1):
        test_input_data_i = test_input_data[i].reshape(1, -1) # (1, 6)
        test_label_data_i = test_label_data[i].reshape(1, -1) # (1, 2)
        output = wp.forward(test_input_data_i)
        outputs[i] = output.flatten()
    
    plt.figure(figsize=(10,5))
    plt.plot(outputs[:, 0], label="Predicted Max Temp")
    plt.plot(outputs[:, 1], label="Predicted Min Temp")
    plt.plot(test_label_data[:, 0], label="Actual Max Temp", linestyle="--")
    plt.plot(test_label_data[:, 1], label="Actual Min Temp", linestyle="--")
    plt.xlabel("Day")
    plt.ylabel("Temperature (C)")
    plt.title("Predicted vs Actual Temperatures")
    plt.legend()
    plt.show()
    
    plt.figure(figsize=(10,5))
    plt.plot(losses, label="Training Loss")
    plt.xlabel("Epoch")
    plt.ylabel("Loss")
    plt.title("Training Loss Over Time")
    plt.legend()
    plt.show()
    
    plt.figure(figsize=(10,5))
    plt.plot(learning_rates, label="Learning Rate")
    plt.xlabel("Epoch")
    plt.ylabel("Learning Rate")
    plt.title("Learning Rate Over Time")
    plt.legend()
    plt.show()
    

最初結果を入れる配列を定義し、for文で結果を入れています  
その次答えの気温と予測の気温を比べるグラフを表示させ、損失のグラフ、学習率の動きの順番で表示しています

↓プロットの結果  
(特定を避けるため比較は載せられないが四つの線が重なっているイメージ)  
![](https://zenn.dev/zenn-user-upload/868f61a377a9-20260306.png)  
![](https://zenn.dev/zenn-user-upload/822c1d42eea2-20260306.png)

コマンドラインで気温予測
------------

    # print("明日の気温の予測をします。数値を入力してください。単位は省略してください。")
    # d = input("日付 (YYYY/MM/DD): ")
    # d = date_to_cos(d)
    # max_temp = float(input("最高気温 (C): "))
    # min_temp = float(input("最低気温 (C): "))
    # avg_temp = float(input("平均気温 (C): "))
    # precipitation = float(input("降水量 (mm): "))
    # wind_speed = float(input("平均風速 (m/s): "))
    # input_data_tomorrow = np.array([[d, max_temp, min_temp, avg_temp, precipitation, wind_speed]])
    # predicted_tomorrow = wp.forward(input_data_tomorrow)
    
    # print(f"明日の予測 - 最高気温: {predicted_tomorrow[0, 0]:.2f} C, 最低気温: {predicted_tomorrow[0, 1]:.2f} C")
    

（毎回ctrl+Cするのめんどくさいからコメントアウトされてる）と最初書いたのでコメントアウトされてますが、コードを説明します  
これはinputで日付、最高気温、最低気温、平均気温、降水量、平均風速を入力させそれを学習済みのモデルに計算させその予測を表示させてます  
日付はそのまま読み込めないのでdata\_to\_cosしてます

おわりに
----

ai用のライブラリなしでai作るとaiの構造がわかってきやすいです  
ニューロンの数、損失関数、活性化関数、学習率など結構aiはいじれるところが沢山あるので楽しいです  
ぜひaiに予測させたいことあったら自分でnumpyとかのみでコード書いてみてください

さようなら

[Source](https://zenn.dev/free_eerf/articles/4606b98b86537e)