
``` cpp
#include <bits/stdc++.h>

//#include <atcoder/all>

using namespace std;

//using namespace atcoder;

#define rep(i, n) for (int i = 0; i < (int)(n); i++)

using ll = long long;

using ull = unsigned long long;

using ld = long double;

const ll modint = 1000000007;

const ll mint = 998244353;

//using mint = modint998244353;

const ll INF = 1LL<<60;

  

template<typename T> using vc = vector<T>;

template<typename T> using vv = vc<vc<T>>;

  

template<class T> using pq = priority_queue<T, vector<T>>;//大きい順

template<class T> using pq_g = priority_queue<T, vector<T>, greater<T>>;//小さい順

  
  
  

int main()

{

    int V,E;

    cin>>V>>E;

    vv<ll> dist(V+1,vc<ll>(V+1,INF));

    vv<ll> dp(1<<V,vc<ll>(V,INF));

    rep(i,E)

    {

        int A,B;ll C;

        cin>>A>>B>>C;

        dist[A][B]=C;

  

    }

    dp[1][0]=0;

    for (int i = 0; i < 1<<V; i++)

    {

        for (int j = 0; j < V; j++)

        {

            if(dp[i][j]==INF){continue;}

            for (int k = 0; k < V; k++)

            {

                if(dist[j][k]==INF){continue;}

                if(i&1<<k){continue;}

                int next=i|(1<<k);

                dp[next][k]=min(dp[next][k],dp[i][j]+dist[j][k]);

            }

        }

    }

    ll ans=INF;

    for (int i = 0; i < V; i++)

    {

        if(dp[(1<<V)-1][i]==INF){continue;}

        if(dist[i][0]==INF){continue;}

        ans=min(ans,dp[(1<<V)-1][i]+dist[i][0]);

    }

    if(ans>=INF)

    {

        cout<<-1<<endl;

    }

    else

    {

        cout<<ans<<endl;

    }

  
  

    return 0;

	}
```