
# Basic
## Simple Usage
```java
public Future<String> boilerPlateFuture() throws InterruptedException {
    CompletableFuture<String> completableFuture
            = new CompletableFuture<>();

    //		System.out.println(Thread.currentThread().getName());
    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        //			System.out.println(Thread.currentThread().getName());
        completableFuture.complete("Hello, I am " + Thread.currentThread().getName());
        return null;
    });

    return completableFuture;
}
```

## get() VS join()
get throws checked exception when future completed exceptionally
join() throws unchecked exception when future completed exceptionally
```java
String combined = Stream.of(future1, future2, future3)
  .map(CompletableFuture::join)
  .collect(Collectors.joining(" "));
```

## Supply, Apply and Accept
```java
Future<Void> future = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do supplyAsync");
    return "Hello";
}).thenApply(s -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do thenApply");
    return s + " World";
}).thenAccept(s -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do thenAccept");
    log.info("future result : {}", s);
});

log.info("do main");
```

Result : 
```
10:33:41.1 [main] - do main
10:33:42.1 [ForkJoinPool.commonPool-worker-1] - do supplyAsync
10:33:43.1 [ForkJoinPool.commonPool-worker-1] - do thenApply
10:33:44.1 [ForkJoinPool.commonPool-worker-1] - do thenAccept
10:33:44.1 [ForkJoinPool.commonPool-worker-1] - future result : Hello World
```

# Weaving Futures

## Supply and Compose

```java
Future<Void> future = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do supplyAsync");
    return "Hello";
}).thenCompose(s -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do thenCompose");
    return CompletableFuture.supplyAsync(() -> {
        try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
        log.info("do supplyAsync");
        return s + " World";
    }).thenAccept(t -> {
        try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
        log.info("do thenAccept-chained");
    });
}).thenAccept(s -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do thenAccept");
    log.info("future result : {}", s);
});

log.info("do main");
```

Result :
```
10:55:06.6 [main] - do main
10:55:07.6 [ForkJoinPool.commonPool-worker-1] - do supplyAsync
10:55:08.6 [ForkJoinPool.commonPool-worker-1] - do thenCompose
10:55:09.6 [ForkJoinPool.commonPool-worker-2] - do supplyAsync
10:55:10.6 [ForkJoinPool.commonPool-worker-2] - do thenAccept-chained
10:55:11.6 [ForkJoinPool.commonPool-worker-2] - do thenAccept
10:55:11.6 [ForkJoinPool.commonPool-worker-2] - future result : null
```

This result doesn't seem differ with result from Supply and Apply.
The difference is :
thenApply : Nested future (map)
thenCompose : Chaining independent future (flatMap)

`thenCompose()` is useful when assembling modularized code :
```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 1);
CompletableFuture<Double> future2 = CompletableFuture.supplyAsync(() -> 1.0);

CompletableFuture<CompletableFuture<Double>> nested = future1.thenApply(i -> future2);
CompletableFuture<Double> chained = future1.thenCompose(i -> future2);
```

Future of thenCompose is executed in sequential (it's just chaining).
Future of `thenCombine` works in parallel.

## Supply and Combine

```java
Future<Void> future = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do supplyAsync");
    return "Hello";
}).thenCombine(
        CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
            log.info("do supplyAsync");
            return " World";
        }),
        (s1, s2) -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
            log.info("do combining");
            return s1 + s2;
        }
).thenAccept(s -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do thenAccept");
    log.info("future result : {}", s);
});

log.info("do main");
```

Result :
```
10:51:32.5 [main] - do main
10:51:33.5 [ForkJoinPool.commonPool-worker-1] - do supplyAsync
10:51:33.5 [ForkJoinPool.commonPool-worker-2] - do supplyAsync
10:51:34.5 [ForkJoinPool.commonPool-worker-2] - do combining
10:51:35.5 [ForkJoinPool.commonPool-worker-2] - do thenAccept
10:51:35.5 [ForkJoinPool.commonPool-worker-2] - future result : Hello World
```

## Accept Both
`thenAcceptBoth` is kind of `thenCombine().thenAccept`
```java
Future<Void> future = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do supplyAsync");
    return "Hello";
}).thenAcceptBoth(
        CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
            log.info("do supplyAsync");
            return " World";
        }),
        (s1, s2) -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
            log.info("do accepting");
            log.info("future result : {}", s1 + s2);
        }
);

log.info("do main");
```

Result :
```
11:01:16.8 [main] - do main
11:01:17.8 [ForkJoinPool.commonPool-worker-2] - do supplyAsync
11:01:17.8 [ForkJoinPool.commonPool-worker-1] - do supplyAsync
11:01:18.8 [ForkJoinPool.commonPool-worker-2] - do accepting
11:01:18.8 [ForkJoinPool.commonPool-worker-2] - future result : Hello World
```

## Multiple Futures
```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    log.info("future1");
    return 1;
});
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
    log.info("future2");
    return 2;
});
CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> {
    log.info("future3");
    return 3;
});
List<CompletableFuture> listOfFuture = new ArrayList<>();
listOfFuture.add(future1);
listOfFuture.add(future2);
listOfFuture.add(future3);

CompletableFuture.allOf(listOfFuture.toArray(new CompletableFuture[listOfFuture.size()]))
        .thenAccept(s -> {
            log.info("allOf");
            listOfFuture.stream()
                    .map(CompletableFuture::join)
                    .forEach(r -> log.info("result : {}", r));
        }).join();
```
Result :
```
12:45:08.7 [ForkJoinPool.commonPool-worker-1] - future1
12:45:08.7 [ForkJoinPool.commonPool-worker-3] - future3
12:45:08.7 [ForkJoinPool.commonPool-worker-2] - future2
12:45:08.7 [ForkJoinPool.commonPool-worker-2] - allOf
12:45:08.7 [ForkJoinPool.commonPool-worker-2] - result : 1
12:45:08.7 [ForkJoinPool.commonPool-worker-2] - result : 2
12:45:08.7 [ForkJoinPool.commonPool-worker-2] - result : 3
```


# Exception Handling
## handle
`handler()` is similar to `thenApply()` except it gives way to handle exception occured in previously nested future.
```java
String name = null;
Future<Void> future = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do supplyAsync");
    if (name == null) throw new RuntimeException("Computation error!");
    return "Hello, " + name;
}).handle((s, ex) -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do handle");
    return ex != null ? "Hello, Unknown!" : s;
}).thenAccept(s -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do accept");
    log.info("future result : {}", s);
});

log.info("do main");
```

## exceptionally

1. one
    ```java
    Future<Integer> future = CompletableFuture.supplyAsync(() -> {
        int val = 1;
        log.info("do supplyAsync");
        return val;
    }).thenApply(s -> {
        log.info("do thenApply 1");
        if (1 == 1) throw new RuntimeException();
        return s + 1;
    }).thenApply(s -> {
        log.info("do thenApply 2");
        return s + 1;
    }).exceptionally(ex -> 100);

    log.info("result : {}", future.get());
    ```
    ```
    12:28:07.8 [ForkJoinPool.commonPool-worker-1] - do supplyAsync
    12:28:07.8 [ForkJoinPool.commonPool-worker-1] - do thenApply 1
    12:28:07.8 [main] - result : 100
    ```
1. two
    ```java
    Future<Integer> future = CompletableFuture.supplyAsync(() -> {
        int val = 1;
        log.info("do supplyAsync");
        return val;
    }).thenApply(s -> {
        log.info("do thenApply 1");
        return s + 1;
    }).thenApply(s -> {
        log.info("do thenApply 2");
        if (1 == 1) throw new RuntimeException();
        return s + 1;
    }).exceptionally(ex -> 100);

    log.info("result : {}", future.get());
    ```
    ```
    12:29:00.5 [ForkJoinPool.commonPool-worker-1] - do supplyAsync
    12:29:00.5 [ForkJoinPool.commonPool-worker-1] - do thenApply 1
    12:29:00.5 [ForkJoinPool.commonPool-worker-1] - do thenApply 2
    12:29:00.5 [main] - result : 100
    ```
1. three
    ```java
    Future<Integer> future = CompletableFuture.supplyAsync(() -> {
        int val = 1;
        log.info("do supplyAsync");
        return val;
    }).thenApply(s -> {
        log.info("do thenApply 1");
        return s + 1;
    }).exceptionally(ex -> 100)
    .thenApply(s -> {
        log.info("do thenApply 2");
        if (1 == 1) throw new RuntimeException();
        return s + 1;
    });

    log.info("result : {}", future.get());
    ```
    ```
    12:30:01.183 [ForkJoinPool.commonPool-worker-1] INFO rakuten.travel.reservation.steps.StepsApplication - do supplyAsync
    12:30:01.187 [ForkJoinPool.commonPool-worker-1] INFO rakuten.travel.reservation.steps.StepsApplication - do thenApply 1
    12:30:01.187 [ForkJoinPool.commonPool-worker-1] INFO rakuten.travel.reservation.steps.StepsApplication - do thenApply 2
    Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.RuntimeException
	    at ...
    Caused by: java.lang.RuntimeException
    	at ...
    ```

## completeExceptionally
Complete Future with Exception
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
    log.info("do supplyAsync");
    return "Hello";
});
future.completeExceptionally(new RuntimeException("Failed"));
future.get(); // This line will cause ExecutionException
```
