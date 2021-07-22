# 有版本的值（Versioned Value）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/versioned-value.html

Store every update to a value with a new version, to allow reading historical values.

将值的每次更新连同新版本一同存储起来，以允许读取历史值。

**2021.6.22**

## 问题

在分布式系统中，节点需要知道对于某个键值而言，哪个值是最新的。有时，它们知道过去的值，这样，它们就可以对一个值的变化做出恰当的反应了。


## 解决方案

在每个值里面都存储一个版本号。版本在每次更新时递增。这样，每次更新就都可以转换为一次新的写入，而无需阻塞读取。客户端可以读取特定版本号的对应的历史值。

考虑一个复制的键值存储的简单例子。集群的领导者处理所有对键值存储的写入。它将写入请求保存在[预写日志](write-ahead-log.md)中。预写日志使用[领导者和追随者](leader-and-followers.md)进行复制。在到达[高水位标记](high-water-mark.md)时，领导者将预写日志的条目应用到键值存储中。这是一种标准的复制方法，称之为[状态机复制（state-machine-replication）](https://en.wikipedia.org/wiki/State_machine_replication)。大多数数据系统，其背后如果像 Raft 这样的共识算法支撑，都是这样实现的。在这种情况下，键值存储中会保存一个整数的版本计数器。每次根据预写日志应用键值与值的写命令时，这个计数器都要递增。然后，它会根据递增之后的版本计数器，构建一个新的键值。这样一来，不存在值的更新，但每次写入请求都会向后面的存储中附加一个新的值。


```java
class ReplicatedKVStore…

  int version = 0;
  MVCCStore mvccStore = new MVCCStore();

  @Override
  public CompletableFuture<Response> put(String key, String value) {
      return server.propose(new SetValueCommand(key, value));
  }

  private Response applySetValueCommand(SetValueCommand setValueCommand) {
      getLogger().info("Setting key value " + setValueCommand);
      version = version + 1;
      mvccStore.put(new VersionedKey(setValueCommand.getKey(), version), setValueCommand.getValue());
      Response response = Response.success(version);
      return response;
  }
```