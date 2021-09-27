# 追随者读取（Follower Reads）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/follower-reads.html

为来自追随者的读取请求提供服务，获取更好的吞吐和更低的延迟。

**2021.7.1**

## 问题

使用领导者和追随者模式时，如果有太多请求发给领导者，它可能会出现过载。此外，在多数据中心的情况下，客户端如果在远程的数据中心，向领导者发送的请求可能会有额外的延迟。