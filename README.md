# goto
Goto is a programming language that transpiles to Go.

# Design Principles
- **Ecosystem Compatibility over Perfection**
  - Given multiple similarly good solutions, we aim to choose the best that maintains compatibility with packages written in Go. While we prefer using sum-type enums for errors, providing syntactic sugar for the usual if err != nil pattern is a sufficient improvement that works better with how existing Go projects work.
- **Balance Familiarity and Readibility**
  - Even though we have a strong preference for Rust-like syntax, we apply syntactic changes only where we find Go's syntax to be overly cluttered or hard to read. That means we use `func Function(a uint8) string` instead of `pub fn Function(a: u8) -> String`.

## Syntax
Goto has a slightly different Syntax than Go, however most syntactic elements map directly to Go or are syntactic sugar that abbreviates common patterns in Go:

### Types
#### Collection types
|       | Goto             | Go
|-------|------------------|----------
| Array | [[uint8]]        | [][]uint8
| Map   | [string : uint8] | map[string]uint8

### Enums
Goto introduces proper enums to Go:

```goto
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

```goto
type Color enum {
    // Multiple named associated values
    Colorful struct { r, g, b uint8 }
    // A single unnamed associated value
    Grayscale uint8
}
```

```go
// TODO: We probably have to implement our own type for this to work
```
### Returning an error
Goto has an error type: Just add ! after the type annotation of a return type.
You can return a new error using the throw keyword:
```goto
func Hello(name string) string! {
    if name == "" {
        throw "empty name"
    }

    message := "Hi, {name:v}. Welcome!"
    return message
}
```

is equivalent to

```go
func Hello(name string) (string, error) {
    if name == "" {
        return "", errors.New("empty name")
    }

    message := fmt.Sprintf("Hi, %v. Welcome!", name)
    return message, nil
}
```

#### Propagating Errors
Goto uses ! for error propagation:
```goto
func Hello2(name string) string! {
    greeting := Hello(name)!
    return "{greeting}:v, have a nice day!"
}
```

is equivalent to

```go
func Hello2(name string) (string, error) {
    greeting, err := Hello(name)!
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("%v, have a nice day!", v)
}
```
