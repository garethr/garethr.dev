---
title: "More Than Hello World"
date: 2020-03-15T17:25:28Z
---

I really like creating demo applications. I partly put together demo apps so I have something to demo whatever feature
or product I happen to be working on at the time. But I mainly build demo applications so I can experiment with
some interesting tool or another. I've been writing [a bunch of demo apps recently](https://gist.github.com/garethr/1599cb36cb348d7793a8a501c70085ad)
which I might write about separately, in this post I wanted to talk about the hello world demo, and why I like to
go a little further.


## Hello World

The classic hello world demo is as simple as possible. Let's say in Python:

```python
print("hello world")
```

Or in Go:

```golang
package main
import "fmt"
func main() {
    fmt.Println("hello world")
}
```

You're probably not demoing a language though, you're probably demonstrating some tool or library. But the principle of
answering the question of how you can provide the smallest runnable example that demonstrates the point in question holds.
Take this example of the Gin library.

```golang
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

Writing these sorts of examples is important, especially if you're trying to attract some iterest in your new library
or tool. If you're audience is mainly developers I always think putting the code upfront, near the top of the README, is
the way to go. [Gin](https://github.com/gin-gonic/gin) does just that with the above example.

But I'm wanting to talk about demos that go beyond hello world, often in a slighly gratuitous way.


## Learning by writing code

In my current role (Director of Product at [Snyk](https://snyk.io)) I'm not responsible for running applications or services.
That has pros and cons, but one of the pros is that I can investigate tools that do the same thing, without having to pick one
to live with for ever. I might decide several competing tools are interesting and look at demos or integrations with all of them.

Demos are a good excuse for me to write some code too. I still learn new things best when I can build something to test out assumptions.
Plus it means I'm not writing code on the critical path, which is a good thing given everything else I'm responsible for on an
average day.

Writing demo apps is also a good way of thinking about stories, and understanding how whatever tool or technology you're experimenting
with fits into some wider narrative.


## Beyond hello world

Going beyond hello world invariably means a few things to me. The most important is having something that's end-to-end, that doesn't just
show the API, but shows a resulting applications that's packaged up for use. This invariably means combining several tools
together. I've always been an integrator at heart. This makes the demo more like a real-world application, though often without the real-world constrains.
A few things I like to consider:

* How does logging work?
* What about instrumentation?
* How is the demo application configured?
* How is the demo application deployed?
* How is the demo application tested?
* What build tool(s) should be in place?

I don't always address all of these points in all of my demo applications, but I often take one or two of them and experiment. This typically makes
writing the demo a good learning exercise too.


## Documenting demos

I like to document my demos, normally with a nice README. I also generally leave a Makefile around too. These are as much for future me as anyone else
who might want to take a look. The more specific you build demos the less likely other people are to run them directly, which is fine. They are still
incredibly useful for learning patterns from. Given folks might not run them as well, or I might not have time to run them to demonstrate something,
I'm finding it useful to capture a screenshot of anything interesting too.

The Makefile is for capturing the commands needed to run the demo. For me this is less part of the demo (unless I'm demonstrating how cool Make is) and
more about havin a structured way to write down the commands. The fact it's executable and fairly ubiquitous is a nice bonus.


## A few recent examples

What does it look like to put some of the above ideas into practice? Let's take a look at some recent demos I've been using.
I made a handy [index page](https://gist.github.com/garethr/1599cb36cb348d7793a8a501c70085ad) which includes:

* [A Go application with a over-the-top Bazel build system](https://github.com/garethr/snykly)
* [A Spring Boot application using Jib and Cloud Native Build Packs to build a Docker image](https://github.com/garethr/snykier)
* [A Ruby application deployed using the new k14s toolchain](https://github.com/garethr/snykit)
* [A Lambda application writting using Chalice](https://github.com/garethr/snyker)
* [A Node.js app using Tilt, Helm and UBI](https://github.com/garethr/snykin)
* [A Python app demonstrating a large number of open policy agent integrations](https://github.com/garethr/snyky)

It's notable that most of these use GitHub Actions for one thing or another. It's increasingly becoming my favourite swiss army knife for gluing things together.

I've purposefully not talked about recording demos here, mainly because personally I find that less useful. The demos I create are often fairly specialised
vs the more repeatable general purpose product demo that definitely benefits from a recorded version.

Let me know what you think of the above demos, or if you have tips of your own.
