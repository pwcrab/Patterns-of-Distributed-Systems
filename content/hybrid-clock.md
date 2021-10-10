# 混合时钟（Hybrid Clock）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/hybrid-clock.html

使用系统时间戳和逻辑时间戳的组合，这样时间-日期就可以当做版本，用以排序。

**2021.6.24**

## 问题

采用[有版本的值（Versioned Value）](versioned-value.md)时，如果用 [Lamport 时钟](lamport-clock.md)当做版本，存储版本时，客户端并不知道实际的日期-时间。对于客户端而言，有时需要能够访问到像 01-01-2020 这样采用日期-时间的版本，而不仅仅是像 1、2、3 这样的整数。

