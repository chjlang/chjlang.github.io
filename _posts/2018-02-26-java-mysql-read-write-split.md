---
layout: post
author: chjlang
title:  "使用MySQL的ReplicationDriver实现数据库读写分离"
date:   2018-02-26 15:51:00 +8000
categories: java
tags:
    - java
    - study
---

## 为什么要进行数据库读写分离

生产环境中的数据库一般是有一个主库来处理写的请求，另外通过异步复制的方式更新从库上的数据，这些从库一方面可以作为数据备份，在主库挂掉时，可以从这些从库恢复数据，另一方面可以扩展数据库集群的读的能力，客户端可以把所有读请求只发送到这些从库，降低主库的压力。如果数据库的读的压力特别大，可以设置多个从库来水平扩展数据库集群的读性能。

## 使用MySQL中的ReplicationDriver进行读写分离

读写分离的基本原则是所有的写请求都发送到主库，读请求发送到从库。在实际写代码时，我们可以分别与主库和从库分别建立连接，若要执行的事务需要包含写操作，则使用与主库建立的连接，否则使用与从库建立的连接。但是如果要手动写这些代码，需要自己维护多条连接的状态，代码容易出错。还好MySQL提供了一个叫ReplicationDriver的驱动可以帮忙我们很容易地实现读写分离，在spring boot只需要如下设置即可，spring boot会自动使用ReplicationDriver驱动去建立数据库连接，

```
spring.datasource.url=jdbc:mysql:replication://masterHost:masterPort,slaveHost1:slavePort1,slaveHost2:slavePort2/dbName
```

之后，代码中所有标记了只读（加上```@Transactional(readOnly = true)```注解）的事务操作都会把请求发到slave节点上，其它没标记为只读的请求会发向master节点。这样通过简单的配置和注解就能在spring boot非常方便地实现MySQL的读写分离了。

简单例子：
```java
@Service
class DemoDAO {

	// 没有标记为readOnly，请求会发向master
	@Transactional
	void save(Object obj) {
		// db save...
	}

	// 标记为了readOnlye，请求会发向slave
	@Transactional(readOnly = true)
	Object get(int id) {
		// db save
	}
}
```

若直接使用JDBC的API也是可以使用ReplicationDriver的，但需要手动配置一下driver以及连接上的readOnly属性，简单例子如下：

```java
import java.sql.Connection;
import java.sql.ResultSet;
import java.util.Properties;
 
import com.mysql.jdbc.ReplicationDriver;
 
public class ReplicationDriverDemo {
 
  public static void main(String[] args) throws Exception {
    ReplicationDriver driver = new ReplicationDriver();
 
    Properties props = new Properties();
 
    props.put("autoReconnect", "true");
    props.put("roundRobinLoadBalance", "true");
 
    props.put("user", "foo");
    props.put("password", "bar");
 
    Connection conn =
        driver.connect("jdbc:mysql:replication://master,slave1,slave2,slave3/test",
            props);
 
    // 写操作，设置readOnly为false，请求会发向master
    conn.setReadOnly(false);
    conn.setAutoCommit(false);
    conn.createStatement().executeUpdate("UPDATE some_table ....");
    conn.commit();
 
    // 只读操作，设置readOnly为true，请求会发向slave
    conn.setReadOnly(true);
 
    ResultSet rs =
      conn.createStatement().executeQuery("SELECT a,b FROM alt_table");
 
     .......
  }
}
```
### 原理

ReplicationDriver的工作原理是将一条逻辑连接对应多条物理连接，这些物理连接分别连接在host里指定的master和slave节点，例如连接填写的是```jdbc:mysql:replication://master,slave1,slave2,slave3/test```，则ReplicationDriver的一条逻辑连接包含四条分别连接master,slave1,slave2和slave3的物理连接。该逻辑连接在MySQL connector/J中被封装成一个代理，对外提供JDBC定义的Connection接口，内部维护物理连接的状态。详见[ReplicationConnectionProxy](https://github.com/mysql/mysql-connector-j/blob/release/5.1/src/com/mysql/jdbc/ReplicationConnectionProxy.java)


然后使用JDBC中Connection的setReadOnly这个方法作为hook，在设置事务只读属性调用该方法时切换底层的物理连接。见代码：

```java
public synchronized void setReadOnly(boolean readOnly) throws SQLException {
    if (readOnly) {
        if (!isSlavesConnection() || this.currentConnection.isClosed()) {
            boolean switched = true;
            SQLException exceptionCaught = null;
            try {
                switched = switchToSlavesConnection();
            } catch (SQLException e) {
                switched = false;
                exceptionCaught = e;
            }
            if (!switched && this.readFromMasterWhenNoSlaves && switchToMasterConnection()) {
                exceptionCaught = null; // The connection is OK. Cancel the exception, if any.
            }
            if (exceptionCaught != null) {
                throw exceptionCaught;
            }
        }
    } else {
        if (!isMasterConnection() || this.currentConnection.isClosed()) {
            boolean switched = true;
            SQLException exceptionCaught = null;
            try {
                switched = switchToMasterConnection();
            } catch (SQLException e) {
                switched = false;
                exceptionCaught = e;
            }
            if (!switched && switchToSlavesConnectionIfNecessary()) {
                exceptionCaught = null; // The connection is OK. Cancel the exception, if any.
            }
            if (exceptionCaught != null) {
                throw exceptionCaught;
            }
        }
    }
    this.readOnly = readOnly;

    /*
     * Reset masters connection read-only state if 'readFromMasterWhenNoSlaves=true'. If there are no slaves then the masters connection will be used with
     * read-only state in its place. Even if not, it must be reset from a possible previous read-only state.
     */
    if (this.readFromMasterWhenNoSlaves && isMasterConnection()) {
        this.currentConnection.setReadOnly(this.readOnly);
    }
}
```

### 注意的问题
实际使用数据库时，我们一般使用连接池，即与数据库建立并保持多条连接，每次需要使用时都是从该连接池中取一条空闲的连接出来，使用完毕后把该连接归还到连接池，这样可以避免每次执行数据库操作时都需要新建数据库连接带来的开销，提升效率。

使用数据库连接池很重要的一点就是从数据库拿到一条连接后，需要验证这条连接是否还是有效的。因为数据库服务器的连接数资源是有限且宝贵的，数据库会关闭那些长期没有操作的空闲连接。因此如果你从连接池中拿到一条连接且该连接已经很长时间没有对数据库进行操作的话，该连接很可能已经被服务端那边关闭了。这时如果直接使用该连接的话，会看到类似以下的异常:
```
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
The last packet successfully received from the server was 18,058 milliseconds ago.  The last packet sent successfully to the server was 0 milliseconds ago.

```
所以从连接池拿到一条连接后，需要先验证它是否仍然有效，MySQL提供了validation query这个选项让用户指定一条查询，使用该连接前先执行这条查询去验证连接是否生效。既然每次拿到该连接时都要执行该查询，那么这条查询必需是轻量级的，否则会产生大量额外的开销。网上很多人都使用的是```SELECT 1```，（spring boot在1.5.x之前的默认配置也是这样写的）认为最简单，不需要执行实际的数据查找。

实际上MySQL官方推荐的validation query应该以```/* ping */```开头，这样MySQL会使用ping一下服务器的方式去验证连接是否还有效，不会触发MySQL的查询或计算逻辑，更加轻量。更重要的是，对于使用ReplicationDriver的连接来说，以```/* ping */```开头的validation query会触发MySQL connector/J对一条逻辑连接下的多条物理连接都去发ping请求，保证该连接下的所有物理连接都是可用的。而其它的validation query只会在其中一条物理连接上执行，从而导致只验证了其中一条连接，但实际执行查询可能是另一条已经被关闭的连接，从而报错，抛出的异常信息与没设置validation query的情况一样：
```
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
The last packet successfully received from the server was 18,058 milliseconds ago.  The last packet sent successfully to the server was 0 milliseconds ago.

```

因此，**设置validation query时应该设置为```/* ping */ SELECT 1```**。spring boot 1.5.x之前的版本的默认设置也是设置了```SELECT 1```，见[DatabaseDriver](https://github.com/spring-projects/spring-boot/blob/1.4.x/spring-boot/src/main/java/org/springframework/boot/jdbc/DatabaseDriver.java#L69)，当时我发现了这个问题并且在github在提了[issue](https://github.com/spring-projects/spring-boot/issues/11958)，spring的人确认是bug，可惜没有提PR，反而被另一个人看到我的issue后给他们提了PR并且merged了[见PR](https://github.com/spring-projects/spring-boot/pull/11981)（错过了给spring贡献代码的机会，虽然只是加几个字符。。）
在使用1.5.x之前spring boot版本的同学可以在配置文件中配置一下就可以覆盖spring boot的默认配置：
```
spring.datasource.tomcat.validationQuery=/* ping */ SELECT 1
```
## 总结
本文主要介绍了数据库读写分离的用处，如何使用MySQL的ReplicationDriver去在实际开发时使用读写分离并分析了ReplicationDriver的实现原理，同时指出使用数据库连接池时需要注意的验证连接的问题。

