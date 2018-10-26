---
title: mybatis代码生成工具
date: 2018-10-19 13:52:20
categories:
- mybatis
tags:
- mybatis
- 代码生成
---

使用方式之一====MAVEN

XML配置参数说明

其他注意事项

<!--more-->

## 使用方式之一====MAVEN

### pom

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.3.1</version>
</dependency>

<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.7</version>
</dependency>


<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.7</version>
    <configuration>
        <configurationFile>src/main/resources/generatorConfig.xml</configurationFile>
        <verbose>true</verbose>
        <overwrite>true</overwrite>
    </configuration>
    <executions>
        <execution>
            <id>Generate MyBatis Artifacts</id>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```



## generatorConfig.xml

```xml
<classPathEntry
                location="C:/repository/mysql/mysql-connector-java/5.1.26/mysql-connector-java-5.1.26.jar"/>
<context id="sqlserverTables" targetRuntime="MyBatis3">
    <!-- 生成的pojo，将implements Serializable -->
    <plugin type="org.mybatis.generator.plugins.SerializablePlugin"></plugin>
    <commentGenerator>
        <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
        <property name="suppressAllComments" value="true" />
    </commentGenerator>

    <!-- 数据库链接URL、用户名、密码 -->
    <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                    connectionURL="jdbc:mysql://数据库地址"
                    userId="daojia_root" password="aaa111">
    </jdbcConnection>

    <!-- 默认false，把JDBC DECIMAL 和 NUMERIC 类型解析为 Integer 
		true，把JDBC DECIMAL和 NUMERIC 类型解析为java.math.BigDecimal -->
    <javaTypeResolver>
        <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>

    <!-- 生成model模型，对应的包路径，以及文件存放路径(targetProject)，targetProject可以指定具体的路径,如./src/main/java，
            也可以使用“MAVEN”来自动生成，这样生成的代码会在target/generatord-source目录下 -->
    <!--<javaModelGenerator targetPackage="com.joey.mybaties.test.pojo" targetProject="MAVEN"> -->
    <javaModelGenerator targetPackage="entity"
                        targetProject="MAVEN">
        <property name="enableSubPackages" value="true" />
        <!-- 从数据库返回的值被清理前后的空格 -->
        <property name="trimStrings" value="true" />
    </javaModelGenerator>

    <!--对应的mapper.xml文件 -->
    <sqlMapGenerator targetPackage="mappers"
                     targetProject="MAVEN">
        <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>

    <!-- 对应的Mapper接口类文件 -->
    <javaClientGenerator type="XMLMAPPER"
                         targetPackage="dao" targetProject="MAVEN">
        <property name="enableSubPackages" value="true" />
    </javaClientGenerator>


    <!-- 列出要生成代码的所有表，这里配置的是不生成Example文件 -->
    <table tableName="t_app_order" domainObjectName="Order">
    </table>
</context>
```

### table

必填

tableName ：表名

可选

schema 

catalog 

alias 

domainObjectName ：生成对象的名称

mapperName ：生成的dao和xml文件的名称：默认是加后缀mapper

sqlProviderName ：生成实体的的sqlProvider，默认是加后缀SqlProvider 

enableInsert ：insert方法，默认true

enableSelectByPrimaryKey :查询根据主键，默认true

enableSelectByExample :根据example查询，默认true

enableUpdateByPrimaryKey :根据主键更新

enableDeleteByPrimaryKey ：根据主键删除,不推荐-----改为false

enableDeleteByExample ：根据example删除  ，不推荐---改为false

enableCountByExample ：根据example计数，true

enableUpdateByExample :根据example更新

selectByPrimaryKeyQueryId 

selectByExampleQueryId 

modelType 

escapeWildcards 

delimitIdentifiers 

delimitAllColumns 

## 其他注意事项

### example 使用

| 方法                                       | 说明                                          |
| ------------------------------------------ | --------------------------------------------- |
| example.setOrderByClause(“字段名 ASC”);    | 添加升序排列条件，DESC为降序                  |
| example.setDistinct(false）                | 去除重复，boolean型，true为选择不重复的记录。 |
| criteria.andXxxIsNul                       | 添加字段xxx为null的条件                       |
| criteria.andXxxIsNotNull                   | 添加字段xxx不为null的条件                     |
| criteria.andXxxEqualTo(value)              | 添加xxx字段等于value条件                      |
| criteria.andXxxNotEqualTo(value)           | 添加xxx字段不等于value条件                    |
| criteria.andXxxGreaterThan(value)          | 添加xxx字段大于value条件                      |
| criteria.andXxxGreaterThanOrEqualTo(value) | 添加xxx字段大于等于value条件                  |
| criteria.andXxxLessThan(value)             | 添加xxx字段小于value条件                      |
| criteria.andXxxLessThanOrEqualTo(value)    | 添加xxx字段小于等于value条件                  |
| criteria.andXxxIn(List<？>)                | 添加xxx字段值在List<？>条件                   |
| criteria.andXxxNotIn(List<？>)             | 添加xxx字段值不在List<？>条件                 |
| criteria.andXxxLike(“%”+value+”%”)         | 添加xxx字段值为value的模糊查询条件            |
| criteria.andXxxNotLike(“%”+value+”%”)      | 添加xxx字段值不为value的模糊查询条件          |
| criteria.andXxxBetween(value1,value2)      | 添加xxx字段值在value1和value2之间条件         |
| criteria.andXxxNotBetween(value1,value2)   | 添加xxx字段值不在value1和value2之间条件       |

### sqlprovider使用

在类中写sql，mapper接口上标注方法，四不像

## 参考文献

- http://www.mybatis.org/generator/configreference/table.html
- http://www.cnblogs.com/wangkeai/p/6934683.html