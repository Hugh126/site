---
title: CompletableFuture
date: 2024-04-28 19:19:02
tags:
- java
categories: Spring+
---
你是否曾经为了追求任务更高性能而使用多线程，又是否因为多线程任务组合搞得代码散乱。那么一个适合任务编排的神奇来了
<!--more-->
# 一，使用方法

方法太多，层次不同，但有规律可寻。


## 1. 开始异步调用

> runAsync / supplyAsync

* runAsync 无返回值
* supplyAsync 有返回值

```java
    @Test
    void test0() throws ExecutionException, InterruptedException {
        CompletableFuture.runAsync(() -> {
           log.info("异步task1");
        });

        CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
            String name = "异步task2";
            log.info(name);
            return name;
        });
  
        log.info(cf2.get());
    }

11:29:09.352 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - 异步task1
11:29:09.352 [ForkJoinPool.commonPool-worker-2] INFO java8.CompletableFutureTest - 异步task2
```

类似的都有一个参数类型为 Executor 的同名方法，其实就是指定执行的线程池。默认的是 ForkJoinPool

```java
    /**
     * Default executor -- ForkJoinPool.commonPool() unless it cannot
     * support parallelism.
     */
    private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```


## 2.强大的管道

> thenRun / thenApply / thenAccept

* thenRun          <Runnable> 无参无输出
* thenApply      <Function>    入参并返回
* thenAccept     <Consumer>  入参无返回

```java
    @Test
    void test1() {
        CompletableFuture<Void> completableFuture = new CompletableFuture();
        completableFuture.thenRun(() -> {
            log.info("task 1");
        }).supplyAsync(() -> {
            log.info("task 2");
            return "task2";
        }).thenApply((param) -> {
            String result = String.format("[apply param = %s]", param);
            log.info(result);
            return result;
        }).thenAccept((consumeData) -> {
            log.info("[accept = {}]", consumeData);
        });
        completableFuture.complete(null);
    }


11:39:19.631 [main] INFO java8.CompletableFutureTest - task 1
11:39:19.631 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - task 2
11:39:19.635 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - [apply param = task2]
11:39:19.635 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - [accept = [apply param = task2]]
```

同样的，对应的方法有 支持异步的， 和 支持自定义执行线程池的

> 注意：默认情况下，使用的是ForkJoinPool.commonPool


## 3.CompletableFuture组合

> thenCompose / thenCombine / allOf / anyOf

thenCompose      将链式 CompletableFuture 调用拍平，避免了嵌套 get

thenCombine       两个 CompletableFuture 完成后组合调用

CompletableFuture<Void> allOf (CompletableFuture<?>... cfs)        多个 CompletableFuture 组合，所有任务完成，异常返回

> 不会反映过程结果，典型用例为
>
> CompletableFuture.allOf(c1, c2, c3).join();

CompletableFuture<Object> anyOf (CompletableFuture<?>... cfs)   多个 CompletableFuture 组合，任意 1 个任务完成，异常返回

```java
    CompletableFuture<String> foo1(String id) {
        return CompletableFuture.supplyAsync(() -> "id = " + id );
    }

    CompletableFuture<String> foo2(String idKey) {
        return CompletableFuture.supplyAsync(() -> String.format("desc = [%s]", idKey));
    }

    @Test
    void test3() throws ExecutionException, InterruptedException {
        CompletableFuture<String> f1 = foo1("001");
        CompletableFuture<CompletableFuture<String>> f2 = f1.thenApplyAsync((key) -> {
            return foo2(key);
        });
        log.warn("end = " + f2.get().get());
        log.info("end2 = " + f1.thenCompose((key) -> foo2(key)).get());
    }


15:19:18.838 [main] WARN java8.CompletableFutureTest - end = desc = [id = 001]
15:19:18.842 [main] INFO java8.CompletableFutureTest - end2 = desc = [id = 001]


    @Test
    void test4() throws ExecutionException, InterruptedException {
        System.out.println(foo1("foo1").thenCombine(foo2("foo2"), (a, b) -> a + "__" + b).get());
    }

id = foo1__desc = [foo2]

```

## 4.异常处理

> exceptionly / handle / whenComplete

获取结果（异常处理）

借用一个总结：

|                     | handle | whenComplete | exceptionly   |
| --------------------- | -------- | -------------- | --------------- |
| 访问成功            | Yes    | Yes          | No            |
| 访问失败            | Yes    | Yes          | Yes           |
| 能从失败中恢复      | Yes    | No           | Yes           |
| 能转换结果从 T 到 U | Yes    | No           | No            |
| 成功时触发          | Yes    | Yes          | No            |
| 失败时触发          | Yes    | Yes          | Yes           |
| 有异步版本          | Yes    | Yes          | Yes (12 版本) |

可以看出，handle 各种情况都支持的很好。一般用它就是了

```java
    @Test
    void test5() throws ExecutionException, InterruptedException {
        CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> {
            log.info("--supplyAsync1--");
            return null;
        }).thenApply((p) -> {
            log.info("--thenApply2--");
            Assert.notNull(p, "参数不允许为空");
            return Integer.valueOf(2);
        }).handle( (res, ex) -> {
            if(ex != null) {
                log.error(ex.getMessage());
            }else {
                log.info("--success--");
            }
            return res;
        });
        log.warn("end " + cf.get());
    }

15:55:29.020 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - --supplyAsync1--
15:55:29.024 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - --thenApply2--
15:55:29.026 [ForkJoinPool.commonPool-worker-9] ERROR java8.CompletableFutureTest - java.lang.IllegalArgumentException: 参数不允许为空
15:55:29.026 [main] WARN java8.CompletableFutureTest - end null
```

> 注意：
>
> handle 中不允许抛出异常；参数为 BiFunction，有返回值
>
> whenComplete 在 get 的时候，如果有异常会继续抛出异常；参数为 BiConsumer，无返回值


## 5.结果获取

* get / join

两者都是 获取结果的，只是 join 更偏向于 Future 组合，其异常会被封装。

* complate

主动计算，如果未完成，则会 Set 传入的值。

```java
    @Test
    void test6() throws ExecutionException, InterruptedException {
        CompletableFuture<String> cf = CompletableFuture.runAsync(() -> {
            log.info("--111--");
        }).supplyAsync(() -> {
            log.info("--222--");
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info("--333---");
            return "333";
        });
        log.info(String.valueOf(cf.complete("123")));
        log.warn("end = " + cf.get());
    }

16:23:43.479 [main] INFO java8.CompletableFutureTest - true
16:23:43.479 [ForkJoinPool.commonPool-worker-9] INFO java8.CompletableFutureTest - --111--
16:23:43.484 [main] WARN java8.CompletableFutureTest - end = 123
```

# 二，示例

## 2.1 多任务并行，正确或异常

多任务并发执行，获取正确或错误结果

```java
    @Test
    void multiTaskRunAndReturn() {
        //记录开始时间
        Long start = System.currentTimeMillis();
        //任务
        final List<Integer> taskList = Arrays.asList(1, 2, 3, 4, 5);
        List<String> resultList = new ArrayList<>();
        Map<String, String> errorList = new HashMap<>();
        log.warn("start");
        Stream<CompletableFuture<String>> completableFutureStream = taskList.stream()
                .map(num -> {
                            return CompletableFuture
                                    .supplyAsync(() -> doubleInteger(num))
                                    .handle((res, th) -> {
                                        if (th == null) {
                                            log.info("任务" + num + "完成! result=" + res + ", " + LocalTime.now().toString());
                                            resultList.add(res.toString());
                                        } else {
                                            log.error("任务" + num + "异常! e=" + th + ", " +  LocalTime.now().toString());
                                            errorList.put(num.toString(), th.getMessage());
                                        }
                                        return "";
                                    });
                        }
                );
        CompletableFuture[] completableFutures = completableFutureStream.toArray(CompletableFuture[]::new);
        CompletableFuture.allOf(completableFutures)
                .whenComplete((v, th) -> {
                    log.warn("所有任务执行完成触发\n resultList=" + resultList + "\n errorList=" + errorList+ "\n耗时=" + (System.currentTimeMillis() - start));
                }).join();
        log.warn("end");

    }


    //根据数字判断线程休眠的时间
    public static Integer doubleInteger(Integer i) {
        try {
            log.warn("任务" + i + " 开始 ...");
            TimeUnit.SECONDS.sleep(i);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (i ==3) {
            throw new IllegalArgumentException("");
        }
        return 2 * i;
    }
```


## 2.2 任务组，组内并行，组外依赖

两组任务可分组执行，但任务组2依赖任务组1结果。 实现方式很多，但是要实现优雅，易扩展可不容易：

```java

/**
 * 将两队任务组合
 *
 * task1组完成后，再完成task2，组装成1个 CompletableFuture
 */
@Slf4j
public class CompletableFutureTest2 {

    List<String> task1ResultList = new ArrayList<>();

    CompletableFuture<String> runTask1(String name) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("taskName={}",name);
            return name + "_Result_" + Thread.currentThread().getId();
        }).handle((res, e) -> {
            // 不侵入任务本身 收集结果
            if (e == null) {
                task1ResultList.add(res);
            }else {
                task1ResultList.add(e.getMessage());
            }
            return res;
        });
    }

    CompletableFuture<String> runTask2(CompletableFuture preTask, String name) {
        return preTask.thenRunAsync(() -> {
            log.info("taskName={}",name);
            // 利用task1的结果 process
            log.warn("preResult=" + task1ResultList.stream().collect(Collectors.joining(",")));
        });
    }

    @Test
    void test3() {
        // 1
        List<CompletableFuture> list1 = new ArrayList<>();
        for (int i=0; i<3; i++) {
            CompletableFuture cf1 = runTask1("task1");
            list1.add(cf1);
        }
        CompletableFuture[] arr1 = list1.toArray(new CompletableFuture[list1.size()]);
        CompletableFuture<Void> f11 = CompletableFuture.allOf(arr1);


        // 2
        List<CompletableFuture> list2 = new ArrayList<>();
        for(int i=0;i<7;i++) {
            CompletableFuture cf2 = runTask2( f11 , "task2");
            list1.add(cf2);
        }
        CompletableFuture[] arr2 = list1.toArray(new CompletableFuture[list2.size()]);
        CompletableFuture.allOf(arr2).join();
    }


}

```


## 2.3 泡茶

设想现在有个任务：泡茶。需要步骤大概可以分为烧水和准备茶具、茶叶，最终泡茶。

在这个任务重，烧水、准备茶具茶叶是分开的两个支线，他们是无无关联，可以并行的。示意如下：

1. 任务支线1  洗水壶 --\> 烧开水
2. 任务支线2  洗茶壶 --\> 洗茶杯 --\> 取茶叶
3. 任务汇总   泡茶

使用`CompletableFuture` 可以实现如下：

```java

/**
 * 任务支线1  洗水壶 --> 烧开水
 * 任务支线2  洗茶壶 --> 洗茶杯 --> 取茶叶
 * 任务汇总   泡茶
 */
@Slf4j
public class TeaMakingExample {

    public static void randomSleep() {
        try {
            TimeUnit.SECONDS.sleep(new Random().nextInt(5));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    final static List<String> resList = new ArrayList<>();

    static CompletableFuture buildSupplier(Supplier<String> function) {
        return CompletableFuture.supplyAsync(function).handle((res, ex) -> {
            resList.add(res);
            return res;
        });
    }

    public static void main(String[] args) {

        CompletableFuture<String> task1 = CompletableFuture.supplyAsync(() -> {
            log.info("任务11：洗水壶");
            randomSleep();
            return "干净的水壶";
        }).thenApplyAsync( x -> {
            log.info("任务12：烧开水");
//            log.info("参数=" + x);
            randomSleep();
            return "开水";
        }).handle((res, ex) -> {
            resList.add(res);
            return res;
        });

        CompletableFuture<String> task21 = buildSupplier(() -> {
            log.info("任务21: 洗茶壶");
            randomSleep();
            return "干净的茶壶";
        });

        CompletableFuture<String> task22 = buildSupplier(() -> {
            log.info("任务22: 洗茶杯");
            randomSleep();
            return "干净的茶杯";
        });

        CompletableFuture<String> task23 = buildSupplier(() -> {
            log.info("任务23: 取茶叶");
            randomSleep();
            return "干净的茶叶";
        });

        // 组装任务
        CompletableFuture<Void> allTasks = CompletableFuture.allOf(task1, task21, task22, task23).thenRunAsync( () -> {
            log.warn("---所有材料准备齐全---");
            log.warn("[{}]", resList.stream().collect(Collectors.joining(",")));
            log.warn("===最终步骤：：泡茶===");
        });

        // 阻塞直到所有 CompletableFuture 完成
        allTasks.join();

    }
}
```
