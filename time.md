# package: time

今回は時刻の取得の仕方を確認

## type Time

```go
type Time struct {
	sec int64
	nsec int32
	loc *Location
}
```

* `Time` は `nanosecond` の精度を持っている
* ポインタではなく値として保持するべき
* 複数のgoroutineから同時に利用できる
* `Before`, `After`, `Equal` で比較できる
* `Sub` は `Time` の値の差を `Duration` で返す
* `Add` は `Time` と `Duration` を加算して `Time` を返す
* `Time` のゼロ値は `January 1, year 1, 00:00:00.000000000 UTC`
* `IsZero` でゼロ値かどうかチェックできる
* `Location` と紐付いていて `Format`, `Hour`, `Year` などの表示計算に参照される
* `Local`, `UTC`, `In` は特定の場所の `Time` を返す
* 場所の変更は表示を変更するだけで、インスタンスの変更はしない
* `==` での比較は、時刻と場所を対象とする
* 同一の `Location` が設定されていることを保証できないなら、mapやdatabaseのキーとして利用すべきではない

## func Now

```go
// Provided by package runtime.
func now() (sec int64, nsec int32)

// Now returns the current local time.
func Now() Time {
	sec, nsec := now()
	return Time{sec + unixToInternal, nsec, Local}
}
```

* `Now` は `now` を呼び出している
* `now` は `runtime` パッケージで提供されている？？？
* `now` は `sec`, `nsec` を返す
* `Now` は以下のように初期化された `Time` を返す
	* sec:  `sec + unixToInternal`
	* nsec: `nsec`
	* loc:  `Local`

## func runtime.now

`now` は `runtime/sys_GOOS_GOARCH.s` の中に定義されている。

```
// func now() (sec int64, nsec int32)
TEXT time·now(SB),NOSPLIT,$16
	// Be careful. We're calling a function with gcc calling convention here.
	// We're guaranteed 128 bytes on entry, and we've taken 16, and the
	// call uses another 8.
	// That leaves 104 for the gettime code to use. Hope that's enough!
	MOVQ	runtime·__vdso_clock_gettime_sym(SB), AX
	CMPQ	AX, $0
	JEQ	fallback
	MOVL	$0, DI // CLOCK_REALTIME
	LEAQ	0(SP), SI
	CALL	AX
	MOVQ	0(SP), AX	// sec
	MOVQ	8(SP), DX	// nsec
	MOVQ	AX, sec+0(FP)
	MOVL	DX, nsec+8(FP)
	RET
fallback:
	LEAQ	0(SP), DI
	MOVQ	$0, SI
	MOVQ	runtime·__vdso_gettimeofday_sym(SB), AX
	CALL	AX
	MOVQ	0(SP), AX	// sec
	MOVL	8(SP), DX	// usec
	IMULQ	$1000, DX
	MOVQ	AX, sec+0(FP)
	MOVL	DX, nsec+8(FP)
	RET
```


