# Visa Clearing Style Guide

## Table of Contents

- [Introduction](#introduction)
- [Guideline](#guideline)
  - [Rebase Team Policy](#rebase-team-policy)
  - [Errors](#errors)
    - [Error Types](#error-types)
    - [Error Wrapping](#error-wrapping)
    - [Error Naming](#error-naming)
- [Style](#style)
  - [Prefix Unexported Consts with _](#prefix-unexported-consts-with-_)
- [Patterns](#patterns)
  - [Test Tables](#test-tables)
  - [Mocks](#mocks)
  - [Name pattern for branches](#name-pattern-for-branches)

## Introduction

Styles are the conventions that guides our code. The term style is a bit of a misnomer, since these
conventions cover far more than just source file formatting—gofmt handles that for us.

The goal of this guide is to manage this complexity by describing in detail the Dos and Don'ts of 
writing Go code at Visa Clearing Team. These rules exist to keep the code base manageable while 
still allowing engineers to use Go language features productively.

This guide was created by [Diego Almeida] as a way to bring new team members up to speed with using
Go and can be changed based on feedback from others.

  [Diego Almeida]: https://github.com/diegoalmeidameli

This documents idiomatic conventions in Go code that we follow at Visa Team. A lot of these are 
general guidelines for Go, while others extend upon external
resources:

1. [Effective Go](https://golang.org/doc/effective_go.html)
2. [Go Common Mistakes](https://github.com/golang/go/wiki/CommonMistakes)
3. [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)

All code should be error-free when run through `golangci-lint` and `go vet`. We recommend setting 
up your editor to:

- Run `goimports` on save
- Run `golangci-lint` and `go vet` to check for errors

You can find information in editor support for Go tools here: <https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins>


## Guideline

### Rebase Team Policy

**Avoid creating a merge commit**. When a `feature` branch's development is complete, rebase the
`feature` branch onto `main` branch using the following commands:

```
git checkout feature/some-feature
git rebase main
```

This moves the entire `feature` branch to begin on the tip of the `main` branch, effectively 
incorporating all of the new commits in `main`. Notice that rebasing re-writes the project history 
by creating brand new commits for each commit in the original branch.

Rationale: The excessive merging clutters up the history, and rebasing is a way to avoid it. Too
many merges can make it hard for a human to follow the history. When using rebase, the final
version in the public repository will be logical and easy to follow, not preserving any of the 
hiccups that a developer has had along the way.

In addition, use an interactive rebase to squash the branch down to the minimum number of 
meaningful commits with great commit messages.

Example:

```
git rebase --interactive --autosquash main
```

### Errors

#### Error Types

Consider the following before choosing the option that best suits your use case.

- Does the caller need to match the error so that they can handle it?
  If so, we should support the [`errors.Is`] or [`errors.As`] functions declaring a top-level error 
  variable or a custom type. 
- Is the error message a static string or is it a dynamic string that requires contextual information?
  For the former we can use [`errors.New`], but for the latter we should use [`fmt.Errorf`] or a 
  custom error type.
- Are we propagating a new error returned by a downstream function?
  If so, see the [section on error wrapping](#error-wrapping).

[`errors.Is`]: https://golang.org/pkg/errors/#Is
[`errors.As`]: https://golang.org/pkg/errors/#As

| Error matching? | Error Message | Guidance                            |
|-----------------|---------------|-------------------------------------|
| No              | static        | [`errors.New`]                      |
| No              | dynamic       | [`fmt.Errorf`]                      |
| Yes             | static        | top-level `var` with [`errors.New`] |
| Yes             | dynamic       | custom `error` type                 |

[`errors.New`]: https://golang.org/pkg/errors/#New
[`fmt.Errorf`]: https://golang.org/pkg/fmt/#Errorf

For example, use [`errors.New`] for an error with a static string.
Export this error as a variable to support matching it with `errors.Is` if the caller needs to 
match and handle this error.

<table>
<thead><tr><th>No error matching</th><th>Error matching</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

if err := foo.Open(); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

ErrResourceNotFound = errors.New("resource not found")

func Get() error {
  return ErrResourceNotFound
}

// package bar

if err := foo.Open(); err != nil {
  if errors.Is(err, foo.ErrResourceNotFound) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

For an error with a dynamic string, use [`fmt.Errorf`] if the caller does not need to match it
and a custom `error` if the caller does need to match it.

<table>
<thead><tr><th>No error matching</th><th>Error matching</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

func Open(file string) error {
  return &NotFoundError{File: file}
}


// package bar

if err := foo.Open("testfile.txt"); err != nil {
  var notFound *NotFoundError
  if errors.As(err, &notFound) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

Note that if you export error variables or types from a package, they will become part of the 
public API of the package.

#### Error Wrapping

There are three main options for propagating errors if a call fails:

- return the original error as-is
- add context with `fmt.Errorf` and the `%w` verb
- add context with `fmt.Errorf` and the `%v` verb

Return the original error as-is if there is no additional context to add. This maintains the 
original error type and message. This is well suited for cases when the underlying error message
has sufficient information to track down where it came from.

Otherwise, add context to the error message where possible so that instead of a vague error such 
as "connection refused", you get more useful errors such as "call service foo: connection refused".

Use `fmt.Errorf` to add context to your errors, picking between the `%w` or `%v` verbs based on 
whether the caller should be able to match and extract the underlying cause.

- Use `%w` if the caller should have access to the underlying error.
  This is a good default for most wrapped errors, but be aware that callers may begin to rely 
  on this behavior. So for cases where the wrapped error is a known `var` or type, document and 
  test it as part of your function's contract.
- Use `%v` to obfuscate the underlying error.
  Callers will be unable to match it, but you can switch to `%w` in the future if needed.

When adding context to returned errors, keep the context succinct by avoiding
phrases like "failed to", which state the obvious and pile up as the error
percolates up through the stack:

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %w", err)
}
```

</td><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %w", err)
}
```

</td></tr><tr><td>

```
failed to x: failed to y: failed to create new store: the error
```

</td><td>

```
x: y: new store: the error
```

</td></tr>
</tbody></table>

However once the error is sent to another system, it should be clear the
message is an error (e.g. an `err` tag or "Failed" prefix in logs).

See also [Don't just check errors, handle them gracefully].

  [`"pkg/errors".Cause`]: https://godoc.org/github.com/pkg/errors#Cause
  [Don't just check errors, handle them gracefully]: https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

#### Error Naming

For error values stored as global variables, use the prefix `Err`. For unexported ones, use `err`.

```go
var (
  // The following two errors are exported
  // so that users of this package can match them
  // with errors.Is.

  ErrBrokenLink = errors.New("link is broken")
  ErrResourceNotFound = errors.New("resource not open")

  // This error is not exported because
  // we don't want to make it part of our public API.
  // We may still use it inside the package
  // with errors.Is.

  errNotFound = errors.New("not found")
)
```

For custom error types, use the suffix `Error` instead.

```go
// Similarly, this error is exported
// so that users of this package can match it
// with errors.As.

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

// And this error is not exported because
// we don't want to make it part of the public API.
// We can still use it inside the package
// with errors.As.

type resolveError struct {
  Path string
}

func (e *resolveError) Error() string {
  return fmt.Sprintf("resolve %q", e.Path)
}
```
## Style

### Prefix Unexported Consts with _

Prefix unexported top-level `const`s with `_` to make it clear when and where they are used.

Exception: Unexported error values, which should be prefixed with `err`.

Rationale: Top-level constants have a package scope. Using a generic name makes it easy to 
accidentally use the wrong value in a different file.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

</td><td>

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

</td></tr>
</tbody></table>

## Patterns

### Test Tables

Use table-driven tests with [subtests] to avoid duplicating code when the core test logic is 
repetitive.

  [subtests]: https://blog.golang.org/subtests

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

</td><td>

```go
// func TestSplitHostPort(t *testing.T)

tt := []struct{
  input     string
  expectedHost string
  expectedPort string
}{
  {
    input:     "192.0.2.0:8000",
    expectedHost: "192.0.2.0",
    expectedPort: "8000",
  },
  {
    input:     "192.0.2.0:http",
    expectedHost: "192.0.2.0",
    expectedPort: "http",
  },
  {
    input:     ":8000",
    expectedHost: "",
    expectedPort: "8000",
  },
  {
    input:     "1:8",
    expectedHost: "1",
    expectedPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.input, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.input)
    require.NoError(t, err)
    assert.Equal(t, tt.expectedHost, host)
    assert.Equal(t, tt.expectedPort, port)
  })
}
```

</td></tr>
</tbody></table>

Test tables make it easier to add context to error messages, reduce duplicate logic, and add new 
test cases.

We follow the convention that the slice of structs is referred to as `tt` (table test) and each 
test case as `tc`. Further, we encourage explicating the input and output values for each test 
case with `input` and `expected` prefixes.

```go
tt := []struct{
  input     string
  expectedHost string
  expectedPort string
}{
  // ...
}

for _, tc := range tt {
  // ...
}
```

### Mocks

When configuring the methods of a mock, we should specify the number of times we expect a certain method to be executed. There are three methods available for this: `Once()`, `Twice()` and `Times(n int)`.

At the end of each test, we should validate that the mock was executed as expected, for this we use the `AssertExpectations(t testing.T)` method.

Avoid adding logic in table test
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go

testcases := []testcase{
    {
        name: "Error",
        expectedErr: errors.New("unexpected error"),
        anotherExpectedErr: nil,
        paramOne: context.Background(),
        paramTwo: "test",
    },
    {
        name: "Another Error",
        expectedErr: nil,
        anotherExpectedErr: errors.New("unexpected error"),
        paramOne: context.Background(),
        paramTwo: "test",
    },
}

for tc := range testcases {
    tt.Run(func(t testing.T) {
        serviceMock := service.Mock{}
		 
        if (tc.anoterExpecterErr != nil) {
            serviceMock.On("Get", tc.paramOne, tc.paramTwo).Return(tt.expectedErr)
        } else {
            serviceMock.On("Find", tc.paramOne, tc.paramTwo).Return(tt.anoterExpectedErr)
        }
		
        // test body
    })
}

```

</td><td>

```go

testcases := []testcase{
    {
        name: "Error",
        expectedErr: errors.New("unexpected error"),
        paramOne: context.Background(),
        paramTwo: "test",
		newServiceMock: func() *service.Mock {
            serviceMock := &service.Mock{}
            serviceMock.On("Get", context.Background(), "test").Return(errors.New("unexpected error")).Once()
            return serviceMock    
		}
    },
    {
        name: "Another Error",
        expectedErr: nil,
        paramOne: context.Background(),
        paramTwo: "test",
        newServiceMock: func() *service.Mock {
            serviceMock := &service.Mock{}
            serviceMock.On("Find", context.Background(), "test").Return(errors.New("another unexpected error")).Twice()
            return serviceMock
        }
    },
}

for tc := range testcases {
    tt.Run(func(t testing.T) {
        serviceMock := tc.newServiceMock()
        
        // test body

        setviceMock.AssertExpectations(t)
    })
}

```

</td></tr>
</tbody></table>

### Name pattern for branches

Name your branch with the prefix of the card generated by Jira plus keywords that make sense for 
the context of the activity. Examples:
```
feature/gen-9870-auth-synchronization
bugfix/gen-12508-prevents-wrong-patch
```

