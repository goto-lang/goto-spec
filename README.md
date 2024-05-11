# goto
Goto is a programming language that can be mixed with Go.

# Design Principles
- **Ecosystem Compatibility over Perfection**
  - Given multiple similarly good solutions, we aim to choose the best that maintains compatibility with packages written in Go. While we prefer using sum-type enums for errors, providing syntactic sugar for the usual if err != nil pattern is a sufficient improvement that works better with how existing Go projects work.
- **Balance Familiarity and Readibility**
  - Even though we have a strong preference for Rust-like syntax, we apply syntactic changes only where we find Go's syntax to be overly cluttered or hard to read. That means we use `func Function(a uint8) string` instead of `pub fn Function(a: u8) -> String`.

## Syntax
Goto has a slightly different Syntax than Go, however most syntactic elements map directly to Go or are syntactic sugar that abbreviates common patterns in Go:

### Types
#### Collection types
**Status:** ⚠️ Partially Implemented

|        		 | Goto			| Go
|------------------------|----------------------|-------------------
| Array  		 | [][]uint8		| [][]uint8
| Map    		 | [string]uint8	| map[string]uint8
| Option 		 | ?string		| *non-existent*
| Result    		 | !string		| (string, error)
| Nillable Pointer	 | *string		| *string
| Non-nillable Pointer	 | ^string		| *non-existent*
| Bi-directional channel | <->string		| chan string
| Send-only channel	 | <-string		| chan<- string
| Receive-only channel	 | ->string		| <-chan string

#### Enums
**Status:** ⛔️ Not Implemented

Goto introduces proper enums to Go:

```go
// season.goto

type Season enum {
        Summer = 0
	Autumn = 1
	Winter = 2
  	Spring = 3
}

type Animal enum {
	Deer
	Cat
	Dog
	Fish
}
```

is equivalent to

```go
// season.go

type Season string

const (
	Summer Season = 0
	Autumn Season = 1
	Winter Season = 2
	Spring Season = 3
)

type Animal int64

const (
	Deer Animal = iota
	Cat
	Dog
	Fish
)
```

...but you can go further in Goto! Unlike in Go, enums cases in Goto can have associated values:

```go
// colors.goto

type Color enum {
	// Multiple named associated values
	Colorful struct { r, g, b uint8 }
	// A single unnamed associated value
	Grayscale uint8
}
```

```go
// colors.go

// TODO: We probably have to implement our own type for this to work
```
### Strict Null-Safety
**Status:** ⛔️ Not Implemented

As Goto offers non-nillable pointers besides the usual Go-like pointers, nil-checks can be enforced by the compiler. Like in Go, you are explicitly encouraged to make nil a valid receiver as much as possible.

```go
// nillsafe.goto

func doSomething(nillable *something) {
	if some := nillable; some != nil {

	}
}
```

### Immutability by default


### String interpolation
**Status:** ✅ Implemented

Goto has built-in string interpolation, so:
```go
// apples.goto

func apples(count int) -> string {
	return "You have \(count:d) apples!"
}
```

is equivalent to

```go
// apples.go

import "fmt"

func apples(count int) -> string {
	return fmt.Sprintf("You have %d apples!", count)
}
```

Goto string interpolation supports all formatting 'verbs' mentioned in the ["fmt" module documentation](https://pkg.go.dev/fmt).
If no formatting verb is given, Goto defaults to :v, so the above example can be written as:

```go
// apples.goto

func apples(count int) -> string {
	return "You have \(count) apples!"
}
```

### Error handling
#### Returning an error
**Status:** 🚧 Work In Progress

Goto has an error return type: Just add ! after the type annotation of a return type.
You can return a new error using the throw keyword:
```go
// hello.goto

func Hello(name string) !string {
	if name == "" {
		throw "empty name"
	}

	message := "Hi, \(name:v). Welcome!"
	return message
}
```

is equivalent to

```go
// hello.go

func Hello(name string) (string, error) {
	if name == "" {
		return "", errors.New("empty name")
	}

	message := fmt.Sprintf("Hi, %v. Welcome!", name)
	return message, nil
}
```

Note that Goto is a bit stricter when it comes to discarding return values: If a function has a return value, a call to it cannot be used as a statement - so discarding the return value(s) is only allowed explicitly.

#### Propagating Errors
**Status:** ⛔️ Not Implemented

Goto uses prefix try for error propagation:
```go
// hello2.goto

func Hello2(name string) !string {
	greeting := try Hello(name)
	return "\(greeting), have a nice day!"
}
```
This is the same as `try` in Zig and postfix `?` in Rust. While a postfix operator would be more chainable, a prefix keyword makes the control flow more clearly visible -- therefore better matching the spirit of Go.

is equivalent to

```go
// hello2.go

func Hello2(name string) (string, error) {
	greeting, err := Hello(name)
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%v, have a nice day!", v)
}
```

### Operator Overloading
**Status:** ⛔️ Not Implemented
*TODO: Read Go proposal on operator overloading (https://github.com/golang/go/issues/27605)*

### Go Expression
**Status:** ⛔️ Not Implemented

In Goto, the `go` keyword can be used as an expression. It returns a channel-like 'state' for each return value of the original function. A 'state' object blocks when receiving before its value was written and returns its inner value on all subsequent receives without blocking.

```go
// shopping.goto

func loadItems() -> []Item {
	// Under the hood, the returned value is not a channel but a 'state' that returns the last received value
	items, discounts := go fetchOffers()
	// blocks until items received a value
	for _, item := range <-items {
		print(item)
	}

	// instantly evaluates as discounts and items would be tied together
	if len(<-discounts) == 0 {
		print("No discounts!")
	}

	// instantly returns the stored value as items was already received before
	return <-items
}
```

is equivalent to the following Go code:

```go
// shopping.go

func loadItems() -> []Item {
	// Under the hood, the returned value is not a channel but a 'state' that returns the last received value
	var items State[[]Item]
	var discounts State[[]Discount]
	go func() {
		items, itemsChannel = state.New()
		discounts, discountsChannel = state.New()
		i, d := fetchOffers()
		itemsChannel <- i
		discountsChannel <- d
	}()

	// blocks until items received a value
	for _, item := range items.Read() {
		print(item)
	}

	// instantly evaluates as discounts and items would be tied together
	if len(discounts.Read()) == 0 {
		print("No discounts!")
	}

	// instantly returns the stored value as items was already received before
	return items.Read()
}
```

*TODO:* The internal compiler structure for a `state` could look like this:

*Note: This relies on the sender to only ever send one value. This is okay if we only use this as compiler intrinsics for `go` expressions*
```go
// state.go

type State[T any] struct {
	inner    T
	source   <-chan T
	once     Once
}

func New[T any]() -> (State[T], chan<- T) {
	channel := make(chan T, 1)
	state := State{source: channel}
	return state, channel
}

func (st *State[T]) Read[T any]() -> T {
	st.once.Do(func() { st.inner = <-st.source })

	return st.inner
}
```

# Mascot
Goto's mascot is an orange gopher. It is a fan-modified illustration, the unmodified version was created by Takuya Ueda (https://twitter.com/tenntenn) and was obtained under the Creative Commons 3.0 Attributions license (https://creativecommons.org/licenses/by/3.0/deed.en). The original Go gopher was designed by Renee French. (http://reneefrench.blogspot.com/)
