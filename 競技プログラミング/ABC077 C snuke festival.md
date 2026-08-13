#二分探索
### 使う知識


二分探索(lower_bound, upper_bound) 

### 書いたコード
```C++ ctest.cpp
#include <bits/stdc++.h>

#include <atcoder/all>

#define rep(i, n) for (int i = 0; i < (int)(n); i++)

using namespace std;

using namespace atcoder;

using ll = long long;

using ull = unsigned long long;

using ld = long double;

const int mod = 1000000007;

  

int main()

{

    int N;

    cin>>N;

    vector<int> A(N);

    vector<int> B(N);

    vector<int> C(N);

    ll ans=0;

    ll Bcount=0;

    ll Ccount=0;

    rep(i,N)cin>>A[i];

    rep(i,N)cin>>B[i];

    rep(i,N)cin>>C[i];

    sort(A.begin(),A.end());

    sort(B.begin(),B.end());

    sort(C.begin(),C.end());

  

  
  

    for (int i = 0; i < N; i++)

    {

        ans+=(lower_bound(A.begin(),A.end(),B[i])-A.begin())*(N-(upper_bound(C.begin(),C.end(),B[i])-C.begin()));

  

    }

    cout<<ans;

  

    return 0;

}

```
### 学んだこと

互いに関係のある三つの変数について考える時は真ん中を固定するとよい。

要素数からupper_bound(lower_bound)のイテレータを引くと、指定した数よりも大きい(以上)の要素数を求めることができる。