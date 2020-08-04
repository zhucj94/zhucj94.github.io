---
title: MyBatis源码之Executor
tags: MyBatis
declare: true
abbrlink: f8953bf7
date: 2020-08-02 18:01:17
---
### 前言
在阅读MyBatis源码前，建议先去github上clone MyBatis以及MyBatis parent源码，在本地搭建测试环境，以供idea在源码中打断点调试。

---
### Executor及其子类
![avatar](/images/mybatis/executor.png)
<!--more-->
![avatar](/images/mybatis/executor_model.png)


#### Executor
所有执行器的父类，定义了提交，回归，查询（DQL），修改(不只是update，是所有DML)，获取事务等方法。

#### BaseExecutor
BaseExecutor采用**模板方法模式**，实现大多数共用的功能，只将如下方法交给子类实现。一级缓存核心代码就位于BaseExecutor中（关于一级缓存，请看  https://zhucj94.github.io/article/7dd9b5ec.html ）
```java
  protected abstract int doUpdate(MappedStatement ms, Object parameter) throws SQLException;

  protected abstract List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException;

  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;

  protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
      throws SQLException;
```
以query为例
```java
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    ...
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }

  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ...
    List<E> list;
    try {
      queryStack++;
      // 从一级缓存获取
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //未获取到查询数据库
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
   ...
  }

  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    //设置缓存占位符，不明白含义
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      //交给子类  
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

#### SimpleExecutor
默认的Executor，Configuration.newExecutor
```java
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    // 未指定executorType时，默认simple
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
SimpleExecutor的逻辑很简单，获取StatementHandler，每次都创建Statement，将数据交给StatementHandler
```java
 ...
 @Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      //获取StatementHandler
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }

  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    // 创建Statement
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
  ...
```

#### ReuseExecutor
ReuseExecutor与Simple的不同在于，当sql语句相同时，，**ReuseExecutor会重用Statement**
```java
private final Map<String, Statement> statementMap = new HashMap<>();
...
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    BoundSql boundSql = handler.getBoundSql();
    String sql = boundSql.getSql();
    // 如果statementMap中有相同的Statement，就复用
    if (hasStatementFor(sql)) {
      stmt = getStatement(sql);
      applyTransactionTimeout(stmt);
    } else {
      // 否则就新建，并且置入map
      Connection connection = getConnection(statementLog);
      stmt = handler.prepare(connection, transaction.getTimeout());
      putStatement(sql, stmt);
    }
    handler.parameterize(stmt);
    return stmt;
  }

  private boolean hasStatementFor(String sql) {
    try {
      Statement statement = statementMap.get(sql);
      return statement != null && !statement.getConnection().isClosed();
    } catch (SQLException e) {
      return false;
    }
  }

  private Statement getStatement(String s) {
    return statementMap.get(s);
  }

  private void putStatement(String sql, Statement stmt) {
    statementMap.put(sql, stmt);
  }
...
```
#### BatchExecutor
BatchExecutor同样会重用Statment，且会批量提交
```java
  public static final int BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002;

  private final List<Statement> statementList = new ArrayList<>();
  private final List<BatchResult> batchResultList = new ArrayList<>();
  private String currentSql;
  private MappedStatement currentStatement;

  public BatchExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
  }

  @Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    // 如果sql与currentSql相同，MappedStatement与currentStatement相同，就获取batchResultList的最后一个元素，并将参数加入
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      int last = statementList.size() - 1;
      stmt = statementList.get(last);
      applyTransactionTimeout(stmt);
      handler.parameterize(stmt);// fix Issues 322
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      //如果不相同，就新建BatchResult和Statement，并将其add到list，且设置currentSql和currentStatement为当前值
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);    // fix Issues 322
      currentSql = sql;
      currentStatement = ms;
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    //通过上面两个操作，就形成了两个list下标一一对应的且部分复用的情况（间隔调用相同方法，不会复用Statement）。

    handler.batch(stmt);
    //统一返回-2147482646，此时还没有批量提交，无法得知操作行数，统一返回一个负数
    //所以当executor为batch时，insert，delete，update方法，不能使用返回值>0判断方法是否成功。
    return BATCH_UPDATE_RETURN_VALUE;
  }

  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
     ...
      for (int i = 0, n = statementList.size(); i < n; i++) {
        Statement stmt = statementList.get(i);
        applyTransactionTimeout(stmt);
        BatchResult batchResult = batchResultList.get(i);
        try {
          //循环提交  
          batchResult.setUpdateCounts(stmt.executeBatch());
     ...
  }

```

#### CachingExecutor
CachingExecutor使用装饰模式，增强上述的Executor，二级缓存的调用逻辑也位于CachingExecutor中，关于二级缓存请看（ https://zhucj94.github.io/article/3f89eca3.html ）
```java
// 被装饰的Executor
private final Executor delegate;

public CachingExecutor(Executor delegate) {
    this.delegate = delegate;
    delegate.setExecutorWrapper(this);
}

 @Override
  public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    flushCacheIfRequired(ms);
    // 将update逻辑叫给被装饰的Executor
    return delegate.update(ms, parameterObject);
  }
```


本文执行器图片转自
https://blog.csdn.net/landywu1985/article/details/106736621
