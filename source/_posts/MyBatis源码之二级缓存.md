---
title: MyBatis源码之二级缓存
declare: true
tags: MyBatis
abbrlink: 3f89eca3
date: 2020-07-26 14:51:37
---
### 前言
在阅读MyBatis源码前，建议先去github上clone MyBatis以及MyBatis parent源码，在本地搭建测试环境，以供idea在源码中打断点调试。

---
MyBatis二级缓存的调用逻辑位于**CachingExecutor**中

### Cache及其子类
Mybatis使用Cache及子类来囊括所有缓存实现
所有子类如下图（idea查看 类右键 -> Diagrams -> show Diagrams -> ctrl+alt+b选择所有子类）
![avatar](/images/mybatis/cache_child.png)
<!--more-->
```java
public interface Cache {
  /**
   * @return The identifier of this cache
   */
  String getId();

  /**
   * 插入缓存
   */
  void putObject(Object key, Object value);

  /**
   * 获取缓存
   */
  Object getObject(Object key);

  /**
   * 移除缓存
   */
  Object removeObject(Object key);

  /**
   * 清空缓存
   */
  void clear();

  /**
   * @return The number of elements stored in the cache (not its capacity).
   */
  int getSize();

}

```

这些子类中有4个类为缓存淘汰策略
1. SoftCache（软引用，在系统要发生内存溢出的异常之前,将会把这些对象列进回收范围之中进行第二次回收）
2. LruCache
3. FifoCache
4. WeakCache（弱引用，只能生存到下一次垃圾收集发生之前）

####  SoftCache
```java
public class SoftCache implements Cache {
  //强引用双端队列
  private final Deque<Object> hardLinksToAvoidGarbageCollection;
  //弱引用队列
  private final ReferenceQueue<Object> queueOfGarbageCollectedEntries;
  //被装饰Cache
  private final Cache delegate;
  private int numberOfHardLinks;

  ...
  @Override
  public void putObject(Object key, Object value) {
    //清除所有未加入强引用队列的软引用
    removeGarbageCollectedItems();
    delegate.putObject(key, new SoftEntry(key, value, queueOfGarbageCollectedEntries));
  }

  @Override
  public Object getObject(Object key) {
    Object result = null;
    @SuppressWarnings("unchecked") // assumed delegate cache is totally managed by this cache
    // 从被修饰的Cache中获取值
    SoftReference<Object> softReference = (SoftReference<Object>) delegate.getObject(key);
    if (softReference != null) {
      result = softReference.get();
      //如果被回收，移除被修饰Cache的键值
      if (result == null) {
        delegate.removeObject(key);
      } else {
        //如果没被回收，将弱引升级为强引用，避免被回收，供下次命中使用
        // See #586 (and #335) modifications need more than a read lock
        synchronized (hardLinksToAvoidGarbageCollection) {
          hardLinksToAvoidGarbageCollection.addFirst(result);
          if (hardLinksToAvoidGarbageCollection.size() > numberOfHardLinks) {
            hardLinksToAvoidGarbageCollection.removeLast();
          }
        }
      }
    }
    return result;
  }
 ...
  private void removeGarbageCollectedItems() {
    SoftEntry sv;
    while ((sv = (SoftEntry) queueOfGarbageCollectedEntries.poll()) != null) {
      delegate.removeObject(sv.key);
    }
  }
 // 软引用内部类，key为强引用，value为软应用
  private static class SoftEntry extends SoftReference<Object> {
    private final Object key;

    SoftEntry(Object key, Object value, ReferenceQueue<Object> garbageCollectionQueue) {
      //将value包装为软引用，并加入garbageCollectionQueue
      super(value, garbageCollectionQueue);
      this.key = key;
    }
  }
```

####  SerializedCache
通过序列化及反序列化，来进行**防御性拷贝**，即每次通过相同key获取的对象都是不同的对象，以防止A线程拿到对象后，修改对象，B线程拿到修改后的对象，关于防御性拷贝可以看 https://zhucj94.github.io/article/ad01bc3e.html
```java
public class SerializedCache implements Cache {

  private final Cache delegate;
  ...
  @Override
  public void putObject(Object key, Object object) {
    if (object == null || object instanceof Serializable) {
      delegate.putObject(key, serialize((Serializable) object));
    } else {
      throw new CacheException("SharedCache failed to make a copy of a non-serializable object: " + object);
    }
  }

  @Override
  public Object getObject(Object key) {
    Object object = delegate.getObject(key);
    return object == null ? null : deserialize((byte[]) object);
  }

  private byte[] serialize(Serializable value) {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos)) {
      oos.writeObject(value);
      oos.flush();
      return bos.toByteArray();
    } catch (Exception e) {
      throw new CacheException("Error serializing object.  Cause: " + e, e);
    }
  }

  private Serializable deserialize(byte[] value) {
    Serializable result;
    try (ByteArrayInputStream bis = new ByteArrayInputStream(value);
        ObjectInputStream ois = new CustomObjectInputStream(bis)) {
      result = (Serializable) ois.readObject();
    } catch (Exception e) {
      throw new CacheException("Error deserializing object.  Cause: " + e, e);
    }
    return result;
  }
```

####  TransactionalCache
```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  ...
   @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // 查询二级缓存
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 将查询转交给被装饰的Executor，进入前文所提到的BaseExecutor的子类
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

}
```
CachingExecutor初始化时，会初始化TransactionalCacheManager，二级缓存是由TransactionalCacheManager来管理的，TransactionalCacheManager会使用TransactionalCache来装饰Cache，**TransactionalCache的设计十分有意思，put缓存只是暂存于当前线程中，只有当commit时才会将缓存写入。**
```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();

  public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
  }

  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }

  public void putObject(Cache cache, CacheKey key, Object value) {
    getTransactionalCache(cache).putObject(key, value);
  }

  public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }

  public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.rollback();
    }
  }
  // 使用TransactionalCache装饰Cache
  private TransactionalCache getTransactionalCache(Cache cache) {
    return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
  }

}
```
```java
public class TransactionalCache implements Cache {
  private final Cache delegate;
  private boolean clearOnCommit;
  private final Map<Object, Object> entriesToAddOnCommit;
  //未命中列表
  private final Set<Object> entriesMissedInCache;

  ...
  @Override
  public Object getObject(Object key) {
    // issue #116
    // 从被装饰的Cache中获取
    Object object = delegate.getObject(key);
    // 如果值为空，加入未命中列表
    if (object == null) {
      entriesMissedInCache.add(key);
    }
    // issue #146
    if (clearOnCommit) {
      return null;
    } else {
      return object;
    }
  }

  // put只是将键值存入entriesToAddOnCommit，而不交给被装饰的Cache，从entriesToAddOnCommit名字也能看出，即提交后再添加，这样其他线程就不能看到未提交的数据。
  @Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
  }

  public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
  }
  ...
  private void flushPendingEntries() {
    //将entriesToAddOnCommit的键值交给被装饰的Cache
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    //如果未命中列表数据不在entriesToAddOnCommit中，则在被装饰的Cache存入null值，防止缓存穿透
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }

}
```
**关于flushPendingEntries我有个疑问，当A线程entriesMissedInCache存在key test，B线程插入了test且提交，此时A线程再执行delegate.putObject(test, null);，这样会造成在下一次清空缓存前，关于key的查询均为脏数据**

### 命中条件
```
  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql);
  }
```
实际是交个被装饰的Executor，调用BaseExecutor.createCacheKey，在一级缓存中详细介绍过createCacheKey，命中条件如下，与一级缓存不同在于不用相同的session，至于为什么请看Cache链的组装这一小节。
1. **相同的方法（MappedStatement）**
2. **相同sql，参数**
3. **相同查询范围，即偏移量，limit**

### 清空条件
```java
  private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    if (cache != null && ms.isFlushCacheRequired()) {
      tcm.clear(cache);
    }
  }
```
二级缓存清空，必须设置flushCache="true"。调用情况如下，即查询和修改均会清空缓存。
![avatar](/images/mybatis/2_cache_clear.png)



### Cache链的组装
组装调用位于MapperBuilderAssistant，采用建造者模式，《effective java》上提到过，当参数较多时，使用建筑中模式代替构造。
```java
  public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
```
进入CacheBuilder
```java
  public Cache build() {
    setDefaultImplementations();
    // 获取基础Cache，默认为PerpetualCache，与一级缓存一致
    Cache cache = newBaseCacheInstance(implementation, id);
    setCacheProperties(cache);
    // issue #352, do not apply decorators to custom caches
    if (PerpetualCache.class.equals(cache.getClass())) {
      for (Class<? extends Cache> decorator : decorators) {
        //使用缓存淘汰策略装饰Cache，默认为LruCache
        cache = newCacheDecoratorInstance(decorator, cache);
        setCacheProperties(cache);
      }
      cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
      cache = new LoggingCache(cache);
    }
    return cache;
  }

   private Cache setStandardDecorators(Cache cache) {
    try {
      MetaObject metaCache = SystemMetaObject.forObject(cache);
      if (size != null && metaCache.hasSetter("size")) {
        metaCache.setValue("size", size);
      }
      if (clearInterval != null) {
        cache = new ScheduledCache(cache);
        ((ScheduledCache) cache).setClearInterval(clearInterval);
      }
      if (readWrite) {
        cache = new SerializedCache(cache);
      }
      //使用日志装饰Cache，打印缓存命中率等信息
      cache = new LoggingCache(cache);
      //使用同步装饰Cache，加锁
      cache = new SynchronizedCache(cache);
      if (blocking) {
        //使用阻塞装饰Cache，当在缓存中未找到该元素时，它将对缓存键设置锁定，其他线程将等待直到该元素被填充，而不是访问数据库，即防止缓存击穿
        cache = new BlockingCache(cache);
      }
      return cache;
    } catch (Exception e) {
      throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
    }
  }
```
构建完后，Cache对象如下，注意**delegate**，**除了PerpetualCache外，均为装饰器，并且一层装饰一层，组成非标准的责任链**
![avatar](/images/mybatis/cache_ins.png)

调用栈信息如下
![avatar](/images/mybatis/2_cache_invoke.png)
即二级缓存的构建是在SqlSessionFactory初始化时，这也解释了为什么二级缓存命中条件不包含相同的session了

