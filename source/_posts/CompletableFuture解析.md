---
title: CompletableFuture解析
date: 2018-04-08 00:16:54
tags: [Java]
categories: [Java]
description: 本文对Java8新增的CompletableFuture进行详细解析
---
## 初出茅庐

什么是CompletableFuture？个人理解，CompletableFuture是针对Java 5中Future的一个扩展，可能有人会问什么是Future，Future是对将来某个时间点完成的结果。

先简单看一个Future的例子

```
import java.util.concurrent.*;

public class FutureAndCallableExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        Callable<String> callable = () -> {
            // Perform some computation
            System.out.println("Entered Callable");
            Thread.sleep(2000);
            return "Hello from Callable";
        };

        System.out.println("Submitting Callable");
        Future<String> future = executorService.submit(callable);

        // This line executes immediately
        System.out.println("Do something else while callable is getting executed");

        System.out.println("Retrieve the result of the future");
        // Future.get() blocks until the result is available
        String result = future.get();
        System.out.println(result);

        executorService.shutdown();
    }

}
```

```Java
# Output
Submitting Callable
Do something else while callable is getting executed
Retrieve the result of the future
Entered Callable
Hello from Callable
```

Future接口包括以下几个方法

> - boolean cancel(boolean)
> - boolean isCancelled()
> - boolean isDone()
> - V get()
> - V get(long,TimeUnit)

既然有了Future这种基于异步的接口，那么为什么Java8又推出了CompletableFuture呢？

来看下Future的几个局限性

> - 不能手动触发完成
> - 如果不阻塞，无法对Future的结果进行进一步操作
> - 不支持链式操作
> - 不支持组合操作
> - 无异常处理

## 小试牛刀

一个CompletableFuture的例子

```
CompletableFuture<String> completableFuture = new CompletableFuture<String>();
// 如果你不手动触发完成的操作，get()将被永远阻塞
String result = completableFuture.get()

completableFuture.complete("Future's Result")
```



CompletableFuture基本使用

> - static CompletableFuture<Void> runAsync(Runnable runnable)
> - static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
> - static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
> - static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)

```
// 当不需要获取返回值使用runAsync() 
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    // Simulate a long-running Job   
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("I'll run in a separate thread than the main thread.");
});
```

```
// 当需要获取返回值时使用supplyAsync()
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "Result of the asynchronous computation";
});
```

```
// 指定线程池的使用
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "Result of the asynchronous computation";
}, executor);
```

这里线程池的数量的大小有一个公式的可以作为参考

Nthreads = NCPU * UCPU * (1 + W/C)

> * NCPU是处理器的核数目，可以通过Runtime.getRuntime().availableProcessors()得到
> * UCPU是cpu的利用率，介于0~1之间
> * W/C是等待时间与计算时间的比率

## 勤学苦练

### CompletableFuture的链式操作

> * thenApply()
> * thenAccept()
> * thenRun()


```
// thenApply()对CompletableFuture的记过进行链式操作
CompletableFuture<String> welcomeText = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return "Rajeev";
}).thenApply(name -> {
    return "Hello " + name;
}).thenApply(greeting -> {
    return greeting + ", Welcome to the CalliCoder Blog";
});
System.out.println(welcomeText.get());
// Prints - Hello Rajeev, Welcome to the CalliCoder Blog
```

```
// thenAccept()不依赖CompletableFuture的结果，但是要等到获取到结果再处理
CompletableFuture.supplyAsync(() -> {
    return ProductService.getProductDetail(productId);
}).thenAccept(product -> {
    System.out.println("Got product detail from remote service " + product.getName())
});
```

```
// thenRun()不依赖上一操作的结果，另起一个线程处理
CompletableFuture.supplyAsync(() -> {
    // Run some computation  
}).thenRun(() -> {
    // Computation Finished.
});
```
### CompletableFuture的合并操作
> * thenCompose()
> * thenCombine()

```
// thenCompose()合并两个CompletableFuture操作，但是两个操作存在依赖关系
CompletableFuture<User> getUsersDetail(String userId) {
    return CompletableFuture.supplyAsync(() -> {
        UserService.getUserDetails(userId);
    }); 
}

CompletableFuture<Double> getCreditRating(User user) {
    return CompletableFuture.supplyAsync(() -> {
        CreditRatingService.getCreditRating(user);
    });
}

CompletableFuture<Double> result = getUserDetail(userId)
.thenCompose(user -> getCreditRating(user));
```

```
// thenCombine()合并两个无关的操作
CompletableFuture<Double> weightInKgFuture = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return 65.0;
});

System.out.println("Retrieving height.");
CompletableFuture<Double> heightInCmFuture = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return 177.8;
});

System.out.println("Calculating BMI.");
CompletableFuture<Double> combinedFuture = weightInKgFuture
        .thenCombine(heightInCmFuture, (weightInKg, heightInCm) -> {
    Double heightInMeter = heightInCm/100;
    return weightInKg/(heightInMeter*heightInMeter);
});

System.out.println("Your BMI is - " + combinedFuture.get());
```
### CompletableFuture的批量操作
> * static CompletableFuture<Void>   allOf(CompletableFuture<?>... cfs)
> * static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)


```
// allOf()对多个任务进行操作
CompletableFuture<String> downloadWebPage(String pageLink) {
    return CompletableFuture.supplyAsync(() -> {
        // Code to download and return the web page's content
    });
} 
List<String> webPageLinks = Arrays.asList(...)    // A list of 100 web page links

// Download contents of all the web pages asynchronously
List<CompletableFuture<String>> pageContentFutures = webPageLinks.stream()
        .map(webPageLink -> downloadWebPage(webPageLink))
        .collect(Collectors.toList());


// Create a combined Future using allOf()
CompletableFuture<Void> allFutures = CompletableFuture.allOf(
        pageContentFutures.toArray(new CompletableFuture[pageContentFutures.size()])
);

// When all the Futures are completed, call `future.join()` to get their results and collect the results in a list -
CompletableFuture<List<String>> allPageContentsFuture = allFutures.thenApply(v -> {
   return pageContentFutures.stream()
           .map(pageContentFuture -> pageContentFuture.join())
           .collect(Collectors.toList());
});

// Count the number of web pages having the "CompletableFuture" keyword.
CompletableFuture<Long> countFuture = allPageContentsFuture.thenApply(pageContents -> {
    return pageContents.stream()
            .filter(pageContent -> pageContent.contains("CompletableFuture"))
            .count();
});

System.out.println("Number of Web Pages having CompletableFuture keyword - " + 
        countFuture.get());

```

```
// anyOf()当有一个任务执行完成便返回结果
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return "Result of Future 1";
});

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return "Result of Future 2";
});

CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return "Result of Future 3";
});

CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(future1, future2, future3);

System.out.println(anyOfFuture.get()); // Result of Future 2
```

### CompletableFuture异常处理

> * exceptionally()
> * handle()

```
Integer age = -1;

// exceptionally()如果你处理过一次该错误，则回调链中不会再传播该错误
CompletableFuture<String> maturityFuture = CompletableFuture.supplyAsync(() -> {
    if(age < 0) {
        throw new IllegalArgumentException("Age can not be negative");
    }
    if(age > 18) {
        return "Adult";
    } else {
        return "Child";
    }
}).exceptionally(ex -> {
    System.out.println("Oops! We have an exception - " + ex.getMessage());
    return "Unknown!";
});

System.out.println("Maturity : " + maturityFuture.get()); 

```

```
Integer age = -1;

// handle(res,ex)异常存在则res为null，否则ex为null
CompletableFuture<String> maturityFuture = CompletableFuture.supplyAsync(() -> {
    if(age < 0) {
        throw new IllegalArgumentException("Age can not be negative");
    }
    if(age > 18) {
        return "Adult";
    } else {
        return "Child";
    }
}).handle((res, ex) -> {
    if(ex != null) {
        System.out.println("Oops! We have an exception - " + ex.getMessage());
        return "Unknown!";
    }
    return res;
});

System.out.println("Maturity : " + maturityFuture.get());

```

## 华山论剑

待续。。。