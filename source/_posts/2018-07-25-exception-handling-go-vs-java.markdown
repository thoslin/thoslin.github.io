---
layout: post
title: "Exception Handling: Go vs. Java"
date: 2018-07-25 11:25
comments: true
categories: 
---
After reading this post [Why Go Gets Exceptions Right](https://dave.cheney.net/2012/01/18/why-go-gets-exceptions-right), I have some thoughts and would like to write down here.

The first thing bumps to my head is why we're using exceptions in the first place. The answer seems clear, we want to signal the caller of our function that something went wrong.

So how do we do it? In Java we have exceptions, specially checked exceptions with ```throws``` keyword in the method signature. Checked means it will be checked by the compiler at the compile time, as a way to inform the caller that certain exceptions are expected to be thrown from the method.
While in go, with the ability to return multiple values, go informs the caller by returning an ```error```. It is part of the function signature, the contract between the function and the caller, the caller should receive the result and the error correctly, otherwise the program won't compile.

Second thing I'm wondering is that does Go really handle exceptions better than Java? Let's see how Go and Java handle them differently.

If we categorize the exceptions into following categories, language-ignostically(they're all concepts from Java, but can be interchangable for discussion):

- *Checked exceptions*. Something goes wrong, unpreventable but recoverable.
- *Runtime exceptions*. Something may go wrong, but preventable.
- *Errors*. Soemthing goes wrong, unpreventable and unrecoverable.

***Checked exceptions***

As I mentioned above, checked exceptions are enforced for both Java and Go. And for the caller, they should handle the exception properly. Either try-catch in Java, or check if err is not nil in Go. And in go, you can ignore the error by using ```_```, similarly, in Java you can try-catch the exception and does nothing about it.

***Runtime exceptions***

Runtime exceptions is the exceptions you can’t always detect at the compile time. It could be a null pointer exception, or a index out of bounds exception. In Java, as runtime exception is unchecked, you don't need to explicitly throws them in the method. In Go, you don't need to specify an error in the return statement. And for the caller, you have no idea what went wrong, you basically do nothing with it. Or you can explicitly try-catch in Java or [defer-recover](https://blog.golang.org/defer-panic-and-recover) in Go, otherwise, for Java the exception will bubble up the call stack until some exception handler catches it. And for Go, the panicking will also climb up the stack of the current goroutine until some recover happens. If none of this happens until main method, the program crashes.

***Error***

Errors are unpreventable and unrecoverable exceptions, like out of memory error. In Java you don't catch it and In Go you should just let it panic.

So viewing from this angle, the way Go and Java handles exceptions are almost the same. That's why I don't think Go is better than Java in this regard. On the contrary, there are things I don’t like about error handling in Go:

- Not like Java, Go does not grap the error stacktrace by default. You may say it is not that expensive as Java and give you some flexibilities. However it is not good for debugging. If you’d like to print the stacktrace, you'll have to use libraries like github.com/pkg/errors to wrap the error with stack trace.
- Not like Java, Go does not differentiate errors in function signature, you just have one error, it is like ```throws Exception``` in Java. You don’t know the exact errors you are expecting. Explicit is better than implicit.
