---
layout: default
title: Reducing nesting in Go functions with early returns
date: "2015-11-02"
aliases:
- /2015/11/02/reducing-go-nesting.html
---

One of my favorite localized Go refactorings is reducing nesting by using `return` as early as possible. Take this example, based on a recently-refactored function at Heroku:

```go
func example() error {
  if err := start(); err != nil {
    existing, err := fetchExisting()
    if err != nil {
      return err
    }

    if existing.IsWorking() {
      return nil
    } else {
      return errors.New("found existing but it's not working")
    }
  }

  return nil
}
```

This can be hard to follow due to nesting inside `if ...; err != nil` and use of `else`. I prefer to `return` as early as possible and avoid `else`. Applying that, it now looks something like this:

```go
func example() error {
  if err := start(); err == nil {
    return err
  }

  existing, err := fetchExisting()
  if err != nil {
    return err
  }

  if !existing.IsWorking() {
    return errors.New("found existing but it's not working")
  }

  return nil
}
```

This style is mentioned in [Effective Go](https://golang.org/doc/effective_go.html#if) but perhaps isn't well-known enough. It's a simple change but one that I feel greatly improves readability.
