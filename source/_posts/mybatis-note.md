---
title: Mybatis学习笔记
date: 2019-04-08 13:44:13
categories:
- Java
- Mybatis
tags:
- Java
- Mybatis
---

<!--more-->

### Mybatis简介

开源免费框架，原名叫iBatis

作用：数据访问层框架，底层是对JDBC的封装



优点：

- 使用mybatis时不需要编写实现类，只需要写执行的sql命令





### Mybatis简单使用

mybatis-config.xml：全局配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="default">
        <environment id="default">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3308/learnjsp?useSSL=false"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>


    <mappers>
        <mapper resource="UserMapper.xml"/>
    </mappers>
</configuration>
```





mapper.xml文件：编写需要执行的SQL命令，把XML文件理解成实现类



UserMapper.xml：配置了sql语句，以及sql的封装规则

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--namespace:命名空间-->
<mapper namespace="com.test.UserMapper">
    <!--
    id:sql语句的唯一标识
    parameterType:定义参数类型
    resultType:返回值类型
    如果方法返回值返回的是list,在resultType中写List的泛型
#{id}:外部传入的参数
    -->
    <select id="selectUser" resultType="com.test.model.User">
        select uid,uname,pwd,sex,age from t_user where id = #{id}
    </select>
</mapper>
```





测试

```java
package com.test.test;

import com.test.model.User;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;
import java.util.List;

/**
 * @author ：L1nker4
 * @date ： 创建于  2019/3/10 17:19
 * @description： test
 */
public class Test {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        //获取sqlSession实例，能直接执行已经映射的sql语句
        SqlSession openSession =  sqlSessionFactory.openSession();
        List<User> user = openSession.selectList("com.test.UserMapper.selectUser");
        System.out.println(user);

        //关闭session
        openSession.close();
    }
}

```



Mybatis使用步骤：

1. 根据配置文件，创建一个SqlSessionFactory对象

2. sql映射文件，配置sql语句

3. 将sql映射文件注册到全局配置文件中

4. 写代码

   1. 根据全局配置文件得到SqlSessionFactory
   2. 使用sqlSession工厂，获取到sqlSession对象使用它来执行增删改查，一个SqlSession就是代表和数据库的一次会话，用完关闭。
   3. 使用sql的唯一标识来告诉Mybatis执行哪个sql，sql语句都保存在sql映射文件中。





接口式编程

	原生：		Dao	====>	DaoImpl
	
	Mybatis	      Mapper ====>	xxMapper.xml



mapper接口没有实现类，但是mybatis会为这个接口生成一个代理对象。



mybatis的配置文件：包含数据库连接池的信息，事务管理器信息等，系统运行环境。

sql映射文件：保存了每一个sql语句的映射信息。





### Mybatis全局配置文件

Mybatis的配置文件包含了影响Mybatis行为的设置（setting）和属性（properties）信息。



configuration 配置

	properties属性
	
	settings 设置
	
	typeAliases 类型命名
	
	typeHandlers 类型处理器
	
	objectFactory 对象工厂
	
	plugins	插件
	
	environments 环境
	
		environment 环境变量
	
			transactionManager	事务管理器
	
			dataSource 数据源

databaseIdProvider 提供多个数据库厂商

mappers 映射器





#### properties 引入配置

引入外部properties配置文件的内容

resource：类路径下的资源

url：引入网络路径i心爱的资源

```xml
<properties resource="db.properties"></properties>
```





#### setting Mybatis设置

![1553326004994](C:\Users\l1nke\AppData\Roaming\Typora\typora-user-images\1553326004994.png)



setting：用来设置每一个设置项

	name：设置项名
	
	value：值

```xml
<settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```



#### typeAliases 别名处理器

typeAliases：可以为Java类型起别名

	type：指定要起别名的类型全类名，默认别名就是类名小写
	
	alias：指定新的别名，



```xml
<typeAliases>
        <typeAlias type="com.test.model.User" alias="User"/>
</typeAliases>
```



##### package：批量起别名

	name：指定包名（为当前包以及下面的所有后代包的每一个类都起一个默认别名）

```xml
<package name="com.test.entities">
```

批量起别名的情况下，使用@Alias注解为某个类型指定新的别名

```java
@Alias("per")
public class Person{
    
}
```



#### environments 环境

Mybatis可以配置多种环境，default属性指定使用某种环境

environment：配置一个具体的环境信息，必须有以下两个标签，id代表当前环境的唯一标识

	transactionManager：事务管理器，
	
			type：事务管理器的类型，JDBC和MANAGED
	
			自定义事务管理器，实现TransactionFactory接口即可
	
	dataSource：数据源
	
			type：数据源类型，UNPOOLED，POOLED，JNDI	
	
			自定义数据源：实现DataSourceFactory接口


​		

#### databaseIdProvider 提供多数据库厂商

```xml
<databaseIdProvider type="DB_VENDOR">
        <!--为不同的数据库厂商起别名-->
        <property name="MySQL" value="mysql"/>
        <property name="Oracle" value="oracle"/>
</databaseIdProvider>
```



在sql映射文件中使用databaseId指定查询哪个数据库（值由配置文件中的别名决定）

```xml
<select id="selectUser" resultType="User" databaseId="mysql">
        select uid,uname,pwd,sex,age from t_user;
    </select>
```



#### mappers 映射

将sql映射注册到全局配置中

```xml
<mappers>
        <mapper resource="UserMapper.xml"/>
</mappers>
```

注册配置文件

resource：引入类路径下的映射文件

url：引入磁盘或者网络路径下的映射文件



注册接口

```xml
<mapper class="com.test.dao.UserMapper" />
```

class：引用接口

接口和映射文件同名，并且存在于同一目录下



##### 批量注册

```xml
<package name="xxx.com.dao"></package>
```

配置文件和dao一个目录



### Mybatis映射文件



cache：命名空间的二级缓存配置

cache-ref：其他命名空间缓存配置的引用。

resultMap：自定义结果集映射

sql：抽取可重用语句块

insert：映射插入语句

update：映射更新语句

delete：映射删除语句

select：映射查询语句



mybatis允许增删改直接定义为：long，boolen，Integer。void



```java
//这种方式需要手动提交事务，openSession.commit();
SqlSession openSession =  sqlSessionFactory.openSession();
//这种不需要提交事务
SqlSession openSession =  sqlSessionFactory.openSession(true);
```

MySQL支持自增主键，自增主键的获取，mybatis利用statement.getGenratedKeys()，

```sql
<insert id="addUser" parameterType="com.test.model.User" useGeneratedKeys="true" keyProperty="id">
        insert into t_user(uname, pwd, sex, age)
        values (#{uname},#{pwd},#{sex},#{age})
</insert>
```



useGeneratedKeys="true"，使用自增主键获取主键值策略



#### 参数处理

单个参数

```xml
<delete id="deleteUserById">
        delete from t_user where uid = #{uid}
</delete>
```



多个参数（多个参数会被封装成一个map）

key：param1，paramN

value：传入的参数值

#{}是从map中获取指定的key的值

```xml
<select id="getUserByIdAndUname" resultType="User">
        select * from t_user where uid = #{param1} and uname = #{param2}
 </select>
```



###### 使用@Param注解

```java
public User getUserByIdAndUname(@Param("uid") Integer uid, @Param("uname") String uname);
```

@Param注解指定对应的key值



#### #和$的取值区别

${}：取出的值直接拼接在sql语句中，会有安全问题

#{}：是以预编译的形式，将参数设置到sql语句中，PreparedStatement

大多情况下，取参数都应该使用#{}



原生jdbc不支持占位符的地方，可以使用$进行取值

比如 ：分表，按照年份分表拆分

```xml
select * from ${year}_salary where xxx;
```



#{}可以规定参数的一些规则

javaType，jdbcType，mode（存储过程），numericScale，resultMap，typeHandler，jdbcTypeName，



jdbcType





#### select元素

id：唯一标识符

parameterType：参数类型

resultType：返回值类型，如果返回的是一个集合，要写集合中元素的类型。



###### 多条记录封装成一个map

```xml
<select id="getUserByUnameReturnMap" resultType="com.test.model.User">
        select * from t_user where uname like #{uname}
</select>
```

```java
@MapKey("uid")
    public Map<Integer,User> getUserByUnameReturnMap(String uname);
MapKey标注某个元素为主键
```

###### association指定联合的JavaBean对象

property：指定哪个属性是联合的对象

javaType：指定这个属性对象的类型





###### association定义关联对象的封装规则

select：表明当前属性是调用select指定的方法查出的结果

column：指定将哪一列的值传给这个方法

流程：使用select指定的方法（传入column指定的这列参数的值）查出对象，并封装给property指定的属性



```xml
<association property="dept" select="com.test.dao.UserMapper.getUserByIdAndUname" column="d_id"/>
```





##### 延时加载

在关联查询中，例如：查询用户订单情况时，只查用户信息而不查订单，可以启用延时加载

```xml
<settings>
     <setting name="lazyLoadingEnabled" value="true"/>
     <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```



##### collection定义关联集合

```xml
<resultMap id="MyUser" type="com.test.model.User">
        <id column="uid" property="uid"/>
        <result column="uname" property="uname"/>
        <result column="pwd" property="pwd"/>
        <result column="sex" property="sex"/>
        <result column="age" property="age"/>
    
        <collection property="User" ofType="com.test.model.Department">
            <id column="eid" property="id"/>
            <result column="username" property="username"/>
        </collection>
    </resultMap>
```

        ofType：指定集合里面元素类型

collection封装一个集合





###### 将多列的值传递过去

将多列的值封装map传递过去

column={key1=column1,key2=column2}





#### discriminator鉴别器

mybatis可以通过discriminator判断某列的值，然后根据某列的值改变封装行为

例如：如果查出的是女生，就把部门信息查询出来，否则不查询

	   如果是男生，就把username赋值给email



```xml
<discriminator javaType="String" column="gender">
            <!--女生 resultType：指定封装的结果类型-->
  <case value="0" resultType="">
     <association property="dept" select="com.test.dao.UserMapper.getUserByIdAndUname" column="d_id"/>
  </case>
  <case value="1" resultType="">
     <association property="dept" select="com.test.dao.UserMapper.getUserByIdAndUname" column="d_id"/>
  </case>
</discriminator>
```





### 动态SQL

#### if 判断

```xml
<select id="getUserByConditionIf" resultType="com.test.model.User">
        select * from t_user
        where
            <if test="uid != null">
                uid = #{uid}
            </if>

            <if test="uname != null">
                and uname = #{}
            </if>
    </select>
```

test：OGNL表达式

遇见特殊字符需要转义



#### where

可以防止因为if语句不显示，直接拼接and报错

```xml
<where>
   <if test="uid != null">
       uid = #{uid}
   </if>

   <if test="uname != null">
       and uname = #{}
   </if>
</where>
```

where标签会将所有的查询条件都包括在内。mybatis会将where标签中拼接的sql多出来的and

或者or去掉，**where只会去掉第一个多出来的and或者or**



#### Trim 字符串截取

自定义字符串截取规则

```xml
<trim prefix="" prefixOverrides="" suffix="" suffixOverrides="">
```

prefix：前缀，trim标签体中是整个字符串拼串后的结果，prefix给拼串后的整个字符加一个前缀

prefixOverrides：前缀覆盖，去掉整个字符串前面多余的字符

suffix：后缀，suffix给拼串后的整个字符加一个后缀

suffixOverrides：后缀覆盖



```xml
<trim prefix="where" suffixOverrides="and">
```

上述代码解释：在字符串前面加一个where，将后缀的and去掉





#### choose-when-otherwise 分支选择

相当于switch-case

需求：如果带了id就用id查，如果带了lastName就用lastName查

```xml
<select id="getUserByUnameReturnMap" resultType="com.test.model.User">
        select * from t_user
        <where>
            <choose>
                <when test="id!=null">
                    id = #{id}
                </when>
                <when test="uname!=null">
                    uname = #{uname}
                </when>
            </choose>
        </where>
</select>
```



#### set

set元素可以用于动态包含需要更新的列,可以动态删除多余的逗号



#### foreach

```xml
<foreach collection="ids" item="item_id" separator="," open="">
       #{item_id}
</foreach>
```

collection：指定要遍历的集合
    	list类型的参数会特殊处理封装在map中，map的key就叫list
item：将当前遍历出的元素赋值给指定的变量
separator：每个元素中的分隔符
open：遍历死哦有的结果拼接一个开始的字符
close：拼接一个结束的字符
index：索引



#### 内置参数

不止方法传过来的参数可以用来判断并取值

mybatis默认还有两个内置参数

_parameter：代表整个参数

	单个参数：_parameter就是这个参数
	
	多个参数：参数会被封装成一个map，_parameter就代表这个map

_databaseId：如果配置了databaseIdProvider标签，_databaseId就代表当前数据库的别名

```xml
<if test="_databaseId=='mysql'">
	xxx
</if>

<if test="_parameter!=null"></if>
```



#### bind

可以将OGNL表达式的值绑定到一个变量中的值

```xml
<bind name="_lastName" value="lastName">
    <select>
    	select * from users where lastName = #{_lastName}
    </select>
```





#### sql

```xml
<sql id="Base_Column_List">
    emp_id, emp_name, gender, email, d_id
  </sql>

<insert id="addEmp">
	insert into tbl_emp (
    <include refid="Base_Column_List"></include>
    )
    values xxxxx
</insert>
```

sql标签里面只能用`${}`进行取值





### MyBatis缓存

分为一级缓存（本地缓存）和二级缓存（全局缓存）

- 默认情况下，只有一级缓存（SqlSession级别的缓存，也称为本地缓存）开启
- 二级缓存需要手动开启和配置，基于namespace级别的缓存
- 为了提高拓展性，MyBatis定义了缓存接口Cache，我们可以通过实现Cache接口来自定义二级缓存



#### 一级缓存

与数据库同一次会话期间查询到的数据会放在本地缓存中，以后如果获取相同的数据，直接去缓存中取。

失效情况：

- sqlsession不同
- sqlsession相同，查询条件不同
- sqlsession相同，两次查询之间执行了增删改操作
- sqlsession相同，手动清除了一级缓存



#### 二级缓存

基于namespace级别的缓存，一个namespace对应一个二级缓存。



##### 开启全局二级缓存配置

```xml
<setting name="cacheEnabled" value="true"></setting>
```



##### mapper.xml中配置

```xml
<mapper namespace="">
	<cache></cache>
</mapper>
```

- eviction：缓存回收策略
  - LRU	最近最少使用的：移除最长时间不被使用的对象
  - FIFO        先进先出，按对象进入缓存的顺序来移除他们 
  - SOFT       软引用，移除基于垃圾回收器状态和软引用规则的对象
  - WEAK      弱引用，更积极地移除基于垃圾回收器状态和弱引用规则的对象
  - 默认是LRU
- flushInterval：缓存刷新间隔（默认不清空）单位 毫秒
- readOnly：是否只读
  - true：只读，mybatis认为所有从缓存中获取数据的操作都是只读操作，不会修改数据，不安全，速度快
  - false：非只读
- size：缓存中存放多少元素
- type：指定自定义缓存的全类名，实现Cache接口



##### 实体类实现序列化接口