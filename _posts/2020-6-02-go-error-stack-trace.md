---
layout: post
title: Errors with stack trace
comments: true
categories: Go
---

Often when debugging Go code, errors stack trace is often built with following pattern:

```go
if err != nil {
    return fmt.Errorf("Failed to do something: %w", err)
}
```

One error is wrapped with another, message is concatenated, with some delimiter in this case `:`. We end up with error information hard to pin to a place where it actualy happend, which results in long trail and many steps of going file by file, error to error until we actually find it. So how could we improve our error experiance ?Before we start lets rewind a little bit and answer a simple question. What is error ?

**Errors are values**. This is very crucial thing to understand, if you come to Go from another enviroment you have to move one from a mental object of *try/catch*. Once you do that handling errors becomes clear and simple.

So what does it really mean ?

>  Values can be programmed, and since errors are values, errors can be programmed.

Most common way of handling errors is simply comparing them with `nil` value. Because **errors are values**.

```go
if err != nil {
    // something went wrong
}
```

This also means we can add any information to error, for example stack trace.

# Error as value

[error](https://golang.org/pkg/builtin/#error) type is simply an interface, which means by creating our custom error type that implements that interface we could add any extra information. Lets define our error as follow:

```go
type AppError struct {
	trace string
	err   error
}

// Error returns the string representation of the error message.
func (e *AppError) Error() string {
	return fmt.Sprintf("%s\n%s", e.trace, e.err)
}

func (e *AppError) Unwrap() error {
	return e.err
}
```

Our error will contain wrapped error, plus a stack trace where it happened. *AppError* type will implement two important methods: [Error](https://golang.org/pkg/builtin/#error) and [Unwrap](https://golang.org/pkg/errors/#Unwrap). Since Go 1.13 introduces [new features](https://blog.golang.org/go1.13-errors) to the errors and fmt standard library packages to simplify working with errors that contain other errors. It makes it even easier to handle them.

## Unwrap
*Unwrap* method that returns its contained error:

```go
func (e *AppError) Unwrap() error {
	return e.err
}
```

# Adding stack trace

To add stack trace to our error we will use [runtime.Caller](https://golang.org/pkg/runtime/#Caller)

> Caller reports file and line number information about function invocations on the calling goroutine's stack. The argument skip is the number of stack frames to ascend, with 0 identifying the caller of Caller. (For historical reasons the meaning of skip differs between Caller and Callers.) The return values report the program counter, file name, and line number within the file of the corresponding call. The boolean ok is false if it was not possible to recover the information.

End result should look something like this:

```go
func main() {
	fmt.Printf("%s", AppenStackTrace(fmt.Errorf("internal error")))
}
```

`AppenStackTrace` Simply wraps `error` using our `AppError` type and appends stack trace information:

```go
func AppenStackTrace(err error) *AppError {
	if err == nil {
		panic("nil error provided")
	}

	var buf bytes.Buffer

	frame := getFrame(2)

	fmt.Fprintf(&buf, "%s", frame.File)
	fmt.Fprintf(&buf, ":%d", frame.Line)
	fmt.Fprintf(&buf, " %s", frame.Function)

	return &AppError{
		err:   err,
		trace: buf.String(),
	}
}

func getFrame(calldepth int) *runtime.Frame {
	pc, file, line, ok := runtime.Caller(calldepth)
	if !ok {
		return nil
	}

	frame := &runtime.Frame{
		PC:   pc,
		File: file,
		Line: line,
	}

	funcForPc := runtime.FuncForPC(pc)
	if funcForPc != nil {
		frame.Func = funcForPc
		frame.Function = funcForPc.Name()
		frame.Entry = funcForPc.Entry()
	}

	return frame
}
```

To build complete stack trace, we could create a method as follow:

```go
func (e *AppError) StackTrace() (string, error) {
	var buf bytes.Buffer

	if e.trace != "" {
		if _, err := fmt.Fprintf(&buf, "%s", e.trace); err != nil {
			return "", err
		}
	}

	if e.err == nil {
		return buf.String(), nil
	}

	var next *AppError
	if errors.As(e.err, &next) {
		stackTrace, err := next.StackTrace()
		if err != nil {
			return "", err
		}

		buf.WriteString(fmt.Sprintf("\n%s", stackTrace))
	} else {
		return fmt.Sprintf("%s\n%s", buf.String(), e.err), nil
	}

	return buf.String(), nil
}
```

We simply build buffer with each error information and then dig dipper into our error chain. We use [errors.As](https://golang.org/pkg/errors/#As).
> The As function tests whether an error is a specific type.

Running following example:

```go
func main() {
	err := AppenStackTrace(testOne())
	fmt.Printf("%s", err)
}

func testOne() error {
	return fmt.Errorf("testOne: %w", AppenStackTrace(testTwo()))

}

func testTwo() error {
	return fmt.Errorf("testTwo: %w", AppenStackTrace(testThree()))
}

func testThree() error {
	return AppenStackTrace(fmt.Errorf("internal error"))
}
```

should give us output:

```sh
/tmp/sandbox073776686/prog.go:74 main.main
testOne: /tmp/sandbox073776686/prog.go:82 main.testOne
testTwo: /tmp/sandbox073776686/prog.go:87 main.testTwo
/tmp/sandbox073776686/prog.go:91 main.testThree
internal error
```

Printing error uses `.Error()` method from `error` interface. Which under the hood unwraps each error using `Unwrap` method, because we wrapped them with `fmt.Errorf` and `%w` print format. Lets print our stack trace using our builtin `StackTrace` method:

```go

func main() {
	t, _ := err.StackTrace()
	fmt.Printf("%s", t)
}
```

output:

```sh
/tmp/sandbox073776686/prog.go:74 main.main
/tmp/sandbox073776686/prog.go:82 main.testOne
/tmp/sandbox073776686/prog.go:87 main.testTwo
/tmp/sandbox073776686/prog.go:91 main.testThree
internal error
```

# Conclusion

Handling errors is pretty simple, having them as values allows us to do really powerful things. We can append any information we want, for example stack trace as we did in the example above. Defining custom type for error is great way of doing that, and not necessarily has to be done one the whole scope of application. Instead of calling our type `AppError` we could define many types, where each of them would hold different information. Let's say `QueryError` to handle persistence model errors with query information or `HttpError` to store *http* response code and/or user error message (without stack trace).

As you can see simple check `err != nil` is not a drop in replacement of *try/catch*. It's important to change a mental object of error handling while working with Go, which is not that hard and requires only a little of time getting used to it. Please see [full code snippet here](https://gist.github.com/vardius/56b224de0a69522a24f021642449db17).
