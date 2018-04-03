---
title: Intellij+ JRebel +maven+jetty实现热部署
abbrlink: be301519
date: 2018-04-03 22:39:37
tags:
categories:
---

pom.xml

```
            <plugin>
                <groupId>org.mortbay.jetty</groupId>
                <artifactId>jetty-maven-plugin</artifactId>
                <configuration>
                    <scanIntervalSeconds>1</scanIntervalSeconds>
                    <reload>automatic</reload>
                    <stopPort>9966</stopPort>
                    <stopKey>foo</stopKey>
                    <contextXml>${project.basedir}/src/main/resources/jetty-context.xml</contextXml>
                    <connectors>
                        <connector implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">
                            <port>8080</port>
                            <maxIdleTime>60000</maxIdleTime>
                        </connector>
                    </connectors>
                    <webAppSourceDirectory>${basedir}/WebRoot</webAppSourceDirectory>
                    <webAppConfig>
                        <contextPath>/absurd</contextPath>
                    </webAppConfig>
                </configuration>
            </plugin>
```

关于这部分的参数，详见[链接](http://www.importnew.com/17936.html)

安装jrebel以及破解见
[IDEA破解](http://blog.csdn.net/mark_sssss/article/details/51259117)
[jrebel破解](http://my.oschina.net/boltwu/blog/676606?p={{currentPage-1}})

![qq 20160909115113](https://cloud.githubusercontent.com/assets/7789698/18374922/c4e9734c-7683-11e6-9c79-e0a8d53ddbf5.png)
![qq 20160909115148](https://cloud.githubusercontent.com/assets/7789698/18374928/d61dfc78-7683-11e6-9781-0d567d9e8bc1.png)

${project.basedir}/src/main/resources/jetty-context.xml

```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configure PUBLIC "-//Mort Bay Consulting//DTD Configure//EN" "http://www.eclipse.org/jetty/configure.dtd">
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <Call name="setAttribute">
        <Arg>org.eclipse.jetty.server.webapp.WebInfIncludeJarPattern</Arg>
        <Arg>.*/.*jsp-api-[^/]\.jar$|./.*jsp-[^/]\.jar$|./.*taglibs[^/]*\.jar$</Arg>
    </Call>
</Configure>
```

ctrl+shift+f9编译当前class
ctrl+f9编译全部