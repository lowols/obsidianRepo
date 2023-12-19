# Mybatis

---

# 第⼀部分：⾃定义持久层框架

## 1.1 分析JDBC操作问题

JDBC（Java DataBase Connectivity），即Java数据库连接。简而言之，就是通过Java语言来操作数据库。

我们可以把JDBC理解成是官方定义的一套操作所有关系型数据库的规则，规则即接口。

也就是说，官方定义了一套操作所有关系型数据库的接口，然后让各个数据厂商（Mysql、Oracle等）用实现类去实现这套接口，再把这些实现类打包（数据驱动jar包），并提供数据驱动jar包给我们使用。

我们可以使用这套JDBC接口进行编程，但是真正执行的代码是驱动jar包中的实现类。

为什么？因为JDBC是通过接口来调用方法的，当你导入了驱动jar包（实现类）后，那调用的方法肯定是实现类里面的方法。

> JDBC和各个数据库驱动之间的关系，好比JVM规范和具体的java虚拟机（hotspot）之间的关系。

```java
    public static void main(String[] args) {
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet resultSet = null;
        try {
            //1、导入驱动jar包
            //2、加载数据库驱动
            Class.forName("com.mysql.jdbc.Driver");
            //3、通过驱动管理类获取数据库链接
            connection =
                    DriverManager.getConnection("jdbc:mysql://localhost:3306/mybatis?characterEncoding=utf-8", "root", "root");
            //4、定义sql语句，？表示占位符
            String sql = "select * from user where username = ?";
            //5、获取预处理statement，避免直接拼接参数，导致sql注入问题
            preparedStatement = connection.prepareStatement(sql);
            //6、设置参数，第⼀个参数为sql语句中参数的序号(从1开始)，第⼆个参数为设置的参数值
            preparedStatement.setString(1, "tom");
            //7、向数据库发出sql执⾏查询，查询出结果集
            resultSet = preparedStatement.executeQuery();
            //8、遍历查询结果集
            while (resultSet.next()) {
                int id = resultSet.getInt("id");
                String username = resultSet.getString("username");
                // 封装User
                user.setId(id);
                user.setUsername(username);
            }
            System.out.println(user);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 释放资源，略
        }
    }
```

### JDBC问题总结：

- 数据库连接创建、释放频繁造成系统资源浪费，从⽽影响系统性能。

- Sql语句在代码中硬编码，造成代码不易维护，实际应⽤中sql变化的可能较⼤，sql变动需要改变java代码。

- 使⽤preparedStatement向占有位符号传参数存在硬编码，因为sql语句的where条件不⼀定，可能多也可能少，修改sql还要修改代码，系统不易维护。

- 对结果集解析存在硬编码(查询列名)，sql变化导致解析代码变化，系统不易维护，如果能将数据库记录封装成pojo对象解析⽐较⽅便

## 1.2 问题解决思路

①使⽤数据库连接池初始化连接资源

②将sql语句抽取到xml配置⽂件中

③使⽤反射、内省等底层技术，⾃动将实体与表进⾏属性与字段的⾃动映射

> 所有的框架都是向着一个目标前进：开闭原则。所有结果集解析转换实体的硬编码代码，都有一个共同的处理逻辑，根据结果集的列名转换实体字段名。这时，有一个框架利用反射技术把这一过程自动化，可以为我们节省大量的代码。

## 1.3 ⾃定义框架设计

### 使⽤端：

提供核⼼配置⽂件：

sqlMapConfig.xml : 存放数据源信息，引⼊mapper.xml

Mapper.xml : sql语句的配置⽂件信息

### 框架端：

1. 读取配置⽂件
   读取完成以后以流的形式存在，我们不能将读取到的配置信息以流的形式存放在内存中，不好操作，可以创建javaBean来存储
   
   1. （1）Configuration : 存放数据库基本信息、Map<唯⼀标识，Mapper> 唯⼀标识：namespace + "."+id
   
   2. （2）MappedStatement：sql语句、statement类型、输⼊参数java类型、输出参数java类型

2. 解析配置文件
   创建sqlSessionFactoryBuilder类：
   
   ⽅法：sqlSessionFactory build()：
   
   第⼀：使⽤dom4j解析配置⽂件，将解析出来的内容封装到Configuration和MappedStatement中
   
   第⼆：创建SqlSessionFactory的实现类DefaultSqlSession

3. 创建SqlSessionFactory：
   
   ⽅法：openSession() : 获取sqlSession接⼝的实现类实例对象

4. 创建sqlSession接⼝及实现类：主要封装crud⽅法
   
   ⽅法：selectList(String statementId,Object param)：查询所有
   
   selectOne(String statementId,Object param)：查询单个
   
   具体实现：封装JDBC完成对数据库表的查询操作

#### 涉及到的设计模式：

Builder构建者设计模式、⼯⼚模式、代理模式

## 1.4 ⾃定义框架实现

在使⽤端项⽬中创建配置配置⽂件

创建 sqlMapConfig.xml

```xml
<configuration>
    <!--数据库连接信息-->
    <property name="driverClass" value="com.mysql.jdbc.Driver"></property>
    <property name="jdbcUrl" value="jdbc:mysql:///zdy_mybatis"></property>
    <property name="user" value="root"></property>
    <property name="password" value="root"></property>
    <!--引⼊sql配置信息-->
    <mapper resource="mapper.xml"></mapper>
</configuration>
```

mapper.xml

```xml
<mapper namespace="User">
    <select id="selectOne" paramterType="com.lagou.pojo.User"
            resultType="com.lagou.pojo.User">
        select * from user where id = #{id} and username =#{username}
    </select>

    <select id="selectList" resultType="com.lagou.pojo.User">
        select * from user
    </select>
</mapper>
```

User实体

```java
public class User {
    //主键标识
    private Integer id;
    //用户名
    private String username;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' + '}';
    }
}
```

再创建⼀个Maven⼦⼯程并且导⼊需要⽤到的依赖坐标

```xml
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
        <java.version>1.8</java.version>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.17</version>
        </dependency>

        <dependency>
            <groupId>c3p0</groupId>
            <artifactId>c3p0</artifactId>
            <version>0.9.1.2</version>
        </dependency>

        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.12</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>

        <dependency>
            <groupId>dom4j</groupId>
            <artifactId>dom4j</artifactId>
            <version>1.6.1</version>
        </dependency>

        <dependency>
            <groupId>jaxen</groupId>
            <artifactId>jaxen</artifactId>
            <version>1.1.6</version>
        </dependency>
    </dependencies>
```

Configuration

```java
public class Configuration {
    //数据源
    private DataSource dataSource;
    //map集合：    key:statementId value:MappedStatement
    private Map<String,MappedStatement> mappedStatementMap = new HashMap<String, MappedStatement>();
}
//setter,getter略
```

```java
public class MappedStatement {
    //id
    private Integer id;
    //sql语句
    private String sql;
    //输入参数
    private Class<?> paramterType;
    //输出参数
    private Class<?> resultType;
}
//setter,getter略
```

Resources

```java
public class Resources {
    public static InputStream getResourceAsSteam(String path){ InputStream resourceAsStream =
            Resources.class.getClassLoader.getResourceAsStream(path);
        return resourceAsStream;
    }
}
```

SqlSessionFactoryBuilder

```java
public class SqlSessionFactoryBuilder {
    private Configuration configuration;

    public SqlSessionFactoryBuilder() {
        this.configuration = new Configuration();
    }

    public SqlSessionFactory build(InputStream inputStream) throws DocumentException, PropertyVetoException, ClassNotFoundException {
        //1.解析配置文件，封装Configuration XMLConfigerBuilder
        XmlConfigerBuilder = new
                XMLConfigerBuilder(configuration);
        Configuration configuration =
                xmlConfigerBuilder.parseConfiguration(inputStream);
        //2.创建  sqlSessionFactory
        SqlSessionFactory sqlSessionFactory = new
                DefaultSqlSessionFactory(configuration);
        return sqlSessionFactory;
    }
}
```

XMLConfigerBuilder

```java
public class XMLConfigerBuilder {
    private Configuration configuration;

    public XMLConfigerBuilder(Configuration configuration) {
        this.configuration = new Configuration();
    }

    public Configuration parseConfiguration(InputStream inputStream) throws
            DocumentException, PropertyVetoException, ClassNotFoundException {
        Document document = new SAXReader().read(inputStream);
        //<configuation>
        Element rootElement = document.getRootElement();
        List<Element> propertyElements =
                rootElement.selectNodes("//property");
        Properties properties = new Properties();
        for (Element propertyElement : propertyElements) {
            String name = propertyElement.attributeValue("name");
            String value = propertyElement.attributeValue("value");
            properties.setProperty(name, value);
        }

        //连接池
        ComboPooledDataSource comboPooledDataSource = new
                ComboPooledDataSource();

        comboPooledDataSource.setDriverClass(properties.getProperty("driverClass"));
        comboPooledDataSource.setJdbcUrl(properties.getProperty("jdbcUrl"));
        comboPooledDataSource.setUser(properties.getProperty("username"));
        comboPooledDataSource.setPassword(properties.getProperty("password"));

        //填充  configuration
        configuration.setDataSource(comboPooledDataSource);
        //mapper 部分
        List<Element> mapperElements = rootElement.selectNodes("//mapper");
        XMLMapperBuilder xmlMapperBuilder = new
                XMLMapperBuilder(configuration);
        for (Element mapperElement : mapperElements) {
            String mapperPath = mapperElement.attributeValue("resource");
            InputStream resourceAsSteam =
                    Resources.getResourceAsSteam(mapperPath);
            xmlMapperBuilder.parse(resourceAsSteam);
        }
        return configuration;
    }
}
```

sqlSessionFactory 接口及D efaultSqlSessionFactory 实现类

```java
public interface SqlSessionFactory {
    public SqlSession openSession();
}
public class DefaultSqlSessionFactory implements SqlSessionFactory { private Configuration configuration;
    public DefaultSqlSessionFactory(Configuration configuration) { this.configuration = configuration;
    }
    public SqlSession openSession(){
        return new DefaultSqlSession(configuration);
    }
}
```

sqlSession 接⼝及 DefaultSqlSession 实现类

```java
public interface SqlSession {
    public <E> List<E> selectList(String statementId, Object... param) Exception;
    public <T> T selectOne(String statementId,Object... params) throws Exception;
    public void close() throws SQLException;
}
```

```java
public class DefaultSqlSession implements SqlSession {
    private Configuration configuration;

    public DefaultSqlSession(Configuration configuration) {
        this.configuration = configuration;
        //处理器对象
        private Executor simpleExcutor = new SimpleExecutor();
        public <E> List<E> selectList(String statementId, Object...param) throws Exception {
            MappedStatement mappedStatement =
                    configuration.getMappedStatementMap().get(statementId);
            List<E> query = simpleExcutor.query(configuration,
                    mappedStatement, param);
            return query;
        }
        //selectOne中调用selectList
        public <T> T selectOne(String statementId, Object...params) throws Exception {
            List<Object> objects = selectList(statementId, params);
            if (objects.size() == 1) {
                return (T) objects.get(0);
            } else {
                throw new RuntimeException("返回结果过多 ");
            }
        }
        public void close () throws SQLException {
            simpleExcutor.close();
        }
    }
}
```

Executor

```java
public interface Executor {
    <E> List<E> query(Configuration configuration, MappedStatement mappedStatement,Object[] param) throws Exception;
    void close() throws SQLException;
}
```

SimpleExecutor

```java
public class SimpleExecutor implements Executor {
    private Connection connection = null;

    public <E> List<E> query(Configuration configuration, MappedStatement
            mappedStatement, Object[] param) throws SQLException, NoSuchFieldException, IllegalAccessException, InstantiationException, IntrospectionException,
            InvocationTargetException {
        //获取连接
        connection = configuration.getDataSource().getConnection();
        // select * from user where id = #{id} and username = #{username} String sql = mappedStatement.getSql();
        //对sql进行处理
        BoundSql boundsql = getBoundSql(sql);
        // select * from where id = ? and username = ?
        String finalSql = boundsql.getSqlText();
        //获取传入参数类型
        Class<?> paramterType = mappedStatement.getParamterType();
        //获取预编译preparedStatement对象
        PreparedStatement preparedStatement =
                connection.prepareStatement(finalSql);
        List<ParameterMapping> parameterMappingList =
                boundsql.getParameterMappingList();
        for (int i = 0; i < parameterMappingList.size(); i++) {
            ParameterMapping parameterMapping = parameterMappingList.get(i);
            String name = parameterMapping.getName();
            //反射
            Field declaredField = paramterType.getDeclaredField(name);
            declaredField.setAccessible(true);
            //参数的值
            Object o = declaredField.get(param[0]);
            //给占位符赋值
            preparedStatement.setObject(i + 1, o);
        }
        ResultSet resultSet = preparedStatement.executeQuery();
        Class<?> resultType = mappedStatement.getResultType();
        ArrayList<E> results = new ArrayList<E>();
        while (resultSet.next()) {
            ResultSetMetaData metaData = resultSet.getMetaData();
            (E) resultType.newInstance();
            int columnCount = metaData.getColumnCount();
            for (int i = 1; i <= columnCount; i++) {
                //属性名
                String columnName = metaData.getColumnName(i);
                //属性值
                Object value = resultSet.getObject(columnName);
               //创建属性描述器，为属性生成读写方法
                PropertyDescriptor propertyDescriptor = new
                        PropertyDescriptor(columnName, resultType);
                //获取写方法
                Method writeMethod = propertyDescriptor.getWriteMethod();
                //向类中写入值
                writeMethod.invoke(o, value);
            }
            results.add(o);
        }
        return results;
    }

    @Override
    public void close() throws SQLException {
        connection.close();
    }

    private BoundSql getBoundSql(String sql) {
        //标记处理类：主要是配合通用标记解析器GenericTokenParser类完成对配置文件等的解 析工作，其中TokenHandler主要完成处理
        ParameterMappingTokenHandler parameterMappingTokenHandler = new ParameterMappingTokenHandler();
        //GenericTokenParser :通用的标记解析器，完成了代码片段中的占位符的解析，然后再根 据给定的标记处理器(TokenHandler)来进行表达式的处理
        //三个参数：分别为openToken (开始标记)、closeToken (结束标记)、handler (标记 处  理器)
        GenericTokenParser genericTokenParser = new GenericTokenParser("# {", "}", parameterMappingTokenHandler);
        String parse = genericTokenParser.parse(sql);
        List<ParameterMapping> parameterMappings =
                parameterMappingTokenHandler.getParameterMappings();
        BoundSql boundSql = new BoundSql(parse, parameterMappings);
        return boundSql;
    }
}
```

BoundSql

```java
public class BoundSql {
    //解析过后的sql语句
    private String sqlText;
    //解析出来的参数
    private List<ParameterMapping> parameterMappingList = new
            ArrayList<ParameterMapping>();

    public BoundSql(String sqlText, List<ParameterMapping>
            parameterMappingList) {
        this.sqlText = sqlText;
        this.parameterMappingList = parameterMappingList;
    }

    public String getSqlText() {
        return sqlText;
    }

    public void setSqlText(String sqlText) {
        this.sqlText = sqlText;
    }

    public List<ParameterMapping> getParameterMappingList() {
        return parameterMappingList;
    }

    public void setParameterMappingList(List<ParameterMapping>
                                                parameterMappingList) {
        this.parameterMappingList = parameterMappingList;
    }
}
```

## 1.5 ⾃定义框架优化

通过上述我们的⾃定义框架，我们解决了JDBC操作数据库带来的⼀些问题：例如频繁创建释放数据库连接，硬编码，⼿动封装返回结果集等问题，但是现在我们继续来分析刚刚完成的⾃定义框架代码，有没有什么问题？

问题如下：

* dao的实现类中存在重复的代码，整个操作的过程模板重复(创建sqlsession,调⽤sqlsession⽅法，关闭 sqlsession)

* dao的实现类中存在硬编码，调⽤sqlsession的⽅法时，参数statement的id硬编码

解决：使⽤代理模式来创建接⼝的代理对象

```java
@Test
public void test2()throws Exception{
        InputStream resourceAsSteam=Resources.getResourceAsSteam(path： "sqlMapConfig.xml")
        SqlSessionFactory build=new
        SqlSessionFactoryBuilder().build(resourceAsSteam);
        SqlSession sqlSession=build.openSession();
        User user=new User();
        user.setld(l);
        user.setUsername("tom");
        //代理对象
        UserMapper userMapper=sqlSession.getMappper(UserMapper.class);User userl=userMapper.selectOne(user);
        System.out.println(userl);
}
```

在sqlSession中添加⽅法

```java
public interface SqlSession {
     public <T> T getMappper(Class<?> mapperClass);
```

实现类

```java
@Override
public<T> T getMappper(Class<?> mapperClass){
        T o=(T)Proxy.newProxyInstance(mapperClass.getClassLoader(),new Class[]{mapperClass},new InvocationHandler(){
@Override
public Object invoke(Object proxy,Method method,Object[]args)
        throws Throwable{
                          // selectOne
                          String methodName=method.getName();
                          // className:namespace
                          String className=method.getDeclaringClass().getName();
                          //statementid
                          String key=className+"."+methodName;
                          MappedStatement mappedStatement=
                          configuration.getMappedStatementMap().get(key);
                          Type genericReturnType=method.getGenericReturnType();ArrayList arrayList=new ArrayList<> ();
                          //判断是否实现泛型类型参数化
                          if(genericReturnType instanceof ParameterizedType){
                            return selectList(key,args);
                            return selectOne(key,args);
                          }
             });
        return o;
}
```

> 框架就像一个摩托这样的现代机器，各个模块各司其职，分工明确（加载xml信息，解析xml节点，调用jdbc接口...）。
> 
> 从框架代码可以看到好的代码习惯：
> 
> - 高内聚的代码，每个类的职责明确，只做一件事
> 
> - 低耦合，面向接口编程，不直接依赖对象。

# 第⼆部分：Mybatis相关概念

## 2.1 对象/关系数据库映射（ORM）

ORM全称Object/Relation Mapping：表示对象-关系映射的缩写

ORM完成⾯向对象的编程语⾔到关系数据库的映射。当ORM框架完成映射后，程序员既可以利⽤⾯向对象程序设计语⾔的简单易⽤性，⼜可以利⽤关系数据库的技术优势。ORM把关系数据库包装成⾯向对象的模型。ORM框架是⾯向对象设计语⾔与关系数据库发展不同步时的中间解决⽅案。采⽤ORM框架后，应⽤程序不再直接访问底层数据库，⽽是以⾯向对象的⽅式来操作持久化对象，⽽ORM框架则将这些⾯向对象的操作转换成底层SQL操作。ORM框架实现的效果：把对持久化对象的保存、修改、删除等操作，转换为对数据库的操作

## 2.2 Mybatis简介

MyBatis是⼀款优秀的基于ORM的半⾃动轻量级持久层框架，它⽀持定制化SQL、存储过程以及⾼级映射。MyBatis避免了⼏乎所有的JDBC代码和⼿动设置参数以及获取结果集。MyBatis可以使⽤简单的XML或注解来配置和映射原⽣类型、接⼝和Java的POJO （Plain Old Java Objects,普通⽼式Java对 象）为数据库中的记录。

## 2.3 Mybatis历史

原是apache的⼀个开源项⽬iBatis, 2010年6⽉这个项⽬由apache software foundation 迁移到了google code，随着开发团队转投Google Code旗下，ibatis3.x正式更名为Mybatis ，代码于2013年11⽉迁移到Github。iBATIS⼀词来源于“internet”和“abatis”的组合，是⼀个基于Java的持久层框架。iBATIS提供的持久层框架包括SQL Maps和Data Access Objects(DAO)

## 2.4 Mybatis优势

Mybatis是⼀个半⾃动化的持久层框架，对开发⼈员开说，核⼼sql还是需要⾃⼰进⾏优化，sql和java编码进⾏分离，功能边界清晰，⼀个专注业务，⼀个专注数据。

# 第三部分：Mybatis基本应⽤

## 3.1 快速⼊⻔

MyBatis官⽹地址：http://www.mybatis.org/mybatis-3/

### 3.1.1 开发步骤：

①添加MyBatis的坐标

②创建user数据表

③编写User实体类

④编写映射⽂件UserMapper.xml

⑤编写核⼼⽂件SqlMapConfig.xml

⑥编写测试类

### 3.1.1 环境搭建：

1. 导⼊MyBatis的坐标和其他相关坐标
   
   ```xml
       <properties>
           <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
           <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
           <java.version>1.8</java.version>
           <maven.compiler.source>1.8</maven.compiler.source>
           <maven.compiler.target>1.8</maven.compiler.target>
       </properties>
       <!--mybatis坐标-->
       <dependency>
           <groupId>org.mybatis</groupId>
           <artifactId>mybatis</artifactId>
           <version>3.4.5</version>
       </dependency>
       <!--mysql驱动坐标-->
       <dependency>
           <groupId>mysql</groupId>
           <artifactId>mysql-connector-java</artifactId>
           <version>5.1.6</version>
           <scope>runtime</scope>
       </dependency>
       <!--单元测试坐标-->
       <dependency>
           <groupId>junit</groupId>
           <artifactId>junit</artifactId>
           <version>4.12</version>
           <scope>test</scope>
       </dependency>
       <!--⽇志坐标-->
       <dependency>
           <groupId>log4j</groupId>
           <artifactId>log4j</artifactId>
           <version>1.2.12</version>
       </dependency>
   ```

2. 创建user数据表
   ![](imgs/2023-11-08-11-20-23-image.png)

3. 编写User实体
   
   ```java
   public class User {
   
   private int id;
   
   private String username;
   
   private String password;
   
   //省略get个set⽅法
   
   }
   ```

4. 编写UserMapper映射⽂件
   
   ```xml
   <?xml version="1.0" encoding="UTF-8" ?><!DOCTYPE mapper  
           PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  
           "http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
   <mapper namespace="userMapper">  
       <select id="findAll" resultType="com.lagou.domain.User">  
           select * from User  
       </select>  
   </mapper>
   ```

5. 编写MyBatis核⼼⽂件
   
   ```xml
   <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN“
   "http://mybatis.org/dtd/mybatis-3-config.dtd">
        <configuration>
        <environments default="development">
        <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
        <property name="driver" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql:///test"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
        </dataSource>
        </environment>
        </environments>
        <mappers>
        <mapper resource="com/lagou/mapper/UserMapper.xml"/>
        </mappers>
        </configuration>
   ```

6. 编写测试代码

   ```java
   //加载核⼼配置⽂件  
   InputStream resourceAsStream=  
   Resources.getResourceAsStream("SqlMapConfig.xml");  
   //获得sqlSession⼯⼚对象  
   SqlSessionFactory sqlSessionFactory=new  
   SqlSessionFactoryBuilder().build(resourceAsStream);  
   //获得sqlSession对象  
   SqlSession sqlSession=sqlSessionFactory.openSession();  
   //执⾏sql语句  
   List<User> userList=sqlSession.selectList("userMapper.findAll");  
   //打印结果  
   System.out.println(userList);  
   //释放资源  
   sqlSession.close();
   ```

### 3.1.4 MyBatis的增删改查操作

   1. 编写UserMapper映射⽂件

      ```xml
      <mapper namespace="userMapper">
       <insert id="add" parameterType="com.lagou.domain.User">
           insert into user values(#{id},#{username},#{password})
       </insert>
      </mapper>
      ```

   2. 编写插⼊实体User的代码

      ```java
      InputStream resourceAsStream =
      Resources.getResourceAsStream("SqlMapConfig.xml");
      SqlSessionFactory sqlSessionFactory = new
                          SqlSessionFactoryBuilder().build(resourceAsStream); 
      SqlSession sqlSession = sqlSessionFactory.openSession();
      int insert = sqlSession.insert("userMapper.add", user);
      System.out.println(insert);
      //提交事务
      sqlSession.commit();
      sqlSession.close();
      ```

   3. 插入操作注意问题

      - 插入语句使用insert标签
      - 在映射文件中使用parameterType属性指定要插入的数据类型 
      - Sql语句中使用#{实体属性名}方式引用实体中的属性值
      - 插⼊操作使⽤的API是sqlSession.insert(“命名空间.id”,实体对象);
      - 插⼊操作涉及数据库数据变化，所以要使⽤sqlSession对象显示的提交事务，即sqlSession.commit()

### 3.1.5 MyBatis的修改数据操作

   1. 编写UserMapper映射⽂件
   
      ```xml
      <mapper namespace="userMapper">
          <update id="update" parameterType="com.lagou.domain.User">
              update user set username=#{username},password=#{password} where id=#
              {id}
          </update>
      </mapper>
      ```
   
      
   
   2. 编写修改实体User的代码
   
      ```java
      InputStream resourceAsStream =
      Resources.getResourceAsStream("SqlMapConfig.xml");
      SqlSessionFactory sqlSessionFactory = new
      SqlSessionFactoryBuilder().build(resourceAsStream);
      SqlSession sqlSession = sqlSessionFactory.openSession();
      int update = sqlSession.update("userMapper.update", user);
      System.out.println(update);
      sqlSession.commit();
      sqlSession.close();
      ```
   
   3. **修改操作注意问题**
   
      - 修改语句使⽤update标签
      - 修改操作使⽤的API是sqlSession.update(“命名空间.id”,实体对象);



## 3.1.6 MyBatis的删除数据操作

1. 编写UserMapper映射⽂件

   ```xml
   <mapper namespace="userMapper">
       <delete id="delete" parameterType="java.lang.Integer">
           delete from user where id=#{id}
       </delete>
   </mapper>
   ```

2. 编写删除数据的代码

   ```java
   InputStream resourceAsStream =
   Resources.getResourceAsStream("SqlMapConfig.xml");
   SqlSessionFactory sqlSessionFactory = new
   SqlSessionFactoryBuilder().build(resourceAsStream);
   SqlSession sqlSession = sqlSessionFactory.openSession();
   int delete = sqlSession.delete("userMapper.delete",3);
   System.out.println(delete);
   sqlSession.commit();
   sqlSession.close();
   ```

3. 删除操作注意问题

   - 删除语句使⽤delete标签
   - Sql语句中使⽤#{任意字符串}⽅式引⽤传递的单个参数
   - 删除操作使⽤的API是sqlSession.delete(“命名空间.id”,Object);

### 3.1.5 MyBatis的映射⽂件概述

![image-20231108182848460](imgs/image-20231108182848460.png)



### 3.1.6 ⼊⻔核⼼配置⽂件分析：

MyBatis核⼼配置⽂件层级关系

![image-20231108183131926](imgs/image-20231108183131926.png)

**MyBatis常⽤配置解析**

1. environments标签
   数据库环境的配置，⽀持多环境配置
   ![image-20231108183502545](imgs/image-20231108183502545.png)
   其中，事务管理器（transactionManager）类型有两种：
   
   - •JDBC：这个配置就是直接使⽤了JDBC 的提交和回滚设置，它依赖于从数据源得到的连接来管理事务作⽤域。
   - MANAGED：这个配置⼏乎没做什么。它从来不提交或回滚⼀个连接，⽽是让容器来管理事务的整个⽣命周期（⽐如 JEE 应⽤服务器的上下⽂）。 默认情况下它会关闭连接，然⽽⼀些容器并不希望这样，因此需要将 closeConnection 属性设置为 false 来阻⽌它默认的关闭⾏为。
   
   其中，数据源（dataSource）类型有三种：
   
   - UNPOOLED：这个数据源的实现只是每次被请求时打开和关闭连接。
   - POOLED：这种数据源的实现利⽤“池”的概念将 JDBC 连接对象组织起来。
   - JNDI：这个数据源的实现是为了能在如 EJB 或应⽤服务器这类容器中使⽤，容器可以集中或在外部配置数据源，然后放置⼀个 JNDI 上下⽂的引⽤。
   
2. mapper标签
   该标签的作⽤是加载映射的，加载⽅式有如下⼏种：
   `•使⽤相对于类路径的资源引⽤，例如：

   <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>

   •使⽤完全限定资源定位符（URL），例如：

   <mapper url="file:///var/mappers/AuthorMapper.xml"/>

   •使⽤映射器接⼝实现类的完全限定类名，例如：

   <mapper class="org.mybatis.builder.AuthorMapper"/>

   •将包内的映射器接⼝实现全部注册为映射器，例如：

   <package name="org.mybatis.builder"/>`

### 3.1.7 Mybatis相应API介绍

**SqlSession⼯⼚构建器SqlSessionFactoryBuilder**

常⽤API：SqlSessionFactory build(InputStream inputStream)

通过加载mybatis的核⼼⽂件的输⼊流的形式构建⼀个SqlSessionFactory对象

```java
String resource = "org/mybatis/builder/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
SqlSessionFactory factory = builder.build(inputStream);
```

其中， Resources ⼯具类，这个类在 org.apache.ibatis.io 包中。Resources 类帮助你从类路径下、⽂件系统或⼀个 web URL 中加载资源⽂件。

**SqlSession⼯⼚对象SqlSessionFactory**

SqlSessionFactory 有多个个⽅法创建SqlSession 实例。常⽤的有如下两个：

![image-20231108194751081](imgs/image-20231108194751081.png)

**SqlSession会话对象**

SqlSession 实例在 MyBatis 中是⾮常强⼤的⼀个类。在这⾥你会看到所有执⾏语句、提交或回滚事务和获取映射器实例的⽅法。

执⾏语句的⽅法主要有：

```java
<T> T selectOne(String statement, Object parameter)
<E> List<E> selectList(String statement, Object parameter)
int insert(String statement, Object parameter)
int update(String statement, Object parameter)
int delete(String statement, Object parameter)
```

操作事务的⽅法主要有：

```java
void commit() 
void rollback()
```

## 3.2 Mybatis的Dao层实现

### 3.2.1 传统开发⽅式

编写UserDao接⼝

```java
public interface UserDao {
 List<User> findAll() throws IOException;
}
```

编写UserDaoImpl实现

```java
public class UserDaoImpl implements UserDao {
 public List<User> findAll() throws IOException {
     InputStream resourceAsStream =
                   Resources.getResourceAsStream("SqlMapConfig.xml");
     SqlSessionFactory sqlSessionFactory = new
               SqlSessionFactoryBuilder().build(resourceAsStream);
     SqlSession sqlSession = sqlSessionFactory.openSession();
     List<User> userList = sqlSession.selectList("userMapper.findAll");
     sqlSession.close();
     return userList;
  }
}
```

测试传统⽅式

```java
@Test
public void testTraditionDao() throws IOException {
     UserDao userDao = new UserDaoImpl();
     List<User> all = userDao.findAll();
     System.out.println(all);
}
```

### 3.2.2 代理开发⽅式

**代理开发⽅式介绍**

采⽤ Mybatis 的代理开发⽅式实现 DAO 层的开发，这种⽅式是我们进⼊企业后的主流。

Mapper 接⼝开发⽅法只需要程序员编写Mapper 接⼝（相当于Dao 接⼝），由Mybatis 框架根据接⼝

定义创建接⼝的动态代理对象，代理对象的⽅法体同上边Dao接⼝实现类⽅法。

Mapper 接⼝开发需要遵循以下规范：

1.  Mapper.xml⽂件中的namespace与mapper接⼝的全限定名相同
2. Mapper接⼝⽅法名和Mapper.xml中定义的每个statement的id相同
3. Mapper接⼝⽅法的输⼊参数类型和mapper.xml中定义的每个sql的parameterType的类型相同
4. Mapper接⼝⽅法的输出参数类型和mapper.xml中定义的每个sql的resultType的类型相同

编写UserMapper接⼝
![image-20231108195902674](imgs/image-20231108195902674.png)

测试代理⽅式

```java
    @Test
    public void testProxyDao() throws IOException {
        InputStream resourceAsStream =
                Resources.getResourceAsStream("SqlMapConfig.xml");
        SqlSessionFactory sqlSessionFactory = new
                SqlSessionFactoryBuilder().build(resourceAsStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        //获得MyBatis框架⽣成的UserMapper接⼝的实现类
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        User user = userMapper.findById(1);
        System.out.println(user);
        sqlSession.close();
    }
```



# 第四部分：Mybatis配置⽂件深⼊

## 4.1 核⼼配置⽂件SqlMapConfig.xml

### 4.1.1 MyBatis核⼼配置⽂件层级关系

![image-20231108183131926](imgs/image-20231108183131926.png)

## 4.2 MyBatis常⽤配置解析

1. environments标签
   数据库环境的配置，⽀持多环境配置
   ![image-20231108200748167](imgs/image-20231108200748167.png)
   
2. Properties标签
   实际开发中，习惯将数据源的配置信息单独抽取成⼀个properties⽂件，该标签可以加载额外配置的properties⽂件
   ![image-20231108201041228](imgs/image-20231108201041228.png)
   
3. typeAliases标签
   类型别名是为Java 类型设置⼀个短的名字。原来的类型名称配置如下
   ![image-20231108201225633](imgs/image-20231108201225633.png)

   配置typeAliases，为com.lagou.domain.User定义别名为user
   ![image-20231108201256548](imgs/image-20231108201256548.png)

   上⾯我们是⾃定义的别名，mybatis框架已经为我们设置好的⼀些常⽤的类型的别名
   ![image-20231108201342714](imgs/image-20231108201342714.png)

## 4.2 映射配置⽂件mapper.xml

**动态sql语句**

**动态sql语句概述**

Mybatis 的映射⽂件中，前⾯我们的 SQL 都是⽐较简单的，有些时候业务逻辑复杂时，我们的 SQL是

动态变化的，此时在前⾯的学习中我们的 SQL 就不能满⾜要求了。

参考的官⽅⽂档，描述如下：
![image-20231108201617879](imgs/image-20231108201617879.png)

**动态 SQL 之**我们根据实体类的不同取值，使⽤不同的 SQL语句来进⾏查询。⽐如在 id如果不为空时可以根据id查询，如果username 不同空时还要加⼊⽤户名作为条件。这种情况在我们的多条件组合查询中经常会碰到。

```xml
<select id="findByCondition" parameterType="user" resultType="user">
    select * from User
    <where>
        <if test="id!=0">
            and id=#{id}
        </if>
        <if test="username!=null">
            and username=#{username}
        </if>
    </where>
</select>
```

当查询条件id和username都存在时，控制台打印的sql语句如下：

```java
        //获得MyBatis框架⽣成的UserMapper接⼝的实现类
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        User condition = new User();
        condition.setId(1);
        condition.setUsername("lucy");
        User user = userMapper.findByCondition(condition);
```

![image-20231108202043364](imgs/image-20231108202043364.png)

当查询条件只有id存在时，控制台打印的sql语句如下：

```java
//获得MyBatis框架⽣成的UserMapper接⼝的实现类
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
User condition = new User();
condition.setId(1);
User user = userMapper.findByCondition(condition);
```

![image-20231108202228266](imgs/image-20231108202228266.png)

**动态 SQL 之**
循环执⾏sql的拼接操作，例如：SELECT * FROM USER WHERE id IN (1,2,5)。

```xml
<select id="findByIds" parameterType="list" resultType="user">
    select * from User
    <where>
        <foreach collection="list" open="id in(" close=")" item="id"
                 separator=",">
            #{id}
        </foreach>
    </where>
</select>
```

测试代码片段如下：

```java
… … …
//获得MyBatis框架⽣成的UserMapper接⼝的实现类
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
int[] ids = new int[]{2,5};
List<User> userList = userMapper.findByIds(ids);
System.out.println(userList);
… … …
```

![image-20231108202606452](imgs/image-20231108202606452.png)

foreach标签的属性含义如下：

标签⽤于遍历集合，它的属性：

•collection：代表要遍历的集合元素，注意编写时不要写#{}

•open：代表语句的开始部分

•close：代表结束部分

•item：代表遍历集合的每个元素，⽣成的变量名

•sperator：代表分隔符

**SQL⽚段抽取**

Sql 中可将重复的 sql 提取出来，使⽤时⽤ include 引⽤即可，最终达到 sql 重⽤的⽬的

```xml
<!--抽取sql⽚段简化编写-->
<sql id="selectUser" select * from User</sql>
<select id="findById" parameterType="int" resultType="user">
    <include refid="selectUser"></include> where id=#{id}
</select>
<select id="findByIds" parameterType="list" resultType="user">
    <include refid="selectUser"></include>
    <where>
        <foreach collection="array" open="id in(" close=")" item="id"
separator=",">
            #{id}
            </foreach>
    </where>
</select>
```

# 第五部分：Mybatis复杂映射开发

## 5.1 ⼀对⼀查询

⽤户表和订单表的关系为，⼀个⽤户有多个订单，⼀个订单只从属于⼀个⽤户

⼀对⼀查询的需求：查询⼀个订单，与此同时查询出该订单所属的⽤户

![image-20231109145352032](imgs/image-20231109145352032.png)

### 5.1.2⼀对⼀查询的语句

对应的sql语句：select * from orders o,user u where o.uid=u.id;

查询的结果如下：
![image-20231109145444296](imgs/image-20231109145444296.png)

### 5.1.3 创建Order和User实体

```java
public class Order {
    private int id;
    private Date ordertime;
    private double total;
    //代表当前订单从属于哪⼀个客户
    private User user;
}
public class User {

    private int id;
    private String username;
    private String password;
    private Date birthday;
}
```

### 5.1.4 创建OrderMapper接⼝

```java
public interface OrderMapper {
     List<Order> findAll();
}
```

### 5.1.5 配置OrderMapper.xml

```xml
<mapper namespace="com.lagou.mapper.OrderMapper">
    <resultMap id="orderMap" type="com.lagou.domain.Order">
        <result column="uid" property="user.id"></result>
        <result column="username" property="user.username"></result>
        <result column="password" property="user.password"></result>
        <result column="birthday" property="user.birthday"></result>
    </resultMap>
    <select id="findAll" resultMap="orderMap">
        select * from orders o,user u where o.uid=u.id
    </select>
</mapper>
```

其中还可以配置如下：

```xml
<resultMap id="orderMap" type="com.lagou.domain.Order">
    <result property="id" column="id"></result>
    <result property="ordertime" column="ordertime"></result>
    <result property="total" column="total"></result>
    <association property="user" javaType="com.lagou.domain.User">
        <result column="uid" property="id"></result>
        <result column="username" property="username"></result>
        <result column="password" property="password"></result>
        <result column="birthday" property="birthday"></result>
    </association>
</resultMap>
```

### 5.1.6 测试结果

```java
OrderMapper mapper = sqlSession.getMapper(OrderMapper.class);
List<Order> all = mapper.findAll();
for(Order order : all){
  System.out.println(order);
}
```

![image-20231109150427019](imgs/image-20231109150427019.png)



## 5.2 ⼀对多查询

### 5.2.1 ⼀对多查询的模型

⽤户表和订单表的关系为，⼀个⽤户有多个订单，⼀个订单只从属于⼀个⽤户

⼀对多查询的需求：查询⼀个⽤户，与此同时查询出该⽤户具有的订单
![image-20231109150733433](imgs/image-20231109150733433.png)

### 5.2.2 ⼀对多查询的语句

对应的sql语句：select *,o.id oid from user u left join orders o on u.id=o.uid;

查询的结果如下：

![image-20231109150903600](imgs/image-20231109150903600.png)

### 5.2.3 修改User实体

```java
public class Order {
    private int id;
    private Date ordertime;
    private double total;
    //代表当前订单从属于哪⼀个客户
    private User user;
}
public class User {

    private int id;
    private String username;
    private String password;
    private Date birthday;
    //代表当前⽤户具备哪些订单
    private List<Order> orderList;
}
```



### 5.2.4 创建UserMapper接⼝

```java
public interface UserMapper {
   List<User> findAll();
}
```

### 5.2.5 配置UserMapper.xml

```xml
<mapper namespace="com.lagou.mapper.UserMapper">
    <resultMap id="userMap" type="com.lagou.domain.User">
        <result column="id" property="id"></result>
        <result column="username" property="username"></result>
        <result column="password" property="password"></result>
        <result column="birthday" property="birthday"></result>
        <collection property="orderList" ofType="com.lagou.domain.Order">
            <result column="oid" property="id"></result>
            <result column="ordertime" property="ordertime"></result>
            <result column="total" property="total"></result>
        </collection>
    </resultMap>
    <select id="findAll" resultMap="userMap">
        select *,o.id oid from user u left join orders o on u.id=o.uid
    </select>
</mapper>
```

### 5.2.6 测试结果

```java
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
List<User> all = mapper.findAll();
for(User user : all){
    System.out.println(user.getUsername());
    List<Order> orderList = user.getOrderList();
    for(Order order : orderList){
          System.out.println(order);
    }
    System.out.println("----------------------------------");
}
```

![image-20231109151552747](imgs/image-20231109151552747.png)

## 5.3 多对多查询

### 5.3.1 多对多查询的模型

⽤户表和⻆⾊表的关系为，⼀个⽤户有多个⻆⾊，⼀个⻆⾊被多个⽤户使⽤

多对多查询的需求：查询⽤户同时查询出该⽤户的所有⻆⾊
![image-20231109151930974](imgs/image-20231109151930974.png)

### 5.3.2 多对多查询的语句

对应的sql语句：select u.*,r.*,r.id rid from user u left join user_role ur on u.id=ur.user_id

 inner join role r on ur.role_id=r.id;

查询的结果如下：
![image-20231109152103734](imgs/image-20231109152103734.png)

### 5.3.3 创建Role实体，修改User实体

```java
public class User {
    private int id;
    private String username;
    private String password;
    private Date birthday;
    //代表当前用户具备哪些订单
    private List<Order> orderList;
    //代表当前用户具备哪些角色
    private List<Role> roleList;
}

public class Role {

    private int id;
    private String rolename;
}
```

### 5.3.4 添加UserMapper接⼝⽅法

```java
List<User>findAllUserAndRole();
```

### 5.3.5 配置UserMapper.xml

```xml
    <resultMap id="userRoleMap" type="com.lagou.domain.User">
        <result column="id" property="id"></result>
        <result column="username" property="username"></result>
        <result column="password" property="password"></result>
        <result column="birthday" property="birthday"></result>
        <collection property="roleList" ofType="com.lagou.domain.Role">
            <result column="rid" property="id"></result>
            <result column="rolename" property="rolename"></result>
        </collection>
    </resultMap>
    <select id="findAllUserAndRole" resultMap="userRoleMap">
        select u.*,r.*,r.id rid from user u
        left join user_role ur on u.id=ur.user_id
        inner join role r on ur.role_id=r.id
    </select>
```

### 5.3.6 测试结果

```java
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        List<User> all = mapper.findAllUserAndRole();
        for(User user : all){
            System.out.println(user.getUsername());
            List<Role> roleList = user.getRoleList();
            for(Role role : roleList){
                System.out.println(role);
            }
            System.out.println("----------------------------------");
        }
```

![image-20231109153549614](imgs/image-20231109153549614.png)



## 5.4 知识小结

MyBatis多表配置方式：
一对一配置：使用做配置
一对多配置：使用 +做配置
多对多配置：使用 +做配置

# 第六部分：  Mybatis注解开发

## 6.1 MyBatis的常用注解

> 注解主要的功能就是把Mapper映射文件里的sql，映射配置项，提取到了注解中。这样查找修改sql时应该会比较方便。
>
> 还有一点，在处理复杂映射查询时，注解方式通过分别多次执行sql，节省一次sql的join。即原来的一个join的sql变成了两个简单sql，框架内部多次执行sql，拼装结果。

这几年来注解开发越来越流行， Mybatis也可以使用注解开发方式，这样我们就可以减少编写Mapper

映射文件了。我们先围绕一些基本的CRUD来学习，再学习复杂映射多表操作。

@Insert：实现新增

@Update：实现更新

@Delete：实现删除

@Select：实现查询

@Result：实现结果集封装

@Results：可以与@Result 一起使用，封装多个结果集

@One：实现一对一结果集封装

@Many：实现一对多结果集封装

## 6.2 MyBatis的增删改查

我们完成简单的user表的增删改查的操作

```java
    private UserMapper userMapper;

    @Before
    public void before() throws IOException {
        InputStream resourceAsStream =
                Resources.getResourceAsStream("SqlMapConfig.xml");
        SqlSessionFactory sqlSessionFactory = new
                SqlSessionFactoryBuilder().build(resourceAsStream); 
        SqlSession sqlSession = sqlSessionFactory.openSession(true);
        userMapper = sqlSession.getMapper(UserMapper.class);
    }

    @Test
    public void testAdd() {
        User user = new User();
        user.setUsername("测试数据 ");
        user.setPassword("123");
        user.setBirthday(new Date());
        userMapper.add(user);
    }
    @Test
    public void testUpdate() throws IOException {
        User user = new User();
        user.setId(16);
        user.setUsername("测试数据修改 ");
        user.setPassword("abc");
        user.setBirthday(new Date());
        userMapper.update(user);
    }
```

修改MyBatis的核心配置文件，我们使用了注解替代的映射文件，所以我们只需要加载使用了注解的 Mapper接口即可

```xml
<mappers>
    <!--扫描使用注解的类-->
    <mapper class="com.lagou.mapper.UserMapper"></mapper>
</mappers>
```

或者指定扫描包含映射关系的接口所在的包也可以

```xml
<mappers>
<!--扫描使用注解的类所在的包-->
<package name="com.lagou.mapper"></package>
</mappers>
```

## 6.3 MyBatis的注解实现复杂映射开发

> 这里可以看出，在处理复杂映射查询时（一对一，一对多，多对多），注解方式通过分别多次执行sql，节省一次sql的join。即原来的一个join的sql变成了两个简单sql，框架内部多次执行sql，拼装结果。

实现复杂关系映射之前我们可以在映射⽂件中通过配置来实现，使⽤注解开发后，我们可以使⽤

@Results注解，@Result注解，@One注解，@Many注解组合完成复杂关系的配置
![image-20231109185853075](imgs/image-20231109185853075.png)

![image-20231109185915133](imgs/image-20231109185915133.png)

## 6.4 ⼀对⼀查询

### 6.4.1 ⼀对⼀查询的模型

⼀个订单只从属于⼀个⽤户:

⼀对⼀查询的需求：查询⼀个订单，与此同时查询出该订单所属的⽤户

**对应的sql语句：**

```sql
select * from orders;
select * from user where id=查询出订单的uid;
```

**使⽤注解配置Mapper**

```java
    public interface OrderMapper {
        @Select("select * from orders")
        @Results({
                @Result(id=true,property = "id",column = "id"),
                @Result(property = "ordertime",column = "ordertime"), @Result(property = "total",column = "total"),
                @Result(property = "user",column = "uid",
                        javaType = User.class,
                        one = @One(select =
                                "com.lagou.mapper.UserMapper.findById"))
        })
        List<Order> findAll();
    }
```

```java
public interface UserMapper {

    @Select("select * from user where id=#{id}")
    User findById(int id);

}
```

**测试结果**

```java
    @Test
    public void testSelectOrderAndUser() {
        List<Order> all = orderMapper.findAll();
        for(Order order : all){
            System.out.println(order);
        }
    }
```

![image-20231109190727903](imgs/image-20231109190727903.png)

## 6.5 一对多查询

***一对多查询的模型***

用户表和订单表的关系为，一个用户有多个订单，一个订单只从属于一个用户

一对多查询的需求：查询一个用户，与此同时查询出该用户具有的订单
**一对多查询的语句**

```sql
select * from user;

select * from orders where uid=查询出用户的id;
```

**实体**

```java
public class Order {

private int id;
private Date ordertime;
private double total;

//代表当前订单从属于哪一个客户
private User user;
}

public class User {

private int id;
private String username;
private String password;
private Date birthday;
//代表当前用户具备哪些订单
private List<Order> orderList;
}
```

 **使用注解配置UserMapper接口**

```java
public interface UserMapper {
    @Select("select * from user")
    @Results({
            @Result(id = true,property = "id",column = "id"),
            @Result(property = "username",column = "username"),
            @Result(property = "password",column = "password"),
            @Result(property = "birthday",column = "birthday"),
            @Result(property = "orderList",column = "id",
                    javaType = List.class,
                    many = @Many(select =
                            "com.lagou.mapper.OrderMapper.findByUid"))
    })
    List<User> findAllUserAndOrder();
}

public interface OrderMapper {
    @Select("select * from orders where uid=#{uid}")
    List<Order> findByUid(int uid);

}
```

**测试结果**

```java
        List<User> all = userMapper.findAllUserAndOrder();
        for(User user : all){
            System.out.println(user.getUsername());
            List<Order> orderList = user.getOrderList();
            for(Order order : orderList){
                System.out.println(order);
            }
            System.out.println("-----------------------------");
        }
```

![image-20231109191909452](imgs/image-20231109191909452.png)

## 6.6 多对多查询

**多对多查询的模型**

⽤户表和⻆⾊表的关系为，⼀个⽤户有多个⻆⾊，⼀个⻆⾊被多个⽤户使⽤

多对多查询的需求：查询⽤户同时查询出该⽤户的所有⻆⾊

**对应的sql语句：**

```sql
select * from user;
select * from role r,user_role ur where r.id=ur.role_id and ur.user_id=⽤户的id
```

**创建Role实体，修改User实体**

```java
public class User {
    private int id;
    private String username;
    private String password;
    private Date birthday;
    //代表当前用户具备哪些订单
    private List<Order> orderList;
    //代表当前用户具备哪些角色
    private List<Role> roleList;
}

public class Role {

    private int id;
    private String rolename;
}
```

**使用注解配置Mapper**

```java
public interface UserMapper {
    @Select("select * from user")
    @Results({
            @Result(id=true,property="id",column="id"),
            @Result(property="username",column="username"),
            @Result(property="password",column="password"),
            @Result(property="birthday",column="birthday"),
            @Result(property="roleList",column="id",javaType=List.class,
            many=@Many(select=
                    "com.lagou.mapper.RoleMapper.findByUid"))})
    List<User>findAllUserAndRole();}


public interface RoleMapper {
    @Select("select * from role r,user_role ur where r.id=ur.role_id and ur.user_id=#{uid}")
    List<Role> findByUid(int uid);
}
```

**测试结果**

```java
    public void test（）{
        UserMapper mapper=sqlSession.getMapper(UserMapper.class);
        List<User> all=mapper.findAllUserAndRole();
        for(User user:all){
            System.out.println(user.getUsername());
            List<Role> roleList=user.getRoleList();
            for(Role role:roleList){
                System.out.println(role);
            }
            System.out.println("----------------------------------");
        }
    }
```

![image-20231109192706111](imgs/image-20231109192706111.png)



# 第七部分：Mybatis缓存

## 7.1 ⼀级缓存

1. 在⼀个sqlSession中，对User表根据id进⾏两次查询，查看他们发出sql语句的情况

   ```java
   @Test
   public void test1(){
   //根据 sqlSessionFactory 产⽣ session
    SqlSession sqlSession = sessionFactory.openSession();
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    //第⼀次查询，发出sql语句，并将查询出来的结果放进缓存中
    User u1 = userMapper.selectUserByUserId(1);
    System.out.println(u1);
    //第⼆次查询，由于是同⼀个sqlSession,会在缓存中查询结果
    //如果有，则直接从缓存中取出来，不和数据库进⾏交互
    User u2 = userMapper.selectUserByUserId(1);
    System.out.println(u2);
    sqlSession.close();
   }
   ```

   查看控制台打印情况：
   ![image-20231115141814226](imgs/image-20231115141814226.png)

2. 同样是对user表进⾏两次查询，只不过两次查询之间进⾏了⼀次update操作。

   ```java
   @Test
   public void test2(){
    //根据 sqlSessionFactory 产⽣ session
    SqlSession sqlSession = sessionFactory.openSession();
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    //第⼀次查询，发出sql语句，并将查询的结果放⼊缓存中
    User u1 = userMapper.selectUserByUserId( 1 );
    System.out.println(u1);
    //第⼆步进⾏了⼀次更新操作，sqlSession.commit()
    u1.setSex("⼥");
    userMapper.updateUserByUserId(u1);
    sqlSession.commit();
    //第⼆次查询，由于是同⼀个sqlSession.commit(),会清空缓存信息
    //则此次查询也会发出sql语句
    User u2 = userMapper.selectUserByUserId(1);
    System.out.println(u2);
    sqlSession.close();
   }
   ```

   查看控制台打印情况：
   ![image-20231115142045183](imgs/image-20231115142045183.png)

3. 总结
   1、第⼀次发起查询⽤户id为1的⽤户信息，先去找缓存中是否有id为1的⽤户信息，如果没有，从 数据

   库查询⽤户信息。得到⽤户信息，将⽤户信息存储到⼀级缓存中。

   2、 如果中间sqlSession去执⾏commit操作（执⾏插⼊、更新、删除），则会清空SqlSession中的 ⼀

   级缓存，这样做的⽬的为了让缓存中存储的是最新的信息，避免脏读。

   3、 第⼆次发起查询⽤户id为1的⽤户信息，先去找缓存中是否有id为1的⽤户信息，缓存中有，直 接从

   缓存中获取⽤户信息

![image-20231115142227348](imgs/image-20231115142227348.png)

**⼀级缓存原理探究与源码分析**

⼀级缓存到底是什么？⼀级缓存什么时候被创建、⼀级缓存的⼯作流程是怎样的？相信你现在应该会有

这⼏个疑问，那么我们本节就来研究⼀下⼀级缓存的本质

⼤家可以这样想，上⾯我们⼀直提到⼀级缓存，那么提到⼀级缓存就绕不开SqlSession,所以索性我们

就直接从SqlSession，看看有没有创建缓存或者与缓存有关的属性或者⽅法

![image-20231115142335097](imgs/image-20231115142335097.png)

调研了⼀圈，发现上述所有⽅法中，好像只有clearCache()和缓存沾点关系，那么就直接从这个⽅ 法⼊
⼿吧，分析源码时，**我们要看它(此类)是谁，它的⽗类和⼦类分别⼜是谁**，对如上关系了解了，你才 会
对这个类有更深的认识，分析了⼀圈，你可能会得到如下这个流程图

![image-20231115142458262](imgs/image-20231115142458262.png)

再深⼊分析，流程⾛到**Perpetualcache**中的clear()⽅法之后，会调⽤其**cache.clear()**⽅法，那 么这个

cache是什么东⻄呢？点进去发现，cache其实就是private Map cache = new

HashMap()；也就是⼀个Map，所以说cache.clear()其实就是map.clear()，也就是说，缓存其实就是

本地存放的⼀个map对象，每⼀个SqISession都会存放⼀个map对象的引⽤，那么这个cache是何 时创

建的呢？

你觉得最有可能创建缓存的地⽅是哪⾥呢？我觉得是**Executor**，为什么这么认为？因为Executor是 执

⾏器，⽤来执⾏SQL请求，⽽且清除缓存的⽅法也在Executor中执⾏，所以很可能缓存的创建也很 有可

能在Executor中，看了⼀圈发现Executor中有⼀个createCacheKey⽅法，这个⽅法很像是创 建缓存的

⽅法啊，跟进去看看，你发现createCacheKey⽅法是由BaseExecutor执⾏的，代码如下

```java
CacheKey cacheKey = new CacheKey();
//MappedStatement 的 id
// id就是Sql语句的所在位置包名+类名+ SQL名称
cacheKey.update(ms.getId());
// offset 就是 0
cacheKey.update(rowBounds.getOffset());
// limit 就是 Integer.MAXVALUE
cacheKey.update(rowBounds.getLimit());
//具体的SQL语句
cacheKey.update(boundSql.getSql());
//后⾯是update 了 sql中带的参数
cacheKey.update(value);
...
if (configuration.getEnvironment() != null) {
// issue #176
cacheKey.update(configuration.getEnvironment().getId());
}
```

创建缓存key会经过⼀系列的update⽅法，udate⽅法由⼀个CacheKey这个对象来执⾏的，这个

update⽅法最终由updateList的list来把五个值存进去，对照上⾯的代码和下⾯的图示，你应该能 理解

这五个值都是什么了

![image-20231115142935296](imgs/image-20231115142935296.png)

这⾥需要注意⼀下最后⼀个值，configuration.getEnvironment().getId()这是什么，这其实就是 定义在

mybatis-config.xml中的标签，⻅如下。

```xml
<environments default="development">
<environment id="development">
<transactionManager type="JDBC"/>
<dataSource type="POOLED">
 <property name="driver" value="${jdbc.driver}"/>
 <property name="url" value="${jdbc.url}"/>
 <property name="username" value="${jdbc.username}"/>
 <property name="password" value="${jdbc.password}"/>
 </dataSource>
</environment>
</environments>
```

那么我们回归正题，那么创建完缓存之后该⽤在何处呢？总不会凭空创建⼀个缓存不使⽤吧？绝对不会

的，经过我们对⼀级缓存的探究之后，我们发现⼀级缓存更多是⽤**于查询操作，毕竟⼀级缓存也叫做查**

**询缓存吧，为什么叫查询缓存我们⼀会⼉说**。我们先来看⼀下这个缓存到底⽤在哪了，我们跟踪到

query⽅法如下：

```java
Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds
rowBounds, ResultHandler resultHandler) throws SQLException {
 BoundSql boundSql = ms.getBoundSql(parameter);
 //创建缓存
 CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
 return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
@SuppressWarnings("unchecked")
Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds
rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
throws SQLException {
 ...
 list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
if (list != null) {
 //这个主要是处理存储过程⽤的。
 handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
 } else {
 list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key,
boundSql);
 }
 ...
}

// queryFromDatabase ⽅法
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter,
RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql
boundSql) throws SQLException {
 List<E> list;
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
 try {
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

如果查不到的话，就从数据库查，在queryFromDatabase中，会对localcache进⾏写⼊。 localcache

对象的put⽅法最终交给Map进⾏存放

```java
private Map<Object, Object> cache = new HashMap<Object, Object>();
 @Override
 public void putObject(Object key, Object value) { cache.put(key, value);
}
```

## 7.2 ⼆级缓存

⼆级缓存的原理和⼀级缓存原理⼀样，第⼀次查询，会将数据放⼊缓存中，然后第⼆次查询则会直接去

缓存中取。但是⼀级缓存是基于sqlSession的，⽽⼆级缓存是基于mapper⽂件的namespace的，也 就

是说多个sqlSession可以共享⼀个mapper中的⼆级缓存区域，并且如果两个mapper的namespace 相

同，即使是两个mapper,那么这两个mapper中执⾏sql查询到的数据也将存在相同的⼆级缓存区域 中

![image-20231115201730402](D:\learning\lg\imgs/image-20231115201730402.png)

> 可以看到，Mybatis框架的缓存和Mysql自带的缓存一样，都面对数据更新，缓存全部清空失效的问题。所以效率不高。MySql在8版本移除了缓存。

如何使⽤⼆级缓存

1. 开启⼆级缓存和⼀级缓存默认开启不⼀样，⼆级缓存需要我们⼿动开启

   ⾸先在全局配置⽂件sqlMapConfig.xml⽂件中加⼊如下代码:

   ```xml
   <!--开启⼆级缓存-->
   <settings>
    <setting name="cacheEnabled" value="true"/>
    </settings>
   ```

   其次在UserMapper.xml⽂件中开启缓存

   ```xml
   <!--开启⼆级缓存-->
   <cache></cache>
   ```

   我们可以看到mapper.xml⽂件中就这么⼀个空标签，其实这⾥可以配置,PerpetualCache这个类是

   mybatis默认实现缓存功能的类。我们不写type就使⽤mybatis默认的缓存，也可以去实现Cache接⼝

   来⾃定义缓存。
   ![image-20231115202403462](imgs/image-20231115202403462.png)

```java
public class PerpetualCache implements Cache {
 private final String id;
 private MapcObject, Object> cache = new HashMapC);
 
 public PerpetualCache(St ring id) { this.id = id;
}
```

我们可以看到⼆级缓存底层还是HashMap结构

```java
public class User implements Serializable(
 //⽤户ID
 private int id;
 //⽤户姓名
 private String username;
 //⽤户性别
 private String sex;
}
```

开启了⼆级缓存后，还需要将要缓存的pojo实现Serializable接⼝，为了将缓存数据取出执⾏反序列化操

作，因为⼆级缓存数据存储介质多种多样，不⼀定只存在内存中，有可能存在硬盘中，如果我们要再取

这个缓存的话，就需要反序列化了。所以mybatis中的pojo都去实现Serializable接⼝

3. 测试
   ⼀、测试⼆级缓存和sqlSession⽆关

   ```java
   @Test
   public void testTwoCache(){
    //根据 sqlSessionFactory 产⽣ session
    SqlSession sqlSession1 = sessionFactory.openSession();
    SqlSession sqlSession2 = sessionFactory.openSession();
    
    UserMapper userMapper1 = sqlSession1.getMapper(UserMapper. class );
    UserMapper userMapper2 = sqlSession2.getMapper(UserMapper. class );
    //第⼀次查询，发出sql语句，并将查询的结果放⼊缓存中
    User u1 = userMapper1.selectUserByUserId(1);
    System.out.println(u1);
    sqlSession1.close(); //第⼀次查询完后关闭 sqlSession
    
    //第⼆次查询，即使sqlSession1已经关闭了，这次查询依然不发出sql语句
    User u2 = userMapper2.selectUserByUserId(1);
    System.out.println(u2);
    sqlSession2.close();
   ```

   可以看出上⾯两个不同的sqlSession,第⼀个关闭了，第⼆次查询依然不发出sql查询语句
   ⼆、测试执⾏commit()操作，⼆级缓存数据清空

   ```java
   @Test
   public void testTwoCache(){
    //根据 sqlSessionFactory 产⽣ session
    SqlSession sqlSession1 = sessionFactory.openSession();
    SqlSession sqlSession2 = sessionFactory.openSession();
    SqlSession sqlSession3 = sessionFactory.openSession();
    String statement = "com.lagou.pojo.UserMapper.selectUserByUserld" ;
    UserMapper userMapper1 = sqlSession1.getMapper(UserMapper. class );
    UserMapper userMapper2 = sqlSession2.getMapper(UserMapper. class );
    UserMapper userMapper3 = sqlSession2.getMapper(UserMapper. class );
    //第⼀次查询，发出sql语句，并将查询的结果放⼊缓存中
    User u1 = userMapperl.selectUserByUserId( 1 );
    System.out.println(u1);
    sqlSessionl .close(); //第⼀次查询完后关闭sqlSession
    
    //执⾏更新操作，commit()
    u1.setUsername( "aaa" );
    userMapper3.updateUserByUserId(u1);
    sqlSession3.commit();
    
    //第⼆次查询，由于上次更新操作，缓存数据已经清空(防⽌数据脏读)，这⾥必须再次发出sql语
    User u2 = userMapper2.selectUserByUserId( 1 );
    System.out.println(u2);
    sqlSession2.close();
   }
   ```

   查看控制台情况：
   ![image-20231115202807762](imgs/image-20231115202807762.png)

4. useCache和flushCache
   mybatis中还可以配置userCache和flushCache等配置项，userCache是⽤来设置是否禁⽤⼆级缓 存

   的，在statement中设置useCache=false可以禁⽤当前select语句的⼆级缓存，即每次查询都会发出 sql

   去查询，默认情况是true,即该sql使⽤⼆级缓存

   ```xml
   <select id="selectUserByUserId" useCache="false"
   resultType="com.lagou.pojo.User" parameterType="int">
    select * from user where id=#{id}
   </select>
   ```

   这种情况是针对每次查询都需要最新的数据sql,要设置成useCache=false，禁⽤⼆级缓存，直接从数 据

   库中获取。

   在mapper的同⼀个namespace中，如果有其它insert、update, delete操作数据后需要刷新缓 存，如

   果不执⾏刷新缓存会出现脏读。

   设置statement配置中的flushCache="true”属性，默认情况下为true,即刷新缓存，如果改成false则 不

   会刷新。使⽤缓存时如果⼿动修改数据库表中的查询数据会出现脏读。

   ```xml
   <select id="selectUserByUserId" flushCache="true" useCache="false"
   resultType="com.lagou.pojo.User" parameterType="int">
    select * from user where id=#{id}
   </select>
   ```

   ⼀般下执⾏完commit操作都需要刷新缓存，flushCache=true表示刷新缓存，这样可以避免数据库脏

   读。所以我们不⽤设置，默认即可

## 7.3 ⼆级缓存整合redis

上⾯我们介绍了 mybatis⾃带的⼆级缓存，但是这个缓存是单服务器⼯作，⽆法实现分布式缓存。 那么

什么是分布式缓存呢？假设现在有两个服务器1和2,⽤户访问的时候访问了 1服务器，查询后的缓 存就

会放在1服务器上，假设现在有个⽤户访问的是2服务器，那么他在2服务器上就⽆法获取刚刚那个 缓

存，如下图所示

![image-20231116103954756](imgs/image-20231116103954756.png)

为了解决这个问题，就得找⼀个分布式的缓存，专⻔⽤来存储缓存数据的，这样不同的服务器要缓存数

据都往它那⾥存，取缓存数据也从它那⾥取，如下图所示：
![image-20231116104024490](C:\Users\PC\AppData\Roaming\Typora\typora-user-images\image-20231116104024490.png)

如上图所示，在⼏个不同的服务器之间，我们使⽤第三⽅缓存框架，将缓存都放在这个第三⽅框架中,

然后⽆论有多少台服务器，我们都能从缓存中获取数据。

这⾥我们介绍**mybatis**与**redis**的整合。

刚刚提到过，mybatis提供了⼀个eache接⼝，如果要实现⾃⼰的缓存逻辑，实现cache接⼝开发即可。

mybati s本身默认实现了⼀个，但是这个缓存的实现⽆法实现分布式缓存，所以我们要⾃⼰来实现。

redis分布式缓存就可以，mybatis提供了⼀个针对cache接⼝的redis实现类，该类存在mybatis-redis包

中

实现

1. pom⽂件

   ```xml
   <dependency>
    <groupId>org.mybatis.caches</groupId>
    <artifactId>mybatis-redis</artifactId>
    <version>1.0.0-beta2</version>
   </dependency>
   ```

2. 配置文件
   Mapper.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
   "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   <mapper namespace="com.lagou.mapper.IUserMapper">
       
   <cache type="org.mybatis.caches.redis.RedisCache" />
       
   <select id="findAll" resultType="com.lagou.pojo.User" useCache="true">
      select * from user
   </select>
   ```

3. .redis.properties

   ```properties
   redis.host=localhost
   redis.port=6379
   redis.connectionTimeout=5000
   redis.password=
   redis.database=0
   ```

4. 测试

   ```java
   @Test
   public void SecondLevelCache(){
    SqlSession sqlSession1 = sqlSessionFactory.openSession();
    SqlSession sqlSession2 = sqlSessionFactory.openSession();
    SqlSession sqlSession3 = sqlSessionFactory.openSession();
    IUserMapper mapper1 = sqlSession1.getMapper(IUserMapper.class);
    lUserMapper mapper2 = sqlSession2.getMapper(lUserMapper.class);
    lUserMapper mapper3 = sqlSession3.getMapper(IUserMapper.class);
    User user1 = mapper1.findUserById(1);
    sqlSession1.close(); //清空⼀级缓存
    
    User user = new User();
    user.setId(1);
    user.setUsername("lisi");
    mapper3.updateUser(user);
    sqlSession3.commit();
    User user2 = mapper2.findUserById(1);
    System.out.println(user1==user2);
   }
   ```

**源码分析**
RedisCache和⼤家普遍实现Mybatis的缓存⽅案⼤同⼩异，⽆⾮是实现Cache接⼝，并使⽤jedis操作缓

存；不过该项⽬在设计细节上有⼀些区别；

```java
public final class RedisCache implements Cache {
 public RedisCache(final String id) {
 if (id == null) {
 throw new IllegalArgumentException("Cache instances require anID");
}
 this.id = id;
 RedisConfig redisConfig =
 RedisConfigurationBuilder.getInstance().parseConfiguration();
 pool = new JedisPool(redisConfig, redisConfig.getHost(),
 redisConfig.getPort(),
 redisConfig.getConnectionTimeout(),
 redisConfig.getSoTimeout(), redisConfig.getPassword(),
redisConfig.getDatabase(), redisConfig.getClientName());
}
```

RedisCache在mybatis启动的时候，由MyBatis的CacheBuilder创建，创建的⽅式很简单，就是调⽤

RedisCache的带有String参数的构造⽅法，即RedisCache(String id)；⽽在RedisCache的构造⽅法中，

调⽤了 RedisConfigu rationBuilder 来创建 RedisConfig 对象，并使⽤ RedisConfig 来创建JedisPool。

RedisConfig类继承了 JedisPoolConfig，并提供了 host,port等属性的包装，简单看⼀下RedisConfig的

属性：

```java
public class RedisConfig extends JedisPoolConfig {
private String host = Protocol.DEFAULT_HOST;
private int port = Protocol.DEFAULT_PORT;
private int connectionTimeout = Protocol.DEFAULT_TIMEOUT;
private int soTimeout = Protocol.DEFAULT_TIMEOUT;
private String password;
private int database = Protocol.DEFAULT_DATABASE;
private String clientName;
```

RedisConfig对象是由RedisConfigurationBuilder创建的，简单看下这个类的主要⽅法：

```java
        public RedisConfig parseConfiguration(ClassLoader classLoader) {
            Properties config = new Properties();
            InputStream input =
                    classLoader.getResourceAsStream(redisPropertiesFilename);
            if (input != null) {
                try {
                    config.load(input);
                } catch (IOException e) {
                    throw new RuntimeException(
                            "An error occurred while reading classpath property '"
                                    + redisPropertiesFilename
                                    + "', see nested exceptions", e);
                } finally {
                    try {
                        input.close();
                    } catch (IOException e) {
                        // close quietly
                    }
                }
            }
            RedisConfig jedisConfig = new RedisConfig();
            setConfigProperties(config, jedisConfig);
            return jedisConfig;
        }
```

核⼼的⽅法就是parseConfiguration⽅法，该⽅法从classpath中读取⼀个redis.properties⽂件:

```properties
host=localhost
port=6379
connectionTimeout=5000
soTimeout=5000
password= database=0 clientName=
```

并将该配置⽂件中的内容设置到RedisConfig对象中，并返回；接下来，就是RedisCache使⽤

RedisConfig类创建完成edisPool；在RedisCache中实现了⼀个简单的模板⽅法，⽤来操作Redis：

```java
private Object execute(RedisCallback callback) {
 Jedis jedis = pool.getResource();
 try {
   return callback.doWithRedis(jedis);
 } finally {
   jedis.close();
 }
}
```

模板接⼝为RedisCallback，这个接⼝中就只需要实现了⼀个doWithRedis⽅法⽽已：

```java
public interface RedisCallback {
 Object doWithRedis(Jedis jedis);
}
```

接下来看看Cache中最重要的两个⽅法：putObject和getObject，通过这两个⽅法来查看mybatis-redis

储存数据的格式：

```java
        @Override
        public void putObject(final Object key, final Object value) {
            execute(new RedisCallback() {
                @Override
                public Object doWithRedis(Jedis jedis) {
                    jedis.hset(id.toString().getBytes(), key.toString().getBytes(),
                            SerializeUtil.serialize(value));
                    return null;
                }
            });
        }
        @Override
        public Object getObject(final Object key) {
            return execute(new RedisCallback() {

                @Override
                public Object doWithRedis(Jedis jedis) {
                    return SerializeUtil.unserialize(jedis.hget(id.toString().getBytes(),
                            key.toString().getBytes()));
                }
            });
        }
```



可以很清楚的看到，mybatis-redis在存储数据的时候，是使⽤的hash结构，把cache的id作为这个hash

的key (cache的id在mybatis中就是mapper的namespace)；这个mapper中的查询缓存数据作为 hash

的field,需要缓存的内容直接使⽤SerializeUtil存储，SerializeUtil和其他的序列化类差不多，负责 对象

的序列化和反序列化；

> 这段代码内部匿名对象直接访问外部对象的变量，对初学者看来可以有点奇怪。
>
> 这里通过引入一个模板模式，让代码更结构化。
>
> 1. 每个匿名对象代表一类使用redis的操作（put与get代码分开）
> 2. 使用redis的释放资源这类公共操作，封装在execute方法里。

# 第⼋部分：Mybatis插件

## 8.1 插件简介

⼀般情况下，开源框架都会提供插件或其他形式的拓展点，供开发者⾃⾏拓展。这样的好处是显⽽易⻅

的，⼀是增加了框架的灵活性。⼆是开发者可以结合实际需求，对框架进⾏拓展，使其能够更好的⼯

作。以MyBatis为例，我们可基于MyBati s插件机制实现分⻚、分表，监控等功能。由于插件和业务 ⽆

关，业务也⽆法感知插件的存在。因此可以⽆感植⼊插件，在⽆形中增强功能

## 8.2 Mybatis插件介绍

Mybati s作为⼀个应⽤⼴泛的优秀的ORM开源框架，这个框架具有强⼤的灵活性，在四⼤组件

(Executor、StatementHandler、ParameterHandler、ResultSetHandler)处提供了简单易⽤的插 件扩

展机制。Mybatis对持久层的操作就是借助于四⼤核⼼对象。MyBatis⽀持⽤插件对四⼤核⼼对象进 ⾏

拦截，对mybatis来说插件就是拦截器，⽤来增强核⼼对象的功能，增强功能本质上是借助于底层的 动

态代理实现的，换句话说，MyBatis中的四⼤对象都是代理对象
![image-20231116120113883](imgs/image-20231116120113883.png)

**MyBatis所允许拦截的⽅法如下：**

执⾏器Executor (update、query、commit、rollback等⽅法)；

SQL语法构建器StatementHandler (prepare、parameterize、batch、updates query等⽅ 法)；

参数处理器ParameterHandler (getParameterObject、setParameters⽅法)；

结果集处理器ResultSetHandler (handleResultSets、handleOutputParameters等⽅法)；

## 8.3 Mybatis插件原理

在四⼤对象创建的时候

1、每个创建出来的对象不是直接返回的，⽽是interceptorChain.pluginAll(parameterHandler);

2、获取到所有的Interceptor (拦截器)(插件需要实现的接⼝)；调⽤ interceptor.plugin(target);返

回 target 包装后的对象

3、插件机制，我们可以使⽤插件为⽬标对象创建⼀个代理对象；AOP (⾯向切⾯)我们的插件可 以

为四⼤对象创建出代理对象，代理对象就可以拦截到四⼤对象的每⼀个执⾏；

**拦截**
插件具体是如何拦截并附加额外的功能的呢？以ParameterHandler来说

```java
public ParameterHandler newParameterHandler(MappedStatement mappedStatement,
Object object, BoundSql sql, InterceptorChain interceptorChain){
 ParameterHandler parameterHandler =
 
mappedStatement.getLang().createParameterHandler(mappedStatement,object,sql);
 parameterHandler = (ParameterHandler)
 interceptorChain.pluginAll(parameterHandler);
 return parameterHandler;
}
public Object pluginAll(Object target) {
 for (Interceptor interceptor : interceptors) {
  target = interceptor.plugin(target);
 }
 return target;
}
```

interceptorChain保存了所有的拦截器(interceptors)，是mybatis初始化的时候创建的。调⽤拦截器链

中的拦截器依次的对⽬标进⾏拦截或增强。interceptor.plugin(target)中的target就可以理解为mybatis

中的四⼤对象。返回的target是被重重代理后的对象

如果我们想要拦截Executor的query⽅法，那么可以这样定义插件：

```java
@Intercepts({
 @Signature(
  type = Executor.class,
  method = "query",
  args=
{MappedStatement.class,Object.class,RowBounds.class,ResultHandler.class}
  )
}) 
public class ExeunplePlugin implements Interceptor {
 //省略逻辑
}
```

除此之外，我们还需将插件配置到sqlMapConfig.xm l中。

```xml
<plugins>
 <plugin interceptor="com.lagou.plugin.ExamplePlugin">
 </plugin>
</plugins>
```

这样MyBatis在启动时可以加载插件，并保存插件实例到相关对象(InterceptorChain，拦截器链) 中。

待准备⼯作做完后，MyBatis处于就绪状态。我们在执⾏SQL时，需要先通过DefaultSqlSessionFactory

创建 SqlSession。Executor 实例会在创建 SqlSession 的过程中被创建， Executor实例创建完毕后，

MyBatis会通过JDK动态代理为实例⽣成代理类。这样，插件逻辑即可在 Executor相关⽅法被调⽤前执

⾏。

以上就是MyBatis插件机制的基本原理

## 8.4 ⾃定义插件

1. 插件接⼝
   Mybatis 插件接⼝-Interceptor

   • Intercept⽅法，插件的核⼼⽅法

   • plugin⽅法，⽣成target的代理对象

   • setProperties⽅法，传递插件所需参数

2. ⾃定义插件
   设计实现⼀个⾃定义插件

   ```java
   Intercepts ({//注意看这个⼤花括号，也就这说这⾥可以定义多个@Signature对多个地⽅拦截，都⽤
   这个拦截器
   @Signature (type = StatementHandler .class , //这是指拦截哪个接⼝
    method = "prepare"，//这个接⼝内的哪个⽅法名，不要拼错了
    args = { Connection.class, Integer .class}),//// 这是拦截的⽅法的⼊参，按
   顺序写到这，不要多也不要少，如果⽅法重载，可是要通过⽅法名和⼊参来确定唯⼀的
   })
   public class MyPlugin implements Interceptor {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    // //这⾥是每次执⾏操作的时候，都会进⾏这个拦截器的⽅法内
    
    Override
    public Object intercept(Invocation invocation) throws Throwable {
    //增强逻辑
    System.out.println("对⽅法进⾏了增强....")；
    return invocation.proceed(); //执⾏原⽅法
   }
    
    /**
     * //主要是为了把这个拦截器⽣成⼀个代理放到拦截器链中
    * ^Description包装⽬标对象 为⽬标对象创建代理对象
    * @Param target为要拦截的对象
    * @Return代理对象
    */
    Override
    public Object plugin(Object target) {
      System.out.println("将要包装的⽬标对象："+target);
      return Plugin.wrap(target,this);
    }
    
    /**获取配置⽂件的属性**/
    //插件初始化的时候调⽤，也只调⽤⼀次，插件配置的属性从这⾥设置进来
    Override
    public void setProperties(Properties properties) {
      System.out.println("插件配置的初始化参数："+properties );
    }
   }
   ```

   sqlMapConfig.xml

   ```xml
   <plugins>
    <plugin interceptor="com.lagou.plugin.MySqlPagingPlugin">
      <!--配置参数-->
      <property name="name" value="Bob"/>
    </plugin>
   </plugins>
   ```

   mapper接⼝

   ```java
   public interface UserMapper {
    List<User> selectUser();
   }
   ```

   mapper.xml

   ```xml
   <mapper namespace="com.lagou.mapper.UserMapper">
    <select id="selectUser" resultType="com.lagou.pojo.User">
      SELECT
      id,username
      FROM
      user
    </select>
   </mapper>
   ```

   **测试类**

   ```java
   public class PluginTest {
   @Test
   public void test() throws IOException {
    InputStream resourceAsStream =
      Resources.getResourceAsStream("sqlMapConfig.xml");
    SqlSessionFactory sqlSessionFactory = new
      SqlSessionFactoryBuilder().build(resourceAsStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    List<User> byPaging = userMapper.selectUser();
    for (User user : byPaging) {
      System.out.println(user);
     }
    }
   }
   ```

   ## 8.5 源码分析
   
   执⾏插件逻辑
   
   Plugin实现了 InvocationHandler接⼝，因此它的invoke⽅法会拦截所有的⽅法调⽤。invoke⽅法会 对
   
   所拦截的⽅法进⾏检测，以决定是否执⾏插件逻辑。该⽅法的逻辑如下：
   
   ```java
           // -Plugin
           public Object invoke(Object proxy, Method method, Object[] args) throws
                   Throwable {
               try {
                   /*
                    *获取被拦截⽅法列表，⽐如：
                    * signatureMap.get(Executor.class), 可能返回 [query, update,commit]
                    */
               }
               Set<Method> methods =
                       signatureMap.get(method.getDeclaringClass());
               //检测⽅法列表是否包含被拦截的⽅法
               if (methods != null && methods.contains(method)) {
                   //执⾏插件逻辑
                   return interceptor.intercept(new Invocation(target, method,
                           args));
                   //执⾏被拦截的⽅法
                   return method.invoke(target, args);
               } catch(Exception e){
               }
           }
   ```
   
   invoke⽅法的代码⽐较少，逻辑不难理解。⾸先,invoke⽅法会检测被拦截⽅法是否配置在插件的
   
   @Signature注解中，若是，则执⾏插件逻辑，否则执⾏被拦截⽅法。插件逻辑封装在intercept中，该
   
   ⽅法的参数类型为Invocationo Invocation主要⽤于存储⽬标类，⽅法以及⽅法参数列表。下⾯简单看
   
   ⼀下该类的定义
   
   ```java
   public class Invocation {
   private final Object target;
   private final Method method;
   private final Object[] args;
   public Invocation(Object targetf Method method, Object[] args) {
   this.target = target;
   this.method = method;
   //省略部分代码
   public Object proceed() throws InvocationTargetException,
   IllegalAccessException { //调⽤被拦截的⽅法
   >> —
   ```
   
   关于插件的执⾏逻辑就分析结束

## 8.6 pageHelper分⻚插件

MyBati s可以使⽤第三⽅的插件来对功能进⾏扩展，分⻚助⼿PageHelper是将分⻚的复杂操作进⾏封

装，使⽤简单的⽅式即可获得分⻚的相关数据

开发步骤：

① 导⼊通⽤PageHelper的坐标

② 在mybatis核⼼配置⽂件中配置PageHelper插件

③ 测试分⻚数据获取
**导⼊通⽤PageHelper坐标**

```xml
    <dependency>
        <groupId>com.github.pagehelper</groupId>
        <artifactId>pagehelper</artifactId>
        <version>3.7.5</version>
    </dependency>
    <dependency>
        <groupId>com.github.jsqlparser</groupId>
        <artifactId>jsqlparser</artifactId>
        <version>0.9.1</version>
    </dependency>
```

 **在mybatis核⼼配置⽂件中配置PageHelper插件**

```xml
 <!--注意：分⻚助⼿的插件 配置在通⽤馆mapper之前*-->*
 <plugin interceptor="com.github.pagehelper.PageHelper">
   <!—指定⽅⾔ —>
   <property name="dialect" value="mysql"/>
 </plugin>
```

 **测试分⻚代码实现**

```java
 @Test
 public void testPageHelper() {
   //设置分⻚参数
   PageHelper.startPage(1, 2);
   List<User> select = userMapper2.select(null);
   for (User user : select) {
     System.out.println(user);
   }
 }
```

**获得分⻚相关的其他参数**

```java
//其他分⻚的数据
PageInfo<User> pageInfo = new PageInfo<User>(select);
System.out.println("总条数："+pageInfo.getTotal());
System.out.println("总⻚数："+pageInfo. getPages ());
System.out.println("当前⻚："+pageInfo. getPageNum());
System.out.println("每⻚显万⻓度："+pageInfo.getPageSize());
System.out.println("是否第⼀⻚："+pageInfo.isIsFirstPage());
System.out.println("是否最后⼀⻚："+pageInfo.isIsLastPage());
```

## 8.7 通⽤ mapper

**什么是通⽤Mapper**
通⽤Mapper就是为了解决单表增删改查，基于Mybatis的插件机制。开发⼈员不需要编写SQL,不需要在DAO中增加⽅法，只要写好实体类，就能⽀持相应的增删改查⽅法

**如何使⽤**

1. ⾸先在maven项⽬，在pom.xml中引⼊mapper的依赖

   ```xml
   <dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper</artifactId>
    <version>3.1.2</version>
   </dependency>
   ```

2.  Mybatis配置⽂件中完成配置

   ```xml
   <plugins>
    <!--分⻚插件：如果有分⻚插件，要排在通⽤mapper之前-->
    <plugin interceptor="com.github.pagehelper.PageHelper">
      <property name="dialect" value="mysql"/>
    </plugin>
    <plugin interceptor="tk.mybatis.mapper.mapperhelper.MapperInterceptor">
      <!-- 通⽤Mapper接⼝，多个通⽤接⼝⽤逗号隔开 -->
      <property name="mappers" value="tk.mybatis.mapper.common.Mapper"/>
    </plugin>
   </plugins>
   ```

3. 实体类设置主键

   ```java
   @Table(name = "t_user")
   public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    private String username;
   }
   ```

4. 定义通⽤mapper

   ```java
   import com.lagou.domain.User;
   import tk.mybatis.mapper.common.Mapper;
   public interface UserMapper extends Mapper<User> {
   }
   ```

5. 测试

   ```java
           public class UserTest {
               @Test
               public void test1() throws IOException {
                   Inputstream resourceAsStream =
                           Resources.getResourceAsStream("sqlMapConfig.xml");
                   SqlSessionFactory build = new
                           SqlSessionFactoryBuilder().build(resourceAsStream);
                   SqlSession sqlSession = build.openSession();
                   UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
                   User user = new User();
                   user.setId(4);
                   //(1)mapper基础接⼝
                   //select 接⼝
                   User user1 = userMapper.selectOne(user); //根据实体中的属性进⾏查询，只能有—个返回值
                   List<User> users = userMapper.select(null); //查询全部结果
                   userMapper.selectByPrimaryKey(1); //根据主键字段进⾏查询，⽅法参数必须包含完整的主键属性，查询条件使⽤等号
                   userMapper.selectCount(user); //根据实体中的属性查询总数，查询条件使⽤等号
                   // insert 接⼝
                   int insert = userMapper.insert(user); //保存⼀个实体，null值也会保存，不会使⽤数据库默认值
                   int i = userMapper.insertSelective(user); //保存实体，null的属性不会保存，会使⽤数据库默认值
                   // update 接⼝
                   int i1 = userMapper.updateByPrimaryKey(user);//根据主键更新实体全部字段，null值会被更新
                   // delete 接⼝
                   int delete = userMapper.delete(user); //根据实体属性作为条件进⾏删除，查询条件 使⽤等号
                   userMapper.deleteByPrimaryKey(1); //根据主键字段进⾏删除，⽅法参数必须包含完整的主键属性
                   //(2)example⽅法
                   Example example = new Example(User.class);
                   example.createCriteria().andEqualTo("id", 1);
                   example.createCriteria().andLike("val", "1");
                   //⾃定义查询
                   List<User> users1 = userMapper.selectByExample(example);
               }
           }
   ```

   # 第九部分：Mybatis架构原理
   
   ## 9.1架构设计
   
   ![image-20231116164135251](imgs/image-20231116164135251.png)

我们把Mybatis的功能架构分为三层：

(1) API接⼝层：提供给外部使⽤的接⼝ API，开发⼈员通过这些本地API来操纵数据库。接⼝层⼀接收

到 调⽤请求就会调⽤数据处理层来完成具体的数据处理。

MyBatis和数据库的交互有两种⽅式：

a. 使⽤传统的MyBati s提供的API ；

b. 使⽤Mapper代理的⽅式

(2) 数据处理层：负责具体的SQL查找、SQL解析、SQL执⾏和执⾏结果映射处理等。它主要的⽬的是根

据调⽤的请求完成⼀次数据库操作。

(3) 基础⽀撑层：负责最基础的功能⽀撑，包括连接管理、事务管理、配置加载和缓存处理，这些都是

共 ⽤的东⻄，将他们抽取出来作为最基础的组件。为上层的数据处理层提供最基础的⽀撑

## 9.2主要构件及其相互关系

| 构件             | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| SqlSession       | 作为MyBatis⼯作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能 |
| Executor         | MyBatis执⾏器，是MyBatis调度的核⼼，负责SQL语句的⽣成和查询缓存的维护 |
| StatementHandler | 封装了JDBC Statement操作，负责对JDBC statement的操作，如设置参数、将Statement结果集转换成List集合。 |
| ParameterHandler | 负责对⽤户传递的参数转换成JDBC Statement所需要的参数，       |
| ResultSetHandler | 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；    |
| TypeHandler      | 负责java数据类型和jdbc数据类型之间的映射和转换               |
| MappedStatement  | MappedStatement维护了⼀条＜select \| update \| delete \| insert＞节点的封 装 |
| SqlSource        | 负责根据⽤户传递的parameterObject，动态地⽣成SQL语句，将信息封装到BoundSql对象中，并返回 |
| BoundSql         | 表示动态⽣成的SQL语句以及相应的参数信息                      |

![image-20231116170503286](imgs/image-20231116170503286.png)

## 9.3总体流程

1. **加载配置并初始化**
   **触发条件：**加载配置⽂件
   配置来源于两个地⽅，⼀个是配置⽂件(主配置⽂件conf.xml,mapper⽂件*.xml),—个是java代码中的

   注解，将主配置⽂件内容解析封装到Configuration,将sql的配置信息加载成为⼀个mappedstatement

   对象，存储在内存之中

2. **接收调⽤请求**
   **触发条件**：调⽤Mybatis提供的API

   **传⼊参数：**为SQL的ID和传⼊参数对象

   **处理过程：**将请求传递给下层的请求处理层进⾏处理。

3. **处理操作请求**
   **触发条件：**API接⼝层传递请求过来

   **传⼊参数：**为SQL的ID和传⼊参数对象

   **处理过程：**
   (A) 根据SQL的ID查找对应的MappedStatement对象。

   (B) 根据传⼊参数对象解析MappedStatement对象，得到最终要执⾏的SQL和执⾏传⼊参数。

   (C) 获取数据库连接，根据得到的最终SQL语句和执⾏传⼊参数到数据库执⾏，并得到执⾏结果。

   (D) 根据MappedStatement对象中的结果映射配置对得到的执⾏结果进⾏转换处理，并得到最终的处

   理 结果。

   (E) 释放连接资源。

4. **返回处理结果**
   将最终的处理结果返回。

# 第⼗部分：Mybatis源码剖析

## 10.1传统⽅式源码剖析：

**源码剖析-初始化**

```java
Inputstream inputstream = Resources.getResourceAsStream("mybatisconfig.xml");
   //这⼀⾏代码正是初始化⼯作的开始。
   SqlSessionFactory factory = new
SqlSessionFactoryBuilder().build(inputStream);
```

进⼊源码分析：

```java
 public SqlSessionFactory build(InputStream inputStream) {
     //调⽤了重载⽅法
     return build(inputStream, null, null);
 }
 // 2.调⽤的重载⽅法
 public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
     try {
         // XMLConfigBuilder是专⻔解析mybatis的配置⽂件的类
         XMLConfigBuilder parser = new XMLConfigBuilder(inputstream,
                 environment, properties);
         //这⾥⼜调⽤了⼀个重载⽅法。parser.parse()的返回值是Configuration对象
         return build(parser.parse());
     } catch (Exception e) {
         throw ExceptionFactory.wrapException("Error building
                 SqlSession.", e)
     }
 }
```

MyBatis在初始化的时候，会将MyBatis的配置信息全部加载到内存中，使⽤

org.apache.ibatis.session.Configuration 实例来维护

下⾯进⼊对配置⽂件解析部分：

⾸先对Configuration对象进⾏介绍:

```
Configuration对象的结构和xml配置⽂件的对象⼏乎相同。
回顾⼀下xml中的配置标签有哪些：
properties (属性)，settings (设置)，typeAliases (类型别名)，typeHandlers (类型处理
器)，objectFactory (对象⼯⼚)，mappers (映射器)等 Configuration也有对应的对象属性来封
装它们
也就是说，初始化配置⽂件信息的本质就是创建Configuration对象，将解析的xml数据封装到
Configuration内部属性中
```

```java
/**
 * 解析 XML 成 Configuration 对象。
 */
public Configuration parse () {
    //若已解析，抛出BuilderException异常
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be
                used once.");
    }
    //标记已解析
    parsed = true;
    // 解析 XML configuration 节点
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
/**
 *解析XML
 */
private void parseConfiguration (XNode root){
    try {
        //issue #117 read properties first
        // 解析 <properties /> 标签
        propertiesElement(root.evalNode("properties"));
        // 解析〈settings /> 标签
        Properties settings =
                settingsAsProperties(root.evalNode("settings"));
        //加载⾃定义的VFS实现类
        loadCustomVfs(settings);
        // 解析 <typeAliases /> 标签
        typeAliasesElement(root.evalNode("typeAliases"));
        //解析<plugins />标签
        pluginElement(root.evalNode("plugins"));
        // 解析 <objectFactory /> 标签
        objectFactoryElement(root.evalNode("objectFactory"));
        // 解析 <objectWrapperFactory /> 标签
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        // 解析 <reflectorFactory /> 标签
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        // 赋值 <settings /> ⾄ Configuration 属性
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue
        // #631
        // 解析〈environments /> 标签
        environmentsElement(root.evalNode("environments"));
        // 解析 <databaseIdProvider /> 标签
        databaseldProviderElement(root.evalNode("databaseldProvider"));
        // 解析 <typeHandlers /> 标签
        typeHandlerElement(root.evalNode("typeHandlers"));
        //解析<mappers />标签
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper
                Configuration.Cause:" + e, e);
    }
}
```

介绍⼀下 MappedStatement ：

作⽤：MappedStatement与Mapper配置⽂件中的⼀个select/update/insert/delete节点相对应。

mapper中配置的标签都被封装到了此对象中，主要⽤途是描述⼀条SQL语句。

**初始化过程：**回顾刚开 始介绍的加载配置⽂件的过程中，会对mybatis-config.xm l中的各个标签都进⾏

解析，其中有mappers 标签⽤来引⼊mapper.xml⽂件或者配置mapper接⼝的⽬录。

```xml
<select id="getUser" resultType="user" >
 select * from user where id=#{id}
</select>
```

这样的⼀个select标签会在初始化配置⽂件时被解析封装成⼀个MappedStatement对象，然后存储在

Configuration对象的mappedStatements属性中，mappedStatements 是⼀个HashMap，存储时key

=全限定类名+⽅法名，value =对应的MappedStatement对象。

在configuration中对应的属性为

```java
Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>
("Mapped Statements collection")
```

在 XMLConfigBuilder 中的处理：

```java
private void parseConfiguration(XNode root) {
 try {
     //省略其他标签的处理
     mapperElement(root.evalNode("mappers"));
   } catch (Exception e) {
     throw new BuilderException("Error parsing SQL Mapper Configuration.Cause:" + e, e);
   }
 }
```

到此对xml配置⽂件的解析就结束了，回到步骤2.中调⽤的重载build⽅法

```java
// 5.调⽤的重载⽅法
public SqlSessionFactory build(Configuration config) {
 //创建了 DefaultSqlSessionFactory 对象，传⼊ Configuration 对象。
 return new DefaultSqlSessionFactory(config);
}
```

**源码剖析-执⾏SQL流程**

先简单介绍**SqlSession** **：**

SqlSession是⼀个接⼝，它有两个实现类：DefaultSqlSession (默认)和

SqlSessionManager (弃⽤，不做介绍)

SqlSession是MyBatis中⽤于和数据库交互的顶层类，通常将它与ThreadLocal绑定，⼀个会话使⽤⼀

个SqlSession,并且在使⽤完毕后需要close

```java
public class DefaultSqlSession implements SqlSession {
private final Configuration configuration;
private final Executor executor;
j
```

SqlSession中的两个最重要的参数，configuration与初始化时的相同，Executor为执⾏器
**Executor:**Executor也是⼀个接⼝，他有三个常⽤的实现类：

BatchExecutor (重⽤语句并执⾏批量更新)

ReuseExecutor (重⽤预处理语句 prepared statements)

SimpleExecutor (普通的执⾏器，默认)

继续分析，初始化完毕后，我们就要执⾏SQL 了

```java
SqlSession sqlSession = factory.openSession();
List<User> list =
sqlSession.selectList("com.lagou.mapper.UserMapper.getUserByName");
```

获得 sqlSession

```java
//6. 进⼊ openSession ⽅法。
public SqlSession openSession() {
    //getDefaultExecutorType()传递的是SimpleExecutor
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null,
                    false);
}
//7. 进⼊penSessionFromDataSource。
//ExecutorType 为Executor的类型，TransactionIsolationLevel为事务隔离级别，autoCommit是否开启事务
//openSession的多个重载⽅法可以指定获得的SeqSession的Executor类型和事务的处理
private SqlSession openSessionFromDataSource(ExecutorType execType,
                                             TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try{
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory =
                getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(),
                level, autoCommit);
        //根据参数创建指定类型的Executor
        final Executor executor = configuration.newExecutor(tx, execType);
        //返回的是 DefaultSqlSession
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch(Exception e){
        closeTransaction(tx); // may have fetched a connection so lets call close()
    }
}
```

执⾏ sqlsession 中的 api

```java
//8.进⼊selectList⽅法，多个重载⽅法。
public <E > List < E > selectList(String statement) {
    return this.selectList(statement, null);
    public <E > List < E > selectList(String statement, Object parameter)
    {
        return this.selectList(statement, parameter, RowBounds.DEFAULT);
        public <E > List < E > selectList(String statement, Object
            parameter, RowBounds rowBounds) {
        try {
            //根据传⼊的全限定名+⽅法名从映射的Map中取出MappedStatement对象
            MappedStatement ms =
                    configuration.getMappedStatement(statement);
            //调⽤Executor中的⽅法处理
            //RowBounds是⽤来逻辑分⻚
            // wrapCollection(parameter)是⽤来装饰集合或者数组参数
            return executor.query(ms, wrapCollection(parameter),
                    rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error querying
                    database. Cause: + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
```

**源码剖析-executor**

继续源码中的步骤，进⼊executor.query()

```java
//此⽅法在SimpleExecutor的⽗类BaseExecutor中实现
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds
        rowBounds, ResultHandler resultHandler) throws SQLException {
    //根据传⼊的参数动态获得SQL语句，最后返回⽤BoundSql对象表示
    BoundSql boundSql = ms.getBoundSql(parameter);
    //为本次查询创建缓存的Key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}

//进⼊query的重载⽅法中

public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundS
        throws SQLException {
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
        list = resultHandler == null ? (List<E>) localCache.getObject(key)
                : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter,
                    boundSql);
        } else {
            //如果缓存中没有本次查找的值，那么从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds,
                    resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() ==
                LocalCacheScope.STATEMENT) { // issue #482 clearLocalCache();
        }
    }
    return list;
}

//从数据库查询
private <E> List<E> queryFromDatabase(MappedStatement ms, Object
        parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key,
                                      BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        //查询的⽅法
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    //将查询结果放⼊缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
                         
// SimpleExecutor中实现⽗类的doQuery抽象⽅法
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds
        rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException
{
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        //传⼊参数创建StatementHanlder对象来执⾏查询
        StatementHandler handler =
                configuration.newStatementHandler(wrapper, ms, parameter, rowBounds,
                        resultHandler, boundSql);
        //创建jdbc中的statement对象
        stmt = prepareStatement(handler, ms.getStatementLog());
        // StatementHandler 进⾏处理
        return handler.query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}                        

//创建Statement的⽅法
private Statement prepareStatement(StatementHandler handler, Log
        statementLog) throws SQLException {
    Statement stmt;
    //条代码中的getConnection⽅法经过重重调⽤最后会调⽤openConnection⽅法，从连接池中获得连接。
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}
                         
//从连接池获得连接的⽅法
protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
        log.debug("Opening JDBC Connection");
    }
    //从连接池获得连接
    connection = dataSource.getConnection();
    if (level != null) {
        connection.setTransactionIsolation(level.getLevel());
    }
}
                         
                 

```

上述的Executor.query()⽅法⼏经转折，最后会创建⼀个StatementHandler对象，然后将必要的参数传

递给StatementHandler，使⽤StatementHandler来完成对数据库的查询，最终返回List结果集。

从上⾯的代码中我们可以看出，Executor的功能和作⽤是：

```
(1、根据传递的参数，完成SQL语句的动态解析，⽣成BoundSql对象，供StatementHandler使⽤；
(2、为查询创建缓存，以提⾼性能
(3、创建JDBC的Statement连接对象，传递给*StatementHandler*对象，返回List查询结果。
```

**源码剖析-StatementHandler**

StatementHandler对象主要完成两个⼯作：

- 对于JDBC的PreparedStatement类型的对象，创建的过程中，我们使⽤的是SQL语句字符串会包

  含若⼲个？占位符，我们其后再对占位符进⾏设值。StatementHandler通过

  parameterize(statement)⽅法对 S tatement 进⾏设值；

- StatementHandler 通过 List query(Statement statement, ResultHandler resultHandler)⽅法来

  完成执⾏Statement，和将Statement对象返回的resultSet封装成List；

进⼊到 StatementHandler 的 parameterize(statement)⽅法的实现：

```java
public void parameterize(Statement statement) throws SQLException {
 //使⽤ParameterHandler对象来完成对Statement的设值
 parameterHandler.setParameters((PreparedStatement) statement);
}
```

```java
/**
 * ParameterHandler 类的 setParameters(PreparedStatement ps) 实现
 * 对某⼀个Statement进⾏设置参数
 */
public void setParameters(PreparedStatement ps) throws SQLException {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
        for (int i = 0; i <
                parameterMappings.size(); i++) {
            ParameterMapping parameterMapping =
                    parameterMappings.get(i);
            if (parameterMapping.getMode() != ParameterMode.OUT) {
                Object value;
                String propertyName = parameterMapping.getProperty();
                if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448
                    ask first for additional params
                    value = boundSql.getAdditionalParameter(propertyName);
                } else if (parameterObject == null) {
                    value = null;
                } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                    value = parameterObject;
                } else {
                    MetaObject metaObject = configuration.newMetaObject(parameterObject);
                    value = metaObject.getValue(propertyName);
                }
                // 每⼀个 Mapping都有⼀个 TypeHandler，根据 TypeHandler 来对
                preparedStatement 进 ⾏设置参数
                TypeHandler typeHandler = parameterMapping.getTypeHandler();
                JdbcType jdbcType = parameterMapping.getJdbcType();
                if (value == null && jdbcType == null) jdbcType =
                        configuration.getJdbcTypeForNull();
                //设置参数
                typeHandler.setParameter(ps, i + 1, value, jdbcType);
            }
        }
    }
}
```

从上述的代码可以看到,StatementHandler的parameterize(Statement)⽅法调⽤了

ParameterHandler的setParameters(statement)⽅法，

ParameterHandler的setParameters(Statement )⽅法负责根据我们输⼊的参数，对statement对象的

?占位符处进⾏赋值。

进⼊到StatementHandler 的 List query(Statement statement, ResultHandler resultHandler)⽅法的

实现：

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler)
        throws SQLException {
    // 1.调⽤preparedStatemnt。execute()⽅法，然后将resultSet交给ResultSetHandler处
    理
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    //2.使⽤ ResultHandler 来处理 ResultSet
    return resultSetHandler.<E> handleResultSets(ps);
}
```

从上述代码我们可以看出，StatementHandler 的List query(Statement statement, ResultHandler

resultHandler)⽅法的实现，是调⽤了 ResultSetHandler 的 handleResultSets(Statement)⽅法。

ResultSetHandler 的 handleResultSets(Statement)⽅法会将 Statement 语句执⾏后⽣成的 resultSet

结 果集转换成List结果集

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
    //多ResultSet的结果集合，每个ResultSet对应⼀个Object对象。⽽实际上，每个Object是List<Object>对象。
    // 在不考虑存储过程的多ResultSet的情况，普通的查询，实际就⼀个ResultSet，也 就是说，multipleResults最多就⼀个元素。
    final List<Object> multipleResults = new ArrayList<>();
    int resultSetCount = 0;
    //获得⾸个ResultSet对象，并封装成ResultSetWrapper对象
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    //获得ResultMap数组
    //在不考虑存储过程的多ResultSet的情况，普通的查询，实际就⼀个ResultSet，也 就是
    说，resultMaps就⼀个元素。
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount); // 校验
    while (rsw != null && resultMapCount > resultSetCount) {
        //获得ResultMap对象
        ResultMap resultMap = resultMaps.get(resultSetCount);
        //处理ResultSet，将结果添加到multipleResults中
        handleResultSet(rsw, resultMap, multipleResults, null);
        //获得下⼀个ResultSet对象，并封装成ResultSetWrapper对象
        rsw = getNextResultSet(stmt);
        //清理
        cleanUpAfterHandlingResultSet();
        // resultSetCount ++
        resultSetCount++;
    }
}
//因为'mappedStatement.resultSets'只在存储过程中使⽤，本系列暂时不考虑，忽略即可
String[] resultSets = mappedStatement.getResultSets();
if(resultSets!=null)
{
    while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping =
                nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
            String nestedResultMapId =
                    parentMapping.getNestedResultMapId();
            ResultMap resultMap =
                    configuration.getResultMap(nestedResultMapId);
            handleResultSet(rsw, resultMap, null, parentMapping);
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }
    }
    //如果是multipleResults单元素，则取⾸元素返回
    return collapseSingleResultList(multipleResults);
}
```

## 10.2 Mapper代理⽅式：

回顾下写法:

```java
public static void main(String[] args) {
    //前三步都相同
    InputStream inputStream =
            Resources.getResourceAsStream("sqlMapConfig.xml");
    SqlSessionFactory factory = new
            SqlSessionFactoryBuilder().build(inputStream);
    SqlSession sqlSession = factory.openSession();
    //这⾥不再调⽤SqlSession的api,⽽是获得了接⼝对象，调⽤接⼝中的⽅法。
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    List<User> list = mapper.getUserByName("tom");
}
```

思考⼀个问题，通常的Mapper接⼝我们都没有实现的⽅法却可以使⽤，是为什么呢？答案很简单动态

代理。

开始之前，介绍⼀下MyBatis初始化时对接⼝的处理：MapperRegistry是Configuration中的⼀个属性，

它内部维护⼀个HashMap⽤于存放mapper接⼝的⼯⼚类，每个接⼝对应⼀个⼯⼚类。mappers中可以

配置接⼝的包路径，或者某个具体的接⼝类。

```xml
<mappers>
 <mapper class="com.lagou.mapper.UserMapper"/>
 <package name="com.lagou.mapper"/>
</mappers>
```

当解析mappers标签时，它会判断解析到的是mapper配置⽂件时，会再将对应配置⽂件中的增删 改

查标签 封装成MappedStatement对象，存⼊mappedStatements中。(上⽂介绍了)当

判断解析到接⼝时，会建此接⼝对应的MapperProxyFactory对象，存⼊HashMap中，key =接⼝的字节码对象，value =此接⼝对应的MapperProxyFactory对象。

**源码剖析-getmapper()**

进⼊ sqlSession.getMapper(UserMapper.class )中

```java
//DefaultSqlSession 中的 getMapper
public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
}
//configuration 中的给 g etMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}
//MapperRegistry 中的 g etMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //从 MapperRegistry 中的 HashMap 中拿 MapperProxyFactory
    final MapperProxyFactory<T> mapperProxyFactory =
            (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the
                MapperRegistry.");
    }
    try {
        //通过动态代理⼯⼚⽣成示例。
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause:
                " + e, e);
    }
}
//MapperProxyFactory 类中的 newInstance ⽅法
public T newInstance(SqlSession sqlSession) {
    //创建了 JDK动态代理的Handler类
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession,
            mapperInterface, methodCache);
    //调⽤了重载⽅法
    return newInstance(mapperProxy);
}
//MapperProxy 类，实现了 InvocationHandler 接⼝
public class MapperProxy<T> implements InvocationHandler, Serializable {
    //省略部分源码
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache;
    //构造，传⼊了 SqlSession，说明每个session中的代理对象的不同的！
    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface,
                       Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }
    //省略部分源码
}
```

**源码剖析-invoke()**

在动态代理返回了示例后，我们就可以直接调⽤mapper类中的⽅法了，但代理对象调⽤⽅法，执⾏是

在MapperProxy中的invoke⽅法中

```java
public Object invoke(Object proxy, Method method, Object[] args) throws
        Throwable {
    try {
        //如果是Object定义的⽅法，直接调⽤
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else if (isDefaultMethod(method)) {
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    // 获得 MapperMethod 对象
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //重点在这：MapperMethod最终调⽤了执⾏的⽅法
    return mapperMethod.execute(sqlSession, args);
}
```

进⼊execute⽅法：

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    //判断mapper中的⽅法类型，最终调⽤的还是SqlSession中的⽅法 
    switch (command.getType()) {
        case INSERT: {
            //转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            //执⾏INSERT操作
            // 转换 rowCount
            result = rowCountResult(sqlSession.insert(command.getName(),
                    param));
            break;
        }
        case UPDATE: {
            //转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 转换 rowCount
            result = rowCountResult(sqlSession.update(command.getName(),
                    param));
            break;
        }
        case DELETE: {
            //转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 转换 rowCount
            result = rowCountResult(sqlSession.delete(command.getName(),
                    param));
            break;
        }
        case SELECT:
            //⽆返回，并且有ResultHandler⽅法参数，则将查询的结果，提交给 ResultHandler 进⾏处理
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
                //执⾏查询，返回列表
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
                //执⾏查询，返回Map
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
                //执⾏查询，返回Cursor
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
                //执⾏查询，返回单个对象
            } else {
                //转换参数
                Object param = method.convertArgsToSqlCommandParam(args);
                //查询单条
                result = sqlSession.selectOne(command.getName(), param);
                if (method.returnsOptional() &&
                        (result == null ||
                                !method.getReturnType().equals(result.getClass()))) {
                    result = Optional.ofNullable(result);
                }
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " +
                    command.getName());
    }
    //返回结果为null，并且返回类型为基本类型，则抛出BindingException异常
    if (result == null && method.getReturnType().isPrimitive()
            && !method.returnsVoid()) {
        throw new BindingException("Mapper method '" + command.getName() + "
                attempted to return null from a method with a primitive
        return type(" + method.getReturnType() + "). ");
    }
    //返回结果
    return result;
}
```

> 源码分析资料  https://mp.weixin.qq.com/s/DN7WKHhNAB861vx-cGX1gw

> 两种Mybatis使用方式对比，传统方式，动态代理Mapper类方式
>
> https://blog.csdn.net/qq_43702747/article/details/120356359
>
> https://blog.csdn.net/Self_discipline8/article/details/115792901?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-115792901-blog-120356359.235%5Ev38%5Epc_relevant_anti_t3_base&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-115792901-blog-120356359.235%5Ev38%5Epc_relevant_anti_t3_base&utm_relevant_index=2
>
> 如果我们添加Mapper映射文件时，遵循一定的规则（namespace是接口名全路径，id是方法名），就可以激活动态代理模式。当然激活动态代理后，仍可以手动编写接口实现类，按传统方式使用（麻烦，一般都用动态代理方式）。

## 10.3 ⼆级缓存源码剖析：

⼆级缓存构建在⼀级缓存之上，在收到查询请求时，MyBatis ⾸先会查询⼆级缓存，若⼆级缓存未命

中，再去查询⼀级缓存，⼀级缓存没有，再查询数据库。

⼆级缓存------》 ⼀级缓存------》数据库

与⼀级缓存不同，⼆级缓存和具体的命名空间绑定，⼀个Mapper中有⼀个Cache，相同Mapper中的

MappedStatement共⽤⼀个Cache，⼀级缓存则是和 SqlSession 绑定。

**启⽤⼆级缓存**

1. 开启全局⼆级缓存配置：

   ```xml
   <settings>
    <setting name="cacheEnabled" value="true"/>
   </settings>
   ```

2.  在需要使⽤⼆级缓存的Mapper配置⽂件中配置标签

   ```xml
    <cache></cache>
   ```

3. 在具体CURD标签上配置 **useCache=true**

   ```xml
    <select id="findById" resultType="com.lagou.pojo.User" useCache="true">
      select * from user where id = #{id}
    </select>
   ```

**标签** **< cache/>** **的解析**
根据之前的mybatis源码剖析，xml的解析⼯作主要交给XMLConfigBuilder.parse()⽅法来实现

```java
 // XMLConfigBuilder.parse()
 public Configuration parse() {
     if (parsed) {
         throw new BuilderException("Each XMLConfigBuilder can only be used
                 once.");
     }
     parsed = true;
     parseConfiguration(parser.evalNode("/configuration"));// 在这⾥
     return configuration;
 }
                                    
// parseConfiguration()
// 既然是在xml中添加的，那么我们就直接看关于mappers标签的解析
private void parseConfiguration(XNode root) {
    try {
        Properties settings =
                settingsAsPropertiess(root.evalNode("settings"));
        propertiesElement(root.evalNode("properties"));
        loadCustomVfs(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectionFactoryElement(root.evalNode("reflectionFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        // 就是这⾥
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration.
                Cause: " + e, e);
    }
}
                                   
// mapperElement()
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // 按照我们本例的配置，则直接⾛该if判断
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream =
                            Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new
                            XMLMapperBuilder(inputStream, configuration, resource,
                            configuration.getSqlFragments());
                    // ⽣成XMLMapperBuilder，并执⾏其parse⽅法
                    mapperParser.parse();
                } else if (resource == null && url != null && mapperClass ==
                        null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new
                            XMLMapperBuilder(inputStream, configuration, url,
                            configuration.getSqlFragments());
                    mapperParser.parse();
                } else if (resource == null && url == null && mapperClass !=
                        null) {
                    Class<?> mapperInterface =
                            Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException("A mapper element may only
                            specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```

我们来看看解析Mapper.xml

```java
// XMLMapperBuilder.parse()
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
        // 解析mapper属性
        configurationElement(parser.evalNode("/mapper"));
        configuration.addLoadedResource(resource);
        bindMapperForNamespace();
    }
    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
}

// configurationElement()
private void configurationElement(XNode context) {
    try {
        String namespace = context.getStringAttribute("namespace");
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        builderAssistant.setCurrentNamespace(namespace);
        cacheRefElement(context.evalNode("cache-ref"));
        // 最终在这⾥看到了关于cache属性的处理
        cacheElement(context.evalNode("cache"));
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        sqlElement(context.evalNodes("/mapper/sql"));
        // 这⾥会将⽣成的Cache包装到对应的MappedStatement
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. Cause: " + e,
                e);
    }
}

 // cacheElement()
 private void cacheElement(XNode context) throws Exception {
     if (context != null) {
         //解析<cache/>标签的type属性，这⾥我们可以⾃定义cache的实现类，⽐如redisCache，
         如果没有⾃定义，这⾥使⽤和⼀级缓存相同的PERPETUAL
         String type = context.getStringAttribute("type", "PERPETUAL");
         Class<? extends Cache> typeClass =
                 typeAliasRegistry.resolveAlias(type);
         String eviction = context.getStringAttribute("eviction", "LRU");
         Class<? extends Cache> evictionClass =
                 typeAliasRegistry.resolveAlias(eviction);
         Long flushInterval = context.getLongAttribute("flushInterval");
         Integer size = context.getIntAttribute("size");
         boolean readWrite = !context.getBooleanAttribute("readOnly", false);
         boolean blocking = context.getBooleanAttribute("blocking", false);
         Properties props = context.getChildrenAsProperties();
         // 构建Cache对象
         builderAssistant.useNewCache(typeClass, evictionClass, flushInterval,
                 size, readWrite, blocking, props);
     }
 }
```

先来看看是如何构建Cache对象的

**MapperBuilderAssistant.useNewCache()**

```java
public Cache useNewCache(Class<? extends Cache> typeClass,
                         Class<? extends Cache> evictionClass,
                         Long flushInterval,
                         Integer size,
                         boolean readWrite,
                         boolean blocking,
                         Properties props) {
    // 1.⽣成Cache对象
    Cache cache = new CacheBuilder(currentNamespace)
    //这⾥如果我们定义了<cache/>中的type，就使⽤⾃定义的Cache,否则使⽤和⼀级缓存相同的PerpetualCache
            .implementation(valueOrDefault(typeClass, PerpetualCache.class))
            .addDecorator(valueOrDefault(evictionClass, LruCache.class))
            .clearInterval(flushInterval)
            .size(size)
            .readWrite(readWrite)
            .blocking(blocking)
            .properties(props)
            .build();
    // 2.添加到Configuration中
    configuration.addCache(cache);
    // 3.并将cache赋值给MapperBuilderAssistant.currentCache
    currentCache = cache;
    return cache;
}
```

我们看到⼀个Mapper.xml只会解析⼀次标签，也就是只创建⼀次Cache对象，放进configuration中，

并将cache赋值给MapperBuilderAssistant.currentCache

**buildStatementFromContext(context.evalNodes("select|insert|update|delete"));将Cache**
**包装到MappedStatement**

```java
 // buildStatementFromContext()
 private void buildStatementFromContext(List<XNode> list) {
     if (configuration.getDatabaseId() != null) {
         buildStatementFromContext(list, configuration.getDatabaseId());
     }
     buildStatementFromContext(list, null);
 }
 //buildStatementFromContext()
 private void buildStatementFromContext(List<XNode> list, String
         requiredDatabaseId) {
     for (XNode context : list) {
         final XMLStatementBuilder statementParser = new
                 XMLStatementBuilder(configuration, builderAssistant, context,
                 requiredDatabaseId);
         try {
             // 每⼀条执⾏语句转换成⼀个MappedStatement
             statementParser.parseStatementNode();
         } catch (IncompleteElementException e) {
             configuration.addIncompleteStatement(statementParser);
         }
     }
 }

// XMLStatementBuilder.parseStatementNode();
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");
    ...
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);
    ...
    // 创建MappedStatement对象
    builderAssistant.addMappedStatement(id, sqlSource, statementType,
            sqlCommandType,
            fetchSize, timeout, parameterMap,
            parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache,
            useCache, resultOrdered,
            keyGenerator, keyProperty, keyColumn,
            databaseId, langDriver, resultSets);
}

 // builderAssistant.addMappedStatement()
 public MappedStatement addMappedStatement(
         String id,
           ...) {
     if (unresolvedCacheRef) {
         throw new IncompleteElementException("Cache-ref not yet resolved");
     }
     id = applyCurrentNamespace(id, false);
     boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
     //创建MappedStatement对象
     MappedStatement.Builder statementBuilder = new
             MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
     ...
     .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
             .useCache(valueOrDefault(useCache, isSelect))
             .cache(currentCache);// 在这⾥将之前⽣成的Cache封装到MappedStatement
     ParameterMap statementParameterMap =
             getStatementParameterMap(parameterMap, parameterType, id);
     if (statementParameterMap != null) {
         statementBuilder.parameterMap(statementParameterMap);
     }
     MappedStatement statement = statementBuilder.build();
     configuration.addMappedStatement(statement);
     return statement;
 }
```

我们看到将Mapper中创建的Cache对象，加⼊到了每个MappedStatement对象中，也就是同⼀个

Mapper中所有的2

有关于标签的解析就到这了。

**查询源码分析**

**CachingExecutor**

```java
// CachingExecutor
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds
        rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 创建 CacheKey
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key,
            boundSql);
}
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds
        rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
        throws SQLException {
    // 从 MappedStatement 中获取 Cache，注意这⾥的 Cache 是从MappedStatement中获取的
    // 也就是我们上⾯解析Mapper中<cache/>标签中创建的，它保存在Configration中
    // 我们在上⾯解析blog.xml时分析过每⼀个MappedStatement都有⼀个Cache对象，就是这⾥
    Cache cache = ms.getCache();
    // 如果配置⽂件中没有配置 <cache>，则 cache 为空
    if (cache != null) {
        //如果需要刷新缓存的话就刷新：flushCache="true"
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            // 访问⼆级缓存
            List<E> list = (List<E>) tcm.getObject(cache, key);
            // 缓存未命中
            if (list == null) {
                // 如果没有值，则执⾏查询，这个查询实际也是先⾛⼀级缓存查询，⼀级缓存也没有的话，则进⾏DB查询
                        list = delegate.<E>query(ms, parameterObject, rowBounds,
                        resultHandler, key, boundSql);
                // 缓存查询结果
                tcm.putObject(cache, key, list);
            }
            return list;
        }
    }
    return delegate.<E>query(ms, parameterObject, rowBounds, resultHandler,
            key, boundSql);
}
```

**如果设置了flushCache="true"，则每次查询都会刷新缓存**

```xml
<!-- 执⾏此语句清空缓存 -->
<select id="findbyId" resultType="com.lagou.pojo.user" useCache="true"
flushCache="true" >
    select * from t_demo
</select>
```



如上，注意⼆级缓存是从 MappedStatement 中获取的。由于 MappedStatement 存在于全局配置

中，可以被多个 CachingExecutor 获取到，这样就会出现线程安全问题。除此之外，若不加以控制，多个

事务共⽤⼀个缓存实例，会导致脏读问题。⾄于脏读问题，需要借助其他类来处理，也就是上⾯代码中

tcm 变量对应的类型。下⾯分析⼀下。





> 一二级缓存的问题 https://mp.weixin.qq.com/s/K6v6NeWriZ8TgJmMosRjOQ

> Mybatis的二级缓存为了避免未提交的事务导致脏读，在调用Mybatis的commit方法（注意不是jdbc的commit）时，才会讲缓存数据放到真正的二级缓存中，未调用前，只存在每个事务自己的二级缓存中。
>
> https://blog.csdn.net/qq_51274370/article/details/122644057

> 二级缓存的开启（自带二级缓存）： **!!!因为二级缓存是事务性查询缓存机制 每个会话需要关闭或提交后 才会把查询结果放在二级缓存中**
>
> **注意！！！如果在Sqlsession中是自动提交需要将会话关闭后查询结果才会放在缓存中**



> 事务性预缓存要解决的问题：
>
> 1. 脏读情况1，避免一个事务缓存自己修改的数据（先改数据，又查询这个数据），然后回滚。导致其他事务在缓存读到脏数据。
> 2. 脏读情况2，还有就是多个mapper文件使用不同的命名空间（意味着多个二级缓存），但都引用了一个表，这样修改数据后，有一个二级缓存的数据没有及时清理，导致脏读。解决方法：多个mapper使用同一个命名空间（同一个二级缓存）
> 3. 避免删除有效二级缓存数据，避免修改操作事务最后回滚了，还是清空了二级缓存
>
> https://mp.weixin.qq.com/s/REwezs5Zk1MGH6SaWh_qsg
>
> https://blog.csdn.net/yangxiaofei_java/article/details/111144248

**TransactionalCacheManager**

```java
/** 事务缓存管理器 */
public class TransactionalCacheManager {
    // Cache 与 TransactionalCache 的映射关系表
    private final Map<Cache, TransactionalCache> transactionalCaches = new
            HashMap<Cache, TransactionalCache>();
    public void clear(Cache cache) {
        // 获取 TransactionalCache 对象，并调⽤该对象的 clear ⽅法，下同
        getTransactionalCache(cache).clear();
    }
    public Object getObject(Cache cache, CacheKey key) {
        // 直接从TransactionalCache中获取缓存
        return getTransactionalCache(cache).getObject(key);
    }
    public void putObject(Cache cache, CacheKey key, Object value) {
        // 直接存⼊TransactionalCache的缓存中
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
    private TransactionalCache getTransactionalCache(Cache cache) {
        // 从映射表中获取 TransactionalCache
        TransactionalCache txCache = transactionalCaches.get(cache);
        if (txCache == null) {
            // TransactionalCache 也是⼀种装饰类，为 Cache 增加事务功能
            // 创建⼀个新的TransactionalCache，并将真正的Cache对象存进去
            txCache = new TransactionalCache(cache);
            transactionalCaches.put(cache, txCache);
        }
        return txCache;
    }
}
```

TransactionalCacheManager 内部维护了 Cache 实例与 TransactionalCache 实例间的映射关系，该类

也仅负责维护两者的映射关系，真正做事的还是 TransactionalCache。TransactionalCache 是⼀种缓

存装饰器，可以为 Cache 实例增加事务功能。我在之前提到的脏读问题正是由该类进⾏处理的。下⾯分

析⼀下该类的逻辑。

**TransactionalCache**

```java
public class TransactionalCache implements Cache {
    //真正的缓存对象，和上⾯的Map<Cache, TransactionalCache>中的Cache是同⼀个
    private final Cache delegate;
    private boolean clearOnCommit;
    // 在事务被提交前，所有从数据库中查询的结果将缓存在此集合中
    private final Map<Object, Object> entriesToAddOnCommit;
    // 在事务被提交前，当缓存未命中时，CacheKey 将会被存储在此集合中
    private final Set<Object> entriesMissedInCache;
    @Override
    public Object getObject(Object key) {
        // 查询的时候是直接从delegate中去查询的，也就是从真正的缓存对象中查询
        Object object = delegate.getObject(key);
        if (object == null) {
            // 缓存未命中，则将 key 存⼊到 entriesMissedInCache 中
            entriesMissedInCache.add(key);
        }
        if (clearOnCommit) {
            return null;
        } else {
            return object;
        }
    }
    @Override
    public void putObject(Object key, Object object) {
        // 将键值对存⼊到 entriesToAddOnCommit 这个Map中中，⽽⾮真实的缓存对象
        delegate 中
        entriesToAddOnCommit.put(key, object);
    }
    @Override
    public Object removeObject(Object key) {
        return null;
    }
    @Override
    public void clear() {
        clearOnCommit = true;
        // 清空 entriesToAddOnCommit，但不清空 delegate 缓存
        entriesToAddOnCommit.clear();
    }
    public void commit() {
        // 根据 clearOnCommit 的值决定是否清空 delegate
        if (clearOnCommit) {
            delegate.clear();
        }
        // 刷新未缓存的结果到 delegate 缓存中
        flushPendingEntries();
        // 重置 entriesToAddOnCommit 和 entriesMissedInCache
        reset();
    }
    public void rollback() {
        unlockMissedEntries();
        reset();
    }
    private void reset() {
        clearOnCommit = false;
        // 清空集合
        entriesToAddOnCommit.clear();
        entriesMissedInCache.clear();
    }
    private void flushPendingEntries() {
        for (Map.Entry<Object, Object> entry :
                entriesToAddOnCommit.entrySet()) {
            // 将 entriesToAddOnCommit 中的内容转存到 delegate 中
            delegate.putObject(entry.getKey(), entry.getValue());
        }
        for (Object entry : entriesMissedInCache) {
            if (!entriesToAddOnCommit.containsKey(entry)) {
                // 存⼊空值
                delegate.putObject(entry, null);
            }
        }
    }
    private void unlockMissedEntries() {
        for (Object entry : entriesMissedInCache) {
            try {
                // 调⽤ removeObject 进⾏解锁
                delegate.removeObject(entry);
            } catch (Exception e) {
                log.warn("...");
            }
        }
    }
}
```

存储⼆级缓存对象的时候是放到了TransactionalCache.entriesToAddOnCommit这个map中，但是每

次查询的时候是直接从TransactionalCache.delegate中去查询的，所以这个⼆级缓存查询数据库后，设

置缓存值是没有⽴刻⽣效的，主要是因为直接存到 delegate 会导致脏数据问题

**为何只有SqlSession提交或关闭之后？**

那我们来看下SqlSession.commit()⽅法做了什么

**SqlSession**

```java
@Override
public void commit(boolean force) {
    try {
        // 主要是这句
        executor.commit(isCommitOrRollbackRequired(force));
        dirty = false;
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error committing transaction. 
                Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
// CachingExecutor.commit()
@Override
public void commit(boolean required) throws SQLException {
    delegate.commit(required);
    tcm.commit();// 在这⾥
}
// TransactionalCacheManager.commit()
public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
        txCache.commit();// 在这⾥
    }
}
// TransactionalCache.commit()
public void commit() {
    if (clearOnCommit) {
        delegate.clear();
    }
    flushPendingEntries();//这⼀句
    reset();
}
// TransactionalCache.flushPendingEntries()
private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
        // 在这⾥真正的将entriesToAddOnCommit的对象逐个添加到delegate中，只有这时，⼆
        级缓存才真正的⽣效
        delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
        if (!entriesToAddOnCommit.containsKey(entry)) {
            delegate.putObject(entry, null);
        }
    }
}
```

**⼆级缓存的刷新**
我们来看看SqlSession的更新操作

```java
public int update(String statement, Object parameter) {
    int var4;
    try {
        this.dirty = true;
        MappedStatement ms = this.configuration.getMappedStatement(statement);
        var4 = this.executor.update(ms, this.wrapCollection(parameter));
    } catch (Exception var8) {
        throw ExceptionFactory.wrapException("Error updating database. Cause:
                " + var8, var8);
    } finally {
        ErrorContext.instance().reset();
    }
    return var4;
}
public int update(MappedStatement ms, Object parameterObject) throws
        SQLException {
    this.flushCacheIfRequired(ms);
    return this.delegate.update(ms, parameterObject);
}
private void flushCacheIfRequired(MappedStatement ms) {
    //获取MappedStatement对应的Cache，进⾏清空
    Cache cache = ms.getCache();
    //SQL需设置flushCache="true" 才会执⾏清空
    if (cache != null && ms.isFlushCacheRequired()) {
        this.tcm.clear(cache);
    }
}
```

MyBatis⼆级缓存只适⽤于不常进⾏增、删、改的数据，⽐如国家⾏政区省市区街道数据。⼀但数据变

更，MyBatis会清空缓存。因此⼆级缓存不适⽤于经常进⾏更新的数据。

**总结：**

在⼆级缓存的设计上，MyBatis⼤量地运⽤了装饰者模式，如CachingExecutor, 以及各种Cache接⼝的

装饰器。

- ⼆级缓存实现了Sqlsession之间的缓存数据共享，属于namespace级别

- ⼆级缓存具有丰富的缓存策略。

- ⼆级缓存可由多个装饰器，与基础缓存组合⽽成

- ⼆级缓存⼯作由 ⼀个缓存装饰执⾏器CachingExecutor和 ⼀个事务型预缓存TransactionalCache

完成。

 **10.4** **延迟加载源码剖析：**

**什么是延迟加载？**
**问题**
      在开发过程中很多时候我们并不需要总是在加载⽤户信息时就⼀定要加载他的订单信息。此时就是我

们所说的延迟加载。

**举个栗⼦**

```
* 在⼀对多中，当我们有⼀个⽤户，它有个100个订单
 在查询⽤户的时候，要不要把关联的订单查出来？
 在查询订单的时候，要不要把关联的⽤户查出来？
 
* 回答
 在查询⽤户时，⽤户下的订单应该是，什么时候⽤，什么时候查询。
 在查询订单时，订单所属的⽤户信息应该是随着订单⼀起查询出来。
```

**延迟加载**

   就是在需要⽤到数据时才进⾏加载，不需要⽤到数据时就不加载数据。延迟加载也称懒加载。

```
* 优点：
 先从单表查询，需要时再从关联表去关联查询，⼤⼤提⾼数据库性能，因为查询单表要⽐关联查询多张表
速度要快。
 
* 缺点：
 因为只有当需要⽤到数据时，才会进⾏数据库查询，这样在⼤批量数据查询时，因为查询⼯作也要消耗时
间，所以可能造成⽤户等待时间变⻓，造成⽤户体验下降。
 
* 在多表中：
 ⼀对多，多对多：通常情况下采⽤延迟加载
 ⼀对⼀（多对⼀）：通常情况下采⽤⽴即加载
 
* 注意：
 延迟加载是基于嵌套查询来实现的
```

**实现**

**局部延迟加载**

​    在association和collection标签中都有⼀个fetchType属性，通过修改它的值，可以修改局部的加载策

略。

```xml
<!-- 开启⼀对多 延迟加载 -->
<resultMap id="userMap" type="user">
    <id column="id" property="id"></id>
    <result column="username" property="username"></result>
    <result column="password" property="password"></result>
    <result column="birthday" property="birthday"></result>
    <!--
    fetchType="lazy" 懒加载策略
    fetchType="eager" ⽴即加载策略
    -->
    <collection property="orderList" ofType="order" column="id"
                select="com.lagou.dao.OrderMapper.findByUid" fetchType="lazy">
    </collection>
</resultMap>
<select id="findAll" resultMap="userMap">
    SELECT * FROM `user`
</select>
```

**全局延迟加载**

在Mybatis的核⼼配置⽂件中可以使⽤setting标签修改全局的加载策略。

```xml
<settings>
 <!--开启全局延迟加载功能-->
 <setting name="lazyLoadingEnabled" value="true"/>
</settings>
```

**注意**

```xml
<!-- 关闭⼀对⼀ 延迟加载 -->
<resultMap id="orderMap" type="order">
    <id column="id" property="id"></id>
    <result column="ordertime" property="ordertime"></result>
    <result column="total" property="total"></result>
    <!--
    fetchType="lazy" 懒加载策略
    fetchType="eager" ⽴即加载策略
    -->
    <association property="user" column="uid" javaType="user"
                 select="com.lagou.dao.UserMapper.findById" fetchType="eager">
    </association>
</resultMap>
<select id="findAll" resultMap="orderMap">
    SELECT * from orders
</select>
```

**延迟加载原理实现**

它的原理是，使⽤ CGLIB 或 Javassist( 默认 ) 创建⽬标对象的代理对象。当调⽤代理对象的延迟加载属

性的 getting ⽅法时，**进⼊拦截器⽅法**。⽐如调⽤ a.getB().getName() ⽅法，进⼊拦截器的

invoke(...) ⽅法，发现 a.getB() 需要延迟加载时，那么就会单独发送事先保存好的查询关联 B

对象的 SQL ，把 B 查询上来，然后调⽤ a.setB(b) ⽅法，于是 a 对象 b 属性就有值了，接着完

成 a.getB().getName() ⽅法的调⽤。这就是延迟加载的基本原理

总结：延迟加载主要是通过动态代理的形式实现，通过代理拦截到指定⽅法，执⾏数据加载。

**延迟加载原理（源码剖析)**

MyBatis延迟加载主要使⽤：Javassist，Cglib实现，类图展示：
![image-20231129183102907](imgs/image-20231129183102907.png)

**Setting** **配置加载：**

```java
public class Configuration {
    /**
     * aggressiveLazyLoading：
     * 当开启时，任何⽅法的调⽤都会加载该对象的所有属性。否则，每个属性会按需加载（参考
     * lazyLoadTriggerMethods).
     * 默认为true
     */
    protected boolean aggressiveLazyLoading;
    /**
     * 延迟加载触发⽅法
     */
    protected Set<String> lazyLoadTriggerMethods = new HashSet<String>
            (Arrays.asList(new String[]{"equals", "clone", "hashCode", "toString"}));
    /**
     * 是否开启延迟加载
     */
    protected boolean lazyLoadingEnabled = false;
    /**
     * 默认使⽤Javassist代理⼯⼚
     *
     * @param proxyFactory
     */
    public void setProxyFactory(ProxyFactory proxyFactory) {
        if (proxyFactory == null) {
            proxyFactory = new JavassistProxyFactory();
        }
        this.proxyFactory = proxyFactory;
    }
    //省略...
}
```

**延迟加载代理对象创建**

Mybatis的查询结果是由ResultSetHandler接⼝的handleResultSets()⽅法处理的。ResultSetHandler

接⼝只有⼀个实现，DefaultResultSetHandler，接下来看下延迟加载相关的⼀个核⼼的⽅法

```java
//#mark 创建结果对象
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap,
                                  ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new
            ArrayList<Class<?>>();
    final List<Object> constructorArgs = new ArrayList<Object>();
    //#mark 创建返回的结果映射的真实对象
    Object resultObject = createResultObject(rsw, resultMap,
            constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {
            // 判断属性有没配置嵌套查询，如果有就创建代理对象
            if (propertyMapping.getNestedQueryId() != null&& propertyMapping.isLazy()) {
                //#mark 创建延迟加载代理对象
                resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, 
                        configuration, objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    this.useConstructorMappings = resultObject != null ;;
    !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
}
```

默认采⽤javassistProxy进⾏代理对象的创建

![image-20231129184127008](imgs/image-20231129184127008.png)

JavasisstProxyFactory实现

```java
 public class JavassistProxyFactory implements org.apache.ibatis.executor.loader.ProxyFactory {
 /**
  * 接⼝实现
  * @param target ⽬标结果对象
  * @param lazyLoader 延迟加载对象
  * @param configuration 配置
  * @param objectFactory 对象⼯⼚
  * @param constructorArgTypes 构造参数类型
  * @param constructorArgs 构造参数值
  * @return
  */
 @Override
 public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) 
     return JavassistProxyFactory.EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
 }
 //省略...


/**
 * 代理对象实现，核⼼逻辑执⾏
 */
private static class EnhancedResultObjectProxyImpl implements MethodHandler {
    /**
     * 创建代理对象
     *
     * @param type
     * @param callback
     * @param constructorArgTypes
     * @param constructorArgs
     * @return
     */
    static Object crateProxy(Class<?> type, MethodHandler callback,
                             List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
        ProxyFactory enhancer = new ProxyFactory();
        enhancer.setSuperclass(type);
        try {
            //通过获取对象⽅法，判断是否存在该⽅法
            type.getDeclaredMethod(WRITE_REPLACE_METHOD);
            // ObjectOutputStream will call writeReplace of objects returned by
            writeReplace
            if (log.isDebugEnabled()) {
                log.debug(WRITE_REPLACE_METHOD + " method was found on bean " + type + ", make sure it returns this");
            }
        } catch (NoSuchMethodException e) {
            //没找到该⽅法，实现接⼝
            enhancer.setInterfaces(new Class[]{WriteReplaceInterface.class});
        } catch (SecurityException e) {
            // nothing to do here
        }
        Object enhanced;
        Class<?>[] typesArray = constructorArgTypes.toArray(new
                Class[constructorArgTypes.size()]);
        Object[] valuesArray = constructorArgs.toArray(new
                Object[constructorArgs.size()]);
        try {
            //创建新的代理对象
            enhanced = enhancer.create(typesArray, valuesArray);
        } catch (Exception e) {
            throw new ExecutorException("Error creating lazy proxy. Cause: " + e, e);
        }
        //设置代理执⾏器
        ((Proxy) enhanced).setHandler(callback);
        return enhanced;
    }
    /**
     * 代理对象执⾏
     *
     * @param enhanced    原对象
     * @param method      原对象⽅法
     * @param methodProxy 代理⽅法
     * @param args        ⽅法参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object enhanced, Method method, Method methodProxy,
                         Object[] args) throws Throwable {
        final String methodName = method.getName();
        try {
            synchronized (lazyLoader) {
                if (WRITE_REPLACE_METHOD.equals(methodName)) {
                    //忽略暂未找到具体作⽤
                    Object original;
                    if (constructorArgTypes.isEmpty()) {
                        original = objectFactory.create(type);
                    } else {
                        original = objectFactory.create(type, constructorArgTypes,
                                constructorArgs);
                    }
                    PropertyCopier.copyBeanProperties(type, enhanced, original);
                    if ( lazyLoader.size() & gt;
                    0){
                        return new JavassistSerialStateHolder(original,
                                lazyLoader.getProperties(), objectFactory, constructorArgTypes,
                                constructorArgs);
                    } else{
                        return original;
                    }
                } else {
                    //延迟加载数量⼤于0
                    if ( lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)){
                        //aggressive ⼀次加载性所有需要要延迟加载属性或者包含触发延迟加载⽅法
                        if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                            log.debug( "==> laze lod trigger method:" +
                                    methodName + ",proxy method:"
                            +methodProxy.getName() + "class:
                            +enhanced.getClass());
                            //⼀次全部加载
                            lazyLoader.loadAll();
                        } else if (PropertyNamer.isSetter(methodName)) {
                            //判断是否为set⽅法，set⽅法不需要延迟加载
                            final String property =
                                    PropertyNamer.methodToProperty(methodName);
                            lazyLoader.remove(property);
                        } else if (PropertyNamer.isGetter(methodName)) {
                            final String property =
                                    PropertyNamer.methodToProperty(methodName);
                            if (lazyLoader.hasLoader(property)) {
                                //延迟加载单个属性
                                lazyLoader.load(property);
                                log.debug( "load one :"+methodName);
                            }
                        }
                    }
                }
            }
            return methodProxy.invoke(enhanced, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
}
```



**注意事项**

1. IDEA调试问题 当配置aggressiveLazyLoading=true，在使⽤IDEA进⾏调试的时候，如果断点打到

代理执⾏逻辑当中，你会发现延迟加载的代码永远都不能进⼊，总是会被提前执⾏。 主要产⽣的

原因在aggressiveLazyLoading，因为在调试的时候，IDEA的Debuger窗体中已经触发了延迟加载

对象的⽅法。



> 延迟加载原理 https://mp.weixin.qq.com/s/76x1flH6xTz4BRs3LJuCuQ

# 第⼗⼀部分：设计模式
虽然我们都知道有3类23种设计模式，但是⼤多停留在概念层⾯，Mybatis源码中使⽤了⼤量的设计模式，观察设计模式在其中的应⽤，能够更深⼊的理解设计模式Mybati s⾄少⽤到了以下的设计模式的使⽤：

| 模式         | mybatis体现                                                                                                                             |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| Builder模式  | 例如SqlSessionFactoryBuilder、Environment;                                                                                              |
| ⼯⼚⽅法模式 | 例如SqlSessionFactory、TransactionFactory、LogFactory                                                                                   |
| 单例模式     | 例如 ErrorContext 和 LogFactory；                                                                                                       |
| 代理模式     | Mybatis实现的核⼼，⽐如MapperProxy、ConnectionLogger，⽤的jdk的动态代理还有executor.loader包使⽤了 cglib或者javassist达到延迟加载的效果 |
| 组合模式     | 例如SqlNode和各个⼦类ChooseSqlNode等                                                                                                    |
| 模板⽅法模式 | 例如 BaseExecutor 和 SimpleExecutor，还有 BaseTypeHandler 和所有的⼦类例如IntegerTypeHandler；                                          |
| 适配器模式   | 例如Log的Mybatis接⼝和它对jdbc、log4j等各种⽇志框架的适配实现；                                                                         |
| 装饰者模式   | 例如Cache包中的cache.decorators⼦包中等各个装饰者的实现；                                                                               |
| 迭代器模式   | 例如迭代器模式PropertyTokenizer；                                                                                                       |

接下来对Builder构建者模式、⼯⼚模式、代理模式进⾏解读，先介绍模式⾃身的知识，然后解读在

Mybatis中怎样应⽤了该模式。


