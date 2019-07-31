TBSchedule 4.1.1 Finally Released
===

- date: 2019-07-31
- tags: scheduling, distributed

------------

# Intro
TBSchedule is a non-centralised, zookeeper adapted, console included, task/strategy supported and light-weight scheduling-in-place library from taobao(taobao.com). It was released as an open source project in about (2010~2012) by Taobao in the website `http://code.taobao.org/p/tbschedule` (But it has been down in 2017), with no license given. And after that it seems that there is no further developing on it.

It's not a common distributed scheduling framework like ones support external shell, uploaded jars, cross languages and etc. Its main purpose is the scenarios of spring-quartz but don't settle for quartz's design of `High Availability` and `Clustering`. It's another choice to try with.

Project location(Under Apache® License): https://github.com/jasonjoo2010/tbschedule

# Evolution
## Traditional Versions (3.3.3.2 and earlier)
It uses `zookeeper` communicating with zookeeper server directly and sometimes suffers jitters on unstable connections. Surely only zookeeper storage is supported.

A console server is provided as a client tool to manage the strategies, tasks and do operations under specified namespace.

But there are problems in integrating:
* Use log4j for logging but not interface.
* Console is mixed with core library.
* Workers' distribution always includes the leader node. Which leads heavier load on leader.
* Some small bugs when do operations.

## Enhanced Versions (3.3.4 to 4.0.x)
Enhanced versions solve all the problem in traditional version and:

* Separate console to an individual module.
* Make `zookeeper` up to date.
* Integrate `slf4j` as logging library.
* Shuffle first when do new scheduling.
* Introduce extension module. (4.0.x)
* Introduce `curator`. (4.0.x)
* Restructure console into spring-boot project. (4.0.x)
* Publish to maven central repository. (Under group id `com.yoloho.schedule`)

So in 4.0.x you can use tbschedule easily (included in central repo) and logging friendly (slf4j) and more balanced (workers load balancing). If you implement queue consumer using Redis' list you can easily use `tbschedule-extension-task` to do it.

But it's not the end.

For `more adaptive`, it's not enough to support zookeeper only. It should change depend on what the project use. For example a redis/jdbc only project we should not have to introduce a zookeeper cluster for scheduling. And also we should provide easier ways to integrate like xml/annotations.

For `more international`, we should make more documentation using english.

For `more reliable`, we should write more unit tests.

For `learn easily`, we should provide some demonstrations.

So that's why 4.1.x is raised.

## Multiple Storages (4.1.x)
Over half of codes have been restructured. `SPI` is used to support different `default` storage implementation and it also can be specified manually. Which means we can use multiple storages together in single project (If there's such scenario, eg. memory and zookeeper together).

Here are storages supported:

* Memory. For local only scheduling driven by annotations. (EnableScheduleMemory/Strategy/Task)
* Zookeeper. (By `curator-framework`)
* Redis. (By `enhanced-cache`)

`JDBC` is on the way.

In spring boot it's easy to initialize:

```java
@EnableSchedule(
    // Storage is zookeeper by default
    address = "192.168.123.106:2181",
    rootPath = "/test/demo/tmp",
    username = "test",
    password = "test"
)
```

And then you can raise a `console` instance locally which has been packed and uploaded into maven central repository to manage the tasks and strategies.

Details please refer to: https://github.com/jasonjoo2010/tbschedule

# Summary
It borns to be a light-weight and flexible scheduling and hope it could be a good choice on this.

Thanks to the `Taobao's tbschedule`. Thanks to their spirit of open source. Thanks to Xuannan who has done most of the contribution of Taobao's tbschedule.


