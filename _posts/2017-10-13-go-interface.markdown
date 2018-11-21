---
layout: post
title:  "Golang interface"
date:   2017-10-13 16:14:02 +0800
categories: golang
comments: true
---
Golang 的 interface(接口) 是一个很神奇的东西，定义为：
- 接口类型是由一组方法定义的集合
- 接口类型的值可以存放实现这些方法的任何值

<!--more-->

也就是说，只要一个值实现了这个接口的方法，那么他就是这个接口。最简单的例子是一个空的接口`interface{}`。因为这个接口是空的，
那么所有的类型，都可以认为是实现了这个接口，那么他就是这个接口。所以函数
```
func foo(a interface{}) {
  return
}
```
是可以接受任何类型的值作为参数的。
定义接口的语法如下：
```
type fooInterface interface{
  Foo() string
}
```

这样，我们就算定义了一个接口，这个接口需要有方法`Foo() string`。我们只用定义一个 struct
```
type foo struct{}

func (f foo) Foo() string {
  return "foo"
}
```
这样，这个 `foo` 就算实现了`fooInterface` 这个接口。
```
func(a fooInterface) {
  fmt.Println("ok")
}(&foo{}) // "ok"
```

接口在很多地方都可以应用，比如抽象一个数据库数据对象，或者写一个功能的多种算法实现。都是非常简便的方法。

## 接口反推类型
有的时候我们需要通过接口来反推类型。举个例子，比如用 yaml 写了一个文件。你需要解析它，读取数据。
```
	func Unmarshal(in []byte, out interface{}) (err error)
```
Unmarshal 的接口长这样，out 就是你要传递进去的参数，通常，你可以直接定义一个接口体，传进去，就像 yaml 给出的例子。
```
type T struct {
    F int `yaml:"a,omitempty"`
    B int
}
var t T
yaml.Unmarshal([]byte("a: 1\nb: 2"), &t)
```

但是有时候，数据比较多样化，结构顶部下来的时候，就可以使用接口 + type assertion 的方式来做了。
```
var t interface{}
yaml.Unmarshal([]byte("a: 1\nb: 2"), &t)
fmt.Println(t) // map[a:1 b:2]
```
我们可以看到，打印出来了一个 `map[a:1 b:2]`。
```
k := t.(map[string]interface{})
```
上面这种就是 type assertion 了，就得到了一个 `map[string]interface{}` 类型的数据了。需要注意的是，如果类型不匹配的话，
程序是会 panic 的。为了避免 panic
```
k, ok := t.(map[string]interface{})
```
加上一个 ok 就可以了。通过判断 ok 的值，可以判断是否成功。

## embedded + interface
golang 的 struct 的 field 可以是 embeded 的。
```
type bar struct {
  foo
}
```
通过 embedded 引入 struct 的成员比较有意思，如果 embedded 实现了一个借口，那么这个 struct 同样实现了这个接口。这个特性也比较有意思。
灵活使用的话，可以省很多事情。

---
介绍就这样了，这一周写 cms 用到的一些特性。很久没写上层的业务，发现上层的业务还是蛮考验代码功底的。
