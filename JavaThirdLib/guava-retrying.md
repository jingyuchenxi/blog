guava-retrying
[TOC]
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

## 源码分析
### 源码结构
![]
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




