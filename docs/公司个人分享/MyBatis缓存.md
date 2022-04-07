# MyBatis源码阅读

# 一、事件

## 脏数据读取

学习地图在调用积分组件时，返回结果不正确，读取到脏数据；

也就是更新了数据库，但读取到的还是之前的数据；

# 二、过程

## 1.自测接口

相同参数调用接口返回无误、远程 debug时可以查到数据；

## 2.测试事务

```properties
// 打印事务日志
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=debug
```

```java
// 获取当前事务名
TransactionSynchronizationManager.getCurrentTransactionName();
// 获取当前线程ID
Thread.currentThread().getId();
```



A方法包含方法X和方法Y

> 情况一：A带事务1，方法X带事务2，方法Y不带事务
>
> 方法2查询异常；

疑惑：打印日志后，发现是在同一个线程内，并且Y与A处于同一事务，为何无法查询出事务2已提交的操作？

> 情况二：A带事务，方法X带事务，方法Y带事务
>
> 可以查询出数据；

原因：同一个线程内，在rc隔离级别下，不同的事务管理器，可以读取已提交的数据；

> 情况三：A带事务，方法X不带事务，方法Y不带事务
>
> 可以查询出数据；

原因：同一线程内，同一事务内，可以读取当前事务进行的数据库操作；

> 情况四：A不带事务，方法X带事务，方法Y不带事务
>
> 可以查询出数据；

原因：调用方不带事务时，X的事务提交后不会影响查询；

> 情况五：A不带事务，方法X带事务，方法Y带事务
>
> 可以查询出数据；

原因：调用方不带事务时，方法上的事务只要已提交，则互不影响；

> 情况六：A不带事务，方法X不带事务，方法Y不带事务
>
> 无事务；

原因：无事务，不影响任何操作；



## 3.仔细查看调用方调用逻辑

在一个事务1内，先调用一次方法Y，再调用方法X，最后还会调用一次方法Y

# 三、结论

## 1.前提

同一线程内的两个事务，事务B嵌套在事务A中，数据库隔离级别：读提交rc

## 2.小知识

>一级缓存是基于 PerpetualCache（MyBatis自带）的 HashMap 本地缓存，作用范围为 session 域内。当 session flush（刷新）或者 close（关闭）之后，该 session 中所有的 cache（缓存）就会被清空。
>
>在参数和 SQL 完全一样的情况下，我们使用同一个 SqlSession 对象调用同一个 mapper 的方法，往往只执行一次 SQL。因为使用 SqlSession 第一次查询后，MyBatis 会将其放在缓存中，再次查询时，如果没有刷新，并且缓存没有超时的情况下，SqlSession 会取出当前缓存的数据，而不会再次发送 SQL 到数据库。
>
>由于 SqlSession 是相互隔离的，所以如果你使用不同的 SqlSession 对象，即使调用相同的 Mapper、参数和方法，MyBatis 还是会再次发送 SQL 到数据库执行，返回结果。

## 3.缓存时序图

图找不到了orz

## 4.测试代码

```java
    @Test
    public void getPerson() {
        System.out.println("getPerson本地缓存范围: " + factory.getConfiguration().getLocalCacheScope());
        SqlSession sqlSession =  factory.openSession(true);

        Demo1Mapper demo1Mapper = sqlSession.getMapper(Demo1Mapper.class);
        System.out.println("Mapper1在同一会话中查询：" + demo1Mapper.selectByPrimaryKey(1L));
        System.out.println("Mapper1在同一会话中查询：" + demo1Mapper.selectByPrimaryKey(1L));
        System.out.println("Mapper1在同一会话中查询：" + demo1Mapper.selectByPrimaryKey(1L));

        sqlSession.close();
    }

    @Test
    public void getPersonBySingleSession() {
        System.out.println("开始在单个会话下更新后获取用户信息：");

        SqlSession sqlSession =  factory.openSession(true);
        Demo1Mapper demo1Mapper2 = sqlSession.getMapper(Demo1Mapper.class);

        count++;
        System.out.println("Mapper2在同一会话中查询：" + demo1Mapper2.selectByPrimaryKey(1L));
        Demo1 demoPO = demo1Mapper2.selectByPrimaryKey(1L);
        demoPO.setAge(count);
        demo1Mapper2.updateByPrimaryKeySelective(demoPO);
        System.out.println("将更新age字段为：" + count);
        System.out.println("Mapper2在同一会话中更新后查询：" + demo1Mapper2.selectByPrimaryKey(1L));

        sqlSession.close();
    }

    @Test
    public void getPersonByMultipleSession() {
        System.out.println("开始在多个会话下同时获取用户信息");

        SqlSession sqlSession1 =  factory.openSession(true);
        SqlSession sqlSession2 =  factory.openSession(true);
        Demo1Mapper demo1Mapper3 = sqlSession1.getMapper(Demo1Mapper.class);
        Demo1Mapper demo1Mapper4 = sqlSession2.getMapper(Demo1Mapper.class);
        count++;
        System.out.println("Mapper3在多会话中查询：" + demo1Mapper3.selectByPrimaryKey(1L));
        System.out.println("Mapper4在多会话中查询：" +demo1Mapper4.selectByPrimaryKey(1L));
        Demo1 demoPO = demo1Mapper3.selectByPrimaryKey(1L);
        demoPO.setAge(count);
        demo1Mapper3.updateByPrimaryKeySelective(demoPO);
        System.out.println("将更新age字段为：" + count);
        System.out.println("Mapper3在多会话中更新后查询：" + demo1Mapper3.selectByPrimaryKey(1L));
        System.out.println("Mapper4在Mapper3更新后查询：" +demo1Mapper4.selectByPrimaryKey(1L));

        sqlSession1.close();
        sqlSession2.close();
    }
```



## 5.MyBatis 缓存实现流程

​		前面提到过，MyBatis的一级缓存存在于SqlSession的生命周期中，在同一个sqlSession中查询时，MyBatis会把执行的方法和参数通过算法生成缓存的键值，将键值和查询结果一起放入Map对象中。 **如果同一个SqlSession中执行的方法和参数完全一致**，那么通过算法会生成相同的键值。 当Map缓存对象中已经存在该键值时，则会返回缓存中的对象。所以反复使用相同参数执行同一个方法时，总是返回同一个对象。

## 6.部分源码

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 若无缓存则查询数据库
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    // 若查询堆栈为0
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        // 如果本地缓存范围为statement的话，就会清空缓存
        clearLocalCache();
      }
    }
    return list;
  }
```

## 7.总结

当我们有一个线程A来执行此方法时，发现此方法开启了事务，**而事务，又是基于数据库Connection连接的**，所以事务B进行前若是已进行过查询数据库Y操作，会导致在事务B提交后（数据库X操作），再进行相同Y操作时，读取的数据是处于事务A下mybatis 一级缓存中的结果。

So，在写HSF服务时，需要注意是否会有同项目的模块进行调用，并且加上了事务进行了多次不同业务端调用；

同理，在调用时也需要注意是否调用的RPC服务，实现在本项目内；

