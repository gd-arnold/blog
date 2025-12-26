---
title: "How I think about Kubernetes"
date: "2025-12-26"
summary: "More than a container orchastrator"
description: "More than a container orchastrator"
autonumber: false
math: true
hideBackToTop: true
hidePagination: true
---

Kubernetes is most often described as a container orchestration tool, but after you’ve spent some time with it, that mental model doesn’t always feel like the most useful way to think about what’s happening.

In many cases for me Kubernetes feels less like an orchestrator and more like a platform where you declare the **desired state** of your infrastructure and let the system **continuously** work to match your **intent**.

That’s why I like to think of Kubernetes as a **runtime for declarative infrastructure with a type system**. Let me break it down.

### Type System

Consider this manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
```

If you’ve spent a bit of time with k8s you probably know what this is. It’s a resource of type **Pod** - the smallest unit Kubernetes schedules and runs. This one runs an nginx container and declares that the container listens on port 80.

**Pods** don’t usually live on their own. In real systems you wrap them in higher-level resources: a **ReplicaSet** to keep **N** copies around, a **Deployment** to manage rollouts on top of that, a **Service** to give them a stable network identity and an **Ingress** to route traffic between **Services**.

You see how Kubernetes comes with a whole vocabulary of resource kinds that are responsible for different parts of your infrastructure and the more you think about it, the more that vocabulary starts to behave a lot like a type system.

A **Pod** isn’t “just a container” and a **Deployment** isn’t “just a script that creates **Pods**”. Each one is a type with a strict definition and specific semantics - what it means, what fields are valid, how it’s allowed to change and which other types it composes with. 

Just like a standard type system in a programming language, except instead of types like **int**, **char**, and **string**, you get **Pod**, **Deployment** and **Service**.

And just like in a programming language, you can define your own types too. That’s what CRDs are - new resource kinds that extend the API, with operators that implement what those new types mean.

Traditionally we think about workloads as processes or containers. Kubernetes still runs those, but it abstracts them away by giving your infrastructure a **type system**.

### Runtime

The manifest I showed before is just a file in which you declare a desired state. Now let's examine what happens when you apply it to your cluster.

```bash
kubectl apply -f pod.yaml
```

Kubernetes doesn’t execute your YAML the way a script runs. What happens is you submit a typed declaration to a runtime and that runtime **continuously** works to make the infrastructure match your **intent**.

- The API server accepts your object. It parses the request, checks that the object is valid, applies admission and policy rules and then stores it as part of cluster's state.
- The cluster now "remembers" your intent. That object becomes **durable state**. It’s not a command that ran it’s a thing that exists (until you modify or delete it).
- Controllers continuously **reconcile**. Controllers watch for relevant objects and react to changes. They compare desired state (what you declared) with actual state (what exists right now) and take actions to reduce the difference.
- In our case, the scheduler picks a node to place the **Pod** on.
- The kubelet makes it real on the node. The node agent turns “a **Pod** should exist here” into “containers are actually running here”.

**declare → persist → reconcile → place → execute, on repeat.**

The part that makes Kubernetes feel different from orchestration is the word **continuously**. Kubernetes doesn’t just try once. If something drifts, the runtime tries to pull it back.

That’s why manual fixes often don’t work:
- deleting a **Pod** doesn’t remove the workload if the desired state still says it should exist
- editing live resources can get reverted if a controller owns that intent

This isn’t Kubernetes being annoying. It’s Kubernetes doing what a runtime does - enforcing the meaning of your declarations.

Once you see the runtime behavior **spec** and **status** start to feel like inputs and outputs:
- spec is what you want
- status is what the runtime observed and computed

This is a very practical way to debug. When something isn’t working I look at whether the runtime is progressing toward the **spec** and what it says in **status** about why it isn’t.

Kubernetes controllers don’t just loop blindly though. They react to changes.
- you change a declaration
- the runtime notices and converges
- you remove something
- the runtime reverts it if the spec still demands it

Once you see that, **GitOps** fits into the picture perfectly:
- Kubernetes stores the desired state inside the cluster.
- GitOps states that true desired state lives in Git.
- A GitOps controller continuously compares **Git (desired)** vs **cluster (actual)** and applies changes until they match.

So in this model, Git becomes your “source code repository", the cluster the runtime and the GitOps controller is the piece that keeps runtime state aligned with source.

If you `kubectl edit` something that GitOps manages, you’re changing runtime state, but the source of truth is still Git, so the GitOps controller will likely revert it back.

That’s not a fight between tools, it’s two reconciliations doing their job:
- Kubernetes reconciles toward the API objects
- GitOps reconciles the API objects toward Git

Which leads to a really practical rule of thumb. If something is GitOps managed, treat kubectl as a debugging tool, not as the way you make changes.

---

Thinking of Kubernetes as a **runtime for declarative infrastructure** instead of a mere orchestrator results in very practical approaches to operate your cluster.
- change the desired state, not the symptoms
- let reconciliation do the work
- make ownership explicit (especially with GitOps)
- use the type system (choose the right resource kind) instead of inventing your own conventions
