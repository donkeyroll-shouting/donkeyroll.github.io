---
layout: post
title:  "Error Handling in Go"
date:   2023-04-05 21:36:04 -0000
categories: golang
---

# Why

Golang has multiple approaches to handle the error. It is very important to handle the error and write descriptive error messages. Otherwise, you will have a headache when you try to debug the program later.

# Best practice

## Add error description

You should not directly return the error immediately. I know you've done this many times but **please don't next time**. It is hard to know which layer of codes cause the issue if there is something wrong.
```go
func badError() error {
    if err := something(); err != nil {
        return err
    }
    return nil
}
```

Instead, you should 
```go
func goodError() error {
    if err := something(); err != nil {
        return fmt.Errorf("failed to do something: %v", err)
    }
    return nil
}
```

## Log error in retries and keep the last error

```go
func goodError() error {
    var err error
    if retryErr := retry.Retry(func() (bool, error){
        if err = something(); err 1= nil {
            klog.Infof("Failed to do something: %v", err)
            return false, nil
        }
        return true, nil
    }); retryErr != nil{
        return fmt.Errorf("failed to do something: %v, last error: %v", retryErr, err)
    }
    return nil
}
```

## Log error for error based decisions

```go
func badError() bool {
    if err := something(); err != nil {
        return false
    }
    return true
}
```

```go
func betterError() bool {
    if err := something(); err != nil {
        klog.Errorf("Failed to do something: %v", err)
        return false
    }
    return true
}
```

```go
func badError() bool {
    if err := something(); err != nil {
        if isNotFound(err) {
            return false, nil
        }
        return true, fmt.Errorf("failed to do something: %v", err)
    }
    return true, nil
}
```

## Handle error in defer function

```go

func cleanup() error {
    return something.Clean()
}

func badError() error {
    defer cleanup()
    if err := something(); err != nil {
        return fmt.Errorf("failed to do something: %v", err)
    }
    return nil
}
```

```go
func goodError() (err error) {
    defer func() {
        if tmpErr := cleanup(); tmpErr != nil {
            err = tmpErr
        }
    } ()
    err := something()
    if err != nil {
        return fmt.Errorf("failed to do something: %v", err)
    }
    return nil
}
```

# Detect unhandled error

In the worst case, you might write some codes without handling the returned error, however, golang compiler cannot detect if you don't handle the error at all. It will be risky because you hide problematic logics in your codes and you will never find out.

```go
func something() error {
    return errors.New("failed to do something error")
}

func badError() error {
    something()
    return nil
}
```

To detect those issues, there are tools like [errcheck][errcheck], [gometalinter][gometalinter] etc

[errcheck]: github.com/kisielk/errcheck
[gometalinter]: github.com/alecthomas/gometalinter

You can simply run the tool for example
```
errcheck -ignorepkg 'fmt,os,encoding/binary' -ignoretests -ignoregenerated [YOUR_PACKAGE_PATH]
```

The result will show the detected functions that return error but you didn't handle it correctly.


# Reference

https://stackoverflow.com/questions/43898074/is-there-a-way-to-find-not-handled-errors-in-go-code

https://stackoverflow.com/questions/57740428/handling-errors-in-defer

https://go.dev/blog/error-handling-and-go