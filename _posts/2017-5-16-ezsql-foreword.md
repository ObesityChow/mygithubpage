---
layout: post
title: "EZSql | 引子"
date: 2017-5-16
description: "看着空空如也的Github, 决定挖一个大坑"
tag: Tech-iOS
---   

在iOS开发中不可避免地要用到本地存储,通常在小数据量的情况下可以直接使用`NSKeyedArchiver`进行对象持久化,然而在数据量大且有复杂操作需求的情景下,`NSKeyedArchiver`的性能实在不堪入目. 这时候就有两个选择:`CoreData`和`Sqlite`

`CoreData`不用多说,作为亲儿子,基本每次iOS更新都会提供新的Api,性能也在逐步提高,但是考虑到CoreData属于仅用于Apple系设备的编程,学习成本高且不利于跨界.而作为一名程序员或多或少都接触过数据库,此时`Sqlite`的优势便凸显而出,同时,在跨平台情况下,`Sqlite`的可移植性也是无法忽视的优势.

## 挖了个新坑

日常iOS开发中一般使用FMDB操作数据库,但是在日益复杂的需求中,FMDB的Sql语句拼接功能略显羸弱,且市面上大部分框架对树状对象结构支持得都不是很好.

所以,用惯了MyBatis之后,决定自己造一个类MyBatis的SqlBuilder,起名EZSql,一来丰富简历,二来提供iOS开发中数据库操作的另一个选择

## 功能设计

### Mapper.xml
格式类似Mybatis
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" ?>
<mapper class="MyMapper">
    <query id="testSelect" resultType="House">
        SELECT col1,col2,col3 from myTable
        <if test="filter != null">
            where
            1=1
            <if test="filter.min_col2 != null">
                and col2 &gt; #{filter.min_col2}
            </if>
            <if test="filter.max_col2 != null">
                and col2 &lt; #{filter.min_col2}
            </if>
            <if test="filter.col3_options != null">
                and
                <foreach collection="filter.col3_options" separator=" or " open="(" close=")" item="option">
                    ABS(col3 - #{option}) &lt; 4
                </foreach>

            </if>
        </if>

    </query>

</mapper>
{% endhighlight %}

初步准备支持`<resultMap>` `<query>` `<if>` `<foreach>`标签以及`${}` `#{}`占位符,提供基础的Sql渲染功能

### protocol
EZSql的调用方法和Mybatis十分类似,只需要定义一个和Java中interface作用类似的protocol

{% highlight objectivec %}
@protocol MyMapper <EZSqlMapper>
- (Model *)testSelect:(NSDictionary *)params; //第一个参数前的方法名必须和xml中query的id对应
- (void)itemWithNoParams;
@end
{% endhighlight %}

使用工厂方法获得Mapper
{% highlight objectivec %}
    id <MyMapper> mapper = [EZSMapperFactory mapperWithProtocol:@protocol(MyMapper)];
{% endhighlight %}

即可调用protocol的方法了
{% highlight objectivec %}
    [mapper itemWithNoParams];
{% endhighlight %}

## 结构设计

EZSql分为`Parser`, `Renderer`, `ResultMap`, `Mapper`, `MapperFactory`几个主要对象.

### Parser

顾名思义,Parser提供对xml文件的解析,生成ResultMap对象和Renderer对象的不同子类,以供Mapper使用

### Renderer

`Renderer`及其子类负责对SQL语句的具体渲染, 其公有方法为`- (NSString *)renderWithParams:(NSDictionary *)`,具体渲染流程如图

以`"SELECT * FROM ${table} WHERE col1 = #{text}"`为例,传入键值对

{% highlight objectivec %}
    @{
      @"table":@"table1",
      @"text":@"text1"
    }
{% endhighlight %}

![](http://img.nufe-cst.cn/ezsqlrendererpool.png)

即针对不同的pattern生成不同的`Renderer`对象,由`Mapper`进行线性调取,即可完成一次SQL语句的拼接工作.

### ResultMap

`ResultMap`用于将SQL输出的键值对转换成Model对象.功能类似YYModel.

### Mapper

`Mapper`用于实际消息的转发, 每个XML文件对应一个单例的`Mapper`,收到消息后进行对应调用并输出结果.

### MapperFactory

`MapperFactory`用于Mapper类的创建, 即动态生成Mapper的子类并创建实例进行缓存,这是为了解决不同`Protocol`可能存在同名方法的问题,所以针对每个`Protocol`, MapperFactory`会调用runtime方法进行Mapper子类的生成. 具体Protocol定义的消息由Mapper进行识别.

-----

从零开始写框架任重道远. 我会将开发日志在博客上更新, 源码已经托管到[Github](https://github.com/ObesityChow/EZSql),期待有大牛指导我这个小开发仔
