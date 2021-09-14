# 版本向量（Version Vector）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/version-vector.html

维护一组计数器，每个集群节点一个，以检查并发的更新。

**2021.6.29**

## 问题

如果允许多个服务器对同样的键值进行更新，那么有值在一组副本中并发地更新就显得非常重要了。

## 解决方案

每个键值都同一个版本向量关联在一起，版本向量为集群的每个节点维护一个数字。

从本质上说，版本向量就是一组计数器，每个节点一个。三节点（blue, green, black）的版本向量可能看上去是这样：[blue: 43, green: 54, black: 12]。每次一个节点有内部更新，它都会更新它自己的计数器，因此，green 节点有更新，就会将版本向量修改为[blue: 43, green: 55, black: 12]。两个节点通信时，它们会同步彼此的向量时间戳，这样就检测出任何同步的更新。

一个典型的版本向量实现是下面这样：

```java
class VersionVector…

  private final TreeMap<String, Long> versions;

  public VersionVector() {
      this(new TreeMap<>());
  }

  public VersionVector(TreeMap<String, Long> versions) {
      this.versions = versions;
  }

  public VersionVector increment(String nodeId) {
      TreeMap<String, Long> versions = new TreeMap<>();
      versions.putAll(this.versions);
      Long version = versions.get(nodeId);
      if(version == null) {
          version = 1L;
      } else {
          version = version + 1L;
      }
      versions.put(nodeId, version);
      return new VersionVector(versions);
  }

```

存储在服务器上的每个值都关联着一个版本向量

```java
class VersionedValue…

  public class VersionedValue {
      String value;
      VersionVector versionVector;

      public VersionedValue(String value, VersionVector versionVector) {
          this.value = value;
          this.versionVector = versionVector;
      }

      @Override
      public boolean equals(Object o) {
          if (this == o) return true;
          if (o == null || getClass() != o.getClass()) return false;
          VersionedValue that = (VersionedValue) o;
          return Objects.equal(value, that.value) && Objects.equal(versionVector, that.versionVector);
      }

      @Override
      public int hashCode() {
          return Objects.hashCode(value, versionVector);
      }

```
