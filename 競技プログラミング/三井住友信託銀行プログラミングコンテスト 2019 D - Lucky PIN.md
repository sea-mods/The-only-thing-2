#全探索

工夫して全探索をする問題。

### 問題文

AtCoder 社は、オフィスの入り口に $3$ 桁の暗証番号を設定することにしました。

AtCoder 社には $N$ 桁のラッキーナンバー $S$ があります。社長の高橋君は、$S$ から $N-3$ 桁を消して残りの $3$ 桁を左から読んだものを暗証番号として設定することにしました。

このとき、設定されうる暗証番号は何種類あるでしょうか？

ただし、ラッキーナンバーや暗証番号はいずれも $0$ から始まっても良いものとします。
### 制約

- $4 \leq N \leq 30000$
- $S$ は半角数字からなる長さ $N$ の文字列

単純にSに対して全探索を行うと30000^3 = 27,000,000,000,000回行う必要があり、TLEしてしまう。
暗証番号になりえる数の組み合わせは最大で10^3=1000通りなので、そこから考える。
暗証番号の組み合わせをVとすると、Vの各桁に対して、Sを左から走査して貪欲法をする。

Vの値の取り方はリストを作り、商や余剰をとって１桁ずつ格納する

(例)
'''c++
C[3]={i / 100, (i / 10) % 10, i % 10};
'''

### 自分が書いたコード

```C++ ctest.cpp
#include <bits/stdc++.h>

#include <atcoder/all>

#define rep(i, n) for (int i = 0; i < (int)(n); i++)

using namespace std;

using namespace atcoder;

using ll = long long;

using ull = unsigned long long;

const int mod = 1000000007;

  

int main()

{

    int N;

    string S;

    int ans=0;

    cin>>N>>S;

  

    for (int i = 0; i < 1000; i++)

    {

        int count=0;

        int C[3]={i/100,(i/10)%10,(i%100)%10};

        for (int j = 0; j < N; j++)

        {

            if(S[j]=='0'+C[count])

            {

                count++;

            }

            if(count==3)

            {

                break;

            }

        }

        if(count==3)

        {
		    ans++;
		}

    }
    cout<<ans;

    return 0;

}
```



### 学んだこと

ある数に対して一桁ずつ抜き出すにはStringを使う以外にも計算で各行を抜き出してリストに入れる方法がある。


### その他

DPを使った解法もある。


