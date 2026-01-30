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

