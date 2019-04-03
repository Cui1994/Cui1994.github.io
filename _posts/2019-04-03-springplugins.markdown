---
layout: post
category: "Java"
title:  "两个常用的 Mybatis 插件"
tags: [Java, Plugin, Mybatis]
---

* content
{:toc}

![](https://picsum.photos/800/300/?image=296)
得空试了试 Mybatis 两个常用的插件，**MyBatis Generator（MBG）**和 **Mybatis PageHelper（分页插件）**，效果确实是有的，解放了生产力，不用再对着表中的字段一行行手写 PO 对象和 Mapper 了，处理重复的分页需求时也会优雅许多。但话又说回来，既然是工具，肯定会有蹩脚的地方，作为使用方，要么了解这些 point 并加以注意，要么想办法定制或者改造这些工具为自己所用，显然后者要高明一些，但需要额外的动手成本，需要一些权衡。本文对这两个插件的简单用法和使用过程中的一些问题做点笔记。





## MBG

MBG 简单来讲就是一个代码生成器，可以直接连接数据库，根据表生成对应实体类、Mapper 接口和 sql 文件，项目初期借助这个东西可以避免大量重复劳动。中文文档在这边 [MyBatis Generator 介绍](https://www.kancloud.cn/wizardforcel/java-opensource-doc/152983)。

添加项目依赖，这部分要注意两点。
- 要在 plugin 里添加 mysql 驱动的依赖，否则会在执行 plugin 时找不到相应的驱动包。
- mysql 驱动的版本需要与 mysql 的版本相匹配，否则会出现因为无法识别主键而没有生成主键相关接口等一系列怪问题。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <configuration>
                <verbose>true</verbose>
                <overwrite>true</overwrite>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>${mysql.version}</version>
                </dependency>
            </dependencies>
            <version>1.3.5</version>
        </plugin>
    </plugins>
</build>
```

配置`generatorConfig.xml`文件，这里的配置规则在文档中有详细说明，这里记录下我目前感觉好用的配置项。
- 注释项全部禁用，避免每次生成后引发版本控制问题。
    ```xml
    <commentGenerator>
      <property name="suppressAllComments" value="true" />
      <property name="suppressDate" value="true" />
    </commentGenerator>
    ```
- 数据表生成的配置项中，将 Example 相关的方法全部禁用，Example 方法目前个人感觉意义不大，有需要完全可以手写，而且生成的代码很难维护，用到生产上估计是自寻死路。
    ```xml
    <table schema="test" tableName="student" domainObjectName="Student"
      enableCountByExample="false" enableUpdateByExample="false"
      enableDeleteByExample="false" enableSelectByExample="false"
      selectByExampleQueryId="false">
    </table>
    ```

之后运行插件，就可以顺利生成对应的代码文件了。下面说一个基本上都会遇到的小坑，MBG 在生成 DO 类时，会用 Byte 类型来表述 MySQL 中的 tinyint 类型的字段。这个大家普遍给出的解决方案是重新指定新的`JavaTypeResolver`类，将 Integer 和 tinyint 进行关联。当然也可以直接修改相应的 DO 类，将 Byte 改为 Integer。（我是这么干的）

相比上面的问题，DO 类中自动生成的 set 和 get 方法才是最让我蛋疼的，如果想用 lombok 库中的注解来修饰 DO 类，需要手动将这些 set 和 get 方法删掉，不过目前来看没什么好的解决办法，毕竟工具只是工具，想一下手不动也不太可能。

总的来说 MBG 还是蛮推荐使用的，简洁高效，默认生成的一些主键查询的方法基本都会用到，省时省力。

## Pagehelper

相比 MBG，分页插件做的事要稍微高级一点，你只需要实现一个`getList`的接口，它会这个接口的 sql 进行拦截和修饰，使其具有分页功能。效果就是，你不需要为所有类似运营台的分页需求都去重复实现 LIMIT OFFSET 的 sql，也不需要新增 COUNT 查询，分页插件直接包办，代码也很优雅。官方文档在这 [Mybatis-PageHelper](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/en/HowToUse.md)

这里我在 Spring Boot 中引入 pagehelper。
```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.2.3</version>
</dependency>
```

并在`application.properties`中引入相关配置，这样配置就完成了。
```
#pagehelper
pagehelper.helperDialect=mysql
pagehelper.reasonable=true
pagehelper.supportMethodsArguments=true
pagehelper.params=count=countSql
```

使用姿势有很多，可以参照文档来，这里列出个人比较喜欢的一种，分页逻辑应该放在 Service 或者 Manager 层内。
```java
@Override
public StudentListDTO getStudentList(@NotNull Integer page, @NotNull Integer pageSize) {
PageHelper.startPage(page, pageSize);
List<Student> students = studentDao.selectList();
Long total = ((Page) students).getTotal();
return StudentListDTO.builder()
    .studentList(students)
    .total(total)
    .build();
}
```

我们其实只实现了`selectList()`的接口，借助 PageHelper 实现了分页功能。如果去查看 MySQL 的 general-log 的话可以看到 sql 中加入了 LIMIT 部分，同时在执行这条 sql 之前还额外执行了一次 count 查询。
```
2019-04-02T17:31:30.093839Z   220 Query SELECT count(0) FROM student
2019-04-02T17:31:30.102127Z   220 Query SELECT

    id,
    name,
    school_id,
    grade,
    class_no,
    created_at,
    updated_at,
    is_deleted

    FROM
    student
    ORDER BY
    id
    DESC LIMIT 2
```

关于分页插件的使用，总的看下来没什么问题，但用到生产时需要当心，毕竟隐式添加了数据库查询。

(End)
