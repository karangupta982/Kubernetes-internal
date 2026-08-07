These three failures happen **before your application even starts**.

That's why they're extremely important.

A beginner usually thinks:

> "My application isn't working."

An experienced Kubernetes engineer asks:

> **"Did my application even get a chance to start?"**

These failures teach exactly that.

---

# First understand the container lifecycle

Before learning these failures, understand where they happen.

A Pod doesn't magically become Running.

It goes through multiple stages.

```text
Deployment Created
        │
        ▼
Scheduler selects a Node
        │
        ▼
Kubelet receives Pod
        │
        ▼
Pull Image
        │
        ▼
Create Container
        │
        ▼
Inject Secrets
Inject ConfigMaps
Mount Volumes
        │
        ▼
Start Container Process
        │
        ▼
Run Startup Probe
        │
        ▼
Run Readiness Probe
        │
        ▼
Pod Ready
```

Now notice something.

Different failures happen at different stages.

| Stage            | Possible Failure                 |
| ---------------- | -------------------------------- |
| Pull Image       | ErrImagePull, ImagePullBackOff   |
| Create Container | CreateContainerConfigError       |
| Start Process    | CrashLoopBackOff                 |
| Running          | OOMKilled                        |
| Health Check     | Readiness/Liveness Probe Failure |

Knowing **where** a failure occurs immediately narrows your investigation.

---

# 1. ErrImagePull

This is the **first failure** that can happen.

---

## What is an Image?

Think of Docker Image as

```text
Operating System

+

Application

+

Libraries

+

Runtime
```

Example

```yaml
image: nginx:1.27
```

or

```yaml
image: mycompany/payment:v2
```

---

When Kubernetes schedules your Pod,

the Kubelet asks the container runtime

```text
"Please download this image."
```

If downloading fails,

you get

```text
ErrImagePull
```

Notice

Your application has **not started yet.**

---

## Common reasons

### Wrong image name

Example

```yaml
image: ngnix:latest
```

Notice typo.

Should be

```yaml
image: nginx:latest
```

---

### Wrong tag

```yaml
image: nginx:v999
```

No such version exists.

Registry returns

```text
404
```

---

### Private Registry

Example

```yaml
image: mycompany/backend:v1
```

Registry requires authentication.

No credentials.

Download denied.

---

### Network issue

Node cannot reach Docker Hub.

---

### Registry unavailable

Docker Hub outage.

AWS ECR outage.

GCR outage.

---

## What engineer sees

```bash
kubectl get pods
```

Output

```text
STATUS

ErrImagePull
```

---

## Investigation

### Step 1

```bash
kubectl describe pod
```

Events

```text
Failed to pull image

rpc error

manifest unknown
```

or

```text
authentication required
```

or

```text
image not found
```

Notice

Events are much more useful here than logs.

---

Why?

Because

Container never started.

No logs exist.

---

## Your operator

Collect

```text
Pod Status

Waiting Reason

Events
```

Suppose Event says

```text
manifest unknown
```

Diagnosis

```text
Cause

Invalid Image Tag

Confidence

99%

Recommendation

Verify image tag.
```

---

Another example

```text
authentication required
```

Recommendation

```text
Check imagePullSecrets
```

---

## Important lesson

Not every failure needs logs.

Some failures happen before logs exist.

---

# 2. ImagePullBackOff

This confuses almost everyone.

---

## Biggest misconception

People think

ErrImagePull

and

ImagePullBackOff

are different problems.

They aren't.

One comes after the other.

---

Timeline

```text
Pull image

↓

Failed

↓

ErrImagePull

↓

Retry

↓

Failed

↓

Retry

↓

Failed

↓

Retry

↓

ImagePullBackOff
```

Exactly like CrashLoopBackOff.

---

ImagePullBackOff means

> Kubernetes has repeatedly failed to download the image and is now waiting before trying again.

Again

BackOff means

Exponential delay.

---

## Example

```text
Try

↓

Fail

↓

10 sec

↓

Try

↓

Fail

↓

20 sec

↓

Try

↓

Fail

↓

40 sec

↓

Try
```

---

Engineer investigates exactly the same way

```bash
kubectl describe pod
```

Events

```text
Failed to pull image

Back-off pulling image
```

---

Operator

should not diagnose

```text
ImagePullBackOff
```

Instead

look deeper

Events

↓

authentication

↓

DNS

↓

manifest unknown

↓

connection timeout

and diagnose

those.

---

Notice the pattern?

Exactly like

CrashLoopBackOff.

---

# Difference

| Status           | Meaning                                   |
| ---------------- | ----------------------------------------- |
| ErrImagePull     | Pull failed first time                    |
| ImagePullBackOff | Kubernetes keeps retrying and backing off |

Root cause

Same.

---

# 3. CreateContainerConfigError

This one teaches a completely different concept.

The image downloaded successfully.

Container has **still not started.**

---

Why?

Because Kubernetes needs to prepare container configuration.

Imagine Deployment

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: payment-secret
      key: password
```

Question

Does Secret exist?

Suppose

No.

Can Kubernetes start container?

No.

Because environment variable cannot be injected.

Result

```text
CreateContainerConfigError
```

---

Notice

Application never started.

Again

No logs.

---

Other reasons

Missing ConfigMap

```yaml
configMapRef:
  name: app-config
```

ConfigMap deleted.

---

Missing Secret

```yaml
secretKeyRef
```

Secret deleted.

---

Invalid Volume Configuration

---

Bad environment variable reference

---

## Engineer investigation

```bash
kubectl describe pod
```

Events

```text
Secret "payment-secret" not found
```

Done.

Problem found.

---

Another

```text
ConfigMap "app-config" not found
```

---

No need

for

```bash
kubectl logs
```

Because application never started.

---

## Your operator

Collect

Events

Pod Status

Waiting Reason

Volumes

Secrets

ConfigMaps

---

Diagnosis

```text
Cause

Missing Secret

Confidence

100%

Recommendation

Create Secret

payment-secret
```

---

Another

```text
ConfigMap missing
```

Recommendation

```text
Apply ConfigMap before Deployment.
```

---

# Let's compare these three

| Failure                    | Did Image Download? | Did Container Start? | Logs Available? | Main Investigation |
| -------------------------- | ------------------- | -------------------- | --------------- | ------------------ |
| ErrImagePull               | ❌ No                | ❌ No                 | ❌ No            | Events             |
| ImagePullBackOff           | ❌ No                | ❌ No                 | ❌ No            | Events             |
| CreateContainerConfigError | ✅ Yes               | ❌ No                 | ❌ No            | Events + Spec      |

Notice a pattern?

For these failures,

**Events are king.**

Logs don't exist because the application never executed.

---

# A better architecture for your operator

Originally, we thought about collecting everything (logs, events, metrics) for every failure. That would work, but it's inefficient.

A smarter operator would follow an investigation strategy similar to an experienced engineer:

```text
Pod Failure

↓

What phase failed?

───────────────

Image Pull?
        │
        ▼
Read Events

───────────────

Container Creation?
        │
        ▼
Read Events + Pod Spec

───────────────

Container Started?
        │
        ▼
Read Logs + Exit Code

───────────────

Running?
        │
        ▼
Read Metrics + Node State
```

Now your operator isn't blindly collecting everything. It first determines **where in the Pod lifecycle the failure occurred**, then gathers only the evidence relevant to that phase.

This is more scalable, faster, and mirrors how senior Kubernetes engineers investigate incidents. It's also a much stronger design to explain during interviews because it shows you're modeling the Kubernetes lifecycle rather than writing a collection of unrelated rules.
