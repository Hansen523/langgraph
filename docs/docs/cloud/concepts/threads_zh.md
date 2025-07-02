# 线程

一个线程包含一系列[运行](../../concepts/assistants.md#execution)累积的状态。当执行运行时，助手底层图的[状态](../../concepts/low_level.md#state)将被持久化到线程中。

可以获取线程的当前和历史状态。为了持久化状态，必须在执行运行之前创建线程。

线程在特定时间点的状态称为[检查点](../../concepts/persistence.md#checkpoints)。检查点会被持久化，可用于在以后恢复线程的状态。

## 了解更多

* 有关线程和检查点的更多信息，请参阅[LangGraph概念指南](../../concepts/persistence.md)中的相关章节。
* LangGraph平台API提供了多个端点用于创建和管理线程及线程状态。更多详情请参见[API参考文档](../../cloud/reference/api/api_ref.html#tag/threads)。