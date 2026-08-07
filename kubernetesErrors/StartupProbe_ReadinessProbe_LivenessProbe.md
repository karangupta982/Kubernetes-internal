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

---

# 2. Readiness Probe Failure

This one is extremely important.

---

## Biggest misconception

People think

Readiness failure means

Application crashed.

Wrong.

Application is still running.

Kubernetes simply stops sending traffic.

---

Imagine

Three Pods

```text
Pod A

Running

Ready

✓

Pod B

Running

Ready

✓

Pod C

Running

Not Ready

✗
```

Service

```text
Client

↓

Service

↓

Pod A

↓

Pod B
```

Pod C receives

Zero traffic.

---

Notice

Pod C still exists.

Still consumes CPU.

Still consumes Memory.

It just doesn't receive requests.

---

## Why?

Suppose application depends on

Database.

Database goes down.

Application decides

```text
"I'm alive.

But I cannot serve requests."
```

Health endpoint returns

```text
HTTP 503
```

Readiness probe fails.

Kubernetes removes Pod from Service endpoints.

Beautiful.

No restart needed.

---

## Engineer investigation

```bash
kubectl describe pod
```

Events

```text
Readiness probe failed

HTTP probe failed

503
```

Now engineer asks

Why is /health returning 503?

Maybe

* database down
* Redis unavailable
* Kafka unavailable
* cache loading

Now investigation shifts to dependencies.

---

## Operator Diagnosis

Evidence

```text
Pod Running

Container Running

Readiness Failed

HTTP 503
```

Diagnosis

```text
Application unhealthy

Traffic removed

Investigate dependencies
```

Notice

Operator should NOT recommend restart.

Restart won't fix database outage.

---

# 3. Liveness Probe Failure

This one actually causes restarts.

---

## Purpose

Imagine

Your Go application

has

Deadlock.

Example

```go
mutex.Lock()

...

mutex.Lock()
```

Application

doesn't crash.

CPU usage low.

Memory normal.

But

Every request hangs forever.

Kubernetes thinks

Container still running.

Actually

Application is dead.

---

Liveness Probe solves this.

Every few seconds

Kubernetes asks

```text
GET /health
```

Application doesn't respond.

Timeout.

Probe fails.

After repeated failures

Kubelet restarts container.

---

## Timeline

```text
Application Running

↓

Deadlock

↓

Liveness Probe

↓

Timeout

↓

Failure

↓

Failure

↓

Failure

↓

Container Restarted
```

---

## Engineer investigation

```bash
kubectl describe pod
```

Events

```text
Liveness probe failed

Timeout

Container restarted
```

Now engineer checks

* application logs
* thread dump
* CPU usage
* deadlock
* infinite loop

---

## Operator Diagnosis

Evidence

```text
Restart Count Increasing

Liveness Probe Failed

Exit Code 143
```

Diagnosis

```text
Container restarted due to failed liveness checks

Investigate application responsiveness
```

Notice

Operator cannot know

Whether

Deadlock

Infinite loop

Slow database

GC pause

caused timeout.

It should not guess.

---

# Difference between all three

This table is worth memorizing because interviewers love this question.

| Probe            | Is Container Running? | Receives Traffic? | Restarted? |
| ---------------- | --------------------- | ----------------- | ---------- |
| Startup Failed   | ❌ Usually No          | ❌ No              | ✅ Yes      |
| Readiness Failed | ✅ Yes                 | ❌ No              | ❌ No       |
| Liveness Failed  | ✅ Yes (initially)     | Usually Yes       | ✅ Yes      |

---

# Real production example

Imagine

Payment Service

```text
Starting

↓

Loading configuration

↓

Connecting to PostgreSQL

↓

Loading 2GB fraud model

↓

Listening

↓

Serving requests
```

Correct probe behavior

```text
Startup Probe

↓

Waits

↓

Application starts

↓

Readiness Probe

↓

Database available

↓

Traffic begins

↓

Liveness Probe

↓

Runs forever
```

Now imagine PostgreSQL goes down.

Startup?

Already completed.

No effect.

Liveness?

Application still responds.

Passes.

Readiness?

Returns 503.

Fails.

Kubernetes removes Pod from Service.

Exactly what you want.

No unnecessary restart.

---

# How your operator should investigate probe failures

Instead of immediately saying:

> "Readiness probe failed."

It should ask a series of questions, almost like a detective:

```text
Probe Failure

↓

Which probe?

↓

Startup
Readiness
Liveness

↓

HTTP?
TCP?
Exec?

↓

Status Code?
Timeout?
Connection Refused?

↓

Is the container running?

↓

Was it restarted?

↓

What happened just before the failure?
```

That investigation tree is much more useful than a single rule like:

```go
if readinessFailed {
    ...
}
```

because the **same readiness failure** could be caused by:

* database unavailable
* Redis unavailable
* slow startup
* bad `/health` implementation
* network policy blocking traffic
* DNS resolution failure

Your operator should collect enough evidence to narrow it down without pretending it knows the exact application bug.
