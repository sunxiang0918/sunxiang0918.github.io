title: DB2下使用Hibernate自动重建表导致整个DB2卡住的问题分析
date: 2014-07-28 09:51:28
tags:
- JAVA
- Hibernate
- DB2
toc: false
comments: true
---

#DB2下使用Hibernate自动重建表导致整个DB2卡住的问题分析


##问题现象:
今天在做高级检索的时候,当用户在界面配置了高级检索的字段后,需要程序自动的把AI3_ADVANCEDSEARCH表删除后,重建.本来这个功能在XDA和xSpace上是能正常工作的.
这两个系统所使用的数据库是 MYSQL和ORACLE.一切正常.  现在TVBS使用的是DB2. 在做测试的时候发现.**每当重建表的时候,整个重建的线程都卡住了**.同时,**使用DB2的SQL管理工具,对任意的表进行修改/删除/新建,都会卡住**.这肯定是有问题的.

##原因分析过程:

1. 最开始就怀疑是hibernate和DB2的配合的问题,于是打断点.一路跟踪到它卡住的地方.  
![](/img/2014/07/28/1.png)  
就是这里,调用JDBC的statement.executeUpdate就会卡住. 这句是标准的JDBC的代码.其中statement的实例是DB2的驱动中的对象,于是<span style="background:yellow">怀疑是DB2的驱动问题</span>.于是从网上重新找驱动.
<!--more-->
2. DB2的驱动是个锤子,官网上根本下不到.需要验证码,验证码IBM又不发给我们.于是只能大海捞针的到处找.终于找到了DB2 9.5的驱动.<span style="background:red">于是换上新的驱动,继续测试. 结果依旧.</span>问题没有解决.看来不是驱动的问题.

3. 换一种思路.写了一段小的JDBC程序来执行创建表的操作:  
![](/img/2014/07/28/2.png)  
这样操作又能成功. 于是对比分析两次执行的差别.<span style="background:yellow">发现  statement对象的类型不同</span>.
	简单的JDBC使用的是 db2驱动中的类`com.ibm.db2.jcc.am.tm.`而MAMSpace中使用的是JBOSS封装过的类`org.jboss.resource.adapter.jdbc.WrappedStatement` 
	于是<span style="background:yellow">怀疑是JBOSS的JNDI数据库连接池的问题.</span>
	
4. 马上<span style="background:yellow">更换我们dcmp使用的连接池.</span>不是用JBOSS的JNDI数据库连接池.然后再次进行测试.
	<span style="background:red">结果……问题依旧.....</span>
	
5. 重启DB2....再次把小程序和MAMSpace都断点到同一位置.然后<span style="background:yellow">开始怀疑是否是事务等问题.</span>于是分析两个statment对象中属性的不同. 但是....一个驱动.居然混淆了的.  
![](/img/2014/07/28/3.png)  
它所有的字段,属性全部都是 a b c d 的命名.<span style="background:red">大致对比了一下.还是没发现问题.</span>

6. 无耐.只有重新再来.经过五六次的不断断点,不断重启DB2. **发现一个很怪异的现象**.
    就是 每次卡住以后,只有重启DB2才能恢复 DB2对数据表的DDL功能.
并且,<span style="background:yellow">重启后,会发现其实这个时候DB2已经把 AI3_ADVANCEDSEARCH表建出来了. 但是缺少一个字段.</span>
	建表语句:
	
	```sql
	create table "AI3_ADVANCESEARCH" (ID bigint not null,ENTITYCONTENTID varchar(255),ENTITYID bigint,ENTITYTYPE varchar(255),rEvtl varchar(255),primary key (ID))
	```
	
	卡住重启后,建出来的表:   
	![](/img/2014/07/28/4.png)  
	注意,与建表语句相比,差了 rEvtl字段.  于是<span style="background:yellow">开始怀疑是否是这个字段有问题.</span>

7.再一次重启DB2和DCMP,并断点到执行建表语句的位置. 这次直接把内存中的SQL语句给更改了.改成:

```sql
	create table "AI3_ADVANCESEARCH" (ID bigint not null,ENTITYCONTENTID varchar(255),ENTITYID bigint,ENTITYTYPE varchar(255),primary key (ID))
```
也就是去掉rEvtl字段,只建4个基本字段.
<span style="background:green">果然这次程序没有被卡住.</span>
于是,马上回跳程序.<span style="background:yellow">在不重启服务器和DB2的情况下</span>,执行有rEvtl字段的建表语句.也就是会被卡住的DDL. <span style="background:green">结果执行成功.并没有被卡住.....</span>  无语了.只能<span style="background:yellow">怀疑是DB2的内部处理中有甚么特殊的地方.</span>

8. 再一次重启DB2和DCMP.这次直接把内存中SQL语句给更改成:

```sql
create table "AI3_ADVANCESEARCH" (ID bigint not null,ENTITYCONTENTID varchar(255),ENTITYID bigint,ENTITYTYPE varchar(255),NHNXL varchar(255),primary key (ID))
```
也就是第5个字段改成大写.<span style="background:red">测试结果还是失败</span>...看来<span style="background:yellow">不是字段大小写的问题.</span>

9. 再一次重启DB2和DCMP.这次直接把内存中SQL语句中第五个字段的字段类型改成bigint.:

```sql
create table "AI3_ADVANCESEARCH" (ID bigint not null,ENTITYCONTENTID varchar(255),ENTITYID bigint,ENTITYTYPE varchar(255),OSEBi bigint,primary key (ID))
```
<span style="background:red">测试结果再次失败</span>....看来<span style="background:yellow">不是字段类型的问题.</span>

10. 再一次重启DB2和DCMP.这次直接把内存中SQL语句改成:

```sql
create table AI3_ADVANCESEARCH (ID bigint not null,ENTITYCONTENTID varchar(255),ENTITYID bigint,ENTITYTYPE varchar(255),ENldl varchar(255),primary key (ID)) IN TBL_SMAM INDEX IN IDX_SMAM
```
也就是把表名的双引号去掉.增加上表空间. <span style="background:red">测试结果依然失败</span>.....看来<span style="background:yellow">不是表名和表空间的问题.</span>

11. 再一次重启DB2和DCMP.这次直接把内存中SQL语句改成:

```sql
create table AI3_ADVANCESEARCH (ID bigint not null,ENTITYCONTENTID varchar(255),ENTITYID bigint,QMwQH varchar(255),primary key (ID)) IN TBL_SMAM INDEX IN IDX_SMAM
```
也就是把ENTITYTYPE这个字段删掉,<span style="background:red">测试结果还是不行</span>....

12. 转变一下思路, 先在SQL管理器中把表删除了. 然后再在程序里面创建表.<span style="background:green">就OK了.</span>
转变一下,由此可见,<span style="background:yellow">问题并不在于Create表的地方.应该是drop表出的问题.</span>

13. 既然出在drop表的地方.那么我在这个地方是不是可以不调用drop表喃?于是写了一段存储过程.

```sql
CREATE PROCEDURE "MAMDBA"."DROPADVANCESEARCH"()
LANGUAGE SQL
SPECIFIC DROPADAVANCESEARCH
BEGIN
	IF EXISTS (select * from sysibm.systables where TID <> 0 and name = 'AI3_ADVANCESEARCH' ) then
		drop table AI3_ADVANCESEARCH;
	END IF;
END
```
然后在调用删除的地方直接调用存储过程来进行删除: “call dropadvancesearch()”.    <span style="background:red">测试结果还是不得行.</span>

14. 再一次的对比了一下成功的情况. 发现<span style="background:yellow">如果drop表的语句是通过hibernate执行的,那么,虽然在SQL管理器中看到表确实是删除了,日志中事务确实是提交了.但是再次建表的时候如果新建的表结构与被删除的表表结构不同,DB2就会被卡住.</span> 重启DB2后,被删除的表又冒出来了.<span style="background:yellow">这可能和DB2的某个执行有关.但是网上DB2的资料太少了.找不出问题的原因</span>.因此,<span style="background:blue">最后.只能在删除表的地方判断下是否是DB2,如果是,那么就直接使用JDBC来删除表而不使用hibernate来执行删除</span>. 重启后,<span style="background:green">问题解决.</span>  
![](/img/2014/07/28/5.png) 

___
##总结:
这个问题相当的奇怪,到最后都没找出真正的问题原因.在MYSQL和Oracle中不存在这个问题.因此可以确定应该是DB2中的某些处理相当奇怪,造成了在Hibernate管理事务的情况下.会造成DB2删除表在DB2内部发生死锁.于是乎,导致整个DB2停止响应.当重启后,真正的表并没有删除,因此,如果又建立一张和以前被删除的表的名字一样的表的话,是有问题的.