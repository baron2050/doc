---
title: java-commons-pool2
---

```java
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

使用说明：

第一步：创建对象

第二步：实现BasePooledObjectFactory类

第三步：初始化GenericObjectPool

第四步：使用并释放

```java
package ml.doibest.demos.testpools;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.pool2.BasePooledObjectFactory;
import org.apache.commons.pool2.PooledObject;
import org.apache.commons.pool2.impl.DefaultPooledObject;

@Slf4j
public class ClientPoolFactory extends BasePooledObjectFactory<ManagerConnection> {

    @Override
    public ManagerConnection create() throws Exception {
        ManagerConnection conn = new ManagerConnection();
        try {
            conn.connect();
        } catch (Exception ex) {
            log.error("connect exception.", ex);
            return null;
        }
        return conn;
    }

    @Override
    public PooledObject<ManagerConnection> wrap(ManagerConnection managerConnection) {
        return new DefaultPooledObject<>(managerConnection);
    }
}
```

```java
package ml.doibest.demos.testpools;

import cn.hutool.extra.ssh.Sftp;

public class ManagerConnection {
    private String hostname = "127.0.0.1";
    private int port = 22;
    protected String password = "doibest";
    private Sftp sftp;
    public String getHostname() {
        return this.hostname;
    }
    public int getPort() {
        return this.port;
    }
    public String getPassword() {
        return this.password;
    }
    public Sftp getSftp() {
        return this.sftp;
    }
    public void connect() {
        sftp = new Sftp(hostname, 22, "admin01", password);
    }
    public void disconnect() {
        sftp.close();
    }
}
```

```java
package ml.doibest.demos.testpools;

import org.apache.commons.pool2.impl.GenericObjectPool;

public class Run {
    public static void main(String[] args) throws Exception {
        GenericObjectPool<ManagerConnection> pool = new GenericObjectPool<ManagerConnection>(new ClientPoolFactory());
        pool.setTestOnBorrow(true);
        pool.setMaxTotal(5);
        pool.setMinIdle(3);
        pool.setMaxIdle(1);
        pool.setMaxWaitMillis(5);
        for (int i = 0; i < 5; i++) {
            pool.addObject();
        }
        for (int i = 0; i < 500; i++) {
            ManagerConnection managerConnection = null;
            try {
                managerConnection = pool.borrowObject();
                System.out.println(managerConnection.password);
                System.out.println("池"+pool.getNumIdle());
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if(managerConnection != null) pool.returnObject(managerConnection);
            }
        }
    }
}
```

