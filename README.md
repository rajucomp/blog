
# Debugging a Concurrency Regression in Apache Commons Pool

This note describes the investigation and resolution of a concurrency regression in `GenericObjectPool` related to enforcement of the `maxIdle` configuration under concurrent access.

The regression was introduced as part of the fix for **POOL-425** and later reported as **POOL-426**.

---

## Background

`GenericObjectPool` supports a `maxIdle` configuration parameter that limits the number of idle objects retained by the pool when they are not in use. A negative value indicates that no upper bound is enforced.

The intended invariant is:

> When `maxIdle >= 0`, the number of idle objects retained by the pool must not exceed `maxIdle`.

Maintaining this invariant is important to prevent unbounded growth of idle objects under load.

---

## Original Issue (POOL-425)

[**POOL-425**](https://issues.apache.org/jira/browse/POOL-425) addressed incorrect behavior when `maxIdle` was configured with a negative value. The fix ensured that negative values were treated consistently as “no limit.”

While this change corrected the original issue, it altered the control flow in `addObject` in a way that introduced a concurrency regression.

[Commit](https://github.com/apache/commons-pool/commit/c5d63adfca0d1063cce6a3a53f4be585cf48c4d1)

---

## Regression Description (POOL-426)

In [**POOL-426**](https://issues.apache.org/jira/browse/POOL-426), it was observed that under concurrent calls to `addObject`, the pool could retain more idle objects than permitted by `maxIdle`.



The issue occurs when multiple threads add objects to the pool concurrently. Under these conditions, the pool may temporarily exceed the configured idle limit, violating the expected invariant.

The issue is timing-dependent and does not reliably reproduce without concurrency stress testing. On local testing, the failure was observed intermittently.

[PR for the failing test](https://github.com/apache/commons-pool/pull/451)

---

## Root Cause Analysis

Inspection of the `addObject` implementation showed that the check against `maxIdle` and the subsequent decision to add an object were not effectively synchronized across concurrent threads.

Both the operations need to be atomic i.e. checking the size of the `idleObjects` and adding the pooled object to it. They cannot happen in isolation.

When multiple threads execute this logic concurrently, each may independently observe the size of the `idleObjects` as being below `maxIdle`, leading to multiple objects being added even though only one should be allowed.

The underlying issue is that the invariant is enforced using a non-atomic check-then-act sequence in a concurrent context.

---

## Reproduction via Concurrency Testing

To validate the issue, a concurrency-focused unit test was added that:

- Starts multiple threads simultaneously using a cyclic barrier
- add objects to the pool concurrently
- Asserts that the idle object count does not exceed `maxIdle` after completion

The test reproduces the issue intermittently, consistent with a race condition. Including this test helps ensure that future changes do not reintroduce the regression.

```java
    /*https://issues.apache.org/jira/browse/POOL-426*/
    @Test
    @Timeout(value = 60000, unit = TimeUnit.MILLISECONDS)
    void testAddObjectRespectsMaxIdleLimit() throws Exception {
        final GenericObjectPoolConfig<String> config = new GenericObjectPoolConfig<>();
        config.setJmxEnabled(false);
        try (GenericObjectPool<String> pool = new GenericObjectPool<>(new SimpleFactory(), config)) {
            assertEquals(0, pool.getNumIdle(), "should be zero idle");
            pool.setMaxIdle(1);
            pool.addObject();
            pool.addObject();
            assertEquals(1, pool.getNumIdle(), "should be one idle");

            pool.setMaxIdle(-1);
            pool.addObject();
            pool.addObject();
            pool.addObject();
            assertEquals(4, pool.getNumIdle(), "should be four idle");
        }
    }

    @RepeatedTest(10)
    @Timeout(value = 10, unit = TimeUnit.SECONDS)
    void testAddObjectConcurrentCallsRespectsMaxIdleLimit() throws Exception {
        final int maxIdleLimit = 5;
        final int numThreads = 10;
        final CyclicBarrier barrier = new CyclicBarrier(numThreads);

        withConcurrentCallsRespectMaxIdle(maxIdleLimit, numThreads, pool ->
            getRunnables(numThreads, barrier, pool, (a, b) -> {
            b.await(); // Wait for all threads to be ready
            a.addObject();
        }));
    }

    void withConcurrentCallsRespectMaxIdle(int maxIdleLimit, int numThreads, Function<GenericObjectPool<String>, List<Runnable>> operation) throws Exception {
        final GenericObjectPoolConfig<String> config = new GenericObjectPoolConfig<>();
        config.setJmxEnabled(false);
        try (GenericObjectPool<String> pool = new GenericObjectPool<>(new SimpleFactory(), config)) {
            assertEquals(0, pool.getNumIdle(), "should be zero idle");
            pool.setMaxIdle(maxIdleLimit);
            pool.setMaxTotal(-1);

            final List<Runnable> tasks = operation.apply(pool);

            final ExecutorService executorService = Executors.newFixedThreadPool(numThreads);
            try {
                tasks.forEach(executorService::submit);
                executorService.shutdown();
                assertTrue(executorService.awaitTermination(60, TimeUnit.SECONDS),
                    "Executor did not terminate in time");
            } finally {
                executorService.shutdownNow(); // Ensure cleanup
            }

            assertTrue(pool.getNumIdle() <= maxIdleLimit,
                "Concurrent addObject() calls should not exceed maxIdle limit of " + maxIdleLimit +
                    ", but found " + pool.getNumIdle() + " idle objects");
        }
    }

    @FunctionalInterface
    public interface PoolOperation {
        void execute(GenericObjectPool<String> pool, CyclicBarrier barrier) throws Exception;
    }

    private List<Runnable> getRunnables(final int numThreads,
                                        final CyclicBarrier barrier,
                                        final GenericObjectPool<String> pool,
                                        final PoolOperation operation) {
        List<Runnable> tasks = new ArrayList<>();

        for (int i = 0; i < numThreads; i++) {
            tasks.add(() -> {
                try {
                    operation.execute(pool, barrier);
                } catch (Exception e) {
                    // do nothing
                }
            });
        }
        return tasks;
    }
```

---

## Fix Overview

The fix ensures that enforcement of the `maxIdle` constraint remains correct under concurrent returns by preventing multiple threads from bypassing the invariant simultaneously.

The behavior for `maxIdle < 0` remains unchanged.

In addition to the code change, the concurrency test is included to provide regression coverage.

The fix was accepted upstream and is tracked under [**POOL-426**](https://issues.apache.org/jira/browse/POOL-426).

[PR for the fix](https://github.com/apache/commons-pool/pull/452)

---

## Lessons Learned

- Concurrency regressions can be introduced even by localized fixes that appear correct in isolation.
- Invariants enforced by check-then-act logic require careful synchronization in concurrent code.
- Deterministic concurrency tests, even if probabilistic in failure rate, are valuable for validating correctness under contention.

---

## Closing Remarks

Apache Commons Pool is widely used in latency- and throughput-sensitive systems, making correctness under concurrency particularly important. This issue underscores the value of conservative synchronization and targeted concurrency testing when modifying core pooling logic.
