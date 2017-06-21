# package: time

今回は `time.go` を中心に

## type Time

* `Time` は `nanosecond` の精度を持っている
* ポインタではなく値として保持するべき
* 複数のgoroutineから同時に利用できる
* `Before`, `After`, `Equal` で比較できる
* `Sub` は `Time` の値の差を `Duration` で返す
* `Add` は `Time` と `Duration` を加算して `Time` を返す
* `Time` のゼロ値は `January 1, year 1, 00:00:00.000000000 UTC`
* `IsZero` でゼロ値かどうかチェックできる


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
