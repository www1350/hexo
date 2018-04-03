---
title: tomcat 启用gzip
date: tomcat 启用gzip 21:47:45
tags: tomcat
categories: 容器 
---

编辑conf/server.xml文件

``` xml
<Connector
port="8080"               maxHttpHeaderSize="8192"
               maxThreads="150" minSpareThreads="25" maxSpareThreads="75"
               enableLookups="false" redirectPort="8443" acceptCount="100"
               connectionTimeout="20000" disableUploadTimeout="true"
      compression="on"
compressionMinSize="2048"
noCompressionUserAgents="gozilla,traviata" 
compressableMimeType="text/html,text/xml,text/javascript,text/css,text/plain" />
```

1) compression="on" 打开压缩功能
2) compressionMinSize="2048" 启用压缩的输出内容大小，这里面默认为2KB
3) noCompressionUserAgents="gozilla, traviata" 对于以下的浏览器，不启用压缩<60;
4) compressableMimeType="text/html,text/xml" 压缩类型


http://hongjiang.info/index/tomcat/