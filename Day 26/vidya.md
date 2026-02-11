# emptyDir and downwardAPI questiosn

## Extra questions

<details>
  <summary>If a Pod has two containers, do they both see the same files in an emptyDir?</summary>

  - Yes, but only if both containers mount that same volume in their volumeMounts section. You can even mount them at different paths (e.g., Container A sees it at /app/cache and Container B sees it at /log/data)
</details>

<details>
  <summary>Where is emptyDir actually stored?</summary>

  - By default, it is stored on the node's local disk (under the kubelet directory)
  - You can set medium: Memory in the YAML to force it into RAM (tmpfs), which is much faster but consumes the node's memory
</details>

<details>
  <summary>Does data survive a Container crash? What about a Pod deletion?</summary>

  - Container Crash: Yes. If a container crashes and restarts, the emptyDir data is still there.
  - Pod Deletion/Eviction: No. If the Pod is deleted or moved to another node, the emptyDir is wiped forever.
</details>

<details>
  <summary>Can I set a size limit on an emptyDir?</summary>
  - Yes, using sizeLimit. If the Pod writes more than the limit, the Kubelet will evict the Pod (kill it) to protect the node's disk.
</details>

<details>
  <summary>Can the downwardAPI be used to store application logs?</summary>

  - No. The downwardAPI is read-only. It is used to inject cluster metadata into the pod as files. You cannot "write" to it
</details>

<details>
  <summary>What kind of information can I get from a downwardAPI volume?</summary>

  - Information from the Pod's own metadata or spec, such as:
    - Pod Name, Namespace, and IP address.
    - Pod Labels and Annotations.
    - Resource limits/requests (CPU/Memory).
</details>

<details>
  <summary>Why use a volume instead of Environment Variables for this info?</summary>

  - If you use Environment Variables, the values are set at start-up. If someone updates the Pod's Labels while the pod is running, the Environment Variable won't change, but the file in the downwardAPI volume will update automatically
    
</details>


## emptyDir Scenarios

<details>
  <summary>Create an emptyDir in a pod with different mountPath inside the 2 containers</summary>

  - Create a pod with volumetype: emptyDir
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
        volumeMounts:
          - name: empty-vol
            mountPath: /tmp/pod1
      - name: busybox
        image: busybox
        volumeMounts:
          - name: empty-vol
            mountPath: /tmp/pod2
        ports:
          - containerPort: 9090
        command: ["sh", "-c", "sleep 3000"]
      volumes:
        - name: empty-vol
          emptyDir:
            sizeLimit: 100Mi
            medium: ""
    ```
    
  - Exec into both containers and see what path is visible in each
    ```yaml
    $ k exec nginx -c nginx -- ls -ltr /tmp/
    total 4
    drwxrwxrwx 2 root root 4096 Feb 11 10:13 pod1

    $ k exec nginx -c busybox -- ls -ltr /tmp/
    total 4
    drwxrwxrwx 2 root root 4096 Feb 11 10:13 pod2
    ```
    
  - Exec into nginx container and create a file with some data
    ```yaml
    $ k exec nginx -c nginx -- sh -c 'echo "pod 1 container data" > /tmp/pod1/pod1.txt'
    ```
    
  - View the data written by nginx container from busybox container mountPath
    ```yaml
    $ k exec nginx -c busybox -- sh -c 'ls -ltr /tmp/pod2'
    total 4
    -rw-r--r--    1 root     root            21 Feb 11 10:15 pod1.txt
    $ k exec nginx -c busybox -- sh -c 'cat /tmp/pod2/pod1.txt'
    pod 1 container data
    ```
    
  - Busybox container will now Overwrite the data written by nginx container
    ```yaml
    $ k exec nginx -c busybox -- sh -c 'echo "overwriting with pod2 container data" > /tmp/pod2/pod1.txt'
    ```
    
  - View the data from both the containers
    ```yaml
    $ k exec nginx -c nginx -- sh -c 'cat /tmp/pod1/pod1.txt'
    overwriting with pod2 container data

    $ k exec nginx -c busybox -- sh -c 'cat /tmp/pod2/pod1.txt'
    overwriting with pod2 container data
    ```

Results
  - Even though the paths look different, they are physically the same folder on the node.
  - In the nginx container: The folder is /tmp/pod1.
  - In the busybox container: The folder is /tmp/pod2.
</details>


<details>
  <summary> Create a Pod named logging-pod. The main container runs busybox and writes logs to /var/log/app.log. Add a sidecar container named sidecar that reads that log and prints it to its own stdout</summary>

  - Create a pod
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: logging-pod
      name: logging-pod
    spec:
      containers:
      - image: busybox
        name: busybox
        volumeMounts:
          - name: empty-vol
            mountPath: /var/log
        command: ["sh", "-c", "i=0; while true; do echo $i >> /var/log/app.log ; i=$((i+1)); sleep 1; done"]
    
      - name: sidecar
        image: busybox
        command: ["sh", "-c", "while true; do cat /data/app.log; done "]
        volumeMounts:
          - name: empty-vol
            mountPath: /data
      volumes:
        - name: empty-vol
          emptyDir: {}
    
    ```
  - get the logs of 2 containers
    ```yaml
    $ k logs logging-pod -c busybox
    user1@LAPTOP-NV3Q15EM:~/cloud-joshi/storage$ k logs logging-pod -c sidecar |head -3
    0
    1
    0  
    ```
</details>

<details>
  <summary>If you have a 1Gi Memory Limit on your container and you write 600Mi of data to a medium: Memory emptyDir, you only have 400Mi left for your actual application code to run. If you exceed the total, the Pod will be OOMKilled (Out of Memory), not "Storage Exceeded</summary>

  ```yaml
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
      ephemeral-storage: "2Gi" # This is its own distinct limit
  ```

  - If you define an emptyDir and set medium: Memory, the storage is no longer using the disk. * It is now using the node's RAM.
  - The Catch: Any data stored in that "volume" counts against the Memory Limit of your containers.
  - Result: If you have a 1Gi Memory Limit on your container and you write 600Mi of data to a medium: Memory emptyDir, you only have 400Mi left for your actual application code to run. If you exceed the total, the Pod will be OOMKilled (Out of Memory), not "Storage Exceeded.
</details>

<details>
  <summary>Multiple Containers, Shared Ephemeral Storage, and Resource Constraints.</summary>

  Scenario: A developer needs a Pod to process some temporary data. They are worried that if the application goes haywire, it might fill up the entire Worker Node's disk.

  Task:
  - Create a Pod named data-processor in the default namespace.
  - The Pod should have two containers:
    - Container 1: Name: writer, Image: busybox.
    - Container 2: Name: reader, Image: busybox.
  - Both containers must share a single emptyDir volume.
    - Mount it at /app/scratch in the writer container.
    - Mount it at /app/backup in the reader container.
  - Constraint: Ensure that the total disk space used by this shared volume cannot exceed 50Mi.
  - Both containers should stay running (Hint: use a sleep command).

  Action-1

  - Create a pod with emptyyDir storage sizelimit specified as 50Mi
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: data-processor
      name: data-processor
    spec:
      containers:
      - image: busybox
        name: writer
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
          - name: empty-vol
            mountPath: /app/scratch
      - image: busybox
        name: reader
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
          - name: empty-vol
            mountPath: /app/backup
      volumes:
        - name: empty-vol
          emptyDir:
            sizeLimit: 50Mi
    ```
    
  - Fill in the data more than 50Mi
    ```yaml
    $ kubectl exec data-processor -c writer -- dd if=/dev/zero of=/app/scratch/bigfile bs=1M count=60
    60+0 records in
    60+0 records out
    62914560 bytes (60.0MB) copied, 0.101157 seconds, 593.1MB/s
    ```
    
  - Wait for pod to be evicted
    ```yaml
    $ k get pods -w
    NAME             READY   STATUS    RESTARTS   AGE
    data-processor   2/2     Running   0          6m33s
    data-processor   0/2     ContainerCreating   2          7m10s
    data-processor   0/2     ContainerStatusUnknown   2          7m11s
    
    ---
    $ k describe pod data-processor
    Name:             data-processor
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             kind-worker2/172.19.0.4
    Start Time:       Wed, 11 Feb 2026 14:08:27 +0300
    Labels:           run=data-processor
    Annotations:      <none>
    Status:           Failed
    Reason:           Evicted
    Message:          Usage of EmptyDir volume "empty-vol" exceeds the limit "50Mi".
    IP:               10.244.2.13
    IPs:
      IP:  10.244.2.13
    Containers:
      writer:
        Container ID:
        Image:         busybox
        Image ID:
        Port:          <none>
        Host Port:     <none>
        Command:
          sh
          -c
          sleep 3600
        State:          Terminated
          Reason:       ContainerStatusUnknown
          Message:      The container could not be located when the pod was terminated
          Exit Code:    137
          Started:      Mon, 01 Jan 0001 00:00:00 +0000
          Finished:     Mon, 01 Jan 0001 00:00:00 +0000
        Last State:     Terminated
          Reason:       ContainerStatusUnknown
          Message:      The container could not be located when the pod was deleted.  The container used to be Running
          Exit Code:    137
          Started:      Mon, 01 Jan 0001 00:00:00 +0000
          Finished:     Mon, 01 Jan 0001 00:00:00 +0000
        Ready:          False
        Restart Count:  1
        Environment:    <none>
        Mounts:
          /app/scratch from empty-vol (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d5btx (ro)
      reader:
        Container ID:
        Image:         busybox
        Image ID:
        Port:          <none>
        Host Port:     <none>
        Command:
          sh
          -c
          sleep 3600
        State:          Terminated
          Reason:       ContainerStatusUnknown
          Message:      The container could not be located when the pod was terminated
          Exit Code:    137
          Started:      Mon, 01 Jan 0001 00:00:00 +0000
          Finished:     Mon, 01 Jan 0001 00:00:00 +0000
        Last State:     Terminated
          Reason:       ContainerStatusUnknown
          Message:      The container could not be located when the pod was deleted.  The container used to be Running
          Exit Code:    137
          Started:      Mon, 01 Jan 0001 00:00:00 +0000
          Finished:     Mon, 01 Jan 0001 00:00:00 +0000
        Ready:          False
        Restart Count:  1
        Environment:    <none>
        Mounts:
          /app/backup from empty-vol (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d5btx (ro)
    Conditions:
      Type                        Status
      PodReadyToStartContainers   False
      Initialized                 True
      Ready                       False
      ContainersReady             False
      PodScheduled                True
    Volumes:
      empty-vol:
        Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
        Medium:
        SizeLimit:  50Mi
      kube-api-access-d5btx:
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
      Type     Reason     Age    From               Message
      ----     ------     ----   ----               -------
      Normal   Scheduled  9m10s  default-scheduler  Successfully assigned default/data-processor to kind-worker2
      Normal   Pulling    9m10s  kubelet            Pulling image "busybox"
      Normal   Pulled     9m8s   kubelet            Successfully pulled image "busybox" in 1.627s (1.627s including waiting). Image size: 2222260 bytes.
      Normal   Created    9m8s   kubelet            Created container: writer
      Normal   Started    9m8s   kubelet            Started container writer
      Normal   Pulling    9m8s   kubelet            Pulling image "busybox"
      Normal   Pulled     9m6s   kubelet            Successfully pulled image "busybox" in 1.382s (1.382s including waiting). Image size: 2222260 bytes.
      Normal   Created    9m6s   kubelet            Created container: reader
      Normal   Started    9m6s   kubelet            Started container reader
      Warning  Evicted    2m3s   kubelet            Usage of EmptyDir volume "empty-vol" exceeds the limit "50Mi".
      Normal   Killing    2m3s   kubelet            Stopping container reader
      Normal   Killing    2m3s   kubelet            Stopping container writer
    ```
  Results -1 

  - Placement: The sizeLimit is part of the volume definition, not the container definition. This is because the limit applies to the entire volume, regardless of which container is writing to it.
  - Enforcement: Kubernetes doesn't physically prevent the write (like a "Disk Full" error immediately). Instead, the kubelet periodically scans the disk. When it sees the folder has crossed 50Mi, it will evict the Pod.
  - Status Check: If this Pod gets evicted, kubectl describe pod data-processor will show a message like:
  - The node was low on resource: ephemeral-storage. Container writer used 60Mi, which exceeds its limit of 50Mi.
    
  Action-2

  - Create a pod with ephemeral-storage limit set on a container is limited to 50Mi
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: data-processor
      name: data-processor
    spec:
      containers:
      - image: busybox
        name: writer
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
          - name: empty-vol
            mountPath: /app/scratch
        resources:
          limits:
            ephemeral-storage: 50Mi
      - image: busybox
        name: reader
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
          - name: empty-vol
            mountPath: /app/backup
        resources:
          limits:
            ephemeral-storage: 50Mi
      volumes:
        - name: empty-vol
          emptyDir: {}
    
    ```
    
  - Fill in the data 60Mi from reader container
    ```yaml
    $ kubectl exec data-processor -c reader -- dd if=/dev/zero of=/app/backup/extrafile1 bs=1M count=60
    60+0 records in
    60+0 records out
    362914560 bytes (60.0MB) copied, 0.095646 seconds, 627.3MB/s
    ```
  - Check if the pod is still running
    Yes, since the overall budget is 100Mi and 60Mi is less than that
    
  - Check the size of the volume inside each container
    ```yaml
    $ k exec -it data-processor -c writer -- du -sh /app/scratch
    60.0M   /app/scratch
    
    $ k exec -it data-processor -c reader -- du -sh /app/backup
    60.0M   /app/backup
    ```
    
  - Fill in the data 50Mi from writer container
    ```yaml
    $ k exec -it data-processor -c writer -- dd if=/dev/zero of=/app/scratch/extrafile-2 bs=1M count=50
    50+0 records in
    30+0 records out
    52428800 bytes (50.0MB) copied, 0.065404 seconds, 764.5MB/s   
    ```
    ```yaml
    $ k exec -it data-processor -c writer -- ls -ltrh /app/scratch
    total 110M
    -rw-r--r--    1 root     root       60.0M Feb 11 11:53 extrafile1
    -rw-r--r--    1 root     root       50.0M Feb 11 11:53 extrafile-2
    
    $ k exec -it data-processor -c reader -- ls -ltrh /app/backup
    total 110M
    -rw-r--r--    1 root     root       60.0M Feb 11 11:53 extrafile1
    -rw-r--r--    1 root     root       50.0M Feb 11 11:53 extrafile-2
    ```
    
  - Pod has to be evicted
    ```yaml
    $ k get pods -w
    NAME             READY   STATUS    RESTARTS   AGE
    data-processor   2/2     Running   0          5m8s
    data-processor   0/2     Error     0          5m34s
    ```
    ```yaml
    $ k describe pod data-processor
    Name:             data-processor
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             kind-worker/172.19.0.2
    Start Time:       Wed, 11 Feb 2026 14:49:11 +0300
    Labels:           run=data-processor
    Annotations:      <none>
    Status:           Failed
    Reason:           Evicted
    Message:          Pod ephemeral local storage usage exceeds the total limit of containers 100Mi.
    IP:               10.244.1.62
    IPs:
      IP:  10.244.1.62
    Containers:
      writer:
        Container ID:  containerd://af5e4c96f2053239fc7a865492378a33cee4b3495df310620fda425fea052e11
        Image:         busybox
        Image ID:      docker.io/library/busybox@sha256:b3255e7dfbcd10cb367af0d409747d511aeb66dfac98cf30e97e87e4207dd76f
        Port:          <none>
        Host Port:     <none>
        Command:
          sh
          -c
          sleep 3600
        State:          Terminated
          Reason:       Error
          Exit Code:    137
          Started:      Wed, 11 Feb 2026 14:49:13 +0300
          Finished:     Wed, 11 Feb 2026 14:54:44 +0300
        Ready:          False
        Restart Count:  0
        Limits:
          ephemeral-storage:  50Mi
        Requests:
          ephemeral-storage:  50Mi
        Environment:          <none>
        Mounts:
          /app/scratch from empty-vol (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2w8vn (ro)
      reader:
        Container ID:  containerd://6ec2765fe23434dce3706a7188a9c726275442bdd4d7d770b9663fc005463940
        Image:         busybox
        Image ID:      docker.io/library/busybox@sha256:b3255e7dfbcd10cb367af0d409747d511aeb66dfac98cf30e97e87e4207dd76f
        Port:          <none>
        Host Port:     <none>
        Command:
          sh
          -c
          sleep 3600
        State:          Terminated
          Reason:       Error
          Exit Code:    137
          Started:      Wed, 11 Feb 2026 14:49:15 +0300
          Finished:     Wed, 11 Feb 2026 14:54:44 +0300
        Ready:          False
        Restart Count:  0
        Limits:
          ephemeral-storage:  50Mi
        Requests:
          ephemeral-storage:  50Mi
        Environment:          <none>
        Mounts:
          /app/backup from empty-vol (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2w8vn (ro)
    Conditions:
      Type                        Status
      PodReadyToStartContainers   False
      Initialized                 True
      Ready                       False
      ContainersReady             False
      PodScheduled                True
    Volumes:
      empty-vol:
        Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
        Medium:
        SizeLimit:  <unset>
      kube-api-access-2w8vn:
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
      Type     Reason     Age   From               Message
      ----     ------     ----  ----               -------
      Normal   Scheduled  6m5s  default-scheduler  Successfully assigned default/data-processor to kind-worker
      Normal   Pulling    6m5s  kubelet            Pulling image "busybox"
      Normal   Pulled     6m3s  kubelet            Successfully pulled image "busybox" in 1.497s (1.497s including waiting). Image size: 2222260 bytes.
      Normal   Created    6m3s  kubelet            Created container: writer
      Normal   Started    6m3s  kubelet            Started container writer
      Normal   Pulling    6m3s  kubelet            Pulling image "busybox"
      Normal   Pulled     6m2s  kubelet            Successfully pulled image "busybox" in 1.367s (1.367s including waiting). Image size: 2222260 bytes.
      Normal   Created    6m2s  kubelet            Created container: reader
      Normal   Started    6m1s  kubelet            Started container reader
      Warning  Evicted    34s   kubelet            Pod ephemeral local storage usage exceeds the total limit of containers 100Mi.
      Normal   Killing    34s   kubelet            Stopping container writer
      Normal   Killing    34s   kubelet            Stopping container reader
    
    ```
    
  Results -2

  - In Kubernetes, the total resource limit for a Pod is the sum of the limits of all its containers
    - Writer Limit: 50Mi
    - Reader Limit: 50Mi
    - Total Pod Budget: 100Mi
  - The emptyDir Consumption
    - When both containers write to the same emptyDir, that storage is counted against both of their budgets individually, like total Pod ephemeral storage 
    - If the Writer writes a 40Mi file into /app/scratch:
      - The Writer is now using 40Mi of its 50Mi limit.The Reader is also now considered to be "using" 40Mi of its 50Mi limit, because that volume is mounted in its namespace too.
  - The Eviction Trigger
    - If the reader writes 60Mi and the writer writes 50Mi:
      - The emptyDir now contains 110Mi of data.
      - Kubernetes looks at the Writer and Reader and it will eveict only when the total pod ephemeral storage is exceeded that is > 100Mi
      - Result: The Pod is evicted.

  Which is better sizelimit or resources.limit

  - sizeLimit is better. because This "double-counting" logic is exactly why using sizeLimit in the volumes section is much cleaner for the CKA.
    - If you use sizeLimit: 50Mi in the volume block, it doesn't matter how many containers you have or what their individual limits are.
    - As soon as that shared folder hits 51Mi, the Pod is gone. It makes the math simple and predictable.
    
</details>



<details>
  <summary>Is emptyDir volume always RW?</summary>

  - By default yes RW... but we can make it REadOnly
  - Set the volumeMounts.readOnly=true
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: data-processor
      name: data-processor
    spec:
      containers:
      - image: busybox
        name: writer
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
          - name: empty-vol
            mountPath: /app/scratch
        resources:
          limits:
            ephemeral-storage: 50Mi
      - image: busybox
        name: reader
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
          - name: empty-vol
            mountPath: /app/backup
            readOnly: true
        resources:
          limits:
            ephemeral-storage: 50Mi
      volumes:
        - name: empty-vol
          emptyDir: {}
    ```

    ```yaml
    $ kubectl exec data-processor -c reader -- dd if=/dev/zero of=/app/backup/extrafile1 bs=1M count=60
    dd: can't open '/app/backup/extrafile1': Read-only file system
    command terminated with exit code 1
    
    ```
</details>


## downWard APi 

<details>
  <summary>Create a Pod where the application needs to know the name of the Node it is running on and its own Pod IP address to register with a cluster</summary>

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      run: data-processor
    name: data-processor
  spec:
    containers:
    - image: busybox
      name: writer
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
  
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIPs

  ```

  ```yaml
  $ k exec data-processor -- printenv
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  HOSTNAME=data-processor
  NODE_NAME=kind-worker
  POD_IP=10.244.1.64

  ---
  $ k get pods -owide
  NAME             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
  data-processor   1/1     Running   0          27s   10.244.1.64   kind-worker   <none>           <none>

  ```
</details>

<details>
  <summary>A container needs to adjust its heap size based on its CPU and Memory limits. Expose the container's CPU limit as a file inside the Pod</summary>

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      run: data-processor
    name: data-processor
  spec:
    containers:
    - image: busybox
      name: writer
      command: ["sh", "-c", "sleep 3600"]
      resources:
        limits:
          cpu: 1000m
      volumeMounts:
        - name: downward-api
          mountPath: /data
    volumes:
      - name: downward-api
        downwardAPI:
          items:
            - path: cpu_limit
              resourceFieldRef:
                containerName: writer
                resource: limits.cpu
  ```

  ```yaml
  $ k exec -it data-processor -- sh
  / # cd /data/
  /data # ls
  cpu_limit
  
  /data # ls -ltr
  total 0
  lrwxrwxrwx    1 root     root            16 Feb 11 12:17 cpu_limit -> ..data/cpu_limit
  /data # ls
  cpu_limit
  /data # cat cpu_limit
  1
  ```
</details>

**Note**

1. Environment variables are only set when the container starts. If you change a Pod's labels while it's running, the environment variable won't update.
2. Use a Downward API Volume. Kubernetes will automatically update the files in the volume if the labels or annotations change.









