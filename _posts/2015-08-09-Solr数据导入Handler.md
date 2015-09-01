---
title: <翻译>Solr 数据导入Handler
date: 2015-08-09
tags:
- JAVA
- Lucene
- Solr
comments: true
---

Solr 数据导入Handler
---

由于在我们新的全文检索引擎中使用了Solr作为底层,因此当时是想直接使用Solr提供的DataImportHandler来进行数据的导入,于是就有在它的官网上看了相关的内容,并且试着进行了翻译.虽然最后我们还是自己实现了一套数据导入,但是它的这篇文章写的确实非常的好.

Data Import Request Handler
---
大多数系统都会把他们的数据存储在关系型数据库或者XML文件中,并且还要对这些数据进行检索.`DataImportHandler`就是Solr提供的一个工具:可以通过配置的方式驱动工具全量(`full builds`)或增量的把数据导入到Solr中.

除了这篇文章外,还可以参见[DataImportHandlerFaq](https://wiki.apache.org/solr/DataImportHandlerFaq).如果想快速开始,可以浏览[DIHQuickStart](https://wiki.apache.org/solr/DIHQuickStart)

#概况
##目标

* 能从关系型数据库中读取数据
* 能通过配置从多个表以及列数据中构造Solr的文档(`Document`)
* 能更新上述的Solr文档
* 能根据配置提供全导入的功能
* 能检测增量的数据新增与更新,并做增量导入(我们假设存在一个最后更新时间的时间戳字段来保证增量的工作)
* 能自动的计划与调度全更新与增量更新
* 能通过配置从xml/http/file 中读取数据并创建索引
* 能提供插件机制来保证用户可以选择任何的数据源(ftp,scp等等)和格式(JSON,csv等等)进行索引工作.

<!--more-->

#设计概况
要使用这个`Handler`,就必须在`solrconfig.xml`文件中做如下的注册:

```xml
<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
      <str name="config">/home/username/data-config.xml</str>
    </lst>
  </requestHandler>
```
顾名思义,它是一个[SolrRequestHandler](https://wiki.apache.org/solr/SolrRequestHandler)的实现.它可以从两个地方进行配置:

* `solrconfig.xml`.可以在这里增加数据配置文件的位置
* 数据源也可以配置在这里,或者直接配置在`data-config.xml`文件中
* `data-config.xml`
    * 如何获取数据(查询,Url等等)
    * 读取什么数据(结果集(`resultset`)列,`xml`的字段等等)
    * 如果处理数据(修改/新增/删除 字段)

#从关系型数据库导入的用法
为了使用这个Handler,需要参照以下几个步骤:

* 定义一个`data-config.xml`文件,并且在`solrconfig.xml`文件的`DataImportHandler`小节指定它的存放位置
* 定义数据源连接信息
* 打开`http://localhost:8983/solr/dataimport`网址以验证`DataImportHandler`的所有东西都正常工作
* 使用全导入命令(`full-import`)执行从数据库到Solr索引的全导入操作
* 使用增量导入命令(`delta-import`)执行Solr索引的增量导入(获取新的 插入与更新)

##配置数据源
在`dataConfig`标签下直接增加`dataSource`标签

```xml
<dataSource type="JdbcDataSource" driver="com.mysql.jdbc.Driver" url="jdbc:mysql://localhost/dbname" user="db_username" password="db_password"/>
```

* 数据源的配置也可以直接在`solrConfig.xml`中配置,这个可以参照[#solrconfigdatasource](#solrconfigdatasource)
* `type`属性是用于定义数据源实现类的,是选填值,默认实现是`JdbcDataSource`
* 当有多个数据源的时候,`name`属性可以用来当数据源的标识.
* 剩下的其他属性全是特定的数据源实现所特有的.
* 如果你要自己实现数据源插件,可以看[这里](https://wiki.apache.org/solr/DataImportHandler#datasource)

###Oracle数据库实例
首先你需要下载并安装[Oralce JDBC驱动](http://www.oracle.com/technetwork/database/features/jdbc/index-091264.html)到Solr安装目录的`lib`文件夹下.

```xml
<dataSource name="jdbc" driver="oracle.jdbc.driver.OracleDriver" url="jdbc:oracle:thin:@//hostname:port/SID" user="db_username" password="db_password"/>
```

###多数据源
在实现中有一种可能就是需要不止一个数据源.为了配置多个数据源,需要再配置多个`dataSource`标签.这时数据源的`name`属性就有意义了.当有多个数据源的时候,每一个数据源的`name`属性都不能重复,是唯一的.比如:

```xml
<dataSource type="JdbcDataSource" name="ds-1" driver="com.mysql.jdbc.Driver" url="jdbc:mysql://db1-host/dbname" user="db_username" password="db_password"/>
<dataSource type="JdbcDataSource" name="ds-2" driver="com.mysql.jdbc.Driver" url="jdbc:mysql://db2-host/dbname" user="db_username" password="db_password"/>
```
在你的实体(`entityies`)中这样引用:

```xml
..
<entity name="one" dataSource="ds-1" ...>
   ..
</entity>
<entity name="two" dataSource="ds-2" ...>
   ..
</entity>
..
```

##配置JDBC数据源
`JdbcDataSource` 允许有以下属性:

* **driver**(必填):Jdbc驱动的类名
* **url**(必填):jdbc链接的url地址.(当使用jndi连接的时候,不是必填的.)
* **user**:用户名
* **password**:用户密码
* **jndiName**:当使用jndi连接数据库的时候的jndi名称
* **batchSize**:jdbc连接的批量提交大小.使用`-1`可以防止`setFetchSize()`方法抛出异常
* **convertType**:(true/false)自动读取目标Solr中的数据类型,默认是`false`
* **autoCommit**:自动提交,如果设置为`false`,那么它会调用`setAutoCommit(false)`. 详细参见[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
* **readOnly**:如果设置这个为`true`,那么它在连接的时候会调用`setReadOnly(true)`,`setAutoCommit(true)`,`setTransactionIsolation(TRANSACTION_READ_UNCOMMITTED)`,`setHoldability(CLOSE_CURSORS_AT_COMMIT)`.详细参见[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
* **transactionIsolation**:事务级别,可以有以下几个值:[`TRANSACTION_READ_UNCOMMITTED`, `TRANSACTION_READ_COMMITTED`, `TRANSACTION_REPEATABLE_READ`,`TRANSACTION_SERIALIZABLE`,`TRANSACTION_NONE`].详细参见[Solr1.4](https://wiki.apache.org/solr/Solr1.4)

直接放入`dataSource`标签的其他标签会直接被传入到JDBC驱动

##在data-config.xml中进行配置
一个Solr的文档可以被认为是一个去标准化的、拥有多个字段、其值来自不同表的`schema`.

`data-config.xml`首先定义了一个`Document`元素.一个`document`代表了一种文档.一个文档包含了一个或多个根实体(`root entity`).一个根实体又可以包含多个子实体,这些子实体反过来又可以再包含其他的实体.一个实体就是关系数据库中的一个表或试图.每一个实体都可以包含多个字段.每一个字段相当于关系型数据库查询语句的结果中的返回字段.对于每一个字段,都是查询结果集中的列名.如果结果集的列名与Solr的字段名不相同,那么就需要填写`name`属性.而其他的必要属性比如`type`可以直接从`SolrSchema.xml`中继承(当然,也可以被重写)

为了从数据库中获取数据,我们的设计理念围绕着为每一个实体都提供给用户**模板化的SQL**(`templatized sql`).它能当用户需要时,提供给用户完整的`SQL`功能.而根实体就是其字段能被其他子实体所关联的中心表.

###data-config的Schema定义
`dataconfig`并没有严格的schema定义.实体与字段的属性取决于它们依赖的`processeor`处理器和`transformer`转换器

一个实体默认的属性都有:

* **name**(必填):用于标识一个实体的唯一名字
* **processor**:当数据源不是关系型数据库时必填.默认值是`SqlEntityProcessor`
* **transformer**:被实体引用的转换器(详细见转换器章节)
* **dataSource**:实体所使用的数据源名称(当有多个数据源的时候使用)
* **threads**:设置是否使用多线程来进行实体的提取,这个值必须被设置在根实体上.详细可以参见[Solr3.1](https://wiki.apache.org/solr/Solr3.1)
    * 注意:并不是所有的DataImportHandler组件都是线程安全的.如果要使用这个特性,那么请确保测试完全!
    * 与多线程相关的显著BUG已经在[Solr3.6](https://wiki.apache.org/solr/Solr3.6)中被修复.如果在旧的版本上使用这个特性,建议升级版本.请看Solr3.6.0的[SOLR-3011](https://issues.apache.org/jira/browse/SOLR-3011).你也需要从[SOLR-3360](https://issues.apache.org/jira/browse/SOLR-3360)开始修复
    * `threads`属性已经在[Solr3.6](https://wiki.apache.org/solr/Solr3.6)中被废弃,并且会在[Solr4.0](https://wiki.apache.org/solr/Solr4.0)中被删除.详细可以参见[SOLR-3262](https://issues.apache.org/jira/browse/SOLR-3262)
    * **pk**:实体的主键字段.这个属性是**可选**的,并且只有在增量更新中才有用.它并没有与`schema.xml`中定义的`uniqueKey`字段相关联,虽然他们两个可以是相同的.
    * **rootEntity**:在默认情况下,所有在`document`下的实体都是根实体.如果这个值被设置为`false`,只有直接在`document`下的实体才会被认为是实体.根实体所有返回的列都会被Solr创建成一个文档.
    * **onError**:(abort|skip|continue).默认值是`abort`.`skip`会跳过当前的文档.`continue`会继续处理,就当错误没有发生.[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
    * **preImportDeleteQuery**:当全导入执行前会使用这个属性的值来清理索引,而不是使用`*:*`来清除原有的索引.这个属性只能赋值给`document`节点下的直接根实体.
    * **postImportDeleteQuery**:当全导入执行完成后会使用这个属性的值来再执行一次.同样,这个属性只能赋值给`document`节点下的直接根实体.

`SqlEntityProcessor`这个实体节点的属性有:

* **query**(必填):查询数据库所执行的SQL字符串
* **deltaQuery**:当增量导入时所使用的查询语句
* **parentDeltaQuery**:当增量导入时所使用的查询语句
* **deletedPkQuery**:当增量导入时所使用的查询语句
* **deltaImportQuery**:(当增量导入时所使用的查询语句).如果不提供这个语句,DIH会在识别增量后尝试构建查询语句(这是非常容易出错的).命名空间`${dih.delta.<column-name>}`可以在这个查询语句中使用,比如:`select * from tbl where id=${dih.delta.id}`. [Solr1.4](https://wiki.apache.org/solr/Solr1.4)

##操作命令
DIH提供了基于HTTP请求的API.这些就是他能执行的操作:

* full-import:当请求`http://<host>:<port>/solr/dataimport?command=full-import`地址的时候,全导入操作会开始执行.
    * 这个操作会在一个新的线程中被执行.并且相应中的`status`属性会显示为`busy`.
    * 这个操作所执行的时间取决于数据的大小.
    * 当全导入命令被执行时,它会在`conf/dataimpoort.properties`文件([这个文件可以进行配置](https://wiki.apache.org/solr/DataImportHandler#Configuring_The_Property_Writer))中记录操作开始的时间
    * 存储下来的时间戳会在增量导入执行的时候被使用
    * 在全导入的过程中,Solr的查询不会被锁定
    * 它拥有几个扩展的参数:
        * entity:直接定义在`document`节点下的实体的名字.使用这个参数可以有选择性的执行一个或多个实体的导入,设置多个`entity`参数可以一次性的执行多个实体的导入.如果没有这个参数,所有的实体都会被导入.
        * clean:(默认是true).在执行全导入前,先清理一次索引.
        * commit:(默认是true).当操作之后,提交文档
        * optimize:(在Solr3.6之前是true,以后是false).当全导入之后,执行整理的操作.请注意:这是一个非常昂贵的操作.并且通常在增量导入时是无意义的.
        * debug:(默认是false).使用调试模式执行,它在开发模式下可以被使用([参见这里](https://wiki.apache.org/solr/DataImportHandler#interactive))
            * 请注意,在调试模式下,文档不会被自动的提交.如果你想在调试模式下提交结果.那么需要显式的在请求中增加`commit=true`
* delta-import: 为了增量的导入并且检测数据变化.可以执行`http://<host>:<port>/solr/dataimport?command=delta-import`命令.它同样的支持`clean` `commit` `optimize`和`debug`参数
* status: 执行`http://<host>:<port>/solr/dataimport`这个命令用来返回当前的执行状态.它会提供一份详细的统计数据,记录了`创建`、`删除` `查询数` `结果集` `状态`等等数据.
* reload-config:当`data-config`文件被改变了,并且你想不重启Solr而重新加载这个文件的时候,执行`http://<host>:<port>/solr/dataimport?command=reload-config`
* abort:使用`http://<host>:<port>/solr/dataimport?command=abort`这个来中止数据的导入

##全导入实例
让我们来看一个例子,假如在数据库中我们有如下的定义:
![](https://wiki.apache.org/solr/DataImportHandler?action=AttachFile&do=get&target=example-schema.png)
这是一个可以满足Solr Schema定义的关系模型.我们会使用这个例子来给`DataImportHandler`创建一个`data-config.xml`.我们使用[HSQLDB](http://hsqldb.org/)数据库来创建这个Schema.要运行它,请执行以下的步骤:

1. 找到solr下载文件夹中的`example/example-DIH`目录.它包含了一个完整的配置好了的RSS操作例子(稍后会说明)的Solr实例
2. 使用`example-DIH/solr`目录作为你的Solr主目录.在`/examples`目录下执行`java -Dsolr.solr.home="./example-DIH/solr/" -jar start.jar`来启动你的Solr
3. 访问[http://localhost:8983/solr/db/dataimport](http://localhost:8983/solr/db/dataimport)来验证配置是否正确
4. 访问[http://localhost:8983/solr/db/dataimport?command=full-import](http://localhost:8983/solr/db/dataimport?command=full-import)开始执行全导入

这个Solr的目录是一个多核([MultiCore](https://wiki.apache.org/solr/MultiCore))的Solr.它包含了两个核心,一个是这个例子所示的数据库例子,另一个是一个RSS导入的例子(新特性)

* 这个例子中的`data-config.xml`配置如下:

    ```xml
    <dataConfig>
<dataSource driver="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:/temp/example/ex" user="sa" />
    <document name="products">
        <entity name="item" query="select * from item">
            <field column="ID" name="id" />
            <field column="NAME" name="name" />
            <field column="MANU" name="manu" />
            <field column="WEIGHT" name="weight" />
            <field column="PRICE" name="price" />
            <field column="POPULARITY" name="popularity" />
            <field column="INSTOCK" name="inStock" />
            <field column="INCLUDES" name="includes" />

            <entity name="feature" query="select description from feature where item_id='${item.ID}'">
                <field name="features" column="description" />
            </entity>
            <entity name="item_category" query="select CATEGORY_ID from item_category where item_id='${item.ID}'">
                <entity name="category" query="select description from category where id = '${item_category.CATEGORY_ID}'">
                    <field column="description" name="cat" />
                </entity>
            </entity>
        </entity>
    </document>
</dataConfig>
    ```
在这里,根实体是一个名叫`item`的表,它的主键字段名为`id`.数据可以通过`select * from item`这个语句查询.每一个实体拥有多个`features`,这些值是从表`feature`中的`description`字段来的.`feature`实体的查询语句是:

```xml
<entity name="feature" query="select description from feature where item_id='${item.id}'">
       <field name="feature" column="description" />
   </entity>
```
在表`feature`中的外键`item_id`字段关联了`item`表的主键`id`字段为*item*的每一行数据获取多行*feature*数据.按照类似的方式,我们可以关联`item`和`category`(这是一个多对多关系).需要注意,我们是如何使用模板SQL来通过中间表`item_category`来关联两张表记录的:

```xml
<entity name="item_category" query="select category_id from item_category where item_id='${item.id}'">
                <entity name="category" query="select description from category where id = '${item_category.category_id}'">
                    <field column="description" name="cat" />
                </entity>
            </entity>
```

###一个更短的dataConfig配置
在上一个例子中,它映射了多个数据库字段到Solr的字段中.而还有一种可能是我们会映射所有的数据库字段到Solr字段中,并且它们字段的名字都是相同的(忽略大小写).也许当任何内置的转换器被使用的时候(见 转换器章节),你才需要添加一个字段.  
一个更短的版本例子如下:

```xml
<dataConfig>
    <dataSource driver="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:/temp/example/ex" user="sa" />
    <document>
        <entity name="item" query="select * from item">
            <entity name="feature" query="select description as features from feature where item_id='${item.ID}'"/>
            <entity name="item_category" query="select CATEGORY_ID from item_category where item_id='${item.ID}'">
                <entity name="category" query="select description as cat from category where id = '${item_category.CATEGORY_ID}'"/>
            </entity>
        </entity>
    </document>
</dataConfig>
```

##使用增量导入命令
当访问[http://localhost:8983/solr/dataimport?command=delta-import](http://localhost:8983/solr/dataimport?command=delta-import)地址的时候,增量导入操作会被执行.这个操作会在一个新的线程开始执行,并且相应中的`status`属性会显示`busy`.取决于你数据的大小,这个操作会花费一些时间.在任何时候,你都可以访问[http://localhost:8983/solr/dataimport](http://localhost:8983/solr/dataimport)来查看当前的状态.

当增量导入命令被执行的时候,它会从`conf/dataimport.properties`文件中读取开始的时间.这个时间会被用于执行增量查询,并且当完成增量导入后,会更新`conf/dataimport.properties`文件中的时间戳.

**提示**:在Solr中,增量导入是一种比全导入跟有效率的方式,但是它需要一些少量的配置:[DataImportHandlerDeltaQueryViaFullImport](https://wiki.apache.org/solr/DataImportHandlerDeltaQueryViaFullImport)

###增量导入的实例
我们会使用与全导入相同的数据库例子.请注意,这个例子中的数据库schema已经被更新了,在每一个表中现在都包含了一个`last_modified`的时间戳字段.你可能需要重新下载数据库,因为它最近已经更新了.我们使用这个时间戳字段来决定每一个表中的哪些记录自从上一次被索引后有所变动.

请看一下一下的data-config.xml配置

```xml
<dataConfig>
    <dataSource driver="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:/temp/example/ex" user="sa" />
    <document name="products">
        <entity name="item" pk="ID"
                query="select * from item"
                deltaImportQuery="select * from item where ID='${dih.delta.id}'"
                deltaQuery="select id from item where last_modified &gt; '${dih.last_index_time}'">
            <entity name="feature" pk="ITEM_ID"
                    query="select description as features from feature where item_id='${item.ID}'">
            </entity>
            <entity name="item_category" pk="ITEM_ID, CATEGORY_ID"
                    query="select CATEGORY_ID from item_category where ITEM_ID='${item.ID}'">
                <entity name="category" pk="ID"
                       query="select description as cat from category where id = '${item_category.CATEGORY_ID}'">
                </entity>
            </entity>
        </entity>
    </document>
</dataConfig>
```
需要注意`deltaQuery`属性,这个属性拥有一个能够判定*item*表变化的SQL语句.同时注意变量`${dataimporter.last_index_time}`,这是一个`DataImportHandler`提供的变量,用于表示上一次全更新或增量更新运行的时间戳.你可以在`data-config.xml`文件中的任意SQL中使用这个变量,它会在执行中被替换成真正的`last_index_time`时间.

**注意**:

* 上一个例子中的`deltaQuery`语句只用来监听了*item*表的变化,而没有监听其他表.你其实可以监听通过下面的一条SQL查询来监听所有关联的子表的变化.搞清楚它们的细节是留给各位的一个练习:

    ```sql
select id from item where id in
                                (select item_id as id from feature where last_modified > '${dih.last_index_time}')
                                or id in
                                (select item_id as id from item_category where item_id in
                                    (select id as item_id from category where last_modified > '${dih.last_index_time}')
                                or last_modified &gt; '${dih.last_index_time}')
                                or last_modified &gt; '${dih.last_index_time}'
    ```

* 书写一个如上面那样复杂的`deltaQuery`查询语句并不是一个愉快的任务,所以我们提供了一套其他的方法来完成相同的目标.

    ```xml
<dataConfig>
    <dataSource driver="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:/temp/example/ex" user="sa" />
    <document>
            <entity name="item" pk="ID" query="select * from item"
                deltaImportQuery="select * from item where ID=='${dih.delta.id}'"
                deltaQuery="select id from item where last_modified &gt; '${dih.last_index_time}'">
                <entity name="feature" pk="ITEM_ID"
                    query="select DESCRIPTION as features from FEATURE where ITEM_ID='${item.ID}'"
                    deltaQuery="select ITEM_ID from FEATURE where last_modified > '${dih.last_index_time}'"
                    parentDeltaQuery="select ID from item where ID=${feature.ITEM_ID}"/>


            <entity name="item_category" pk="ITEM_ID, CATEGORY_ID"
                    query="select CATEGORY_ID from item_category where ITEM_ID='${item.ID}'"
                    deltaQuery="select ITEM_ID, CATEGORY_ID from item_category where last_modified > '${dih.last_index_time}'"
                    parentDeltaQuery="select ID from item where ID=${item_category.ITEM_ID}">
                <entity name="category" pk="ID"
                        query="select DESCRIPTION as cat from category where ID = '${item_category.CATEGORY_ID}'"
                        deltaQuery="select ID from category where last_modified &gt; '${dih.last_index_time}'"
                        parentDeltaQuery="select ITEM_ID, CATEGORY_ID from item_category where CATEGORY_ID=${category.ID}"/>
            </entity>
        </entity>
    </document>
</dataConfig>
    ```
在这里我们为除了根的每一个实体(其实就只有两个)指定了三个查询语句.

* `query`属性为Solr文档的全导入提供了所需要的字段.
* `deltaImportQuery`属性为增量导入提供了所需要的字段.
* `deltaQuery`属性提供了从上一次索引以来变化了的实体的主键
* `parentDeltaQuery`属性使用当前变化了的表中的记录(依赖`deltaQuery`查询)来提供给父表的变化的记录中.这个属性是必须的,因为当一个子表中的记录被变化的时候,我们需要重新生成包含这个字段的文档的索引

让我们再重申一下结论:

* 通过`query`提供的每一行数据,这个查询语句是每一个子实体都会被调用一次的.
* 通过`deltaQuery`提供的每一行数据,`parentDeltaQuery`会被执行
* 如果根实体或子实体中任何一个数据被改变,我们都会重新生成包含这些记录的Solr的文档

注意:`deltaImportQuery`是Solr1.4才有的特性.最初这个语句会根据`query`属性自动的生成,但是错误率很高.并且有可能会使用全导入的命令去做增量导入. 详细请见[这里](http://wiki.apache.org/solr/DataImportHandlerFaq#fullimportdelta)

在[Solr3.1](https://wiki.apache.org/solr/Solr3.1)中,这个Handler会检测以确认在所有的查询中,你提供的主键字段是在结果集中.在一种情况下,要求在从1.4升级到3.1的时候给主键字段一个SQL的别名`did`:

```sql
SELECT MAX(did) FROM ${dataimporter.request.dataView}
```
需要被改成:

```sql
SELECT MAX(did) AS did FROM ${dataimporter.request.dataView}
```

#配置属性写入器
[Solr4.1](https://wiki.apache.org/solr/Solr4.1)中,在`dataConfig`节点下新增加了一个`propertyWriter`节点.属性`last_index_time`的值会被这个处理器转换成文本并且保存到`properties`文件中.并且会在下一次的导入时被当做`${dih.last_index_time}`变量被使用.这一个节点就是用来描述如何控制属性值写入`properties`文件的. 

```xml
<propertyWriter dateFormat="yyyy-MM-dd HH:mm:ss" type="SimplePropertiesWriter" directory="data" filename="my_dih.properties" locale="en_US" />
```

* 首先,这个节点是可选的,它拥有默认的语言环境、目录和文件名.`type`属性在单机情况下默认为[SimplePropertiesWriter](https://wiki.apache.org/solr/SimplePropertiesWriter).而在[SolrCloud](https://wiki.apache.org/solr/SolrCloud)情况下,默认是`ZKPropertiesWriter`
* `type`:这个节点是表示写入器的实现类,除非整个`<propertyWriter/>`都被忽略,否则它是必填的.
* `filename`:(`SimplePropertiesWriter`)默认的名字就是DIH请求处理器的名字然后加上`.properties`后缀,比如:`dataimport.properties`
* `directory`:(`SimplePropertiesWriter`) 它的默认值是`conf`目录
* `dateFormat`:(`SimplePropertiesWriter`/`ZKPropertiesWriter`)使用`java.text.SimpleDateFormat`的正则表达式来把日期转换为文本.默认值是`yyyy-MM-dd HH:mm:ss`.而对于JDBC的转义语法,需要使用`{'ts' yyyy-MM-dd HH:mm:ss}`
* `locale`:(`SimplePropertiesWriter`/`ZKPropertiesWriter`)在Solr4.1中,默认的语言环境是根语言环境.这一点与Solr4.0以及之前都不同.之前的版本始终使用的是机器的默认语言环境

#使用XML/HTTP数据源
`DataImportHandler`同样能使用基于HTTP的数据源来创建索引.包括从`REST/XML`的API和`RSS/ATOM`的订阅来创建索引.

##配置URLDataSource或HttpDataSource数据源
在[Solr1.4](https://wiki.apache.org/solr/Solr1.4)开始,`HttpDataSource`已经由`URLDataSource`所替代而被废弃.

一个简单的配置在`dataConfig.xml`中的`URLDataSource`和`HttpDataSource`的配置样例如下所示:

```xml
<dataSource name="b" type="!HttpDataSource" baseUrl="http://host:port/" encoding="UTF-8" connectionTimeout="5000" readTimeout="10000"/>
<!-- or in Solr 1.4-->
<dataSource name="a" type="URLDataSource" baseUrl="http://host:port/" encoding="UTF-8" connectionTimeout="5000" readTimeout="10000"/>
```
它们所拥有的扩展属性有这些:

* **baseUrl**(可选):当在开发、测试或者生产环境下有不同的IP地址和端口的时候你需要使用这个属性来把变化隔离在`solrconfig.xml`文件中.
* **encoding**(可选):在默认情况下编码的值同响应头上的编码是相同的,但是你可以使用这个属性来覆盖它.
* **connectionTimeout**(可选):默认的超时时间为5000毫秒
* **readTimeout**(可选):默认的读取超时时间为10000毫秒

##在data-config.xml文件中进行配置
一个由`xml/http`数据源提供的实体除了上面所说的默认属性外,还有以下这些属性:

* processor(必须):并且其值必须是`XPathEntityProcessor`
* url(必须):这个url地址会被用来调用REST API.(可以模板化).如果这个数据源是一个文件,那么这个属性就必须是文件的路径.
* stream(可选):当xml非常大的时候,可以设置这个为`true`
* forEach(必须):它是一个用来圈定一条记录范围的`XPATH`表达式,如果存在多种记录的类型,那么使用`/`来分隔.当`useSolrAddSchema`被设置为`true`的时候,这个属性可以被忽略
* xsl(可选):提供一个XSL文件的全路径或者url用来在生成索引前使用XSL来转换来源数据.
* useSolrAddSchema(可选):如果数据源提供的数据与solr要增加的数据具有相同的schema结构,那么久没必要再指定任何字段了,只需要把它设置成true就可以了.
* flatten(可选):当这个值被设置为`true`的时候,所有xml节点下的值都会被抽取到一个字段中,而不会考虑它们节点的名字.

而实体的每一个字段除了上文提到的标准属性外,还有以下的属性:

* **xpath**(可选):这个一个xpath表达式,用于描述在记录中的每一个字段是如何被映射到索引的字段上的.如果该字段不是来自xml中的某个节点值(有可能是通过`transformer`转换器生成的),那么这个属性是可以被忽略的.如果索引的一个字段在`schema`中被标记为`multivalued`,并且提供的xpath表达式也能返回多个值的话.那么,不需要任何额外的配置,它就能够被`XPathEntityProcessor`自动的处理.
* **commonField**:可以是`true`/`false`.如果它为`true`,那么当这个字段在创建SolrDocument时如果没有值,它仍然可以拷贝上一个记录的相同字段的值作为自己的值.

如果一个API支持分块多次调用来获取完整的数据(当数据太大时可能会发生).`XPathEntityprocessor`也能通过一个`transformer`支持的.当这个`transformer`返回的记录中包含有`$hasMore`并且值为`true`的时候.这个处理器会使用相同的URL模板再发送一次请求(模板中的实际值会在调用前重新计算).这个转换器同样支持在返回的结果中包含下一次请求的url地址.这要求返回的结果中包含`$nextUrl`属性,其值是完整的下一次请求的URL地址.

`XPathEnityProcessor`实现了一个支持Xpath表达式子集的流式解析器.它不支持完整的XPATH语法,但是如下所示的常用的语法是支持的:

```
 xpath="/a/b/subject[@qualifier='fullTitle']"
 xpath="/a/b/subject/@qualifier"
 xpath="/a/b/c"
 xpath="//a/..."
 xpath="/a//b..."
```

##HttpDataSource的例子
在[Solr1.4](https://wiki.apache.org/solr/Solr1.4)开始,`HttpDataSource`已经由`URLDataSource`所替代而被废弃.

下载在数据库章节提供的完整的导入例子,然后我们会尝试在这个例子中索引[Slashdot RSS feed](http://rss.slashdot.org/Slashdot/slashdot).

它的`data-config.xml`文件如下所示:

```xml
<dataConfig>
        <dataSource type="HttpDataSource" />
        <document>
                <entity name="slashdot"
                        pk="link"
                        url="http://rss.slashdot.org/Slashdot/slashdot"
                        processor="XPathEntityProcessor"
                        forEach="/RDF/channel | /RDF/item"
                        transformer="DateFormatTransformer">

                        <field column="source"       xpath="/RDF/channel/title"   commonField="true" />
                        <field column="source-link"  xpath="/RDF/channel/link"    commonField="true" />
                        <field column="subject"      xpath="/RDF/channel/subject" commonField="true" />

                        <field column="title"        xpath="/RDF/item/title" />
                        <field column="link"         xpath="/RDF/item/link" />
                        <field column="description"  xpath="/RDF/item/description" />
                        <field column="creator"      xpath="/RDF/item/creator" />
                        <field column="item-subject" xpath="/RDF/item/subject" />

                        <field column="slash-department" xpath="/RDF/item/department" />
                        <field column="slash-section"    xpath="/RDF/item/section" />
                        <field column="slash-comments"   xpath="/RDF/item/comments" />
                        <field column="date" xpath="/RDF/item/date" dateTimeFormat="yyyy-MM-dd'T'hh:mm:ss" />
                </entity>
        </document>
</dataConfig>
```
这个就是它的数据配置.如果你看过`Slashdot RSS`的结构,它有一些头结点,比如标题、连接、主题等等.这些会通过`Xpath`表达式被映射成为Solr的来源、来源链接和主题字段.这一个摘要(`feed`)会包含多个`item`元素,每一个元素都包含了一个新闻.因此,我们期望的是对每一个`item`都创建一个Solr的文档.

`XPathEntityprocessor`被设计成为流式的、逐行的读取xml,它使用`forEach`属性来标记*一行*.在这个例子中,`forEach`属性被赋值为`/RDF/channel | /RDF/item`.这意味着在这个XML中会有两种类型的行记录(它使用了Xpath语法来表示`或者`,以便能表示多种类型).当它读取到一行记录后,它会尝试尽可能多的在记录中读取定义中存在的字段.因此,在这种情况下,当它读取到`/RDF/channel`这种记录的时候,它会获取`source`、`source-link`、`source-subject`三个字段.经过处理,它会发现这个记录并不具备创建索引所必须的`pk`字段,所以它并不会真正的创建Solr的文档,但是由于这三个字段都被标识为`commonField=true`.因此,它会继续持有这几个值以方便后续行的处理.
当它继续一行一行向后解析XML并遇到`/RDF/item`记录的时候,它会获取到所有除了在头上那3个值以外的所有值.但是由于那三个字被标记为共有字段,所以处理器会把先前获取到的三个字段的值放入当前创建的文档当中.

那么当一个实体中的`transformer`属性为`DateFormatTransformer`的时候会发生什么呢?具体可以参见[DateFormatTransformer](#DateFormatTransformer)章节.

你可以使用同`RSS/ATOM`、`XML`数据源、其他Solr服务器甚至是具有良好结构的XHTML文档索引一样的方法从`REST`API中建立索引.当然,我们对XPATH的支持也是有限制的(不支持通配符,只支持全路径等等).但是我们已经尝试了保证常用的通用用例能被覆盖到.并且它还是一个流式的解析器,它非常的快并且在超大的XML文档解析时也能保持比较低的内存消耗.它并不支持命名空间,但是它可以处理xmls的命名空间.当你提供XPATH的时候,只需要丢掉命名空间不管即可(比如要映射`<dc:subject>`节点只需要提供的XPATH包含`subject`即可).这非常的简单,并不需要你写一行的代码!

注意:并不像数据库,你并不能在使用`XPathEntityProcessor`的时候省略字段的定义声明.它依赖于在定义中声明的字段的XPATH,才能从xml中提取出来值.

##从维基百科提取索引的例子
下面的`data-config.xml`被用于提取维基百科dump全部的索引(只有最近的英文的文章).这个文件从维基百科网站下载下来的时候是`pages-articles.xml.bz2`,解压后有将近40GB的大小.

```xml
<dataConfig>
        <dataSource type="FileDataSource" encoding="UTF-8" />
        <document>
        <entity name="page"
                processor="XPathEntityProcessor"
                stream="true"
                forEach="/mediawiki/page/"
                url="/data/enwiki-20130102-pages-articles.xml"
                transformer="RegexTransformer,DateFormatTransformer"
                >
            <field column="id"        xpath="/mediawiki/page/id" />
            <field column="title"     xpath="/mediawiki/page/title" />
            <field column="revision"  xpath="/mediawiki/page/revision/id" />
            <field column="user"      xpath="/mediawiki/page/revision/contributor/username" />
            <field column="userId"    xpath="/mediawiki/page/revision/contributor/id" />
            <field column="text"      xpath="/mediawiki/page/revision/text" />
            <field column="timestamp" xpath="/mediawiki/page/revision/timestamp" dateTimeFormat="yyyy-MM-dd'T'hh:mm:ss'Z'" />
            <field column="$skipDoc"  regex="^#REDIRECT .*" replaceWith="true" sourceColName="text"/>
       </entity>
        </document>
</dataConfig>
```
`schema.xml`文件中相关的部分如下所示:

```xml
<field name="id"        type="string"  indexed="true" stored="true" required="true"/>
<field name="title"     type="string"  indexed="true" stored="false"/>
<field name="revision"  type="int"    indexed="true" stored="true"/>
<field name="user"      type="string"  indexed="true" stored="true"/>
<field name="userId"    type="int"     indexed="true" stored="true"/>
<field name="text"      type="text_en"    indexed="true" stored="false"/>
<field name="timestamp" type="date"    indexed="true" stored="true"/>
<field name="titleText" type="text_en"    indexed="true" stored="true"/>
...
<uniqueKey>id</uniqueKey>
<copyField source="title" dest="titleText"/>
```
在50分钟内提取了8338182篇文章,所消耗的内存峰值是4GB.该测试是在Solr4.3.1中测试的,`ramBufferSizeMB`设置为`256MB`.整个维基百科的dump文件存放在7200转速的希捷机械硬盘中,而Solr的索引存放在海盗船的SSD中.

请注意:有许多维基百科的文章是被重定向到其他的文章中,在Solr1.4后使用`$skipDoc`可以忽略这些文章.同样的,`$skipDoc`只有在正则表达式匹配的情况下被返回.

##使用增量导入命令
仅仅只有`SqlEntityProcessor`的实体处理器支持增量操作.`XPathEntityProcessor`处理器目前还没有实现增量的操作.因此,不幸的是,目前还没有方法来支持XML的增量导入.如果你想在`XPathEntityProcessor`中实现这些方法:这些方法在`EntityProcessor.java`中有解释.

#从邮件中创建索引
请参见[MailEntityProcessor](https://wiki.apache.org/solr/MailEntityProcessor)

#Tika的集成
请参见[TikaEntityProcessor](https://wiki.apache.org/solr/TikaEntityProcessor)

#扩展功能的API工具
我们目前展示过的例子都是非常简单的.不可能将用户的所有需求都通过一个xml配置文件就满足的.所以我们提供了一些抽象的类让用户自己来实现,从而实现更高级的功能.

##Transformer
每一个被实体所使用的字段值都可以直接被索引处理器所使用或者是通过`Transformer`转换器加工一下再使用,甚至有可能完全的生成新的字段值也是可能的.`transformer`转换器必须被配置在实体的属性中:

```xml
<entity name="foo" transformer="com.foo.Foo" ... />
```
注意:转换器的值必须是一个完整的类路径.如果路径是`org.apache.solr.handler.dataimport`那么包名可以被省略. 如果一个类是在`solr`的包范围类,那么`solr.<classname>`同样可以工作.这个规则适用于所有的扩展类,比如`DataSource`、`EntityProcessor`和`Evaluator`

类`Foo`必须继承自抽象类`org.apache.solr.hander.dataimport.Transformer`,这个类只有一个抽象的方法.

实体的`transformer`属性同时可以使用`,`配置多个转换器.在这种情况下转换器会按照他们定义是时的顺序链式的进行处理.这意味着当字段从数据源中被读取出来后,这些字段会按照实体定义的字段以及顺序被配置的第一个转换器筛选一次.如果转换器处理出了结果.那么所有被筛选过的实体的字段都会被用作下一个转换器的输入进行处理.

一个转换器可以被用于修改一个从数据源中获取的字段的值.也可以用于创建一个新的未定义的字段.如果转换器的操作执行失败,或者正则匹配失败,那么字段会保持原样,已存在的字段不会被修改,未定义的字段也不会被生成.上述的链式处理允许一个字段的值一次又一次不断的被转换器所改变.并且一个转换器还允许使用其他实体的字段用于处理字段的值.

###RegexTransformer
有一个名为由DIH提供的内置的名为`RegexTransformer`的转换器.它能帮助我们使用正则表达式来提取和操作字段的值.它真正的实现类是`org.apache.solr.handler.dataimport.RegexTransformer`,但是在使用的时候我们可以省略它的包名.

**属性**
`RegexTransformer`只有`regex`和`splitBy`可被使用,其他的属性会被它所忽略.

* **regex**:这是一个用来匹配字段或者sourceColName值的正则表达式.如果没有`replaceWith`属性,那么每一个正则表达式组('regex group')会被提取为一个值,并且返回值的集合.
* **sourceColName**:需要使用正则表达式提取值的来源字段名,如果没有这个字段,那么来源和目标字段的名字是相同的.
* **splitBy**:被用来标识分隔字符串的,它会把分隔后的多个值使用集合返回出去
* **groupName**:用逗号分隔的字段列名,被用来标识正则表达式组,每一组的匹配结果都会被存储到另外一个不同的字段中去.如果一些组并没有被命名,那么在两个逗号之间留一个空格.
* **replaceWith**:跟随着`regex`属性一起使用,其效果等同于调用方法`new String(<sourceColVal>).replaceAll(<regex>, <replaceWith>)`

例子:

```xml
<entity name="foo" transformer="RegexTransformer"
query="select full_name , emailids from foo"/>
... />
   <field column="full_name"/>
   <field column="firstName" regex="Mr(\w*)\b.*" sourceColName="full_name"/>
   <field column="lastName" regex="Mr.*?\b(\w*)" sourceColName="full_name"/>

   <!-- another way of doing the same -->
   <field column="fullName" regex="Mr(\w*)\b(.*)" groupNames="firstName,lastName"/>
   <field column="mailId" splitBy="," sourceColName="emailids"/>
</entity>
```
在这个例子中,`regex`属性以及`sourceColName`属性就是由转换器来使用的.它从查询结果集中读取`full_name`字段的值并且提供给`firstName`和`lastName`这两个新的字段使用.因此,即使在查询结果中只有一个`full_name`字段.但是在Solr文档中还是能获取到两个衍生的扩展字段:`firstName`和`lastName`.这些新的字段,只有在正则匹配时才会被创建.

而表中的`emailids`字段又是由逗号分隔的.因此,最终它提供了一个或多个email的id.并且我们把它提取到了Solr的一个多值(`multivalued`)字段:`mailId`中.

正则表达式的匹配默认是区分大小写的.可以使用`(?i)`以及`(?u)`标志(`u`可以在Unicode编码中使用, `i`只能是US-ASCII)来告诉表达式全部或者有一部分匹配是忽略大小写的.另外的一些标志和操作可以参考Java的正则表达式类`java.util.regex`,比照到它来设置.

```xml
<!-- matches Apples and apples -->
<field column="just_apples" regex="(?iu)(apples)" />
```
注意:这个转换器可以被用于使用`splitBy`属性来分隔字符串,或者使用`replaceWith`属性来替换字符串中的每一个值,又或者使用` groupNames`来表示一组正则表达式组.到底是用来做什么这取决于它定义时`splitBy`、`replaceWith`和`groupNames`这几个属性到底哪一个最前面.当它匹配到第一个动作后,后面的属性标签会被忽略.

###ScriptTransformer
这个转换器能让你书写`Javascript`或者其他JAVA支持的脚本语言.你必须在`JAVA6`中才能使用此功能.

```xml
<dataConfig>
        <script><![CDATA[
                function f1(row)        {
                    row.put('message', 'Hello World!');
                    return row;
                }
        ]]></script>
        <document>
                <entity name="e" pk="id" transformer="script:f1" query="select * from X">
                ....
                </entity>
        </document>
</dataConfig>
```
另外一个复杂点的例子:

```xml
<dataConfig>
        <script><![CDATA[
                function CategoryPieces(row)    {
                    var pieces = row.get('category').split('/');
                    var arr = new java.util.ArrayList();
                    for (var i=0; i<pieces.length; i++) {
                       arr.add(pieces[i]);
                    }
                    row.put('categorypieces', arr);
                    row.remove('category');
                    return row;
                }
        ]]></script>
        <document>
                <entity name="e" pk="id" transformer="script:CategoryPieces" query="select * from X">
                ....
                </entity>
        </document>
</dataConfig>
```

* 你可以把在`dataConfig`节点下放入一个`script`节点.在默认情况下假定是`Javascript`.但是当你如果要使用其他的语言,那么在`script`节点上增加一个`language="MyLanguage"`的属性即可(必须是java 6 支持的语言)
* 你可以在转换器中写任意多个的函数.每一个函数都必须能使用`Map<String,Object>`来接收每一行的数据,并且返回一行记录(在应用转换后)
* 可以使用`row.remove(keyname)`来从一行记录中删除项目
* 要给一个字段增加多个项目值,请`var arr = new java.util.ArrayList()`.你不能直接使用`JavaScript`中的数组
* `JAVA`中的`Map`对象能被变成文档
* `JAVA`中的`ArrayList`对象能被变成文档
* 要在实体中使用一个函数,可以在`entity`节点中写成`transformer="script:<function-name>"`.
* 在上面的例子中,`f1`这个`javascript`函数会在实体e返回的每一行记录上执行一次.
* 它会被当成一个java的转换器来执行.抽象类`Transformer`的`transformRow(Map<String,Object> , Context context)`方法有两个入参.但由于它是一个`javascript`脚本,因此第二个参数有可能被忽略,并且还能正常工作.

###DateFormatTransformer
DIH内置了一个名为`DateFormatTransformer`的转换器.这个转换器对于把`java.util.Date`实例转换成日期/时间字符串非常的有用.

```xml
<field column="date" xpath="/RDF/item/date" dateTimeFormat="yyyy-MM-dd'T'HH:mm:ss" locale="en" />
```

**属性**
`DateFormatTransformer`转换器仅有一个特有的`dateTimeFormat`属性.其他属性和它无关.

* **dateTimeFormat**:字段要被格式化的格式.它必须遵循java[SimpleDateFormat](http://java.sun.com/j2se/1.4.2/docs/api/java/text/SimpleDateFormat.html)的语法.
* **sourceColName**:指明哪一个列的数据会被日期格式化处理.如果不填这个属性,那么来源和目的字段都是相同的.
* **locale**:日期转换器所使用的语言环境(可选).如果没有设置语言环境,在Solr4.1以及之后的版本默认是`ROOT`语言环境(在Solr4.1之前使用的是机器默认的语言环境)

上述的字段被定义在`RSS`的例子中用来解析RSS摘要中的发布时间.

###NumberFormatTransformer
它可以用来把一个字符串解析成一个数字.它使用了JAVA中的`NumberFormat`类:

```xml
<field column="price" formatStyle="number" />
```
默认情况下,`NumberFormat`使用系统默认的语言环境来解析给定的字符串.可选的,你可以如下那样指定语言环境(请参阅`java.util.Locale`的javadoc来获取更多的信息):

```xml
<field column="price" formatStyle="number" locale="de-DE" />
```

**属性**
`NumberFormatTransformer`仅使用`formatStyle`这一个属性.

* **formatStyle**:用于指定某一个字段被转换成的格式,这个属性的值只能是`number`|`percent`|`integer`|`currency`中的一个.它会使用java`NumberFormat`的语法.
* **sourceColName**:指明哪一列的数据会被NumberFormat处理.如果不提供这个属性,那么源和目标字段相同
* **locale**:解析字符串时所使用的语言环境(可选).如果没有设置语言环境,在Solr4.1以及之后的版本默认是`ROOT`语言环境(在Solr4.1之前使用的是机器默认的语言环境)

###TemplateTransformer
可用于覆盖或修改已经存在的任意Solr字段或者创建一个新的Solr字段.分配给该属性的值是一个基于静态模板的可使用DIH变量的字符串.如果模板字符串包含占位符或变量,那么在转换器开始执行前必须定义它们.一个未定义的变量会导致整个模板的转换会被忽略.比如:

```xml
<entity name="e" transformer="TemplateTransformer" ..>
<field column="namedesc" template="hello${e.name},${eparent.surname}" />
...
</entity>
```
模板的这个规则同样适用于`query`,`url`等等.它可以帮助我们连接多个值或者为一个字段插入一些另外的字符.仅当字段上有`template`属性时它才会被执行.

**属性**

* template:模板字符串.在上面的例子中有`${e.name}`和`$eparent.surname`两个占位符.他们必须在被执行前提供值.

###HTMLStripTransformer
[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
可以在一个字符串字段中去掉HTML标签

```xml
<entity name="e" transformer="HTMLStripTransformer" ..>
<field column="htmlText" stripHTML="true" />
...
</entity>
```

它会使用`org.apache.solr.analysis.HTMLStripReader`类来去掉一个字段值中的HTML标签.

**属性**

* **stripHTML**:一个用于标记是否在这个字段上执行`HTMLStripTransformer`的布尔值

###ClobTransformer
[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
它可以用来从数据库的`Clob`类型字段中获取一个字符串.比如:

```xml
<entity name="e" transformer="ClobTransformer" ..>
<field column="hugeTextField" clob="true" />
...
</entity>
```

**属性**

* **clob**:用于标记是否在这个字段上执行`ClobTransformer`转换的布尔值
* **sourceColName**:标记数据来源字段是哪一个.如果不设置这个属性,那么来源和目标字段相同.

###LogTransformer
[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
它可以用来处理控制台或日志文件中的日志数据,比如:

```xml
<entity ...
transformer="LogTransformer"
logTemplate="The name is ${e.name}" logLevel="debug" >
....
</entity>
```
与其他的转换器不同,它不会给实体中的任何一个字段增加属性,仅仅会在`entity`节点上增加.
有效的`logLevels`是:
1. trace
2. debug
3. info
4. warn
5. error
这些值是区分大小写的(要求全小写)

###转换器的例子
[Solr1.4](https://wiki.apache.org/solr/Solr1.4)
下面的例子展示了使用转换器链来大量重复的处理变量.
我们保持`solrconfig.xml`不变并且重用一些转换器.两个实体的列名同样会在转换器中使用.

假设我们现在有一些XML文档,其中每一个都描述了一组图片.这些图片储存在XML文档的image子文件夹中.有一个性存储了图片文件名的属性关联了一段简介和一个描述这这图片详情的文档.最后,提供了一个稍小的png格式的图标文件,这个文件的名字就是在图片的名称前加一个`s`.
我们期望Solr能索引图片的绝对路径、它的图标以及详细的描述.下面就展示一下如何配置`solrconfig.xml`和DIH的`data-config.xml`文件来索引这些数据:

```xml
<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
       <str name="config">data-config.xml</str>
       </lst>
    <lst name="invariants">
       <!-- Pass through the prefix which needs stripped from
            an absolute disk path to give an absolute web path  -->
       <str name="img_installdir">/usr/local/apache2/htdocs</str>
       </lst>
    </requestHandler>
```

```xml
 <dataConfig>
 <dataSource name="myfilereader" type="FileDataSource"/>
   <document>
     <entity name="jc" rootEntity="false" dataSource="null"
             processor="FileListEntityProcessor"
             fileName="^.*\.xml$" recursive="true"
             baseDir="/usr/local/apache2/htdocs/imagery"
             >
       <entity name="x"rootEntity="true"
               dataSource="myfilereader"
               processor="XPathEntityProcessor"
               url="${jc.fileAbsolutePath}"
               stream="false" forEach="/mediaBlock"
               transformer="DateFormatTransformer,TemplateTransformer,RegexTransformer,LogTransformer"
               logTemplate="      processing ${jc.fileAbsolutePath}"
               logLevel="info"
               >

         <field column="fileAbsPath"     template="${jc.fileAbsolutePath}" />

         <field column="fileWebPath"     template="${x.fileAbsolutePath}"
                                         regex="${dataimporter.request.img_installdir}(.*)" replaceWith="$1"/>

         <field column="fileWebDir"      regex="^(.*)/.*" replaceWith="$1" sourceColName="fileWebPath"/>

         <field column="imgFilename"     xpath="/mediaBlock/@url" />
         <field column="imgCaption"      xpath="/mediaBlock/caption"  />
         <field column="imgSrcArticle"   xpath="/mediaBlock/source"
                                         template="${x.fileWebDir}/${x.imgSrcArticle}/"/>

         <field column="uid"             regex="^(.*)$" replaceWith="$1#${x.imgFilename}" sourceColName="fileWebPath"/>

         <!-- if imgFilename is not defined all the following will also not be defined -->
         <field column="imgWebPathFULL"  template="${x.fileWebDir}/images/${x.imgFilename}"/>
         <field column="imgWebPathICON"  regex="^(.*)\.\w+$" replaceWith="${x.fileWebDir}/images/s$1.png"
                                         sourceColName="imgFilename"/>

       </entity>
     </entity>
   </document>
  </dataConfig>
```

###实现自定义的转换器
你可以非常简单的实现一个你自己的转换器,具体的文档请参阅[DIHCustomTransformer](https://wiki.apache.org/solr/DIHCustomTransformer)

##EntityProcessor

每一个的实体都会被一个名叫`SqlEntityProcessor`的默认实体处理器处理.它非常适合处理使用关系型数据库作为数据源的系统.而对于其他种类的数据源,比如REST或者NOSQL数据库,你可以选择扩展`org.apache.solr.handler.dataimport.Entityprocessor`这个抽象类.它被设计为流式逐行的处理实体.一个最简单的实现自己的实体处理类的方式就是继承自`EntityProcessorBase`类并且重写它的`public Map<String,Object> nextRow()`方法.`EntityProcessor`依赖于数据源来获取数据.数据源的返回类型对于`EntityProcessor`非常的重要.

###SqlEntityProcessor
它是默认处理器.它的数据源必须是`DataSource<Iterator<Map<String, Object>>>`种类的.`JdbcDataSource`就可以被它使用.

###XPathEntityProcessor
它用于索引XML类型的数据.它的数据源必须是`DataSource<Reader>`类型的.`URLDataSource`或者`FileDataSource`都可以提供给`XPathEntityProcessor`使用.

###FileListEntityProcessor
这是一个简单的实体处理器.它可以被用来基于某些条件枚举文件系统上的一系列文件.它不需要使用数据源.它的实体的属性有:

* fileName:(必须)标示文件的一个正则表达式
* baseDir:(必须)基础目录(绝对路径)
* recursive:是否递归扫描文件,默认是`false`
* excludes:一个标示需要排除的文件的正则表达式
* newerThan:一个日期参数.使用`yyyy-MM-dd HH:mm:ss`的格式.它也可以是一个日期计算字符串(`'NOW-3DAYS'`).这个单引号是必须的.或者也可以是一种有效的`VariableResolver`格式,比如(`${var.name}`)
* olderThan:一个日期参数,规则同上
* biggerThan:一个整形参数
* smallerThan:一个整形参数
* rootEntity:除非你只希望索引文件名,否则必须在`document`节点的直接`entity`节点上设置为`false`.这意味着,对于从根实体中获取的每一行记录都会被创建成为一个Solr/Lucene的文档.但可能在这种情况下,我们并不想把每一个文件都创建成一个document.我们希望由实体`x`生成的每一行数据成为一个文档.由于实体`f`是`rootEntity=false`的,直接在它下面的实体会自动的成为一个根实体.并且每一行都会变成一个文档.
* dataSource:如果是使用的Solr1.3,那么这个值必须被设置成`null`.因为它不使用任何的数据源.在Solr1.4中已经不需要设置这个属性了.它仅仅表示我们不会创建一个数据源实例(在大多数情况下,只会使用一个数据源(`JdbcDataSource`)并且所有的实体都使用它.在`FileListEntityProcessor`中,并不需要使用数据源)

例子:

```xml
<dataConfig>
    <dataSource type="FileDataSource" />
    <document>
        <entity name="f" processor="FileListEntityProcessor" baseDir="/some/path/to/files" fileName=".*xml" newerThan="'NOW-3DAYS'" recursive="true" rootEntity="false" dataSource="null">
            <entity name="x" processor="XPathEntityProcessor" forEach="/the/record/xpath" url="${f.fileAbsolutePath}">
                <field column="full_name" xpath="/field/xpath"/>
            </entity>
        </entity>
    </document>
</dataConfig>
```

不要忘了`rootEntity`属性.如上所示,`FileListEntityProcessor`会隐式的在实体`X`中生成`fileDir`、`file`、`fileAbsolutePath`、`fileSize`、`fileLastModified`字段.需要特别指出的是`FileListEntityProcessor`返回的路径列表和随后的实体必须使用`FileDataSource`来获取文件的内容.

###CachedSqlEntityProcessor
这个处理器是`SqlEntityProcessor`的一个扩展.这个实体处理器可以通过命中缓存来减少真正执行数据库的查询.对于只有一个SQL执行的根实体,在大多数情况下是不起作用的.

例1:

```xml
<entity name="x" query="select * from x">
    <entity name="y" query="select * from y where xid=${x.id}" processor="CachedSqlEntityProcessor">
    </entity>
<entity>
```
这个例子和其他的例子一样.当一个查询被执行,那么它的查询结果会被缓存起来,再当遇到相同的查询时,它会直接从缓存中获取数据并返回.

例2:

```xml
<entity name="x" query="select * from x">
    <entity name="y" query="select * from y" processor="CachedSqlEntityProcessor"  where="xid=x.id">
    </entity>
<entity>
```
这个例子和上面的例子在`where`属性上稍微有一点不一样.在这个例子中查询条件从数据库表中命中了所有的记录,并且把这些记录都放入了缓存.而神奇的地方在于`where`的值.缓存会在实体`y`中定义一个`xid`的键.在之后的每一次执行查询实体的时候,都会直接从缓存中命中,并且把`x.id`的值赋予`xid`,然后返回整行的数据.

表达式左边是`y`中的列,表达式右边的值会从缓存中获取并赋值.

上面例2的另外一种写法是使用`cacheKey`和`cacheLookup`参数:

```xml
<entity name="x" query="select * from x">
    <entity name="y" query="select * from y" processor="CachedSqlEntityProcessor" cacheKey="xid" cacheLookup="x.id">
    </entity>
<entity>
```

注意:在Solr 3.6, 3.6.1, 4.0-Alpha & 4.0-Beta版本中`cacheKey`属性被重命名为`cachePk`.它从4.0和3.6.2又变了回来.参阅:[SOLR-3850](https://issues.apache.org/jira/browse/SOLR-3850)

关于DIH的更多缓存的选项可以看[SOLR-2382](https://issues.apache.org/jira/browse/SOLR-2382).这些额外的选项包括:在NOSQL的实体中使用缓存,扩展缓存的实现,持久化缓存,把DIH的输出写入缓存而不是Solr,使用已经创建了的缓存作为DIH实体的输入以及增量更新缓存数据.其中的一些特性会在`Solr3.6` `Solr4.0` 中提供.

###PlainTextEntityProcessor
这个实体处理器会从数据源中读取所有的内容,并且写入到一个被称为`plainText`的隐藏字段中.文本的内容不会做任何的解析,但是你可以根据需要,自己增加转换器来处理`plainText`中的数据,甚至是增加一些字段出来.
例子:

```xml
<entity processor="PlainTextEntityProcessor" name="x" url="http://abc.com/a.txt" dataSource="data-source-name">
   <!-- copies the text to a field called 'text' in Solr-->
  <field column="plainText" name="text"/>
</entity>
```
需要确保数据源是`DataSource<Reader> (FileDataSource, URLDataSource)`类型的

###LineEntityProcessor
这个实体处理会会从数据源中一行一行的读取所有的内容.每读取一行都会把返回值放入一个叫`rawLine`的字段中.同样,文本的内容不会做任何的解析,但是你可以根据需要,自己增加转换器来处理`rawLine`中的数据,甚至是增加一些字段出来.

读取的行数据可以使用`acceptLineRegex`和`omitLineRegex`这两个正则表达式来进行过滤.给实体增加的属性有:

* **url**:一个必填的属性,用于指定一个与已配置的数据源相兼容的输入文件路径.如果这个值是个相对路径并且你使用的是`FileDataSource`或`URLDataSource`.那么它相当于就是`baseLoc`.
* **acceptLineRegex**:一个可选的属性,用于把所有于这个正则表达式不匹配的数据丢掉.
* **skipLineRegex**:一个可选的属性,用于在`acceptLineRegex`执行之后,再把所有符合这个正则表达式的数据丢掉.

例子:

```xml
<entity name="jc"
        processor="LineEntityProcessor"
        acceptLineRegex="^.*\.xml$"
        skipLineRegex="/obsolete"
        url="file:///Volumes/ts/files.lis"
        rootEntity="false"
        dataSource="myURIreader1"
        transformer="RegexTransformer,DateFormatTransformer"
        >
   ...
```
当你期望把一个文件中的每一行都当做一个Solr文档的时候,你会需要这个处理器.在大多数情况下,每一行会读取一个路径,而这又会被其他的实体处理器(比如`XPathEntityProcessor`)所使用.

[SOLR-2549](https://issues.apache.org/jira/browse/SOLR-2549)补丁让`LineEntityProcessor`支持了可变长度和带有分隔符的文件,这在之前是需要使用转换器的.

###SolrEntityProcessor
[Solr3.6](https://wiki.apache.org/solr/Solr3.6)
这个实体处理器可以从其他的Solr实例与核心(`cores`)中导入数据.这个数据是根据一个指定的查询来获取.这个实体处理器在你想从你的Solr索引中稍微修改一下然后放入其他的索引中时非常的有用.在一些情况下,所有的数据可能会只存放在Solr的索引中.那么`SolrEntityProcessor`只能复制那些在源索引中`stored`了的字段.`SolrEntityProcessor`支持以下一些属性:

* **url**:(必填)源Solr实体或核心的URL地址
* **query**:(必填)在源索引中最主要的查询语句
* **fq**:在源索引中执行的任何过滤条件(逗号分隔)
* **rows**:每一次迭代所返回的行数,默认是50
* **fl**:需要从源索引中返回的字段名(逗号分隔)
* **qt**:指定哪一个执行器会被使用
* **wt**:指定返回的格式(`javabin`或者`xml`).如果你导入Solr1.x以后的版本的索引,可以使用xml格式.`javabin`格式在1.4.1和3.1.0版本之间有变动.
* **timeout**:查询超时时间,单位是秒.你可以用它来做安全的故障点,以防止索引会话被长时间的阻塞.默认情况下超时时间为5分钟.

例子:

```xml
<dataConfig>
  <document>
    <entity name="sep" processor="SolrEntityProcessor" url="http://localhost:8983/solr/db" query="*:*"/>
  </document>
</dataConfig>
```

##DataSource
可以从`org.apache.solr.handler.dataimport.DataSource`([源码](http://svn.apache.org/viewvc/lucene/dev/trunk/solr/contrib/dataimporthandler/src/java/org/apache/solr/handler/dataimport/DataSource.java?view=markup))扩展一个类用来作为数据源.它必须在数据源的配置中被定义.

```xml
<dataSource type="com.foo.FooDataSource" prop1="hello"/>
```
它可以在标准的实体中使用.

###JdbcDataSource
这是默认的实现.例子可以见[这里](#jdbcdatasource).它的签名为:

```java
public class JdbcDataSource extends DataSource<Iterator<Map<String, Object>>>
```
它被设计来从数据库中一行一行的读取数据.每一行就是一个Map

###URLDataSource
这个数据源通常为`XPathEntityProcessor`从`file://`或`http://`地址提供数据.请参阅[这里](#httpds).它的签名为:

```java
public class URLDataSource extends DataSource<Reader>
```

###HttpDataSource
`HttpDataSource`数据源已经在Solr1.4以后被`URLDataSource`所替代.它和`URLDataSource`的功能是一样的,仅仅是名字不同而已.

###FileDataSource
它可以向`URLDataSource`那样被使用,只是他是用磁盘上的文件中获取数据.唯一与`URLDataSource`不同的地方在于访问磁盘文件的时候,路径名定义不一样.它的签名是这样的:

```java
public class FileDataSource extends DataSource<Reader>
```
属性包括:

* **basePath**:(可选)当不是绝对路径的时候的一个相对的基础路径.
* **encoding**:(可选)在Solr4.1以后的版本中,默认是UTF-8编码(在4.1之前,默认的编码是和机器的默认编码一致)

###FieldReaderDataSource
它可以像`URLDataSource`那样被使用,它的签名是:

```java
public class FieldReaderDataSource extends DataSource<Reader>
```
当一个数据库字段中包含了一个XML并且期望使用`XPathEntityProcessor`来处理这些字段内容的时候它非常的有用.这个数据源可以这样配置:

```xml
<dataSource name="f" type="FieldReaderDataSource" encoding="UTF-8" />
```
`encoding`属性是可选的,在Solr4.1以后的版本中,默认是UTF-8编码(在4.1之前,默认的编码是和机器的默认编码一致)

要使用这个数据源的实体必须有一个名为`dataFiled="field-name"`的变量来保存url值.例如,如果一个叫`dbEntity`的父实体有一个`xmlData`字段.那么它的子实体就会是这样的:

```xml
<entity dataSource="f" processor="XPathEntityProcessor" dataField="dbEntity.xmlData"/>
```

###ContentStreamDataSource
它会使用POST的数据作为数据源.它可以在任何使用`DataSource<Reader>`作为数据源的实体处理器中被使用.

##EventListeners
事件监听器会注册`onImportStart`和`onImportEnd`两个事件.他们必须实现[EventListener](http://lucene.apache.org/solr/api/org/apache/solr/handler/dataimport/EventListener.html)接口.

```xml
<dataConfig>
<document onImportStart ="com.FooStart" onImportEnd="comFooEnd">
....
</document>
</dataConfig>
```

##特殊指令
一些特殊的指令可以给DIH的任何组件加入某些变量,从而影响它们的返回.

* **$skipDoc**:跳过当前的文档.不把它加入到Solr中.这个是一个布尔值
* **$skipRow**:跳过当前的行.文档会从其他实体的记录中被增加.这是一个布尔值
* **$docBoost**:当前文档的权重.这个值是一个数字或者表示数字的字符串
* **$deletedDocById**:使用id从Solr中删除一个文档.这个值就是文档的唯一键.注意的是这个指令仅仅能删除那些已经被提交到索引中的数据.
* **$deleteDocByQuery**:使用一个查询语句来删除Solr中的文档.其值必须是一个Solr的查询语句.

注意:在Solr3.4,`$deleteDocById`和`$deleteDocByQuery`不会增加已删除数(`#deletes processed`)的统计.同样的,如果一个组件只是使用这些特殊的指令来删除文档.那么DIH是不会提交数据变化的.在Solr3.4以后,`commit`动作会始终被执行.并且在每一次调用`$deleteDocById`或`$deleteDocByQuery`后,已删除数的统计也会加1.并且被删除的文档数量在一次调用可以删除多个文档的情况下(特别是使用$deleteDocByQuery)统计可能不准确.更多的信息可以参阅[SOLR-2492](https://issues.apache.org/jira/browse/SOLR-2492)

##在solrConfig.xml中增加数据源
你可以像在`data-config.xml`增加数据源那样在`solrconfig.xml`中配置数据源.只是数据源的属性可能有些不同:

```xml
 <requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
      <str name="config">/home/username/data-config.xml</str>
      <lst name="datasource">
         <str name="driver">com.mysql.jdbc.Driver</str>
         <str name="url">jdbc:mysql://localhost/dbname</str>
         <str name="user">db_username</str>
         <str name="password">db_password</str>
      </lst>
    </lst>
  </requestHandler>
```

#架构
下面的图展示了一个样例配置的逻辑流程.
![](https://wiki.apache.org/solr/DataImportHandler?action=AttachFile&do=get&target=DataImportHandlerOverview.png)

这个用例如下:他有三个数据源,其中两个是关系型数据库(`jdbc1`,`jdbc2`),还有一个是`xml/http`的

* `jdbc1`和`jdbc2`是两个配置在`solrconfig.xml`中的`JdbcDataSource`实例
* `http`是一个`HttpDataSource`的实例
* 根实体使用`jdbc1`数据源和名为`A`的表.实体名为了方便和表名相同.
* 实体A有B和C两个子实体.B使用`http`数据源而C使用`jdbc2`数据源
* 首先在实体A上执行`command=full-import`命令
* 实体A的`query`每一行查询结果会被赋值给实体B,C
* B和C中的查询语句使用了A的列来构造它们的查询占位符(比如`${A.a}`)
    * B有一个url(B是一个xml/http的数据源)
    * C有一个查询语句
* C有两个转换器(`f`和`g`)
* C的每一行结果都会顺序的交给`f`和`g`处理(链式处理转换器).每一个转换器可以改变它们的输入.注意:转换器`g`从`f`中输入一行记录会输出两行结果.
* 所有实体的输出最后都会被组合成一个文档
    * 需要注意的是来自C的中间记录`C.1`,`C.2`,`f(C.1)`,`f(C1)`会被忽略

##字段声明
在`entity`标签中声明字段可以帮助我们提供更多的不能被自动得出的信息.它依赖于从结果中获取的`column`的值.你显示的在配置中增加的字段,这等于在Solr的schema.xml中增加了一些隐式的字段.在schema.xml中会自动的从配置文件中继承所有的属性.当你增加字段条目的时候,你不能增加额外的配置.

* 实体处理器提供的字段与schema.xml中所配置的字段名字不相同.
* 内置的转换器需要额外的信息来决定哪些字段会被处理,以及如何处理.
* `XPathEntityprocessor`或其他的处理器需要每一个字段都提供额外的信息.

##什么是一行记录?
在DIH中,一行记录就是一个Map(`Map<String, Object>`).在这个map中,key是字段的名字.value可以是任意合法的Solr类型的数据.value同样可以是任意合法的Solr类型的集合(这可能会映射到一个多值字段(`multiValued field`)中去).在关系型数据库中一条查询不能提供一个多值字段.但是可以通过关联其他的实体查询来创建一个多值字段.如果一个父实体的记录关联了子实体返回的多条记录,那么他们就可以被放入一个多值字段中去.如果数据源是`xml`.那就可以直接返回一个多值字段.

##变量解算器
变量解算器(`VariableResolver`)可以替换那些像`${name}`的占位符.它是一个多级Map.每一个命名空间就是第一个Map.同时命名空间以`.`符号分隔.例如这有一个占位符`${item.ID}`,`item`就是一个命名空间(是一个Map),`ID`就是那个命名空间中的一个值.命名空间可以像`${item.x.ID}`那样嵌套,`x`又是另外一个命名空间.可以从上下文中获取当前的变量解算器结果的引用.或者可以在关系型数据库的`query`或HTTP的`url`中直接使用`${name}`这样的占位符.
        
##识别器--在查询或URL地址中自定义格式
虽然命名空间的概念十分的有用,当时有可能用户希望在查询或url地址中插入一些计算后的值.比如:有一个日期对象并且你的数据源也能接受一些自定义的日期格式化.

###formatDate
使用这个函数来把日期格式化成字符串,它有4个参数(在Solr4.1以前,它只有两个):

1. 一个日期类型的变量或者日期计算表达式(`datemath expression`)
2. 一个日期格式字符串.具体的格式请参考`java.text.SimpleDateFormat`的`javadoc`.(在Solr4.1以后,必须用单引号括起来,在Solr1.4-4.0,引号是可选的,在Solr1.4以前不能有单引号)
3. 一个用于格式化日期的语言环境变量(可选).需要用单引号括起来.具体可以参考`java.util.Locale`的`javadoc`文档.如果忽略它,那么默认会使用根的语言环境(在Solr4.1以前,会始终使用机器默认的语言环境)
4. 一个时区码(可选)具体的可以参考`java.util.TimeZone#getTimeZone`的`javadoc`文档.如果不填,默认情况下使用机器的时区.如果要设置时区码,那么在第三个参数上必须填入语言环境(`Locale`)

* 变量的例子:`${dataimporter.functions.formatDat(item.ID,'yyyy-MM-dd HH:mm')}`
* 日期计算表达式的例子:`'${dataimporter.functions.formatDate('NOW-3DAYS', 'yyyy-MM-dd HH:mm')}`
* 指定语言环境的例子:`${dataimporter.functions.formatDate(item.ID, 'yyyy-MM-dd HH:mm', 'th_TH')}`
* 指定时区的例子:`'${dataimporter.functions.formatDate(item.ID, 'yyyy-MM-dd HH:mm', 'en_US', 'GMT-8:00')}`

###escapeSql
使用这个可以对特殊的SQL字符进行转义.例如`${dataimporter.functions.escapeSql(item.ID)}`.它只需要一个参数,并且这个值必须是在`VaraiableResolver`中合法的.

###encodeUrl
使用这个可以编码URL地址.比如:`${dataimporter.functions.encodeUrl(item.ID)}`.它只需要一个参数,并且这个值必须是在`VaraiableResolver`中合法的.

##自定义识别器
在DIH中提供了插件机制来自定义函数.只要实现[Evaluator](http://lucene.apache.org/solr/api/org/apache/solr/handler/dataimport/Evaluator.html)接口,并且把它配置在`data-config.xml`中即可.下面的例子展示了如果定义一个转换字符串为小写的识别器.

```xml
<dataConfig>
  <function name="toLowerCase" class="foo.LowerCaseFunctionEvaluator"/>
  <document>
    <entity query="select * from table where name='${dataimporter.functions.toLowerCase(dataimporter.request.user)'">
      <!- ......field declarations......->
    </entity>
  </document>
</dataConfig>
```

实现类是:`LowerCaseFunctionEvaluator`
以下代码基于Solr4.1的基础修改:

```java
public class LowerCaseFunctionEvaluator extends Evaluator{
    public String evaluate(String expression, Context context) {
      List<Object> l = parseParams(expression, context.getVariableResolver());
      if (l.size() != 1) {
          throw new RuntimeException("'toLowerCase' must have only one parameter ");
      }
      return l.get(0).toString().toLowerCase(Locale.ROOT);
    }
  }
```
像`decode` `load` `run`等函数可以帮助我们使用复杂SQL语句

###访问请求参数
当使用DIH时,所有的HTTP请求参数都会发送给Solr.这时你能使用`${dataimporter.request.command}`命名空间来访问请求中的命令是什么.

#交互开发模式
要开启这个模式,需要在DIH的UI界面上把`Debug Mode`按钮滑向右边.它会在HTML文本区中显示当前你DIH的配置,并且你能修改它.在配置的下面有一个名为`Raw Debug-Response`的区域,它包含了你点击在屏幕左边的`Execute with this Configuration`蓝色按钮后从DIH得到的响应.

一些注意事项:

* 你可以在115至118(?)配置启动和行参数来调试文档.
* 选择`verbose`下拉选项来获取中间步骤的详细的信息.执行了什么查询,什么被转换器进行了处理,输出是什么.
* 如果在运行过程中发生异常,错误堆栈信息会显示在这里
* 一些实体提供的字段,如果这些字段没有在schema.xml中显示的被`field`标签声明,那么转换器可能不能在文档中访问它们.

#调度(Scheduling)

* DataImportScheduler
* 版本: 1.2
* 最后修订时间: 20.09.2010.
* 作者: Marko Bonaci
* Jira: [http://issues.apache.org/jira/browse/SOLR-2305](http://issues.apache.org/jira/browse/SOLR-2305)
* Download JAR: [http://code.google.com/p/solr-data-import-scheduler/](http://code.google.com/p/solr-data-import-scheduler/)
* 允许调度DIH的增量和全导入
* 发送HTTP Post请求给Solr服务器
* 需要进一步的完善
* 在Tomcat6中完成测试
* 还没有提交到trunk中
* 计划在Solr4.1中提供

TODO:

* 允许用户创建多个调度任务(`List<DataImportScheduler>`)
* 增加取消功能(以便能在不停止应用程序和服务器的情况下,停止`DIHScheduler`的后台执行).目前,可以在`dataimport.properties`文件中把`syncEnabled`参数设置为`非1`来关闭同步执行,但是后台线程仍然的存活.并且每一次执行都要去重新加载一次properties属性文件(这样就能同步的进行热部署)
* 尽可能的使用Solr的类
* 增加`javadoc`文档

##已具备:

* 在DIH配置同一地方工作.
* `solr.home/conf/`文件夹下的`dataimport.properties`文件有强制属性(具体见下面例子中的dataimport.properties文件)
* 在Solr的web.xml中声明`ApplicationListener`
* 在war包部署前,下载或构建jar文件放入solr.war的`web-inf/lib`文件夹中

修订:

* v1.2:
    * 成为了`core-aware`(现在不会关心是单核心还是多核心的Solr了)
    * 调度时间参数化(分钟)
    *
* v1.1:
    * 现在使用`SolrResourceLoader`来获取`solr.home`
    * 如果响应代码不是200,那么重加载属性文件
    * 使用`slf4j`来记录日志
* v1.0:
    * 初始化版本

##SolrDataImportProperties

* 使用[java.util.Properties](http://download.oracle.com/javase/6/docs/api/java/util/Properties.html)来从`dataimport.properties`中加载配置

```java
package org.apache.solr.handler.dataimport.scheduler;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.util.Properties;

import org.apache.solr.core.SolrResourceLoader;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SolrDataImportProperties {
        private Properties properties;

        public static final String SYNC_ENABLED         = "syncEnabled";
        public static final String SYNC_CORES           = "syncCores";
        public static final String SERVER               = "server";
        public static final String PORT                 = "port";
        public static final String WEBAPP               = "webapp";
        public static final String PARAMS               = "params";
        public static final String INTERVAL             = "interval";

        private static final Logger logger = LoggerFactory.getLogger(SolrDataImportProperties.class);

        public SolrDataImportProperties(){
//              loadProperties(true);
        }

        public void loadProperties(boolean force){
                try{
                        SolrResourceLoader loader = new SolrResourceLoader(null);
                        logger.info("Instance dir = " + loader.getInstanceDir());

                        String configDir = loader.getConfigDir();
                        configDir = SolrResourceLoader.normalizeDir(configDir);
                        if(force || properties == null){
                                properties = new Properties();

                                String dataImportPropertiesPath = configDir + "\\dataimport.properties";

                                FileInputStream fis = new FileInputStream(dataImportPropertiesPath);
                                properties.load(fis);
                        }
                }catch(FileNotFoundException fnfe){
                        logger.error("Error locating DataImportScheduler dataimport.properties file", fnfe);
                }catch(IOException ioe){
                        logger.error("Error reading DataImportScheduler dataimport.properties file", ioe);
                }catch(Exception e){
                        logger.error("Error loading DataImportScheduler properties", e);
                }
        }

        public String getProperty(String key){
                return properties.getProperty(key);
        }
}
```

##ApplicationListener

* 这个类实现了[javax.servlet.ServletContextListener](http://download.oracle.com/javaee/6/api/javax/servlet/ServletContextListener.html)接口(用于监听Web应用初始化及销毁事件)
* 使用了`HTTPPostScheduler`.提供[java.util.Timer](http://download.oracle.com/javase/6/docs/api/java/util/Timer.html)和上下文属性Map方便定时方法的调用
* 计时器本质上是一个用于安排任务在后台线程执行的工具
* 不要忘了在Solr的`web.xml`中增加下面的监听器定义:

    ```xml
    <listener>
   <listener-class>org.apache.solr.handler.dataimport.scheduler.ApplicationListener</listener-class>
 </listener>
    ```
* 为了保证调度器的类能在DIH中被访问,你需要下载jar文件到你的Solr.war的`web-inf\lib`文件夹中(你可以直接在war包中增加这个文件,或者在部署后的解压文件夹中放入)

```java
package org.apache.solr.handler.dataimport.scheduler;

import java.util.Calendar;
import java.util.Date;
import java.util.Timer;

import javax.servlet.ServletContext;
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ApplicationListener implements ServletContextListener {

        private static final Logger logger = LoggerFactory.getLogger(ApplicationListener.class);

        @Override
        public void contextDestroyed(ServletContextEvent servletContextEvent) {
                ServletContext servletContext = servletContextEvent.getServletContext();

                // get our timer from the context
                Timer timer = (Timer)servletContext.getAttribute("timer");

                // cancel all active tasks in the timers queue
                if (timer != null)
                        timer.cancel();

                // remove the timer from the context
                servletContext.removeAttribute("timer");

        }

        @Override
        public void contextInitialized(ServletContextEvent servletContextEvent) {
                ServletContext servletContext = servletContextEvent.getServletContext();
                try{
                        // create the timer and timer task objects
                        Timer timer = new Timer();
                        HTTPPostScheduler task = new HTTPPostScheduler(servletContext.getServletContextName(), timer);

                        // get our interval from HTTPPostScheduler
                        int interval = task.getIntervalInt();

                        // get a calendar to set the start time (first run)
                        Calendar calendar = Calendar.getInstance();

                        // set the first run to now + interval (to avoid fireing while the app/server is starting)
                        calendar.add(Calendar.MINUTE, interval);
                        Date startTime = calendar.getTime();

                        // schedule the task
                        timer.scheduleAtFixedRate(task, startTime, 1000 * 60 * interval);

                        // save the timer in context
                        servletContext.setAttribute("timer", timer);

                } catch (Exception e) {
                        if(e.getMessage().endsWith("disabled")){
                                logger.info("Schedule disabled");
                        }else{
                                logger.error("Problem initializing the scheduled task: ", e);
                        }
                }
        }

}
```

##HTTPPostScheduler

* 这个类继承至[java.util.TimerTask](http://download.oracle.com/javase/6/docs/api/java/util/TimerTask.html),它实现了`java.lang.Runnable`
* 它表示了`DIHScheduler`的主线程
* 在适当的情况下,使用DIH参数并设置默认值
* 使用DIH的参数来组装完整的URL
* 使用HTTP POST请求URL地址

```java
package org.apache.solr.handler.dataimport.scheduler;

import java.io.IOException;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.URL;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Timer;
import java.util.TimerTask;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


public class HTTPPostScheduler extends TimerTask {
        private String syncEnabled;
        private String[] syncCores;
        private String server;
        private String port;
        private String webapp;
        private String params;
        private String interval;
        private String cores;
        private SolrDataImportProperties p;
        private boolean singleCore;

        private static final Logger logger = LoggerFactory.getLogger(HTTPPostScheduler.class);

        public HTTPPostScheduler(String webAppName, Timer t) throws Exception{
                //load properties from global dataimport.properties
                p = new SolrDataImportProperties();
                reloadParams();
                fixParams(webAppName);

                if(!syncEnabled.equals("1")) throw new Exception("Schedule disabled");

                if(syncCores == null || (syncCores.length == 1 && syncCores[0].isEmpty())){
                        singleCore = true;
                        logger.info("<index update process> Single core identified in dataimport.properties");
                }else{
                        singleCore = false;
                        logger.info("<index update process> Multiple cores identified in dataimport.properties. Sync active for: " + cores);
                }
        }

        private void reloadParams(){
                p.loadProperties(true);
                syncEnabled = p.getProperty(SolrDataImportProperties.SYNC_ENABLED);
                cores           = p.getProperty(SolrDataImportProperties.SYNC_CORES);
                server          = p.getProperty(SolrDataImportProperties.SERVER);
                port            = p.getProperty(SolrDataImportProperties.PORT);
                webapp          = p.getProperty(SolrDataImportProperties.WEBAPP);
                params          = p.getProperty(SolrDataImportProperties.PARAMS);
                interval        = p.getProperty(SolrDataImportProperties.INTERVAL);
                syncCores       = cores != null ? cores.split(",") : null;
        }

        private void fixParams(String webAppName){
                if(server == null || server.isEmpty())  server = "localhost";
                if(port == null || port.isEmpty())              port = "8080";
                if(webapp == null || webapp.isEmpty())  webapp = webAppName;
                if(interval == null || interval.isEmpty() || getIntervalInt() <= 0) interval = "30";
        }

        public void run() {
                try{
                        // check mandatory params
                        if(server.isEmpty() || webapp.isEmpty() || params == null || params.isEmpty()){
                                logger.warn("<index update process> Insuficient info provided for data import");
                                logger.info("<index update process> Reloading global dataimport.properties");
                                reloadParams();

                        // single-core
                        }else if(singleCore){
                                prepUrlSendHttpPost();

                        // multi-core
                        }else if(syncCores.length == 0 || (syncCores.length == 1 && syncCores[0].isEmpty())){
                                logger.warn("<index update process> No cores scheduled for data import");
                                logger.info("<index update process> Reloading global dataimport.properties");
                                reloadParams();

                        }else{
                                for(String core : syncCores){
                                        prepUrlSendHttpPost(core);
                                }
                        }
                }catch(Exception e){
                        logger.error("Failed to prepare for sendHttpPost", e);
                        reloadParams();
                }
        }


        private void prepUrlSendHttpPost(){
                String coreUrl = "http://" + server + ":" + port + "/" + webapp + params;
                sendHttpPost(coreUrl, null);
        }

        private void prepUrlSendHttpPost(String coreName){
                String coreUrl = "http://" + server + ":" + port + "/" + webapp + "/" + coreName + params;
                sendHttpPost(coreUrl, coreName);
        }


        private void sendHttpPost(String completeUrl, String coreName){
                DateFormat df = new SimpleDateFormat("dd.MM.yyyy HH:mm:ss SSS");
                Date startTime = new Date();

                // prepare the core var
                String core = coreName == null ? "" : "[" + coreName + "] ";

                logger.info(core + "<index update process> Process started at .............. " + df.format(startTime));

                try{

                    URL url = new URL(completeUrl);
                    HttpURLConnection conn = (HttpURLConnection)url.openConnection();

                    conn.setRequestMethod("POST");
                    conn.setRequestProperty("type", "submit");
                    conn.setDoOutput(true);

                        // Send HTTP POST
                    conn.connect();

                    logger.info(core + "<index update process> Request method\t\t\t" + conn.getRequestMethod());
                    logger.info(core + "<index update process> Succesfully connected to server\t" + server);
                    logger.info(core + "<index update process> Using port\t\t\t" + port);
                    logger.info(core + "<index update process> Application name\t\t\t" + webapp);
                    logger.info(core + "<index update process> URL params\t\t\t" + params);
                    logger.info(core + "<index update process> Full URL\t\t\t\t" + conn.getURL());
                    logger.info(core + "<index update process> Response message\t\t\t" + conn.getResponseMessage());
                    logger.info(core + "<index update process> Response code\t\t\t" + conn.getResponseCode());

                    //listen for change in properties file if an error occurs
                    if(conn.getResponseCode() != 200){
                        reloadParams();
                    }

                    conn.disconnect();
                    logger.info(core + "<index update process> Disconnected from server\t\t" + server);
                    Date endTime = new Date();
                    logger.info(core + "<index update process> Process ended at ................ " + df.format(endTime));
                }catch(MalformedURLException mue){
                        logger.error("Failed to assemble URL for HTTP POST", mue);
                }catch(IOException ioe){
                        logger.error("Failed to connect to the specified URL while trying to send HTTP POST", ioe);
                }catch(Exception e){
                        logger.error("Failed to send HTTP POST", e);
                }
        }

        public int getIntervalInt() {
                try{
                        return Integer.parseInt(interval);
                }catch(NumberFormatException e){
                        logger.warn("Unable to convert 'interval' to number. Using default value (30) instead", e);
                        return 30; //return default in case of error
                }
        }
}
```

##dataimport.properties 例子

* 从下面的导入调度配置中拷贝所有的东西到你的`dataimport.properties`文件,并修改它们的参数
* 无论你是多核心还是单核心的Solr,把`dataimport.properties`文件放入你的`solr.home/conf`下(不是`solr.home/core/conf`)

```
#Tue Jul 21 12:10:50 CEST 2010
metadataObject.last_index_time=2010-09-20 11\:12\:47
last_index_time=2010-09-20 11\:12\:47


#################################################
#                                               #
#       dataimport scheduler properties         #
#                                               #
#################################################

#  to sync or not to sync
#  1 - active; anything else - inactive
syncEnabled=1

#  which cores to schedule
#  in a multi-core environment you can decide which cores you want syncronized
#  leave empty or comment it out if using single-core deployment
syncCores=coreHr,coreEn

#  solr server name or IP address
#  [defaults to localhost if empty]
server=localhost

#  solr server port
#  [defaults to 80 if empty]
port=8080

#  application name/context
#  [defaults to current ServletContextListener's context (app) name]
webapp=solrTest_WEB

#  URL params [mandatory]
#  remainder of URL
params=/select?qt=/dataimport&command=delta-import&clean=false&commit=true

#  schedule interval
#  number of minutes between two runs
#  [defaults to 30 if empty]
interval=10
```

#哪里可以下载
`DataImportHandler`是Solr新加的功能,你可以从这找到它:

* 从[Solr website](http://lucene.apache.org/solr/)下载每日构建版
* 然后按照全导入的例子一步一步的尝试它.

对于`DataImportHandler`未来的发展和历史的讨论,你可以参见`Solr JIRA` [SOLR-469](http://issues.apache.org/jira/browse/SOLR-469) 

请提供你的意见,建议或代码贡献来帮助我们实现新的功能

我们希望通过更多的例子来展示这个工具的强大.我们会不定时的更新这个文档.

#故障排除

* 如果你在索引国际字符中遇到了问题,请尝试在数据源的配置中设置`encoding`属性为`UTF-8`(见上面的例子).这应该能确保你的国际字符能正确的被索引(使用UTF8).

    ```xml
    <dataSource type="FileDataSource" encoding="UTF-8"/>
    ```
* 如果你从数据库中获取的数据不是你期望的,需要检测以下几个方面:
    1. 转换器链的排查有点棘手.有些转换器可以指定`sourceColName`属性来获取数据,但是他们会把转换的结果放入`column`属性的字段中.所以后面的转换器实际上将作用于同一个没有被转换的数据.为了避免这种情况,最好是在你的sql中使用`AS`语法,而不是使用`sourceColName`:

```xml
<entity name="transaction"
 transformer="ClobTransformer, RegexTransformer"
 query="SELECT CO_TRANSACTION_ID as TID_COMMON, CO_FROM_SERVICE_DT as FROM_SERVICE_DT, CO_TO_SERVICE_DT as TO_SERVICE_DT, CO_PATIENT_LAST_NM as PATIENT_LAST_NM, CO_TOTAL_CLAIM_CHARGE_AMT as TOTAL_CLAIM_CHARGE_AMT FROM TABLE(pkg_solr_import.cb_get_transactions('${document.DOCUMENT_ID}'))">
<field column="TID_COMMON" splitBy="#" clob="true"/>
<field column="FROM_SERVICE_DT" splitBy="#" clob="true"/>
<field column="TO_SERVICE_DT" splitBy="#" clob="true"/>
<field column="PATIENT_LAST_NM" splitBy="#" clob="true"/>
<field column="TOTAL_CLAIM_CHARGE_AMT" splitBy="#" clob="true"/>
</entity>
```
另一个由转换器和`sourceColName`引起的问题是会让`oracle.sql.CLOB@aed3a5`这样的数据进入你索引数据中去.

2. 注意列名的大小写!我建议值使用大写字母.如果指定字段列为`FROM_SERVICE_Dt`,但是查询语句中的列名为`FROM_SERVICE_DT`,那么你不会看到任何的错误,但是你不会在字段中获取任何的数据!


---
原文链接:[https://wiki.apache.org/solr/DataImportHandler](https://wiki.apache.org/solr/DataImportHandler)
翻译:[翔妖除魔](sunxiang.cn)
时间:2015-8-14 23:53:40

            

