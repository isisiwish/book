# Tomcat性能优化 [ [配套教程](https://coding.imooc.com/class/442.html) ]

[TOC]

tomcat服务器在JavaEE项目中使用率非常高，所以在生产环境对tomcat的优化也变得非常重要了。

本专题课在调优压测时我们选择tomcat9，那我们为什么选择9版本呢？因为9的性能更高，更稳定。

在源码研究演示过程中我会选择tomcat7，8，9三套源码进行比较演示性能的差异。

**技术交流q群：963060292**

## 一、Tomcat配置优化

对于tomcat的优化，主要是从2个方面入手，一是，tomcat自身的配置，另一个是tomcat所运行的jvm虚拟机的调优。

### 1、部署安装tomcat9

1、下载并安装：
https://tomcat.apache.org/download-90.cgi

![1590121351339](https://smartan123.github.io/book/library/002-images/1590121351339.png)



2、wget镜像安装

```she
cd /usr/local

wget https://mirrors.cnnic.cn/apache/tomcat/tomcat-9/v9.0.33/bin/apache-tomcat-9.0.33.tar.gz

tar ‐zxvf apache‐tomcat‐9.0.33.tar.gz

mv apache‐tomcat‐9.0.33 tomcat9

cd tomcat9/conf

#修改配置文件，配置tomcat的管理用户
vi tomcat‐users.xml

#写入如下内容：
<role rolename="manager"/>
<role rolename="manager‐gui"/>
<role rolename="admin"/>
<role rolename="admin‐gui"/>
<user username="tomcat" password="tomcat" roles="admin‐gui,admin,manager‐
gui,manager"/>
#保存退出

#如果是tomcat7，配置了tomcat用户就可以登录系统了，但是tomcat9中不行，还需要修改
另一个配置文件，否则访问不了，提示403
vim webapps/manager/META‐INF/context.xml
#将<Valve的内容注释掉
<Context antiResourceLocking="false" privileged="true" >
<!‐‐ <Valve className="org.apache.catalina.valves.RemoteAddrValve"
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /> ‐‐>
<Manager sessionAttributeValueClassNameFilter="java\.lang\.
(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.Cs
rfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>

#保存退出即可

#启动tomcat
cd /usr/local/tomcat9/bin/
./startup.sh && tail ‐f ../logs/catalina.out

#打开浏览器进行测试访问
http://192.168.0.108:8080/
```

![1590121685453](https://smartan123.github.io/book/library/002-images/1590121685453.png)

点击“Server Status”，输入用户名、密码进行登录，tomcat/tomcat 

![1590121775427](https://smartan123.github.io/book/library/002-images/1590121775427.png)

![1590121821075](https://smartan123.github.io/book/library/002-images/1590121821075.png)

进入之后即可看到服务器的信息。（进去看看）

**ps：安全起见，生产环境会禁用这个管理界面，最直接的办法是删除webapp下的默认项目。因为我们根本不需要从界面上部署。**



### 2、禁用ajp协议(8.5.51之前的版本默认是开启的，后续的版都是禁用的)

在服务状态页面中可以看到，默认状态下会启用AJP服务，并且占用8009端口 。

![1584371833741](https://smartan123.github.io/book/library/002-images/1584371833741.png)

**ps：为了演示，需要把server.xml文件ajp connector屏蔽段放开**



什么是AJP呢？
AJP（Apache JServer Protocol）
AJPv13协议是面向包的。WEB服务器和Servlet容器通过TCP连接来交互；为了节省SOCKET创建的昂贵代价，WEB服务器会尝试维护一个永久TCP连接到servlet容器，并且在多个请求和响应周期过程会重用连接。 

![1584372984407](https://smartan123.github.io/book/library/002-images/1584372984407.png)

我们一般是使用Nginx+tomcat的架构，所以用不着AJP协议，所以把AJP连接器禁用。
修改conf下的server.xml文件，将AJP服务禁用掉即可 。

```xml
<!--<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />-->
```

重启tomcat，查看效果。

可以看到AJP服务以及不存在了。

ps：禁用ajp后，看节省了多少内存？？查询某个pid占多少内存

### 3、执行器（线程池） 

在tomcat中每一个用户请求都是一个线程，所以可以使用线程池提高性能。
修改server.xml文件：

```xml
<!‐‐将注释打开（注释没打开的情况下默认10个线程，最小10，最大200）‐‐>
<Executor name="tomcatThreadPool" namePrefix="catalina‐exec‐"
maxThreads="500" minSpareThreads="50"
prestartminSpareThreads="true" maxQueueSize="100"/>
<!‐‐
参数说明：
maxThreads：最大并发数，默认设置 200，一般建议在 500 ~ 1000，根据硬件设施和业
务来判断
minSpareThreads：Tomcat 初始化时创建的线程数，默认设置 25
prestartminSpareThreads： 在 Tomcat 初始化的时候就初始化 minSpareThreads 的
参数值，如果不等于 true，minSpareThreads 的值就没啥效果了
maxQueueSize，最大的等待队列数，超过则拒绝请求
‐‐>
<!‐‐在Connector中设置executor属性指向上面的执行器‐‐>
<Connector executor="tomcatThreadPool" port="8080" protocol="HTTP/1.1"
connectionTimeout="20000"
redirectPort="8443" />
```

保存退出，重启tomcat，查看效果。

**ps：idea中源码启动或者用jvisualvm查看线程的变化**

### 4、3种运行模式 

tomcat的运行模式有3种：

***ps：每个模式都需要线程演示查看***

1)、bio（tomcat7演示，压测看线程增长）

默认的模式,性能非常低下,没有经过任何优化处理和支持，tomcat8.5已经舍弃了该模式，默认就是nio模式。

2)、nio（nio2）

nio(new I/O)，是Java SE 1.4及后续版本提供的一种新的I/O操作方式(即java.nio包及其子包)。Java nio是一个基于缓冲区、并能提供非阻塞I/O操作的Java API，因此nio也被看成是non-blocking I/O的缩写。它拥有比传统I/O操作(bio)更好的并发运行性能。***NIO2异步的本质是数据从内核态到用户态这个过程是异步的，也就是说nio中这个过程必须完成了才执行下个请求，而nio2不必等这个过程完成就可以执行下个请求，nio2的模式中数据从内核态到用户态这个过程是可以分割的。***

3)、apr

apr(Apache portable Run-time libraries/Apache可移植运行库)是Apache HTTP服务器的支持库。

安装起来最困难,但是从操作系统级别来解决异步的IO问题,大幅度的提高性能。推荐使用nio，不过，在tomcat8及后续的版本中有最新的nio2，速度更快，建议使用nio2。

设置nio2：

```xml
<Connector executor="tomcatThreadPool" port="8080"
protocol="org.apache.coyote.http11.Http11Nio2Protocol"
connectionTimeout="20000"
redirectPort="8443" />
```



**ps：为什么nio快呢？**

*简单地说，nio 模式最大化压榨了CPU，把时间片更好利用起来。通俗地说，bio hold住连接不干活也占用线程，nio hold住连接不干活也没关系，让需要处理的连接执行就行了。*

![1584412993647](https://smartan123.github.io/book/library/002-images/1584412993647.png)

可以看到已经设置为nio2了 。

如果通道选择apr，apr需要独立安装。

apr安装步骤：

**1、先安装gcc， expat-devel，perl-5**

```shell
yum install gcc

yum install expat-devel

cd /usr/local

wget ftp://mirrors.ustc.edu.cn/CPAN/src/5.0/perl-5.30.1.tar.gz
tar -xzf perl-5.30.1.tar.gz
cd perl-5.30.1
./Configure -des -Dprefix=$HOME/localperl
make
make install
```

**2、安装apr**

```shell
cd /usr/local

wget https://mirrors.cnnic.cn/apache/apr/apr-1.6.5.tar.gz

tar -zxvf apr-1.6.5.tar.gz

cd  apr-1.6.5

./configure --prefix=/usr/local/apr && make && make install
```

**3、安装apr-util**

```shell
cd /usr/local

wget  https://mirrors.cnnic.cn/apache/apr/apr-util-1.6.1.tar.gz

##安装apr-util前请确认系统是否安装了expat-devel包，如没安装请安装，不然会报错。yum install expat-devel

tar -zxvf apr-util-1.6.1.tar.gz

cd apr-util-1.6.0

./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr && make && make install
```

**4、安装openssl**

```shell
cd /usr/local

wget https://www.openssl.org/source/openssl-1.0.2l.tar.gz

tar -zxvf openssl-1.0.2l.tar.gz

cd openssl-1.0.2l

./configure --prefix=/usr/local/openssl shared zlib && make && make install

##缺少zlib，会报错,所以得先安装zlib

cd /usr/local

**wget  http://www.zlib.net/zlib-1.2.11.tar.gz**

tar -zxvf zlib-1.2.1.tar.gz

cd zlib-1.2.11

##因为要用共享方式安装，所以执行以下命令

make clean && ./configure --shared && make test && make install

cp zutil.h /usr/local/include

cp zutil.c /usr/local/include

##重新执行

./configure --prefix=/usr/local/openssl shared zlib && make && make install

##检查openssl是否安装成功

/usr/local/openssl/bin/openssl version -a 显示1.0.2l版本为成功
```

**5、安装tomcat-native**

```shell
tar /usr/local/tomcat9/bin/tomcat-native.tar.gz

cd /usr/local/tomcat9/bin/tomcat-native-1.2.12-src/native

./configure --with-apr=/usr/local/apr --with-java-home=/usr/local/jdk/ --with-ssl=/usr/local/openssl/ && make && make install
```

**6、使tomcat支持apr配置apr库文件**

##方式1：配置坏境变量：

```shell
echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib" >> /etc/profile

echo "export LD_RUN_PATH=$LD_RUN_PATH:/usr/local/apr/lib" >> /etc/profile && source /etc/profile
```

##方式2：catalina.sh脚本文件：在注释行# Register custom URL handlers下添加一行 

```shell
JAVA_OPTS="$JAVA_OPTS -Djava.library.path=/usr/local/apr/lib"
```

**7、修改tomcat server.xml文件（把protocol修改成org.apache.coyote.http11.Http11AprProtocol）**

```xml
<Connector port="8080" protocol="org.apache.coyote.http11.Http11AprProtocol"
connectionTimeout="20000"
redirectPort="8443" />
```

**8、启动tomcat**

```shell
cd /usr/local/tomcat8/bin

./startup.sh
```

**9、查看tomcat是否以http-apr模式运行，可以查看tomcat管理界面，也可以远程jmx监控查看**

```shell
##连接远程jmx监控需要在catalina.sh文件中加上
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote -Djava.rmi.server.hostname=192.168.0.107 -Dcom.sun.management.jmxremote.port=9999  -Dcom.sun.management.jmxremote.rmi.port=9999 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
```

查看管理页面

![1584455329184](https://smartan123.github.io/book/library/002-images/1584455329184.png)



## 二、部署测试用的java web项目

### 1、部署web应用

部署一个简单的servlet测试用例，模拟业务耗时2000ms

![1590120356923](https://smartan123.github.io/book/library/002-images/1590120356923.png)

### 2、准备好test-web.war上传到linux服务器，进行部署安装

**部署过程见视频**

### 3、访问首页，查看是否已经启动成功：http://192.168.0.108:8080/test-web

![1590120456428](https://smartan123.github.io/book/library/002-images/1590120456428.png)

出现此页面说明部署成功！



## 三、使用Apache JMeter进行测试

### 1、下载安装：

http://jmeter.apache.org/download_jmeter.cgi 

![1590119770127](https://smartan123.github.io/book/library/002-images/1590119770127.png)



### 2、修改主题和语言：主题修改为白底黑字，语言修改为中文

![1590119834399](https://smartan123.github.io/book/library/002-images/1590119834399.png)



### 3、创建首页的测试用例：详见操作

线程组设置：1000个线程，每个线程循环10次，间隔为1秒（具体其他设置见视频）

![1590119898801](https://smartan123.github.io/book/library/002-images/1590119898801.png)

### 4、启动、进行测试：详见操作

设置完毕后点击绿色的三角按钮启动压测（具体其他操作见视频）

![1590119997149](https://smartan123.github.io/book/library/002-images/1590119997149.png)



### 5、聚合报告：聚合报告中要具体查看吞吐量

具体的压测结果在聚合报告中查看，如下图所示

![1590120074217](https://smartan123.github.io/book/library/002-images/1590120074217.png)



## 四、调整tomcat参数进行优化

**以下所有步骤的具体演示见视频**

### 1、禁用AJP服务

具体禁用操作：在conf/server.xml中禁用以下配置：

（tomcat曝出ajp漏洞后，官方已打完补丁，现在该配置默认是屏蔽的。本人在b站有使用python脚本模拟黑客利用ajp漏洞进行攻击获取class文件的免费视频演示，感兴趣的同学可以去看下）：

![1590113891490](https://smartan123.github.io/book/library/002-images/1590113891490.png)

禁用后进行压测，查看吞吐量

![1590114400031](https://smartan123.github.io/book/library/002-images/1590114400031.png)

压测结果如下（禁用ajp）

**平均时间:1.728s，异常率：0.73%，吞吐量：384.3/s**



取消ajp屏蔽后，重启tomcat，继续压测，查看吞吐量

![1590114674399](https://smartan123.github.io/book/library/002-images/1590114674399.png)

压测结果如下（使用ajp）

**平均时间:1.431s，异常率：0.46%，吞吐量：362.8/s**

比较两次压测结果发现，禁用后吞吐量是有所上升的。

### 2、设置线程池

1)、不设置线程池，业务延时设置1000ms，压测后的吞吐量如下

![1590115518298](https://smartan123.github.io/book/library/002-images/1590115518298.png)

**平均时间:4900ms，异常率：0.00%，吞吐量：191.8/s**



2)、不设置线程池，业务延时设置2000ms，压测后的吞吐量如下

![1590115842468](https://smartan123.github.io/book/library/002-images/1590115842468.png)

**平均时间:9666ms，异常率：0.00%，吞吐量：98.5/s**

**业务延时之后，平均时间翻了一倍，吞吐量减少一半，总体性能下降了两倍。**

**由此可见，业务时间执行的长短直接影响吞吐量和执行时间**



3)、设置线程池，最大线程设置200（跟不设置线程池时是一样的，默认最大就是200个线程），继续压测

![1590116819697](https://smartan123.github.io/book/library/002-images/1590116819697.png)

压测后吞吐量如下

![1590116912938](https://smartan123.github.io/book/library/002-images/1590116912938.png)

**平均时间:8884ms，异常率：0.00%，吞吐量：96.3/s**

**跟第二次的压测结果差不多（因为虽然加了线程池，但是最大线程设置的是200，和不加线程池是一样的）**



4)、设置线程池，最大线程设置为400（最大线程数扩大了1倍），继续压测

![1590117083659](https://smartan123.github.io/book/library/002-images/1590117083659.png)

压测后吞吐量如下

![1590117199458](https://smartan123.github.io/book/library/002-images/1590117199458.png)

**平均时间:4946ms，异常率：0.00%，吞吐量：191.7/s**

**由此可见，最大线程扩大1倍后，平均时间缩短了1倍，吞吐量还扩大了1倍，总体性能提升2倍**



5)、设置线程池，最大线程扩大到800（最大线程数扩大了4倍），继续压测

![1590117358502](https://smartan123.github.io/book/library/002-images/1590117358502.png)

压测后吞吐量如下

![1590117530324](https://smartan123.github.io/book/library/002-images/1590117530324.png)

**平均时间:2589ms，异常率：0.00%，吞吐量：358.7/s**

**由此可见，线程数再一次扩大后，时间又缩短了1倍，吞吐量又上升了接近1倍，总体性能提升近2倍**



6)、设置线程池，最大线程扩大到1600（最大线程数扩大了8倍），继续压测（看什么时候出现异常）

![1590117709683](https://smartan123.github.io/book/library/002-images/1590117709683.png)

压测后吞吐量如下

![1590117804724](https://smartan123.github.io/book/library/002-images/1590117804724.png)

**平均时间:2356ms，异常率：0.00%，吞吐量：387.9/s**

**由于jmeter设置的总线程数是1000，这就压制了我们设置的1600的线程总数，所以压测结果和第三次差不多**



7)、设置线程池，最大线程1600不变，将jmeter的压测线程设置为2000，继续压测（这时候还是没有出现异常）

![1590117709683](https://smartan123.github.io/book/library/002-images/1590117709683.png)

压测后吞吐量如下（下图右上角jmeter压测线程数改成了2000）

![1590118102880](https://smartan123.github.io/book/library/002-images/1590118102880.png)

**平均时间:2550ms，异常率：0.00%，吞吐量：682.2/s**

**平均时间差不多，但是吞吐量还是上升了1倍左右，总体性能提升1倍（总体性能的提升幅度比前几次下降了1倍），因此并不是可以无限制的扩大线程数来提升性能，总有出现瓶颈的时候**



8)、设置线程池，最大线程扩到3000，继续压测（看会不会出现异常）

![1590118436454](https://smartan123.github.io/book/library/002-images/1590118436454.png)

压测后吞吐量如下

![1590118530900](https://smartan123.github.io/book/library/002-images/1590118530900.png)

**平均时间:2908ms，异常率：0.00%，吞吐量：607.2/s**

**比上次稍微差点，因为jmeter的线程总数还是2000个，都已经打满，时间增加了，吞吐量也下降了，所以线程超过1600之后，貌似总体性能开始下滑**



9)、设置线程池，最大线程3000不变，jmeter线程增加到4000，继续压测（看会不会出现异常）

![1590118436454](https://smartan123.github.io/book/library/002-images/1590118436454.png)

压测后吞吐量如下

![1590118965685](https://smartan123.github.io/book/library/002-images/1590118965685.png)

**终于出现异常了！**

**平均时间:5027ms，异常率：15.34%（最大达到50%+），吞吐量：715.8/s**

**和上次比，时间拉长了近1倍，异常达到15%，吞吐量比上次多了一点 。由此可见，我的机器的极限就是吞吐量在600-700/s,能承受的线程总数也就是在2000-3000之间**。



**下面还有设置NIO2通道等多种情况的压测性能演示，大家可以观看视频教程，本文不再赘述。**

### 3、总结

性能压测必须经过多次的调试压测，最终才能获得一个较为理想的结果，而且不同的软硬件环境压测出来的结果都是不一样的，所以压测建议在灰度环境压，如果情况允许的话可以直接在生产上压，这样得出的结果更准。



## 五、Tomcat堆栈中常见线程

Tomcat作为一个服务器来讲，必然运行着很多的线程，而每一个线程究竟是干什么的，这个需要非常的清楚，无论是打印断点，还是通过jstack进行线程栈分析，这都是必须要掌握的技能。
本文带你基于Tomcat7，8，9的版本，识别Tomcat堆栈中的线程。

![1590113416912](https://smartan123.github.io/book/library/002-images/1590113416912.png)

### 1、main线程

main线程是tomcat的主要线程，其主要作用是通过启动包来对容器进行点火：

main线程一路启动了Catalina，StandardServer[8005]，StandardService[Catalina]，StandardEngine[Catalina]

​	    		engine内部组件都是异步启动，engine这层才开始继承ContainerBase，engine会调用父类的startInternal()方法，里面由startStopExecutor线程提交FutureTask任务，异步启动子组件StandardHost，

​				StandardEngine[Catalina].StandardHost[localhost]

main->Catalina->StandardServer->StandardService->StandardEngine->**StandardHost**，黑体开始都是异步启动。

​																							->启动Connector

main的作用就是把容器组件拉起来，然后阻塞在8005端口，等待关闭。

![1584195719645](https://smartan123.github.io/book/library/002-images/1584195719645.png)



### 2、localhost-startStop线程

Tomcat容器被点火起来后，并不是傻傻的按照次序一步一步的启动，而是在engine组件中开始用该线程提交任务，按照层级进行异步启动，对于每一层级的组件都是采用startStop线程进行启动，我们观察一下idea中的线程堆栈就可以发现：启动异步，部署也是异步

![1584196385034](https://smartan123.github.io/book/library/002-images/1584196385034.png)



这个startstop线程实际代码调用就是采用的JDK自带线程池来做的，启动位置就是ContainerBase的组件父类的startInternal()：

![1584196048806](https://smartan123.github.io/book/library/002-images/1584196048806.png)

因为从Engine开始往下的容器组件都是继承这个ContainerBase，所以相当于每一个组件启动的时候，除了对自身的状态进行设置，都会启动startChild线程启动自己的孩子组件。

而这个线程仅仅就是在启动时，当组件启动完成后，那么该线程就退出了，生命周期仅仅限于此。

### 3、AsyncFileHandlerWriter线程

日志输出线程：

![1584196714178](https://smartan123.github.io/book/library/002-images/1584196714178.png)

顾名思义，该线程是用于异步文件处理的，它的作用是在Tomcat级别构架出一个输出框架，然后不同的日志系统都可以对接这个框架，因为日志对于服务器来说，是非常重要的功能。

如下，就是juli的配置：

![1584196877893](https://smartan123.github.io/book/library/002-images/1584196877893.png)

该线程主要的作用是通过一个LinkedBlockingDeque来与log系统对接，该线程启动的时候就有了，全生命周期。

### 4、ContainerBackgroundProcessor线程

Tomcat在启动之后，不能说是死水一潭，很多时候可能会对Tomcat后端的容器组件做一些变化，例如部署一个应用，相当于你就需要在对应的Standardhost加上一个StandardContext，也有可能在热部署开关开启的时候，对资源进行增删等操作，这样应用可能会重新reload。

也有可能在生产模式下，对class进行重新替换等等，这个时候就需要在Tomcat级别中有一个线程能实时扫描Tomcat容器的变化，这个就是ContainerbackgroundProcessor线程了：

（本地源码StandardContext类的5212行启动）

![1584200549363](https://smartan123.github.io/book/library/002-images/1584200549363.png)



我们可以看到这个代码，也就是在ContainerBase中：

![1584200704730](https://smartan123.github.io/book/library/002-images/1584200704730.png)

这个线程是一个递归调用，也就是说，每一个容器组件其实都有一个backgroundProcessor，而整个Tomcat就点起一个线程开启扫描，扫完儿子，再扫孙子（实际上来说，主要还是用于StandardContext这一级，可以看到StandardContext这一级：

![1584201033048](https://smartan123.github.io/book/library/002-images/1584201033048.png)

![1584201068001](https://smartan123.github.io/book/library/002-images/1584201068001.png)

我们可以看到，每一次backgroundProcessor，都会对该应用进行一次全方位的扫描，这个时候，当你开启了热部署的开关，一旦class和资源发生变化，立刻就会reload。

**tomcat9中已经被Catalina-Utility线程替代。**

### 5、acceptor线程

Connector(实际是在**AbstractProtocol**类中)初始化和启动之时，启动了Endpoint，Endpoint就会启动**poller**线程和**Acceptor**线程。Acceptor底层就是ServerSocket.accept()。返回Socket之后丢给NioChannel处理,之后通道和poller线程绑定。

***acceptor->poller->exec***

无论是NIO还是BIO通道，都会有Acceptor线程，该线程就是进行socket接收的，它不会继续处理，如果是NIO的，无论是新接收的包还是继续发送的包，直接就会交给Poller，而BIO模式，Acceptor线程直接把活就给工作线程了：

![1584238520638](https://smartan123.github.io/book/library/002-images/1584238520638.png)

如果不配置，Acceptor线程默认开始就开启1个，后期再随着压力增大而增长：

![1584238658710](https://smartan123.github.io/book/library/002-images/1584238658710.png)

上述启动代码在AbstractNioEndpoint的startAcceptorThreads方法中。

### 6、ClientPoller线程

NIO和APR模式下的Tomcat前端，都会有Poller线程：

![1584238019629](https://smartan123.github.io/book/library/002-images/1584238019629.png)

对于Poller线程实际就是继续接着Acceptor进行处理，展开Selector，然后遍历key，将后续的任务转交给工作线程（exec线程），起到的是一个缓冲，转接，和NIO事件遍历的作用，具体代码体现如下（NioEndpoint类）：

![1584238272139](https://smartan123.github.io/book/library/002-images/1584238272139.png)

上述的代码在NioEndpoint的startInternal中，默认开始开启2个Poller线程，后期再随着压力增大增长，可以在Connector中进行配置。

### 7、exe线程（默认10个）

也就是SocketProcessor线程，我们可以看到，上述几个线程都是定义在NioEndpoint内部线程类。NIO模式下，Poller线程将解析好的socket交给SocketProcessor处理，它主要是http协议分析，攒出Response和Request，然后调用Tomcat后端的容器：

![1584239277914](https://smartan123.github.io/book/library/002-images/1584239277914.png)



![1584239091296](https://smartan123.github.io/book/library/002-images/1584239091296.png)

该线程的重要性不言而喻，Tomcat主要的时间都耗在这个线程上，所以我们可以看到Tomcat里面有很多的优化，配置，都是基于这个线程的，尽可能让这个线程减少阻塞，减少线程切换，甚至少创建，多利用。

下面就是NIO模式下创建的工作线程：

![1584239579371](https://smartan123.github.io/book/library/002-images/1584239579371.png)

实际上也是JDK的线程池，只不过基于Tomcat的不同环境参数，对JDK线程池进行了定制化而已，本质上还是JDK的线程池。

### 8、NioBlockingSelector.BlockPoller（默认2个）

Nio方式的Servlet阻塞输入输出检测线程。实际就是在Endpoint初始化的时候启动selectorPool，selectorPool再启动selector，selector内部启动BlokerPoller线程。

![1584242888071](https://smartan123.github.io/book/library/002-images/1584242888071.png)

该线程在前面的NioBlockingPool中讲得很清楚了，其NIO通道的Servlet输入和输出最终都是通过NioBlockingPool来完成的，而NioBlockingPool又根据Tomcat的场景可以分成阻塞或者是非阻塞的，对于阻塞来讲，为了等待网络发出，需要启动一个线程实时监测网络socketChannel是否可以发出包，而如果不这么做的话，就需要使用一个while空转，这样会让工作线程一直损耗。

只要是阻塞模式，并且在Tomcat启动的时候，添加了—D参数 org.apache.tomcat.util.net.NioSelectorShared 的话，那么就会启动这个线程。

大体上启动顺序如下：

```java
//bind方法在初始化就完成了
Endpoint.bind(){
    //selector池子启动
    selectorPool.open(){
        //池子里面selector再启动
         blockingSelector.open(getSharedSelector()){
             //重点这句
              poller = new BlockPoller();
        	  poller.selector = sharedSelector;
        	  poller.setDaemon(true);
              poller.setName("NioBlockingSelector.BlockPoller-"+       (threadCounter.getAndIncrement()));
             //这里启动
              poller.start();
         }
    }
}
```

### 9、AsyncTimeout线程

该线程为tomcat7及之后的版本才出现的，注释其实很清楚，该线程就是检测异步request请求时，触发超时，并将该请求再转发到工作线程池处理（也就是Endpoint处理）。

![1584243575318](https://smartan123.github.io/book/library/002-images/1584243575318.png)

AsyncTimeout线程也是定义在**AbstractProtocol**内部的，在start()中启动。AbstractProtocol是个极其重要的类，**他持有*Endpoint*和*ConnectionHandler***这两个tomcat前端非常重要的类

![1584243290669](https://smartan123.github.io/book/library/002-images/1584243290669.png)

### 10、其他线程（例如ajp相关线程）

ajp工作线程处理的是ajp协议的相关请求，这个请求主要是用于http apache服务器和tomcat之间的数据交换，该数据交换用的就是ajp协议，和exec工作线程差不多，默认也是启动10个，端口号是8009。优化时如果没有用到http apache的话就可以把这个协议关掉。

Tomcat本身还有很多其它的线程，远远不止这些，例如如果开启了sendfile，那么对sendfile就是开启一个线程来进行操作，这种功能的线程开启还有很多。

Tomcat作为一款优秀的服务器，不可能就只有1个线程，而是多个线程之间相互配合完成功能，而且很多功能尽量异步处理，尽可能的减少线程切换。所以线程并不是越多越好，因此线程的控制也尤为关键。

**以上线程的源码分析详细讲解请观看视频**



## 六、NIO连接器前端整体框图

### 1、图解tomcat总体流程（源码详细分析解读见视频）

连接器在Tomcat中是一个重要的组件，叫做Tomcat前端，这个前端框架不是通常我们讲的Web前端，那是structs，javascript，jsp这些内容，这里讲的是以NIO的方式，来描述从socket请求到Request对象的过程，而我们理解的Tomcat后端，通常是以CoyoteAdapter为分界点，后端框架通过Mapper进行映射，可以总结为下面的示意图：

![1590082327148](https://smartan123.github.io/book/library/002-images/1590082327148.png)



Tomcat前端接受的是Socket请求，通过前端框架组件进行http解析，并基于Connector配置的属性做进一步的处理，转化为Tomcat内部的Request对象。

这个位置相当于是一个分界点，也就是CoyoteAdapter类，之后通过Mapper类直接找到Tomcat后端容器中的对应的Servlet，这其中会传过Engine，Host，Context等各种后端容器组件的解析。

最后，转化为Servlet规范的httprequest，作为参数传到业务实现的Servlet中，完成整个请求的过程。

到这里为止，Tomcat前端这块就很清晰明了，在没有见到架构图之前，我们推测，应该貌似有这些处理机制：

1).应该会有线程池做线程支撑的

2).解析http的组件

3).很多线程池是分开的，例如工作线程池和前端socket处理线程池

4).因为我们这篇文章分析的是NIO，所以肯定会有Selector的内容

下面这两张图就是NIO完整的业务流程图和关键组件架构图

### 2、图解tomcat前端详细流程（源码详细分析解读见视频）

![1590082370751](https://smartan123.github.io/book/library/002-images/1590082370751.png)



工作流程的源码注释：

![1590116465754](https://smartan123.github.io/book/library/002-images/1590116465754.png)

![1590116509674](https://smartan123.github.io/book/library/002-images/1590116509674.png)

![1590116636000](https://smartan123.github.io/book/library/002-images/1590116636000.png)

### 3、源码解读tomcat前端关键组件初始化和启动过程（详细分析解读见视频）

![1590082449638](https://smartan123.github.io/book/library/002-images/1590082449638.png)



### 4、Http11NioProtocol（源码详细分析见视频）

http1.1的协议类，实际上这个类的初始化是由对应的Connector类进行初始化，我们可以看看server.xml中关于连接器的配置：

![1590111405669](https://smartan123.github.io/book/library/002-images/1590111405669.png)

对应的协议是HTTP/1.1，对于Connector来讲，有很多的协议处理器：

![1590111922131](https://smartan123.github.io/book/library/002-images/1590111922131.png)

对于普通的NIO连接器，其就是Http11NioProtocol这个类。

从源码分析上来看，直接使用当前线程上下文的类加载器进行加载Http11NioProtocol，并对其构造方法进行初始化。

Http11NioProtocol作为协议的实现者，它持有两大组件：

**一是Endpoint**，默认的就是NioEndpoint，这个类是线程池，socoket的转接类，将NIO通道中的socket组包，交给handler来进行处理。

**二就是Handler**，也就是Http11ConnectionHandler，设置给Endpoint，这个Http11ConnectionHandler类主要的作用是将前面组包socket包，转换成内部的Request对象，最后发给Tomcat的后端。

这两大组件其实也就是上图的最主要的核心部分，是Tomcat前端框架的灵魂。

**该组件详细源码讲解见视频**

### 5、NioEndPoint（源码详细分析见视频）

NioEndPoint类持有三大线程池：

Acceptor（tomcat9版本独立出来了）

PollerEvent

Poller（相当于是reactor）

Worker（exec即SocketProcessor）

从以下类的注释就可以看出来：

![1590111995051](https://smartan123.github.io/book/library/002-images/1590111995051.png)

**a.socket acceptor线程池**

这个线程池中里面的每一个线程中运行的就是NioEndPoint.Acceptor类。这个Acceptor主要的作用并不是直接将这个socket的流取过来，双方进行交互，如果这么做的话，那Tomcat基本就和一个普通的socket程序没有什么区别了。

这个Acceptor首先根据是否是SSL配置，使用Tomcat自身扩展的NioChannel来包装SocketChannel，之所以包装的目的是要给对NIO的channel加很多的功能，NioChannel持有socketChannel的一个引用，如果是SSL配置的话，那么就启用的是SecurityNioChannel类进行包装。

**b.PollerEvent数组**

PollerEvent是poller线程池处理的任务单元，这个类也是一个Runable线程。

PollerEvent不单单还有前面包装的NioChannel，还持有NioEndPoint.KeyAttachment类的一个引用，KeyAttachment类的作用主要是对Connector中的一些socket属性进行解析，然后设置到对应的SocketChannel通道中。

因为Tomcat作为前端的服务器，网络请求很多，所以对于一个Poller线程池，上述的从Acceptor过来的PollerEvent事件会非常的多，因此这里采用一个队列的模式做一下缓冲

**c.socket poller线程池**

Poller中维护者的是一个Selector对象，其实在Tomcat的前端中存在了n多个的Selector对象，当前这个Selector主要是用于从Acceptor传过来的NioChannel进行感兴趣事件的NIO注册操作，并轮询感兴趣的事件发生。

![1590112181998](https://smartan123.github.io/book/library/002-images/1590112181998.png)

这里还有一个Queue队列，这里采用的是SynchronizedQueue，需要注意的是，这里并不是JDK包中的SynchronizedQueue同步队列，而是tomcat中自定义实现的SynchronizedQueue，不要产生混淆，实现思路很简单，就是一个普通的数组演变的。

总结一下就是，poller线程主要是完成了NIO的selectkey的操作，这一步比较关键，之所以在前面又加了一个Acceptor线程，是因为每一次数据报进来后，都需要对其进行“加工一下”，再转发给poller进行selectkey感兴趣事件的获取。

到这里为止，poller线程仍旧没有进行处理，它继续将接力棒交接给工作线程池。

**d.工作线程池**

poller线程中最后一步时候processKey方法，这个方法最终会调用processSocket方法：

![1590112676397](https://smartan123.github.io/book/library/002-images/1590112676397.png)

SocketProcessor是工作线程池中的工作方法，上述工作线程池中一共有两个选择，当JDK5之前，SocketProcessor类本身也是一个Runable线程，直接可以执行run方法，这就没有什么线程池的概念了；而在JDK5之后，ThreadExecutor是JDK默认的线程池，Tomcat中集成了进来，也就是调用其executor.execute方法，将SocketProcessor任务传进去。

对于Tomcat的工作线程池的分解，在前面已经做过专题的讲解。

SocketProcessor任务中，一共分两个步骤，第一步是进行socket的handshake，也就是握手，对于正常的http来讲没有什么多余的操作，对于SSL可以看到在握手阶段，按照SSL的会话的交互，双方进行密码协商，这一步默认的话是调用JSSE的SSLEngine进行交互，返回SSLEngineResult，当握手成功后，该请求就可以交给给handler进行处理，这个handler就是下面要讲的Http11ConnectionHandler。

### 6、Http11ConnectionHandler（源码详细分析见视频）

Http11ConnectionHandler两个分析重点：

1)、Http11ConnectionHandler持有Http11NioProcessor类，Http11NioProcesso负责解析http协议。

2)、Http11NioProcesso解析完http协议，攒出request和response传递给CoyoteAdaptor，经过容器层层转发后抵达业务Servlet。

到这里为止，NIO的前端的逻辑就完成了。

### 7、总结

NIO的前端框架主要是由三个不同的线程依次分工协作：

1)、Acceptor线程将socketchannel取出， 传递给Poller线程（会产生线程阻塞，因此包装成PollEvent加入缓存队列）。

2)、Poller线程执行的就是NIO的selectkey，拿到通道中感兴趣的事件，轮询获取，然后将感兴趣的selectkey和keyattachment传递给工作线程池进行处理。
3)、工作线程池调用http11ConnectionHandler进行http协议的解析，然后将解析出来的内容包装成Request，Reponse对象，传递给分界点CoyoteAdapter，最终执行到业务中。



## 七、BIO连接器前端整体框图

### 1、BIO框图源码解读（tomcat8.5后抛弃）（源码详细分析解读见视频）

上一讲讲解过NIO的框图，可以看到，NIO通道是目前Tomcat7以后的默认的通道的推荐配置，在Tomcat6和以前的配置中，BIO是主流的配置。

只需要修改protocol协议部分即可，而后续还有APR协议，NIO2.0的协议，都是修改这个字段。

对于BIO的整体框图，基本和NIO保持类似，整体流程变化不大，如下图所示：

![1590081042566](https://smartan123.github.io/book/library/002-images/1590081042566.png)

### 2、Http11Protocol类详解

与NIO一样，这个Http11Protocol是默认的BIO的http1.1协议的处理类，Tomcat除了有NIO，BIO，其实还有两个通道：

1)、APR是高性能通道，

2)、NIO2是基于纯异步IO的通道，这个会在后面的Tomcat中进行讲解。

Http11Protocol类中，依然持有Endpoint和handler的引用，只不过，BIO对应的Endpoint是JIOEndpoint，对应的handler是Http11ConnectionHandler。

### 3、JIoEndPoint（tomcat7.x版本）

JIoEndpoint是BIO的端点类，它和NIO一样，也是维护着线程池，只不过因为没有Selector.select，没有SocketChannel的通道的注册，所以相比NIO模式，没有Poller线程是非常容易理解的，反倒是NIO的三个线程不容易理解，BIO可以看做就是基于Socket进行操作。

首先，初始化的JIoEndpoint的时候，会调用bind方法绑定Serversocket到对应的端口，bind方法是初始化构造JIoEndpoint的重要步骤，他的主要作用就是建立ServerSocketFactory。

根据SSL信道或者是普通的http的信道，Tomcat都实现了ServerSocketFactory，普通的http通道的ServerSocketFactory就是DefaultServerSocketFactory类，其工厂方法就是创建ServerSocket，很容易理解。

对于SSL通道的ServerSocketFactory是JSSEServerSocketFactory这个类创建的是SSLServerSocket。

其次，JIoEndpoint启动的时候，会将Acceptor线程和工作线程池启动起来。

除此之外，还启动了一个专门的线程，这个线程就是检查异步请求的Timeout的，后续会有专门的介绍针对于Tomcat的异步请求。

工作线程池，使用的就是JDK自带的ThreadPoolExecutor。

可以从线程的堆栈看到，对应的http-bio-8443-exec-n 这种线程都是工作线程池。

如果在tomcat中没有指定工作线程池的设置，那么都走的是JDK自带的ThreadPoolExecutor的默认值。

JIoEndpoint是BIO的端点类，它和NIO一样（NIO里面是NioEndpoint），也是维护着线程池，只不过因为没有Selector.select，所有只有2个线程池：

1）、acceptor

2）、worker（exec）

### 4、Acceptor线程

Acceptor线程的主要作用和NIO一样，如果没有网络IO数据，该线程会一直serversocket.accept阻塞住。

当有数据的时候，首先将socekt数据设置Connector配置的一些属性，然后将该接力棒传递给工作线程池。

最后一步processSocket方法，也是非常简单。



直接调用工作线程池，将SocketProcessor作为工作任务传入到工作线程池中执行。

这一步相比NIO的架构，缺少了NIO通道中的PollerEvent一个缓存队列，NIO中有这样的一个队列是因为需要从Acceptor到Poller线程，中间传递需要一个缓存的地方，而可以看到上述的BIO中的代码，如果工作线程池已经满载了，会根据JDK的ThreadPoolExecutor的策略是缓存，还是直接拒绝，或者是timeout等待，只不过BIO将这块的策略决断交给了ThreadPoolExecutor来做了。

对于Acceptor线程中还有一个重要的作用，就是控制连接的个数，这个在NIO通道的分析中没有讲解，这里看一下，Acceptor线程在while轮询的时候，每一次最开始都会检查一下当前的最大的连接数超出没有，如果超出了，就直接按照既定的序列调用LimitLatch进行锁定。

我们发现，实际上LimitLatch也是模仿JDK中的读写锁，内部持有一个Sync的类，这个类继承了JDK中隐藏功与名的AQS队列，这个AQS队列还是比较著名的，之前我的课中在分析JDK源码的时候，多次在n个并发类中都看到过他的踪迹，其实现几乎全部是CAS锁的实现。

### 5、SocketProcessor线程

SocketProcessor是工作任务，用于传入到工作线程池中，输入就是Acceptor传过来的socketWapper包装。

如果是SSL交互的话，Tomcat开放了握手的这个环节，但是并没有对应的实现，这个是因为SSL下的握手实现在SUN的包中做的，JDK提供的SSLServerSocket的接口已经隐藏了这个细节，我们可以从handshake这个第二步看到（这部分视频中有详细分析）

Tomcat中直接就可以拿到SSLSession，这个类可以获得相当于SSL已经是握手成功了，否则就会出现失败。

对于为什么保留beforeHandShake和handshake这两个步骤，是为了和NIO通道中的SSLEngine交互的接口做个兼容而已。

暂且不用管它，最重要的步骤就是第3步，也就是handler.process这一步，通过Http11ConenctionHandler进行处理http协议，并封装出Response和Request两个对象，传递给后端的容器。

SocketProcessor工作任务就是将Acceptor传过来的socketWapper包装传入到工作线程池中。

### 6、总结

BIO的流程基本上和NIO通道一样，BIO的结构因为缺少了selector和轮询，相比NIO少了一部分的内容，整体上就是使用的ServerSocket来进行通信的，一线程一请求的模式，代码看起来清晰易懂。但是，由于BIO的模型比较落后，在大多数的场景下，不如NIO，而现在Tomcat新版本也是NIO是默认的配置，8.5版本之后完全抛弃了BIO通道。



## 八、Tomcat的BIO和NIO通道及对性能的影响

### 1、BIO的缺点

前面两个章节，我们分别看了BIO和NIO两种Tomcat通道的实现方式。

BIO的方式，就是传统的一线程，一请求的模式，也就是说，当同时有1000个请求过来，如果Tomcat设置了最大Accept线程数为500，那么第一批的500个线程直接进入线程池中进行执行，而其余500个根据Accept的限制的数量在服务器端的操作系统的内核位置的socket缓冲区进行阻塞，一直到前面500个线程处理完了之后，Acceptor组件再逐步的放进来。

分析一下，这种模式的好处就是可以让一个请求在cpu轮转时间片切换中最大限度的执行，如果业务请求不是很长时间的事务处理，通常在一个时间片内肯定能做完当前的请求，这样的效率算是相当的高了，因为其减少了最耗时也是最头疼的线程上下文切换。

1.但是，如果事务执行比较长的时间，例如等待一个IO数据库的操作，那么这个工作线程就会根据cpu轮转不断的进行切换，因为请求数在大并发的时候有很多，所以不得不设置一个很高的Accept线程数，那么从cpu的耗费的资源上来看，甚至有70%的时间浪费在线程切换中，而没有真正的时间去做请求处理和业务，这是第一个问题。

2.其次，BIO每一次链接的建立和释放都需要重新来过一遍，例如一个socket进来之后，通常会对其SocketOptions的属性进行设置，包括各种Connector中配置都要与其进行一一对应，加上前面说的socket的建立，很多请求通道的资源的初始化都得重新创建，得不到复用，这个是第二个问题。

3.最后，BIO的方式，网络IO的阻塞等待是会让Accept线程工作效率降低很多的。

所以，基于这3个问题，特别是最后一个问题，引出了NIO的模型。

**总结一下就是：**

**1、如果IO处理时间长，那么bio大多数时间耗在线程切换中**

**2、IO通道得不到复用**

**3、Acceptor线程工作效率较低**



### 2、NIO的解决之道

NIO的架构分为三个线程池，这里再次梳理一下：

1）.Acceptor专门接socket请求，当发现又请求进来后，基于Tomcat配置的SocketOptions和一些属性的设置完毕，包装成SocketChannel，也就是NIO的socket通道抽象，塞入PollerEvent直接扔到队列当中。

2）.Poller线程从队列中挨个获取PollerEvent，调用Poller线程自己持有的selector选择器，注册SocketChannel到当前的selector选择器中，然后进行selectKey的工作，这样Acceptor传递过来的SocketChannel中感兴趣的事件，就会被轮询出来，当接收事件接收之后，需要注册OP_READ事件或者OP_WRITE事件，当OP_READ事件或者OP_WRITE事件发生时，开始调用工作线程池。

3）.工作线程池就是SocketProcessor，这个就是具体的工作线程，SocketProcessor的任务就是Poller线程从SocketChannel通道中轮询出来的数据包，进行解析，传递给后端的handler进行http的解析，解析出来的Request，Reponse对象，，直接调用CoyoteAdapter传递到后端的容器，通过Mapper，映射到对应的业务Servlet中。可以看到，从SocketProcessor一直到最终的业务Servlet实现，这些都是一个线程，这个线程就是工作线程。

对比Tomcat的BIO的架构，因为没有selector轮询的操作，所以并没有Poller线程，BIO中的Acceptor线程的作用依然是对socket简单的处理和属性包装，然后将socket直接扔到工作线程中来。NIO相当于是多了一个线程池，从流程上来讲，应该是多了一道手续，但是通过NIO本身基于事件触发的机制造成，Acceptor线程没必要设置的过多，这样从线程的数量上来看，大大的减少线程切换的频率，其次基于事件进行触发，将Acceptor线程执行效率中的网络IO延迟降低到最低，大大提升了Acceptor线程的执行效率。从这两点上来看，Tomcat的NIO在前面分析的BIO的三个问题中第一个问题，和第三个问题都有所改善，特别是第三个问题，全面进行了升级。

但是，对于BIO中的第一个问题，由后端事务时间过长导致工作线程池一直在运行，并且运行在一个高峰的数值，不断的进行切换，这种问题，NIO通道也没办法进行处理，这个是由业务来决定的，NIO只能保证降低的是Acceptor线程线程数，对业务帮助也是无能为力的，如果要提升这部分的效率，那就需要应用进行修改，优化JDBC和数据库，或者将业务切段来做，让事务时间尽量控制在一个可控的范畴之内。

对于第二个问题，无论是单纯的NIO和BIO通道都没有办法进行解决，但是HTTP协议中对链接的复用进行更新，在HTTP1.1中，这个keepalive是加到http请求头中的：

Keep-Alive: timeout=5, max=100 
timeout：过期时间5秒（对应httpd.conf里的参数是：KeepAliveTimeout）；

max是最多能承受一百次请求的共享复用，就是在timeout时间内又有新的连接过来，同时max会自动减1，直到为0，强制断掉。 

对应的Tomcat的服务器端的配置：

![1590080211887](https://smartan123.github.io/book/library/002-images/1590080211887.png)

keepAliveTimeout：表示在下次请求过来之前，tomcat保持该连接多久。这就是说假如客户端不断有请求过来，且为超过过期时间，则该连接将一直保持。

maxKeepAliveRequests：表示该连接最大支持的请求数。超过该请求数的连接也将被关闭（此时就会返回一个Connection: close头给客户端）。

如果配置了上述的内容，可以解决BIO上面提出的第二个问题，当一个页面中的第一个请求后，后面的连接可以复用这个socket或者是socketchannel，不用再accept三次握手或者SSL握手了，相当于高效的推动了整体Tomcat的时间链条的处理效率，而对于keepAlive属性的加入，通过BIO和NIO对比测试发现，相当于放大了NIO的优势，导致NIO的测试结果要明显高于BIO一个水平线上，这也就是目前http1.1协议中，为什么Tomcat后续版本默认就是NIO的原因；而如果没有keepAlive属性加入，在大多数的场景下，NIO并没有拉开与BIO太大的差距，甚至有一些场景上，Tomcat的BIO模式反倒是比NIO要高。

这里单纯的对比性能没有任何的意义，因为性能测试是测试在不同应用类型，不同硬件环境，不同软件版本，甚至是不同jdk性能差异都很大，客观因素很多。

**NIO优点总结一下就是：**

**增加了poller线程池做轮询**

**提高了acceptor执行效率**



### 3、NIO vs BIO（详细分析及演示见视频）

以下4个场景分别是：单线程BIO，多线程BIO，模拟NIO，NIO

4个场景最大的不同就是处理IO流部分，所以性能的高低直接取决于如何处理IO这一步。

因为演示步骤比较复杂，详细的分析请大家观看视频，这里不再赘述了。



![1590079380917](https://smartan123.github.io/book/library/002-images/1590079380917.png)

![1590079392963](https://smartan123.github.io/book/library/002-images/1590079392963.png)

![1590079402653](https://smartan123.github.io/book/library/002-images/1590079402653.png)

![1590079413774](https://smartan123.github.io/book/library/002-images/1590079413774.png)



### 4、总结

1）、BIO比NIO少了poller线程池的轮询机制，请求模式为一线程一请求的模式，这就导致了BIO中存在大量的线程上下文切换。

2）、NIO的多路复用的本质是用更少的线程处理多个IO流。



## 九、Tomcat中NIO2通道原理及性能

从Tomcat8开始出现了NIO2通道，这个通道利用了NIO2中的最重要的特性，异步IO的java API。

从性能角度上来说，从纸面上看该IO模型是非常优秀的，这也是很多书籍推崇的最优秀的IO模型，例如《Unix网络编程》这本圣经，但取决于目前操作系统的支持程度和环境，还有业务逻辑代码的编写，NIO2的程序调用并不一定比NIO，甚至比BIO的效率要高。

我们在没有实测的情况之下，本文从源码的角度去分析一下Tomcat8中的这个NIO2通道，后续在相应的章节中，我们会进一步的分析一下Tomcat的4个通道的性能差异。

### 1、NIO2的框图源码解读（源码详细分析解读见视频）

前面我们已经了解了Tomcat的BIO，NIO，APR这三个通道，对于NIO2的通道框图大体上和这些没有太大的区别，如下图所示，少了一个poller线程，多了一个CompletionHandler。

和其他通道一样，Tomcat最前端工作的依然是Endpoint类中的Acceptor线程，该线程主要任务是接收socket包，简单解析并封装socket，对其进行包装为SocketWrapper后，交给工作线程。

在NIO2的通道下，Acceptor线程结束之后，并不会直接调用工作线程也就是SocketProcessor，而是利用NIO2的机制，利用CompleteHandler完成处理器去异步处理任务。

![1590076539802](https://smartan123.github.io/book/library/002-images/1590076539802.png)

这正是CompleteHandler完成处理器的一个特性。

再对比NIO，BIO两个通道：

我们不用像BIO通道那样去拿着SockerWrapper在工作线程进行阻塞读，这样工作线程中的时间会占据网络IO读取的时间，导致大并发模式下工作线程暴涨，这也就是经常我们看到很多cpu为什么被占到99%的原因，再怎么设置工作线程无济于事，因为大量的cpu线程切换太耗时间了；

而NIO通道采用Reactor的模式去做这个事，Selector承担了多路分离器这个角色，对于BIO是一大改进，其次java NIO的牛B之处就是操作系统内核缓冲区的就绪通知；

### 2、异步IO的运用（具体源码分析见视频）

经过以上分析我们得知三件事：

1）.NIO2这种纯异步IO，必须要有操作系统支持，并且性能和这个内核态的事件分离器有着非常大的关系。

2）.对于内核分离器通知CompleteHandler的时机是什么，对比NIO的缓冲区，实质是当内核态缓冲区的数据已经复制到用户态缓冲区时候，这个时候触发CompleteHandler，这相当于比NIO的模式更进一步，如下图：

![1590076567707](https://smartan123.github.io/book/library/002-images/1590076567707.png)

NIO只是内核缓冲区就绪才告诉客户端去读，这个时候用户态缓冲区是空的，你得执行完socketChannel.read之后，用户态缓冲区才会填满；

3）.因为NIO2的优势，事件分离器分离器实际是在操作系统内核态的功能，所以不需要用户态搞一个Selector做事件分发。因此，对比NIO的通道框图，可以看到缺少了Poller线程这一个环节。

**以下是部分源码解析（详细解析见视频）**

从代码的角度来看看，Tomcat的NIO2的通道，主要集中在NIO2Endpoint这个类的bind方法。

关注两个点：

1）.AsynchronousChannelGroup是异步通道线程组，通过这个类可以给AsynchronousChannel定义线程池的环境，而ExecutorService就是Tomcat中的特有的线程池。

TaskQueue是队列，Thread工厂针对于创建的线程名称进行了一下修改，并且对于线程池的最大，最小，时间都进行了限定，这个线程池在BIO，NIO通道中也是这个，都是一样的。

定义完AsynchronousChannelGroup的通道线程组，AsynchronousChannel的read就是运行在通道组中的线程组中，包括从操作系统的内核态多路分离器响应的CompleteHandler，也是从该线程池中取出线程进行运行，这个是很重要的，如果每一次都new Thread的话，会有很大的消耗，所以不如都放在一个线程组中随取随用，用完再还；

2）.随即开启 AsynchronousChannel通道，并绑定到对应的端口中，这个API使用的就是JAVA NIO2的API。

之后，Acceptor线程获得socket包，直接进行包装为SocketWrapper，之后的流程如第一节中的源码分析一样，随着读取的执行，异步操作就执行完了，转而Acceptor线程进行下一个循环，读取新socket包；

这时候需要注意的是，在NIO模式下，这个时刻是将SocketWrapper扔给Poller线程，Poller线程中的Selector去轮询key值，而不是NIO2这种的直接就不管不问了，从这一点上也可以看出，NIO2的异步优势就在这，事件触发的机制直接由内核通知，我搞一个CompleteHandler就行，无需在用户态轮询。

### 3、总结

由下图可见，bio，nio都是由用户态发起数据拷贝（read操作），而nio2（aio）则是由操作系统发起数据拷贝，所有的io操作都是由操作系统主动完成。所以io操作和用户业务逻辑的执行都是异步化的。

![1590076600430](https://smartan123.github.io/book/library/002-images/1590076600430.png)



所以从账面上来讲，NIO2通道相比NIO效率高，因为proactor模式本来就比reactor模式要好，另外还省去了Poller线程，但由于多路事件分离器是内核提供的，不同内核提供的多路事件分离器的事件处理效率不一，对NIO2的通道需要基于实际环境和场景压测才能得出最终的结论。

在后续的章节中，会对Tomcat各通道进行压力实际测试对比，并基于各个通道的实测结果进行详细的对比和分析。



## 十、APR通道到底是个怎么回事？

APR通道是Tomcat比较有特色的通道，在早期的JDK的NIO框架不成熟的时候，因为java的网络包的低效，Tomcat使用APR开源项目做网络IO，这样有效的缓解了java语言的不足，提供了一个高性能的直接通过jni接口进行底层IO通信内存使用的这么一个通道。

但是，当JDK的后续版本推出之后，JDK的网络底层库的性能也上来了，各种先进的IO模型，线程模型和APR开源项目几乎不相上下，这个时候，经常会出现一种测试场景是，加上APR通道之后并没有太多的实质提升，这是可以理解的，但是JDK中的SSL信道的性能至少从目前的角度来看，和APR通道基于openssl的引擎信道实现，还有不小的差距，因为SSL协议中定义的握手协议，交互次数比较多，而openssl项目经历多年，性能极为高效，因此从目前的Tomcat的APR通道来看，主推的就是这个SSL/TLS协议的高效支持。

### 1、TomcatAPR通道的架构图

**APR通道底层最终是通过tomcat-native实现的，具体的源码分析讲解请观看视频**

![1590074831386](https://smartan123.github.io/book/library/002-images/1590074831386.png)

### 2、APR通道详解（见源码分析视频）

从上图中可以看到，对于Connector通道总共有这么几种通道：BIO是阻塞式的通道，NIO是利用高性能的linux（windows也有）的poll或者epoll模型，APR通道就是本文中讲的内容，对于目前的JDK还支持NIO2的通道，对于APR来讲，SSL Support区别最大，使用的是openssl作为SSL的信道支持，另外从IO模型角度来看，对于Http请求头的读取，SSL握手因为调用的JNI也是阻塞的，这个是与NIO和NIO2的差距，但是从SSL信道的支持上用的是高效的openssl。APR通道中依然有Acceptor接收线程池，Poller轮询，Worker工作线程池，这些和其它通道的架构区别不大，重要的是其关于socket调用和SSL的握手等内容。**这部分的源码分析见视频**

**总之一句话**

APR通道的Socket全部来自c语言实现的socket，非jdk的socket，直接在tomcat层级调用native方法。

APR通道的SSL信道上下文直接来自于native底层

### 3、Tomcat-Native子项目

tomcat中对于这些jni的调用部分，做出了一个tomcat的子项目，叫做Tomcat-native，在这个调用层级中，一部分是java部分，也就是AprEndpoint类中看到的native方法，这些native方法有很多，这些java的包，对应调用的就是jni的native的C的代码，是一一对应的，如下图所示：

![1590074929764](https://smartan123.github.io/book/library/002-images/1590074929764.png)

对于tomcat-native最好的教程应该是在example目录中，这个目录使用一个例子完整的复现了Tomcat前端APREndpoint的几个线程组件的工作模式；对于test目录也可以从这个点切入进去，是一个好的调试tomcat-native代码的过程。

### 4、APR高性能网络库（Apache Portable Runtime (APR) project）

下载：https://mirrors.cnnic.cn/apache/apr/apr-1.6.5.tar.gz

tomcat-native项目，可以说是作为一个集成包，有点类似于TomEE对于JAVA EE规范的集成，它集成的内容一个是openssl，这个是ssl信道的实现，另外一个就是高性能的apr网络库。

Apache Portable Runtime (APR) project，这个库定位于在操作系统的底层封装出一层抽象的高性能库，在于屏蔽掉操作系统的差异。可以分析出来，APR相当于JDK的一个角色了，只不过它关注的大多在网络IO相关的这块，有原子类，编解码，文件IO，锁，内存申请与释放，内存映射，网络IO，IO多路复用，线程池等等。APR库对众多操作系统都有支持。

总结一下就是，APR提供了对于底层高性能的网络IO的处理，可以解决Tomcat早期网络IO低效的问题。

### 5、Openssl库

tomcat-native除了调用APR网络库保证高性能的网络传输以外，对于SSL/TLS的支持还调用了openssl。对于OpenSSL项目来说，市面上大多数的SSL信道实现都是用OpenSSL做的，这也就是说，如果要OpenSSL暴露出一个漏洞出来，那破坏性都是惊人的。

![1590075035030](https://smartan123.github.io/book/library/002-images/1590075035030.png)

### 6、总结

APR通道只有很小的一部分是java，大部分的源码都是C的，而且和操作系统的环境有着密切的关系，不同操作系统定制的接口不同，性能特色也不同。

如下图所示，java这一层调用的是jni，相当于是一个接口，然后底层tomcat-native，相当于是实现，只不过是用c实现的，然后apr和openssl又是独立的c组件。

![1590075088393](https://smartan123.github.io/book/library/002-images/1590075088393.png)



## 十一、Tomcat中各通道的sendfile支持

sendfile实质是linux系统中一项优化技术，用以发送文件和网络通信时，减少用户态空间与磁盘倒换数据，而直接在内核级做数据拷贝，这项技术是linux2.4之后就有的，现在已经很普遍的用在了C的网络端服务器上了，而对于java而言，因为java是高级语言中的高级语言，至少在C语言的层面上可以提供sendfile级别的接口，举个例子，java中可以通过jni的方式调用c的库，而这种在tomcat中其实就是APR通道，通过tomcat-native去调用类似于APR库，这种调用思路虽然增大了java调用链条，但可以在java层级中获得如sendfile的这种linux系统级优化的支持，可谓是一举多得。

上述的内容，实际就是本章的背景，本文就从系统调用的层级，逐步讲解tomcat中的sendfile是怎么实现的。

### 1、传统的网络传输机制

大家可以在linux上执行 man sendfile 这个命令，查看sendfile的定义

![1590073153559](https://smartan123.github.io/book/library/002-images/1590073153559.png)

上述定义可以看出，sendfile()实际是作用于数据拷贝在两个文件描述符之间的操作函数.这个拷贝操作是在内核中完成的,所以称为"零拷贝".sendfile函数比起read和write函数高效得多,因为read和write是要把数据拷贝到用户应用层操作，多了一个步骤，如下图所示：

![1590072613168](https://smartan123.github.io/book/library/002-images/1590072613168.png)



那么经过sendfile优化过的拷贝机制如下图所示，直接在内核态拷贝，不用经过用户态了，这大大提高了执行效率。

### 2、linux的sendfile机制（零拷贝）

![1590072646394](https://smartan123.github.io/book/library/002-images/1590072646394.png)



### 3、DefaultServlet的sendfile逻辑（具体源码跟踪分析见视频）

对于Tomcat中的静态资源处理，直接对应的就是DefaultServlet了，这个类是嵌入在Tomcat源码中，专门处理静态资源的类，静态资源一般不需要经过处理（也就是不需要拿到用户态内存中去）直接从服务器返回，所以此类文件最适合走sendfile方式，以下是DefaultServlet中和sendfile相关的源码逻辑。

**这部分源码详细分析请查看视频**

![1590072679844](https://smartan123.github.io/book/library/002-images/1590072679844.png)

![1590072689154](https://smartan123.github.io/book/library/002-images/1590072689154.png)

值得注意的一点是，一般http响应的数据包都会进行压缩，这样的好处是能极大的减小带宽占用，而响应头中发现了compression压缩属性，浏览器会自动首先进行解压缩，从而正确的将response响应主体刷到页面中。

但是，当sendfile属性开启后，这个compression压缩属性就不生效了（后面一章会讲解sendfile和compression的互斥性），因此，当需要传输的文件非常大的时候，而网络带宽又是瓶颈的时候，sendfile显然并不是合适之举。

### 4、sendfile在BIO通道中的实现（不支持）（具体源码跟踪分析见视频）

以Tomcat9为例，不同的Tomcat前端通道中的sendfile的java包装是不同的，但实际上都是在调用系统调用sendfile。

对于BIO（**从tomcat8开始已经抛弃BIO通道了，下面源码截图来自于tomcat7**）来说，JIOEndpoint是不支持sendfile的，这个可以通过代码中看出来：

![1590072736960](https://smartan123.github.io/book/library/002-images/1590072736960.png)



### 5、sendfile在NIO通道中的实现（具体源码跟踪分析见视频）

在NIO通道中，有一个useSendfile属性，这个useSendfile属性是做什么的呢？

这个是可以设置在Connector中的，以NIO通道为例，这个useSendfile属性是允许request进行sendfile的总体开关（前面讲的org.apache.tomcat.sendfile.support 属性是针对于每一个request的），这个useSendfile属性在NIO通道中默认就是打开的，当reqeust设置org.apache.tomcat.sendfile.support 属性为true的时候，response就会准备一个SendFileData的数据结构，这个数据结构就是NIO通道下的sendfile的媒介。

因此，NIO的sendfile实现可以分为三个阶段（**具体的源码解析请查看视频**）：

第一阶段，实际上就是前面的XXXDefaultServlet中（不仅仅是DefaultServlet，其它的Servlet只要设置这个属性也可以调用sendfile）对Request的sendfile属性的设置，当该请求设置上述的属性后，证明该请求为sendfile请求。

第二阶段，servlet处理完之后，业务逻辑完成，对应的Response该commit了，而在Response的准备阶段，会初始化这个SendFileData的数据结构，这块的代码逻辑都在Http11NioProcessor类中，下图中的prepareSendfile方法就是从前面DefaultServlet中设置的reqeust属性中拿到file名称，字符位置的start，end，然后将这些属性作为传入的参数，初始化SendFileData实例。

![1590072760621](https://smartan123.github.io/book/library/002-images/1590072760621.png)



第三阶段，我们记得NIO前端通道的Acceptor，Poller线程，Worker线程的三个线程，当Worker线程干完活之后，返回给客户端，依然要通过Poller线程，也就是会重新注册KeyEvent，读取KeyAttachment，这个时候当为sendfile的时候，前面初始化的SendFileData实例是会注册在KeyAttachment上的，上图的processSendfile就是Poller线程的run中的一个判断分支，当为sendfile的时候，Poller线程就对SendFileData数据结构中的file名字取出，通过FileChannel的transferTo方法，这个transferTo方法本质上就是sendfile在tomcat源码中的具体体现，如下图所示

![1590072769371](https://smartan123.github.io/book/library/002-images/1590072769371.png)



### 6、sendfile在APR通道中的实现（具体源码跟踪分析见视频）

在NIO通道中sendfile实现算是比较复杂的了，在APR通道中更加的复杂，我们可以回过头先看看NIO通道中的sendfile，实际是通过每一个Poller线程中的FileChannel的transferTo方法来实现的，对于transferTo方法是阻塞的，这也就意味着，当文件进行sendfile的时候，Poller线程是阻塞的，而我们前面研究过Tomcat前端，Poller线程是很珍贵的，不仅仅是为某几个sendfile服务的，这样会导致Poller线程产生瓶颈，从而拖慢了整个Tomcat前端的效率。

APR通道是开辟一个独立的线程来处理sendfile的，如下图所示，这样做的好处不言自明，Poller就干Poller的事，而遇到Sendfile的需求的时候，sendfile线程就挺身而出，把活儿给接了。

最后，对于APR通道是通过JNI调用的APR库，sendfile自然就不是java的API了

![1590072795214](https://smartan123.github.io/book/library/002-images/1590072795214.png)

![1590072806036](https://smartan123.github.io/book/library/002-images/1590072806036.png)

![1590072813118](https://smartan123.github.io/book/library/002-images/1590072813118.png)

### 7、总结

SendFile实际上是操作系统的优化，Tomcat中基于在不同的通道中有不同的实现，配置也不尽相同，但实际上都是底层操作系统的SendFile的系统调用！



## 十二、Tomcat中的compression压缩属性优化

### 1、http响应头中压缩相关属性

这里着重讲解3个属性。

**传输内容编码：Content-Encoding**

内容编码，即整个数据信息是在服务器端经过怎样的编码处理，然后客户端会以怎么的编码来反向处理，以得到原始的内容，这里的内容编码主要是指压缩编码，即服务器端压缩，客户端解压缩。
可以参考的值为：gzip,compress,deflate和identity。

通常压缩方式都是gzip格式的，当选择gzip的时候，整个html文本会被进行一次gzip格式的压缩。java版本的实现有GZIPOutputstream可以进行gzip的实现了，并且对于servlet可以通过查看httprequest来查看这个属性是否支持gzip，如果支持的话，那么浏览器端也会进行gzip相应的解压。

**传输数据编码：Transfer-Encoding**

数据编码，即表示数据在网络传输当中，使用怎么样的保证方式来保证数据是安全成功地传输处理。可以可以是分段传输，也可以是不分段，直接使用原数据进行传输。
有效的值为：chunked和Identity.

**传输内容格式：Content-Type**

内容格式，即接收的数据最终是以何种的形式显示在浏览器中。

可以是一个图片，还是一段文本，或者是一段html，内容格式额外支持可选参数,charset，即实际内容的字符集。通过字符集，客户端可以对数据进行解编码，以最终显示可以看得懂的文字（而不是一段byte[]或者是乱码)。

***Content-Type是代表着格式，这个一般不会混淆，***

***而Content-encoding这个是内容编码格式，实际上就是压不压缩传输，***

***Trandfer-encoding这个是传输的方式，大白话也就是分不分块，***

***上述的三个属性就是http响应头中的格式，我们主要关注的是Content-encoding，当然我们在解析Tomcat的代码时，还会看到其余的两个属性的踪影。***

### 2、Tomcat源码中的压缩实现

**以下是总体概述，具体的源码分析请观看视频**

对于压缩的处理，是在Tomcat中的响应头中，也就是Response的commit的时候，开始对输出流进行处理，而如果Content-encoding是gzip的话，那么会在Http11Processor中的输出流filter链条中，加上一个GzipOutputFilter。

Http11Processor是Tomcat前端比较重要的处理类，Work工作线程将任务交给Http11Processor开始继续干活，Http11Processor接着会攒出Request和Response，并基于Mapper进行调用，从而进入容器中。

而XXXFilter这里的filter不是容器端的filter，而是在Response进行commit提交的时候，基于响应头的Tomcat的配置，是否执行相关的处理。

以这个compression为例，当在Tomcat中配置了compression的话，GzipOutputFilter就开始自动执行过滤，从上面的代码逻辑可以看到，实际上就是基于流的包装机制，使用GzipOutPutStream来再对当前的流进行一次包装，然后在OutputBuffer最终commit的时候，调用这个GzipOutputFilter，最终执行doWrite方法，让输出流中的字节进行压缩。

从上述的分析可以看出，Tomcat的压缩实现实际上就是GzipOutputStream，只不过采用了GzipOutputFilter责任链的模式，通过流的一层一层的包装，将输出的字节进行了压缩。

**具体的源码分析请观看视频**

### 3、compression压缩属性设置

设置非常简单，但是要注意一点，usesendfile和compression属性必须同时设置，且互斥，如下图所示：

![1590072212126](https://smartan123.github.io/book/library/002-images/1590072212126.png)

### 4、与sendfile的互斥性

我们了解的sendfile，实际是一种操作系统级别的优化手段，直接跳过内存转接，直接从内核缓冲区到网卡缓冲区，相当高效；

但是我们在查询Tomcat文档的时候，发现sendfile和compression是不兼容的，也就是上图中的红色字体部分，这个是为什么呢？

**可以这么来理解，对于compression必然需要在用户空间内存转接中（压缩必须拿到用户态内存中来压）进行操作，也就是下图中用户空间部分，但是sendfile又要求不经过用户空间，所以两者是矛盾的。**

![1590072239137](https://smartan123.github.io/book/library/002-images/1590072239137.png)

### 5、总结

1）、经源码启动分析测试，当配置Compression为gzip时，在Tomcat中是采用GzipOutputStream来实现压缩优化，压缩比约为7:1，压缩比很大，节约了带宽。

2）、当配置Compression为gzip时，在Tomcat中是采用GzipOutputStream来实现的，而更要记住的是，Sendfile和Compression这两个优化选项只能选择其一来使用！



## 十三、Tomcat优化之deferAccept参数

### 1、TCP中的TCP_DEFER_ACCEPT优化参数

在Tomcat中，有很多的web服务器的参数可以配置，很多是Tomcat基于自身逻辑的，如线程池大小调整等等。

但是，也有很多是操作系统级别的参数在Tomcat中的映射。本文中的讲述就是一个TCP协议栈内核级别的deferAccept参数。

**我们先来看看一般的TCP三次握手和传输阶段：**

![1590070632651](https://smartan123.github.io/book/library/002-images/1590070632651.png)

首先，客户端发出一个SYN包，这个包的作用是与服务器端开始尝试进行链接；

然后，服务器端如果存在，基于这个SYN包，回复一个SYN+ACK的包，告知客户端我存在，连吧；

最后，客户端最后回复一个ACK，告知服务器端，客户端已经准备发送数据了，服务器端你准备好吧；

整体的TCP握手的链接阶段就宣告成功，下一阶段开始进入数据传输的第二阶段了；



**上述的流程没什么可说的，只不过我们关注于右侧上图中红色标记的部分。**

当客户端回复的ACK之后，服务器端知道客户端要开始发包了，这样服务器端通过内核的协调，需要唤醒一个数据接收进程，这个Acceptor进程会绑定一个IO句柄用于进行接收，这个句柄按照系统调用来进行理解，也就是网络传输的文件描述符fd。

而我们看看，在服务器端的ESTABLISED建立成功之后，到数据传输可能还有一段距离，假设客户端的程序阻塞，加上网络延时，这个时间就非常的大；



**而当前是什么状态？**

这个状态是服务器端已经消耗了一个进程去等待资源，已经搞了一个fd，甚至操作系统内核级也要时刻准备着，去维护这些状态变化，可以看到，服务器端空消耗这些，而客户端还迟迟不来请求。



**有什么办法优化这个呢？**

可以设想一种机制，服务器端对客户端的最后一个ACK进行视而不见，直接丢弃，这样的话，服务器端就不会启动Acceptor进程，也不会有fd，也不会有上述的消耗，而当客户端真正把数据发送过来了，这个时候服务器端才开始开启Acceptor进程，开始上述的操作。

而这个优化，其实就是TCP_DEFER_ACCEPT属性。



### 2、Tomcat中的deferAccept属性配置与实现

启动本机tomcat后， [       ](http://127.0.0.1:8080/docs/config/http.html)[查看参数](http://127.0.0.1:8080/docs/config/http.html)http://127.0.0.1:8080/docs/config/http.html

我们可以看到，TCP_DEFER_ACCEPT其实是一个操作系统内核级，TCP/IP协议栈的优化参数，只能在系统调用中进行设置，而java语言在包装socket api的时候，并没有开放这块内容，严格意义上来讲，至少目前JVM中没有实现，因此从这个意义上来讲，Tomcat中的NIO，BIO，甚至NIO2通道中都不会有这个参数的优化。

但是，在APR通道中，因为Tomcat前端代码是通过JNI调用的tomcat-native，tomcat-native调用的APR库作为Socket封装，而APR库的socket封装就来源于系统调用的socket，因此这个参数应该是能开放出来。

### 3、总结

Tomcat中的deferAccept属性实际上是操作系统级别的TCP_DEFER_ACCEPT参数的优化，只在APR通道中有实现。



## 十四、Tomcat对keep-alive的实现逻辑及优化

### 1、什么是keepalive？

http协议的早期是，每开启一个http链接，是要进行一次socket，也就是新启动一个TCP链接。

使用keep-alive可以改善这种状态，即在一次TCP连接中可以持续发送多份数据而不会断开连接。通过使用keep-alive机制，可以减少tcp连接建立次数。

举一个例子，用户浏览一个网页时，除了网页本身外，还引用了多个 javascript 文件，多个 css 文件，多个图片文件，并且这些文件都在同一个 HTTP 服务器上，算作一个http请求，而如果浏览器支持keepalive的话，那么请求头中会有如下connection属性，如下图所示：

![1590068724337](https://smartan123.github.io/book/library/002-images/1590068724337.png)

对于keepalive的部分，主要集中在Connection属性当中，这个属性可以设置两个值：

close（告诉WEB服务器或者代理服务器，在完成本次请求的响应后，断开连接，不要等待本次连接的后续请求了）。

keepalive（告诉WEB服务器或者代理服务器，在完成本次请求的响应后，保持连接，等待本次连接的后续请求）。

从整体可以再看看keepalive的优化的结果如下：

![1590068312542](https://smartan123.github.io/book/library/002-images/1590068312542.png)



从上面的分析来看，keepalive这个选项相当好，是否所有的场景都适合开启keepalive呢？

情况1：如果用户浏览一个网页时，除了网页本身外，顶多能引入1,2个 javascript 文件，1,2个图片文件。
情况2：如果用户浏览的是一个动态网页，由程序即时生成内容，并且不引用其他内容。

当情况1的时候，keepalive的作用就不那么明显了，而情况2来说，keepalive开启与不开启没有任何的关系，因为整个网页是动态形成的，在服务器端对html页面进行组装的，因此开不开启都是一个TCP链接。

另外，需要澄清两个事情：

第一个，keep-alive与TIME_WAIT的关系，使用http keep-alive，可以减少服务端TIME_WAIT数量(因为由服务端httpd守护进程主动关闭连接)。道理很简单，相较而言，启用keep-alive，建立的tcp连接更少了，自然要被关闭的tcp连接也相应更少了。

什么是TIME_WAIT呢？

通信双方建立TCP连接后，主动关闭连接的一方就会进入TIME_WAIT状态。

客户端主动关闭连接时，会发送最后一个ack后，然后会进入TIME_WAIT状态，再停留2个MSL时间，进入CLOSED状态，原理如下图所示：

![1590068924782](https://smartan123.github.io/book/library/002-images/1590068924782.png)

### 2、keepalive的配置实现（两个参数）

在不同的web服务器中，肯定都有keepalive的配置，一般配置如下两个参数：

keepAliveTimeout:此时间过后连接就close了，单位是milliseconds

maxKeepAliveRequests:最大长连接个数（1表示禁用，-1表示不限制个数，默认100个，一般设置在100~200之间）

在tomcat中，http11之后，keepalive默认就是开启的。

### 3、Tomcat中Keepalive的实现原理

**以下是总体的步骤，具体详细的源码分析请观看视频。**

**步骤1：准备阶段**

首先准备SocketWrapper，SocketWrapper实际就是socket的包装类，而通过这个包装类加上一些属性，例如keepaliveout时间，keepaliveRequest的次数；其次，keepalive默认就是true，如果当前发现SocketWrapper包装类是不支持keepalive的，这种情况直接keepalive就是false，后续任凭你咋配置tomcat的keepalive的属性，keepalive也不能工作。

**步骤2：启动大循环，识别该请求没有结束（是否keepalive模式开启后，连续的几个请求）跳出循环，释放或者出让工作线程**

首先开启一个大循环，然后判断请求是否是该keepalive期间的最后的一个请求，如果是的话，那么在这里直接就进行break掉，释放掉该工作线程，因为活都已经干完了嘛，如果发现不是最后一个请求，或者后续还有可能有请求，那么这里务必需要将keepalive的模式的状态还要保持住，这些属性如openSocket和readComplete等状态，来保证下一次请求这些状态能正常工作。

通过这段代码就可以分析，在keepalive期间，工作线程池是可以进行释放或者出让的，至少从程序的逻辑上来看，保留了入口。

**步骤3：通过prepareRequest方法解析请求头，基于客户端状态设置keepalive**

这一步其实比较清晰，就是解析http请求头，看看是否支持keepalive；

先看看http协议，再看看请求头中的Connection字段，如果不是keepalive的话，是close的话，那么就需要强制关闭了，最后看看客户端浏览器的agent是否支持，如果上面都可以的话，keepalive就可以设置了，如果一点不行，那么这里面直接就不能执行keepalive的逻辑，如果是Connection:close的话，处理完直接链接关闭。

从这一步上来看，keepalive也不是那么容易就开启的；

**步骤4：设置Tomcat的keepalive**

到这一步了，说明至少环境上是可以满足keepalive了，但是前面讲过Tomcat的配置可以让keepalive停掉；

例如maxKeepAliveRequests如果设置成1了，这里直接keepalive就为false，相当于给禁止了，如果maxKeepAliveRequests大于0，走到这里执行了一次，需要减1，这就用到了前面准备阶段中的SocketWrapper的计数器。

**步骤5：执行Tomcat容器部分，如果出现异常，关掉Keepalive**

这一步就是执行容器，然后基于反馈，如果错误，直接置响应头为Connection:close，keepalive直接就没用了，链接都关了。

**步骤6：设置request的keepalive阶段，看是否各变量符合跳出大循环**

到这里，大循环任务已经完成，最后检验一下，如果出现错误，这里就会通过breakKeepAliveLoop跳出大循环；

如果一切正常，当前的Request的阶段就是STAGE_KEEPALIVE阶段；

### 4、总结

本文关注keepalive的原理，Tomcat中的配置与Tomcat中对keepalive的基本实现，大家还可以从线程池的视角，看看通过不同通道在keepalive下，究竟有哪些异同，从而分析出keepalive参数对性能为什么这么关键的原因。



## 十五、调整和tomcat相关的JVM参数进行优化

### 1、设置串行垃圾回收器（nio模式，最大线程1000）

压测步骤：

1）、在tomcat启动脚本catalina.sh里设置以下脚本：

年轻代、老年代均使用串行收集器，初始堆内存64M，最大堆内存512M，打印gc时间戳等信息，生成gc日志文件

JAVA_OPTS="-XX:+UseSerialGC -Xms64m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:
+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:../logs/gc.log"

2）、设置后启动tomcat，使用jmeter进行压测（jmeter设置线程为1000，每个线程循环10次），访问test_web

![1590064622218](https://smartan123.github.io/book/library/002-images/1590064622218.png)

3）、查看吞吐量

**压测结果：平均时间1.585s，吞吐量378.6/s，异常1.12%**

![1590064794634](https://smartan123.github.io/book/library/002-images/1590064794634.png)

将gc.log拷贝出来，改名gc1.log。预备比较

### 2、设置并行垃圾回收器（nio模式，最大线程1000）

压测步骤：

1）、在tomcat启动脚本catalina.sh里设置以下脚本：

年轻代、老年代均改成并行垃圾收集器，初始堆内存64M，最大堆内存512M，打印gc时间戳等信息，生成gc日志文件。

#JAVA_OPTS="-XX:+UseParallelGC -XX:+UseParallelOldGC -Xms64m -Xmx512m -XX:+PrintGCDetails -XX
:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:../logs/gc.log"

2）、删除gc.log 

 rm -rf gc.log

3）、设置后重启tomcat，使用jmeter进行压测（jmeter设置线程为1000，每个线程循环10次），访问test_web,查看吞吐量

**压测结果：平均时间1.161s，吞吐量407.7/s，异常0.40%**

![1590065401961](https://smartan123.github.io/book/library/002-images/1590065401961.png)

将gc.log拷贝出来，改名gc2.log。预备比较

**分析结论：**

可以看出设置成并行垃圾收集器之后平均执行时间减少了，吞吐量增加了，异常率也减少了，总体性能有了很大的提高。

### 3、查看gc日志文件

将gc1.log和gc2.log文件分别上传到gceasy.io进行在线分析，分析结果如下：



**gc1.log中的gc总次数是13次**

![1590066198018](https://smartan123.github.io/book/library/002-images/1590066198018.png)



**gc2.log中gc总次数12次，比串行时少了1次，性能是有所提升的。**

![1590066087727](https://smartan123.github.io/book/library/002-images/1590066087727.png)

### 4、调整年轻代大小

再次重新设置启动参数，依然是并行垃圾收集器，不过我们增加了初始化堆内存和最大堆内存，分别设置为128m和1024m。

JAVA_OPTS="-XX:+UseParallelGC -XX:+UseParallelOldGC -Xms128m -Xmx1024m -XX:NewSize=64m -XX:M
axNewSize=256m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHe
apAtGC -Xloggc:../logs/gc.log"

设置完后再次重启，用jmeter进行压测（压测参数不变），结果如下：

![1590066490885](https://smartan123.github.io/book/library/002-images/1590066490885.png)



**压测结果：平均时间0.943s，吞吐量433.5/s，异常0.29%**

性能再一次的得到了提升。再次分析gc.log 如下图：

![1590066772082](https://smartan123.github.io/book/library/002-images/1590066772082.png)

**gc收集总次数减少为8次，从gc的收集次数也再次证明了调整参数后性能的确得到了极大的提升。**

### 5、设置G1垃圾回收器(jdk9之后默认G1，测试用的jdk8)

再次重新设置启动参数，修改垃圾收集器为G1收集器，参数如下：

JAVA_OPTS="-XX:+UseG1GC -Xms128m -Xmx1024m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+Pr
intGCDateStamps -XX:+PrintHeapAtGC -Xloggc:../logs/gc.log"

重启tomcat后使用jmeter再次压测（压测参数不变），压测结果如图：

![1590067084119](https://smartan123.github.io/book/library/002-images/1590067084119.png)

**压测结果：平均时间0.897s，吞吐量431.2/s，异常0.14%**

总体性能再一次得到了提升。

### 6、总结

通过不断的调优，我们得出4次压测结果如下：

**第1次压测结果：平均时间1.585s，吞吐量378.6/s，异常1.12%**

**第2次压测结果：平均时间1.161s，吞吐量407.7/s，异常0.40%**

**第3次压测结果：平均时间0.943s，吞吐量433.5/s，异常0.29%**

**第4次压测结果：平均时间0.897s，吞吐量431.2/s，异常0.14%**

平均时间一次比一次短，吞吐量一次比一次大，异常率一次比一次少，所以总体性能一次比一次优越。

**结论**：对tomcat性能优化需要不断的进行参数调整，然后测试结果，可能每次调优结果都有差异，这就需要借助于gc的可视化工具来看gc的情况，再帮我我们做出决策应该调整哪些参数，从而达到一个相对理想的优化效果。