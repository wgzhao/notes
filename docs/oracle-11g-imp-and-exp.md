# Oracle 11g 导入导出基本操作

[Source](http://www.liuhuachao.com/blog/2018/10/23/oracle/ "Permalink to oracle 11g 数据库的导入导出")

## 1\. oracle 11g 数据库的导入导出

* *

* oracle11g 数据库的导入/导出，就是我们通常所说的oracle数据的还原/备份。
* 数据库导入:把.dmp 格式文件从本地导入到数据库服务器中。
* 数据库导出:把数据库服务器中的数据导出到本地生成.dmp格式文件。

## 2\. oracle 11g 数据库导入导出的3种方式

* *

* 传统方式：exp(导出)和imp（导入）;
* 数据泵方式：expdp导出和impdp（导入）;
* 第三方工具：PL/sql Develpoer，Navicat等;

## 3\. 3种方法的优缺点

* *

3.1 exp/imp:

* 优点:代码书写简单易懂,从本地即可直接导入，不用在服务器中操作，降低难度，减少服务器上的操作也就 保证了服务器上数据文件的安全性。
* 缺点：这种导入导出的速度相对较慢，合适数据库数据较少的时候。如果文件超过几个G，大众性能的电 脑，至少需要4~5个小时左右。

3.2 expdp/impdp

* 优点:导入导出速度相对较快,几个G的数据文件一般在1~2小时左右。
* 缺点:代码相对不易理解,要想实现导入导出的操作，必须在服务器上创建逻辑目录(不是真正的目录)。我们 都知道数据库服务器的重要性，所以在上面的操作必须慎重。所以这种方式一般由专业的程序人员来完 成(不一定是DBA(数据库管理员)来干,中小公司可能没有DBA)。

3.3 PL/sql Develpoer:

* 优点：封装了导入导出命令，无需每次都手动输入命令。方便快捷，提高效率。
* 缺点：长时间应用会对其产生依赖，降低对代码执行原理的理解。

## 4.导入导出方法与实例

* *

### 4.1 传统方式(exp/imp)

4.1.1 通用命令

* exp(imp) username/password@servicename:1521 file=”d:\\temp.dmp” full = y;

4.1.2 参数解释

* exp:导出命令，导出时必写。
* imp:导入命令，导入时必写,每次操作，二者只能选择一个执行。
* username:导出数据的用户名，必写;
* password:导出数据的密码，必写;
* @:地址符号，必写;
* servicename:Oracle的服务名，必写;
* 1521:端口号，1521是默认的可以不写,非默认要写;
* file=”e:\\temp.dmp” : 文件存放路径地址，必写;
* full=y :表示全库导出。可以不写，则默认为no,则只导出用户下的对象

4.1.3 导出导入实例

* 按全库导 exp(imp) system/wisense@orcl file=”e:\\temp.dmp” full = y;
* 按用户导 exp(imp) system/wisense@orcl file=”e:\\temp.dmp” owner(username1,username2,username3,…);
* 按表名导 exp(imp) system/wisense@orcl file=”e:\\temp.dmp” tabels= (table1,table2,table3,…);
* 按表空间导 exp(imp) system/wisense@orcl file=”e:\\temp.dmp” tablespaces= (tablespace1,tablespace2,tablespace3,…);

### 4.2 数据泵方式（expdp/impdp）

4.2.1 通用命令 expdp(impdp) username/password@servicename:1521 schemas=username directory=dumpdir dumpfile=temp.dmp [remap\_schema=origin\_username:target\_username] [remap\_tablespace=origin\_tablespace:target\_tablespace] [logfile=file1.log];

4.2.2 参数解释

* exp:导出命令，导出时必写;
* imp:导入命令，导入时必写,每次操作，二者只能选择一个执行;
* username:导出数据的用户名，必写;
* password:导出数据的密码，必写;
* @:地址符号，必写;
* servicename:Oracle的服务名，必写;
* 1521:端口号，1521是默认的可以不写,非默认要写;
* schemas：导出操作的用户名;
* remap\_schema=源数据库用户名:目标数据库用户名,二者不同时必写，相同可以省略;
* remap\_tablespace=源数据库表空间:目标数据库表空间,二者不同时必写，相同可以省略;
* directory:创建的文件夹名称;
* dumpfile：导出的文件;
* logfile:导出的日志文件,可以不写；

4.2.3 导出导入实例

* 按全库导 expdp(impdp) system/wisense@orcl directory=data\_pump\_dir dumpfile=temp.dmp FULL=y;
* 按用户导 expdp(impdp) system/wisense@orcl schemas=username directory=data\_pump\_dir dumpfile=temp.dmp;
* 按表名导 expdp(impdp) system/wisense@orcl tables=emp,dept directory=data\_pump\_dir dumpfile=temp.dmp;
* 按表空间导 expdp(impdp) system/wisense@orcl tablespaces=temp,example directory=data\_pump\_dir dumpfile=temp.dmp;
* 按查询条件导 expdp(impdp) system/wisense@orcl tables=emp query=’sqlwhere’ directory=data\_pump\_dir dumpfile=temp.dmp;
* 改变用户和表空间导入 impdp system/wisense@orcl schemas=username directory=data\_pump\_dir dumpfile=temp.dmp remap\_schema=origin\_username:target\_username remap\_tablespace=origin\_tablespace:target\_tablespace
* 追加数据 impdp system/wisense@orcl schemas=system table\_exists\_action directory=data\_pump\_dir dumpfile=temp.dmp ;

4.2.4 其他命令

* 连接登录 sqlplus /nolog conn system/wisense
* 查看表空间 select \* form dba\_tablespaces;
* 创建表空间 create tablespace ‘wisense’ datafile ‘d:\\app\\Administrator\\oradata\\orcl\\wisense.dbf’ size 100M autoextend on next 1M maxsize 20480M（unlimited） extent management local;
* 创建逻辑目录 create directory dumpdir as ‘d:\\oracle\_dump\_dir';
* 创建用户 create user username identified by username default tablespace ts\_username;
* 给用户赋予在指定目录的操作权限 grant read,write on directory dumpdir to username;