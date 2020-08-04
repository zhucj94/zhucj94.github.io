---
title: MyBatis源码之一级缓存
tags: MyBatis
declare: true
abbrlink: 7dd9b5ec
date: 2020-07-19 17:39:32
---
### 前言
在阅读MyBatis源码前，建议先去github上clone MyBatis以及MyBatis parent源码，在本地搭建测试环境，以供idea在源码中打断点调试。

---
MyBatis一级缓存的调用逻辑位于**BaseExecutor**中

### 存储结构
```java
protected PerpetualCache localCache
```
PerpetualCache是通过HashMap来存储的
```java
public class PerpetualCache implements Cache {

  private final String id;

  private final Map<Object, Object> cache = new HashMap<>();
  ...
}
```
<!--more-->

localCache的key是CacheKey对象
```java
public class CacheKey implements Cloneable, Serializable {
  ...
  private static final int DEFAULT_MULTIPLIER = 37;
  private static final int DEFAULT_HASHCODE = 17;
  //hash计算时乘的倍数,默认构造会赋值DEFAULT_MULTIPLIER
  private final int multiplier; 
  //hash码
  private int hashcode; 
  //不乘倍数的hash码累加值
  private long checksum;
  //调用update方法的次数
  private int count;
  //所有update对象
  private List<Object> updateList;
  ...
  // 每次update便修改hashcode，count，checksum，updateList
  public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

    count++;
    checksum += baseHashCode;
    baseHashCode *= count;

    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
  }
  ...
  // 比较hashcode，count，checksum，updateList
  @Override
  public boolean equals(Object object) {
    if (this == object) {
      return true;
    }
    if (!(object instanceof CacheKey)) {
      return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    if (hashcode != cacheKey.hashcode) {
      return false;
    }
    if (checksum != cacheKey.checksum) {
      return false;
    }
    if (count != cacheKey.count) {
      return false;
    }

    for (int i = 0; i < updateList.size(); i++) {
      Object thisObject = updateList.get(i);
      Object thatObject = cacheKey.updateList.get(i);
      if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
      }
    }
    return true;
  }

  @Override
  public int hashCode() {
    return hashcode;
  }
  ...
}
```

### 缓存的命中条件
```java
  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    //1.方法名
    cacheKey.update(ms.getId());
    //2.偏移量
    cacheKey.update(rowBounds.getOffset());
    //3.查询数量上限
    cacheKey.update(rowBounds.getLimit());
    //4.sql
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    // 解析参数
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        //5.参数
        cacheKey.update(value);
      }
    }
    //6.环境
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```
从上面代码可以看出一级缓存命中条件是
1. **相同的方法（MappedStatement）**
2. **相同sql，参数**
3. **相同查询范围，即偏移量，limit**
4. **相同的session**，此代码中未体现，但是localCache在BaseExecutor中，而BaseExecutor的创建是在初始化session时，调用栈信息如下图。

![avatar](/images/mybatis/1_cache_invoke.png)



### 缓存清除条件
PerpetualCache通过clear方法来清空缓存
```java
public class PerpetualCache implements Cache {
  ...
  @Override
  public void clear() {
    cache.clear();
  }
  ...
}
```
调用情况如下图
![avatar](/images/mybatis/first_cache_clear.png)
1. 回滚
2. 提交
3. 修改
4. 查询，当设置flushCache="true"时，默认为false
```java
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ...
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    ...
  }
```
**由于事务提交会清除缓存，所以要用到一级缓存，还需要在同一事务中**
