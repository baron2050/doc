---
title: java多线程常用基础包
---

## java多线程

cas，countlunch，threadpool，

异步变同步：

```
package ml.doibest.demos.异步变同步;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;
/**
 * @Author admin01
 * @Date 2021/5/6 下午2:44
 * @Version 1.0
 */
public class test {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(()->{
            CountDownLatch countDownLatch = new CountDownLatch(1);
            new Thread(()->{
                //给一个消息处理器添加一个消费者
                try{
                    TimeUnit.SECONDS.sleep(3);
                    countDownLatch.countDown();
                }catch (Exception ex){

                }
            }).start();
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "ok";
        });
        System.out.println(completableFuture.get());

    }
}
```

# java常用基础包

```
<!-- https://mvnrepository.com/artifact/org.apache.commons/commons-lang3 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.11</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.apache.commons/commons-collections4 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>

guava  hutool 
```

