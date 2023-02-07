---
layout: single
title: Customize Go tests with Flag options
tags: go
---

Often some tests of a Go package need to interact with another system.
You can mock it but at a certain point, the tests should run against a real system located at a certain address, using a some credentials, and so on.
The real system has some parameters that should not be hardcoded because they are ephemeral or because is not safe.
To customize these parameters there are two ways: reading from **environment variables** or reading for **test command options**.
The former is the most convenient strategy to customize not only tests but also the Go code itself, such as the url address of an API.
The latter is restricted to tests only, so it is better to use it only for testing options, like to run only unit tests, short tests, or tests that don't cost you money.

In this article I will focus on the test command options.

The command `go test` creates a main entry point function to run all other tests in the Go package.
Internally, it executes the instruction `flag.Parse()` to parse eventual options passed with `go test`.
For instance, to pass an integer variable to the tests, define a flag variable:
```go
var short = flag.Bool("short", false, "Run only short tests")
```
To run only "short" tests, run the command `go test -short=true`.

Remember to add `flag.Parse()` in the `TestMain(m *testing.M)` function in case it is defined in your code to setup and teardown the tests.
