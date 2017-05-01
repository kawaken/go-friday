# net/http #1

```go
func hello_world(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World")
}

func main() {

	http.HandleFunc("/", hello_world)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal("ListenAndServe: ", err)
	}

}
```

今回は `http.HandleFunc` の辺り

## type `Handler`

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

https://golang.org/src/net/http/server.go#L81

* interface
* `ServeHTTP(ResponseWriter, *Request)` を持つ
* `ServeHTTP` は `reply headers` と `data` を `ResponseWriter` に書き込んでから `return` すべき
* `return` がリクエストが終了した合図になる
  * `ServeHTTP` が終了するときや終了した後に、 `ResponseWriter` を利用したり `Request.Body` を読んだりすることは有効ではない
* `ResponseWriter` に書き込む前に `Request.Body` を読み取ること
* `server` は `Handler` が `panic` を起こしたら `recover` してリクエストを中断するが `ErrAbortHandler` を使用して `panic` を起こす

## type `HandlerFunc`

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

https://golang.org/src/net/http/server.go#L1938

* `HandlerFunc` に `ServeHTTP` が実装されている
* `Handler` に対応した `func`
  * `func(ResponseWriter, *Request)` を `HandlerFunc` にキャストすることで、`Handler` になる
* `HandlerFunc.ServeHTTP` の中で自分自身を呼び出している

## func `Handle`

```go
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }
```

https://golang.org/src/net/http/server.go#L2292

* `DefaultServeMux` のラッパー

## func `HandleFunc`

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

https://golang.org/src/net/http/server.go#L2297

* `DefaultServeMux.HandleFunc` のラッパー

## var `DefaultServeMux`

```go
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux
```

https://golang.org/src/net/http/server.go#L2131

* `DefaultServeMux` は `http` パッケージのデフォルトの `ServeMux`
* 型は `*ServeMux`

## type `ServeMux`

```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
	explicit bool
	h        Handler
	pattern  string
}
```

https://golang.org/src/net/http/server.go#L2116

* `ServeMux` 自体も `Handler`
* `Mux` = `multiplexer`
* 登録された `pattern` にマッチした `Handler` が実行される
* URLが複数の `pattern` に一致した場合には、長いほうが優先される
* `/` で終了する `pattern` (`rooted subtree`) は他の `pattern` と一致しない全てのパスに一致する
* `rooted subtree` の末尾に `/` が付かないパスのリクエストの場合には、`rooted subtree` にマッチさせる
* `pattern` はホスト名で始めることができる
    * ホスト名が指定されている場合は、ホストが含まれている `pattern` が優先される

### func `(*ServeMux) Handle`

https://golang.org/src/net/http/server.go#L2243

```go
if mux.m[pattern].explicit {
    panic("http: multiple registrations for " + pattern)
}

if mux.m == nil {
	mux.m = make(map[string]muxEntry)
}
```

* `pattern` が存在しない => `mux.m[pattern]` は `muxEntry` のゼロ値 => `explicit == false`
* `nil` の `map` にアクセスしても `panic` は起きない

https://play.golang.org/p/G33OJK-7T_

```go
// Helpful behavior:
// If pattern is /tree/, insert an implicit permanent redirect for /tree.
// It can be overridden by an explicit registration.
n := len(pattern)
if n > 0 && pattern[n-1] == '/' && !mux.m[pattern[0:n-1]].explicit {
	// If pattern contains a host name, strip it and use remaining
	// path for redirect.
	path := pattern
	if pattern[0] != '/' {
		// In pattern, at least the last character is a '/', so
		// strings.Index can't be -1.
		path = pattern[strings.Index(pattern, "/"):]
	}
	url := &url.URL{Path: path}
	mux.m[pattern[0:n-1]] = muxEntry{h: RedirectHandler(url.String(), StatusMovedPermanently), pattern: pattern}
}
```

* `pattern` が `rooted subtree` かつ、 `/` なしの `pattern` が登録されていない場合の処理
* `pattern` にホスト名を含む場合には、ホスト名なしの `pattern` を生成
* リダイレクト用の `muxEntry` を登録
    * `explicit == false` になっている

pathの生成のところ
https://play.golang.org/p/y8kKEwywc5

### func `(*ServeMux) HandleFunc`

https://golang.org/src/net/http/server.go#L2285

```go
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}
```

* `mux.Handle` のラッパー
* `func(ResponseWriter, *Request)` を `HandlerFunc` にキャストして渡している
