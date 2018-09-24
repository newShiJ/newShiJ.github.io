---
layout: post
title:  "利用Mybatis拦截器实现分页查询"
date:   2018-09-24
categories: jekyll update
---

# 手写Mybatis拦截器
### 版本 Spring Boot 2.0.3.RELEASE


## Mybatis自定义拦截器

如果有阅读过我之前一篇博客 [Hibernate 刷新上下文](https://blog.csdn.net/cmmchenmm/article/details/82802084) 的朋友应该还记得 Hibernate 的上下文中可以添加自定义的事件监听器。当初是为了解决一个类似于二段提交的的问题，后面我利用 Hibernate 自带的上下文事件监听器算是比较优雅的处理了。所以当时就想看看 Mybatis 这边有没有什么类似的方式处理，于是就有了这篇文章。

我看可以来先看看[Mybatis 官网](http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins)上对拦截器的介绍。Mybatis 官网对拦截器称呼为`插件（plugins）`官网的介绍也比较简单，关键就是一个小 demo 如下

```java
// ExamplePlugin.java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  public Object intercept(Invocation invocation) throws Throwable {
    return invocation.proceed();
  }
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }
  // 可以通过 Properties 获取到你想要的一些配置信息
  public void setProperties(Properties properties) {
  }
}
```
```xml
<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="org.mybatis.example.ExamplePlugin">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```

### Spring Boot 自定义 Mybatis 拦截器
我们可以根据官网上的介绍来自己写一个简单的 Mybatis 拦截器，我写的简易代码如下。在拦截上直接声明@Component即可注册

```java
@Intercepts({
        @Signature(
                type = Executor.class,
                method = "update", args = {MappedStatement.class, Object.class}
        )
})
@Component
public class MyIntertceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("进入拦截器");
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```


可以看出核心就是实现 `public Object intercept(Invocation invocation) throws Throwable`这么一个方法。效果就当执行 update 相关操作（insert ，update 语句）时会触发执行，打印出`进入拦截器`。

我们可以来看下[Mybatis 官网](http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins)的介绍


>MyBatis 允许你在已映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

>Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)

我这里是使用了 Executor 执行时处理的拦截器，有对应着上面几种情况时的处理。

## 实现简易的分页查询

### 设计思路
* 调用形式
* 数据库方言
* 拦截器逻辑

#### 调用方法
有使用过Mybatis分页插件 PageHelper的应该都知道是先调用一个静态方法，对下条sql语句进行拦截，在new 一个分页对象时自动处理。

在PageHelper中是利用了ThreadLocal 本地线程变量副本来处理的，当执行那个方法时往ThreadLocal设置一个分页参数值，所以它每次只对下一条SQL语句有效。所以这里我也准备这么做。在new 分页对象时remove掉ThreadLocal中的变量值 代码如下

```java
public class PageResult <T>{
    private long total;

    private List<T> data;

    public PageResult(List<T> data) {
        this.data = data;
        PageInterceptor.PageParm pageParm = PageInterceptor.PARM_THREAD_LOCAL.get();
        if(pageParm != null){
            total = pageParm.totalSize;
            PageInterceptor.PARM_THREAD_LOCAL.remove();
        }
    }

    public long getTotal() {
        return total;
    }

    public List<T> getData() {
        return data;
    }
}

```

```java
@Intercepts({
        @Signature(
                type = Executor.class,method = "query",
                args = {MappedStatement.class,Object.class,RowBounds.class,ResultHandler.class}
        )
})
@Component
public class PageInterceptor implements Interceptor {
	...
	
	static final ThreadLocal<PageParm> PARM_THREAD_LOCAL = new ThreadLocal<>();

    static class PageParm{
        // 分页开始位置
        int offset;
        // 分页数量
        int limit;
        // 总数
        long totalSize;
    }

    /**
     * 开始分页
     * @param pageNum 当前页码 从0开始
     * @param pageSize 每页长度
     */
    public static void startPage(int pageNum,int pageSize){
        int offset = pageNum * pageSize;
        int limit = pageSize;
        PageParm pageParm = new PageParm();
        pageParm.offset = offset;
        pageParm.limit = limit;
        PARM_THREAD_LOCAL.set(pageParm);
    }
}
```

#### 数据库方言问题 构建分页SQL

我这里用了一个策略模式，定义好一个方言接口，不同的数据使用不同的方言实现，在注入时生声明，目前我只有一个MySQL所以也不算完全的策略模式。一个分页是需要两条语句的，一个是count 一个是 limit。

```java
public interface Dialect {

    /**
     * 获取countSQL语句
     * @param targetSql
     * @return
     */
    default String getCountSql(String targetSql){
        return String.format("select count(1) from (%s) tmp_count",targetSql);
    }

    String getLimitSql(String targetSql, int offset, int limit);
}
```

```java
@Component //我这里直接指定了，当然最好是使用 @bean 这样把它new出来更好一些
public class MysqlDialect implements Dialect {

    private static final String PATTERN = "%s limit %s, %s";

    private static final String PATTERN_FIRST = "%s limit %s";

    @Override
    public String getLimitSql(String targetSql, int offset, int limit) {
        if (offset == 0) {
            return String.format(PATTERN_FIRST, targetSql, limit);
        }

        return String.format(PATTERN, targetSql, offset, limit);
    }
}
```

### 拦截器核心逻辑

在贴出代码之前，我想先感谢一下 [buzheng](https://github.com/buzheng/mybatis-pageable)同学，因为这里面的拦截器核心逻辑有很大一部分就是参考他写的Mybatis分页中拦截器的实现。

```java
@Override
public Object intercept(Invocation invocation) throws Throwable {
    final Object[] args = invocation.getArgs();
    PageParm pageParm = PARM_THREAD_LOCAL.get();
    //判断是否需要进分页
    if(pageParm != null){
        final MappedStatement ms = (MappedStatement)args[MAPPED_STATEMENT_INDEX];
        Object param = args[PARAMETER_INDEX];
        BoundSql boundSql = ms.getBoundSql(param);
        // 获取总数
        pageParm.totalSize = queryTotal(ms,boundSql);
        // 重新设置SQL语句映射
        args[MAPPED_STATEMENT_INDEX] = copyPageableMappedStatement(ms,boundSql);
    }
    Object proceed = invocation.proceed();
    return proceed;
}
```
获取数据的总数量 -> count

```java
/**
 * 查询总记录数 基本上属于直接抄的
 * @param mappedStatement
 * @param boundSql
 * @return
 * @throws SQLException
 */
private long queryTotal(MappedStatement mappedStatement, BoundSql boundSql) throws SQLException {

    Connection connection = null;
    PreparedStatement countStmt = null;
    ResultSet rs = null;
    try {

        connection = mappedStatement.getConfiguration().getEnvironment().getDataSource().getConnection();

        String countSql = this.dialect.getCountSql(boundSql.getSql());

        countStmt = connection.prepareStatement(countSql);
        BoundSql countBoundSql = new BoundSql(mappedStatement.getConfiguration(), countSql,
                boundSql.getParameterMappings(), boundSql.getParameterObject());

        setParameters(countStmt, mappedStatement, countBoundSql, boundSql.getParameterObject());

        rs = countStmt.executeQuery();
        long totalCount = 0;
        if (rs.next()) {
            totalCount = rs.getLong(1);
        }

        return totalCount;
    } catch (SQLException e) {
        logger.error("查询总记录数出错", e);
        throw e;
    } finally {
        if (rs != null) {
            try {
                rs.close();
            } catch (SQLException e) {
                logger.error("exception happens when doing: ResultSet.close()", e);
            }
        }

        if (countStmt != null) {
            try {
                countStmt.close();
            } catch (SQLException e) {
                logger.error("exception happens when doing: PreparedStatement.close()", e);
            }
        }

        if (connection != null) {
            try {
                connection.close();
            } catch (SQLException e) {
                logger.error("exception happens when doing: Connection.close()", e);
            }
        }
    }
}
/**
 * 对SQL参数(?)设值
 *
 * @param ps
 * @param mappedStatement
 * @param boundSql
 * @param parameterObject
 * @throws SQLException
 */
private void setParameters(PreparedStatement ps, MappedStatement mappedStatement, BoundSql boundSql,
                           Object parameterObject) throws SQLException {
    ParameterHandler parameterHandler = new DefaultParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler.setParameters(ps);
}
```

利用方言接口替换原始的SQL语句

```java
private MappedStatement copyPageableMappedStatement(MappedStatement ms, BoundSql boundSql) {
    PageParm pageParm = PARM_THREAD_LOCAL.get();
    String pageSql = dialect.getLimitSql(boundSql.getSql(),pageParm.offset,pageParm.limit);
    SqlSource source = new StaticSqlSource(ms.getConfiguration(),pageSql,boundSql.getParameterMappings());
    return copyFromMappedStatement(ms,source);
}

/**
 * 利用新生成的SQL语句去替换原来的MappedStatement
 * @param ms
 * @param newSqlSource
 * @return
 */
private MappedStatement copyFromMappedStatement(MappedStatement ms,SqlSource newSqlSource) {
    MappedStatement.Builder builder = new MappedStatement.Builder(ms.getConfiguration(),ms.getId(),newSqlSource,ms.getSqlCommandType());

    builder.resource(ms.getResource());
    builder.fetchSize(ms.getFetchSize());
    builder.statementType(ms.getStatementType());
    builder.keyGenerator(ms.getKeyGenerator());
    if(ms.getKeyProperties() != null && ms.getKeyProperties().length !=0){
        StringBuffer keyProperties = new StringBuffer();
        for(String keyProperty : ms.getKeyProperties()){
            keyProperties.append(keyProperty).append(",");
        }
        keyProperties.delete(keyProperties.length()-1, keyProperties.length());
        builder.keyProperty(keyProperties.toString());
    }

    //setStatementTimeout()
    builder.timeout(ms.getTimeout());

    //setStatementResultMap()
    builder.parameterMap(ms.getParameterMap());

    //setStatementResultMap()
    builder.resultMaps(ms.getResultMaps());
    builder.resultSetType(ms.getResultSetType());

    //setStatementCache()
    builder.cache(ms.getCache());
    builder.flushCacheRequired(ms.isFlushCacheRequired());
    builder.useCache(ms.isUseCache());

    return builder.build();
}
```

这样在执行了分页查询的时候，会额外执行一条count语句，并且把原来的SQL换成带有limit的语句最终查询的结果就如下

```java
@GetMapping("/all")
public Object all(){
    PageInterceptor.startPage(1,2);
    List<Model> all = dao.findAll();
    PageResult<Model> modelPageResult = new PageResult<>(all);
    return modelPageResult;
}

```

```json
{  
   total:3,
   data:-   [  
      -      {  
         id:"2",
         name:null,
         code:"123"
      }
   ]
}
```
我的代码已经放在了[github](https://github.com/newShiJ/Mybatis-Pageable)上欢迎大家随时star

github 地址`https://github.com/newShiJ/Mybatis-Pageable`
