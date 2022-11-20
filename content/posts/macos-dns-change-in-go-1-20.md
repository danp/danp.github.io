---
title: macOS DNS resolving change in Go 1.20
date: "2022-11-08"
---

# Summary

Starting in Go 1.20[^1], DNS lookups when running on macOS will be done via the system instead of via Go's built-in resolver.
That's even when cgo is not available, such as when cross-compiling from Linux.

That should reduce surprises for people using tools cross-compiled for macOS from Linux, such as CLIs like terraform and kubectl.
It should also make it easier for developers of those tools as building on macOS may no longer be necessary.

[^1]: Unless something changes before the 1.20 release, of course.

# The problem

When a Go program does something like:

``` go
resp, err := http.Get("https://www.google.com")
```

To make the HTTP request, the program needs to find IP addresses for `www.google.com`.
For some time, there have been [two ways it can do this](https://pkg.go.dev/net#hdr-Name_Resolution):

* Via the system, similar to how a natively compiled C program would do it
* Via Go's built-in resolver

Typically the built-in resolver is used when cgo is not available.
This is often the case when programs are cross-compiled, such as building with `GOOS=darwin` on a Linux system.

For Linux the two methods are mostly interchangeable.
The built-in resolver tries to mimic glibc's behavior.

For macOS, though, the two can behave quite differently.

For example, [macOS supports extra configuration](https://www.freebsd.org/cgi/man.cgi?query=resolver&apropos=0&sektion=5&manpath=Darwin+8.0.1%2Fppc&arch=default&format=html#SEARCH_STRATEGY) under the `/etc/resolver` directory.
This has no equivalent in glibc so the built-in resolver does not support it.

Further, macOS has other ways to configure DNS, such as when connected to a VPN or using [Tailscale MagicDNS](https://tailscale.com/kb/1081/magicdns/).
This extra configuration can be inspected somewhat via the `scutil --dns` command but not as easily as a standard file like `/etc/resolv.conf`.

These differences have led to a number of Go issues and discussions over the years, namely:

* [Issue 12524](https://go.dev/issue/12524): net: Support the /etc/resolver DNS resolution configuration hierarchy on OS X when cgo is disabled
* [Issue 16345](https://go.dev/issue/16345): net: revisit unconditional use of cgo lookups for darwin (opened by me!)
* [Issue 22902](https://go.dev/issue/22902): net: LookupHost shows different results between GODEBUG=netdns=cgo and go

These manifest as surprising behavior for users, such as [these issues in projects](https://github.com/golang/go/issues/12524#issuecomment-1287547959).

# The change

In [CL 446178](https://go.dev/cl/446178), Go was changed to use the system directly when resolving.
This is done even when cgo is not available by making the necessary syscalls directly.

A similar technique is already used by the Go runtime and [to do some certificate verification since Go 1.18](https://go.dev/doc/go1.18#crypto/x509).

This addresses issue 12524 above since always using the system to resolve means `/etc/resolver` will be considered.
It means all those other ways of configuring DNS are now considered, too, even when cgo is not available.

# Seeing it in action

This little program looks up a host on my [tailnet](https://tailscale.com/kb/1136/tailnet/).
Using just the name `l1` relies on the system being used to resolve since the configuration to make it work is outside `/etc/resolv.conf`.

``` go
package main

import (
	"fmt"
	"net"
)

func main() {
	// l1 is a host on my tailnet
	addrs, err := net.LookupHost("l1")
	if err != nil {
		panic(err)
	}
	fmt.Println(addrs)
}
```

First, I'll build it for macOS on a Linux system using Go 1.19:

``` shell
GOOS=darwin GOARCH=arm64 go build
```

And then run it on my macOS system:

``` shell
> ./example
panic: lookup l1 on 192.168.1.1:53: no such host
```

This is because the extra macOS resolver configuration is not being considered.

If I build it again using Go built from tip, including CL 446178[^2], then run that:

``` shell
> ./example
[100.123.252.39]
```

It works!

You can try it yourself now with [gotip](https://pkg.go.dev/golang.org/dl/gotip) or wait for the upcoming Go 1.20 betas and release candidates.

[^2]: And [CL 448020](https://go.dev/cl/448020), which I discovered was needed as part of writing this post.

# Conclusion

I hope this resolves some long-standing confusion around macOS and DNS.

Thanks to the Go team for the change and to everyone who has kept up with these related issues over the years.
