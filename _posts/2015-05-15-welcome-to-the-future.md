---
layout: post
keywords: blog, scala
description: blog
title: "译文：欢迎来到未来"
categories: [Translation]
tags: [Scala, Translation]
group: archive
icon: file-o
featured: true
---
## 译者前言
xp最近迷上了Scala这门第一眼看起来像是 *Java with Haskell* 的语言。刚学习完 [Martin Odersky](http://en.wikipedia.org/wiki/Martin_Odersky) 在Coursera上的关于Scala函数式编程语言的课程后，惊喜的发现作者还有一门进阶的 [Principles of Reactive Programming](https://class.coursera.org/reactive-002) 课程。

这一篇博客翻译自来自 [Daniel Westheide](http://danielwestheide.com/) 的 Scala 课程系列的第八篇 **[Welcome to the Future](http://danielwestheide.com/blog/2013/01/09/the-neophytes-guide-to-scala-part-8-welcome-to-the-future.html)**，其“生动形象”的介绍了 Scala 的 Future 的使用。所有权利由原作者 Daniel Westeide 保留。

正文译文
-----
作为一个有远见有热情的Scala开发者，你也许已经听说过Scala的并发策略了，或者这正是Scala吸引到你的首要原因。基于Scala的语言特性支持，描述一个并发问题变得很简单，并且编写一个规范的并发程序要远比其他使用底层并发API接口的语言要容易许多。

Scala实现并发机制的基石叫做*Future*，另一个是*Actor*。这篇文章的主题是Future，我会向你以函数编程的方式介绍Future的强大之处。

为了让本篇文章的例子运行，你请确保你正在使用Scala 2.9.3版本或更高版本。本篇文章中所讨论的Future版本是在2.10.0中被正式加入Scala，并在之后向后移植到了2.93版本中。最开始的时候，Future是作为Akka并发工具包早前版本的一部分。需要注意的是，Akka中的Future API与Scala的目前的版本稍有区别。

-----

## 顺序执行代码的缺点

假设你要准备一杯卡普奇诺咖啡。你可以简单的一步步执行以下步骤：

1. 把备好的咖啡豆磨成粉
1. 烧水
1. 用刚才磨好的咖啡粉和热水调制一杯浓缩咖啡
1. 打一些奶泡
1. 将浓缩咖啡和奶泡混合，完成一杯卡普奇诺咖啡

以上步骤翻译成Scala：

```scala

import scala.util.Try

// Some type aliases, just for getting more meaningful method signatures:
type CoffeeBeans = String
type GroundCoffee = String

case class Water(temperature: Int)

type Milk = String
type FrothedMilk = String
type Espresso = String
type Cappuccino = String
// dummy implementations of the individual steps:
def grind(beans: CoffeeBeans): GroundCoffee = s"ground coffee of $beans"
def heatWater(water: Water): Water = water.copy(temperature = 85)
def frothMilk(milk: Milk): FrothedMilk = s"frothed $milk"
def brew(coffee: GroundCoffee, heatedWater: Water): Espresso = "espresso"
def combine(espresso: Espresso, frothedMilk: FrothedMilk): Cappuccino = "cappuccino"

// some exceptions for things that might go wrong in the individual steps
// (we'll need some of them later, use the others when experimenting
// with the code):
case class GrindingException(msg: String) extends Exception(msg)

case class FrothingException(msg: String) extends Exception(msg)

case class WaterBoilingException(msg: String) extends Exception(msg)

case class BrewingException(msg: String) extends Exception(msg)

// going through these steps sequentially:
def prepareCappuccino(): Try[Cappuccino] = for {
  ground <- Try(grind("arabica beans"))
  water <- Try(heatWater(Water(25)))
  espresso <- Try(brew(ground, water))
  foam <- Try(frothMilk("milk"))
} yield combine(espresso, foam)
```

这样的做法有几点优势：你会得到一系列具有很高可读性、循序渐进的指令。并且，由于规避了上下文切换，你几乎不大可能在调制卡普奇诺的过程中找不到北。

反过来看这样做的缺点：按顺序依次执行意味着在调制咖啡过程中的每个阶段，你的大脑和身体都处于阻塞状态，也就意味着在每个任务流程完成前，什么也做不了了。只有完成了前一个任务，你才可以开始比如烧热水以及剩下的步骤。

显然，宝贵资源在等待的过程中浪费掉了。为了解决这个问题，你也许会向同时启动多个步骤让它们并发执行。比方说，当你看到咖啡粉磨好而且水烧开后，你才开始冲泡浓缩咖啡，在此同时你也启动了牛奶打泡机。

软件开发与准备一杯卡普奇诺咖啡并没有什么两样。一个网络服务器有许多处理HTTP请求和生成响应的线程。你自然不想让这些宝贵的线程们在等待一条数据库查询或者调用另一个HTTP服务的时候被阻塞。为此，你转而使用非同步编程模型和非阻塞型的IO，使得服务器在处理数据库查询请求并返回结果的等待过程中，服务器线程依然可以同时处理其他的请求。

> 我听说你喜欢回调函数，那我就在你的回调函数里放进我的回调函数！

显然，你已经听说过回调函数的各种问题 - 这正是Node.js被许多很cool的同学所诟病的地方。Node.js还有一些其他的库大量的使用回调函数同其他服务进行交互。但遗憾的是，这种策略很容易产生在回调函数中调用回调函数所引发的一团混乱，使得代码阅读和除虫变得十分困难。

接下来，你马上就会看到Scala的`Future`类也允许回调函数，而且Scala还提供了一些更方便的替代方案。靠它们，你很可能会在之后抛弃回调函数。

> 我知道Futures，它们根本就是毫无用处吗！

你也许已经了解了其他的语言中的`Future`实现，比如Java。但是Java所提供的`Future`类并没有提供太多有用的功能：检查它们是否完成，或者等待它们完成后做一些事情。简而言之，Java的Future类几乎就是毫无用处的，也就导致没有人喜欢用他们。

如果你觉得Scala的Future也像是Java一样的实现，那就准备好大吃一惊吧。黑喂狗！

-----

## Future的语义

Scala的`Future[T]`类属于`scala.concurrent`包。`Future[T]` 是一种容器类型，表示为一个*最终输出类型为 `T` 的计算过程*。额，计算过程中也许会出错或者超时 - 如果Future完成时，他的结果有可能根本没有执行成功，这时候结果所包含的就是一个异常对象。

`Future` 是一个一次性写入的容器。当一个future执行完成后，这个容器是不可更改的。而且，`Future` 只提供了一个可以读取计算结果的接口。把计算结果写入一个future的任务是通过 `Promise` 对象完成的。因此，这两个类在API设计中有着清晰的区分。在这篇文章中，我们着重介绍前者(`Future`)。`Promise`类会在下一篇系列教程中介绍。

-----

## 编写Futures

使用Scala futures有多种编写方式，接下来我们会用`Future`类重写之前的卡普奇诺咖啡例子来展示它们。首先，把所有函数改写成返回含有阻塞计算过程的`Future`对象的函数，使得这些函数可以并发执行。

```scala
import scala.concurrent.future
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.util.Random

def grind(beans: CoffeeBeans): Future[GroundCoffee] = Future {
  println("start grinding...")
  Thread.sleep(Random.nextInt(2000))
  if (beans == "baked beans") throw GrindingException("are you joking?")
  println("finished grinding...")
  s"ground coffee of $beans"
}

def heatWater(water: Water): Future[Water] = Future {
  println("heating the water now")
  Thread.sleep(Random.nextInt(2000))
  println("hot, it's hot!")
  water.copy(temperature = 85)
}

def frothMilk(milk: Milk): Future[FrothedMilk] = Future {
  println("milk frothing system engaged!")
  Thread.sleep(Random.nextInt(2000))
  println("shutting down milk frothing system")
  s"frothed $milk"
}

def brew(coffee: GroundCoffee, heatedWater: Water): Future[Espresso] = Future {
  println("happy brewing :)")
  Thread.sleep(Random.nextInt(2000))
  println("it's brewed!")
  "espresso"
}
```

这段代码有几个地方需要进一步解释。

首先，`Future`类在它的[伴生对象](http://daily-scala.blogspot.jp/2009/09/companion-object.html)中有一个两个柯里化参数的工厂方法 -- `apply`:

```scala
object Future {
  def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T]
}
```

非同步执行的计算过程被作为 *按变量名* 参数传递给第一个参数。第二个参数的类型为 `implicit`，意味着只要我们在上下文中的某处定义了一个 `implicit` 值后，这个参数会自动匹配到这个值。一般来说，只要保证导入了全局执行上下文对象(`import ExecutionContext.Implicits.global`)就可以了。

一个 `ExecutionContext` 对象是一个执行future的上下文，你可以把它看做成一个类似于线程池的东西。上面说定义了一个隐含的 `ExecutionContext` 对象，剩下来我们只要传入第一个参数就可以了。人们通常会把第一个参数可以被花括号`{}`包裹而不是圆括号`()`。这样做得原因是，花括号使得代码看起来更像是在使用Scala的语言特性，而不是调用一个常规的方法。对于所有 `Future` 的API来说，所有接口都有一个 `implicit` 类型的 `ExecutionContext` 参数。

另外，在上面的例子里，我们实际上没有真的计算任何东西。为了展示目的，我们在“计算”过程里加入了一些随机时长的sleep。再加入了一些打印命令，这样你在跑上面的代码的时候可以更清晰感受一下这段代码实际*运算*时的非确定性和并发特征。

返回的 `Future` 对象内的计算过程会在 `Future` 对象实例化之后的某个不确定的时间，放入到由 `ExecutionContext` 所管理的某个线程内执行。

## 回调函数

当执行简单任务的时候，使用回调函数完全合适的。future中的回调函数的类型是**部分函数**(partial function)。你可以把一个回调函数传入到future的 `onSuccess` 方法内。当且仅当 `Future` 成功执行完成后，计算好的数值才会作为参数传入回调函数:

```scala
grind("arabica beans").onSuccess { case ground =>
  println("okay, got my ground coffee")
}
```

类似的，你也可以注册一个失败时调用的回调函数注册到 `onFailure` 方法内。这个回调函数的参数是一个 `Throwable` 对象，它只会在 `Future` 执行失败的时候会被调用到。

通常来说，最好的方式是整合两个回调函数然后把它们注册到`onComplete`中。这样的话，回调函数的参数类型为 `Try`：

```scala
import scala.util.{Success, Failure}
grind("baked beans").onComplete {
  case Success(ground) => println(s"got my $ground")
  case Failure(ex) => println("This grinder needs a replacement, seriously!")
}
```

因为在上面的代码里，传入 `grind` 函数的值是"焗豆""(而不是咖啡豆)，`grind` 将在执行时抛出异常，导致 `Future` 的结果为一个 `Failure` 对象。

## 组合future对象

以嵌套的方式使用回调函数会引发各种令人头疼的问题。幸运的是，在Scala中我们完全可以不必这样做！发挥Scala `future` 真正实力的特性是因为**future对象是可以组合的**。

如果你阅读过这个系列教程的前面的文章，你会注意到 `Future` 容器可以使用 `map`、`flatMap`、`filter` 或者 `for` 推导式（笔者：也就是说是一个完备的`Monad`模式）。
因此，Scala可以做到接下来所说的特性实际上不会让你惊讶才对！

留给我们现在的问题是：既然我们可以使用容器的那些操作方法，对于还没有执行完成的 `Future` 类来说这些方法意味着什么呢？

## 映射未来

你是不是总会幻想自己是一个可以改变未来事件的时间旅行者呢？作为一个Scala开发者，你真的可以办得到！假设当你的水烧开时，你想检查水温是否达到要求，你可以把你的 `Future[Water]` 对象映射成一个 `Future[Boolean]` 对象：

```scala
val temperatureOkay: Future[Boolean] = heatWater(Water(25)).map { water =>
  println("we're in the future!")
  (80 to 85).contains(water.temperature)
}
```

定值 `temperatureOkay` 被赋予了一个 `Future[Boolean]` 对象。这个对象在结果成功地计算完成后，会含有一个布尔值。你可以试着把 `heatWater` 方法实现修改一下让它抛出异常(比方说烧水器炸掉了)，然后看一下这会让未来发生什么样的变化: console里面没有打印出 `we're in the future!`。

当你把写好的函数通过 `map` 方法传入future对象后，这意味着这个函数将在未来，或者说是某一种未来的状态中被调用。映射函数(mapping)在你定义的 `Future` 对象成功完成后立刻执行；但在映射触发时的那个时间线并不是你现在所处的那一个。如果你的 `Future[Water]` 挂掉了，你通过 `map` 传入的函数将永远不会被触发，取而代之的，它会返回一个包含了 `Failure` 结果的一个 `Future[Boolean]` 对象。

## 保持扁平化的未来

如果一个 `Future` 对象的结果依赖于另外一个 `Future` 对象，你也许转用 `flatMap` 方法来避免深层嵌套的future结构。

比方说，我们假设测量温度的步骤会需要一段时间来完成，因此你也想实现一个非同步检查温度的方法。这样，你写了一个 `Water` 对象作为参数，`Future[Boolean]` 作为返回值的方法：

```scala
def temperatureOkay(water: Water): Future[Boolean] = Future {
  (80 to 85).contains(water.temperature)
}
```

调用 `flatMap` 而不是 `map`，使得我们得到一个 `Future[Boolean]` 而不是 `Future[Future[Boolean]]`：

```scala
val nestedFuture: Future[Future[Boolean]] = heatWater(Water(25)).map {
  water => temperatureOkay(water)
}
val flatFuture: Future[Boolean] = heatWater(Water(25)).flatMap {
  water => temperatureOkay(water)
}
```

同样的，映射函数只有在 `Future[Water]` 实例在成功执行后才会被调用（在水温可以接受时）。

## for 推导式

在 `flatMap` 调用之外，我们还可以使用 `for` 推导式来实现本质相同但阅读更清晰的代码。我们上面的例子可以重写成:

```scala
val acceptable: Future[Boolean] = for {
  heatedWater <- heatWater(Water(25))
  okay <- temperatureOkay(heatedWater)
} yield okay
```

如下，假设你有多个可以并发执行的计算过程，注意，以下代码中我们在 `for` 推导式内实例化 `Future` 对象：

```scala
def prepareCappuccinoSequentially(): Future[Cappuccino] = {
  for {
    ground <- grind("arabica beans")
    water <- heatWater(Water(20))
    foam <- frothMilk("milk")
    espresso <- brew(ground, water)
  } yield combine(espresso, foam)
}
```

这段代码读起来很棒，但由于 `for` 推导式是 `flatMap` 的另一种表现方式，这意味着 `heatWater` 中创建的 `Future[Water]` 对象只有在 `Future[GroundCoffee]` 成功完成后才会实例化（笔者注：这样会导致我们所创建的future计算过程仍然按照顺序执行，而不是非同步执行），你可以运行一下上面的代码并监控控制台的输出，检查一下是否是按照固定顺序执行。

因此，为了并发正确触发，请保证所有的future对象在 `for` 推导式之前已经被实例化：

```scala
def prepareCappuccino(): Future[Cappuccino] = {
  val groundCoffee = grind("arabica beans")
  val heatedWater = heatWater(Water(20))
  val frothedMilk = frothMilk("milk")
  for {
    ground <- groundCoffee
    water <- heatedWater
    foam <- frothedMilk
    espresso <- brew(ground, water)
  } yield combine(espresso, foam)
}
```

现在，我们在 `for` 推导式之前创建的 `Future` 会在创建后立即同步执行。通过观察控制台，你会发现多次运行结果的顺序是不确定的。唯一可以认定的事情是，"happy brewing"总是在输出的最后一行出现，因为只有它是在我们的 `for` 推导式之内所创建出来的。也就是说，只有当三个`for`外面的`future`完全执行完成后，`combine` 才会在最后执行。

## 失败的投影

你已经知道了 `Future[T]` 是偏向于成功的(success-biased)。基于计算过程正确完成这个假设，这使得允许你可以自由使用 `map`、`flatMap`、`filter` 以及其他 `Future` 的相关接口。有时候，你想对计算结果中可能失败的地方以**优美的函数式编程的方式**进行一些特别的处理。通过调用`Future[T]`的`failed`方法，你可以得到一个失败结果的投影，也就是 `Future[Throwable]`，然后通过调用它的 `map` 方法传入例如只接受失败结果的函数。

-----

## 前景

你已经看到了未来，而且他看起来还不赖！事实上你完全可以把它看做成另一种普通的容器类型。通过组合，以函数编程的方式，使用 `Future` 会是一段良好的体验。

编写并发执行的阻塞代码很简单，只需把需要执行的阻塞代码通过future的工厂方法创建新的 `Future` 对象。但是，最好从开始就不要把代码设计成会有阻塞的代码。为了实现它，我们必须通过使用 `Promise` 类来完成一个 `Future` 对象。实际使用future对象的方式会在此系列的下一部分中介绍。
