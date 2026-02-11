# Pod priority and PreemptionPolic

## Extra Questions

<details>
  <summary>How do you enable Priority for a Pod?</summary>

  - Create a PriorityClass object: This defines a name and an integer value (higher = more important)
  - Assign it to the Pod: Use the priorityClassName field in the Pod's spec.
    
</details>

<details>
  <summary>What is the difference between "Priority" and "Preemption"?</summary>

  - Priority: Affects the Scheduling Queue. If the cluster is full, the scheduler looks at the queue and picks the Pod with the highest priority to try and place first.
  - Preemption: Affects Running Pods. If a high-priority Pod cannot be scheduled because there are no resources, the scheduler will look for lower-priority Pods to kill (evict) to make room
</details>

<details>
  <summary>What does preemptionPolicy: Never do?</summary>

  - if you set preemptionPolicy: Never in your PriorityClass, the Pod will still go to the front of the line in the scheduling queue (it has high priority), but it will not kick anyone else off.
    - It stays "Pending" until resources naturally become free.
      
</details>

<details>
  <summary>If a Pod is preempted, what happens to it?</summary>

  - It will actually be in Pending state
  - The scheduler identifies a lower-priority Pod.
    - 2. The lower-priority Pod gets a Graceful Termination period (SIGTERM).
      3. The status of the preempted Pod will show why it was killed (check kubectl describe pod).
      4. Once the lower-priority Pod is gone, the high-priority Pod is scheduled on that node.
    
</details>

<details>
  <summary>Can a PriorityClass affect the whole cluster by default?</summary>

  - Yes. If you set globalDefault: true in a PriorityClass, any Pod created without a specific priorityClassName will automatically inherit that priority. You can only have one global default in a cluster.
</details>

<details>
  <summary>What are the two built-in PriorityClasses?</summary>

  - system-node-critical: Highest priority. Used for things like kube-proxy or calico.
  - system-cluster-critical: Very high priority. Used for things like coredns.
</details>

<details>
  <summary>global default priority value for any pod created without priorityclass?</summary>

  - 0

</details>

## Scenario

<details>
  <summary>Both pods with default priority is trying to be created on the node which has very less resource available. what happens in this case</summary>

  - Create 2 pods parallely
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: lp-nginx
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            resources:
              requests:
                memory: "64Mi"
                cpu: "4000m"

    ```
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hp-nginx
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            resources:
              requests:
                memory: "64Mi"
                cpu: "4000m"
    
    ```
  - check the pod priority
    ```yaml
    $ k describe pod hp-nginx-b9c68f9bb-4n5b6 |grep Priority
    Priority:         0
    
    $ k describe pod lp-nginx-b9c68f9bb-4sfz7 |grep Priority
    Priority:         0
    
    ```
  - check the num of pods scheduled on the node
    ```yaml
    $ k get pods -owide
    NAME                       READY   STATUS    RESTARTS   AGE     IP            NODE                     NOMINATED NODE   READINESS GATES
    hp-nginx-b9c68f9bb-4n5b6   1/1     Running   0          3m48s   10.244.0.22   cka-2025-control-plane   <none>           <none>
    hp-nginx-b9c68f9bb-8sb2c   1/1     Running   0          3m48s   10.244.0.24   cka-2025-control-plane   <none>           <none>
    hp-nginx-b9c68f9bb-hnzc8   0/1     Pending   0          3m48s   <none>        <none>                   <none>           <none>
    hp-nginx-b9c68f9bb-rbbpj   0/1     Pending   0          3m48s   <none>        <none>                   <none>           <none>
    lp-nginx-b9c68f9bb-4sfz7   1/1     Running   0          3m48s   10.244.0.23   cka-2025-control-plane   <none>           <none>
    lp-nginx-b9c68f9bb-8xcnc   0/1     Pending   0          3m48s   <none>        <none>                   <none>           <none>
    lp-nginx-b9c68f9bb-hms72   0/1     Pending   0          3m48s   <none>        <none>                   <none>           <none>
    lp-nginx-b9c68f9bb-wdsp9   0/1     Pending   0          3m48s   <none>        <none>                   <none>           <none>
    ```
    
  - Results
    - Few pods from both the pod spec is created on the node
    - remainaing are in pending state because of insufficient cpu
      ```yaml
      $ k describe pod lp-nginx-b9c68f9bb-8xcnc
      Name:             lp-nginx-b9c68f9bb-8xcnc
      Namespace:        default
      Priority:         0
      Service Account:  default
      Node:             <none>
      Labels:           app=nginx
                        pod-template-hash=b9c68f9bb
      Annotations:      <none>
      Status:           Pending
      IP:
      IPs:              <none>
      Controlled By:    ReplicaSet/lp-nginx-b9c68f9bb
      Containers:
        nginx:
          Image:      nginx:latest
          Port:       <none>
          Host Port:  <none>
          Requests:
            cpu:        4
            memory:     64Mi
          Environment:  <none>
          Mounts:
            /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w62g6 (ro)
      Conditions:
        Type           Status
        PodScheduled   False
      Volumes:
        kube-api-access-w62g6:
          Type:                    Projected (a volume that contains injected data from multiple sources)
          TokenExpirationSeconds:  3607
          ConfigMapName:           kube-root-ca.crt
          Optional:                false
          DownwardAPI:             true
      QoS Class:                   Burstable
      Node-Selectors:              <none>
      Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                   node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
      Events:
        Type     Reason            Age    From               Message
        ----     ------            ----   ----               -------
        Warning  FailedScheduling  4m34s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. no new claims to deallocate, preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
            
      ```
</details>

<details>
  <summary>The "Non-Aggressive" VIP (PreemptionPolicy)</summary>

  - Task
    - Create a PriorityClass named batch-priority
    - Give it a value of 500000.
    - Set the configuration so that Pods with this class are scheduled before others in the queue, but they are forbidden from evicting lower-priority Pods to make room.

  - Action
    - Check if any pods are already running excluding kube-system namespace
      No
      
    - check what is the pending resource in the node
      ```yaml
      $ k describe node
      Name:               cka-2025-control-plane
      Roles:              control-plane
      Labels:             beta.kubernetes.io/arch=amd64
                          beta.kubernetes.io/os=linux
                          kubernetes.io/arch=amd64
                          kubernetes.io/hostname=cka-2025-control-plane
                          kubernetes.io/os=linux
                          node-role.kubernetes.io/control-plane=
      Annotations:        node.alpha.kubernetes.io/ttl: 0
                          volumes.kubernetes.io/controller-managed-attach-detach: true
      CreationTimestamp:  Wed, 11 Feb 2026 15:58:56 +0300
      Taints:             <none>
      Unschedulable:      false
      Lease:
        HolderIdentity:  cka-2025-control-plane
        AcquireTime:     <unset>
        RenewTime:       Wed, 11 Feb 2026 16:32:49 +0300
      Conditions:
        Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
        ----             ------  -----------------                 ------------------                ------                       -------
        MemoryPressure   False   Wed, 11 Feb 2026 16:32:14 +0300   Wed, 11 Feb 2026 15:58:54 +0300   KubeletHasSufficientMemory   kubelet has sufficient memory available
        DiskPressure     False   Wed, 11 Feb 2026 16:32:14 +0300   Wed, 11 Feb 2026 15:58:54 +0300   KubeletHasNoDiskPressure     kubelet has no disk pressure
        PIDPressure      False   Wed, 11 Feb 2026 16:32:14 +0300   Wed, 11 Feb 2026 15:58:54 +0300   KubeletHasSufficientPID      kubelet has sufficient PID available
        Ready            True    Wed, 11 Feb 2026 16:32:14 +0300   Wed, 11 Feb 2026 15:59:18 +0300   KubeletReady                 kubelet is posting ready status
      Addresses:
        InternalIP:  172.19.0.5
        Hostname:    cka-2025-control-plane
      Capacity:
        cpu:                16
        ephemeral-storage:  1055762868Ki
        hugepages-1Gi:      0
        hugepages-2Mi:      0
        memory:             16115904Ki
        pods:               110
      Allocatable:
        cpu:                16
        ephemeral-storage:  1055762868Ki
        hugepages-1Gi:      0
        hugepages-2Mi:      0
        memory:             16115904Ki
        pods:               110
      System Info:
        Machine ID:                 d0a66e359d7e452b8267ecedc6197bb8
        System UUID:                d0a66e359d7e452b8267ecedc6197bb8
        Boot ID:                    3317330c-070f-4de3-aa0a-b92f6e81109d
        Kernel Version:             6.6.87.2-microsoft-standard-WSL2
        OS Image:                   Debian GNU/Linux 12 (bookworm)
        Operating System:           linux
        Architecture:               amd64
        Container Runtime Version:  containerd://2.2.0
        Kubelet Version:            v1.35.0
        Kube-Proxy Version:
      PodCIDR:                      10.244.0.0/24
      PodCIDRs:                     10.244.0.0/24
      ProviderID:                   kind://docker/cka-2025/cka-2025-control-plane
      Non-terminated Pods:          (9 in total)
        Namespace                   Name                                              CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
        ---------                   ----                                              ------------  ----------  ---------------  -------------  ---
        kube-system                 coredns-7d764666f9-lx274                          100m (0%)     0 (0%)      70Mi (0%)        170Mi (1%)     33m
        kube-system                 coredns-7d764666f9-tn885                          100m (0%)     0 (0%)      70Mi (0%)        170Mi (1%)     33m
        kube-system                 etcd-cka-2025-control-plane                       100m (0%)     0 (0%)      100Mi (0%)       0 (0%)         34m
        kube-system                 kindnet-6b245                                     100m (0%)     100m (0%)   50Mi (0%)        50Mi (0%)      33m
        kube-system                 kube-apiserver-cka-2025-control-plane             250m (1%)     0 (0%)      0 (0%)           0 (0%)         34m
        kube-system                 kube-controller-manager-cka-2025-control-plane    200m (1%)     0 (0%)      0 (0%)           0 (0%)         34m
        kube-system                 kube-proxy-59cs7                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
        kube-system                 kube-scheduler-cka-2025-control-plane             100m (0%)     0 (0%)      0 (0%)           0 (0%)         34m
        local-path-storage          local-path-provisioner-67b8995b4b-s4rlb           0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
      Allocated resources:
        (Total limits may be over 100 percent, i.e., overcommitted.)
        Resource           Requests    Limits
        --------           --------    ------
        cpu                950m (5%)   100m (0%)
        memory             290Mi (1%)  390Mi (2%)
        ephemeral-storage  0 (0%)      0 (0%)
        hugepages-1Gi      0 (0%)      0 (0%)
        hugepages-2Mi      0 (0%)      0 (0%)
      Events:
        Type    Reason          Age   From             Message
        ----    ------          ----  ----             -------
        Normal  RegisteredNode  33m   node-controller  Node cka-2025-control-plane event: Registered Node cka-2025-control-plane in Controller
            
      ```
      Total 16core cpu, out of which 0.950 is already been use. so available are 15cpu
      
    - Create PC
      ```yaml
      $ k create priorityclass  batch-priority --value=500000 --description="high priority" --preemption-policy="Never"
      priorityclass.scheduling.k8s.io/batch-priority created

      ```
      ```yaml
      $ k get pc batch-priority -oyaml
      apiVersion: scheduling.k8s.io/v1
      description: high priority
      kind: PriorityClass
      metadata:
        creationTimestamp: "2026-02-11T13:35:42Z"
        generation: 1
        name: batch-priority
        resourceVersion: "4258"
        uid: e5aee5e7-41fa-4fb4-a206-8054f8f812d2
      preemptionPolicy: Never
      value: 500000
      
      ```
      
    - Create 2 Pods one with priorityclass and the other without it
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hp-nginx
      spec:
        replicas: 4
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            priorityClassName: batch-priority
            containers:
            - name: nginx
              image: nginx:latest
              resources:
                limits:
                  memory: "64Mi"
                  cpu: "4000m"
      ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: lp-nginx
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            containers:
            - name: nginx
              image: nginx:latest
              resources:
                limits:
                  memory: "64Mi"
                  cpu: "5000m"
      
      ```
      
    - Results:
      - Pod with higher priority will comme first in the queue and it will be executed
        ```yaml
        $ k get pods
        NAME                        READY   STATUS    RESTARTS   AGE
        hp-nginx-5f8477b759-4hnkp   1/1     Running   0          3m1s
        hp-nginx-5f8477b759-695mk   1/1     Running   0          3m1s
        hp-nginx-5f8477b759-nfk4r   1/1     Running   0          3m1s
        hp-nginx-5f8477b759-rxmrt   0/1     Pending   0          3m1s
        lp-nginx-6c65cc7544-ctm9z   0/1     Pending   0          3m1s
        ```
        ```yaml
        $ k describe pod hp-nginx-5f8477b759-rxmrt
        Warning  FailedScheduling  3m26s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. no new claims to deallocate, preemption: not eligible due to preemptionPolicy=Never.
        ```
      - whereas the lower pod will be in Pending state
        ```yaml
        $ k describe pod lp-nginx-6c65cc7544-ctm9z
        Warning  FailedScheduling  3m33s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. no new claims to deallocate, preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod
        ```
      - If tried to place another pod with higher Priority and PREEMPTIONPOLICY set to Never, It will be in stuck state
        ```yaml
        $ k create priorityclass  high-priority --value=600000 --description="high priority" --preemption-policy="Never"
        priorityclass.scheduling.k8s.io/high-priority created
        ```
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: highest-priority
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: nginx
          template:
            metadata:
              labels:
                app: nginx
            spec:
              priorityClassName: high-priority
              containers:
              - name: nginx
                image: nginx:latest
                resources:
                  requests:
                    memory: "64Mi"
                    cpu: "4000m"
        ```
      - If tried to place another pod with higher Priority and PREEMPTIONPOLICY set to PreemptLowerPriority, It will remove other pods and place itself
        ```yaml
        $ k create priorityclass  top-priority --value=700000 --description="top priority"
        priorityclass.scheduling.k8s.io/top-priority created

        ```
        
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: top-priority
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: nginx
          template:
            metadata:
              labels:
                app: nginx
            spec:
              priorityClassName: top-priority
              containers:
              - name: nginx
                image: nginx:latest
                resources:
                  requests:
                    memory: "64Mi"
                    cpu: "4000m"
        
        ```

        ```yaml
        $ k get pods  -w
        NAME                               READY   STATUS    RESTARTS   AGE
        highest-priority-f6f8bc699-67p52   0/1     Pending   0          2m30s
        hp-nginx-5f8477b759-4hnkp          1/1     Running   0          7m49s
        hp-nginx-5f8477b759-nfk4r          1/1     Running   0          7m49s
        hp-nginx-5f8477b759-rxmrt          0/1     Pending   0          7m49s
        hp-nginx-5f8477b759-tvx5t          0/1     Pending   0          29s
        lp-nginx-6c65cc7544-ctm9z          0/1     Pending   0          7m49s
        top-priority-9469f4464-7dzp2       1/1     Running   0          29s
        ```
</details>


## Errors

<details>
  <summary>Warning  FailedScheduling  19s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu. no new claims to deallocate, preemption: not eligible due to preemptionPolicy=Never.</summary>

  - when preemptionPolicy is set to Never, that Pod is essentially told: "You can only have a seat if there is an empty chair." * The "Why": Even if this Pod has a priorityClassName higher than every other Pod on the node, it is forbidden from kicking them off.
  - The Result: It will sit in a Pending state indefinitely until resources naturally free up (e.g., another Pod finishes or the node is scaled up).
    
</details>

<details>
  <summary>Warning  FailedScheduling  2m32s (x6 over 2m33s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Insufficient cpu</summary>

  - The scheduler is allowed to evict others here, but it simply can't find a "victim" that would actually solve the problem.
  - The "Why": For preemption to work, two things must be true:
    - There must be Pods with lower priority than the pending Pod.
    - Removing those specific lower-priority Pods must free up enough CPU to fit the new Pod.
    - Even if there are lower priority pods, the scheduler might realize that even after killing all of them, the node still wouldn't have enough CPU (perhaps because high-priority pods or system overhead are taking up the bulk of the resource).
    
</details>
