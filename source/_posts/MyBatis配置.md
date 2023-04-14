---
title: MyBatis配置
date: 2023-04-14 19:09:19
tags: MyBatis
categories: 后端学习笔记
description: |
    **_SSM下的MyBatis，之后会写SpringBoot整合MyBatisPlus_**
---

# 起步配置

## 1. 导入Maven坐标

```xml
<!--Mybatis 依赖-->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.5</version>
</dependency>
<!--junit 单元测试-->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13</version>
    <scope>test</scope>
</dependency>
<!--添加slf4j日志api-->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.20</version>
</dependency>
<!--添加logback-classic依赖-->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
<!--添加logback-core依赖-->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-core</artifactId>
    <version>1.2.3</version>
</dependency>
```



## 2. 处理映射关系

### 2.1创建实体类、Mapper对象

​	这里以Account为例：

```java
@Data
@NoArgsConstructor
@Alias("Account")
public class Account {

    private Integer id;

    private String name;

    private Double money;


}
```

​	创建Mapper对象（Mapper都是接口，对应的就是以前的dao接口，里面定义抽象方法，具体的SQL语句写在Mapper对象对应的xml文件中）

​	在Mapper接口中定义方法，方法名就是SQL映射文件中sql语句的id，并保持参数类型（**parameterType**）和返回值类型（**resultType**）一致。

```java
@Mapper
public interface AccountMapper {

    Account selectById(Integer id);

    List<Account> selectByName(String name);

    void add(Account account);
}

```



​	我们如何将Account类，AccountMapper和后面将要编写的AccountMapper.xml建立映射关系？

​	需要在mybaits-config.xml中，定义好对应的映射。

 1. **首先是实体类和Mapper接口的映射：**

    我们在mybaits-config.xml中，使用typeAliases来给实体类起别名。这样在AccountMapper.xml中写SQL语句时，resultType和parameterType就不需要写全限定名了。

```xml
--两种起别名方法，一种是单独给一个类=起别名，另一种是直接扫描实体类的包，
会自动以实体类名的开头大写或小写起别名，比如Countries类的别名就是Countries或countries。
如果在对应实体类上加了@Alias注解，那么别名就是注解中的别名
------------------------------------------------
<typeAliases>
		<typeAlias type="com.cqupt.Demo.entity.Employees" alias="Employees"/>
        <package name="com.cqupt.Demo.entity"/>
</typeAliases>

--不起别名Mapper.xml中的sql语句块：
<select id="selectById" resultType="com.cqupt.Demo.entity.Account">
        select * from account where id = #{id}
</select>

--起别名Mapper.xml中的sql语句块：
<select id="selectById" resultType="Account">
        select * from account where id = #{id}
</select>
```





2. **其次是Mapper接口和Mapper.xml的映射：**

​		Mapper接口中的抽象方法需要通过SQL语句实现，那么MyBatis怎么知道哪个Mapper用哪个xml中的SQL呢？实现Mapper接口和Mapper.xml的映射需要满足一下条件：

​		（1）Mapper接口和Mapper.xml名字必须保持一致

​			![](1.png)

​		（2）Mapper接口和Mapper.xml编译后要放在同文件夹下

​			要使Maven编译后在同一目录下，就要在resources目录下创建com/cqupt/Demo/mapper文件夹,结构与AccountMapper路径相同。

​			注：创建文件夹时，不能以“.”隔开包名，而要以“/”来隔开

​		（3）要在mybatis-config.xml核心配置文件中，声明Mapper.xml文件的位置

​			

```xml
<!--同样两种方式--> 
<mappers>
    	<!--一种是单独指定某个文件，随着数据库表的增多，会越来越多-->
        <mapper resource="mapper/AccountMapper.xml"/>
        <mapper resource="mapper/CountriesMapper.xml"/>
    	<!--另一种是包扫描方式，如果所有Mapper.xml映射文件和同名的Mapper接口放在同一目录下，可以通过扫描包的方式加载所有SQL映射文件。这样就不需要在mybaytis-config.xml中一条条配置不同的mapper resource.-->
        <package name="com.cqupt.Demo.mapper"/>
 </mappers>
```





## 3. 编写配置文件

### 3.1 resources目录下创建logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--
        CONSOLE ：表示当前的日志信息是可以输出到控制台的。
    -->
    <appender name="Console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>【%level】 %blue(%d{HH:mm:ss.SSS}) %cyan(【%thread】) %boldGreen(%logger{15}) - %msg %n</pattern>
        </encoder>
    </appender>

    <logger name="com.itheima" level="DEBUG" additivity="false">
        <appender-ref ref="Console"/>
    </logger>

    <root level="DEBUG">
        <appender-ref ref="Console"/>
    </root>
</configuration>
```

### 3.2 编写MyBatis核心配置文件（整合进SpringBoot后就可以省略了）

mybatis-config.xml

```xml
<!-----------------------------官网内容----------------------------------->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>

<!----------------------------------------------------------------------->
<!-------------------------自己配置的内容--------------------------------->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <!--数据库连接信息-->
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql:///mybaytis?useSSL=false"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <!--加载sql映射文件-->
        <mapper resource="UserMapper.xml"/>
    </mappers>
</configuration>
```

### 3.3 编写Mapper.xml

​	在 main 目录下的 resoures 配置文件目录下创建一个名为 **AccountMapper.xml** 的配置文件(映射user表就是AccountMapper，xx表就是XxMapper)，内容在MyBatis官网

```xml
<!-----------------------------官网内容----------------------------------->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.mybatis.example.BlogMapper">
  <select id="selectBlog" resultType="Blog">
    select * from Blog where id = #{id}
  </select>
</mapper>
<!----------------------------------------------------------------------->
<!-------------------------自己配置的内容----------------------------------->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.cqupt.Demo.mapper.AccountMapper">
	
    <select id="selectById" resultType="com.cqupt.Demo.entity.Account">
        select * from account where id = #{id}
    </select>


    <select id="selectByName" resultType="com.cqupt.Demo.entity.Account">
        select * from account where name = #{name}
    </select>

    <insert id="add" parameterType="com.cqupt.Demo.entity.Account" useGeneratedKeys="true" keyProperty="id">
        insert into account(`name`,money) values (#{name},#{money})
    </insert>
</mapper>
```

​	其中，如果数据库字段名与相应实体类的属性名不一致，可以使用如下几种方法：

```xml
<!--
        解决数据库列名与类的属性名不一致的问题
        1.在sql语句中起别名，让别名与实体类属性名一样，缺点是每次查询都要重新起别名，很麻烦。
    -->
    
    <select id="selectById" resultType="countries">
        select country_id as countryName,
               country_name as countryName,
               region_id as regionId from countries where country_name = #{countryName}
    </select>
    
-----------------------------------------------------------------------------

    
<!--
		2.另一种方法是定义sql片段，之后在select查询中引用该片段：（缺点，不灵活）
	-->
	<sql id="country_colomn">
        country_name as countryName, country_id as countryId, region_id as regionId
    </sql>
    <select id="selectAll" resultType="countries">
        select <include refid="country_colomn"/>from countries;
    </select>

-----------------------------------------------------------------------------

<!--
		3.最后是定义resultMap
	-->	
	<resultMap id="countryResultMap" type="Countries">
        <id property="countryId" column="country_id"/>
        <result property="countryName" column="country_name"/>
        <result property="regionId" column="region_id"/>
    </resultMap>

    <select id="selectById" resultMap="countryResultMap">
        select * from countries where country_id = #{countryId}
    </select>
```

​	resultMap内，id字段表示主键的映射，result字段表示一般列名的映射。这里两个不一致的都是一般列名，所以使用result。 column为数据库中字段名，property为实体类中属性名。

​	在查询时，要将resultTtype改为resultMap，其值为resultMap的id值。





​	配置完成后就能在对应的Mapper.xml中写sql，实现自己想要的功能了。写个测试类：

```java
@Test
    void selectByCountryName() throws Exception{
        String resource = "mybatis-config.xml";
        InputStream resourceAsStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        CountriesMapper mapper = sqlSession.getMapper(CountriesMapper.class);
        System.out.println(mapper.selectByName("Australia").toString());
        sqlSession.close();
    }
```





























































































