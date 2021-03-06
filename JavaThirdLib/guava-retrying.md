guava-retrying
## 简介
**guava-retrying**作为Guava的一个扩展库，可以为任意函数调用创建可配置的重试策略。常用于RPC远程服务调用过程中。
## 使用
### 依赖引入
```
<!-- https://mvnrepository.com/artifact/com.github.rholder/guava-retrying -->
<dependency>
    <groupId>com.github.rholder</groupId>
    <artifactId>guava-retrying</artifactId>
    <version>2.0.0</version>
</dependency>
```
[guava-retrying依赖查询](https://mvnrepository.com/artifact/com.github.rholder/guava-retrying)
### 示例代码
***RetryRespVo.java***
```java
@Data
@Builder
public class RetryRespVo {
    private String id;
    private String name;
    private String age;
}

```
***GuavaRetry.java***
```java
public class GuavaRetry {

    public static void main(String[] args) {
        Retryer<RetryRespVo> retryer = RetryerBuilder.<RetryRespVo>newBuilder()
                .retryIfResult(respVo -> !"xiaoming".equalsIgnoreCase(respVo.getName()))
                .withAttemptTimeLimiter(AttemptTimeLimiters.noTimeLimit())
                .withStopStrategy(StopStrategies.neverStop())
                .withWaitStrategy(WaitStrategies.fixedWait(1, TimeUnit.SECONDS))
                .withBlockStrategy(BlockStrategies.threadSleepStrategy())
                .withRetryListener(new RetryListener() {
                    @Override
                    public <V> void onRetry(Attempt<V> attempt) {
                        if (attempt.hasException()) {
                            System.out.println("自定义异常:" + attempt.getExceptionCause().getMessage());
                        }
                        if (attempt.hasResult()) {
                            System.out.println("callback method value:" + attempt.getResult().toString());
                        }
                    }
                })
                .build();
        try {
            RetryRespVo retryRespVo = retryer.call(() -> RetryRespVo.builder().id("1").name("xiaoming").age("male").build());
            System.out.println("call method return val:" + retryRespVo);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
解释一下上面的代码：
1. 定义 **RetryRespVo** 类，该类作为后续主流程 **Retryer.call()** 方法的返回值，根据业务需要自己定义即可。
2. 在main方法中通过 **RetryerBuilder.newBuilder** 方法构造 **RetryerBuilder** 对象，并根据实际需要自定义多种重试判断谓词(下文源码分析的时候会详细介绍)，以及每次重试前需要执行的终止策略、延时策略、阻塞策略，也可以通过实现 **RetryListener** 接口，监听每次执行的结果，通过 **RetryerBuilder** 的 **build** 方法创建 **Retryer** 对象。
3. 调用 **Retryer.call** 方法，此时call方法的内部，就会按照上面配置的参数来执行自己的业务逻辑了。

## 源码分析
### 目录结构
guava-retrying包目录结构如下：主要包含2个核心类、7个功能接口以及这7个接口对应的默认实现类。

<img src="https://github.com/jingyuchenxi/blog/blob/master/resource/hie.png?raw=true" width="230px" height="250px"/>

### 相关类和接口

###### 核心类
* Retryer 重试的核心类，类中的 **call** 方法实现了整个重试过程。

* RetryerBuilder 主要用来创建一个 **Retryer** 对象，也就是说 **Retryer** 对象不是通过自身的构造方法来创建的。

###### 功能接口
* Attempt 重试。该接口封装每次 **Retryer.call** 的执行结果，并提供一些判断和获取返回值的方法，目前只有 **ResultAttempt** 和 **ExceptionAttempt** 两个实现类，分别封装 **call** 方法执行成功和执行异常时的返回值。
    
* AttemptTimeLimiter 重试时间控制器。该接口用来控制业务方法的执行过程，如设置业务方法执行的超时时间。当执行时间超过设置的域值，方法就会中断并跑出异常。

* StopStrategy 重试停止接口。

* WaitStrategy 重试等待(延时)接口。该接口定义定义两次重试之间的延时。

* BlockStrategy 重试阻塞接口。该接口通过自定义阻塞策略，在两次重试过程之间执行阻塞等待逻辑。

* RetryListener 用来通知调用方每次重试执行的结果。

###### 功能实现类
* AttemptTimeLimiters 重试时间控制器实现类，主要实现了两类控制器 **NoAttemptTimeLimit**、**FixedAttemptTimeLimit**，**NoAttemptTimeLimit** 没有时间控制逻辑，内部直接调用 **Callable** 的 **call** 方法，**FixedAttemptTimeLimit** 则是通过 **guava** 的 **TimeLimiter** 来控制 **call** 方法的执行时间。

* BlockStrategies

* RetryException

* StopStrategies

* WaitStrategies

```
/**
 * Executes the given callable. If the rejection predicate
 * accepts the attempt, the stop strategy is used to decide if a new attempt
 * must be made. Then the wait strategy is used to decide how much time to sleep
 * and a new attempt is made.
 *
 * @param callable the callable task to be executed
 * @return the computed result of the given callable
 * @throws ExecutionException if the given callable throws an exception, and the
 *                            rejection predicate considers the attempt as successful. The original exception
 *                            is wrapped into an ExecutionException.
 * @throws RetryException     if all the attempts failed before the stop strategy decided
 *                            to abort, or the thread was interrupted. Note that if the thread is interrupted,
 *                            this exception is thrown and the thread's interrupt status is set.
 */
public V call(Callable<V> callable) throws ExecutionException, RetryException {
    long startTime = System.nanoTime();
    for (int attemptNumber = 1; ; attemptNumber++) {
        Attempt<V> attempt;
        try {
            V result = attemptTimeLimiter.call(callable);
            attempt = new ResultAttempt<V>(result, attemptNumber, TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime));
        } catch (Throwable t) {
            attempt = new ExceptionAttempt<V>(t, attemptNumber, TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime));
        }

        for (RetryListener listener : listeners) {
            listener.onRetry(attempt);
        }

        if (!rejectionPredicate.apply(attempt)) {
            return attempt.get();
        }
        if (stopStrategy.shouldStop(attempt)) {
            throw new RetryException(attemptNumber, attempt);
        } else {
            long sleepTime = waitStrategy.computeSleepTime(attempt);
            try {
                blockStrategy.block(sleepTime);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RetryException(attemptNumber, attempt);
            }
        }
    }
}
```




