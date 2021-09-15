# 版本向量（Version Vector）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/version-vector.html

集群中的每个节点各自维护一组计算器，以检查并发的更新。

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

### 比较版本向量

版本向量是通过比较每个节点的版本号进行比较的。如果两个版本向量中都拥有相同节点的版本号，而且其中一个的版本号都比另一个高，则认为这个版本向量高于另一个，反之亦然。如果两个版本向量并不是都高于另一个，或是对于拥有不同集群节点的版本号，则二者可以并存。

下面是一些比较的样例。

||||
| {blue:2, green:1} | 大于 | {blue:1, green:1}|
| {blue:2, green:1} | 并存 | {blue:1, green:2}|
| {blue:1, green:1, red: 1} | 大于 | {blue:1, green:1}|
| {blue:1, green:1, red: 1} | 并存 | {blue:1, green:1, pink: 1}|

比较的实现如下：

```java
public enum Ordering {
    Before,
    After,
    Concurrent
}
class VersionVector…

  //This is exact code for Voldermort implementation of VectorClock comparison.
  //https://github.com/voldemort/voldemort/blob/master/src/java/voldemort/versioning/VectorClockUtils.java
  public static Ordering compare(VersionVector v1, VersionVector v2) {
      if(v1 == null || v2 == null)
          throw new IllegalArgumentException("Can't compare null vector clocks!");
      // We do two checks: v1 <= v2 and v2 <= v1 if both are true then
      boolean v1Bigger = false;
      boolean v2Bigger = false;

      SortedSet<String> v1Nodes = v1.getVersions().navigableKeySet();
      SortedSet<String> v2Nodes = v2.getVersions().navigableKeySet();
      SortedSet<String> commonNodes = getCommonNodes(v1Nodes, v2Nodes);
      // if v1 has more nodes than common nodes
      // v1 has clocks that v2 does not
      if(v1Nodes.size() > commonNodes.size()) {
          v1Bigger = true;
      }
      // if v2 has more nodes than common nodes
      // v2 has clocks that v1 does not
      if(v2Nodes.size() > commonNodes.size()) {
          v2Bigger = true;
      }
      // compare the common parts
      for(String nodeId: commonNodes) {
          // no need to compare more
          if(v1Bigger && v2Bigger) {
              break;
          }
          long v1Version = v1.getVersions().get(nodeId);
          long v2Version = v2.getVersions().get(nodeId);
          if(v1Version > v2Version) {
              v1Bigger = true;
          } else if(v1Version < v2Version) {
              v2Bigger = true;
          }
      }

      /*
       * This is the case where they are equal. Consciously return BEFORE, so
       * that the we would throw back an ObsoleteVersionException for online
       * writes with the same clock.
       */
      if(!v1Bigger && !v2Bigger)
          return Ordering.Before;
          /* This is the case where v1 is a successor clock to v2 */
      else if(v1Bigger && !v2Bigger)
          return Ordering.After;
          /* This is the case where v2 is a successor clock to v1 */
      else if(!v1Bigger && v2Bigger)
          return Ordering.Before;
          /* This is the case where both clocks are parallel to one another */
      else
          return Ordering.Concurrent;
  }

  private static SortedSet<String> getCommonNodes(SortedSet<String> v1Nodes, SortedSet<String> v2Nodes) {
      // get clocks(nodeIds) that both v1 and v2 has
      SortedSet<String> commonNodes = Sets.newTreeSet(v1Nodes);
      commonNodes.retainAll(v2Nodes);
      return commonNodes;
  }


  public boolean descents(VersionVector other) {
      return other.compareTo(this) == Ordering.Before;
  }

```