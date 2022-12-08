# 7、调试 Reactor

从命令式和同步编程范式转换到反应式和异步编程范式有时会令人望而生畏。学习曲线中最陡峭的步骤之一是如何在出错时进行分析和调试。

在命令式环境中，调试通常是非常简单的：只要阅读 stacetrace，你就会发现问题的根源，以及更多：这完全是代码的失败吗？失败是否发生在某些库代码中？如果是，那么是代码的哪一部分调用了这个库，可能传入了错误的参数，最终导致了失败。

## 7.1、典型的 Reactor 堆栈跟踪

当切换到异步代码时，事情会变得更加复杂。

请考虑以下堆栈跟踪：

示例 20：一个典型的 Reactor 堆栈跟踪
```
java.lang.IndexOutOfBoundsException: Source emitted more than one item
    at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:129)
    at reactor.core.publisher.FluxFlatMap$FlatMapMain.tryEmitScalar(FluxFlatMap.java:445)
    at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext(FluxFlatMap.java:379)
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:121)
    at reactor.core.publisher.FluxRange$RangeSubscription.slowPath(FluxRange.java:154)
    at reactor.core.publisher.FluxRange$RangeSubscription.request(FluxRange.java:109)
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.request(FluxMapFuseable.java:162)
    at reactor.core.publisher.FluxFlatMap$FlatMapMain.onSubscribe(FluxFlatMap.java:332)
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onSubscribe(FluxMapFuseable.java:90)
    at reactor.core.publisher.FluxRange.subscribe(FluxRange.java:68)
    at reactor.core.publisher.FluxMapFuseable.subscribe(FluxMapFuseable.java:63)
    at reactor.core.publisher.FluxFlatMap.subscribe(FluxFlatMap.java:97)
    at reactor.core.publisher.MonoSingle.subscribe(MonoSingle.java:58)
    at reactor.core.publisher.Mono.subscribe(Mono.java:3096)
    at reactor.core.publisher.Mono.subscribeWith(Mono.java:3204)
    at reactor.core.publisher.Mono.subscribe(Mono.java:3090)
    at reactor.core.publisher.Mono.subscribe(Mono.java:3057)
    at reactor.core.publisher.Mono.subscribe(Mono.java:3029)
    at reactor.guide.GuideTests.debuggingCommonStacktrace(GuideTests.java:995)
```
那里发生了很多事情。我们得到一个 IndexOutOfBoundsException，它告诉我们”源发出了多个项“。

我们可能很快就可以假设这个源是一个 Flux/Mono，正如下面提到 MonoSingle 的一行所证实的那样。所以这似乎是来自 single 操作符的抱怨。

参考 Mono#single 操作符的 Java 文档（Javadoc），我们看到 single 有一个契约：源必须恰好发出一个元素。看来我们有一个源发出了不止一个，因此违反了该契约。

我们能更深入地挖掘并找出源头吗？以下几行不是很有用。它们通过多次调用 subscribe 和 request ，带我们进入一个似乎是反应式链的内部。

通过浏览这些行，我们至少可以开始形成一幅出问题的链条的图片：它似乎涉及一个 MonoSingle、一个 FluxFlatMap 和一个 FluxRange（每个都在跟踪中得到几行，但总的来说涉及到这三个类）。所以一个 range().flatMap().single() 链可能吗？

但是如果我们在应用程序中经常使用这种模式呢？这仍然不能告诉我们太多，简单地寻找 single 并不能找到问题所在。最后一行是我们的一些代码。最后，我们接近了。

不过，等一下。当我们转到源文件时，我们看到的是一个预先存在的 Flux 被订阅了，如下所示：

```
toDebug
     .subscribeOn(Schedulers.immediate())
    .subscribe(System.out::println, Throwable::printStackTrace);
```

所有这些都发生在订阅的时候，但是 Flux 自身并没有被声明。更糟糕的是，当我们到变量声明的地方时，我们看到：

```
public Mono<String> toDebug; //please overlook the public class attribute
```

该变量在声明的地方没有被实例化。我们必须假设最坏的情况，即我们发现可能有几个不同的代码路径在应用程序中设置它。我们仍然不确定是哪一个引起了这个问题。

>注释：
>       这类似于 Reactor 的运行时错误，而不是编译错误。

我们想更容易找出的是操作符在哪里被加入到链中——也就是说，Flux 在哪里被声明。我们通常称之为 Flux 的组装。

## 7.2、激活调试模式- 也就是回溯

>警告：
>       本节描述了启用调试功能的最简单但也是最慢的方法，因为它捕获了每个运算符的堆栈跟踪。有关更细粒度的调试方式，[请参阅替代方案( The checkpoint() Alternative)](https://projectreactor.io/docs/core/3.4.19/reference/#checkpoint-alternative)，有关更高级和性能更高的全局选项，请参阅checkpoint()[生产就绪全局调试(Production-ready Global Debugging)](https://projectreactor.io/docs/core/3.4.19/reference/#reactor-tools-debug)。

尽管堆栈跟踪仍然能够为有经验的人传递一些信息，但是我们可以看到，在更高级的情况下，它本身并不理想。

幸运的是，Reactor 附带了为调试而设计的汇编时工具。

这是通过在应用程序启动时定制 Hooks.onOperator 钩子来完成的（或者至少在引入的 Flux 或 Mono 可以被实例化之前），如下所示：
```
Hooks.onOperatorDebug();
```
这将通过包装操作符的结构并捕获那里的堆栈跟踪来开始检测对 Flux（和 Mono）操作符的调用（在那里它们被组装到链中）。由于这是在声明操作符链时完成的，所以应该在此之前激活钩子，所以最安全的方法是在应用程序开始时激活它。

稍后，如果发生异常，则失败的运算符可以引用该捕获并将其附加到堆栈跟踪。
>提示：
>       我们将此捕获的程序集信息（以及通常由 Reactor 添加到异常中的其他信息）称为traceback。

在下一节中，我们将了解堆栈跟踪的不同之处以及如何解释这些新信息。

## 7.3、在调试模式下读取堆栈跟踪

当我们重用初始示例但激活operatorStacktrace调试功能时，会发生几件事：
    
    (1)指向订阅站点并因此不太有趣的堆栈跟踪在第一帧之后被剪切并放在一边。
    (2)一个特殊的抑制异常被添加到原始异常中（或者如果已经存在则修改）。
    (3)为具有几个部分的特殊异常构造一条消息。
    (4)第一部分将追溯到失败的操作员的组装地点。
    (5)第二部分将尝试显示从该运算符构建并看到错误传播的链
    (6)最后一部分是原始堆栈跟踪

打印后的完整堆栈跟踪如下：
```
ava.lang.IndexOutOfBoundsException: Source emitted more than one item
    at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:127) 
    Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below: 
Assembly trace from producer [reactor.core.publisher.MonoSingle] : 
    reactor.core.publisher.Flux.single(Flux.java:7915)
    reactor.guide.GuideTests.scatterAndGather(GuideTests.java:1017)
Error has been observed at the following site(s): 
    *_______Flux.single ⇢ at reactor.guide.GuideTests.scatterAndGather(GuideTests.java:1017) 
    |_ Mono.subscribeOn ⇢ at reactor.guide.GuideTests.debuggingActivated(GuideTests.java:1071) 
Original Stack Trace: 
        at reactor.core.publisher.MonoSingle$SingleSubscriber.onNext(MonoSingle.java:127)
...

...
        at reactor.core.publisher.Mono.subscribeWith(Mono.java:4363)
        at reactor.core.publisher.Mono.subscribe(Mono.java:4223)
        at reactor.core.publisher.Mono.subscribe(Mono.java:4159)
        at reactor.core.publisher.Mono.subscribe(Mono.java:4131)
        at reactor.guide.GuideTests.debuggingActivated(GuideTests.java:1067)
```
    （1）原始堆栈跟踪被截断为单个帧。
    （2）这是新的：我们看到了捕获堆栈的包装运算符。这是追溯开始出现的地方。
    （3）首先，我们得到一些关于操作员组装地点的详细信息。
    （4）其次，我们得到了错误传播的运营商链的概念，从第一个到最后一个（错误站点到订阅站点）。
    （5）每个看到错误的操作员都与用户类和使用它的行一起被提及。这里我们有一个“根”。
    （6）在这里，我们有一个简单的链条部分。
     (7)堆栈跟踪的其余部分在最后移动......
     (8)…​展示了一些操作员的内部结构（所以我们在这里删除了一些片段）。

如你所见，捕获的堆栈跟踪作为抑制的 OnAssemblyException 附加到原始错误。它有两个部分，但第一部分是最有趣的。它显示了触发异常的操作符的构造路径。在这里，它显示了导致我们问题的 single 是在 scatterAndGather 方法中创建的，其本身是从通过 JUnit 执行的 populateDebug 方法调用的。

现在我们已经掌握了足够的信息来找到罪魁祸首，我们可以对 scatterAndGather 方法进行有意义的研究。
```
private Mono<String> scatterAndGather(Flux<String> urls) {
    return urls.flatMap(url -> doRequest(url))
        .single(); 
}
```
    （1）果然，这是我们的 single。

现在我们可以看到错误的根本原因是执行对几个 URL 的多个 HTTP 调用的 flatMap 和 single 链接在一起，这太过限制了。在简短的指责和与该行作者的快速讨论之后，我们发现他打算使用限制性较小的 take(1) 来代替。

我们已经解决了我们的问题。

现在考虑堆栈跟踪中的以下部分：

```
    Error has been observed at the following site(s):
```

在这个特定示例中，回溯的第二部分不一定有趣，因为错误实际上发生在链中的最后一个运算符（最接近 的那个subscribe）中。考虑另一个例子可能会更清楚：

```
akeRepository.findAllUserByName(Flux.just("pedro", "simon", "stephane"))
              .transform(FakeUtils1.applyFilters)
              .transform(FakeUtils2.enrichUser)
              .blockLast();
```

现在想象一下，在里面findAllUserByName，有一个map失败的。在这里，我们将在回溯的第二部分中看到以下内容：
```
Error has been observed at the following site(s):
    *________Flux.map ⇢ at reactor.guide.FakeRepository.findAllUserByName(FakeRepository.java:27)
    |_       Flux.map ⇢ at reactor.guide.FakeRepository.findAllUserByName(FakeRepository.java:28)
    |_    Flux.filter ⇢ at reactor.guide.FakeUtils1.lambda$static$1(FakeUtils1.java:29)
    |_ Flux.transform ⇢ at reactor.guide.GuideDebuggingExtraTests.debuggingActivatedWithDeepTraceback(GuideDebuggingExtraTests.java:39)
    |_   Flux.elapsed ⇢ at reactor.guide.FakeUtils2.lambda$static$0(FakeUtils2.java:30)
    |_ Flux.transform ⇢ at reactor.guide.GuideDebuggingExtraTests.debuggingActivatedWithDeepTraceback(GuideDebuggingExtraTests.java:40)

```

这对应于收到错误通知的运算符链部分：
    
    (1)异常起源于第一个map. 这个被*连接器识别为根，事实_用于缩进。
    (2)该异常被第二个看到map（实际上都对应于findAllUserByName方法）。
    (3)然后由 afilter和 a看到transform，这表明链的一部分是由可重用的转换函数（这里是applyFilters实用程序方法）构造的。
    (4)最后，它被 anelapsed和 a看到transform。再次elapsed由第二个变换的变换函数应用。

在某些情况下，相同的异常通过多个链传播，“根”标记*_允许我们更好地分离这些链。(observed x times)如果一个站点被多次看到，在调用站点信息之后会有一个。

例如，让我们考虑以下代码段：
```
public class MyClass {
    public void myMethod() {
        Flux<String> source = Flux.error(sharedError);
        Flux<String> chain1 = source.map(String::toLowerCase).filter(s -> s.length() < 4);
        Flux<String> chain2 = source.filter(s -> s.length() > 5).distinct();
        Mono<Void> when = Mono.when(chain1, chain2);
    }
}
```

when在上面的代码中，错误通过两个独立的链传播chain1到chain2. 这将导致包含以下内容的回溯：

```
    Error has been observed at the following site(s):
    *_____Flux.error ⇢ at myClass.myMethod(MyClass.java:3) (observed 2 times)
    |_      Flux.map ⇢ at myClass.myMethod(MyClass.java:4)
    |_   Flux.filter ⇢ at myClass.myMethod(MyClass.java:4)
    *_____Flux.error ⇢ at myClass.myMethod(MyClass.java:3) (observed 2 times)
    |_   Flux.filter ⇢ at myClass.myMethod(MyClass.java:5)
    |_ Flux.distinct ⇢ at myClass.myMethod(MyClass.java:5)
    *______Mono.when ⇢ at myClass.myMethod(MyClass.java:7)
```

我们看到：

    (1)有 3 个“根”元素（这when是真正的根）。
    (2)从 开始的两条链Flux.error是可见的。
    (3)两条链似乎都基于相同的Flux.error来源（observed 2 times）。
    (4)第一条链是Flux.error().map().filter
    (5)第二个链是`Flux.error().filter().distinct()

>提示：
> 关于回溯和抑制异常的说明：由于回溯作为抑制异常附加到原始错误，这可能会在一定程度上干扰使用此机制的另一种类型的异常：复合异常。此类异常可以通过 直接创建Exceptions.multiple(Throwable…​)，也可以由可能加入多个错误源（如Flux#flatMapDelayError）的某些运算符创建。它们可以展开到ListviaExceptions.unwrapMultiple(Throwable)中，在这种情况下，traceback 将被视为组合的一个组件，并且是返回的一部分List。如果这是不可取的，则可以通过Exceptions.isTraceback(Throwable)检查来识别回溯，并通过使用而不是从这种展开中排除Exceptions.unwrapMultipleExcludingTracebacks(Throwable)。

我们在这里处理一种检测形式，创建堆栈跟踪的成本很高。这就是为什么这个调试功能只能以受控方式激活，作为最后的手段。

### 7.3.1、checkpoint() 的替代者

调试模式是全局的，它影响到应用程序内组装成 Flux 或 Mono 的每个操作符。这样做的好处是允许事后调试：无论错误是什么，我们都将获得调试它的附加信息。

正如我们前面看到的，这种全局知道是以影响性能为代价的（由于填充的堆栈跟踪的数据）。如果我们知道可能有问题的操作符，那么这个成本可以降低。但是，我们通常不知道哪些运算符可能有问题，除非我们很容易地观察到错误，看到我们失去了装配信息，然后修改代码来激活装配跟踪，希望再次观察到相同的错误。

在这种情况下，我们必须切换到调试模式并做好准备，以便更好地观察错误的第二次发生，这一次捕获所有附加信息。

如果你能够识别在应用程序中组装的、对可维护性至关重要的反应式链，那么可以使用 checkpoint() 操作符来实现这两种技术的混合。

你可以将这个操作符链接到一个方法链中。checkpoint 操作符的工作方式与钩子版本类似，但只是针对特定链的链接。 	

还有一个 checkpoint(String) 变体，它允许你向装配回溯（assembly traceback）添加唯一的字符串标识符。通过这种方式，堆栈跟踪将被省略，你将依赖于描述来标识装配位置。与常规 checkpoint 相比，checkpoint(String) 的处理成本更低。

最后但并非最不重要的一点是，如果您想向检查点添加更通用的描述，但仍依赖堆栈跟踪机制来识别程序集站点，您可以通过使用checkpoint("description", true)版本来强制执行该行为。我们现在回到回溯的初始消息，增加了 a description，如以下示例所示：
```
Assembly trace from producer [reactor.core.publisher.ParallelSource], described as [descriptionCorrelation1234] : 
	reactor.core.publisher.ParallelFlux.checkpoint(ParallelFlux.java:215)
	reactor.core.publisher.FluxOnAssemblyTest.parallelFluxCheckpointDescriptionAndForceStack(FluxOnAssemblyTest.java:225)
Error has been observed at the following site(s):
	|_	ParallelFlux.checkpoint ⇢ reactor.core.publisher.FluxOnAssemblyTest.parallelFluxCheckpointDescriptionAndForceStack(FluxOnAssemblyTest.java:225)
```
    
    （1）descriptionCorrelation1234是中提供的描述checkpoint。

描述可以是静态标识符或用户可读描述或更广泛的相关 ID（例如，在 HTTP 请求的情况下来自标头）。

>注释：
>       当全局调试与检查点一起启用时，将应用全局调试回溯样式，检查点仅反映在“已观察到错误...​”部分。因此，在这种情况下，重检查点的名称是不可见的。

## 7.4 生产就绪的全局调试

Project Reactor 带有一个单独的 Java 代理，它可以检测您的代码并添加调试信息，而无需支付在每次操作员调用时捕获堆栈跟踪的成本。[该行为与Activating Debug Mode - aka tracebacks](https://projectreactor.io/docs/core/3.4.19/reference/#debug-activate)非常相似，但没有运行时性能开销。

要在您的应用程序中使用它，您必须将其添加为依赖项。

以下示例显示了如何reactor-tools在 Maven 中添加为依赖项：

示例 21. Maven 中的 reactor-tools，在 <dependencies>

>
>```
><dependency>
>    <groupId>io.projectreactor</groupId>
>    <artifactId>reactor-tools</artifactId>
></dependency>
>```
>   (1)如果使用BOM，则无需指定<version>.

示例 22. Gradle 中的 reactor-tools，修改 dependencies块
```
dependencies {
   compile 'io.projectreactor:reactor-tools'
}
```

它还需要显式初始化：

```
ReactorDebugAgent.init();
```

>提示：
>       由于实现将在加载类时对它们进行检测，因此最好将其放置在 main(String[]) 方法中的所有其他内容之前：

```
public static void main(String[] args) {
    ReactorDebugAgent.init();
    SpringApplication.run(Application.class, args);
}
```

如果您不能急切地运行 init（例如在测试中），您也可以重新处理现有的类：

```
ReactorDebugAgent.init();
ReactorDebugAgent.processExistingClasses();
```

>警告：
>       请注意，由于需要遍历所有加载的类并应用转换，因此重新处理需要几秒钟。仅当您看到某些呼叫站点未检测时才使用它。

### 7.4.1 限制

ReactorDebugAgent实现为 Java 代理并使用ByteBuddy执行自连接。自附加可能无法在某些 JVM 上运行，请参阅 [ByteBuddy](https://bytebuddy.net/#/) 的文档了解更多详细信息。

### 7.4.2. 将 ReactorDebugAgent 作为 Java 代理运行

如果您的环境不支持 ByteBuddy 的自连接，您可以reactor-tools作为 Java 代理运行：

```
    java -javaagent reactor-tools.jar -jar app.jar
```

###7.4.3. 在构建时运行 ReactorDebugAgent

也可以reactor-tools在构建时运行。为此，您需要将其作为 ByteBuddy 构建工具的插件应用。

>警告：
>       转换只会应用于您项目的类。不会检测类路径库。

示例 23. 带有 ByteBuddy 的 Maven 插件的reactor-tools
>```
><dependencies>
>	<dependency>
>		<groupId>io.projectreactor</groupId>
>		<artifactId>reactor-tools</artifactId>
>		1.
>		<classifier>original</classifier>  2.
>		<scope>runtime</scope>
>	</dependency>
></dependencies>
>
><build>
>	<plugins>
>		<plugin>
>			<groupId>net.bytebuddy</groupId>
>			<artifactId>byte-buddy-maven-plugin</artifactId>
>			<configuration>
>				<transformations>
>					<transformation>
>						<plugin>reactor.tools.agent.ReactorDebugByteBuddyPlugin</plugin>
>					</transformation>
>				</transformations>
>			</configuration>
>		</plugin>
>	</plugins>
></build>
>```
>  (1)如果使用BOM，则无需指定<version>.
> 
>   (2)classifier这里很重要。

示例 24. 带有 ByteBuddy 的 Gradle 插件的反应器工具

>plugins {
>    id 'net.bytebuddy.byte-buddy-gradle-plugin' version '1.10.9'
>}
> configurations {
>   byteBuddyPlugin
>}
> dependencies {
>           byteBuddyPlugin(
>                       group: 'io.projectreactor',
>                       name: 'reactor-tools',
>                        1.
>			            classifier: 'original', 2.
>	)
>}
>
>byteBuddy {
>transformation {
>plugin = "reactor.tools.agent.ReactorDebugByteBuddyPlugin"
>classPath = configurations.byteBuddyPlugin
>}
>}
>(1)     如果使用BOM，则无需指定version.
>(2)    classifier这里很重要。


## 7.5、记录序列

除了堆栈跟踪调试和分析之外，工具箱中的另一个强大工具是能够以异步序列跟踪和记录事件。

log() 操作符可以做到这一点。在一个序列中链接，它将查看其上游的 Flux 或 Mono 的每个事件（包括 onNext、onError 和 onComplete 与订阅、取消和请求）。

```
**关于日志实现的补充说明**

log 操作符使用 Loggers 工具类，该类通过 SLF4J 获取 Log4J 和 Logback 等常用日志框架，并默认在 SLF4J 不可用时记录到控制台。

控制台回退使用 System.err 作为 WARN 和 ERROR 日志级别，使用 System.out 作为其他级别。

如果你喜欢 JDK java.util.logging 回退，如 3.0.x 中所示，可以通过将 reactor.logging.fallback 系统属性设置为 JDK 来获得它。

在所有情况下，在生产环境中进行日志记录时，都应该注意配置底层日志框架，以使用其最异步和非阻塞的方法。例如，logback 中的 AsyncAppender 或 Log4j 2 中的 AsyncLogger。
```

例如，假设我们激活并配置了logback，并且有一个类似于 range(1,10).take(3) 的链。通过在 take 之前放置一个 log()，我们可以了解它是如何工作的，以及它向上传播到 range 的事件类型，如下面示例所示：

```
Flux<Integer> flux = Flux.range(1, 10)
                         .log()
                         .take(3);
flux.subscribe();
```

这将打印出以下内容（通过记录器的控制台附加程序）：
>
>    10:45:20.200 [main] INFO  reactor.Flux.Range.1 - | onSubscribe([Synchronous Fuseable] FluxRange.RangeSubscription) 1.
>    10:45:20.205 [main] INFO  reactor.Flux.Range.1 - | request(unbounded)  2.
>    10:45:20.205 [main] INFO  reactor.Flux.Range.1 - | onNext(1)  3.
>    10:45:20.205 [main] INFO  reactor.Flux.Range.1 - | onNext(2)
>    10:45:20.205 [main] INFO  reactor.Flux.Range.1 - | onNext(3)
>    10:45:20.205 [main] INFO  reactor.Flux.Range.1 - | cancel()    4.
>
>在这里，除了记录器自己的格式器（时间、线程、级别、信息）之外，log() 操作符还以自己的格式输出一些内容：
>
>    （1）reactor.Flux.Range.1 是日志的自动分类，以防在一个链中多次使用操作符。它允许你区分记录了哪些操作符的事件（在本例中是 range）。通过使用 log(String) 方法签名，可以用你自己的自定义类别覆盖标识符。在几个分隔字符之后，实际的事件被打印出来。这里有一个 onSubscribe 调用、一个 request 调用、三个 onNext 调用和一个 cancel 调用。对于第一行 onSubscribe，我们得到 Subscriber 的实现，它通常对应于特定操作符的实现。在方括号内，我们得到了额外的信息，包括是否可以通过同步或异步融合来自动优化操作符。
>    （2）在第二行，我们可以看到一个无边界的请求是从下游向上传播的。
>    （3）然后这个 range 连续发送 3 个值。
>    （4）在最后一行，我们看到了 cancel()。

最后一行，(4)，是最有趣的。我们可以看到 take 在那里活动。它的工作方式是在看到足够多的元素发出之后，将序列缩短。简而言之，take() 一旦发出用户请求的数量，它就会使源调用 cancel()。

[建议](https://translate.google.com/website?sl=zh-CN&tl=en&hl=zh-CN&client=webapp&u=https://github.com/reactor/reactor-core/edit/main/docs/asciidoc/debugging.adoc)编辑到“[调试反应堆](https://projectreactor.io/docs/core/3.4.19/reference/#debugging)”