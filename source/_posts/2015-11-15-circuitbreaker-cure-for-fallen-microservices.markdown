---
layout: post
title: "CircuitBreaker - cure for fallen microservices"
date: 2015-11-15 19:45:09 +0100
comments: true
description: CircuitBreaker - cure for fallen microservices
keywords: javaee, java, patterns, microservices, hystrix, circuitbreaker
categories: [java, patterns, microservices]
---

-> {% img center-displayed /images/circuitbreaker.jpeg 'image' 'images' %} <-

 In time of microservices, it's relevant how we think about securing remote communication between multiple applications and how we react on network/application failure as a client. If our application depends on many external services and communicates with them remotely, it should be protected in case of network problems or unavailability of the application. One of the basic solutions is to use CircuitBreaker pattern, you can find more detailed description on Martin Fowler's website [http://martinfowler.com/bliki/CircuitBreaker.html](http://martinfowler.com/bliki/CircuitBreaker.html).

<!-- more -->

Main task of CircuitBreaker is to stop communication to some service when failures occure in that service. Default state of circuit is CLOSED state. It means that everything works. When failure rate reach a certain level, like in 10 second window there is minimum 20 requests to a remote service via HTTP and 50% of those requests ends with status code 404/503 or timeout, then CircuitBreaker goes into OPEN state and stops further requests. During the OPEN state all of the invocations to failed service receive fallback result that we need to implement. In case of remote service that provides endpoint that returns list of some entities, the fallback code should provide just empty list. While our circuit is OPEN periodically once at a defined time the state changes to HALF-OPEN state. In HALF-OPEN state only one request is passed to failed service. If the request returns successful response, then it means that failed service is working again and circuit state is back to CLOSED. If response is still incorrect, then circuit stay in OPEN state.

CircuitBreaker advantages :

*   fallback protects user from failure
*   failing fast instead of timeouts and control over network latency 
*   prevents cascading failures in distributed environment
*   failures isolation

Implementing simple CircuitBreaker
---------------------

I want to share with you my implementation of CircuitBreaker which is based on Hystrix library from Netflix. It's not full CircuitBreaker due to missing HALF-OPEN state. Also my implementation depends on Hystrix implementation of Sliding Window algorithm for measuring circuit health. Maybe it's little overhead to use the whole library from Netflix, but this is just example. If i find some spare time, I will implement some simpler solution to get rid of Hystrix dependence and decrease coupling. But this is simple, thread-safe and technology agnostic example which you can use in standard JAVA applications and also in JAVA EE applications.

Circuit core
---------------------

{% codeblock lang:java CircuitBreaker %}
public class CircuitBreaker {

    private static final ScheduledExecutorService closeCircuitBreakerScheduledExecutor = Executors.newSingleThreadScheduledExecutor();
    private final CircuitHealth circuitBreakerHealth;
    private final AtomicBoolean available = new AtomicBoolean(true);
    private final long closeCircuitDelay;
    private final TimeUnit closeCircuitDelayTimeUnit;

    public CircuitBreaker(long closeCircuitDelay, TimeUnit closeCircuitDelayTimeUnit, HealthConfiguration config) {
        this.closeCircuitDelay = closeCircuitDelay;
        this.closeCircuitDelayTimeUnit = closeCircuitDelayTimeUnit;
        this.circuitBreakerHealth = new HystrixHealth(config);
    }

    public void doCall(Callable<Void> callable) throws CircuitBreakerCallableFailure {
        if (!available.get()) {
            throw new CircuitBreakerCallableFailure("CircuitBreaker is OPEN");
        }
        try {
            callable.call();
            circuitBreakerHealth.markSuccess();
        } catch (Throwable ex) {
            circuitBreakerHealth.markFailure();
            onError(ex);
        }
    }

    private void onError(Throwable ex) throws CircuitBreakerCallableFailure {
        if (circuitBreakerHealth.shouldOpen() && available.compareAndSet(true, false)) {
            closeCircuitBreakerScheduledExecutor.schedule(new Callable<Void>() {
                public Void call() throws Exception {
                    available.set(true);
                    circuitBreakerHealth.resetCounter();
                    return null;
                }
            }, closeCircuitDelay, closeCircuitDelayTimeUnit);
        }

        throw new CircuitBreakerCallableFailure(ex);
    }
}
{% endcodeblock %}

In constructor we are passing closeCircuitDelay and closeCircuitDelayTimeUnit variables defining after a certain time circuit should be once again closed after opening. Also we pass HealthConfiguration instance that stores configuration for circuitBreakerHealth instance. We can configure conditions like :

*   errorThresholdPercentage - the percentage threshold of errors
*   requestVolumeThreshold - the minimum number of request in specified time window
*   rollingStatisticalWindowInMilliseconds -  time span in miliseconds
*   rollingStatisticalWindowBuckets - the number of buckets in the statistical window
*   healthSnapshotIntervalInMilliseconds - snapshot interval time in milliseconds

AtomicBoolean available variable is used as availability flag defining whether circuit is open / closed. We are using AtomicBoolean for thread safety, because we assume that code will be used in a multi-threaded environment. 

The most important method in our CircuitBreaker class is 

{% codeblock lang:java %}
void doCall(Callable<Void> callable) throws CircuitBreakerCallableFailure
{% endcodeblock %}

which is responsible for invoking call() method of Callable instance. When circuit availability is set to false then every invocation of doCall ends up throwing an CircuitBreakerCallableFailure exception. If invocation was successful the markSuccess method of CircuitHealth class will be called. Otherwise markFailure, which is for counting errors, will be called together with onError. In case of closed circuit and poor condition of circuit health onError method will open the circuit. 

{% codeblock lang:java %}
closeCircuitBreakerScheduledExecutor.schedule(new Callable<Void>() {
    public Void call() throws Exception {
        available.set(true);
        circuitBreakerHealth.resetCounter();
        return null;
    }
}, closeCircuitDelay, closeCircuitDelayTimeUnit);
{% endcodeblock %}

Probably the easiest solution for the possibility of retry/close circuit is to run scheduled task just after opening circuit. We can use ScheduledExecutorService with some defined configurable delay - like 5 minutes. The purpose of this task should be reseting state and sucess/failure counters.

Circuit health
---------------------

CircuitHealth interface provides methods for counting success/failures, reseting counters and checking whether the circuit should be open. HystrixHealth implementation is based on Hystrix solution. As counter we are using HystrixRollingNumber instance. We also need to store lastHealthCountsSnapshot, which is the last time when HealthCounts instance was created.

{% codeblock lang:java HystrixHealth %}
public class HystrixHealth implements CircuitHealth {
    private final HealthConfiguration config;
    private volatile HealthCounts healthCountsSnapshot = new HealthCounts(0L, 0L, 0);
    private volatile AtomicLong lastHealthCountsSnapshot = new AtomicLong(System.currentTimeMillis());
    private final HystrixRollingNumber counter;

    public HystrixHealth(HealthConfiguration config) {
        this.counter = new HystrixRollingNumber(config::getRollingStatisticalWindowInMilliseconds, config::getRollingStatisticalWindowBuckets);
        this.config = config;
    }

    @Override
    public void resetCounter() {
        this.counter.reset();
        this.lastHealthCountsSnapshot.set(System.currentTimeMillis());
        this.healthCountsSnapshot = new HealthCounts(0L, 0L, 0);
    }

    @Override
    public void markSuccess() {
        this.counter.increment(HystrixRollingNumberEvent.SUCCESS);
    }

    @Override
    public void markFailure() {
        this.counter.increment(HystrixRollingNumberEvent.FAILURE);
    }

    @Override
    public boolean shouldOpen() {
        HealthCounts health = getHealthCounts();
        if (health.getTotalRequests() < config.getRequestVolumeThreshold()) {
            return false;
        }
        return health.getErrorPercentage() >= config.getErrorThresholdPercentage();
    }

    public HealthCounts getHealthCounts() {
        long lastTime = this.lastHealthCountsSnapshot.get();
        long currentTime = System.currentTimeMillis();
        if ((currentTime - lastTime >= config.getHealthSnapshotIntervalInMilliseconds()
                || this.healthCountsSnapshot == null) && this.lastHealthCountsSnapshot.compareAndSet(lastTime, currentTime)) {
            long success = this.counter.getRollingSum(HystrixRollingNumberEvent.SUCCESS);
            long failure = this.counter.getRollingSum(HystrixRollingNumberEvent.FAILURE);
            long totalCount = failure + success;
            int errorPercentage = 0;
            if (totalCount > 0L) {
                errorPercentage = (int) ((double) failure / (double) totalCount * 100.0D);
            }

            this.healthCountsSnapshot = new HealthCounts(totalCount, failure, errorPercentage);
        }

        return this.healthCountsSnapshot;
    }
}
{% endcodeblock %}

The most important part of CircuitHealth is shouldOpen method. Circuit should open when total requests count exceeds predefined boundary in certain, specified in configuration time frames and error rate also exceeds the threshold.

{% codeblock lang:java %}
public boolean shouldOpen() {
    HealthCounts health = getHealthCounts();
    if (health.getTotalRequests() < config.getRequestVolumeThreshold()) {
        return false;
    }
    return health.getErrorPercentage() >= config.getErrorThresholdPercentage();
}
{% endcodeblock %}

Immutable class HealthCounts stores state of circuit health in the time of snapshot. 

{% codeblock lang:java HealthCounts %}
public class HealthCounts {
    private final long totalCount;
    private final long errorCount;
    private final int errorPercentage;

    public HealthCounts(long total, long error, int errorPercentage) {
        this.totalCount = total;
        this.errorCount = error;
        this.errorPercentage = errorPercentage;
    }

    public long getTotalRequests() {
        return this.totalCount;
    }

    public long getErrorCount() {
        return this.errorCount;
    }

    public int getErrorPercentage() {
        return this.errorPercentage;
    }
}
{% endcodeblock %}

[Full application source code](http://github.com/nikom1337/simple-circuitbreaker)

Use case
---------------------

{% codeblock lang:java %}
private CircuitBreaker circuitBreaker = new CircuitBreaker(5, TimeUnit.MINUTES);
private MailService mailService = new MailService();

public String doSomething(String destination, Mail mail) throws Exception {
    circuitBreaker.doCall((destination, mail) -> mailService.sendMail(destination, mail));
}
{% endcodeblock %}
