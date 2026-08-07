# 1. OOMKilled

This is probably the single most common production issue.

---

## What exactly is OOMKilled?

OOM = **Out Of Memory**

Imagine your container has this limit:

```yaml
resources:
  limits:
    memory: 512Mi
```

Your application starts consuming memory

```
100Mi

↓

200Mi

↓

350Mi

↓

490Mi

↓

512Mi

↓

530Mi
```

As soon as it tries to exceed **512Mi**, Linux's **OOM Killer** terminates the process.

Notice something important.

**Kubernetes does NOT kill the process.**

Linux Kernel does.

Kubernetes only observes:

> "Container exited because it was OOMKilled."

This distinction is important.

---

## What happens internally?

Let's assume this Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 1
```

Container starts.

Memory keeps increasing.

Eventually

```
Memory > Limit
```

Linux Kernel

↓

Kills process

↓

Container exits

↓

Kubelet notices

↓

Container status becomes

```
OOMKilled
```

↓

Restart Policy = Always

↓

Container restarted

↓

Memory grows again

↓

Killed again

↓

Restart loop

---

## What does engineer first notice?

Usually

```bash
kubectl get pods
```

Output

```
NAME

payment-api

STATUS

Running

RESTARTS

17
```

Interesting.

Pod is Running.

But restarts keep increasing.

Or

```
CrashLoopBackOff
```

---

## How does engineer investigate?

### Step 1

```bash
kubectl describe pod payment-api
```

Important section

```
Last State:

Terminated

Reason:

OOMKilled

Exit Code:

137
```

Immediately experienced engineers think

> Memory problem.

---

### Step 2

Verify memory graph

Usually in

* Grafana
* Prometheus
* Datadog

Graph

```
100Mi

↓

200Mi

↓

400Mi

↓

510Mi

↓

512Mi

↓

Killed
```

Root cause confirmed.

---

### Step 3

Check resource limits

```bash
kubectl describe pod
```

or

```bash
kubectl get pod -o yaml
```

Find

```yaml
resources:

requests:

memory: 256Mi

limits:

memory: 512Mi
```

Maybe limit too small.

---

### Step 4

Read logs

```
Loading records...

Processing data...

Killed
```

Sometimes logs don't even mention memory.

Because kernel killed the process immediately.

---

## How would your operator diagnose it?

Evidence collected

```
LastState.Reason = OOMKilled

ExitCode = 137

RestartCount = 15

MemoryLimit = 512Mi
```

Diagnosis

```
Cause

Out Of Memory

Confidence

99%

Recommendation

Increase memory limit
OR
Investigate memory leak
```

Notice

The operator shouldn't say

> "Memory leak."

Maybe application legitimately needs more memory.

Only developer can know.

---

## What concepts you learn

* Linux OOM Killer
* Resource Requests
* Resource Limits
* Kubelet
* Container Status
* Exit Codes
* Restart Policy

One failure teaches six Kubernetes concepts.

---

# 2. CrashLoopBackOff

This one confuses almost everyone.

---

## Biggest misconception

People think

> CrashLoopBackOff is the error.

It isn't.

It is Kubernetes saying

> "I tried restarting this container multiple times. It keeps crashing. I'm slowing down before trying again."

It is a **state**, not a root cause.

---

## Example

Application

```go
func main() {
    panic("database connection failed")
}
```

Container

↓

Starts

↓

Panic

↓

Exits

↓

Restart

↓

Panic

↓

Restart

↓

Panic

↓

Restart

Eventually Kubernetes says

```
CrashLoopBackOff
```

Notice

The real problem is

```
panic
```

CrashLoopBackOff is only the consequence.

---

## Why "BackOff"?

Suppose Kubernetes restarted immediately forever.

```
Start

↓

Crash

↓

Start

↓

Crash

↓

Start

↓

Crash

↓

1000 times/sec
```

That would waste CPU.

Instead Kubernetes waits.

```
Restart

↓

Wait 10 sec

↓

Restart

↓

Wait 20 sec

↓

Restart

↓

Wait 40 sec

↓

Restart
```

This increasing wait time is called

**Exponential Backoff**.

---

## What engineer sees

```bash
kubectl get pods
```

```
STATUS

CrashLoopBackOff
```

But still doesn't know

WHY.

---

## Investigation

### Step 1

```bash
kubectl describe pod
```

Events

```
Back-off restarting failed container
```

Not useful.

Still don't know cause.

---

### Step 2

Current logs

```bash
kubectl logs payment-api
```

Sometimes

```
Starting...

Connecting...

```

Nothing useful.

---

### Step 3 (Most Important)

```bash
kubectl logs payment-api --previous
```

Because previous container actually crashed.

Output

```
panic

database timeout

stack trace
```

Now engineer knows.

---

### Step 4

Check Exit Code

```
Exit Code

1
```

Generic failure.

Need logs.

---

Sometimes

```
139
```

Segmentation fault.

Sometimes

```
137
```

Actually OOMKilled.

See?

CrashLoopBackOff itself isn't enough.

---

## How would your operator work?

It sees

```
Container State

Waiting

Reason

CrashLoopBackOff
```

Not enough.

Operator automatically collects

* Previous Logs
* Exit Code
* Last State
* Events

Suppose

```
Last State

OOMKilled
```

Then

Operator says

```
CrashLoopBackOff caused by OOMKilled
```

Another example

```
Last State

Exit Code 1

Logs

panic

nil pointer dereference
```

Operator says

```
CrashLoopBackOff

Likely application panic

Recommendation

Inspect application stack trace
```

Notice

CrashLoopBackOff isn't diagnosed directly.

The operator **keeps digging**.

Exactly like an experienced engineer.

---

# Difference between OOMKilled and CrashLoopBackOff

This is extremely important.

Imagine

```
Container exceeded memory
```

Result

```
OOMKilled
```

Kubernetes restarts it.

It exceeds memory again.

Again killed.

Again restarted.

After multiple retries

```
CrashLoopBackOff
```

Now

Question

What's the root cause?

Answer

```
OOMKilled
```

CrashLoopBackOff is only what Kubernetes reports after repeated failures.

Think of it like

```
Person slips

↓

Falls

↓

Hospitalized
```

Root cause

```
Slipped
```

Not

```
Hospitalized
```

---

# How this changes your operator design

Initially, we discussed matching one rule per error. After understanding these two failures, you can already see that's not enough.

Your operator should work as a **multi-stage diagnosis pipeline**:

```
Pod restarted

↓

Collect evidence

↓

Identify immediate state
(CrashLoopBackOff)

↓

Investigate underlying cause
(OOMKilled, panic, probe failure, etc.)

↓

Generate diagnosis

↓

Suggest recommendation
```

This is much closer to how senior SREs debug incidents. They don't stop at the first status they see—they keep asking **"why?"** until they reach the most likely root cause.
