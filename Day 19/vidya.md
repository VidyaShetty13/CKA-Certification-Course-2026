# Resource requests , limits, limitrange

## Table of contents

- [Practice](#practice)
  - [If you do not specify a memory limit](#if-you-do-not-specify-a-memory-limit)
  - [If you do not specify a cpu limit](#if-you-do-not-specify-a-cpu-limit)
  - [If you do not specify a memory or cpu requests](#if-you-do-not-specify-a-memory-or-cpu-requests)
  - [Exceed a Container's cpu limit](#exceed-a-containers-cpu-limit)
  - [Exceed a Container's memory limit](#exceed-a-containers-memory-limit)
  - [Create a pod with cpu and memory requests and limits at pod-level](#create-a-pod-with-cpu-and-memory-requests-and-limits-at-pod-level)
  - [Create a pod with resource requests and limits at both pod-level and container-level](#create-a-pod-with-resource-requests-and-limits-at-both-pod-level-and-container-level)
  - [pod level resource is specified more than individual container](#pod-level-resource-is-specified-more-than-individual-container)
  - [Resizing CPU without restart for a container](#resizing-cpu-without-restart-for-a-container)
  - [Resizing memory with restart](#resizing-memory-with-restart)
  - [Infeasible resize request for cpu](#infeasible-resize-request-for-cpu)
  - [Pod level resize and restartPolicy behavior](#pod-level-resize-doesnt-contain-restart-policy-but-container-level-has-restartpolicy-so-what-happens-when-the-pod-level-resize-is-performed)
  - [Pod-Level Resources vs QoS](#pod-level-resources-vs-qos)

## Practice

### If you do not specify a memory limit
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 100m
        limits:
          cpu: 200m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "190M", "--vm-hang", "1"]
```

Result: If you do not specify a memory limit, the container is technically allowed to use all the memory available on the Node where it is running.

However, there are three things that will happen in a real Kubernetes cluster
- Node Exhaustion: If the stress tool actually attempts to consume more memory than the physical node has, the Node's Kernel OOM (Out Of Memory) Killer will step in.
- Pod Termination: Because this pod has a Request but no Limit, it is classified as a "Burstable" Quality of Service (QoS) class. If the node runs low on memory, Kubernetes will likely kill this pod first to protect "Guaranteed" pods (pods that have limits set).
- The "Cgroup" Reality: Behind the scenes, containerd creates a memory cgroup for this container with a limit set to the maximum value possible ($2^{64} - 1$ bytes).


### If you do not specify a cpu limit
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 100m
        limits:
          memory: "200Mi"
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "190M", "--vm-hang", "1"]
```

```yaml
$ k get pods -owide
NAME          READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
memory-demo   1/1     Running   0          5m50s   10.244.2.23   kind-worker2   <none>           <none>

---
$ k top pods
NAME          CPU(cores)   MEMORY(bytes)
memory-demo   4000m        0Mi

---
$ k top nodes
NAME                 CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
kind-control-plane   136m         0%       697Mi           4%
kind-worker          24m          0%       296Mi           1%
kind-worker2         4020m        25%      273Mi           1%

```

Result:  f you do not specify a cpu  limit, the container is technically allowed to use all the cpu available on the Node where it is running.

1. "Describe Node" = The Accounting Book (Reservation)
The table you see under Non-terminated Pods in kubectl describe node is not real-time. It is based on Requests and Limits (the "Accounting").

It shows 100m because that is what you Requested in your YAML.

It shows 0 (0%) for Limits because you didn't set one.

It does not care how much CPU the pod is actually using at this second. It only cares about how much of the Node's "capacity" has been officially "reserved" or "budgeted."

2. "Top Pods" = The Speedometer (Actual Usage)
The output of kubectl top pods comes from the Metrics Server (which gets it from cAdvisor). This is Real-Time Usage.

Your memory-demo pod is showing 4001m because your stress --cpu 4 command is actually working! It is consuming 4 full cores right now.

Why the difference matters
Imagine a hotel (the Node):

Describe Node (Requests): The guest booked 1 room (100m). The hotel manager marks 1 room as "taken" in the book.

Top Pods (Actual): The guest brought 40 people and they are all crammed into that 1 room and the hallway (4000m).

In Kubernetes, you can "overcommit." As long as you don't have a Limit, Kubernetes lets the pod "steal" unused CPU from the node. The "Accounting Book" (describe node) only updates if you change the reservation (the YAML), not if the pod gets busy.


### If you do not specify a memory or cpu requests

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        limits:
          memory: "200Mi"
          cpu: 200m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "190M", "--vm-hang", "1"]
```

Result: If you set a limit but leave the request blank, Kubernetes automatically sets the request to match the limit. By making the Request equal to the Limit, your Pod is moved into the Guaranteed Quality of Service (QoS) class (assuming you did this for both CPU and Memory)

---
If you have a limitrange in the namespace, then what happens to the requests?

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - defaultRequest:
      cpu: 100m
      memory: 100Mi
    default:
      cpu: 500m
      memory: 150Mi
    type: Container
---
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        limits:
          memory: "50Mi"
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "19M", "--vm-hang", "1"]
```

The Result: The Implicit Rule wins. Kubernetes will set the Requests to match your Limits (100m/50Mi). The defaultRequest from the LimitRange is ignored because the Pod already "has" a request (implied by the limit).

If your Pod YAML has no resources section at all: The Result: The LimitRange wins.

---

If you have a limitrange in the namespace, then what happens to the limits?

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - defaultRequest:
      cpu: 100m
      memory: 100Mi
    default:
      cpu: 500m
      memory: 150Mi
    type: Container
---
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "19M", "--vm-hang", "1"]

```

Result: The Pod has no limit, so the LimitRange tries to apply the default limit of 150Mi.

### Exceed a Container's cpu limit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 200m
        limits:
          memory: "200Mi"
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

Result: The Pod "memory-demo" is invalid: spec.containers[0].resources.requests: Invalid value: "200m": must be less than or equal to cpu limit of 100m

---

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 100m
        limits:
          memory: "200Mi"
          cpu: 100m
      command: ["stress"]
      args: ["--cpu", "2"]
```
```yaml
$ k top pod
NAME          CPU(cores)   MEMORY(bytes)
memory-demo   100m         0Mi

```
Result: Even though the stress tool is trying its hardest to use 2 full cores (2000m), the Kubernetes (Linux) kernel is "clamping" the process. It only allows the container to use 100m worth of CPU cycles per period.

Why is it exactly 100m?<br>
The Metrics Server (via k top) is showing you the enforced reality.

The container is being throttled.

### Exceed a Container's memory limit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 100m
        limits:
          memory: "150Mi"
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

```yaml
$ k get pods -w
NAME          READY   STATUS      RESTARTS      AGE
memory-demo   0/1     OOMKilled   2 (21s ago)   38s

---
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Thu, 29 Jan 2026 13:47:01 +0300
      Finished:     Thu, 29 Jan 2026 13:47:05 +0300
    Ready:          False

```

Result: When your container tried to grab 200M, it hit the 150Mi (~157MB) ceiling you set, and the Linux kernel stepped in to terminate the process instantly to protect the rest of the Node.

Why Exit Code 137?

In Linux, when a process is killed by a signal, the exit code is 128 + Signal Number.The OOM Killer sends SIGKILL, which is Signal 9.$128 + 9 = 137$.Whenever you see 137, you know the container didn't close gracefully—it was forced shut by the OS.

### Create a pod with cpu and memory requests and limits at pod-level

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 150Mi
      cpu: 100m
    requests:
      memory: 100Mi
      cpu: 100m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
```

Result: Pod is created successfully

### Create a pod with resource requests and limits at both pod-level and container-level

Create a pod with resource requests and limits at both pod-level and container-level

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 150Mi
      cpu: 100m
    requests:
      memory: 100Mi
      cpu: 100m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: "100Mi"
          cpu: 100m
        limits:
          memory: "150Mi"
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
```
Result:  Pod is created successfully

When you specify both, the Container sum must be mathematically compatible with the Pod level values.

Rule: The sum of all container requests must be less than or equal to the Pod-level requests.

Rule: The sum of all container limits must be less than or equal to the Pod-level limits.

Result if violated: The API server will reject the Pod creation immediately with a validation error.

### pod level resource is specified more than individual container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 300Mi
      cpu: 200m
    requests:
      memory: 300Mi
      cpu: 200m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 200Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

Result: OOMKilled :- Despite your Pod having a "budget" of 300Mi, your container will be killed when it tries to use 250M.

Here is why:

Cgroup Hierarchy: Kubernetes creates a "Parent" cgroup for the Pod (limited to 300Mi) and a "Child" cgroup for the container (limited to 200Mi).

The Violation: The stress tool tries to allocate 250M. It hits the Child cgroup limit (200Mi) first.

The Execution: The kernel doesn't care that the parent has 50Mi of "slack" space. Because the container-specific limit was breached, the container is terminated.

Scheduling: The Scheduler will look for a node with 300Mi of available memory (the Pod-level request). It ignores the 200Mi container request for scheduling purposes because the Pod-level request acts as the authoritative "reservation."

### Resizing CPU without restart for a container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 200Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 100m
      command: ["stress"]
      args: ["--cpu", "1"]

---
$ k top pods
NAME          CPU(cores)   MEMORY(bytes)
memory-demo   100m         0Mi
```

```yaml
observation:

1. This is Guaranteed QoS, so only changing the limit cpu from 100m to 200m , will not allow. you need to also change the requests so that is matches

$ k edit pod memory-demo --subresource resize
pod/memory-demo edited


---
updated resources
      resources:
        limits:
          cpu: 200m
          memory: 200Mi
        requests:
          cpu: 200m
          memory: 200Mi

---
$ k top pods
NAME          CPU(cores)   MEMORY(bytes)
memory-demo   200m         0Mi

```

Result: Pod is not restarted


### Resizing memory with restart

1. without restartPolicy in the containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 200Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```
```yaml
$ k edit pod memory-demo --subresource resize
pod/memory-demo edited

---
  Normal   ResizeCompleted  13s                   kubelet            Pod resize completed: {"containers":[{"name":"memory-demo-ctr","resources":{"limits":{"cpu":"100m","memory":"300Mi"},"requests":{"cpu":"100m","memory":"300Mi"}}}]}

    Limits:
      cpu:     100m
      memory:  300Mi
    Requests:
      cpu:        100m
      memory:     300Mi
```
Result: Pod is not restarted

---

2. with resizePolicy in the containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 200Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
      resizePolicy:
        - resourceName: memory
          restartPolicy: RestartContainer
        - resourceName: cpu
          restartPolicy: NotRequired
```

```yaml
# Edit the memory:

$ k edit pod memory-demo --subresource resize
pod/memory-demo edited

---
$ k get pods
NAME          READY   STATUS    RESTARTS      AGE
memory-demo   1/1     Running   1 (56s ago)   2m25s

---

  Normal  Killing          13s (x2 over 91s)    kubelet            Container memory-demo-ctr resize requires restart
  Normal  ResizeCompleted  30s                  kubelet            Pod resize completed: {"containers":[{"name":"memory-demo-ctr","resources":{"limits":{"cpu":"100m","memory":"300Mi"},"requests":{"cpu":"100m","memory":"300Mi"}}}]}

```

Result: The pod container is restarted

### Infeasible resize request for cpu

Next, try requesting an unreasonable amount of CPU, such as 1000 full cores (written as "1000" instead of "1000m" for millicores), which likely exceeds node capacity.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 200Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
      resizePolicy:
        - resourceName: memory
          restartPolicy: RestartContainer
        - resourceName: cpu
          restartPolicy: NotRequired
```

```yaml
$ k edit pod memory-demo --subresource resize
pod/memory-demo edited
---
$ k get pods memory-demo -oyaml
    resources:
      limits:
        cpu: 1k
        memory: 200Mi
      requests:
        cpu: 1k
        memory: 200Mi

status:
  conditions:
  - lastProbeTime: "2026-01-29T11:37:55Z"
    lastTransitionTime: "2026-01-29T11:37:55Z"
    message: 'Node didn''t have enough capacity: cpu, requested: 1000000, capacity:
      16000'
    observedGeneration: 2
    reason: Infeasible
    status: "True"
    type: PodResizePending

  containerStatuses:
  - allocatedResources:
      cpu: 100m
      memory: 200Mi
    containerID: containerd://d57e016d21af508ceb2ac7835ef05750b39a84ac820d7b1d50a666a6b687bb95
    image: docker.io/polinux/stress:latest
    imageID: docker.io/polinux/stress@sha256:b6144f84f9c15dac80deb48d3a646b55c7043ab1d83ea0a697c09097aaad21aa
    lastState: {}
    name: memory-demo-ctr
    ready: true
    resources:
      limits:
        cpu: 100m
        memory: 200Mi
      requests:
        cpu: 100m
        memory: 200Mi
```

Result:

- The spec.containers[0].resources reflects the desired state (cpu: "1000"
- A condition with type: PodResizePending and reason: Infeasible was added to the Pod.
- The condition's message will explain why (Node didn't have enough capacity: cpu, requested: 800000, capacity: ...)
- Crucially, status.containerStatuses[0].resources will still show the previous values (cpu: 800m, memory: 300Mi), because the infeasible resize was not applied by the Kubelet.
- The restartCount will not have changed due to this failed attempt.

### Pod level resize doesnt contain restart policy but container level has restartPolicy so what happens when the Pod level resize is performed?

- Pod-level resource resize does not support or require its own restart policy.
- No Pod-Level Policy: Changes to the Pod's aggregate resources (spec.resources) are always applied in-place without triggering a restart. This is because Pod-level resources act as an overall constraint on the Pod's cgroup and do not directly manage the application runtime within containers.
- Container Policy Still Governs: The resizePolicy must still be configured at the container level (spec.containers[*].resizePolicy). This policy governs whether an individual container is restarted when its resource requests or limits change, regardless of whether that change was initiated by a direct container-level resize or by an update to the overall Pod-level resource envelope

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 300Mi
      cpu: 200m
    requests:
      memory: 300Mi
      cpu: 200m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 200Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
      resizePolicy:
        - resourceName: memory
          restartPolicy: RestartContainer
        - resourceName: cpu
          restartPolicy: NotRequired

```

Note this is supportive only in 1.35 , if its below that version then u get below error
<br>$ kubectl patch pod memory-demo --subresource resize --patch '{"spec":{"resources":{"requests":{"cpu":"250m", "memory":"350Mi"}, "limits":{"cpu":"250m", "memory":"350Mi"}}}}'
<br>The Pod "memory-demo" is invalid: []: Forbidden: pods with pod-level resources cannot be resized


Reference: https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/#example-resizing-pod-level-resources
---

### Pod-Level Resources vs QoS
In Kubernetes, the QoS class is determined by the effective requests and limits of the Pod. With the introduction of Pod-level resources in v1.35, the rules have evolved to include that top-level "budget."

1. If pod level resource requests and limits doesnt match what would be the QoS class?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 300Mi
      cpu: 200m
    requests:
      memory: 200Mi
      cpu: 100m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]

```
Result: qosClass: Burstable
---

2. If pod level resource requests and limits does match what would be the QoS class?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 300Mi
      cpu: 200m
    requests:
      memory: 300Mi
      cpu: 200m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```
Result: qosClass: Guaranteed
---

3. If pod level resource requests and limits `doesnt match` but container resource request and limits `match` what would be the QoS class?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 300Mi
      cpu: 500m
    requests:
      memory: 200Mi
      cpu: 250m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 100Mi
          cpu: 100m
        limits:
          memory: 100Mi
          cpu: 100m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
```
Result: qosClass: Burstable
---

4. If pod level resource requests and limits `does match` but container resource request and limits `doesnt match` what would be the QoS class?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  resources:
    limits:
      memory: 300Mi
      cpu: 200m
    requests:
      memory: 300Mi
      cpu: 200m
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 100Mi
          cpu: 100m
        limits:
          memory: 200Mi
          cpu: 150m
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Result: qosClass: Guaranteed
---
