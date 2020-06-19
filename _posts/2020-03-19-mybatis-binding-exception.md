---
layout: post
title:  "mybatis执行批量插入返回主键时异常原因及源码分析"
date:   2020-03-19 10:00:00 +0800
categories: 后端开发
---


mybatis-3.2.7执行批量插入语句并返回主键时代码报错，经过确认是代码的bug，升级版本到3.3.1后正常。  
下面通过源码分析一下批量插入时返回主键的原理，以及造成这个bug的原因。

错误信息如下：

```java
Caused by: org.apache.ibatis.binding.BindingException: Parameter 'id' not found. Available parameters are [list]
```

下面从头梳理一下这个问题，一般处理批量插入的写法是：

```java

// Service层代码
public class UserService{
    List<User> insertBatch(){
        List<User> userList = new ArrayList();
        // ... 中间省略
        // 批量插入，并且可以获取到插入后的主键
        userDao.insertBatch(userList);
    }
}

// DAO层
public  interface UserDao{
    void insertBatch(List<User> userList);
}

```

```xml
// xml片段
<insert id="insertBatch" parameterType="java.util.List" useGeneratedKeys="true" keyProperty="id"  keyColumn="id">
    insert into User(id, user_name, user_pass, email, address)
    values
    <foreach collection="list" item="item" index="index" separator=",">
        (
            #{item.id}, #{item.userName}, #{item.userPass}, #{item.email}, #{item.address}
        )
    </foreach>
</insert>
```

#### mybatis批量插入并返回主键的理论依赖

JDBC的`java.sql.Statement`接口提供了方法

`ResultSet getGeneratedKeys() throws SQLException;` 这个方法返回可以返回主键

只要在SQL语句执行完成后往实体类填充主键即可实现返回主键的目的

#### 具体代码实现
先简单说一下mybatis的代码结构  
1. SqlSession负责处理最外层的增删改查逻辑，可以直接返回查询的实体类  
2. Executor及其实现类SimpleExecutor, ReuseExecutor, BatchExecutor等进一步包装SqlSession传过来的SQL语句，并且负责处理对象关系映射，这个mybatis作为一个ORM(Object Relation Mapping)的核心  
3. 进一步Executor将处理逻辑递交给StatementHandler，这一层主要负责执行JDBC提供的java.sql.Statement或者java.sql.PreparedStatement语句，并从java.sql.ResultSet中得到SQL执行的结果

完成填充主键这一步就是在StatementHandler这一层，具体是在`org.apache.ibatis.executor.statement.PreparedStatementHandler#update`,代码如下：

```java
  @Override
  public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行sql
    ps.execute();
    // 获取受影响的行数
    int rows = ps.getUpdateCount();
    // 获取参数对象，也就是我们从业务层传进来的userList，后面将会填充其中的主键字段
    Object parameterObject = boundSql.getParameterObject();
    // 这里用到的是Jdbc3KeyGenerator
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    // 进一步调用processAfter方法
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
  }
```

`org.apache.ibatis.executor.keygen.Jdbc3KeyGenerator#processAfter`的主要代码如下  
```java
  @Override
  public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
      // 注意这里的getParameters方法，后面后说明
    processBatch(ms, stmt, getParameters(parameter));
  }

  // 代码摘录，省略部分非关键代码
  public void processBatch(MappedStatement ms, Statement stmt, Collection<Object> parameters) {
    ResultSet rs = null;
    // stmt是就是上面的preparedStatement，通过getGeneratedKeys方法可以获取到主键
    // rs中的数据是一行一行的，每行中可能会有多个主键列
    rs = stmt.getGeneratedKeys();
    final Configuration configuration = ms.getConfiguration();
    final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    final String[] keyProperties = ms.getKeyProperties();
    final ResultSetMetaData rsmd = rs.getMetaData();
    TypeHandler<?>[] typeHandlers = null;
    if (keyProperties != null && rsmd.getColumnCount() >= keyProperties.length) {
    // 这里的parameters代表了userList对象，循环其中的一个即单个user对象，为其设置主键ID
    for (Object parameter : parameters) {
        // 跳到结果集的下一行，如果到了最后一行则循环结束
        if (!rs.next()) {
            break;
        }
        // 参数对象被包装成MetaObject, 之后通过反射修改字段值
        final MetaObject metaParam = configuration.newMetaObject(parameter);
        if (typeHandlers == null) {
            typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
        }
        // 填充主键
        populateKeys(rs, metaParam, keyProperties, typeHandlers);
    }
    }    
  }

```

mybatis-3.2.7与mybatis-3.3.1的代码不同之处就在于
`processBatch(ms, stmt, getParameters(parameter));`
中对入参多了一步处理，调用了`getParameters`方法，代码如下

该方法将参数从Map中抽取出来，之所以这么做是因为一开始mybatis将参数对象放在了map中进行了处理，所以这里要从Map中再提取出来。  

而mybatis-3.2.7的代码没有这一步，导致了后面去设置主键ID的时候，找不到这个属性，因此直接报错。

```java
  private Collection<Object> getParameters(Object parameter) {
    Collection<Object> parameters = null;
    if (parameter instanceof Collection) {
      parameters = (Collection) parameter;
    } else if (parameter instanceof Map) {
      Map parameterMap = (Map) parameter;
      if (parameterMap.containsKey("collection")) {
        parameters = (Collection) parameterMap.get("collection");
      } else if (parameterMap.containsKey("list")) {
        parameters = (List) parameterMap.get("list");
      } else if (parameterMap.containsKey("array")) {
        parameters = Arrays.asList((Object[]) parameterMap.get("array"));
      }
    }
    if (parameters == null) {
      parameters = new ArrayList<Object>();
      parameters.add(parameter);
    }
    return parameters;
  }
```

