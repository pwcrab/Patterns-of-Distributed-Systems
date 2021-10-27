# Gossip 传播（Gossip Dissemination）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/gossip-dissemination.html

使用节点的随机选择进行信息传递，以确保信息可以到达集群中的所有节点，而不会淹没网络。

**2021.6.17**

## 问题

在拥有多个节点的集群中，每个节点都要向集群中的所有其它节点传递其所拥有的元数据信息，无需依赖于共享存储。在一个很大的集群里，如果所有的服务器要和所有其它的服务器通信，就会消耗大量的网络带宽。信息应该能够到达所有节点，即便有些网络连接遇到一些问题。