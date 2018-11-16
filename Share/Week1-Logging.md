### 为什么需要日志
对于每一个开发人员而言，开发过程中总会多多少少需要打印日志，用于调试，特别是逻辑复杂时，需要追踪程序运行中发生了什么，日志就显得很有必要。

在正式说说日志之前，我们假想现在有个刚刚开始学习开发的同学小王。小王正在开发一个Java 应用，并希望服务在启动时告诉他执行到了哪个阶段，他写了类似这样的输出文本，打印到了控制台：

```java
public class Application{
     public static void main(String []args){
        System.out.println("Prepare to run service...");
        // do something
        System.out.println("Service starts up successfully!");
        // or
        System.out.println("Service starts up failed!");
     }
}
```

本地调试完毕后，小王想把代码打包放在服务器上运行，可服务器上没有IDE，命令行运行结束后，再登陆服务器也看不到上次输出的情况。于是小王想把这些输出信息打印到文本文件里，这样就可以看到历史运行信息了。但除了看到这些信息意外，小王还想知道是什么时候打印的，程序代码丰富之后，还需要知道是来自哪个class。出了问题需要查找问题所在，在茫茫日志中，小王想马上知道错误信息是什么，而那些无关日志却比和问题相关日志多的多。服务运行一段时间后，日志文件的大小日益剧增，小王想让服务定时清除几个月以前应该不再有利用价值的的日志文件。服务开始暴露web服务或restful API时，小王想知道他的用户调用服务是否成功，100个调用里面有多少成功，多少失败，这些请求又分别来自哪里。

到这里大家有没有觉得这是否很可能是日志诞生之初希望解决的问题。接下来我们一一来看看小王怎么利用日志，解决他想知道的这些问题。

### 不需要重复造轮子 - Logging Framework
小王只有一个，但是像小王一样的程序员成千上万，如果每个人都根据这些通用的需求写一套日志处理模块/框架，显然是不明智的。这时候就需要引入通用的日志框架(Loggin Framework)了，以Java为例，有定义了日志通用接口的`SLF4J`和基于其实现的`Log4J`, `Logback`和`JUL`。这些框架各有特点，但框架的设计大同小异，其他语言也是类似，我们以Apache开源项目`Log4J`为例，看看一个常用的日志框架是怎么设计的。

![image](https://www.tutorialspoint.com/log4j/images/log4j-arch.jpg)
> 引用自：https://www.tutorialspoint.com/log4j/log4j_architecture.htm

在`Log4J`框架中，`Logger Object`, `Appender Object`和`Layout Object`是三个核心组件，共同决定了日志是如何写到日志文件或控制台中的。

###### Logger Object
`Logger Object` 提供最上层用户可调用输出日志的接口，在上个小王的代码中，若使用`Log4J`输出日志，可改写为以下代码：
```
import org.apache.log4j.Logger;
public class Application{
    static Logger log = Logger.getLogger(log4jExample.class.getName());
     public static void main(String []args){
        
        log.info("Prepare to run service...");
        // do something
        log.info("Service starts up successfully!");
        // or
        log.info("Service starts up failed!");
     }
}
```

小王只需要在Class中定义一个`Logger Object`即可用该框架输出自己的日志信息。那么输出的日志会以什么格式写在哪个日志文件中呢，这就是另外两个Object的工作了。
###### Layout Object
`Layout`顾名思义，指的是输出的日志将以某种形式展示出来，通常来说，日志框架允许用户以配置文件的形式规定日志输出的格式，我们以一个简单的`Log4J`配置为例来说明：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

这里`PatternLayout`是日志框架中常见的一种，以通配符的方式规定每一条输出日志的类型，即每一条文本都是固定的格式，只是每个字段的值根据每次输出的设定而改变。这个例子中，规定了输出格式为`%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n`，意思每一行输出的顺序为：毫秒级的时间戳，thread信息，日志等级(下文会详细说明)，日志需要打印的消息内容。

按照这个设定，上例中，小王程序的日志打印出来应如下：
``` text
17:13:01.540 [main] INFO Application - Prepare to run service...
17:13:01.540 [main] INFO Application - Service starts up successfully!
```
`PatternLayout`支持的通配符有好几种，不必都使用，具体指出的通配符和格式的说明可参见：https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/PatternLayout.html

除了`PatternLayout`，为了支持数据统计和分析或压缩等目的，`log4J`还有`CsvParameterLayout`, `GelfLayout`, `JsonLayout`等，详情可见：https://logging.apache.org/log4j/2.x/manual/layouts.html

###### Appender Object
定义好了日志的输出格式之后，接下来的问题是，日志需要被打印到哪里，除了输出到文本文件，有时候我们还需要输出到控制台标准输出流中，那么`Appender Object`就承担了这部分的工作，提供可配置的选项，让用户指定日志输出的目的地。

耳聪目明的你一定发现了上面XML配置中的这一行：
```text
<Console name="Console" target="SYSTEM_OUT">
```
`Console`表示该使用了该`Appender`的logger 将会把日志输出到控制台中，当然也支持多个`Appender`，如果我们把以上XML配置调整为：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
    <File name="FileLog" target="target/test.log">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </File>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Console"/>
      <AppenderRef ref="FileLog"/>
    </Root>
  </Loggers>
</Configuration>
```

那么`target/test.log`日志文件和标准输出流中将都会打印格式为`%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n`的日志。

值得一提的是`Log4J`之所以被广泛使用，离不开它对多种日志输出的支持，除了可以输出到文件和输出流，还能直接和JDBC对接，将日志以结构化的格式存入数据库，数据库的类型也支持关系型和非关系型数据库。还有其他常见的Apache开源工具，比如Kafka, Cassandra。详情可见：https://logging.apache.org/log4j/2.x/manual/appenders.html

通过以上描述我们可知，一个通用的日志框架，提供给用户易用的日志打印接口，灵活易学的配置，小王不必操心去实现文件或控制流的输出模块，可专心在产品的开发上。

### 给日志分分类
在开发阶段，尤其是业务复杂时，小王总是会打印一些方便自己调试的日志信息，来帮助自己确定每一步逻辑实现正确。但当模块功能开发完成，代码运行在服务器上时，如果这些调试信息也原封不动出现在服务器的日志中时，对于开发人员来说，将是灾难性的：
1. 本地开发时的数据和请求通常是手动出发，量较小，调试日志也容易区分来自哪个输入。但产品生产化之后，处理的请求通常是并发且成千上万的，这时候针对某一功能都打印一遍某个中间变量的值，造成的结果是，出现成千上万的中间调试结果
2. 当出现线上issue时，处理的时限往往时ASAP，这时候开发人员需要迅速定位到和issue相关的日志，看看是否出现了exception，如果这时有大量无关的中间日志，无疑是增加了定位问题的难度
3. 在监控服务时，往往是对日志文件进行监控和文本处理，如果日志文件本身含有大量无关文本信息，也是增加了日志监控模块的负担

但小王想在本地调试时有足够的调试日志，发布服务时，又不希望有这些调试日志的出现。于是，为了解决不同开发阶段和运行环境对于日志的需求，日志框架统一使用`logging level`来对日志进行优先级的区分，不同的package也可以使用不同的优先级，允许用户自定义哪些日志可以出现在哪些日志里。

###### Logging Level
通常来说，日志的优先级由低到高有以下几个分级：
- __ALL__ level为ALL的日志，表示任何level的日志都可以输出，包括用户自定义的level
- __DEBUG__ 通常用作调试阶段打印的日志，开发人员通常用作打印中间结果，或自己关心的一些信息
- __INFO__ 通常表示程序运行progress的信息，例如启动服务时，用来打印启动阶段需要加载的各种依赖的情况，是否正确启动，web 服务启动在哪个端口等等和服务状态相关信息
- __WARN__ 可能出现错误，但还没有造成issue的信息，例如使用了一些deprecated 的函数。开发人员也可自行打印和业务相关的警告信息
- __ERROR__ 程序运行过程中抛出了异常，但不会影响程序继续运行。例如代码中出现了一些exception，但不影响服务运行。这也是我们调试时关心的一类日志
- __FATAL__ 导致服务终止的异常错误。例如和依赖的服务断开了网络连接，磁盘内存等资源不足等异常。但这个错误几乎不可能出现，如果服务已经挂了，怎么会打印日志呢？
- __OFF__ 任何日志都不再输出。通常用来屏蔽某些不相干的package日志

需要注意的是，`logging level`的使用分为两个阶段，即日志打印时指定level 和日志输出时配置的允许输出的level。
以小王的代码为例：
```
import org.apache.log4j.Logger;
public class Application{
    static Logger log = Logger.getLogger(log4jExample.class.getName());
    public static void main(String []args){
        log.setLevel(Level.WARN);
        log.debug("Debug Message!");
        log.info("Info Message!");
        log.warn("Warn Message!");
        log.error("Error Message!");
        log.fatal("Fatal Message!");
    }
}
```

logging level被设置为`WARN`，那么优先级在`WARN`以下的`INFO` `DEBUG`将不会输出，只会输出最后三行信息。logging level也可以在配置文件中指定，甚至可以指定某一个package的level。以`Log4J`的配置为例，如果配置如下：
```xml
  <Loggers>
    <Logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
      <AppenderRef ref="STDOUT"/>
    </Logger>
 
    <Logger name="org.apache.logging.log4j.test2" level="off" additivity="false">
      <AppenderRef ref="File"/>
    </Logger>
 
    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
```

这个配置的意思是`org.apache.logging.log4j.test1`这个package下面的日志信息level为`DEBUG`及以上的log才输出，`org.apache.logging.log4j.test2` level为`OFF`，任何日志都不会输出，而其余的package输出level为`ERROR` 那么`DEBUG` `WARN`这些级别的日志将会被忽略。 

### 谁调用了我的服务 - Access Log
经过了一系列努力，小王的服务终于上线，他机智的提供了HTTP服务，允许他的用户通过发送HTTP请求调用他的服务。但他并没有因此膨胀，他希望了解服务被使用的情况，每天有多次请求，这些请求被成功处理了吗，他的用户来自哪里。

这个时候，小王发现他的服务器上除了他打印的服务日志以为，还有一种叫`Access Log`的东西。`Access Log` 来自于服务器上负责处理HTTP请求的web service，常见的有Nignx, Apache Server。服务器接收并处理一个请求之后，会留下一行如下所示的记录：
```text
127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326
```
这一行日志描述的是：一个叫`frank`的用户，在`10/Oct/2000:13:55:36 -0700`时，从IP为`127.0.0.1`的客户端发来一个`GET`请求，请求的是`/apache_pb.gif` 这个文件，并且成功返回（`200`为成功的状态码）
，返回的数据大小为`2326`字节。
根据`Access Log`小王就可以统计得到他想知道的几个问题。

### 日志太多怎么办 - Log Rotation
小王管理了一段时间服务，但是他又遇到了新的问题，Access Log每天都在增长，而且所有日志都在`access.log`这一个文件中，有没有办法让这些日志按天输出呢，而且一个月之前的日志基本上不会再需要去查看了，能不能定时清理日志只保留最近30天的日志呢？

当然是有办法的，最简单的办法就是写一个定时脚本，每天凌晨0点的时候，将上一天的日志重命名，并新建`access.log`保存今天的日志，同时清理30天之前的日志。写出bash脚本如下所示：
``` bash
cd /path/to/accesslog/
mv access.log access_`date -r access.log +%Y%m%d%H%M%S`.log
touch access.log
service apache restart

# remove files older than 30 days
find /path/to/accesslog/ -type f -name '*.log' -mtime +30 -exec rm {} \;
```

Linux系统中有`logrotate`工具，通过配置时间和文件路径可以达到相同的效果。而日志框架中，也可以通过配置来保留指定日期内，或者按照最大文件大小来实现Log Rotation, 配置如下所示：
```xml
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
 
    <appender name="RollingAppender" class="org.apache.log4j.DailyRollingFileAppender">
       <param name="File" value="app.log" />
       <param name="DatePattern" value="'.'yyyy-MM-dd" />
       <layout class="org.apache.log4j.PatternLayout">
          <param name="ConversionPattern" value="[%p] %d %c %M - %m%n"/>          
       </layout>
    </appender>
    <root>
        <priority value="DEBUG"/>
        <appender-ref ref="RollingAppender" />
    </root>
</log4j:configuration> 
```

### 分布式日志怎么办？
经过小王的不懈努力，服务被越来越多的人使用，服务器的数量也从1台增加到了100台，新的问题又出现了，日志分布在100台服务器上，小王该怎么把日志收集到一个统一的地方管理呢，出现issue时小王也不可能在100台服务器的日志文件中一个个查找，有没有什么办法可以全局的搜索查找日志呢，最好那些access log统计结果也可以一并展示了吧。

小王永远不会是第一个碰到这些问题的人，为了解决这一系列问题，ELK解决方案应运而生，ELK提供了一整套服务，对分布式的日志进行集中式管理，解决了将分布的日志集中收集，传输存储，分析可视化展示，并允许用户自定义指标发送警告。

ELK是由 _Logstash_， _Elasticsearch_ 和 _Kibana_ 三部分组成：

![image](https://www.ibm.com/developerworks/cn/opensource/os-cn-elk/img001.png)

> 引用自：https://www.ibm.com/developerworks/cn/opensource/os-cn-elk/index.html

_Logstash_ 负责将日志搜集起来统一存储，_Elasticsearch_ 是针对非结构化文本数据搜索分析的利器，非常适合处理文本日志，_Kibana_ 是一套可视化交换工具，可以让用户灵活的配置自己的报表，并且可以在UI上执行Elasticsearch命令对日志进行搜索。

这三部分相对独立，由类似功能的其他工具，只要满足接口，也可替换。个人认为，_Grafana_ 比 _Kibana_ 在可视化方面更胜一筹， _Grafana_ 除了支持ES，也支持Mysql，innodb等存储，丰富的插件和社区资源。例如统计Access log中的请求，可通过 _Grafana_ 执行_Elasticsearch_命令，并将统计结果定制化展示在Dashboard上，，很多用户会在共享自己的Dashboard配置，可见：https://grafana.com/dashboards

相关链接：
- ElasticSearch: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
- Logstash: https://www.elastic.co/guide/en/logstash/current/introduction.html
- Kibana: https://www.elastic.co/guide/en/kibana/current/index.html
- Grafana: https://grafana.com/grafana

### 来自大佬的建议
小王已经遇到了大多数日志相关的问题，并找到了解决方法，随着经验的积累，他也有一些心得体会。这一段是zqsg胡说八道，其实是查阅资料时，发现了以为做了十几年DevOps的大佬写了一篇关于日志的十个建议，个人认为很值得一读：http://www.masterzen.fr/2013/01/13/the-10-commandments-of-logging/

其中觉得值得highlight的几个tips：
- 抛出exception，打印error log时，带上track message，不要只打印"User not found", "Process failed"这种信息，这是无用的，定位问题时，需要看到具体的异常信息
- 只打印英文log，因为你无法预测代码会在什么环境下运行，会用什么字符编码，用英文日志才能保证，在任何运行环境下不会因为编码问题，而打印日志异常，甚至影响服务运行
- 打印包含变量值的日志，尽量使用可parse的格式。比如，使用字符拼接打印一个和用户相关的信息：
    ```
    2013-01-12 17:49:37,656 [T1] INFO  c.d.g.UserRequest  User 1334563 plays 4 of spades in game 23425656
    ```
    如果需要提取日志中这一段信息，那就不得不写个正则表达式把这几个数值提取出来，如果打印成JSON格式，就会方便很多
- 考虑到哪些人会查看当前的日志，如果不是服务开发人员，应该用更容易阅读的方式打印日志，避免使用非开发人员不懂的话术

### 不同的编程语言
除了小王，还有很多使用其他编程语言的同学们，这里搜集了一些不同编程语言的Log相关介绍：
- Java: https://blog.scalyr.com/2018/07/started-quickly-spring-boot-logging/
- Python: https://docs.python.org/3/howto/logging-cookbook.html
- PHP: https://www.loggly.com/ultimate-guide/php-logging-basics/
- Go: http://www.golangprograms.com/go-language/logging-go-programs.html
- C++: http://www.drdobbs.com/cpp/logging-in-c/201804215
- NodeJS: https://www.loggly.com/blog/node-js-libraries-make-sophisticated-logging-simpler/

