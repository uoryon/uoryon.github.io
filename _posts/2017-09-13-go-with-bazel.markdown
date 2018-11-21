---
layout: post
title:  "Bazel with golang"
date:   2017-09-13 14:15:02 +0800
categories: golang
---

[Bazel](https://www.bazel.build/) 的细节就不在这里说了，上方链接可以去看看，google 开源的强大的 build 工具。现在团队在使用 Bazel 来 build 项目。我也算小小体验了一下下。应该算摸清楚了在 golang 项目中的玩法。

在 Golang 上，可以使用这个 [rules_go](https://github.com/bazelbuild/rules_go)。它提供了一些 golang 编译相关的方法，类似 go_library, go_test。
通过这些方法，可以非常容易的组合我们的包，把一个包所有的文件放到一个 go_library 中，就能够通过 bazel 来编译这个包了。而且还可以通过指定，name 来表明编译出来的包，这个方式就过于自由了，
按照用法，你可以任意组合 .go 源代码文件，然后打成一个 package 。如果这样做的话，会打破 golang 的package 规则，变得比较奇怪, go build 等工具就不能使用了，
所以推荐还是不要随意去指定名字，用 go_default_library 就可以了。

当然，golang 的话，还有 vendor 机制，我们能保证自己的每个包都写了 BUILD 文件，但是开源项目的话，不是每个人都使用 bazel 的。这样也是可能会阻碍，因为你的 BUILD 里的 deps 没法写。
这样的话，可以使用工具 [gazelle](https://github.com/bazelbuild/rules_go/tree/master/go/tools/gazelle) 来自动生成 BUILD 文件。不过值得注意的是，要看好自己的 bazel 版本以及 rules_go 的版本，因为有可能 gazzle
编译出来的文件中会包含一些低版本中不具有的函数方法。
