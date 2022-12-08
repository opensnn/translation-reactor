# 附录C:常见问题解答、最佳实践和“我如何.......”

## C.1、如何包装同步阻塞调用？

通常情况下，信息源是同步和阻塞的。在 Reactor 应用程序中要处理此类源，请应用以下模式：
```
Mono blockingWrapper = Mono.fromCallable(() -> { 
    return /* make a remote synchronous call */ 
});
blockingWrapper = blockingWrapper.subscribeOn(Schedulers.elastic()); 
```
    （1）使用 fromCallable 创建一个新的 Mono。
    （2）返回异步阻塞资源。
    （3）确保每个订阅都发生在 Schedulers.elastic() 的专用单线程工作进程上。

你应该使用 Mono，因为源返回一个值。你应该使用 Schedulers.elastic，因为它创建了一个专用线程来等待阻塞资源，而不占用其他资源。

注意 subscribeOn 不订阅 Mono。它指定发生订阅调用时使用哪种 Scheduler。

## C.2、我在我的 Flux 上使用了操作符但好像没有应用。出了什么事？

确保 .subscribe() 的变量已经受到你认为应该应用到它的操作符的影响。

Reactor 操作符是装饰者。它们返回一个包装源序列并添加行为的不同实例。这就是为什么使用操作符的首选方法是链接调用。

比较以下两个示例：

没有链接(错误的)
```
Flux<String> flux = Flux.just("foo", "chain");
flux.map(secret -> secret.replaceAll(".", "*")); 
flux.subscribe(next -> System.out.println("Received: " + next));
```
（1）错误就在这里。结果没有附加到 flux 变量。

没有链接(正确的)
```
Flux<String> flux = Flux.just("foo", "chain");
flux = flux.map(secret -> secret.replaceAll(".", "*"));
flux.subscribe(next -> System.out.println("Received: " + next));
```
这个示例甚至更好（因为它更简单）：

用链接(最好的)
```
Flux<String> secrets = Flux
  .just("foo", "chain")
  .map(secret -> secret.replaceAll(".", "*"))
  .subscribe(next -> System.out.println("Received: " + next));
```
第一个版本将输出：

    Received: foo
    Received: chain

而其他两个版本将输出预期的：

    Received: ***
    Received: *****

## C.3、我的 Mono zipWith/zipWhen 从未被调用

例子
```
myMethod.process("a") // this method returns Mono<Void>
        .zipWith(myMethod.process("b"), combinator) //this is never called
        .subscribe();
```
如果源 Mono 是空的或 Mono<Void>（Mono<Void> 对于所有意图和目的都是空的），则永远不会调用某些组合。

对于 zip 静态方法或 zipWith/zipWhen 操作符之类的任何转换器，这都是典型的情况，根据定义，这些运算符需要来自每个源的元素来生成其输出。

因此，在 zip 源上使用数据抑制操作符是有问题的。数据抑制操作符的示例包括：then()、thenEmpty(Publisher<Void>)、ignoreElements() 和 ignoreElement()、when(Publisher…)。

类似地，使用 Function<T,?> 来调整其行为的操作符，如 flatMap，确实需要至少发出一个元素才能使 Function 有机会应用。在空序列（或 <Void>）上应用这些函数永远不会生成元素。

你可以使用 .defaultIfEmpty(T) 和 .switchIfEmpty(Publisher<T>) 分别用默认值或回退 Publisher<T> 替换空的 T 序列，这有助于避免某些情况。注意，这不适用于 Flux<Void>/Mono<Void> 源，因为你只能切换到另一个 Publisher<Void>，它仍然保证为空。下面是 defaultIfEmpty 的一个示例：

在 zipWhen 之前使用 defaultIfEmpty
```
myMethod.emptySequenceForKey("a") // this method returns empty Mono<String>
        .defaultIfEmpty("") // this converts empty sequence to just the empty String
        .zipWhen(aString -> myMethod.process("b")) //this is called with the empty String
        .subscribe();
```
## C.4、如何使用 retryWhen 模拟 retry(3)

retryWhen 操作符可能相当复杂。希望这段代码可以通过尝试模拟更简单的 retry(3) 来帮助你理解它是如何工作的：
```
Flux<String> flux =
Flux.<String>error(new IllegalArgumentException())
    .retryWhen(companion -> companion
    .zipWith(Flux.range(1, 4), 
          (error, index) -> { 
            if (index < 4) return index; 
            else throw Exceptions.propagate(error); 
          })
    );
```
    （1）技巧一：使用 zip 和“可接受重试次数+1”的范围。
    （2）zip 函数允许你计算重试次数，同时跟踪原始错误。
    （3）为了允许三次重试，4 之前的索引返回一个要发出的值。
    （4）为了错误地终止序列，我们在这三次重试后抛出原始异常。

## C.5、如何使用 retryWhen 指数退避？

指数退避会产生重试尝试，每次尝试之间的延迟会越来越大，这样就不会使源系统过载，并可能导致全面崩溃。其理由是，如果源产生错误，则它已经处于不稳定状态，不太可能立即从中恢复。因此，盲目地立即重试可能会产生另一个错误，并增加不稳定性。

自 3.2.0.RELEASE 以来，Reactor 提供了这样一个重新烘焙（retry baked）：Flux.retryBackoff。

对于好奇的人来说，这里是如何用 retryWhen 实现指数退避，它延迟重试并增加每次尝试之间的延迟（伪代码: 延迟 = 尝试数 * 100 毫秒）：
```
Flux<String> flux =
Flux.<String>error(new IllegalArgumentException())
    .retryWhen(companion -> companion
        .doOnNext(s -> System.out.println(s + " at " + LocalTime.now())) 
        .zipWith(Flux.range(1, 4), (error, index) -> { 
          if (index < 4) return index;
          else throw Exceptions.propagate(error);
        })
        .flatMap(index -> Mono.delay(Duration.ofMillis(index * 100))) 
        .doOnNext(s -> System.out.println("retried at " + LocalTime.now())) 
    );
```
    （1）我们记录错误发生的时间。
    （2）我们使用 retryWhen + zipWith 技巧在 3 次重试后传播错误。
    （3）通过 flatMap，我们造成的延迟取决于尝试的索引。
    （4）我们还记录重试发生的时间。

当订阅时，这将失败并在打印出来后终止：

    java.lang.IllegalArgumentException at 18:02:29.338
    retried at 18:02:29.459 
    java.lang.IllegalArgumentException at 18:02:29.460
    retried at 18:02:29.663 
    java.lang.IllegalArgumentException at 18:02:29.663
    retried at 18:02:29.964 
    java.lang.IllegalArgumentException at 18:02:29.964
```
（1）大约 100 毫秒后首次重试
（2）大约 200 毫秒后的第二次重试
（3）大约 300 毫秒后第三次重试
```
## C.6、如何使用 publishOn() 确保线程关联性？

如调度程序中所述，publishOn() 可用于切换执行上下文。publishOn 操作符影响线程上下文，它下面的链中的其他操作符将在其中执行，直到 publishOn 的新出现。因此，publishOn 的位置是重要的。

例如，在下面的示例中，map() 中的 transform 函数是在 scheduler1 的工作线程上执行的，而 doOnNext() 中的 processNext 方法是在 scheduler2 的工作线程上执行的。每个 subscription 都有自己的工作线程，因此推送到相应订阅服务器的所有元素都发布在相同 Thread 上。

单线程调度程序可用于确保链中不同阶段或不同订阅者的线程相关性。
```
EmitterProcessor<Integer> processor = EmitterProcessor.create();
processor.publishOn(scheduler1)
         .map(i -> transform(i))
         .publishOn(scheduler2)
         .doOnNext(i -> processNext(i))
         .subscribe();
```

## C.7什么是上下文日志记录的好模式？(MDC)
大多数日志记录框架允许上下文日志记录，让用户存储反映在日志记录模式中的变量，通常通过Map称为 MDC（“映射诊断上下文”）的方式。这是 Java 中最常见的用法之一ThreadLocal，因此该模式假定被记录的代码与Thread.

在 Java 8 之前，这可能是一个安全的假设，但随着 Java 语言中函数式编程元素的出现，情况发生了一些变化......

让我们以一个命令式 API 为例，它使用模板方法模式，然后切换到更函数式的风格。在模板方法模式中，继承发挥了作用。现在在其更函数式的方法中，传递高阶函数来定义算法的“步骤”。现在，声明式的多于命令式的，这让图书馆可以自由决定每个步骤应该在哪里运行。例如，知道底层算法的哪些步骤可以并行化，库可以使用 来并行ExecutorService执行一些步骤。

这种功能性 API 的一个具体示例是StreamJava 8 中引入的 API 及其parallel()风格。并行使用 MDC 进行日志记录Stream不是免费的午餐：需要确保在每个步骤中捕获并重新应用 MDC。

函数式风格支持这样的优化，因为每个步骤都是线程不可知的并且引用透明，但它可以打破单个Thread. 确保所有阶段都可以访问任何类型的上下文信息的最惯用的方法是通过组合链传递该上下文。在 Reactor 的开发过程中，我们遇到了同样的一般问题，我们希望避免这种非常简单和明确的方法。这就是引入 的原因：它通过执行链传播，Context只要被用作返回值，通过让阶段（运算符）查看其下游阶段的 。因此，Reactor 没有使用 ，而是提供了这个类似地图的对象，它绑定到FluxMonoContextThreadLocalSubscription而不是Thread.

既然我们已经确定 MDC“正常工作”并不是在声明性 API 中做出的最佳假设，那么我们如何才能执行与 Reactive Stream（ 、 和 ）中的事件相关的上下文化onNext日志onError语句onComplete？

当人们想要以直接和明确的方式记录与这些信号相关的信息时，FAQ 的这个条目提供了一种可能的中间解决方案。确保事先阅读将上下文添加到反应序列部分，尤其是写入必须如何发生在操作符链的底部，以便其上方的操作符能够看到它。

要从 MDC 获取上下文信息，最简单的方法是使用少量样板代码Context将日志记录语句包装在运算符中。doOnEach此样板取决于您选择的日志记录框架/抽象以及您要放入 MDC 的信息，因此它必须在您的代码库中。

以下是围绕单个 MDC 变量并专注于记录onNext事件的辅助函数示例，使用 Java 9 增强型OptionalAPI：
```
public static <T> Consumer<Signal<T>> logOnNext(Consumer<T> logStatement) {
	return signal -> {
		if (!signal.isOnNext()) return;  1.doOnEach信号包括onComplete和onError。在此示例中，我们只对日志记录感兴趣onNext
		Optional<String> toPutInMdc = signal.getContext().getOrEmpty("CONTEXT_KEY");        2.	我们将从 Reactor 中提取一个有趣的值（Context请参阅API部分）Context

		toPutInMdc.ifPresentOrElse(tpim -> {        3.	我们MDCCloseable在这个例子中使用了 from SLF4J 2，允许 try-with-resource 语法在执行日志语句后自动清理 MDC
			try (MDC.MDCCloseable cMdc = MDC.putCloseable("MDC_KEY", tpim)) { 
				logStatement.accept(signal.get());  4.正确的日志语句由调用者作为Consumer<T>（onNext 值的消费者）提供
			}
		},
		() -> logStatement.accept(signal.get()));   5.如果预期的密钥没有设置，Context我们使用替代路径，其中没有在 MDC 中放置任何内容
	};
}
```
使用此样板代码可确保我们成为 MDC 的好公民：我们在执行日志语句之前设置一个密钥，并在执行后立即将其删除。不存在为后续日志语句污染 MDC 的风险。

当然，这是一个建议。您可能有兴趣从 中提取多个值Context或在onError. 您可能想要为这些情况创建额外的辅助方法，或者设计一个使用额外的 lambda 来覆盖更多领域的单一方法。

在任何情况下，前面的辅助方法的使用都可能类似于以下响应式 Web 控制器：

```text
@GetMapping("/byPrice")
public Flux<Restaurant> byPrice(@RequestParam Double maxPrice, @RequestHeader(required = false, name = "X-UserId") String userId) {
	String apiId = userId == null ? "" : userId;    1.	我们需要从请求标头中获取上下文信息以将其放入Context

	return restaurantService.byPrice(maxPrice))
			   .doOnEach(logOnNext(r -> LOG.debug("found restaurant {} for ${}",  2.	在这里，我们将我们的辅助方法应用于Flux，使用doOnEach。请记住：运算符会看到Context在它们下面定义的值。
					r.getName(), r.getPricePerPerson())))
			   .contextWrite(Context.of("CONTEXT_KEY", apiId));     3.Context我们使用选择的键将标头中的值写入CONTEXT_KEY。
}
```

在此配置中，restaurantService可以在共享线程上发出其数据，但日志仍将X-UserId为每个请求引用正确的数据。

为了完整起见，我们还可以看到错误记录助手的样子：

```text
public static Consumer<Signal<?>> logOnError(Consumer<Throwable> errorLogStatement) {
	return signal -> {
		if (!signal.isOnError()) return;
		Optional<String> toPutInMdc = signal.getContext().getOrEmpty("CONTEXT_KEY");

		toPutInMdc.ifPresentOrElse(tpim -> {
			try (MDC.MDCCloseable cMdc = MDC.putCloseable("MDC_KEY", tpim)) {
				errorLogStatement.accept(signal.getThrowable());
			}
		},
		() -> errorLogStatement.accept(signal.getThrowable()));
	};
}
```
没有太大变化，除了我们检查Signal是有效的onError，并且我们向Throwable日志语句 lambda 提供所述错误 (a) 之外。

在控制器中应用这个助手与我们之前所做的非常相似：
```text
@GetMapping("/byPrice")
public Flux<Restaurant> byPrice(@RequestParam Double maxPrice, @RequestHeader(required = false, name = "X-UserId") String userId) {
	String apiId = userId == null ? "" : userId;

	return restaurantService.byPrice(maxPrice))
			   .doOnEach(logOnNext(v -> LOG.info("found restaurant {}", v))
			   .doOnEach(logOnError(e -> LOG.error("error when searching restaurants", e))  1.	如果restaurantService发出错误，它将在此处使用 MDC 上下文记录
			   .contextWrite(Context.of("CONTEXT_KEY", apiId));
}
```