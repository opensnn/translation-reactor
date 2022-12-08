# 8.显示Reactor指标
Project Reactor是一个为提高性能和更好地利用资源而设计的库。但要真正了解系统的性能，最好能够监控其各个组件。

这就是 Reactor 提供与 Micrometer 的内置集成的原因。
>提示：如果Micrometer不在类路径上，那么度量将是一个no-op。

## 8.1调度器指标
Reactor 中的每个异步操作都是通过Threading and Schedulers中描述的 Scheduler 抽象完成的。这就是为什么监控您的调度程序很重要，注意开始看起来可疑的关键指标并做出相应的反应。

要启用调度程序指标，您需要使用以下方法：
```
Schedulers.enableMetrics();
```
>警告：检测是在创建调度程序时执行的。建议尽早调用此方法。

>提示：如果您使用的是 Spring Boot，最好在调用之前放置调用SpringApplication.run(Application.class, args)。

一旦启用了调度器指标并提供它在类路径上，Reactor 将使用 Micrometer 的支持来检测支持大多数调度器的执行器。
暴露的指标请参考Micrometer 的文档，例如：

    · executor_active_threads
    · executor_completed_tasks_total
    · executor_pool_size_threads
    · executor_queued_tasks
    · executor_secounds_{count, max, sum}

由于一个调度器可能有多个执行器，每个执行器指标都有一个reactor_scheduler_id标签。
>提示：Grafana + Prometheus 用户可以使用预构建的仪表板，其中包括线程面板、已完成的任务、任务队列和其他方便的指标。

8.2. 发布商指标
有时，能够在反应管道的某个阶段记录指标很有用。

一种方法是手动将值推送到您选择的指标后端。另一种选择是使用 Reactor 的内置指标集成Flux/Mono并解释它们。
考虑以下管道：
```
    listenToEvents()
        .doOnNext(event -> log.info("Received {}", event))
        .delayUntil(this::processEvent)
        .retry()
        .subscribe();
```
要启用此源的指标Flux（从返回listenToEvents()），我们需要打开指标收集：
```
    listenToEvents()
    .name("events")  1.此阶段的每个指标都将被标识为“事件”（可选）。
    .metrics()      2.	Flux#metricsoperator 启用指标报告，使用调用Flux#nameoperator 时提供的名称。如果Flux#name未使用运算符，则默认名称为reactor.
    .doOnNext(event -> log.info("Received {}", event))
    .delayUntil(this::processEvent)
    .retry()
    .subscribe();
```
只需添加这两个运算符就会暴露出一大堆有用的指标！
------------------------------------------------------------------------------
|        指标名称         |        类型          |         描述                  |
-------------------------------------------------------------------------------
|       [姓名].已订阅      |       柜台          | 计算已订阅了多少 Reactor 序列     |
-------------------------------------------------------------------------------
|       [姓名].请求       |    分布概要          |计算所有订阅者向指定 Flux 请求的数量，|
|                        |                    |直到至少有一个请求不受限制的数量      |
---------------------------------------------------------------------------------
| [名称].onNext.delay     |   定时器             |测量 onNext 信号之间的延迟         |
|                          |                   |  （或 onSubscribe 和第一个 onNext|
|                          |                   |  之间的延迟                      |
---------------------------------------------------------------------------------
|[名称].flow.duration      |      定时器         |乘以订阅与序列终止或取消之间经过的持   |
|                           |                    |续时间。添加状态标签以指定导致 计时器结 |
|                          |                    |束的事件( completed, completedEmpty|
|                         |                     |，error, cancelled)。              |
------------------------------------------------------------------------------------
想知道您的事件处理由于某些错误而重新启动了多少次？Read [name].subscribed，因为retry()操作员会在出错时重新订阅源发布者。

对“每秒事件数”指标感兴趣？测量[name].onNext.delay计数率。

想要在侦听器抛出错误时收到警报？[name].flow.duration有status=error标签的是你的朋友。类似地，status=completedandstatus=completedEmpty将允许您区分用元素完成的序列和用空完成的序列。

请注意，当给一个序列命名时，这个序列不能再与其他序列聚合。作为一种妥协，如果你想识别你的序列但仍然可以与其他视图聚合，你可以使用标签作为名称，(tag("flow", "events"))例如调用。

## 8.2.1. 标签
每个指标都有一个type共同的标签，该标签的值要么是要么Flux取决于Mono发布者的性质。

允许用户将自定义标签添加到他们的反应链中
```
listenToEvents()
    .name("events")     1.	此阶段的每个指标都将被标识为“事件”。
    .tag("source", "kafka")  2.将自定义标签“source”设置为值“kafka”。
    .metrics()                  3.source=kafka除上述通用标签外，所有报告的指标都将分配标签
    .doOnNext(event -> log.info("Received {}", event))
    .delayUntil(this::processEvent)
    .retry()
    .subscribe();
```
请注意，根据您使用的监控系统，使用标签时可以认为使用名称是强制性的，否则会导致两个默认命名序列之间的标签集不同。Prometheus 等一些系统可能还需要为每个具有相同名称的指标设置完全相同的标签集。

建议编辑到“公开反应堆指标”





