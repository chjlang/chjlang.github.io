---
layout: post
author: chjlang
title:  "Java事务"
date:   2017-02-22 17:51:00 +8000
categories: java
tags:
    - java
    - study
---

## jdbc事务
* 利用jdbc手动编写事务的代码模板如下：

```java
connection.setAutoCommit(false);
PreparedStatement ps = connection.prepareStatement(sqlString);
try {
    ps.executeUpdate();
    connection.commit();
} catch (SQLException e) {
    connection.rollback();
} finally {
    if (ps != null) {
        ps.close();
    }
    connection.setAutoCommit(false);
}
``` 

## ORM框架事务
*基于 ORM 的框架需要一个事务来触发对象缓存与数据库之间的同步*

以下代码并不会更改数据库

```java
public class TradingServiceImpl {
    @PersistenceContext(unitName="trading") EntityManager em;

    public long insertTrade(TradeData trade) throws Exception {
       em.persist(trade);
       return trade.getTradeId();
    }
}
```

原因如下，（参考[该页面说明](https://www.ibm.com/developerworks/library/j-ts1/)，意思就是ORM框架需要一个事务来将缓存中的对象修改同步到数据库中

>
Notice that Listing 3 invokes the persist() method on the EntityManager to insert the trade order. Simple, right? Not really. This code will not insert the trade order into the TRADE table as expected, nor will it throw an exception. It will simply return a value of 0 as the key to the trade order without changing the database. This is one of the first major pitfalls of transaction processing: ORM-based frameworks require a transaction in order to trigger the synchronization between the object cache and the database. It is through a transaction commit that the SQL code is generated and the database affected by the desired action (that is, insert, update, delete). Without a transaction there is no trigger for the ORM to generate SQL code and persist the changes, so the method simply ends — no exceptions, no updates. If you are using an ORM-based framework, you must use transactions. You can no longer rely on the database to manage the connections and commit the work.
