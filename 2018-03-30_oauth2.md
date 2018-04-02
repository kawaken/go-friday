# golang.org/x/oauth2

Go用のoauth2のためのパッケージ。

## 認証と認可の違い

- 認証(**Auth**eticatio**n** -> Authn)
  - そのユーザーが本人であることを確認する
- 認可(**Auth**ori**z**ation -> Authz)
  - そのユーザーがアクセスできる範囲を確認する

認証をしない認可の例: 年齢制限

```text
ここから先は18歳以上がアクセスできます [YES] [NO]
```

## OAuthとは

エンドユーザーが自身のデータをサードパーティのアプリケーションに利用させるための仕組み。

![OAuthイメージ](https://re.buildinsider.net/enterprise/openid/oauth20/01.gif)

`OAuth Client` が `Resource Owner` に `Authorization Server` を通して `Authorized` され `Resource Server` から `Resource Owner` のデータを利用できるようになる。

## Authorization Code フロー（[The OAuth 2\.0 Authorization Framework](http://openid-foundation-japan.github.io/rfc6749.ja.html#grant-code)）

![Authorization Code フロー](https://re.buildinsider.net/enterprise/openid/oauth20/02.gif)

※ `OAuth Client` の実装者は `Authorization Server` から予め `Client ID`, `Client Secret` を発行してもらっている

1. `Resource Owner` が `OAuth Client` のサービスへアクセスする
1. `OAuth Client` から `Authorization Server` へリダイレクト(`Authorization Endpoint`)する
    - `redirect_url` を設定する
    - `scope` によって必要な権限を設定する
    - `state` を付与することが望ましい
1. `Resource Owner` と `Authorization Server` 間での `認可(Authorization)` の後 `OAuth Client` へリダイレクト(`redirect_url`)する
1. `OAuth Client` は受け取った `Authorization Code` を使用して `Token Endpoint` へアクセスし `Access Token` を発行する
1. `OAuth Client` は発行された `Access Token` を使用し `Resource Server` へアクセスする(WEB API)

### キーワード

- Client ID
  - Clientを識別するID
- Client Secret
  - Clientが保持しているキー情報（Passwordのような扱い）
  - Authorization CodeからAccess Tokenを発行する際に使用する
- redirect_url
  - 認可の後にリダイレクトされるURL
  - Authorization Server側でも設定しておく
- scope
  - Resource Serverからどのような情報を提供するかを定めている識別子
  - サービスによって固有の情報となる
- state
  - csrfを防ぐために使用されるパラメーター
- Authorization Code
  - Resource Ownerが認可を行った際に発行されるコード
  - AccessTokenの取得に使用される
- Access Token
  - Resource Serverが提供しているAPIを利用する際に必要な情報
- Authorization Endpoint
  - Authorization Codeを取得するためにアクセスするURL
- Token Endpoint
  - APIを呼び出したり、AccessTokenを取得する際にアクセスするURL

<https://openid-foundation-japan.github.io/rfc6749.ja.html#grant-code>

## 実装例

<https://github.com/golang/oauth2/blob/master/example_test.go>

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
        Scopes:       []string{"SCOPE1", "SCOPE2"},
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

## oauth2.Configの初期化

`oauth2.Config` にOAuthClientとして必要な情報を設定するための構造体。

### [oauth2.Config](https://github.com/golang/oauth2/blob/master/oauth2.go#L44)

```go
type Config struct {
    // ClientID is the application's ID.
    ClientID string

    // ClientSecret is the application's secret.
    ClientSecret string

    // Endpoint contains the resource server's token endpoint
    // URLs. These are constants specific to each server and are
    // often available via site-specific packages, such as
    // google.Endpoint or github.Endpoint.
    Endpoint Endpoint

    // RedirectURL is the URL to redirect users going through
    // the OAuth flow, after the resource owner's URLs.
    RedirectURL string

    // Scope specifies optional requested permissions.
    Scopes []string
}
```

- `Scopes`
  - OAuthプロバイダによって異なる。リファレンスなどで確認すること

### [oauth2.Endpoint](https://github.com/golang/oauth2/blob/master/oauth2.go#L75)

```go
// Endpoint contains the OAuth 2.0 provider's authorization and token
// endpoint URLs.
type Endpoint struct {
    AuthURL  string
    TokenURL string
}
```

- `Endpoint`
  - 著名なプロバイダについては実装済みなのでそれを利用したほうが良い

#### [Amazon](https://github.com/golang/oauth2/blob/master/amazon/amazon.go)

```go
// Endpoint is Amazon's OAuth 2.0 endpoint.
var Endpoint = oauth2.Endpoint{
    AuthURL:  "https://www.amazon.com/ap/oa",
    TokenURL: "https://api.amazon.com/auth/o2/token",
}
```

#### [GitHub](https://github.com/golang/oauth2/blob/master/github/github.go)

```go
// Endpoint is Github's OAuth 2.0 endpoint.
var Endpoint = oauth2.Endpoint{
    AuthURL:  "https://github.com/login/oauth/authorize",
    TokenURL: "https://github.com/login/oauth/access_token",
}
```

## AuthCodeURLの生成

Authorization Serverの同意画面へリダイレクトするためのURLを生成する。OAuthの仕様上必要なパラメータが自動で設定される。

### [conf.AuthCodeURL](https://github.com/golang/oauth2/blob/master/oauth2.go#L126)

```go
func (c *Config) AuthCodeURL(state string, opts ...AuthCodeOption) string {
    var buf bytes.Buffer
    buf.WriteString(c.Endpoint.AuthURL)
    v := url.Values{
        "response_type": {"code"},
        "client_id":     {c.ClientID},
    }
    if c.RedirectURL != "" {
        v.Set("redirect_uri", c.RedirectURL)
    }
    if len(c.Scopes) > 0 {
        v.Set("scope", strings.Join(c.Scopes, " "))
    }
    if state != "" {
        // TODO(light): Docs say never to omit state; don't allow empty.
        v.Set("state", state)
    }
    for _, opt := range opts {
        opt.setValue(v)
    }
    if strings.Contains(c.Endpoint.AuthURL, "?") {
        buf.WriteByte('&')
    } else {
        buf.WriteByte('?')
    }
    buf.WriteString(v.Encode())
    return buf.String()
}
```

#### state

CSRF対策で使用する。ここで設定したstateの値は `Authorization Server` での認可の後に渡ってくるので、認可の前後で比較する必要がある。

### [oauth2.AuthCodeOption](https://github.com/golang/oauth2/blob/master/oauth2.go#L102)

AuthCodeURLに使用できるオプションのパラメータを表すインターフェース。内部ではsetParamという構造体。

```go
type AuthCodeOption interface {
    setValue(url.Values)
}

type setParam struct{ k, v string }

func (p setParam) setValue(m url.Values) { m.Set(p.k, p.v) }

// SetAuthURLParam builds an AuthCodeOption which passes key/value parameters
// to a provider's authorization endpoint.
func SetAuthURLParam(key, value string) AuthCodeOption {
    return setParam{key, value}
}
```

OAuth2(OpenIDConnect?)の仕様上、使用可能なオプションが規定されている。
<https://github.com/golang/oauth2/blob/master/oauth2.go#L80>

- access_type
  - Access Tokenを更新する際の**ユーザーの状態**
  - `online`, `offline`のどちらか
- approval_prompt
  - 同意画面を強制するかどうか
  - すでに認可済みでも同意画面を出せる
  - `force` を設定する

```go
var (
    // AccessTypeOnline and AccessTypeOffline are options passed
    // to the Options.AuthCodeURL method. They modify the
    // "access_type" field that gets sent in the URL returned by
    // AuthCodeURL.
    //
    // Online is the default if neither is specified. If your
    // application needs to refresh access tokens when the user
    // is not present at the browser, then use offline. This will
    // result in your application obtaining a refresh token the
    // first time your application exchanges an authorization
    // code for a user.
    AccessTypeOnline  AuthCodeOption = SetAuthURLParam("access_type", "online")
    AccessTypeOffline AuthCodeOption = SetAuthURLParam("access_type", "offline")

    // ApprovalForce forces the users to view the consent dialog
    // and confirm the permissions request at the URL returned
    // from AuthCodeURL, even if they've already done so.
    ApprovalForce AuthCodeOption = SetAuthURLParam("approval_prompt", "force")
)
```

## レスポンスの発行と認可後のリクエストの受け取り

このサンプルにはないが、`AuthCodeURL` にリダイレクトする必要がある。

また認可の後にcallbackされるので、そのリクエストを受け取る必要がある。その際に、`state`, `AuthorizationCode`などが渡される。

## Authorization CodeからAccess Tokenの発行

### [config.Exchange](https://github.com/golang/oauth2/blob/master/oauth2.go#L188)

規定されたフローに必要な処理を実施する。

```go
func (c *Config) Exchange(ctx context.Context, code string) (*Token, error) {
    v := url.Values{
        "grant_type": {"authorization_code"},
        "code":       {code},
    }
    if c.RedirectURL != "" {
        v.Set("redirect_uri", c.RedirectURL)
    }
    return retrieveToken(ctx, c, v)
}
```

<https://github.com/golang/oauth2/blob/master/token.go#L153>

```go
func retrieveToken(ctx context.Context, c *Config, v url.Values) (*Token, error) {
    tk, err := internal.RetrieveToken(ctx, c.ClientID, c.ClientSecret, c.Endpoint.TokenURL, v)
    if err != nil {
        if rErr, ok := err.(*internal.RetrieveError); ok {
            return nil, (*RetrieveError)(rErr)
        }
        return nil, err
    }
    return tokenFromInternal(tk), nil
}
```

<https://github.com/golang/oauth2/blob/master/internal/token.go#L175>

```go
func RetrieveToken(ctx context.Context, clientID, clientSecret, tokenURL string, v url.Values) (*Token, error) {
    // providerAuthHeaderWorks: OAuth2の仕様に準拠しているかチェック
    // bustedAuth: OAuth2の仕様に準拠していない => true
    bustedAuth := !providerAuthHeaderWorks(tokenURL)

    // OAuth2の仕様ではBasic認証を使用しないといけないが、対応していないのでリクエストパラメータに含める
    // https://openid-foundation-japan.github.io/rfc6749.ja.html#client-password
    if bustedAuth {
        if clientID != "" {
            v.Set("client_id", clientID)
        }
        if clientSecret != "" {
            v.Set("client_secret", clientSecret)
        }
    }
    req, err := http.NewRequest("POST", tokenURL, strings.NewReader(v.Encode()))
    if err != nil {
        return nil, err
    }
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    // OAuth2の仕様を満たしているので、Basic認証を用いる
    if !bustedAuth {
        req.SetBasicAuth(url.QueryEscape(clientID), url.QueryEscape(clientSecret))
    }
    r, err := ctxhttp.Do(ctx, ContextClient(ctx), req)
    if err != nil {
        return nil, err
    }
    defer r.Body.Close()
    body, err := ioutil.ReadAll(io.LimitReader(r.Body, 1<<20)) // 上限1Mbyteくらい
    if err != nil {
        return nil, fmt.Errorf("oauth2: cannot fetch token: %v", err)
    }
    // ステータスコードが200系じゃなかったらエラーとする
    // providerAuthHeaderWorksでチェックされていないが、実際には準拠してないサーバーなどはここで403とかなりそう
    if code := r.StatusCode; code < 200 || code > 299 {
        return nil, &RetrieveError{
            Response: r,
            Body:     body,
        }
    }

    var token *Token
    content, _, _ := mime.ParseMediaType(r.Header.Get("Content-Type"))
    switch content {
    case "application/x-www-form-urlencoded", "text/plain":
        vals, err := url.ParseQuery(string(body))
        if err != nil {
            return nil, err
        }
        token = &Token{
            AccessToken:  vals.Get("access_token"),
            TokenType:    vals.Get("token_type"),
            RefreshToken: vals.Get("refresh_token"),
            Raw:          vals,
        }
        e := vals.Get("expires_in")
        if e == "" {
            // TODO(jbd): Facebook's OAuth2 implementation is broken and
            // returns expires_in field in expires. Remove the fallback to expires,
            // when Facebook fixes their implementation.
            e = vals.Get("expires")
        }
        // OAuth2の仕様上 expires_in で有効期間(秒数)をレスポンスすることが推奨されているが、
        // パッケージ内の実装としては、日時を Expiry に持たせるようにしている
        expires, _ := strconv.Atoi(e)
        if expires != 0 {
            token.Expiry = time.Now().Add(time.Duration(expires) * time.Second)
        }
    default:
        var tj tokenJSON
        if err = json.Unmarshal(body, &tj); err != nil {
            return nil, err
        }
        token = &Token{
            AccessToken:  tj.AccessToken,
            TokenType:    tj.TokenType,
            RefreshToken: tj.RefreshToken,
            Expiry:       tj.expiry(),
            Raw:          make(map[string]interface{}),
        }
        json.Unmarshal(body, &token.Raw) // no error checks for optional fields
    }
    // Don't overwrite `RefreshToken` with an empty value
    // if this was a token refreshing request.
    if token.RefreshToken == "" {
        token.RefreshToken = v.Get("refresh_token")
    }
    if token.AccessToken == "" {
        return token, errors.New("oauth2: server response missing access_token")
    }
    return token, nil
}
```

### [oauth2.Token](https://github.com/golang/oauth2/blob/master/token.go#L31)

AccessTokenなどAPIの呼び出しに必要な情報を持った構造体。

```go
type Token struct {
    // AccessToken is the token that authorizes and authenticates
    // the requests.
    AccessToken string `json:"access_token"`

    // TokenType is the type of token.
    // The Type method returns either this or "Bearer", the default.
    TokenType string `json:"token_type,omitempty"`

    // RefreshToken is a token that's used by the application
    // (as opposed to the user) to refresh the access token
    // if it expires.
    RefreshToken string `json:"refresh_token,omitempty"`

    // Expiry is the optional expiration time of the access token.
    //
    // If zero, TokenSource implementations will reuse the same
    // token forever and RefreshToken or equivalent
    // mechanisms for that TokenSource will not be used.
    Expiry time.Time `json:"expiry,omitempty"`

    // raw optionally contains extra metadata from the server
    // when updating a token.
    raw interface{}
}
```

<https://github.com/golang/oauth2/blob/master/token.go#L123>

```go
func (t *Token) expired() bool {
	if t.Expiry.IsZero() {
		return false
	}
	return t.Expiry.Round(0).Add(-expiryDelta).Before(time.Now())
}

func (t *Token) Valid() bool {
	return t != nil && t.AccessToken != "" && !t.expired()
}
```

Tokenが有効かどうかをチェックする。

## 受け取ったAccessTokenを用いたClientの生成

### [config.Client](https://github.com/golang/oauth2/blob/master/oauth2.go#L203)

```go
func (c *Config) Client(ctx context.Context, t *Token) *http.Client {
	return NewClient(ctx, c.TokenSource(ctx, t))
}
```

### [oauth2.NewClient](https://github.com/golang/oauth2/blob/master/oauth2.go#L314)

```go
func NewClient(ctx context.Context, src TokenSource) *http.Client {
	if src == nil {
		return internal.ContextClient(ctx)
	}
	return &http.Client{
		Transport: &Transport{
			Base:   internal.ContextClient(ctx).Transport,
			Source: ReuseTokenSource(nil, src),
		},
	}
}
```

### [oauth2.TokenSource](https://github.com/golang/oauth2/blob/master/oauth2.go#L66)

```go
type TokenSource interface {
	// Token returns a token or an error.
	// Token must be safe for concurrent use by multiple goroutines.
	// The returned Token must not be modified.
	Token() (*Token, error)
}
```

### [c.TokenSource](https://github.com/golang/oauth2/blob/master/oauth2.go#L211)

AccessTokenが無効になった場合に、自動的に新しいAccessTokenを発行できるTokenSourceを生成する。

```go
func (c *Config) TokenSource(ctx context.Context, t *Token) TokenSource {
	tkr := &tokenRefresher{
		ctx:  ctx,
		conf: c,
	}
	if t != nil {
		tkr.refreshToken = t.RefreshToken
	}
	return &reuseTokenSource{
		t:   t,
		new: tkr,
	}
}
```

<https://github.com/golang/oauth2/blob/master/oauth2.go#L227>

```go
type tokenRefresher struct {
	ctx          context.Context // used to get HTTP requests
	conf         *Config
	refreshToken string
}

func (tf *tokenRefresher) Token() (*Token, error) {
	if tf.refreshToken == "" {
		return nil, errors.New("oauth2: token expired and refresh token is not set")
	}

    // AccessTokenを更新する際に必要なパラメータを設定
    // https://openid-foundation-japan.github.io/rfc6749.ja.html#token-refresh
	tk, err := retrieveToken(tf.ctx, tf.conf, url.Values{
		"grant_type":    {"refresh_token"},
		"refresh_token": {tf.refreshToken},
	})

	if err != nil {
		return nil, err
	}
	if tf.refreshToken != tk.RefreshToken {
		tf.refreshToken = tk.RefreshToken
	}
	return tk, err
}
```

<https://github.com/golang/oauth2/blob/master/oauth2.go#L260>

```go
type reuseTokenSource struct {
	new TokenSource // called when t is expired.

	mu sync.Mutex // guards t
	t  *Token
}

func (s *reuseTokenSource) Token() (*Token, error) {
	s.mu.Lock()
    defer s.mu.Unlock()
    // AccessTokenが問題なければそれを使用する
	if s.t.Valid() {
		return s.t, nil
    }
    // tokenRefresherが使用され、新しいAccessTokenが発行される
	t, err := s.new.Token()
	if err != nil {
		return nil, err
	}
	s.t = t
	return t, nil
}
```

### [oauth2.Transport](https://github.com/golang/oauth2/blob/master/transport.go#L20)

```go
type Transport struct {
	// Source supplies the token to add to outgoing requests'
	// Authorization headers.
	Source TokenSource

	// Base is the base RoundTripper used to make HTTP requests.
	// If nil, http.DefaultTransport is used.
	Base http.RoundTripper

	mu     sync.Mutex                      // guards modReq
	modReq map[*http.Request]*http.Request // original -> modified
}
```

OAuth2用のTransportで、Authorization Headerの設定やトークンの更新を行う。

- Base
  - 元になるRoundTripper
- Source
  - Authorization Headerの設定に使用するためのTokenSource

### [oauth2.ReuseTokenSource](https://github.com/golang/oauth2/blob/master/oauth2.go#L338)

```go
func ReuseTokenSource(t *Token, src TokenSource) TokenSource {
	// Don't wrap a reuseTokenSource in itself. That would work,
	// but cause an unnecessary number of mutex operations.
	// Just build the equivalent one.
	if rt, ok := src.(*reuseTokenSource); ok {
		if t == nil {
			// Just use it directly.
			return rt
		}
		src = rt.new
	}
	return &reuseTokenSource{
		t:   t,
		new: src,
	}
}
```
