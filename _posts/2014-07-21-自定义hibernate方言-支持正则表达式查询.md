title: 自定义hibernate方言 支持正则表达式查询
date: 2014-07-21 20:00:18
tags:
- JAVA
- Hibernate
comments: true
---

#自定义hibernate方言 支持正则表达式查询

##问题的引出
最近收到一个新的需求,在基于数据库查询中,需要支持更高级的正则表达式的查询.但是这个在Hibernate的HQL中是不支持的.那么就需要想办法支持这个功能.

##解决过程
* 首先需要明确的是查询数据库是否支持正则表达式的查询. 经过网上的查询.发现我们系统支持的`Mysql5.5`以及`Oracle10g`都是支持正则表达式查询的,但是两个的语法稍微有点不一样.

**通用正则表达式语法**:

|符号|意义|
|:--:|:--:|
| `.` |表示匹配任意一个字符|
|`[ ]`|匹配任何单一字符|
|`[0-9]`|表示一个字符的范围| 
|`^` |否定一个字符集合，将匹配除指定字符外的任何东西|
|`\\`|匹配特殊字符转义|
|`*`|0个或多个匹配|
|`+`|1个或多个匹配（等于 {1, }）|
|`?`|0个或1个匹配（等于 {0, 1}）|
|`{n}`|指定数目的匹配|
|`{n, }`|不少于指定数目的匹配|
|`{n ,m}`|匹配数目的范围|

<!--more-->

**标准类别:**

|符号|意义|
|:--:|:--:|
|[:alnum:]|文字数字字符|
|[:alpha:]|文字字符|
|[:blank:]|空白字符|
|[:cntrl:]|控制字符|
|[:digit:]|数字字符|
|[:graph:]|图形字符|
|[:lower:]|小写文字字符|
|[:print:]|图形或空格字符|
|[:punct:]|标点字符|
|[:space:]|空格、制表符、新行、和回车|
|[:upper:]|大写文字字符|
|[:xdigit:]|十六进制数字字符|

**Mysql**: mysql的正则表达式查询是以关键字`REGEXP`和`NOT REGEXP`做标识的.后面的语法和一般的正则表达式是一样的.

它们的用法是:

```sql
SELECT 'Monty!' REGEXP '.*';
SELECT 'new*\n*line' REGEXP 'new\\*.\\*line';
SELECT 'a' REGEXP '^[a-d]';
SELECT 'justalnums' REGEXP '[[:alnum:]]+';
SELECT '!!' REGEXP '[[:alnum:]]+'; 
```
具体更多信息可以参见:[Regular Expressions](https://dev.mysql.com/doc/refman/5.1/en/regexp.html)

**Oracle**:oracle的正则表达式查询是以关键字`REGEXP_LIKE`和`NOT REGEXP_LIKE`做标识的.

它们的用法是:

```sql
select * from TestTable where REGEXP_LIKE ( name, '^\(\d{3}\) \d{3}-\d{4}$' )
select * from TestTable where REGEXP_LIKE ( name, 'new\\*.\\*line' )
select * from TestTable where REGEXP_LIKE ( name, '[[:alnum:]]+' )
```
具体更多信息可以参见:[Using Regular Expressions in Oracle Database](http://docs.oracle.com/cd/B19306_01/appdev.102/b14251/adfns_regexp.htm)

* 分析Hibernate解析HQL的地方.通过断点可以发现,HQL转换成SQL的时候,Hibernate的Dialect方言是起了关键的作用的.它注册了HQL的各种`字段类型``内置函数`以及它们转换成对应数据库SQL的方法.
* `org.hibernate.dialect.Dialect` 这个类主要有 `register*`开头的几个方法.用于注册各种`字段类型``内置函数``关键字`等等.
* 我们需要的就是继承某一数据库的Dialect.然后在构造函数中加上我们需要的自定义的`字段类型``内置函数``关键字`
* 比如在**Mysql**中增加正者表达式,就需要继承`MySQL5Dialect`.然后在构造函数中进行注册:

```java
public class CustomMySQL5Dialect extends MySQL5Dialect{

    public CustomMySQL5Dialect() {
        super();
        /*通过自定义MYSQL的方言. 增加函数的支持
        * 这个的意思就是 当遇到 regexp(name,'rain')的时候,自动的转换成  name REGEXP 'rain'  这句SQL
         * 再比如  regexp(name,'[[:alnum:]]+rain[[:alnum:]]+|[[:alnum:]]+rain|rain[[:alnum:]]+').就转换成
         *  name REGEXP '[[:alnum:]]+rain[[:alnum:]]+|[[:alnum:]]+rain|rain[[:alnum:]]+' 这句SQL.然后返回是一个布尔值
         *  又因为hibernate不支持自定义的检索函数无返回值. 所以 写的时候 必须 写成 where regexp(name,'rain')=true
         *  否则解析会报错
         *  */
        registerFunction("regexp", new StandardSQLFunction(StandardBasicTypes.BOOLEAN, "?1 REGEXP '?2'"));

        registerFunction("noregexp", new StandardSQLFunction(StandardBasicTypes.BOOLEAN, "?1 NOT REGEXP '?2'"));

        /*这个地方替换成了自定义的SQLFunction.详见@see com.sobey.database.hibernate.CustomExactSearchSQLFunction*/
        /*精确查询*/
        registerFunction("exactsearch",new StandardSQLFunction(StandardBasicTypes.BOOLEAN,"( ?1 REGEXP '?2' AND ?1 NOT REGEXP '[[:alnum:][:punct:]]+?2[[:alnum:][:punct:]]+|[[:alnum:][:punct:]]+?2|?2[[:alnum:][:punct:]]+' )"));
    }
}
```

* 在**Oracle**中增加正则表达式,需要继承`Oracle10gDialect`:

```java
public class CustomOracleDialect  extends Oracle10gDialect{

    public CustomOracleDialect() {
        super();

        /*2014-10-20 14:54:28 由于默认情况下 hibernate会把 text映射成为Oracle中的 long型.而不是text类型.
         * 而long是oracle废弃了的,一个表只能有一个long字段.因此,就造成了表表不起的问题.因此就需要我们手动的映射到clob上 */
        registerColumnType(-1, "clob");         //Types.LONGVARCHAR
        registerColumnType(-16, "clob");        //Types.LONGNVARCHAR
        
        /*通过自定义Oracle的方言. 增加函数的支持
        * 这个的意思就是 当遇到 regexp(name,'rain')的时候,自动的转换成   REGEXP_LIKE (name,'rain')  这句SQL
         * 再比如  regexp(name,'[[:alnum:]]+rain[[:alnum:]]+|[[:alnum:]]+rain|rain[[:alnum:]]+').就转换成
         *  name REGEXP '[[:alnum:]]+rain[[:alnum:]]+|[[:alnum:]]+rain|rain[[:alnum:]]+' 这句SQL.然后返回是一个布尔值
         *  又因为hibernate不支持自定义的检索函数无返回值. 所以 写的时候 必须 写成 where regexp(name,'rain')=true
         *  否则解析会报错
         *  */
        registerFunction("regexp", new StandardSQLFunction(StandardBasicTypes.BOOLEAN, "REGEXP_LIKE(?1,'?2')"));

        registerFunction("noregexp", new StandardSQLFunction(StandardBasicTypes.BOOLEAN, "NOT REGEXP_LIKE(?1,'?2')"));

        /*精确查询*/
        registerFunction("exactsearch",new StandardSQLFunction(StandardBasicTypes.BOOLEAN,"(REGEXP_LIKE(?1,'?2') AND NOT REGEXP_LIKE(?1,'[[:alnum:][:punct:]]+?2[[:alnum:][:punct:]]+|[[:alnum:][:punct:]]+?2|?2[[:alnum:][:punct:]]+'))"));

    }
}
```

* 在上述的表达式中 `?1` `?2` 分别表示的是 第一个入参和第二个入参.因为我们定义了**REGEXP**操作是一个**二元操作符**.需要传入字段名以及正则表达式规则.

* 这样自定义后.在配置Hibernate的时候直接指定方言为我们自定义的Dialect:

```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate3.LocalSessionFactoryBean">
		<!-- 引入dataSource -->
		<property name="dataSource" ref="dataSource" />
		<property name="hibernateProperties">
			<props>
				<prop key="hibernate.dialect">xxx.CustomOracleDialect</prop>
				<prop key="hibernate.show_sql">true</prop>
				<prop key="hibernate.format_sql">false</prop>
				...
			</props>
		</property>
</bean>
```

* 这样就可以在HQL中使用我们新定义的函数了.比如:

```sql
from Test where exactsearch(name,'new\\*.\\*line')=true;
```
**注意**这里调用自定义函数由于是`StandardBasicTypes.BOOLEAN`.因此返回值需要和一个布尔值进行对比.这样才不会解析报错.

