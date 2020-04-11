---
title: "Golang String Typecasting Has Unexpected Surprises"
date: 2020-03-29T19:32:58+01:00
draft: false
author: ""
authorLink: ""
description: "I reached for the wrong tool to convert int to string and found a surprise learning moment."
license: ""

tags: [golang, software, types]
categories: [golang]
hiddenFromHomePage: false

featuredImage: ""
featuredImagePreview: ""

toc: true
autoCollapseToc: true
math: true
lightgallery: true
linkToMarkdown: true
share:
  enable: true
comment: true
---

## string(int) != strconv.Itoa(int)

While at work, I was asked to fix a statement that needed to print out an error message when the server wasn't able to connect to the database. The goal was for it to look something like:

`Unable to connect to database localhost:1234 : [Some Error Here]`

The code that wasn't working, looked like:

```go
func printError(host string, port int, err string) {
  fmt.Println("Unable to connect to database %s:%d : %s", host, port, err)
}
```
_This isn't exactly what it looked like, but the behavior is the same._

Because `fmt.Println()` doesn't take formatting parameters like that, the function was printing something like:

`Unable to connect to database %s:%d : %s localhost 1234 [Some Error Here]`

Looking at it, I thought, well... let's just do this then:

```go
fmt.Println("Unable to connect to database", host + ":" + string(port), " : ", err)
```
...and then when it was run to test it...

`Unable to connect to database localhost:Ӓ  :  [some error here]`

I noticed something weird. I passed in `1234` as the `port`, but it displayed it as `Ӓ`. What?

This is the fun part.

Trying running this code and see what pops out.
```go
for i := 0; i <= 10000; i++ {
  fmt.Println(string(i))
}
```
It's printing out nearly every character you can think of, and there's probably more, but it certainly isn't the `int` you passed in.

So, what's exactly happening? To answer that, we need to dive into Golang's [rune](https://www.geeksforgeeks.org/rune-in-golang/).

When you write `string(1234)`, under the hood, Golang is interpreting it like `string(rune(1234))` and you'll find that this unexpected behavior is getting some good criticism on [Golang's Github](https://github.com/golang/go/issues/3939.).

Try running this bit of code:
```go
rune1 := rune(1234)
fmt.Printf("Rune 1: %c; Unicode: %U; Type: %s; Int: %d", rune1, rune1, reflect.TypeOf(rune1), rune1)
```
which outputs
```
Rune 1: Ӓ; Unicode: U+04D2; Type: int32; Int: 1234
```

So, clearly runes are doing some interesting things, and notice that it's a `int32` under the hood. Since `rune` is like `char` in Java, runes are an int32 representation of a character, specifically [Unicode](https://www.ssec.wisc.edu/~tomw/java/unicode.html#x0400). Therefore, when you pass in an `int` to `string()` golang is converting the decimal unicode into it's character equivalent and returning a string `"Ӓ"` and not `'Ӓ'`. Notice the double versus single quotes here.

In conclusion, when you're wanting to convert an `int` to a `string` reach for some of the following methods instead:
```go
strconv.Itoa(1234)
fmt.Sprintf("%d", 1234)
```

These methods are likely to remain unchanged in future versions of Go.