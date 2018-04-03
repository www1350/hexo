---
title: spring实现多数据源同时访问（跨数据库查询）
date: 2016-07-07 22:36:57
tags: spring
categories: spring
---

假如配置了两个数据源dataSourceOne、dataSourceTwo

``` xml
  <bean id="dynamicDataSource" class="com.core.DynamicDataSource">  
        <property name="targetDataSources">  
            <map key-type="java.lang.String">  
                <entry value-ref="dataSourceOne" key="dataSourceOne"></entry>  
                <entry value-ref="dataSourceTwo" key="dataSourceTwo"></entry>  
            </map>  
        </property>  
        <property name="defaultTargetDataSource" ref="dataSourceOne">  
        </property>  
    </bean> 
```

 DynamicDataSource.class

``` java
package com.core;  

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;  

public class DynamicDataSource extends AbstractRoutingDataSource{  

    @Override  
    protected Object determineCurrentLookupKey() {  
        return DatabaseContextHolder.getCustomerType();   
    }  

}  
```

DatabaseContextHolder.class

``` java
package com.core;  

public class DatabaseContextHolder {  

    private static final ThreadLocal<String> contextHolder = new ThreadLocal<String>();  

    public static void setCustomerType(String customerType) {  
        contextHolder.set(customerType);  
    }  

    public static String getCustomerType() {  
        return contextHolder.get();  
    }  

    public static void clearCustomerType() {  
        contextHolder.remove();  
    }  
}  
```

DataSourceInterceptor.class

``` java
package com.core;  

import org.aspectj.lang.JoinPoint;  
import org.springframework.stereotype.Component;  

@Component  
public class DataSourceInterceptor {  

    public void setdataSourceOne(JoinPoint jp) {  
        DatabaseContextHolder.setCustomerType("dataSourceOne");  
    }  

    public void setdataSourceTwo(JoinPoint jp) {  
        DatabaseContextHolder.setCustomerType("dataSourceTwo");  
    }  
}  
```

aop配置

``` xml
    <aop:config>  
        <aop:aspect id="dataSourceAspect" ref="dataSourceInterceptor">  
            <aop:pointcut id="daoOne" expression="execution(* com.dao.one.*.*(..))" />  
            <aop:pointcut id="daoTwo" expression="execution(* com.dao.two.*.*(..))" />  
            <aop:before pointcut-ref="daoOne" method="setdataSourceOne" />  
            <aop:before pointcut-ref="daoTwo" method="setdataSourceTwo" />  
        </aop:aspect>  
```