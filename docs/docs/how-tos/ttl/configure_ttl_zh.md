# 如何为您的LangGraph应用设置TTL（存活时间）

!!! tip "前提条件"

    本指南假设您已熟悉[LangGraph平台](../../concepts/langgraph_platform.md)、[持久化存储](../../concepts/persistence.md)和[跨线程存储](../../concepts/persistence.md#memory-store)等核心概念。

???+ note "仅限LangGraph平台"
    
    TTL功能仅适用于LangGraph平台部署版本。本指南不适用于LangGraph开源版本。

LangGraph平台会持久化保存[检查点](../../concepts/persistence.md#checkpoints)（线程状态）和[跨线程记忆](../../concepts/persistence.md#memory-store)（存储项）。通过在`langgraph.json`中配置存活时间（TTL）策略，可以自动管理这些数据的生命周期，避免数据无限累积。

## 配置检查点TTL

检查点记录了对话线程的状态。设置TTL可确保旧的检查点和线程会被自动删除。

在`langgraph.json`文件中添加`checkpointer.ttl`配置：

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200 
    }
  }
}
```

*   `strategy`: 指定过期后的处理方式。目前仅支持`"delete"`策略，即删除线程中的所有检查点。
*   `sweep_interval_minutes`: 设置系统检查过期检查点的频率（分钟）。
*   `default_ttl`: 设置检查点的默认存活时间（分钟），例如43200分钟=30天。

## 配置存储项TTL

存储项实现了跨线程的数据持久化。为存储项配置TTL有助于通过清理陈旧数据来优化内存使用。

在`langgraph.json`文件中添加`store.ttl`配置：

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```

*   `refresh_on_read`: (可选，默认为`true`) 设为`true`时，通过`get`或`search`访问项目会重置其过期计时器；设为`false`时，仅在`put`操作时刷新TTL。
*   `sweep_interval_minutes`: (可选) 设置系统检查过期项目的频率（分钟）。如未设置，则不会执行清理。
*   `default_ttl`: (可选) 设置存储项的默认存活时间（分钟），例如10080分钟=7天。如未设置，项目默认不会过期。

## 组合配置方案

您可以在同一个`langgraph.json`文件中为检查点和存储项分别配置TTL策略。示例如下：

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```

## 运行时覆盖

通过SDK方法调用（如`get`、`put`和`search`）时，可以指定特定TTL值来覆盖`langgraph.json`中的默认`store.ttl`设置。

## 部署流程

在`langgraph.json`中配置TTL后，需部署或重启LangGraph应用才能使更改生效。本地开发使用`langgraph dev`命令，Docker部署则使用`langgraph up`命令。

更多配置选项详情，请参阅[langgraph.json CLI参考文档][configuration-file]。