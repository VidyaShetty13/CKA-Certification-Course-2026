# NodeName, NodeSelector, Taints and Toleration, node Affinity cases

## NodeName

<details>
  <summary>1. Create a pod on controlplane node which has taint on it</summary>

  - node status
    ```yaml
    $ k describe node kind-control-plane
    Name:               kind-control-plane
    Roles:              control-plane
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-control-plane
                        kubernetes.io/os=linux
                        node-role.kubernetes.io/control-plane=
                        node.kubernetes.io/exclude-from-external-load-balancers=
    Taints:             node-role.kubernetes.io/control-plane:NoSchedule
    Unschedulable:      false
    ```

  - pod placement on controlplane node
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: nginx
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      nodeName: kind-control-plane
    ```

  - pod status
    ```yaml
    $ k get pods  -owide
    NAME    READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
    nginx   1/1     Running   0          6s    10.244.0.10   kind-control-plane   <none>           <none>
    ```

    ```yaml
    $ k describe pod nginx
    Name:             nginx
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             kind-control-plane/172.19.0.2
    Start Time:       Fri, 30 Jan 2026 12:27:30 +0300
    Labels:           run=nginx
    Annotations:      <none>
    Status:           Running
    IP:               10.244.0.10
    IPs:
      IP:  10.244.0.10
    Containers:
      nginx:
        Container ID:   containerd://4fbd8015c29f17797627410c8c974961ef18ea5bbdf39fec1afb1e88e72f4186
        Image:          nginx
        Image ID:       docker.io/library/nginx@sha256:c881927c4077710ac4b1da63b83aa163937fb47457950c267d92f7e4dedf4aec
        Port:           80/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Fri, 30 Jan 2026 12:27:32 +0300
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kbl9m (ro)
    Conditions:
      Type                        Status
      PodReadyToStartContainers   True
      Initialized                 True
      Ready                       True
      ContainersReady             True
      PodScheduled                True
    Volumes:
      kube-api-access-kbl9m:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        Optional:                false
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type    Reason   Age   From     Message
      ----    ------   ----  ----     -------
      Normal  Pulling  39s   kubelet  Pulling image "nginx"
      Normal  Pulled   38s   kubelet  Successfully pulled image "nginx" in 1.432s (1.432s including waiting). Image size: 62870438 bytes.
      Normal  Created  38s   kubelet  Created container: nginx
      Normal  Started  38s   kubelet  Started container nginx
    ```

  **Result**

  - Look at the Events in your describe output. Notice there is no "Scheduled" event from the default-scheduler. Usually, you'd see Successfully assigned default/nginx to kind-control-plane. Here, it goes straight to the kubelet pulling the image
  - The node-role.kubernetes.io/control-plane:NoSchedule taint is a "No Entry" sign held by the Scheduler. Since nodeName tells the Kubelet "this pod belongs to you" directly, the Scheduler never gets a chance to hold up that sign.
  
</details>

## NodeSelector

<details>
  <summary>1. Create a pod on controlplane node which has taint on it</summary>
  
  - node status
    ```yaml
    $ k describe node kind-control-plane
    Name:               kind-control-plane
    Roles:              control-plane
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-control-plane
                        kubernetes.io/os=linux
                        node-role.kubernetes.io/control-plane=
                        node.kubernetes.io/exclude-from-external-load-balancers=
    Taints:             node-role.kubernetes.io/control-plane:NoSchedule
    Unschedulable:      false
    ```
  
  - pod placement on controlplane node
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: nginx
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: kind-control-plane
    ```

  - pod status
    ```yaml
    $ k get pods -owide
    NAME    READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
    nginx   0/1     Pending   0          10m   <none>   <none>   <none>           <none>
    
    $ k describe pod nginx
    Name:             nginx
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             <none>
    Labels:           run=nginx
    Annotations:      <none>
    Status:           Pending
    IP:
    IPs:              <none>
    Containers:
      nginx:
        Image:        nginx
        Port:         80/TCP
        Host Port:    0/TCP
        Environment:  <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pxmwv (ro)
    Conditions:
      Type           Status
      PodScheduled   False
    Volumes:
      kube-api-access-pxmwv:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        Optional:                false
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              kubernetes.io/hostname=kind-control-plane
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type     Reason            Age               From               Message
      ----     ------            ----              ----               -------
      Warning  FailedScheduling  8s (x3 over 10m)  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
  
    ```
    
    **Result**

    Your Pod is going through the Scheduler. In Kubernetes, simply adding a nodeSelector does not bypass the Scheduler. It just gives the Scheduler a "must-have" rule. Since the Scheduler is involved, it respects the Taint, sees that your Pod lacks a Toleration, and marks the Pod as Pending
  
  
</details>


<details>
  <summary>2. Create a pod on controlplane node which doesnt have taint on it</summary>

  - node status
    ```yaml
    $ k describe node kind-worker
    Name:               kind-worker
    Roles:              <none>
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-worker
                        kubernetes.io/os=linux
    Taints:             <none>
    Unschedulable:      false
    ```

  - pod placement on worker node
    ```yaml
    $ cat nginx.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: nginx
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: kind-worker
    ```

  - pod status
    ```yaml
    $ k get pods -owide
    NAME    READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    nginx   1/1     Running   0          4s    10.244.1.27   kind-worker   <none>           <none>
    ```

    ```yaml
    $ k describe pod nginx
    Name:             nginx
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             kind-worker/172.19.0.3
    Start Time:       Fri, 30 Jan 2026 12:19:21 +0300
    Labels:           run=nginx
    Annotations:      <none>
    Status:           Running
    IP:               10.244.1.27
    IPs:
      IP:  10.244.1.27
    Containers:
      nginx:
        Container ID:   containerd://5d5fa3d1e6bcb3541a3d2971738e28bf2fd85519246d7a0b69426cdecdc31675
        Image:          nginx
        Image ID:       docker.io/library/nginx@sha256:c881927c4077710ac4b1da63b83aa163937fb47457950c267d92f7e4dedf4aec
        Port:           80/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Fri, 30 Jan 2026 12:19:23 +0300
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-6nm9d (ro)
    Conditions:
      Type                        Status
      PodReadyToStartContainers   True
      Initialized                 True
      Ready                       True
      ContainersReady             True
      PodScheduled                True
    Volumes:
      kube-api-access-6nm9d:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        Optional:                false
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              kubernetes.io/hostname=kind-worker
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type    Reason     Age   From               Message
      ----    ------     ----  ----               -------
      Normal  Scheduled  18s   default-scheduler  Successfully assigned default/nginx to kind-worker
      Normal  Pulling    17s   kubelet            Pulling image "nginx"
      Normal  Pulled     16s   kubelet            Successfully pulled image "nginx" in 1.625s (1.625s including waiting). Image size: 62870438 bytes.
      Normal  Created    16s   kubelet            Created container: nginx
      Normal  Started    16s   kubelet            Started container nginx

    ```

  **Results**

  Pod is scheduled successfully on worker node by default scheduler
  
</details>


## Taints and Tolerations

<details>
  <summary>1. Place the pod on the nodes only which has taints on it. example place the pod on controlplane node</summary>

  - node status
    ```yaml
    $ k describe node kind-control-plane
    Name:               kind-control-plane
    Roles:              control-plane
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-control-plane
                        kubernetes.io/os=linux
                        node-role.kubernetes.io/control-plane=
                        node.kubernetes.io/exclude-from-external-load-balancers=
    Taints:             node-role.kubernetes.io/control-plane:NoSchedule
    Unschedulable:      false
    ```

  - pod placement on controlplane node
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx
      name: nginx
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              operator: Exists
              effect: NoSchedule
    ```

  - pod status
    ```yaml
    $ k get pods -owide
    NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
    nginx-5f4db66bf7-cs8pk   1/1     Running   0          33s   10.244.0.11   kind-control-plane   <none>           <none>
    nginx-5f4db66bf7-dtkvj   1/1     Running   0          33s   10.244.2.44   kind-worker2         <none>           <none>
    nginx-5f4db66bf7-h8fdl   1/1     Running   0          33s   10.244.1.28   kind-worker          <none>           <none>
    nginx-5f4db66bf7-lghzw   1/1     Running   0          33s   10.244.2.43   kind-worker2         <none>           <none>
    nginx-5f4db66bf7-qqq8t   1/1     Running   0          33s   10.244.1.29   kind-worker          <none>           <none>
    ```

  **Results**

  - By adding tolerations, the pod is no longer repelled by the control plane's taint, making all three nodes eligible for scheduling.
  - The scheduler's balancing algorithm distributed the 5 replicas across the entire cluster, including the control plane node
  - If I change the taint on controlplane node from `NoSchedule` to `NoExecute` then the pod from controlplane node will be immediately evicted and placed into another node. Incase if you want a graceful pod eveiction in this case then add a `tolerationSeconds`. This allows the pod to stay on the tainted node for a specific amount of time (e.g., 300 seconds) before being kicked off.
  - Drawback: If we were expecting the pod placement only on `contolplane node` then this is not achievable with alone `taints`

</details>

## Node-Affinity

<details>
  <summary>1. Place the pod on kind-worker node using NodeAffinity</summary>

  - node status
    ```yaml
    $ k describe node kind-worker
    Name:               kind-worker
    Roles:              <none>
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-worker
                        kubernetes.io/os=linux
    Taints:             <none>
    Unschedulable:      false
    ```

  - pod placement on worker node
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx
      name: nginx
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                    - key: kubernetes.io/hostname
                      operator: In
                      values:
                        - kind-worker
    ```

  - pod status
    ```yaml
    $ k get pods  -owide
    NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    nginx-59f9b78c64-gc9b2   1/1     Running   0          38s   10.244.1.33   kind-worker   <none>           <none>
    nginx-59f9b78c64-nfbx6   1/1     Running   0          38s   10.244.1.34   kind-worker   <none>           <none>
    nginx-59f9b78c64-wb4vk   1/1     Running   0          38s   10.244.1.32   kind-worker   <none>           <none>
    nginx-59f9b78c64-xbntb   1/1     Running   0          38s   10.244.1.31   kind-worker   <none>           <none>
    nginx-59f9b78c64-xqgp9   1/1     Running   0          38s   10.244.1.35   kind-worker   <none>           <none>
    ```

  **Results**

  - Pod is successfully placed on kind-worker using node-affinity
</details>

## Taints and Tolerations with Node Affinity

<details>
  <summary>1. Place the pod on kind-worker node using Taints & Tolerations with  NodeAffinity</summary>

  - node status
    ```yaml
    $ k describe node kind-worker
    Name:               kind-worker
    Roles:              <none>
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-worker
                        kubernetes.io/os=linux
    Taints:             staging=test:NoExecute
    Unschedulable:      false
    ```

  - pod placement on worker node
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx
      name: nginx
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                    - key: staging
                      operator: In
                      values:
                        - test
          tolerations:
            - key: staging
              value: test
              operator: Equal
              effect: NoExecute
    ```

  - pod status
    ```yaml
    $ k get pods -owide
    NAME                     READY   STATUS    RESTARTS   AGE    IP            NODE          NOMINATED NODE   READINESS GATES
    nginx-54ffcb4cb4-2szqz   1/1     Running   0          2m2s   10.244.1.42   kind-worker   <none>           <none>
    nginx-54ffcb4cb4-kdvpv   1/1     Running   0          2m2s   10.244.1.41   kind-worker   <none>           <none>
    nginx-54ffcb4cb4-pmk6s   1/1     Running   0          2m2s   10.244.1.45   kind-worker   <none>           <none>
    nginx-54ffcb4cb4-v9ldg   1/1     Running   0          2m2s   10.244.1.44   kind-worker   <none>           <none>
    nginx-54ffcb4cb4-z4txb   1/1     Running   0          2m2s   10.244.1.43   kind-worker   <none>           <none>
    ```
    
  **Results**

  - with this solution pod is successfully placed on `kind-worker` node
  - If you change/remove the Label (staging=test), the pods will NOT be evicted.
    - NodeAffinity is IgnoredDuringExecution. Removing the label only prevents new pods from arriving; it doesn't kick out existing ones.
    - NoExecute is tied to the Taint, not the Label. As long as the Taint (staging=test:NoExecute) remains on the node, and the Pod has the matching Toleration, the Pod is safe—even if the Label is gone.

</details>

<details>
  <summary>2. what happens to the pod, if a taint is added after the pod is scheduled in that node?</summary>

  - node status
    ```yaml
    $ k describe node kind-worker
    Name:               kind-worker
    Roles:              <none>
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-worker
                        kubernetes.io/os=linux
    Taints:             <none>
    Unschedulable:      false
    ```

  - pod placement on worker node
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx
      name: nginx
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                    - key: kubernetes.io/hostname
                      operator: In
                      values:
                        - kind-worker
    ```

  - pod status
    ```yaml
    $ k get pods  -owide
    NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    nginx-59f9b78c64-gc9b2   1/1     Running   0          38s   10.244.1.33   kind-worker   <none>           <none>
    nginx-59f9b78c64-nfbx6   1/1     Running   0          38s   10.244.1.34   kind-worker   <none>           <none>
    nginx-59f9b78c64-wb4vk   1/1     Running   0          38s   10.244.1.32   kind-worker   <none>           <none>
    nginx-59f9b78c64-xbntb   1/1     Running   0          38s   10.244.1.31   kind-worker   <none>           <none>
    nginx-59f9b78c64-xqgp9   1/1     Running   0          38s   10.244.1.35   kind-worker   <none>           <none>
    ```

  - Now add the taint to `kind-worker` node with `NoSchedule`
    ```yaml
    $ k taint node kind-worker staging=test:NoSchedule
    node/kind-worker tainted
    ```

    **Results**
    
    pods will continue to run on same `kind-worker` node

    ```yaml
    $ k get pods  -owide
    NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    nginx-854b4c87d-6n9np   1/1     Running   0          81s   10.244.1.50   kind-worker   <none>           <none>
    nginx-854b4c87d-d86cc   1/1     Running   0          80s   10.244.1.49   kind-worker   <none>           <none>
    nginx-854b4c87d-hm57r   1/1     Running   0          80s   10.244.1.46   kind-worker   <none>           <none>
    nginx-854b4c87d-sp6gx   1/1     Running   0          80s   10.244.1.48   kind-worker   <none>           <none>
    nginx-854b4c87d-v558p   1/1     Running   0          80s   10.244.1.47   kind-worker   <none>           <none>
    ```
    
  - Now add the taint to `kind-worker` node with `NoExecute`
    ```yaml
    $ k taint node kind-worker staging=test:NoExecute
    node/kind-worker tainted
    ```

    **Results**

    Pods are immediately evicted from that node

    ```yaml
    $ k get pods  -owide
    NAME                    READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
    nginx-854b4c87d-4mhg4   0/1     Pending   0          1s    <none>   <none>   <none>           <none>
    nginx-854b4c87d-b6j8j   0/1     Pending   0          1s    <none>   <none>   <none>           <none>
    nginx-854b4c87d-gm6ks   0/1     Pending   0          1s    <none>   <none>   <none>           <none>
    nginx-854b4c87d-wddxv   0/1     Pending   0          1s    <none>   <none>   <none>           <none>
    nginx-854b4c87d-xzqmk   0/1     Pending   0          1s    <none>   <none>   <none>           <none>
    ```

</details>

<details>
  <summary>3. what happens to the pod, if a labels is removed after the pod is scheduled in that node?</summary>

  - node status
    ```yaml
    $ k describe node kind-worker
    Name:               kind-worker
    Roles:              <none>
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=kind-worker
                        kubernetes.io/os=linux
                        staging: test
    Taints:             <none>
    Unschedulable:      false
    ```

  - pod placement on worker node
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx
      name: nginx
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                    - key: staging
                      operator: In
                      values:
                        - test
    ```

  - pod status
    ```yaml
    $ k get pods  -owide
    NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    nginx-854b4c87d-bjp67   1/1     Running   0          31s   10.244.1.55   kind-worker   <none>           <none>
    nginx-854b4c87d-dnzbm   1/1     Running   0          31s   10.244.1.52   kind-worker   <none>           <none>
    nginx-854b4c87d-fg8hd   1/1     Running   0          31s   10.244.1.54   kind-worker   <none>           <none>
    nginx-854b4c87d-vkrr5   1/1     Running   0          31s   10.244.1.53   kind-worker   <none>           <none>
    nginx-854b4c87d-zs9b5   1/1     Running   0          31s   10.244.1.51   kind-worker   <none>           <none>
    ```

  - remove the label from the node
    ```yaml
    $ k label node kind-worker staging-
    node/kind-worker unlabeled
    ```

  **Reults**
  - pod will continue to run on the same node

    ```yaml
    $ k get pods -owide
    NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    nginx-854b4c87d-bjp67   1/1     Running   0          85s   10.244.1.55   kind-worker   <none>           <none>
    nginx-854b4c87d-dnzbm   1/1     Running   0          85s   10.244.1.52   kind-worker   <none>           <none>
    nginx-854b4c87d-fg8hd   1/1     Running   0          85s   10.244.1.54   kind-worker   <none>           <none>
    nginx-854b4c87d-vkrr5   1/1     Running   0          85s   10.244.1.53   kind-worker   <none>           <none>
    nginx-854b4c87d-zs9b5   1/1     Running   0          85s   10.244.1.51   kind-worker   <none>           <none>

    ```
</details>

## Why can't we just use NodeAffinity? why do we require taints?

Say suppose we have 2 nodes 
- node-1 with taint `staging=prod` and label `staging=prod`
- node-2 without taint and label `staging=dev`
- node-3 without any labels and taints

- Now the pod-1 with nodeaffinity rules saying i shall be placed on only `staging=dev` node, then this will be successfully placed on node-2
- Another pod-2 without any selectors or affinities or tolerations, this will also be placed on node-2 or node-3

Drawback of using only node affinity is :- if we have a pod-3 this will get placed on either node-2 or node-3. therefore, we are not preventing the unwanted pods to the specific node

## Drawbacks 

1. **nodeName:** Inflexible & Brittle: It bypasses the scheduler entirely. If the node name changes, the node is down, or you spell it wrong, the pod will never run. It is not suitable for dynamic clusters.
2. **nodeSelector:** Strict Equality Only: It only supports "must match this exact label." You cannot use logical operators like Exists, DoesNotExist, In, or NotIn.
3. **Taints and Tolerations:** Non-Exclusive: It provides "permission" to enter a node but doesn't "force" the pod there. Without additional rules, pods will still prefer to spread to other available nodes.
4. **nodeAffinity:** Complex Syntax: While it "selects" the pod to be run on specifc node, it doesnt restrict unwanted pods from runnig on specific nodes

## BestChoice

**Taints and Tolerations with nodeAffinity**

This combination solves the drawbacks of each: the Taint keeps unwanted Pods out, and the Affinity pulls the correct Pod in.

