# 什么是mybatis
JDBC数据库缺点：
- 没有池化技术
- sql放在代码中



# 技术本质

- Configuration MyBatis所有的配置信息都保存在Configuration对象之中，配置文件中的大部分配置都会存储到该类中
- SqlSession 作为MyBatis工作的主要顶层API，表示和数据库交互时的会话，完成必要数据库增删改查功能
- Executor MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护
- StatementHandler 封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数等
- ParameterHandler 负责对用户传递的参数转换成JDBC Statement 所对应的数据类型
- ResultSetHandler 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合
- TypeHandler 负责java数据类型和jdbc数据类型(也可以说是数据表列类型)之间的映射和转换
- MappedStatement MappedStatement维护一条<select|update|delete|insert>节点的封装
- SqlSource 负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回
- BoundSql 表示动态生成的SQL语句以及相应的参数信息

## Configuration
Configuration对象存储mybatis的所有配置相关对象。

## 获取数据库源
1、解析mybatis-conf.xml获取env对象，里面包含dataSource相关信息
2、通过DataSourceFactory构造DataSource
3、将Env对象设置到configuration中

## 获取sql
通过mybatis-conf.xml里面的mapper标签加载所有mapper对象
mapper加载mappers文件4种方式：
- __resource__: 直接解析mapper.xml文件，获取namespace，resultMap，以及xml里面的所有sql节点。遍历解析所有sql节点，转换成mapperStatement对象，放入configuration中。最后通过namespace反射获取mapper对应的class对象，通过动态代理生产mapper的对象，并且放入configuration中。

- __url__: 与resource一致

- __class__: 与resource相反，先通过反射获取mapper接口的class对象以及生成动态代理，并添加到configuration中。根据class名称+.xml获取对应的配置文件，并解析resultMap以及相关sql，生产mappedStatement对象，存放到configuration中。

- __package__(优先级最高): 多文件映射，通过ResolverUtil工具类，找出该包路径下所有mapper接口名称，并通过反射获取class对象以及生成动态代理类，装到configuration中。遍历所有mepper对象，获取对应的xml文件并解析生成mappedStatement对象，放入configuration中。

# Executor
``` java
public enum ExecutorType {
  SIMPLE, REUSE, BATCH
}
```
- SimpleExecutor: 简单执行器，是 MyBatis 中默认使用的执行器，每执行一次 update 或 select，就开启一个 Statement 对象，用完就直接关闭 Statement 对象(可以是 Statement 或者是 PreparedStatment 对象)

- ReuseExecutor: 可重用执行器，这里的重用指的是重复使用 Statement，它会在内部使用一个 Map 把创建的 Statement 都缓存起来，每次执行 SQL 命令的时候，都会去判断是否存在基于该 SQL 的 Statement 对象，如果存在 Statement 对象并且对应的connection 还没有关闭的情况下就继续使用之前的 Statement 对象，并将其缓存起来。因为每一个 SqlSession都有一个新的Executor对象，所以我们缓存在ReuseExecutor上的Statement作用域是同一个SqlSession。

- BatchExecutor: 批处理执行器，用于将多个SQL一次性输出到数据库

## 执行sql
1、构造sqlSessionFactory，期间会解析相关配置文件生成configuration对象。

2、通过SqlSessionFactory获取session对象，session中维护了Executor对象，默认是simpleExecutor，executor内置localcache，默认开启一级缓存。

3、通过session调用select|update方法，所有的select都将调用selectList, 所有的增删改都会调用update方法。

4、session调用需要传递mapper的方法名称，通过类.方法名称从configuration中获取对应的mappedStatement对象。并从mappedStatement中获取boundSql对象，该对象存放了真正的sql，并将sql中的占位符替换对应的参数。

5、先查询CacheExecutor二级缓存, 没有的话则通过内置的executor（委托者模式）执行。

6、doQuery中会通过configuraton构造会话处理器StatementHandler，也就是封装了jdbc的相关操作，注册驱动，获取链接，创建会话对象。接着通过prepareStatement方法对进行预编译，防止sql注入。通过paramHandler处理参数类型转换，获取结果集后通过resultSetHandle处理类型转换并生成对应的对象。

# 防止sql注入
PreparedStatement通过预编译，先将#{}替换成？，并对参数转换成字符串增加双引号，通过set方法赋值，从而避免sql注入。而${}则是字符串替换，会引起sql注入问题。

# 一级、二级缓存 
1）一级缓存: 基于 PerpetualCache 的 HashMap 本地缓存，其存储作用域为 Session，当 Session flush 或 close 之后，该 Session 中的所有 Cache 就将清空。 
2）二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap 存储，不同在于其存储作用域为 Mapper(Namespace)，并且可自定义存储源，如 Ehcache。要开启二级缓存，你需要在你的 SQL 映射文件中添加一行
3）对于缓存数据更新机制，当某一个作用域(一级缓存 Session/二级缓存Namespaces)的进行了C/U/D 操作后，默认该作用域下所有 select 中的缓存将被 clear。

# 延迟加载
```sql
SELECT orders.*, user.username FROM orders, USER WHERE orders.user_id = user.id
```
Mybatis仅支持association关联对象和collection关联集合对象的延迟加载，association指的就是一对一，collection指的就是一对多查询。在Mybatis配置文件中，可以配置是否启用延迟加载lazyLoadingEnabled=true|false。 它的原理是，使用CGLIB创建目标对象的代理对象，当调用目标方法时，进入拦截器方法，比如调用a.getB().getName()，拦截器invoke()方法发现a.getB()是null值，那么就会单独发送事先保存好的查询关联B对象的sql，把B查询上来，然后调用a.setB(b)，于是a的对象b属性就有值了，接着完成a.getB().getName()方法的调用。这就是延迟加载的基本原理。