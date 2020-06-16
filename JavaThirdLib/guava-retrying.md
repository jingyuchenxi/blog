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
简单解释一下上述示例代码的执行过程：
为了测试方便，首先我们定义了**RetryRespVo**类，该类作为后续我们主流程Retryer.call()方法的返回值。
我们在main方法中通过 RetryerBuilder.newBuilder方法构造 RetryerBuilder 对象，并根据实际需要设置一系列重试谓词(下文源码分析的时候会详细介绍)，以及一系列停止重试、策略

## 源码分析
### 包目录结构
guava-retrying包目录结构如下：主要包含7个接口类，和

<img src="https://github.com/jingyuchenxi/blog/blob/master/resource/hie.png?raw=true" width="230px" height="250px"/>

### 核心类
* Retryer

* RetryerBuilder

### 功能接口
* Attempt
    
* AttemptTimeLimiter

* BlockStrategy

* StopStrategy

* WaitStrategy

* RetryListener

### 功能实现类
* AttemptTimeLimiters

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




