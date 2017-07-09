Id: 1Bkr
Title: 3 ways to iterate in Go
Format: Markdown
Tags: for-blog, draft, go
CreatedAt: 2017-06-19T06:56:45Z
UpdatedAt: 2017-07-08T19:44:07Z
--------------
@header-image gfx/headers/header-09.jpg
@collection go-cookbook
@status draft
@description 3 different way to implement an iterator in Go: callbacks, channels, struct with Next() function.

Code for this article: https://github.com/kjk/go-cookbook/tree/master/3-ways-to-iterate

Iteration is a frequent need, be it iterating over lines of a file, results or of `SELECT` SQL query or files in a directory.

There are 3 common iteration patterns in Go programs:
* callbacks
* an iterator object with `Next()` method
* channels

## Iteration mixed with processing

Before discussing different ways of designing iteration API, let's see how we would iterate without encapsulating iteration logic.

Our example is iterating over even numbers, starting with 2 up to a given `max` number (inclusive).

To show handling of errors we'll consider `max` less than 0 to be invalid.

This is intentionally the simplest possible iterator so that we can focus on the implementation of the iterator API and not generating the values to iterate over.

Our processing is simple as well: we print the number.

Here's an example of iteration intertwined with processing.

Full example: [3-ways-to-iterate/inlined.go](https://github.com/kjk/go-cookbook/blob/master/3-ways-to-iterate/inlined.go).
```go
func printEvenNumbers(max int) {
	if max < 0 {
		log.Fatalf("'max' is %d, should be >= 0", max)
	}
	for i := 2; i <= max; i += 2 {
		fmt.Printf("%d\n", i)
	}
}
```

This is fine if iteration logic is simple. If our iteration was complex, like iterating over lines in a file, every time we needed to do different processing of the lines, we would end up copy & pasteing a lot of code.

For easy reuse we want to encapsulate complex iteration logic and provide simple API to callers.

## Iterating via callback

The caller provides callback function to be called with each value.

Full example: [3-ways-to-iterate/callback.go](https://github.com/kjk/go-cookbook/blob/master/3-ways-to-iterate/callback.go).

Client side of iteration:

```go
func printEvenNumbers(max int) {
	err := iterateEvenNumbers(max, func(n int) error {
		fmt.Printf("%d\n", n)
		return nil
	})
	if err != nil {
		log.Fatalf("error: %s\n", err)
	}
}
```

We need a way to stop iteration from within the callback which is why the callback returns an `error`. Returning non-nil `error` from callback stops iteration.

Implementation of iterator:

```go
func iterateEvenNumbers(max int, cb func(n int) error) error {
	if max < 0 {
		return fmt.Errorf("'max' is %d, must be >= 0", max)
	}
	for i := 2; i <= max; i += 2 {
		err := cb(i)
		if err != nil {
			return err
		}
	}
	return nil
}
```

This pattern is used in [`filepath.Walk`](https://golang.org/pkg/path/filepath/#Walk) API in standard library.


## Iterating with `Next()`

Another pattern is to implement iterator struct with 3 functions:
* `Next()` advances iterator to next value. It returns false to indicate end of iteration (which can be due to error)
* `Value()` to get the current value of the iterator. The name depends on the kind of value we retrieve
* optional `Err()` function which returns iteration error

Full example: [3-ways-to-iterate/next.go](https://github.com/kjk/go-cookbook/blob/master/3-ways-to-iterate/next.go).

Client code:

```go
func printEvenNumbers(max int) {
	iter := NewEvenNumberIterator(max)
	for iter.Next() {
		fmt.Printf("n: %d\n", iter.Value())
	}
	if iter.Err() != nil {
		log.Fatalf("error: %s\n", iter.Err())
	}
}
```

Notice how `Next()` fits nicely with `for` loop thanks to returning bool and indicating end of iteration with `false`.

Unfortunately, the nice API on the caller side requires complicated implementation of the iterator.

We need to carry state across `Next()` calls and remember iteration errors:

```go
// EvenNumberIterator generates even numbers
type EvenNumberIterator struct {
	max       int
	currValue int
	err       error
}

// NewEvenNumberIterator creates new number iterator
func NewEvenNumberIterator(max int) *EvenNumberIterator {
	var err error
	if max < 0 {
		err = fmt.Errorf("'max' is %d, should be >= 0", max)
	}
	return &EvenNumberIterator{
		max:       max,
		currValue: 0,
		err:       err,
	}
}

// Next advances to next even number. Returns false on end of iteration.
func (i *EvenNumberIterator) Next() bool {
	if i.err != nil {
		return false
	}
	i.currValue += 2
	return i.currValue <= i.max
}

// Value returns current even number
func (i *EvenNumberIterator) Value() int {
	if i.err != nil || i.currValue > i.max {
		panic("Value is not valid after iterator finished")
	}
	return i.currValue
}

// Err returns iteration error.
func (i *EvenNumberIterator) Err() error {
	return i.err
}
```

Notes:
* this method requires the largest amount of boilerplate
* `Next()` should return `false` if there was an error
* `Value()` panics if accessed after iteration has finished


Roughly this pattern is used in standard library:
* [`Rows.Next`](https://golang.org/pkg/database/sql/#Rows.Next) to iterate over results of SQL `SELECT` statement
* [`Scanner.Scan`](https://golang.org/pkg/go/scanner/#Scanner.Scan) to iterate over text
* [`Decoder.Token`](https://golang.org/pkg/encoding/xml/#Decoder.Token) for XML parsing
* [`Reader.Read`](https://golang.org/pkg/encoding/csv/#Reader.Read) in CSV reader

Some of those iterators combine `Next()` and `Value()` into a single function returning multiple values.

## Iterating with a channel

Channels and goroutines are Go's banner features and can be used as iterators.

Full example: [3-ways-to-iterate/channel.go](https://github.com/kjk/go-cookbook/blob/master/3-ways-to-iterate/channel.go).

Caller side:

```go
func printEvenNumbers(max int) {
	for val := range generateEvenNumbers(max) {
		if val.Err != nil {
			log.Fatalf("Error: %s\n", val.Err)
		}
		fmt.Printf("%d\n", val.Int)
	}
}
```

`generateEvenNumbers()` returns a channel which will be closed to indicate end of iteration. Closing the channel ends  `for` loop.

If there is no possibility of failing we can send just values over the channel.

In our case a failure is possiblity, so we have to send a struct that packages the value and possible error:

```go
type IntWithError struct {
	Int int
	Err error
}
```

Generator side:

```go
func generateEvenNumbers(max int) chan IntWithError {
	ch := make(chan IntWithError)
	go func() {
		defer close(ch)
		if max < 0 {
			ch <- IntWithError{
				Err: fmt.Errorf("'max' is %d and should be >= 0", max),
			}
			return
		}

		for i := 2; i <= max; i += 2 {
			ch <- IntWithError{
				Int: i,
			}
		}
	}()
	return ch
}
```

We could use buffered channel, e.g.: `ch := make(chan IntWithError, 128)`. That would speed up things if both generation and processing are time consuming by parallelizing those 2 processes.

## Which way is the best?

The one that best fits your scenario.

The callback pattern makes for a simple implementation of the iterator but callbacks in Go are akward syntax.

Using `Next()` is the hardest to implement but presents nice interface to the caller. It's most commonly used in Go standard library for complex iterators.

Channel-based iterator is easy to implent and use by the caller but most expensive. Only in exceptional circumstances the cost should be of concern.

It's also the only one that is concurrent by nature.