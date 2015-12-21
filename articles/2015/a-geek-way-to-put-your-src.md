A geek way to place your src
===

As a software engineer, our job is to deal with source code every single day. Have you ever thought about how to place or oginize your srource code in a good control?

We don' t want to mess all our code up which leads to potential losing our code and is hard to find when you really have to find pieces of code in your file system. It will get much worse before it gets any better.

It' s all about taking some time to architect your code base before get started coding. It' s bound to be familiar, right? Just like architecting the project before rush into coding, which is bad.

Okay, let' s focus on the topic. btw, the way we talk about is inspired by Golang' s the source code conventino.

1. Put all of your source code into one place
No matter which type of the project, language it chooses, we only need one dir to put place it. You can define as many sub dirs as you would.

```
.
  companyA/...
  orgizationB/...
  test/...
  github.com/longkai/...
  github.com/foo/...
  github.com/bar/...
```

if you have github, you would like create a github.com dir, and place your own code under your account name, others code under others name, etc. which give your a clear way who creates or maintains the project.

2. Bonus
Making your root dir as the ~/src, which is more close to Linux file system and when you build and intall the source code, the binary would located into ~/bin if you use golang. Which is really helpful if your set ~/bin as PATH. You can run your binary derectly~

TODO:
