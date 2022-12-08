# 附录 D：Reactor 扩展

reactor-extra 工件包含额外的操作符和实用程序，适用于具有高级需求的 reactor-extra 的用户。

由于这是一个单独的工件，你需要将其明确添加到你的构建中：
```
dependencies {
     compile 'io.projectreactor:reactor-core'
     compile 'io.projectreactor.addons:reactor-extra' 
}
```
（1）除了核心之外，还添加 reactor 扩展工件。有关使用 BOM 时不需要指定版本的原因、Meven 中的用法等详细信息，请参见[入门 Reactor](https://projectreactor.io/docs/core/3.2.11.RELEASE/reference/#getting)。

## D.1、TupleUtils 和 函数式接口

reactor.function 包包含函数式接口，这些接口补充了 Java 8 Function、Predicate 和 Consumer 接口的 3 到 8 个值。

TupleUtils 提供静态方法，作为这些函数式接口的 lambdas 与相应 Tuple 上的类似接口之间的桥梁。

这允许轻松地处理任何 Tuple 的独立部分，例如：
```
.map(tuple -> {
  String firstName = tuple.getT1();
  String lastName = tuple.getT2();
  String address = tuple.getT3();

  return new Customer(firstName, lastName, address);
});
```
您可以按如下方式重写前面的示例：
```
.map(TupleUtils.function(Customer::new)); 
```
（1）因为 Customer 构造函数符合 Consumer3 函数式接口签名。

## D.2、带 MathFlux 的数学操作符

该reactor.math软件包包含一个提供数学运算符的MathFlux专用版本Flux，包括max、min、sumInt、averageDouble和其他。

## D.3、调度器

Reactor-extra 自带ForkJoinPoolScheduler（在reactor.scheduler.forkjoin包中）：它使用 JavaForkJoinPool来执行任务。