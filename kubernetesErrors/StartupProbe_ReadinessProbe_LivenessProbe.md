# Before understanding probe failures

You need to answer one question.

**How does Kubernetes know whether your application is healthy?**

It doesn't.

Kubernetes doesn't understand Java, Go, Python, Node.js, etc.

It only knows:

> "I'll ask the application, and the application must tell me whether it's healthy."

This is exactly what probes do.

---

# Three types of probes

Many beginners memorize them. Don't.

Instead, think about the questions Kubernetes is asking.

| Probe           | Kubernetes asks               |
| --------------- | ----------------------------- |
| Startup Probe   | "Have you finished starting?" |
| Readiness Probe | "Can I send traffic to you?"  |
| Liveness Probe  | "Are you still alive?"        |

Those are three completely different questions.

---

# Complete lifecycle

Imagine this Go application.

```text
Container Starts

↓

Loading Configuration

↓

Connecting to Database

↓

Loading Cache

↓

Starting HTTP Server

↓

Listening on :8080

↓

Serving Requests
```

Now let's see where each probe runs.

```text
Container Starts

↓

Startup Probe

↓

Application Started Successfully

↓

Readiness Probe

↓

Service Sends Traffic

↓

Liveness Probe

↓

Application Keeps Running
```

Notice

Readiness and Liveness don't even matter until Startup succeeds.

---

# 1. Startup Probe Failure

This is the easiest one.

---

## What problem does Startup Probe solve?

Imagine your application takes **90 seconds** to start.

Why?

Maybe it

* loads a huge ML model
* warms cache
* performs database migrations
* downloads configuration
* initializes thousands of records

This is normal.

But Kubernetes doesn't know that.

---

Suppose Startup Probe is

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080

  periodSeconds: 10

  failureThreshold: 3
```

What does this mean?

Kubernetes waits

```text
10 sec

↓

10 sec

↓

10 sec
```

Only 30 seconds total.

Application actually needs

90 seconds.

Result

```text
Probe Failed

↓

Container Restarted

↓

Probe Failed

↓

Restarted

↓

CrashLoopBackOff
```

Nothing is wrong with the application.

Probe configuration is wrong.

---

## What engineer sees

```bash
kubectl get pods
```

```text
STATUS

CrashLoopBackOff
```

Looks like application is crashing.

Actually

It never got enough time to start.

---

## Investigation

```bash
kubectl describe pod
```

Events

```text
Startup probe failed

connection refused
```

Now engineer checks

```yaml
startupProbe:
```

and realizes

Application startup time

```text
90 sec
```

Probe timeout

```text
30 sec
```

Problem solved.

---

## Operator Diagnosis

Evidence

```text
Events

Startup probe failed

Restart Count

High

Application eventually starts when timeout increased
```

Diagnosis

```text
Cause

Startup probe timeout too aggressive

Confidence

95%

Recommendation

Increase failureThreshold
or
Increase periodSeconds
```
