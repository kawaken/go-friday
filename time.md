# package: time

今回は時刻の取得の仕方を確認

## type Time

**時刻を表す構造体**

### コメント

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
	* この方法での場所の変更は表示を変更するだけで、インスタンスの変更はしない
* `==` での比較は、時刻と場所を対象とする
* 同一の `Location` が設定されていることを保証できないなら、mapやdatabaseのキーとして利用すべきではない

### コード

```go
type Time struct {
	// sec gives the number of seconds elapsed since
	// January 1, year 1 00:00:00 UTC.
	sec int64

	// nsec specifies a non-negative nanosecond
	// offset within the second named by Seconds.
	// It must be in the range [0, 999999999].
	nsec int32

	// loc specifies the Location that should be used to
	// determine the minute, hour, month, day, and year
	// that correspond to this Time.
	// The nil location means UTC.
	// All UTC times are represented with loc==nil, never loc==&utcLoc.
	loc *Location
}
```

* `sec` は `0001-01-01 00:00:00 UTC` からの経過秒数
* `nsec` は正数のナノ秒を保持
	* `[0, 999999999]` の範囲
* `loc` は 時刻を明確にするための `Location`
	* `nil` は `UTC` を表す
	* `UTC` を表すために `&utcLoc` を使用しないこと

## func Now

```go
// Now returns the current local time.
func Now() Time {
	sec, nsec := now()
	return Time{sec + unixToInternal, nsec, Local}
}
```

* `Now` は `now` を呼び出している
* `Now` は以下のように初期化された `Time` を返す
	* sec:  `sec + unixToInternal`
	* nsec: `nsec`
	* loc:  `Local`

## const unixToInternal

**Unix時間とUTCの換算のための定数値**

```go
unixToInternal int64 = (1969*365 + 1969/4 - 1969/100 + 1969/400) * secondsPerDay
```

* 0000年1月1日00:00:00から1971年1月1日00:00:00までの経過時間

## const secondsPerDay

**単位換算のための定数値**

```go
const (
	secondsPerMinute = 60
	secondsPerHour   = 60 * 60
	secondsPerDay    = 24 * secondsPerHour
	secondsPerWeek   = 7 * secondsPerDay
	daysPer400Years  = 365*400 + 97
	daysPer100Years  = 365*100 + 24
	daysPer4Years    = 365*4 + 1
)
```

## func now

```go
// Provided by package runtime.
func now() (sec int64, nsec int32)
```

* `now` は `runtime` パッケージで定義（実装）されている
* `now` は `sec`, `nsec` を返す

`now` は `runtime/sys_GOOS_GOARCH.s` の中に実装されている。

`runtime/sys_linux_amd64.s`

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

1. `runtime·__vdso_clock_gettime_sym` が使用できれば呼び出して、 `sec`, `nsec` を取得して返す
2. `runtime·__vdso_clock_gettime_sym` が使用できなければ、 `runtime·__vdso_gettimeofday_sym` を呼び出して `sec`, `usec` を取得
3. `usec` を1000倍して `nsec` として返す

## func runtime.\_\_vdso\_\*\_sym

**vDSOを利用するためのシンボル**

```go
// initialize with vsyscall fallbacks
var (
	__vdso_time_sym          uintptr = 0xffffffffff600400
	__vdso_gettimeofday_sym  uintptr = 0xffffffffff600000
	__vdso_clock_gettime_sym uintptr = 0
)

func vdso_parse_symbols(info *vdso_info, version int32) {
	if !info.valid {
		return
	}

	for _, k := range sym_keys {
		for chain := info.bucket[k.sym_hash%uint32(len(info.bucket))]; chain != 0; chain = info.chain[chain] {
			sym := &info.symtab[chain]
			typ := _ELF64_ST_TYPE(sym.st_info)
			bind := _ELF64_ST_BIND(sym.st_info)
			if typ != _STT_FUNC || bind != _STB_GLOBAL && bind != _STB_WEAK || sym.st_shndx == _SHN_UNDEF {
				continue
			}
			if k.name != gostringnocopy(&info.symstrings[sym.st_name]) {
				continue
			}

			// Check symbol version.
			if info.versym != nil && version != 0 && int32(info.versym[chain]&0x7fff) != version {
				continue
			}

			*k.ptr = info.load_offset + uintptr(sym.st_value)
			break
		}
	}
}
```

### vDSO

* virtual dynamic shared object
* カーネルが自動的にすべてのユーザー空間アプリケーションのアドレス空間にマッピングを行う小さな共有ライブラリ
* システムコールはオーバーヘッドが大きいのでそれを避けるために使用される

## まとめ

* `Time` は内部では、秒とナノ秒とロケーションを保持している
*  秒は `unixtime` ではなく `UTC`
* `time.Now` は `time.now` を使用している
* `time.now` は `GOOS`, `GOARCH` ごとにアセンブラで実装されている
* `vDSO` を使用し `clock_gettime`, `gettimeofday` を呼び出している
* `time.now` は `unixtime` なので `unixToInternal` を加算している

## 参考

* [clock_getres.3p - Linux manual page](http://man7.org/linux/man-pages/man3/clock_gettime.3p.html)
* [gettimeofday(2) - Linux manual page](http://man7.org/linux/man-pages/man2/gettimeofday.2.html)
* [A Manual for the Plan 9 assembler](http://9p.io/sys/doc/asm.html)
* [X86アセンブラ/x86アーキテクチャ - Wikibooks](https://ja.wikibooks.org/wiki/X86%E3%82%A2%E3%82%BB%E3%83%B3%E3%83%96%E3%83%A9/x86%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3)
* [アセンブラ入門](http://www5c.biglobe.ne.jp/~ecb/assembler/assembler00.html)
[VDSO(arm)の実装をちょっと調べてみました - Qiita](http://qiita.com/akachochin/items/d5d1ba84fefae2f781f3)
* [Man page of VDSO](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/vdso.7.html)
* [Go TimeとLinuxカーネルの関係 - ワザノバ | wazanova](http://wazanova.jp/items/1147)
* [実行ファイル形式のELFって何？ - ITmedia エンタープライズ](http://www.itmedia.co.jp/help/tips/linux/l0448.html)
