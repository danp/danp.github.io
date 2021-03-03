---
layout: default
title: Injecting errors to test idempotency
date: "2016-02-24"
aliases:
- /2016/02/24/injecting-errors-to-test-idempotency.html
---

As part of [Heroku Private Spaces](https://www.heroku.com/private-spaces) we attach extra [Elastic Network Interfaces](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)
to instances, and we do that in Go with [aws-sdk-go](https://github.com/aws/aws-sdk-go/).

The process isn't the most idempotent and we recently discovered we were occasionally leaking ENIs. This could happen if we created an ENI and then encountered an error later in the process, such as when attaching it to an instance. I set out this morning to find a better way.

When an ENI is created, it can be assigned a description. Previously we used descriptions only for the ENI's function (eg, "nat") and not anything specific to the instance the ENI was intended for. After seeing that listing ENIs allows filtering on the description, I decided to make that the foundation of the new approach.

The new process looks like this:

1. try finding the ENI with the right description, based on its function and intended instance's ID
2. if no ENI was found, create it with the description
3. if the ENI was found AND it's attached to an instance already AND its DeleteOnTermination attribute is set to `true`, we're done
4. otherwise, attach it to the instance if necessary
5. set the DeleteOnTermination attribute to `true`

After getting the new process working, I wanted to verify it would be fully idempotent when encountering AWS or other errors. There's not really a knob to turn to make AWS error for the API calls involved in the process so I thought I'd try some simple error injection.

I added this to the code:

```go
var randomError = errors.New("random error!")

func maybeErr(err error, pct int) error {
	if err == nil && rand.Intn(100) < pct {
		err = randomError
	}
	return err
}
```

and then changed blocks of this form:

```go
foo, err := doSomething()
if err != nil {
	return err
}
```

to this:

```go
foo, err := doSomething()
err = maybeErr(err, 30)
if err != nil {
	return err
}
```

This caused calls of `doSomething()`, usually a method call such as [DescribeNetworkInstances](https://godoc.org/github.com/aws/aws-sdk-go/service/ec2#EC2.DescribeNetworkInterfaces), to fail 30% of the time when it would have otherwise succeeded.

Using these changes in a test Space with many instances let me get confidence that the process was idempotent. After a few rounds of scaling up and down I didn't see any leaked ENIs.

aws-sdk-go does have support for [request handlers](https://godoc.org/github.com/aws/aws-sdk-go/aws/request#Handlers) which I could have also used. A randomly-erroring `Send` handler would accomplish much the same thing but `maybeErr` let me inject errors in precise spots. It would be fun to apply generally, though.
