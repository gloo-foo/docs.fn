---
title: fn
---
**`fn` adapts a [gloo-foo](https://github.com/gloo-foo/framework) command — one, or several composed — into an ordinary Go data function.** A gloo command is a `Command[[]byte, []byte]`, a transform over a stream of input lines; ideal for wiring pipelines and Unix executables, but awkward to call from ordinary code where you hold a buffer or a reader and want one back. `fn` closes that gap: it wraps any command in an immutable `Pipeline` whose method values are plain, directly-callable functions over standard data types.

- **Source:** [gloo-foo/fn](https://github.com/gloo-foo/fn)
- **API reference:** [pkg.go.dev/github.com/gloo-foo/fn](https://pkg.go.dev/github.com/gloo-foo/fn)

## Install

```
go get github.com/gloo-foo/fn
```

```go
import "github.com/gloo-foo/fn"
```

## Usage

Wrap a single command with `fn.Of`, or compose several left-to-right with `fn.Chain`, then call the resulting `Pipeline` like an ordinary function. Each stage feeds its output stream to the next.

```go
pipeline := fn.Chain(upper(), keepNonEmpty()) // fn.Pipeline
run := pipeline.String                         // func(string) (string, error)

out, err := run("hello\n\nworld")              // call it like any function
// out == "HELLO\nWORLD\n"
```

The commands themselves are ordinary `Command[[]byte, []byte]` values — a real program passes a `cmd-*` command (`Cat`, `Grep`, `Sort`, …) or one built from the framework's `patterns`:

```go
import (
    gloo "github.com/gloo-foo/framework"
    "github.com/gloo-foo/framework/patterns"
)

func upper() gloo.Command[[]byte, []byte] {
    return patterns.Map(func(line []byte) ([]byte, error) {
        return bytes.ToUpper(line), nil
    })
}
```

### Data shapes

Every `Pipeline` exposes the same command in four shapes, all from one immutable value:

- **`Bytes(input []byte) ([]byte, error)`** — reads the whole input, returns the whole output.
- **`String(input string) (string, error)`** — the same over string data.
- **`Lines(input []byte) ([]string, error)`** — the output lines without terminators.
- **`Reader(input io.Reader) io.ReadCloser`** — a lazy reader that pulls output only as it is read, so unbounded input (`yes | head`) never buffers.

Each buffered form has a `…Context` variant (`BytesContext`, `StringContext`, `LinesContext`, `ReaderContext`) that takes an explicit `context.Context`; the plain forms use `context.Background`. Cancelling the context stops the pipeline.

```go
// Buffered: whole input, whole output.
out, _ := fn.Of(upper()).Bytes([]byte("abc")) // "ABC\n"

// Streaming: output pulled lazily as the reader is consumed.
r := fn.Of(upper()).Reader(strings.NewReader("one\ntwo"))
defer r.Close()
data, _ := io.ReadAll(r)                       // "ONE\nTWO\n"
```

Append a stage to an existing pipeline with `Pipeline.To`, which returns a new pipeline and leaves the receiver unchanged. `fn.Chain()` with no arguments is the identity pipeline — input passes through unchanged.

## Design

Output framing matches a shell filter: each result line is written followed by `\n`, so `fn.Of(cat).String("abc")` yields `"abc\n"`, identical to `printf 'abc' | cat`. Only `Lines` returns the lines without terminators.

A `Pipeline` is an immutable value with value-receiver methods: safe to copy, reuse across calls, and share across goroutines. Each call runs the command over its own fresh input, so one `Pipeline` serves many invocations.

Streaming laziness comes from an `io.Pipe`: `Reader` runs the command in a goroutine that pumps its output through the pipe, and because `io.Pipe` is synchronous the command only advances as the reader consumes it — the source of both the laziness and the backpressure. The caller MUST either read to `io.EOF` or `Close` the returned reader; doing neither leaves the producing goroutine blocked forever.
