
***

# **Dealing with Rejection in Distributed Systems – Summary**

***

## **What is Backpressure?**

Backpressure is a mechanism that prevents a distributed system from being overwhelmed by incoming requests. Instead of blindly accepting all work, the system slows down or rejects requests when resources are saturated.  
**Goal:** Maintain stability while using resources efficiently.

***

## **What Happens Without Backpressure?**

*   **Resource exhaustion:** Memory and CPU spike → out-of-memory crashes.
*   **Cascading failures:** Overloaded nodes fail, causing retries and amplifying load.
*   **Unbounded queues:** Latency skyrockets, clients time out, and system collapses.

***

## **General Principles of Backpressure**

### **Good Practices**

*   Apply throttling **early** (before expensive work like reading payloads).
*   Use **state-based signals** (e.g., in-flight bytes, queue depth) that correlate with overload.
*   Leverage **TCP backpressure** by pausing reads so clients naturally slow down.
*   Adapt dynamically based on resource health (CPU, latency, backlog).

### **Bad Practices**

*   Reject requests **too late** (after allocating buffers and spawning goroutines).
*   Rely on **static rate limits** (brittle, workload-sensitive).
*   Ignore upstream propagation (load shedding without slowing clients).

***

## **WarpStream Agent Example**

WarpStream faced overload scenarios where handler-level checks were too late. They improved by throttling at the **connection read loop** and pausing `Accept()` under severe overload.

### **Pseudo-Code**

```text
loop forever:
    if system severely overloaded:
        pause accepting new connections
        sleep until healthy
    else:
        accept connection
        spawn connection handler

connection handler:
    loop forever:
        if ShouldThrottle():
            sleep briefly
            continue
        read next request from socket
        spawn request handler

request handler:
    if overloaded AFTER read:
        reject request (load shedding)
    else:
        process request normally
```

**Key points:**

*   Severe overload → stop accepting new connections.
*   Moderate overload → pause reading (TCP backpressure).
*   Healthy → process requests normally.
*   Late detection → reject request after read (fallback).

***

## **Insights from Other Systems**

When request cost varies widely, simple metrics like “messages per second” fail. Industry solutions:

*   **Netflix Adaptive Concurrency Control**
    *   Adjust concurrency based on latency trends (Little’s Law).
    *   Inspired by TCP congestion control.

*   **Envoy Adaptive Concurrency Filter**
    *   Compares current latency to ideal latency (minRTT) and adjusts dynamically.

*   **Google SRE Guidance**
    *   Avoid QPS-based limits; throttle based on CPU/memory headroom.
    *   Prefer degraded responses over hard failures.

**Signals to measure for dynamic control:**

*   Latency (proxy for saturation).
*   CPU/memory utilization.
*   Queue depth/backlog.
*   Success/error rates.

***

### **Key Takeaways**

*   Backpressure ≠ just rejecting requests; it’s about **where and how** you apply it.
*   Best signals: **things that pile up** (in-flight bytes/files).
*   Apply throttling **early** and propagate pressure upstream.
*   Dynamic rate limiting is complex; adaptive concurrency based on latency and resource signals is more robust.

***

### **References**

*  WarpStream Blog – Dealing with Rejection in Distributed Systems : https://www.warpstream.com/blog/dealing-with-rejection-in-distributed-systems
*  Google SRE Book – Overload Management : https://sre.google/sre-book/handling-overload/
*  Netflix Tech Blog – Adaptive Concurrency Limits
