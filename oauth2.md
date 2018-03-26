# golang.org/x/oauth2

Go用のoauth2のためのパッケージ。

## OAuthとは

エンドユーザーが自身のデータをサードパーティのアプリケーションに認可するための仕組み。

![OAuthイメージ](https://re.buildinsider.net/enterprise/openid/oauth20/01.gif)

`OAuth Client` が `Resource Owner` に `Authorization Server` を通して `Authorized` され `Resource Server` から `Resource Owner` のデータを利用できる。

## Authorization Code フロー（[The OAuth 2\.0 Authorization Framework](http://openid-foundation-japan.github.io/rfc6749.ja.html#grant-code)）

![Authorization Code フロー](https://re.buildinsider.net/enterprise/openid/oauth20/02.gif)

1. `OAuth Client` の実装者は `Authorization Server` から予め `Client ID`, `Client Secret` を発行してもらっている
1. `Resource Owner` が `OAuth Client` のサービスへアクセスする
1. `OAuth Client` から `Authorization Server` へリダイレクト(`Authorization Endpoint`)する
    * 後ほど使用する `redirect_url` を設定する
    * `scope` パラメーターによって必要な権限を設定する
    * `state` パラメーターを付与することが望ましい
1. `Resource Owner` と `Authorization Server` 間での `認可(Authorization)` の後 `OAuth Client` へリダイレクト(`redirect_url`)する
1. `OAuth Client` は受け取った `Authorization Code` を使用して `Token Endpoint` へアクセスし `Access Token` を発行する
1. `OAuth Client` は発行された `Access Token` を使用し `Resource Server` へアクセスする(WEB API)

### キーワード

* Client ID
* Client Secret
* scope
* state
* Authorization Code
* Access Token
* Authorization Endpoint
* Token Endpoint

## 認証と認可の違い

* 認証(**Auth**eticatio**n** -> Authn)
  * そのユーザーが本人であることを確認する
* 認可(**Auth**ori**z**ation -> Authz)
  * そのユーザーがアクセスできる範囲を確認する

認証をしない認可の例: 年齢制限

```
ここから先は18歳以上がアクセスできます [YES] [NO]
```

## 実装例

cf: <https://github.com/golang/oauth2/blob/master/example_test.go>

```go
package oauth2_test

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"

	"golang.org/x/oauth2"
)

func ExampleConfig() {
	ctx := context.Background()
	conf := &oauth2.Config{
		ClientID:     "YOUR_CLIENT_ID",
		ClientSecret: "YOUR_CLIENT_SECRET",
		Scopes:       []string{"SCOPE1", "SCOPE2"}, // Authorization Serverによって様々
		Endpoint: oauth2.Endpoint{
			AuthURL:  "https://provider.com/o/oauth2/auth",
			TokenURL: "https://provider.com/o/oauth2/token",
		},
	}

	// Redirect user to consent page to ask for permission
	// for the scopes specified above.
	url := conf.AuthCodeURL("state", oauth2.AccessTypeOffline)
	fmt.Printf("Visit the URL for the auth dialog: %v", url)

	// Use the authorization code that is pushed to the redirect
	// URL. Exchange will do the handshake to retrieve the
	// initial access token. The HTTP Client returned by
	// conf.Client will refresh the token as necessary.
	var code string
	if _, err := fmt.Scan(&code); err != nil {
		log.Fatal(err)
	}
	tok, err := conf.Exchange(ctx, code)
	if err != nil {
		log.Fatal(err)
	}

	client := conf.Client(ctx, tok)
	client.Get("...")
}
```

### oauth2.Configの初期化

`oauth2.Config`: OAuthでの連携に必要な情報を格納するための構造体で、連携が便利なメソッドも実装されている。

* `Scope`
    * OAuthプロバイダによって異なる。リファレンスなどで確認すること
* `Endpoint`
    * 著名なプロバイダについては実装済みなのでそれを利用したほうが良い

#### Amazon

cf: <https://github.com/golang/oauth2/blob/master/amazon/amazon.go>

```go
// Endpoint is Amazon's OAuth 2.0 endpoint.
var Endpoint = oauth2.Endpoint{
	AuthURL:  "https://www.amazon.com/ap/oa",
	TokenURL: "https://api.amazon.com/auth/o2/token",
}
```

#### GitHub

cf: <https://github.com/golang/oauth2/blob/master/github/github.go>

```go
// Endpoint is Github's OAuth 2.0 endpoint.
var Endpoint = oauth2.Endpoint{
	AuthURL:  "https://github.com/login/oauth/authorize",
	TokenURL: "https://github.com/login/oauth/access_token",
}
```

#### Google

cf: <https://github.com/golang/oauth2/blob/master/google/google.go>

```go
// Endpoint is Google's OAuth 2.0 endpoint.
var Endpoint = oauth2.Endpoint{
	AuthURL:  "https://accounts.google.com/o/oauth2/auth",
	TokenURL: "https://accounts.google.com/o/oauth2/token",
}

// 他にもいっぱい実装されている
```
