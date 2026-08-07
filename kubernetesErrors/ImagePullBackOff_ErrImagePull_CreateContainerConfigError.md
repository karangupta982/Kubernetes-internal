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
